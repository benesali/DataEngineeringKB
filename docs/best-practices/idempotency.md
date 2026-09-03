# Idempotency

## What it means

An idempotent pipeline or script produces **the same result whether run once or ten times**. Running it again on the same data doesn't create duplicates, corrupt rows, or leave partial state.

This is one of the most important properties of a production pipeline — because pipelines fail and are re-run all the time.

## Why it matters

- Pipeline fails at step 3 of 5 → you fix the bug → you re-run from the beginning → without idempotency, steps 1–2 now have duplicate data
- A release script runs twice (deployment mishap) → without idempotency, every `INSERT` doubled, every `CREATE TABLE` errors out

## Patterns by operation

### CREATE TABLE

```sql
-- Non-idempotent (fails on second run)
CREATE TABLE silver.customers (...)

-- Idempotent
IF OBJECT_ID('silver.customers') IS NULL
    CREATE TABLE silver.customers (...)
```

### ALTER TABLE ADD COLUMN

```sql
-- Non-idempotent
ALTER TABLE silver.customers ADD COLUMN LoyaltyTier VARCHAR(50)

-- Idempotent
IF NOT EXISTS (
    SELECT 1 FROM sys.columns
    WHERE object_id = OBJECT_ID('silver.customers') AND name = 'LoyaltyTier'
)
    ALTER TABLE silver.customers ADD COLUMN LoyaltyTier VARCHAR(50) NULL
```

### INSERT

```sql
-- Non-idempotent (duplicates on re-run)
INSERT INTO dim.Country (CountryCode, CountryName)
VALUES ('DE', 'Germany')

-- Idempotent
IF NOT EXISTS (SELECT 1 FROM dim.Country WHERE CountryCode = 'DE')
    INSERT INTO dim.Country (CountryCode, CountryName)
    VALUES ('DE', 'Germany')
```

Or use MERGE — naturally idempotent for upserts.

> **Fabric SQL Warehouse:** error 24735 is raised when adding a `NOT NULL` column to an existing table — even with a default value supplied. Always declare new columns `NULL` in `ALTER TABLE ADD`. The ETL framework handles the nullability at load time, so a nullable physical column is correct and safe.

### SELECT INTO / CTAS (backup tables)

```sql
-- Non-idempotent (fails if table already exists)
SELECT * INTO backup.Customers_20260806 FROM silver.customers

-- Idempotent
IF OBJECT_ID('backup.Customers_20260806') IS NULL
BEGIN
    DECLARE @sql NVARCHAR(MAX) = N'SELECT * INTO backup.Customers_20260806 FROM silver.customers'
    EXEC sp_executesql @sql
END
```

### Literal typing in shared temp tables

When a temp table is built by CTAS and then extended by a second INSERT, the column type is inferred from the **first** INSERT. If the second INSERT has a longer string value, it is silently truncated.

```sql
-- Anti-pattern: 'ACTIVE' (6 chars) → engine infers VARCHAR(6) for the column
SELECT 'ACTIVE' AS ActivityFlag, AccountId INTO #ActivityData FROM ...

-- Second INSERT: 'INACTIVE' (8 chars) → silently truncated to 'ACTIVEIN'
INSERT INTO #ActivityData SELECT 'INACTIVE', AccountId FROM ...
```

Fix by casting literals in the first INSERT to the widest type needed across all inserts:

```sql
-- Correct: explicit width covers all future values
SELECT CAST('ACTIVE' AS VARCHAR(100)) AS ActivityFlag, AccountId INTO #ActivityData FROM ...
INSERT INTO #ActivityData SELECT 'INACTIVE', AccountId FROM ...   -- fits safely
```

The same applies to flag columns that will hold integers: `CAST(0 AS INT)` is safer than letting the engine infer from a bare `0`.

### UPDATE

Naturally idempotent if the SET values are deterministic:

```sql
UPDATE silver.customers
SET CountryCode = 'DE'
WHERE CountryCode = 'Deutschland'
-- Running twice produces the same result
```

### DROP

```sql
-- Idempotent
IF OBJECT_ID('silver.old_table') IS NOT NULL
    DROP TABLE silver.old_table
```

## MERGE — the general solution for upserts

MERGE combines INSERT and UPDATE in one idempotent operation. It is the standard pattern for incremental loads.

```sql
MERGE INTO silver.customers AS tgt
USING source_data AS src ON tgt.CustomerBK = src.CustomerBK
WHEN MATCHED THEN UPDATE SET tgt.CustomerName = src.CustomerName
WHEN NOT MATCHED THEN INSERT (CustomerBK, CustomerName) VALUES (src.CustomerBK, src.CustomerName)
```

Re-running with the same source data updates existing rows to the same values — no duplicates.

## Watermark idempotency

The watermark records the last successfully processed timestamp. If the watermark state is lost (database dropped, state table corrupted), the pipeline doesn't know where to resume — it might:
- **Re-process everything** (if it defaults to the beginning of time) → duplicates
- **Skip everything** (if it defaults to now) → data gap

**Design the watermark to be recoverable:**

```python
def get_watermark(table_name: str) -> datetime:
    # If no watermark row exists, return a safe default (e.g. 30 days ago)
    # Never return None or raise — always return a value the pipeline can continue from
    row = spark.sql(f"SELECT MAX(WatermarkDT) FROM SystemLog.TS_Watermark WHERE TableName = '{table_name}'").collect()[0][0]
    return row if row is not None else datetime(2020, 1, 1)
```

Combined with an **idempotent MERGE** at the target, re-processing overlapping data from a recovered watermark is safe — MERGE updates existing rows to the same values and inserts new rows once.

**Store watermarks in a durable, backed-up table** — not in a file on the lakehouse where it can be accidentally deleted with the table it tracks.

## Exactly-once in Spark Structured Streaming

Spark Streaming provides at-least-once by default. The checkpoint directory stores:
- Kafka offsets (which messages have been read)
- State (for stateful aggregations like `groupBy + count`)

On restart after failure, Spark replays from the last committed offset. With a Delta MERGE sink (idempotent), re-applying the same batch produces the same result — giving effective exactly-once behavior at the data level.

**Do not delete the checkpoint directory** — it is the only record of which messages have been processed. Deleting it forces a cold restart from the beginning (or the configured starting offset), risking duplicates or gaps.

```python
stream.writeStream \
    .format("delta") \
    .option("checkpointLocation", "/mnt/checkpoints/my_stream") \  # permanent, backed-up path
    .outputMode("append") \
    .table("bronze.events")
```

For Kafka sources specifically: Spark commits Kafka offsets to the checkpoint after each micro-batch succeeds. If the write to Delta fails, offsets are not committed — the batch is retried from the same Kafka offset, re-applying the same records. With MERGE (idempotent), this is safe.
