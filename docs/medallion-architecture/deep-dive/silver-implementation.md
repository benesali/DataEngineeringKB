# Medallion Architecture — Silver Layer Implementation

**Source:** *Building Medallion Architectures* by Piethein Strengholt (O'Reilly, 2025), Ch. 6 (pp. 301–360).

---

## Metadata-Driven Validation Pipeline

### Wiring the Metastore into the Pipeline

[FACT] After populating the `SchemaMetadata` table (described in `medallion_bronze_implementation_silver_intro.md`), the Silver pipeline retrieves that metadata dynamically for each table via a `LookupMetadata` activity inside the ForEach loop:

```sql
@concat('SELECT * FROM [dbo].[SchemaMetadata]
WHERE SchemaName=''AdventureWorks''
AND TableName=''', item().table_name, '''')
```

[FACT] An **If Condition** activity guards the validation notebook with `@greater(activity('LookupMetadata').output.count, 0)` — the validation only runs when metadata is actually present. The False branch lets the pipeline continue without metadata, enabling a **flexible (non-blocking)** approach. A strict approach would abort if metadata is absent.

[FACT] The validation notebook receives three parameters from Data Factory: `schemaName`, `tableName`, and `metadata` (the raw JSON string of all metadata rows for that table). It performs:
1. Parse `metadata` as JSON; exit with error if not valid JSON
2. Load the Bronze Delta table: `spark.read.table(f'bronze.{schemaName}.{tableName}')`
3. Check every column in metadata exists in the DataFrame
4. Exit with failure if any columns are missing: `mssparkutils.notebook.exit(...)`

[ANALYSIS] This validation acts as a **Bronze → Silver gatekeeper**: only tables whose actual schema matches their registered metadata can progress. Additional rules (primary key checks, nullability, uniqueness, field lengths) can be added to the same script. Metadata-driven frameworks use exactly this pattern: a mapping repository declares the expected structure; the framework refuses to process tables that don't match.

---

### Metadata Framework Improvement Areas

[FACT] A production-grade metadata-driven framework should extend beyond the basic example with:
- **Self-service metadata APIs** — expose the metastore via REST API so teams can register new tables without pipeline changes
- **Multi-format handling** — ForEach condition branches for Parquet→Delta, CSV→Delta, XML nested struct de-nesting
- **Operation breadth** — append, merge, full overwrite, SQL templates, user-defined functions configurable per table
- **Schema provisioning** — auto-create Silver DDL from metadata at pipeline start
- **Security automation** — policy-driven row/column masking, RBAC/ABAC user group management
- **Notebook chaining** — execute notebooks from other notebooks for dynamic dependency management
- **Auditing and logging** — record counts, processing times, source vs target counts per table
- **Alerting** — Slack/Teams notifications on success and failure

[ANALYSIS] The benefits of metadata-driven over traditional code-based approaches: fewer notebooks (one universal pipeline vs one per source), parameterizable validations without code deploys, support for parallelism/concurrency across ForEach iterations. The cost: more upfront framework design, and complex transformations still require custom logic.

---

## Data Cleansing in Silver

### What Data Cleansing Is (and Is Not)

[FACT] Data cleansing = profiling data to detect errors → correcting or removing them → **remediation** (fixing the underlying source system so the errors don't recur). Cleansing is distinct from schema validation (which only checks structure).

[FACT] Cleansing requires both business knowledge AND understanding of the processes that generated the data. Without business context, "fixing" data can introduce new errors.

---

### PySpark Cleansing Example

[FACT] Common PySpark data cleansing operations on a customer table:

```python
from pyspark.sql.functions import *
from pyspark.sql.types import StringType

customers = spark.read.table("bronze.adventureworks.customer")

# Drop unneeded columns
customers = customers.drop("PasswordHash", "PasswordSalt", "rowguid")

# Derive gender from title via UDF
def determine_gender(title):
    if title == 'Mr.': return 'Male'
    elif title in ('Ms.', 'Mrs.', 'Miss'): return 'Female'
    else: return 'Unknown'
determine_gender_udf = udf(determine_gender, StringType())
customers = customers.withColumn("Gender", determine_gender_udf(trim(customers["Title"])))

# Strip source-system prefix
strip_prefix_udf = udf(lambda v: v.strip("adventure-works\\"), StringType())
customers = customers.withColumn("SalesPerson", strip_prefix_udf(customers["SalesPerson"]))

# Standardize date format
customers = customers.withColumn("ModifiedDate", date_format(customers["ModifiedDate"], "yyyy-MM-dd"))

# Normalize phone numbers
customers = customers.withColumn("Phone", regexp_replace(customers["Phone"], r"1 \(\d{2}\) ", ""))

# Write to Silver
customers.write.mode("Overwrite").saveAsTable("adventureworks.clean_customer")
```

[FACT] Naming convention: cleaned tables in Silver use a `clean_` prefix (e.g., `clean_customer`). This co-exists with the `hist_` prefix used by SCD2 tables.

---

### PySpark vs Pandas for Cleansing

[FACT] **PySpark (native)**: distributed across all Spark nodes; scales to any dataset size; more verbose.

[FACT] **Pandas**: concise, high-level API; popular in data science; when used with Spark, operations run **on the driver node only**, bypassing distributed workers. This creates a memory bottleneck on large datasets.

[FACT] **Pandas API on Spark** (formerly Koalas): bridges the gap — pandas-syntax code that executes as distributed Spark operations. Use this for large-scale data when pandas readability is preferred.

[ANALYSIS] For Silver layer cleansing in production (large tables, frequent runs): use native PySpark or Pandas API on Spark. Pandas is acceptable for prototyping or small datasets (< 1GB resident in memory).

---

### Quarantine Tables and Pipeline Halt Logic

[FACT] Bad records identified during cleansing should be written to a quarantine table rather than silently dropped:

```python
customers.filter(age > 150).write.format("delta").mode("append") \
  .saveAsTable("adventureworks.customer_quarantine")
```

[FACT] To halt the pipeline on critical quality failures (intrusive approach):

```python
duplicateIds = customers.toPandas().duplicated(subset=['CustomerID']).sum()
title_not_null = customers.toPandas()['Title'].isna().sum()
if duplicateIds > 0 or title_not_null > 0:
    mssparkutils.notebook.exit(f"DQ error: duplicates={duplicateIds}, null_title={title_not_null}")
```

[ANALYSIS] Quarantine tables are the audit trail for data issues — the data engineering team can review them and communicate defects back to source system owners. The quarantine record also proves which bad data was rejected and when.

---

### Real-Time Cleansing with Structured Streaming

[FACT] Data cleansing applies to streaming pipelines too. Example: drop null rows from an Event Hub stream and write clean records directly to Silver:

```python
df = spark.readStream.format("eventhubs").options(**ehConf).load()
output = df.na.drop()
output.writeStream.format("delta").outputMode("append").toTable("adventureworks.events")
```

[ANALYSIS] Real-time cleansing bypasses the Bronze → Silver hop — clean data lands in Silver directly. The trade-off is that Bronze no longer has an immutable raw record. This is acceptable when the source is ephemeral (event queue) and the cleansing rule is simple and stable.

---

### Organizing Cleansing Notebooks

[FACT] Two organizational strategies:
1. **One notebook per data source** — all cleansing for one source in one place; easier monitoring and change tracking
2. **Notebooks organized by activity** — separate notebooks for each cleansing stage; enables parallel execution and team specialization

[ANALYSIS] Data cleansing is **not fully automatable** via metadata. Common checks (cross-reference to reference data, type casting) can be metadata-driven. Complex, dataset-specific corrections (business-specific derivations, compound regex rules) almost always require custom code. A hybrid approach — generic metadata-driven checks + source-specific cleansing notebooks — is the practical standard.

---

## Data Transformation Frameworks and DQ Tools

[FACT] Tool comparison for Silver layer data transformation and quality:

| Tool | Type | Strengths | Limitations |
|---|---|---|---|
| **dbt** | SQL transformation | Dialect-agnostic transpilation; integrated lineage, docs, and tests; open source + SaaS | Jinja templates needed for complex logic; learning curve |
| **Delta Live Tables (DLT)** | Databricks declarative ETL | CDC out-of-order event handling; integrates MLflow + Unity Catalog; batch + streaming | Databricks-only; no free tier; lock-in |
| **Great Expectations** | Python DQ library | Assertion-based validation; generates DQ reports | Focused on DQ only (no transformation); more setup than dbt/DLT |
| **Ataccama / Monte Carlo** | Enterprise DQ platform | AI-driven profiling, remediation workflows, DQ observability dashboards | Commercial cost |
| **Vanilla PySpark** | General-purpose | Maximum flexibility; no vendor lock-in; full Python ecosystem | Custom lineage/testing/streaming must be built manually |
| **Plain SQL** | SQL views/notebooks | Universally understood; portable; simple | Less flexible than Python; no native DQ or lineage |

[ANALYSIS] Choosing between these is an organization-wide standards decision, not a per-pipeline decision. Once chosen, enforce the standard across all data engineering teams — inconsistent tooling across teams creates integration and knowledge-transfer costs.

---

## Denormalization in Silver

[FACT] Denormalization (consolidating multiple tables into fewer, wider tables) is recommended in Silver when:
- Raw data has hundreds of small system tables creating expensive join chains
- Columnar formats (Parquet/Delta) are in use and data is correctly partitioned
- Data scientists or operational reporters need rapid direct access to Silver data

[ANALYSIS] Data **must be validated and cleaned before denormalization** — joining dirty data amplifies errors. Clean first, then join.

[ANALYSIS] When source tables arrive pre-normalized, Silver denormalization is minimal. However, when business logic requires joining lookups or reference data before loading, pre-computing those joins into a temp table avoids expensive repeated JOIN execution within the main transformation query.

---

## Lightweight Enrichments in Silver

[FACT] Silver enrichments add value to existing data without complex business logic. Common types: new derived attributes, geocoding, timestamp additions, unit conversions (imperial → metric), reference data joins, simple aggregations.

[FACT] More advanced enrichment approaches available in Silver:
- **LLM-based enrichment**: Azure OpenAI, Fabric AI Services, or Databricks AI functions can classify, generate, or annotate data (e.g., classify company names by industry using NAICS codes). LLMs are **nondeterministic** — consistency validation is mandatory after LLM enrichment.
- **Feature engineering**: derived attributes for ML models (e.g., calculating `age` from `birth_date`, binning into `u35`/`o35` brackets for churn prediction)
- **MDM integration**: standardize `CountryRegion`/`StateProvince` across sources, add `MasterCustomer` identifier for cross-source entity resolution

[ANALYSIS] Complex business rules (aggregations, derived KPIs, cross-domain logic) belong in Gold, not Silver. Silver enrichments should be simple, universally applicable, and stable across all consumers.

[FACT] Silver can have multiple sublayers for different enrichment purposes: cleansing sublayer, ML-stable feature layer, experimental ML layer, MDM layer. When multiple sublayers exist, metadata and lineage tracking become critical to understand data flow between them.

---

## Data Historization

### Three Historization Methods

[FACT] Three available historization techniques, each suited to different needs:

| Method | Mechanism | Strength | Limitation |
|---|---|---|---|
| **Bronze partitioning** | YYYY/MM/DD folder structure per delivery | Easy access to exactly what source delivered on any date | Isolated snapshots — hard to query changes between dates |
| **Delta time travel** | DeltaLog transaction snapshots; configurable retention (default 7 days) | Restore any table version; precise audit trail | Manual effort to compare between versions; does not auto-identify what changed |
| **SCD2** | Tracks row-level changes with `effectiveDate`, `endDate`, `current` flag | Automatically identifies current vs historical; queryable at any point in time | Setup complexity; merge performance overhead |

[ANALYSIS] These three methods are **complementary**, not mutually exclusive. Bronze partitioning preserves delivery-day snapshots; Delta time travel provides table-level rollback; SCD2 provides row-level change history in a single unified table. Production Medallion architectures typically use all three simultaneously.

---

### SCD2 in Silver: Best Practice

[FACT] When implementing SCD2 in Silver, the output tables use a `hist_` prefix. Each row gets three added columns: `current` (boolean — 1 for active, 0 for expired), `effectiveDate` (when this version became current), `endDate` (9999-12-31 for current rows; the day before the next record's `effectiveDate` for expired rows).

[FACT] SCD2 Silver vs Gold placement debate:
- **Silver SCD2**: recommended when historical data is needed in source system context for ML models or operational reporting; keeps transformation logic out of Gold
- **Gold SCD2**: recommended when historical dimensional data needs to be unified across sources and optimized for query performance

[ANALYSIS] Building SCD2 in the Gold layer via a metadata-driven dimension-loading procedure, using type1/type2 hash columns for efficient change detection, matches the "current in Silver, historical in Gold" pattern cleanly. Silver holds current-only cleaned data; Gold carries the full version history.

---

### SCD2 PySpark Implementation Pattern

[FACT] The `fn_SCD2` function implements SCD2 via a full outer join approach:
1. Read the `clean_` prefixed input table
2. Generate a `hash` key via `sha2(concat_ws('||', *columns))` if no primary key
3. Full outer join the new data against the existing `hist_` table
4. Derive `action` column: `NOACTION` (unchanged), `INSERT` (new key), `DELETE` (key gone from source), `UPDATE` (key exists but values changed)
5. Union the four action partitions: unchanged rows, new rows, deleted rows (mark `current=False`, set `endDate`), old versions of updated rows (mark `current=False`, set `endDate`) + new versions (mark `current=True`)
6. Overwrite the `hist_` table with the combined result

[FACT] PySpark SCD2 is **portable** across table formats — switching from Delta Lake to Iceberg only requires changing `write.format("delta")` to `write.format("iceberg")`. The same logic also works with SQL `MERGE` for Delta-only environments (demonstrated in Chapter 7).

---

### Delta Deletion Vectors for SCD2 Efficiency

[FACT] Delta Deletion Vectors enable **soft deletes** in Delta Lake — deleted row positions are marked in a separate file rather than rewriting data files. Changes are merged into data files only at read time (merge-on-read). This reduces write amplification for high-turnover SCD2 tables.

[FACT] Enable deletion vectors:
```sql
ALTER TABLE tblName SET TBLPROPERTIES ('delta.enableDeletionVectors' = 'true')
```

[ANALYSIS] Deletion vectors are particularly useful for SCD2 tables with frequent updates (high percentage of rows changing each load), where full file rewrites become expensive. For slowly-changing dimensions with few changes per load, the default copy-on-write is often preferable (fewer small files to read).

---

### Silver Pipeline Execution Order

[FACT] The correct execution order for a Silver data pipeline:
1. **Technical validation** (schema + metadata checks → gatekeeper)
2. **Data cleansing** (clean, standardize, normalize)
3. **(Optional) Join and enrich** (denormalize, add reference data, MDM)
4. **Historization** (SCD2 into `hist_` tables)

[ANALYSIS] Order matters: joining dirty data amplifies errors (clean first); historizing unclean data corrupts the historical record. Splitting into multiple parallel pipelines (e.g., cleansing and enrichment in separate pipelines) is acceptable — but the pipelines must coordinate to maintain the correct order.

---

## Optimization Jobs (Delta OPTIMIZE)

[FACT] The `OPTIMIZE` command compacts small Delta files into larger ones and applies Z-ordering for faster query performance. It should be scheduled to run after the main pipeline load — daily or weekly depending on data volume and query frequency.

[ANALYSIS] Delta OPTIMIZE (Spark/Lakehouse) plays the same role as VACUUM/compaction. For warehouse-based Gold layers (T-SQL), table statistics and clustered columnstore indexes fill the same role. The key insight: optimization is a recurring maintenance job, not a one-time setup.

---

## Orchestration with Apache Airflow

[FACT] Apache Airflow defines data engineering workflows as **DAGs (Directed Acyclic Graphs)** — Python scripts that specify tasks and their dependencies. Tasks in a DAG can represent Spark notebooks, SQL queries, API calls, or any Python callable.

[FACT] Airflow integration with Microsoft Fabric uses the `apache-airflow-microsoft-fabric-plugin` package and the `FabricRunItemOperator`, which triggers any Fabric item (notebook, data pipeline) by its `workspace_id` and `item_id`.

[FACT] Fabric offers a **managed Apache Airflow service** built into Data Factory — no installation required. Enable via Fabric settings ("Users can create and use Apache Airflow jobs"), then create an "Apache Airflow job" item in the Workspace.

[FACT] Airflow DAG example for the Silver pipeline:

```python
from airflow import DAG
from apache_airflow_microsoft_fabric_plugin.operators.fabric import FabricRunItemOperator
from airflow.utils.dates import days_ago

with DAG('fabric_dag', description="Medallion architecture ELT", start_date=days_ago(2)) as dag:
    clean_data = FabricRunItemOperator(
        task_id="clean_data",
        workspace_id="<workspace_guid>",
        item_id="<notebook_guid>",
        fabric_conn_id="fabric_conn_id",
        job_type="RunNotebook",
        wait_for_termination=True,
        deferrable=True,
    )
    # Task dependency: >> operator
    clean_data >> historize_data
```

[ANALYSIS] Airflow vs Data Factory trade-off:
- **Airflow**: DAGs in Python → version-controllable (git), complex branching logic, multi-platform (Databricks, Synapse, Fabric, etc.), larger community
- **Data Factory**: visual interface, native Azure integration, simpler for simple linear pipelines, less code
- Azure-first teams: Data Factory for simple linear pipelines; Airflow for complex branching logic or multi-platform orchestration.

---

## Source Reference

| Concept | Book location |
|---|---|
| SchemaMetadata INSERT + LookupMetadata wiring | Ch. 6, pp. 301–305 |
| If Condition + validation notebook (column check) | Ch. 6, pp. 305–311 |
| Metadata framework improvement areas | Ch. 6, pp. 311–314 |
| Data cleansing definition and PySpark example | Ch. 6, pp. 314–321 |
| Pandas vs PySpark for cleansing | Ch. 6, pp. 321–324 |
| Quarantine tables + pipeline halt (mssparkutils.notebook.exit) | Ch. 6, pp. 324–325 |
| Real-time cleansing with Structured Streaming | Ch. 6, pp. 325–326 |
| Notebook organization strategies | Ch. 6, pp. 326–327 |
| dbt, DLT, Great Expectations, Ataccama tool comparison | Ch. 6, pp. 328–331 |
| Denormalization in Silver | Ch. 6, pp. 332–333 |
| Lightweight enrichments + LLM enrichment | Ch. 6, pp. 333–338 |
| MDM in Silver | Ch. 6, pp. 337–338 |
| Three historization methods | Ch. 6, pp. 339–340 |
| SCD2 in Silver vs Gold debate | Ch. 6, pp. 340–342 |
| Delta Deletion Vectors | Ch. 6, pp. 343 |
| fn_SCD2 PySpark implementation | Ch. 6, pp. 345–350 |
| Silver pipeline order + split pipeline strategy | Ch. 6, pp. 351–352 |
| OPTIMIZE command scheduling | Ch. 6, pp. 352–353 |
| Apache Airflow DAGs + FabricRunItemOperator | Ch. 6, pp. 353–360 |
