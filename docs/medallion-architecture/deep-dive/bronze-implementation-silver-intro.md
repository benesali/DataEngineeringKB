# Medallion Architecture — Bronze Implementation Patterns, Schema Management, and Silver Layer Intro

**Source:** *Building Medallion Architectures* by Piethein Strengholt (O'Reilly, 2025), Ch. 5 (pp. 241–292), Ch. 6 intro (pp. 293–300).

---

## Lakehouse Table Implementation Patterns

### Parquet Files → Managed Delta Tables

[FACT] Raw Parquet files ingested to a Bronze Lakehouse are not yet queryable via SQL. A follow-up Spark step is required to create Delta tables. [ANALYSIS] This two-phase pattern (CopyTable → Notebook) separates concerns: the pipeline handles raw extraction; the notebook handles table creation and metadata enrichment.

[FACT] The `load_bronze` Spark notebook performs these steps in order:
1. Receive parameters from Data Factory (`schemaName`, `tableName`, `filePath`)
2. `CREATE SCHEMA IF NOT EXISTS {schemaName}`
3. `DROP TABLE IF EXISTS {schemaName}.{tableName}` (full reload; requires Parquet fallback to exist)
4. Read Parquet: `spark.read.parquet(f"Files/{schemaName}/{filePath}/{tableName}.parquet")`
5. Add metadata column: `df.withColumn("loading_date", current_date())`
6. Write as Delta: `df.write.mode("Overwrite").saveAsTable(f"{schemaName}.{tableName}")`

[FACT] The `loading_date` column added in step 5 serves multiple purposes: change tracking, auditing, and is an input requirement for SCD2 logic in downstream layers. In production, additional metadata columns are typically added: `source_system`, `lineage_id`, pipeline run ID.

[ANALYSIS] The `DROP TABLE + overwrite` approach suits **full loads** only. For incremental/delta deliveries where data must accumulate, use the `createIfNotExists` pattern (or `mode("append")` with `mergeSchema`) instead of dropping.

---

### External Tables vs Managed Delta Tables

[FACT] External tables point to data stored outside the managed environment (HDFS, ADLS) using the `LOCATION` clause in `CREATE TABLE`. The data is not moved or duplicated.

[FACT] Databricks example of creating an external table over Parquet files:
```sql
CREATE TABLE bronze_{schemaName}.{tableName}
USING PARQUET
LOCATION '/mnt/bronze/{schemaName}/{filePath}/{tableName}.parquet'
```

[FACT] External table trade-offs compared to managed Delta tables:

| Aspect | External (Parquet) | Managed Delta |
|---|---|---|
| Data duplication | None — in-place query | Delta log written alongside data |
| Query performance | Slower (no optimized partitioning by default) | Faster (Z-order, liquid clustering, file compaction) |
| Schema enforcement | Not available | Built-in schema enforcement |
| Time travel | Not available | Built-in (DeltaLog) |
| Setup complexity | Minimal | Requires notebook/job to convert |

[ANALYSIS] External tables are acceptable for simple, exploratory use cases. For a production Medallion Bronze layer they are not recommended — the absence of schema enforcement and time travel creates operational risk. [INFERRED] Fabric's current limitation (no external table creation on schema-enabled Lakehouses at time of writing) pushes Fabric users toward managed Delta tables by default.

---

### MERGE Operations (Upsert) for Incremental Loads

[FACT] Delta Lake supports SQL MERGE with conditions `WHEN MATCHED`, `WHEN NOT MATCHED`, `WHEN MATCHED AND`, `WHEN NOT MATCHED BY TARGET`, `WHEN NOT MATCHED BY SOURCE`. Example:

```sql
MERGE INTO Bronze_Customer AS t
USING Landing_Customer AS s
ON t.CustomerID = s.CustomerID
WHEN MATCHED THEN
    UPDATE SET t.Name = s.Name, t.ContactDetails = s.ContactDetails
WHEN NOT MATCHED THEN
    INSERT (CustomerID, Name, ContactDetails, ...)
    VALUES (s.CustomerID, s.Name, s.ContactDetails, ...)
```

[FACT] The same MERGE logic is available via the `DeltaTable` Python API, which allows parameterizing the primary key:

```python
from delta.tables import *
dTable = DeltaTable.forPath(spark, f"abfss://{workspaceId}@onelake.dfs.fabric.microsoft.com/{lakehouseId}/Tables/{schemaName}/{tableName}")
(dTable.alias("original")
 .merge(df.alias("updates"), f"original.{primaryKey} = updates.{primaryKey}")
 .whenMatchedUpdateAll()
 .whenNotMatchedInsertAll()
 .execute())
```

[ANALYSIS] Complex MERGE operations often require deduplication of source data before the merge is safe to execute — duplicate source records with the same key cause non-deterministic results in MATCHED clauses. Deduplication is typically done in Silver or Gold, not Bronze.

---

### Spark Structured Streaming (Event Hubs Integration)

[FACT] Spark Structured Streaming is the recommended approach for real-time Bronze ingestion from message queues. It supports complex event processing, joining streams with static data, and writing to Delta sinks with checkpointing for fault tolerance.

[FACT] Azure Event Hubs integration example:

```python
ehConf = {}
ehConf['eventhubs.connectionString'] = sc._jvm.org.apache.spark.eventhubs.EventHubsUtils.encrypt(connectionString)
df = spark.readStream.format("eventhubs").options(**ehConf).load()
df.writeStream \
  .option("checkpointLocation", "abfss://<abfss_location>") \
  .outputMode("append") \
  .format("delta") \
  .toTable("bronze.weather")
```

[FACT] The `checkpointLocation` stores streaming query metadata in ADLS. It ensures **fault tolerance**: if the stream fails, it resumes from the last committed checkpoint rather than reprocessing from the beginning.

[ANALYSIS] Structured Streaming can also write directly to the Silver layer in a single pipeline, bypassing the Bronze-to-Silver hop for latency-sensitive workloads. The trade-off is loss of Bronze immutability — if transformation logic changes, there's no clean raw baseline to reprocess from.

---

### Change Data Capture (CDC) via Fabric Mirroring

[FACT] Fabric supports CDC through **mirroring** — databases (Azure SQL, Cosmos DB, etc.) are replicated into OneLake as Delta tables using CDC, without custom pipelines.

[FACT] CDC / mirroring constraints and considerations:
- Requires primary keys on source tables for the mirroring process
- A schema change on the source triggers a complete snapshot restart (all data for that table is reseeded)
- Not all column types are supported — verify type compatibility before enabling
- Creates tight coupling to the real-time source database structure; typically requires an additional archiving or lightweight-transformation layer alongside

[ANALYSIS] Mirroring is Fabric's zero-ETL pattern for operational database replication. It reduces pipeline engineering effort but introduces dependency on source system schema stability — schema changes in the source (e.g. Dataverse, Cosmos DB) trigger a full reseed of the mirrored table.

---

## Schema Management in Bronze

### Five Schema Definition Approaches

[FACT] In a Lakehouse Bronze layer, schemas can be managed using five increasingly structured approaches:

| Approach | Description | Best for |
|---|---|---|
| **No schema (Delta inference)** | Let Delta Lake infer and evolve schema; use `mergeSchema=true` | Delta deliveries, accumulating data, rapid onboarding |
| **DataFrame API** | Define `StructType`/`StructField` in Spark code; applied at read time | Complex nested/unstructured JSON, schema-on-read |
| **SQL DDL statements** | Write `CREATE TABLE` in Spark notebooks with explicit types | Migration from RDBMS, reusing existing DDLs |
| **YAML/JSON configuration** | Store schema definitions in separate files with metadata (PII flags, comments) | Many columns, PII governance, version-controlled schemas |
| **Metadata-driven framework** | Central metadata repository (e.g., Azure SQL) drives schema generation and validation | Enterprise scale, automated schema governance |

---

### No-Schema Approach (Delta mergeSchema)

[FACT] The accumulation/append pattern:

```python
from delta.tables import *
from pyspark.sql.functions import current_date
# Delete today's existing data before re-appending
if DeltaTable.isDeltaTable(spark, f"{schemaName}.{tableName}"):
    spark.sql(f"DELETE FROM {schemaName}.{tableName} WHERE loading_date = current_date()")
df = spark.read.parquet(f"Files/{schemaName}/{filePath}/{tableName}.parquet")
df = df.withColumn("loading_date", current_date())
df.write.format("delta") \
    .mode("append") \
    .partitionBy("loading_date") \
    .option("mergeSchema", "true") \
    .saveAsTable(f"{schemaName}.{tableName}")
```

[FACT] With `mergeSchema=true`, new source columns are automatically added to the target table's end; existing rows get NULL for the new columns. Columns removed from the source are NOT removed from the target — they carry NULL values for new records.

[ANALYSIS] The `mergeSchema` approach enables frictionless onboarding of evolving sources. The risk is **uncontrolled schema drift** — undocumented columns accumulate over time, downstream consumers encounter unexpected NULLs or columns, and data quality erodes silently. Downstream Silver validation catches these if validation rules are maintained.

---

### YAML/JSON Schema Configuration

[FACT] Storing schema definitions in YAML allows adding column-level metadata not expressible in Spark types — for example, PII flags:

```yaml
columns:
  - name: CustomerID
    type: integer
  - name: FirstName
    type: string
    PII: true
```

[FACT] PyYAML (`pip install PyYAML`) loads the file into a Python dict; a helper function maps YAML type strings to PySpark `StructField` types. The resulting `StructType` is applied as a schema during `spark.createDataFrame()` or `spark.read.schema(schema).json(...)`.

[ANALYSIS] YAML schemas can be version-controlled separately from pipeline code — making schema changes reviewable as pull requests, independent of the code that applies them. This is particularly valuable for regulated data (GDPR, PII) where schema changes must be auditable.

---

### Databricks Auto Loader

[FACT] Auto Loader is a **Databricks-only** feature (not available in Microsoft Fabric or other Spark platforms). It uses Spark Structured Streaming under the hood but adds:
- **Exactly-once file tracking**: processed file metadata stored in RocksDB at the checkpoint location; each file is guaranteed to be processed exactly once
- **Automatic schema inference and evolution**: as new files arrive, Auto Loader merges newly detected columns into the existing schema
- **Incremental backfill**: `cloudFiles.backfillInterval` replays data from the previous N days on the next run

[FACT] Auto Loader configuration options:

| Option | Effect |
|---|---|
| `cloudFiles.format` | Source file format: `parquet`, `json`, `csv`, `avro`, `orc`, `text`, `binaryFile` |
| `cloudFiles.schemaLocation` | Path where inferred/evolved schema is stored and maintained across runs |
| `cloudFiles.schemaEvolutionMode` | `addNewColumns` (default), `merge`, `failOnNewColumns`, `rescue` |
| `cloudFiles.includeExistingFiles` | Whether to process files already in the path before Auto Loader started |
| `cloudFiles.backfillInterval` | Periodically reprocess files from the last N days |
| `trigger(availableNow=True)` | Process all currently available files then stop (batch-style trigger) |

[FACT] When `cloudFiles.schemaEvolutionMode = rescue`, new columns are NOT added to the schema. Instead, rows containing unexpected columns have those values stored in a `_rescued_data` column (as JSON string). This allows the pipeline to continue without schema change while the team reviews whether the new data should be incorporated.

[ANALYSIS] The `_rescued_data` safety net is valuable for production pipelines where an unexpected upstream schema change should not immediately break downstream consumers. Operations teams can inspect rescued data, decide whether to accept the schema change, and then replay the data through normal processing.

---

### Schema Evolution Best Practices

[FACT] Recommended practices for managing schema evolution in a Medallion architecture:

1. **Plan meticulously** — record schema versions; track every change with date and reason
2. **Test in a safe environment** — apply schema changes to dev/test first; verify downstream consumers before promoting to production
3. **Maintain backward compatibility** — additive changes only (new columns); avoid renaming or removing columns that existing consumers depend on
4. **Have a rollback strategy** — Delta time travel allows restoring table to a previous schema version via `RESTORE TO VERSION`
5. **Use schema migration tools** — Atlas, Liquibase for automated, version-controlled schema migrations
6. **Use metadata-driven approaches or YAML** — avoid hand-editing DDL in production notebooks

[ANALYSIS] These principles are the same standards that make ETL change scripts reliable in any framework: DDL changes are captured in versioned scripts checked into git, idempotency guards protect against double-application, and column additions are `NULL`able to maintain backward compatibility.

---

## Silver Layer: Metadata-Driven Approach (Chapter 6 Intro)

### Why Metadata-Driven for Silver

[FACT] The Silver layer is more automatable than Bronze because Bronze data has already been standardized into Delta format — eliminating the source-system diversity that makes Bronze complex. With predictable input format, Silver transformations (type casts, renames, DQ checks) can be parameterized and driven from a metadata repository.

[ANALYSIS] This is the principle behind metadata-driven Silver frameworks: transformation rules are defined declaratively in a mapping repository; a registration procedure ingests them; framework procedures execute them without custom code per table.

---

### Metastore: A Central Schema Metadata Repository

[FACT] A metastore for a Medallion Silver layer stores schema-level information used to drive automated validation and transformation. Minimum fields for a schema metadata table:

```sql
CREATE TABLE SchemaMetadata (
    Id INT IDENTITY(1,1) PRIMARY KEY,
    SchemaName NVARCHAR(128),
    TableName NVARCHAR(128),
    ColumnName NVARCHAR(128),
    DataType NVARCHAR(128),
    CharacterMaximumLength INT,
    NumericPrecision INT,
    NumericScale INT,
    IsNullable NVARCHAR(3),
    DateTimePrecision INT,
    IsPrimaryKey BIT DEFAULT 0
)
```

[FACT] Schema metadata is bootstrapped by reading `INFORMATION_SCHEMA.COLUMNS` from the source database:
```sql
SELECT
    'AdventureWorks' as SchemaName,
    TABLE_NAME, COLUMN_NAME, DATA_TYPE,
    CHARACTER_MAXIMUM_LENGTH, NUMERIC_PRECISION, NUMERIC_SCALE,
    IS_NULLABLE, DATETIME_PRECISION,
    COLUMNPROPERTY(OBJECT_ID(TABLE_NAME), COLUMN_NAME, 'IsIdentity') as IsPrimaryKey
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'Address'
```

[ANALYSIS] In production, the `SchemaMetadata` table would be extended with: target schema metadata (where to write in Silver), security labels (row/column-level filters), business-level column mappings, owner information, and refresh frequency — encoding the same information that a mapping repository uses to drive metadata-driven transformations.

---

### Metadata-Driven Validation (Silver Gatekeeper Pattern)

[FACT] The metastore-driven validation check runs **before** promoting data from Bronze to Silver:
- Schema checks: column exists, data type matches, nullable/not-nullable as expected
- Completeness checks: row count in range, no fully empty columns
- Key checks: primary key column uniqueness, no NULLs in key columns

[ANALYSIS] Validation failures caught at the Bronze→Silver gate prevent bad data from propagating further. If the validation is intrusive (pipeline halts on failure), the Silver layer is only ever populated with technically valid data. If non-intrusive, bad records are quarantined in a sibling error table while valid records continue.

---

## Source Reference

| Concept | Book location |
|---|---|
| CopyTable Source/Destination configuration | Ch. 5, pp. 241–244 |
| Bronze Lakehouse table creation (Parquet → Delta) | Ch. 5, pp. 250–257 |
| External tables (Databricks) | Ch. 5, pp. 258–260 |
| MERGE operations (SQL + DeltaTable API) | Ch. 5, pp. 261–263 |
| Spark Structured Streaming (Event Hubs) | Ch. 5, pp. 263–267 |
| CDC / Fabric Mirroring | Ch. 5, pp. 267–269 |
| Create tables without defining schemas (mergeSchema) | Ch. 5, pp. 270–272 |
| DataFrame API schema definition | Ch. 5, pp. 272–274 |
| SQL DDL in Spark notebooks | Ch. 5, pp. 274–275 |
| YAML/JSON schema configuration + PyYAML | Ch. 5, pp. 275–279 |
| Metadata-driven approach (teaser) | Ch. 5, pp. 280 |
| Databricks Auto Loader | Ch. 5, pp. 281–285 |
| Third-party schema tools | Ch. 5, pp. 285–286 |
| Schema evolution best practices | Ch. 5, pp. 286–288 |
| Chapter 5 conclusion and Bronze guidance | Ch. 5, pp. 289–292 |
| Silver layer overview (Ch. 6 intro) | Ch. 6, pp. 293–296 |
| Metastore concept and SchemaMetadata table | Ch. 6, pp. 296–300 |
