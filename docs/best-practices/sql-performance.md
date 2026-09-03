# SQL Performance Patterns

Writing SQL that stays fast as data grows — primarily for analytical databases and MPP engines (Fabric, Synapse, Snowflake, BigQuery, Databricks SQL).

## Window functions over correlated subqueries

### The problem

A correlated subquery or `OUTER APPLY` with `TOP 1 … ORDER BY` forces the engine into a **nested-loop**: for every row in the outer table it runs a separate inner query, scans the inner table, sorts it, and returns one row. On MPP columnar stores this does not parallelize and kills performance at scale.

```sql
-- Anti-pattern: OUTER APPLY with TOP 1 ORDER BY
SELECT
    be.[AccountID],
    be.[EntryNo],
    be.[RunningBalance] - prev.PrevBalance AS DiffBalance
FROM finance.AccountEntry be
OUTER APPLY (
    SELECT TOP 1 prevbe.[RunningBalance] AS PrevBalance
    FROM finance.AccountEntry prevbe
    WHERE prevbe.[AccountID] = be.[AccountID]
      AND (prevbe.[PostedAt] < be.[PostedAt]
           OR (prevbe.[PostedAt] = be.[PostedAt]
               AND prevbe.[EntryNo] < be.[EntryNo]))
    ORDER BY prevbe.[PostedAt] DESC, prevbe.[EntryNo] DESC
) prev
```

### The solution

`LAG()` computes the previous value within a partition in a **single scan**. The engine processes all rows at once using the same sort that the correlated subquery also paid for — no nested lookups.

```sql
-- Correct: LAG window function
SELECT
    [AccountID],
    [EntryNo],
    [RunningBalance]
        - COALESCE(LAG([RunningBalance])
            OVER (PARTITION BY [AccountID] ORDER BY [PostedAt], [EntryNo]),
          0) AS DiffBalance
FROM finance.AccountEntry
```

Window functions also make the intent explicit: "previous value in the series, ordered by X, within group Y" reads directly from the `OVER` clause.

**When to apply:** any "delta to previous row", "running total", "rows since last event", or "rank within group" pattern. If you see `TOP 1 … ORDER BY` inside a correlated subquery or `APPLY`, ask whether a window function covers the same logic.

---

## INNER JOIN as an existence filter

`LEFT JOIN + WHERE IS NULL` and `NOT EXISTS` are the conventional ways to exclude rows with no match in a dimension. But when the goal is simply to **filter the fact set down**, an `INNER JOIN` on the dimension does the same job in one step — and the engine can push the filter earlier in the plan.

```sql
-- Anti-pattern: LEFT JOIN + WHERE clause to filter
FROM fact.Sales s
LEFT JOIN dim.DimProduct p ON p.ProductKey = s.ProductKey
WHERE p.ProductKey IS NOT NULL
```

```sql
-- Correct: INNER JOIN as filter (comment is required)
-- INNER JOIN on DimProduct acts as an existence filter:
-- source rows with no matching product are intentionally excluded.
INNER JOIN dim.DimProduct p ON p.ProductKey = s.ProductKey
```

Always add a comment when an `INNER JOIN` is doing filtering work rather than supplying columns — it looks like a lookup join but behaves differently (it drops rows). Without the comment the next developer may "fix" it to a `LEFT JOIN`, silently introducing fan-out.

---

## Static lookup tables: VALUES constructor

Reference data small enough to hardcode should live in a **dedicated view**, not scattered across procedure bodies or inline query strings. When writing that view, use the `VALUES` constructor instead of `UNION ALL` chains.

```sql
-- Anti-pattern: UNION ALL chain
CREATE VIEW stg.OrderStatusType AS
SELECT 'Pending'    AS Id, 'Pending approval'   AS Label
UNION ALL
SELECT 'Confirmed',         'Order confirmed'
UNION ALL
SELECT 'Shipped',           'Shipped to customer'
```

```sql
-- Correct: VALUES constructor
CREATE VIEW stg.OrderStatusType AS
SELECT CAST(t.[Id] AS VARCHAR(40)) AS [Id], CAST(t.[Label] AS VARCHAR(200)) AS [Label]
FROM (VALUES
    ('Pending',   'Pending approval'),
    ('Confirmed', 'Order confirmed'),
    ('Shipped',   'Shipped to customer')
) AS t ([Id], [Label])
```

The `VALUES` form aligns data in columns, is easier to extend (add a row → one line), and keeps type declarations visible at the top. Both forms produce a constant scan — the gain is in maintainability, not runtime performance.

Centralizing the view also means there is one place to add or rename rows, instead of hunting for the hardcoded string across mapping files or procedure bodies.

---

## Incremental filter asymmetry with calendar/date joins

Applying an incremental watermark filter to a temp table that is later joined to a **date dimension or calendar** is a recurring correctness trap.

### Why it breaks

Imagine a temp table that computes an "is active" flag per account from recent transactions, then a calendar join that expands the result to one row per account per month. You add an incremental filter to the flag computation — only process accounts with transactions since the last watermark.

The problem: the flag now only reflects *recent* changes. An account that was active in past months but had no transactions in the incremental window appears INACTIVE in the temp table — even though its historical month rows are correct. The calendar join then re-expands all months for any changed account and sees wrong flag values for old months, silently corrupting history.

```sql
-- Anti-pattern: incremental filter inside a flag temp table that feeds a calendar join
SELECT AccountId, MAX(CASE WHEN TrxDate >= '2024-01-01' THEN 1 ELSE 0 END) AS IsActive
FROM Transactions
WHERE SysUpdDT >= @WatermarkDT   -- wrong: skips unchanged accounts, old-month flags become incorrect
GROUP BY AccountId
```

### The correct approach

Remove the incremental filter from the temp table. The cost is bounded by the table's clustered key. Add a comment explaining the deliberate absence:

```sql
-- No incremental filter here on purpose: this flag must reflect COMPLETE history
-- for the fixed window every run. The outer query re-expands ALL months for any
-- changed account via the unrestricted calendar join, so an incremental filter
-- here would wrongly show untouched old months as INACTIVE.
-- Cost is bounded — clustered on TrxDate, ~6-9M rows.
SELECT AccountId, MAX(CASE WHEN TrxDate >= '2024-01-01' THEN 1 ELSE 0 END) AS IsActive
FROM Transactions
GROUP BY AccountId
```

The incremental filter in the **outer query** (which drives which accounts to reload at all) remains — only the inner temp table is unrestricted.

---

## Dead join removal

When a column is dropped from the DDL, remove every join that existed solely to supply it.

```sql
-- Before: Article and Product columns exist in DDL → joins serve a purpose
LEFT JOIN [AnalyticsLibrary].[TD_Article] ar   ON ar.[ArticleId]  = base.[ArticleId]
LEFT JOIN [AnalyticsLibrary].[TD_Product] prod ON prod.[ProductId] = ar.[ProductId]

-- After: Article/Product columns removed from DDL → both joins are now dead
-- Remove them; they contribute nothing and run on every pipeline execution.
```

Leaving a dead join:
- pays its join cost on every pipeline run (potentially scanning millions of rows per load)
- misleads readers into thinking the column is used somewhere downstream
- can mask future fan-out bugs if the orphaned join table later gains duplicate rows

**Check pattern:** when removing a DDL column, grep the mapping and procedure body for every table that was joined to supply that column. If the only output column from that join is gone, the join is dead — remove it entirely.
