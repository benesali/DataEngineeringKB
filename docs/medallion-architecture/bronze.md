# Bronze Layer (Raw)

> *Source: Building Medallion Architectures — Piethein Strengholt.  
> Code examples: [chapter-05](https://github.com/pietheinstrengholt/building-medallion-architectures-book/tree/main/chapter-05)*

## Purpose

The Bronze layer is the **landing zone for raw data**. Data arrives exactly as it does from the source — no business transformations, no cleaning, no schema changes. It is an immutable historical archive.

## Key characteristics

- **Immutable** — data is never updated or deleted; new loads are appended or versioned
- **Schema-on-read or schema-on-write** — often ingested without strict schema enforcement
- **High volume** — stores every delivery from every source
- **Source of truth for recovery** — if a downstream layer is corrupted, Bronze is the replay source

## What lives in Bronze

- Full CSV / JSON exports from source systems
- CDC (Change Data Capture) streams
- API response payloads
- Kafka / event stream messages
- Files delivered by partners (SFTP, blob storage)

## Full load vs incremental

| Mode | When to use | How it works |
|---|---|---|
| Full load | Source provides complete dataset each time | Overwrite or version-stamp the entire dataset |
| Incremental (append) | Source provides only new records | Add new rows; existing rows unchanged |
| Incremental (merge) | Source provides changed records (CDC) | Upsert using MERGE / APPLY CHANGES |

## Schema management in Bronze

**Schema-on-read** is common — store data as-is (JSON, CSV, Parquet), infer schema at query time. Good for landing unknown or frequently changing data.

**Schema-on-write** with `mergeSchema=true` (Delta Lake) allows schema evolution while still enforcing a baseline structure.

When Bronze becomes "queryable" (Delta format), switch to schema-on-write for reliability.

## Technical validation

Bronze is the right place for **technical checks** — not business rules:

- Is the file format valid? (parseable JSON, correct CSV delimiter)
- Are expected columns present?
- Are data types parseable? (a date column contains date-shaped strings)
- Row count within expected range?

Failed records can be:
- **Intrusive**: halt the pipeline (use when downstream cannot handle bad data)
- **Non-intrusive**: quarantine bad rows, let good rows proceed

## Storage format

Delta Lake (on Parquet) is the standard choice. It gives Bronze:
- ACID transactions (no partial loads)
- Schema evolution via `mergeSchema`
- Time travel (query any previous version)
- Change data feed for downstream Silver

## Auto Loader (Databricks)

Databricks Auto Loader uses Spark Structured Streaming with the `cloudFiles` format to incrementally ingest files as they arrive in object storage — without scanning the entire directory each run.

```python
df = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/mnt/checkpoints/crm_orders/schema")
    .load("/mnt/raw/crm/orders/")
)

df.writeStream \
  .format("delta") \
  .option("checkpointLocation", "/mnt/checkpoints/crm_orders") \
  .option("mergeSchema", "true") \
  .outputMode("append") \
  .table("bronze.crm_orders")
```

Auto Loader tracks which files have been processed via a checkpoint — restarting the stream doesn't reprocess already-loaded files.

## Landing zone → Bronze promotion pattern

Raw files (CSV, JSON, Parquet from SFTP, API, partner drops) first land in an unstructured **landing zone** (a raw folder in object storage). A Bronze ingestion job then:

1. Reads files from the landing zone
2. Adds metadata columns (`_source_file`, `_load_timestamp`, `_batch_id`)
3. Writes to a Delta Bronze table
4. Optionally moves the source file to an archive folder

```python
from pyspark.sql.functions import input_file_name, current_timestamp, lit

df = (
    spark.read
    .option("header", "true")
    .csv("/mnt/landing/crm/customers/")
    .withColumn("_source_file", input_file_name())
    .withColumn("_load_timestamp", current_timestamp())
)

df.write.format("delta").mode("append").saveAsTable("bronze.crm_customers")
```

## Partitioning strategy

Bronze tables are typically partitioned by **ingestion date** — not by business date. This avoids rewriting old partitions when historical data arrives late.

```python
df.write \
  .format("delta") \
  .partitionBy("_load_date") \
  .mode("append") \
  .saveAsTable("bronze.crm_customers")
```

Common partition granularities:
- **Daily** (`LoadDate`) — for most batch pipelines
- **Hourly** (`LoadDate`, `LoadHour`) — for near-real-time or high-volume streaming
- **Year/Month** — for very large historical tables where daily partitions create too many directories

Avoid partitioning on source business keys (CustomerID, OrderID) — cardinality is too high and creates millions of tiny files.
