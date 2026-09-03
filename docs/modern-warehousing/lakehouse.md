# Lakehouse Architecture

## The problem it solves

**Data Lakes** were cheap and flexible — store anything in raw format — but had no ACID guarantees, no schema enforcement, and poor query performance. BI tools couldn't query them directly.

**Data Warehouses** were fast and reliable but expensive, rigid, and couldn't handle unstructured or semi-structured data well.

A **Lakehouse** combines the best of both:
- Cheap object storage (like a lake)
- ACID transactions, schema enforcement, and SQL performance (like a warehouse)

## How it works

The key enabler is an **open table format** (Delta Lake, Apache Iceberg, Apache Hudi) that adds a metadata/transaction layer on top of Parquet files in object storage.

```
Object Storage (S3 / ADLS / GCS)
└── Parquet files
└── Transaction log (Delta / Iceberg / Hudi metadata)
         ↑
    Compute engines (Spark, Trino, DuckDB, Databricks SQL, Fabric SQL)
         ↑
    BI tools / notebooks / APIs
```

Because the format is open, multiple compute engines can query the same data without copying it.

## Key capabilities

| Capability | What it enables |
|---|---|
| ACID transactions | No partial writes; concurrent reads and writes safe |
| Schema enforcement | Define and enforce column types, reject bad data |
| Schema evolution | Add columns, change types without rewriting the table |
| Time travel | Query the table as it was at any prior point |
| Change data feed | Downstream systems can subscribe to row-level changes |
| Partition pruning | Skip irrelevant partitions for fast queries |

## Lakehouse vs Data Warehouse

| | Data Warehouse | Lakehouse |
|---|---|---|
| Storage cost | High | Low (object storage) |
| Semi-structured data | Limited | Native |
| ML / data science access | Requires export | Direct (Spark, Python) |
| SQL query performance | Very high | High (depends on format/engine) |
| ACID | Yes | Yes (with Delta/Iceberg/Hudi) |
| Vendor lock-in | High | Low (open formats) |

## Microsoft Fabric as a Lakehouse

Microsoft Fabric combines a OneLake (ADLS Gen2-backed object storage) with:
- **Lakehouses** — Spark notebooks + Delta tables
- **SQL Warehouses** — full SQL endpoint on the same Delta tables
- **Semantic models** — Power BI directly on top

This is the Medallion architecture on Fabric: Bronze/Silver in Lakehouses, Gold in SQL Warehouses.

## Delta Lake internals

The `_delta_log/` directory is the heart of Delta Lake — a sequential log of every table operation.

```
my_table/
  _delta_log/
    00000000000000000000.json   ← CREATE TABLE
    00000000000000000001.json   ← first INSERT
    00000000000000000002.json   ← MERGE (upsert)
    00000000000000000010.checkpoint.parquet  ← compacted snapshot (every 10 commits)
  part-00000-abc.parquet
  part-00001-def.parquet
```

Each JSON commit file records: which Parquet files were **added**, which were **removed**, and any **metadata changes** (schema, partition definition). Reading the table = replaying the log from the last checkpoint forward.

**Checkpoint files** (`*.checkpoint.parquet`) are generated every 10 commits by default. They compact the log so queries don't replay thousands of JSON files from the beginning.

**VACUUM** removes Parquet files that are no longer referenced by any log entry beyond the retention threshold:

```sql
VACUUM delta.`/mnt/data/my_table` RETAIN 168 HOURS
```

After VACUUM, time travel queries older than the retention period fail — the underlying Parquet files are gone.

## Iceberg vs Delta Lake

| | Delta Lake | Apache Iceberg |
|---|---|---|
| Primary ecosystems | Databricks, Microsoft Fabric, Spark | Snowflake, AWS, multi-engine (Flink, Trino, Spark) |
| Metadata format | JSON transaction log + Parquet checkpoints | Avro manifest files + Parquet/ORC data |
| Hidden partitioning | No — must specify partition column explicitly | Yes — partition transforms (bucket, truncate, year/month/day) invisible to queries |
| Concurrent writes | Optimistic concurrency | Optimistic concurrency; better multi-writer support |
| Catalog requirement | Spark catalog / Unity Catalog | Hive Metastore, AWS Glue, REST catalog, Nessie |
| UniForm / interop | Delta UniForm exports Iceberg metadata for cross-engine reads | Native multi-engine |
| Best for | Azure / Microsoft / Databricks-first stacks | Multi-cloud, Snowflake, AWS-first stacks |

**When to choose Iceberg:** your data must be read by both Snowflake and Spark (or Flink), or you're on AWS with native Glue/Athena integration.  
**When to choose Delta Lake:** you're on Microsoft Fabric or Databricks, or you want the simplest setup with Unity Catalog.

## Fabric OneLake

OneLake is Microsoft Fabric's single logical data lake — one storage account per tenant, shared by all workspaces and all Fabric items (Lakehouses, Warehouses, Semantic Models).

```
OneLake (tenant-wide, ADLS Gen2-backed)
├── Workspace A
│   ├── Lakehouse (Bronze) ← Delta tables, notebooks write here
│   └── Warehouse (Gold)  ← SQL endpoint on Delta tables
└── Workspace B
    └── Lakehouse          ← Shortcut to Workspace A Bronze ← zero-copy reference
```

**OneLake Shortcuts** let one workspace reference data in another — or in external ADLS / S3 / GCS — without physically copying it. A Gold Lakehouse in the domain workspace can shortcut to Bronze in the platform workspace: one copy of the data, multiple consumers.

**One format:** all Lakehouse tables are Delta on Parquet. The SQL Warehouse endpoint reads the same files with full ACID semantics. Spark and T-SQL see the same table.
