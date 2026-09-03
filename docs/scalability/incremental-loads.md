# Incremental vs Full Loads

> *Additional source: Claudia Imhoff et al., "Mastering Data Warehouse Design" (2003), Ch. 8*

## Source interface taxonomy

Before choosing an extraction strategy, classify the interface the source system exposes. Six types exist:

### Snapshot interfaces

**Complete snapshot** — the source sends the *entire* dataset on every extract. The DWH receives all rows, whether changed or not. Simple to implement; requires the ETL layer to compare this snapshot to the prior load to detect what actually changed. Produces large transfer volumes.

**Current snapshot** — the source sends only currently-active rows (e.g. open orders, active accounts). Closed/cancelled rows disappear from the feed. The ETL must infer deletions by noting which rows were present last time but are absent now.

### Delta interfaces

**Columnar delta** — the source sends only the *changed columns* for rows that changed. Efficient but complex: the ETL must merge the partial row against the existing DWH record column by column.

**Row delta** — the source sends complete rows, but only for rows that changed. Efficient and simpler than columnar delta; requires the source to track which rows changed.

**Delta snapshot** — the source sends all rows but includes a change-type indicator column (`I` = insert, `U` = update, `D` = delete). No comparison needed by the ETL; the indicator tells it exactly what to do.

**Transaction interface** — the source sends individual business transactions in chronological order (e.g. a sequence of order events). Each transaction represents one business event; the DWH applies them in sequence to reconstruct current state.

### Database transaction logs

The most reliable extraction method. The source RDBMS records every INSERT, UPDATE, DELETE in a transaction log. The ETL reads the log and replays changes. Tools: Debezium, Oracle GoldenGate, SQL Server CDC. This is the mechanism that makes true CDC possible — without log-based capture, CDC requires application support.

---

## Three techniques for capturing history in the DWH

When loading slowly changing data (e.g. customer addresses, order status), choose a history capture technique based on the source interface available and the volume of changes expected.

### Technique 1: Complete snapshot capture

Load the *entire* source dataset every time. Compare the new snapshot to the previous snapshot row by row to detect changes.

- **Pros:** simple; no reliance on source-side change tracking; complete picture every load
- **Cons:** creates very large staging tables; expensive for high-volume sources; comparison requires full table scans

Use when: source exposes a complete snapshot interface; data volume is manageable.

### Technique 2: Change snapshot capture

Load only rows that have changed since the last extract. Track changes using one of:
- A high-watermark column (`LastModifiedDate`)
- Source-side change tracking (CDC flags)
- A row delta interface

**Handling many-to-many relationships:** when the entity being tracked participates in a many-to-many relationship (e.g. a customer who belongs to multiple groups), use an **associative entity** to model the relationship history — don't duplicate the group assignments on the customer row. This keeps the change snapshot small and the history clean.

- **Pros:** much smaller volumes than complete snapshot; scales to large sources
- **Cons:** depends on source having reliable change tracking; deletes are hard to detect; initial load is still full

### Technique 3: Change snapshot with delta capture

The most sophisticated approach. Separates two types of change:

- **Characteristic changes** (descriptive attributes: name, address, category) → stored in a *snapshot history table* using SCD2-style effective/expiration dates
- **Measurable changes** (quantitative values: account balance, credit limit, order value) → stored in a *delta table* that records each change event

**Change detection using CRC:** instead of comparing columns one by one (expensive at scale), compute a 32-bit Cyclic Redundancy Check (CRC) over all tracked columns for each row. Store the CRC alongside the record. On the next load:
1. Compute CRC for each incoming row
2. Compare incoming CRC to stored CRC
3. If different → the row changed; process it
4. If same → no change; skip

The false positive rate (two different rows producing the same CRC) is approximately 1 in 4 billion — negligible for DWH use.

**Current indicator flag:** each row in the history table carries a `CurrentFlag` (or `IsCurrent`) column. Set to `Y` for the active row; set to `N` when a new version is created. Allows fast retrieval of the current state without date range comparison.

**Database triggers:** in some implementations, source-side database triggers detect changes and write them to a change capture table, which the ETL reads. Reliable but requires source-side schema changes.

---

## Load log pattern

Maintain a load log table that records every extract run:

| Column | Purpose |
|---|---|
| `LoadID` | Unique load identifier |
| `SourceSystem` | Which system was extracted |
| `LoadStartDT` | When the extract began |
| `LoadEndDT` | When the extract completed |
| `RowsExtracted` | How many rows were received |
| `RowsInserted` | New rows added to DWH |
| `RowsUpdated` | Existing rows updated |
| `WatermarkValue` | Watermark after this load (for next run) |

The load log enables: auditing, re-processing (re-run from a specific load), and capacity planning.

---

## Simultaneous vs postload delivery

When new data arrives, dimension and fact loads must be sequenced correctly:

**Postload delivery:** dimension records are fully loaded and committed *before* the fact load begins. Fact rows can always resolve their foreign key lookups. Simple and safe — the standard approach.

