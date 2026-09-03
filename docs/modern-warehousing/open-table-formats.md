# Delta Lake & Open Table Formats

## What is an open table format?

An open table format is a layer on top of Parquet files in object storage that adds:
- A **transaction log** — record of every write operation
- **Schema metadata** — column definitions, statistics
- **ACID semantics** — atomic commits, isolation, durability

The three major formats:

| Format | Originates from | Primary ecosystem |
|---|---|---|
| **Delta Lake** | Databricks (open-sourced) | Databricks, Microsoft Fabric, Spark |
| **Apache Iceberg** | Netflix (open-sourced) | Snowflake, AWS, multi-engine |
| **Apache Hudi** | Uber (open-sourced) | AWS EMR, Spark, streaming |

## Delta Lake

The most widely used format in Azure / Microsoft ecosystems.

### Transaction log

Every write (insert, update, delete, schema change) creates a new JSON entry in `_delta_log/`. The table's current state is derived by replaying the log.

```
_delta_log/
  00000000000000000000.json   ← initial table creation
  00000000000000000001.json   ← first data load
  00000000000000000002.json   ← schema change (new column)
  00000000000000000010.checkpoint.parquet  ← compact snapshot
```

### Time travel

Query any previous version of the table:

```sql
-- Query as of 7 days ago
SELECT * FROM my_table TIMESTAMP AS OF date_sub(current_date(), 7)

-- Query version 5
SELECT * FROM my_table VERSION AS OF 5
```

### MERGE (upsert)

The core operation for SCD2 and incremental loads:

```sql
MERGE INTO target t
USING source s ON t.CustomerBK = s.CustomerBK
WHEN MATCHED AND t.HashDiff <> s.HashDiff THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT (...)
```

### Vacuum

Old Parquet files accumulate. `VACUUM` deletes files older than the retention threshold (default 7 days):

```sql
VACUUM my_table RETAIN 168 HOURS
```

After vacuum, time travel beyond the retention period is no longer possible.

## Apache Iceberg

Increasingly common in multi-cloud and Snowflake environments. Key differences from Delta:

- **Hidden partitioning** — partition transforms are invisible to queries; no partition pruning errors
- **Better concurrency** — optimistic concurrency control for multi-writer scenarios
- **Table format v2** — row-level deletes without rewriting entire files

## Choosing a format

- **Azure / Microsoft Fabric / Databricks** → Delta Lake
- **Snowflake / AWS** → Iceberg (native support)
- **Streaming / AWS EMR** → Hudi or Iceberg

## Delta Lake Z-ordering

Z-ordering co-locates related data within Parquet files based on multiple column values. Files get min/max statistics for Z-ordered columns, enabling **data skipping** — the query engine skips entire files that cannot contain matching values.

```sql
-- Co-locate data by CustomerID and ProductID within each date partition
OPTIMIZE silver.orders ZORDER BY (CustomerID, ProductID)
```

After Z-ordering, a query `WHERE CustomerID = 12345 AND ProductID = 'P001'` reads far fewer files. The Z-order curve maps multi-dimensional column values into a single dimension so that nearby values in all dimensions cluster together in the file layout.

**Liquid Clustering** (Delta Lake 3.x / Databricks): the next generation. No need to pick a static Z-order key — clustering is applied incrementally on writes, adapts over time, and doesn't require a full table OPTIMIZE.

```sql
-- Define clustering columns at table creation
CREATE TABLE silver.orders (...)
CLUSTER BY (CustomerID, ProductID);
```

## Change Data Feed (CDF)

CDF makes Delta Lake tables publish a stream of row-level changes (inserts, updates, deletes). Downstream consumers subscribe to only the changed rows — eliminating the need to re-scan the full table.

Enable CDF on a table:

```sql
ALTER TABLE bronze.crm_customers
SET TBLPROPERTIES (delta.enableChangeDataFeed = true);
```

Read changes between two versions:

```python
changes = (
    spark.read
    .format("delta")
    .option("readChangeFeed", "true")
    .option("startingVersion", 10)
    .option("endingVersion", 20)
    .table("bronze.crm_customers")
)
# Each row has _change_type: 'insert', 'update_preimage', 'update_postimage', 'delete'
```

CDF is the Medallion alternative to CDC at the source — if the source doesn't support CDC, enable CDF on the Bronze Delta table and let Silver subscribe to the change feed.

## Iceberg catalog options

An Iceberg catalog stores table metadata (where the current snapshot is, which manifest files exist). The data itself is always in object storage; the catalog is just the index.

| Catalog | Notes |
|---|---|
| **Hive Metastore** | Traditional; works with Spark, EMR, Hive |
| **AWS Glue** | Managed Hive-compatible catalog on AWS; integrates with Athena, EMR, Redshift |
| **REST catalog** | Open spec; vendor-neutral; used by Tabular, Polaris, Snowflake Open Catalog |
| **Nessie** | Git-like catalog — branching and tagging for data; used by Dremio |
| **Unity Catalog** | Databricks' catalog; supports both Delta and Iceberg via UniForm |

For multi-engine environments (Spark + Flink + Trino reading the same data): a REST catalog or Nessie is the most portable choice. For AWS-only: Glue. For Databricks: Unity Catalog.