**Simultaneous delivery:** dimension and fact rows arrive together in the same feed. The ETL must buffer fact rows until their dimension counterparts are committed, then release them. Required when the source sends the first transaction for a new entity (the entity's dimension row and its first fact row arrive in the same batch).

---

## Full load

Re-load the entire dataset from the source on every run.

```
Every night: DELETE all rows → INSERT all rows from source
```

**Simple to implement** — no need to track what changed.

**Does not scale** — as the source grows to 100M rows, loading everything nightly becomes prohibitive in time and cost.

**When full load is the only option:**
- Source has no `updated_at` column or CDC mechanism
- Source data can be corrected retroactively (backdated changes)
- Table is small enough that full reload is fast

## Incremental load

Process only records that are **new or changed** since the last run.

### Watermark pattern

Track the last processed timestamp. On each run, load only records newer than the watermark.

```python
last_watermark = get_watermark('silver.customers')  # e.g. 2026-08-05 23:00:00

new_data = (
    spark.read.table('bronze.crm_customer')
    .filter(col('updated_at') > last_watermark)
)

# Process and merge new_data into silver
merge_into_silver(new_data)

# Update watermark to max updated_at in this batch
update_watermark('silver.customers', new_data.agg(max('updated_at')))
```

**Prerequisite:** source must have a reliable `updated_at` or `created_at` column. If the source can backdate changes, the watermark will miss them.

### CDC — Change Data Capture

Rather than polling for changed rows, CDC captures every INSERT/UPDATE/DELETE at the database transaction level. The stream of changes is delivered to the pipeline in order.

Tools: Debezium, Azure Data Factory CDC, Kafka Connect.

CDC gives you **exactly what changed and how** — including deletes, which a watermark approach cannot detect.

### MERGE (upsert) pattern

The standard way to apply incremental changes to a Silver or Gold table:

```sql
MERGE INTO silver.customers AS tgt
USING new_data AS src ON tgt.CustomerBK = src.CustomerBK
WHEN MATCHED AND tgt.HashDiff <> src.HashDiff THEN
    UPDATE SET tgt.CustomerName = src.CustomerName, ...
WHEN NOT MATCHED THEN
    INSERT (CustomerBK, CustomerName, ...)
    VALUES (src.CustomerBK, src.CustomerName, ...)
```

The `HashDiff` column is a hash of all tracked attribute columns — change detection in O(1) per row.

## Choosing the right strategy

| Strategy | When to use |
|---|---|
| Full load | Small tables, no reliable change tracking, source corrects history |
| Watermark incremental | Medium-large tables with reliable `updated_at`, no backdating |
| CDC | Large tables, deletes matter, must capture every state transition |

## The idempotency requirement

Regardless of strategy, **incremental loads must be idempotent** — re-running the same load must produce the same result, not duplicate rows.

Design patterns:
- Use MERGE instead of INSERT for upserts
- Use `IF NOT EXISTS` guards on table creation
- Use a watermark that can be re-run safely (process everything > last_watermark, apply MERGE)

See [Idempotency](../best-practices/idempotency.md) for more.

## Delta Lake Change Data Feed (CDF)

When you can't get CDC from the source system directly, enable CDF on the Bronze Delta table. Silver subscribes to the Bronze change feed instead of re-scanning the full table.

```python
# Enable CDF on Bronze table
spark.sql("""
    ALTER TABLE bronze.crm_customers
    SET TBLPROPERTIES (delta.enableChangeDataFeed = true)
""")

# Silver reads only changes since last processed version
last_version = get_last_processed_version("silver.crm_customers")

changes = (
    spark.read
    .format("delta")
    .option("readChangeFeed", "true")
    .option("startingVersion", last_version + 1)
    .table("bronze.crm_customers")
)
# _change_type column: 'insert', 'update_preimage', 'update_postimage', 'delete'
inserts_and_updates = changes.filter(col("_change_type").isin("insert", "update_postimage"))
```

CDF works as a streaming source too — `readStream` with `readChangeFeed=true` gives a continuous stream of Bronze changes for Silver to process in near-real-time.

## Late-arriving data

Late-arriving data = records that have a business timestamp in the past but arrive in the pipeline now. Example: a transaction from 2026-07-28 arrives on 2026-08-07 because the source system had a backlog.

Handling strategies:

| Strategy | When to use | Trade-off |
|---|---|---|
| **Ignore** | Data is always timely; late records are rare and acceptable | Simple; late records missed |
| **Reprocess affected partitions** | Partitioned by business date; can re-run the Silver job for the old date | Correct; can be expensive if many partitions |
| **Watermark + tolerance window** | Streaming: accept records up to N hours late | Configurable; records beyond the window are dropped |
| **Full reload** | Small tables; late records common due to backdating at source | Correct; doesn't scale |

In Spark Structured Streaming, use `withWatermark` to define the late-data tolerance:

```python
stream \
    .withWatermark("EventTimestamp", "2 hours") \  # accept records up to 2 hours late
    .groupBy(window("EventTimestamp", "1 hour"), col("Region")) \
    .agg(sum("Amount"))
```

Records arriving more than 2 hours past the watermark are dropped. State is cleaned up after the watermark advances.

## Exactly-once semantics in streaming

Spark Structured Streaming provides **at-least-once** delivery by default — on failure and restart, the last unconfirmed micro-batch is re-processed. Combined with idempotent sinks (MERGE into Delta), this gives effective exactly-once behavior:

```
Kafka source → Spark (micro-batch) → Delta MERGE (idempotent)
                    ↑ checkpointed offsets
```

If the job crashes mid-batch, it restarts from the last checkpointed Kafka offset. The MERGE re-applies the same rows — because MERGE is idempotent, the result is the same as if the batch had succeeded on the first try.

**True exactly-once** (no re-processing at all) requires both the source to support offset-based replay (Kafka does) and the sink to support atomic offset commit + data write in one transaction — which Kafka sinks can provide via transactional producers.
