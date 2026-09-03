# ETL Design Patterns

> *Primary source: Paulraj Ponniah, "Data Warehousing Fundamentals for IT Professionals" (2010), Ch. 12*

ETL — Extract, Transform, Load — is the backbone of any data warehouse. It moves data from operational source systems into the DWH, reshaping it along the way. This page covers the design patterns and key decisions for each phase.

---

## Extraction

Extraction pulls data from source systems into a staging area. The goal is to get the data out with minimal impact on the source.

### Full vs incremental extraction

**Full extraction:** pull the entire source dataset on every run. Simple, but does not scale. Typically used for small tables or when no change-tracking mechanism exists.

**Incremental extraction:** pull only records that are new or changed since the last run. Requires one of:
- A reliable `last_modified` or `created_at` column in the source
- Source-system CDC (Change Data Capture)
- Transaction log reading

See [Incremental Loads](../scalability/incremental-loads.md) for the full source interface taxonomy and change-capture techniques.

### Push vs pull

**Pull (ETL-initiated):** the ETL tool queries the source at scheduled intervals. Standard approach; no source-side changes needed.

**Push (source-initiated):** the source system sends data to the ETL layer when changes occur. Lower latency; requires source-side messaging or trigger support. Common in event-driven architectures (Kafka producers).

### Online vs staging extraction

**Online extraction:** ETL reads directly from the live operational database. Risk: long-running queries degrade source system performance during business hours.

**Staging extraction:** the source exports a file or snapshot to a staging area (SFTP, blob storage, database dump) at off-peak hours; ETL reads from staging. Decouples ETL from source availability.

Staging extraction is the standard for DWH loads — it avoids contention on the source and gives a consistent point-in-time snapshot.

---

## Transformation

Transformation reshapes and cleanses data to match the DWH target model.

### Types of transformations

| Type | What it does | Example |
|---|---|---|
| **Format conversion** | Change data type or encoding | `VARCHAR` date → `DATE`; ANSI to UTF-8 |
| **Data cleansing** | Fix quality issues | Trim whitespace; replace dummy values; standardize case |
| **Deduplication** | Remove duplicate records | Match on name+address; keep the most recent or most complete record |
| **Aggregation** | Summarize detail records | Sum daily transactions to weekly totals |
| **Surrogate key assignment** | Look up or generate DWH surrogate key | `CustomerBK = 'C-12345'` → `CustomerKey = 88712` |
| **Integration** | Merge records from multiple sources | Combine CRM customer + ERP customer into one DimCustomer row |
| **Derivation** | Compute new columns from existing data | `FullName = FirstName + ' ' + LastName`; classify customer tier from revenue |
| **Splitting** | Break one column into multiple | Parse "City, State, ZIP" into three columns |
| **Lookup / reference** | Enrich with data from reference tables | Map ISO country code to country name |
| **Pivoting** | Rotate rows to columns or vice versa | Key-value pairs → fixed column layout |

### Structural transformations

Source data often has a different structural shape than the DWH target:

- **Denormalization:** collapse normalized source tables into a flat dimension row (resolve FK chain into descriptive columns)
- **Normalization:** break a flat source record into multiple DWH dimension entities
- **Hierarchy flattening:** recursively traverse a parent-child source structure and emit one flat row per entity with all ancestor columns

---

## Loading

Loading writes the transformed data into the DWH target tables.

### Refresh vs update (append/upsert)

| Mode | How it works | When to use |
|---|---|---|
| **Refresh (truncate + reload)** | Delete all existing data; load the full dataset | Small tables; tables with no history; when incremental is not possible |
| **Append** | Add only new rows; never modify existing | Immutable event log tables; audit tables |
| **Upsert (MERGE)** | Insert new rows; update changed rows; optionally delete removed rows | Most dimension and fact tables with incremental loads |
| **SCD2 insert** | Close the current row (set `ValidTo`); insert a new row | Slowly changing dimensions with history tracking |

### Refresh vs update threshold

For a given table, choose update (incremental) over refresh (full reload) when:
- The table is large enough that full reload is slow or expensive
- A reliable change-detection mechanism exists in the source
- The percentage of changed rows per load is small relative to the total (e.g. <10% change rate)

Stick with full refresh when:
- The table is small (the simplicity benefit outweighs the efficiency gain)
- Source data can be retroactively corrected (backdated changes that watermarks would miss)
- No reliable change-tracking exists

### Bulk load vs row-by-row

**Bulk load:** write all rows in a single batch using the database's bulk insert mechanism (`COPY INTO`, `BULK INSERT`, `bcp`). Orders of magnitude faster than row-by-row. Standard for DWH loads.

**Row-by-row:** use only when business logic requires per-row decision-making that cannot be batched (rare in DWH ETL; more common in real-time streaming).

### Loading order matters

Load in FK dependency order to satisfy referential integrity:
1. Reference/lookup tables (countries, currencies, business calendars)
2. Dimension tables (load surrogate keys before fact tables need them)
3. Fact tables (all FKs must already exist in their dimensions)

---

## ETL tool categories

| Category | Characteristics | Examples |
|---|---|---|
| **GUI-based ETL tools** | Visual flow designer; built-in connectors for common sources; metadata repository; job scheduling; error handling and recovery | Informatica PowerCenter, IBM DataStage, Talend |
| **ELT / cloud-native tools** | Push transformation into the target warehouse using SQL; no separate ETL server | dbt, Azure Data Factory Data Flow, Synapse Pipelines |
| **Code-based frameworks** | Write transformations in Python/Scala/SQL; full flexibility; requires engineering skill | Apache Spark, PySpark, dbt Core |
| **Streaming ETL** | Continuous micro-batch or event-driven processing; sub-second latency | Apache Kafka + Kafka Connect, Spark Structured Streaming, Azure Event Hub |

### Key ETL tool capabilities to evaluate

- **Metadata repository:** does the tool track what came from where? (lineage)
- **Parallel processing:** can it scale out by splitting source data across threads or nodes?
- **Restart/recovery:** after a failure mid-load, can it resume from the point of failure rather than re-loading from scratch?
- **Error routing:** can failed rows be quarantined to a reject file/table rather than failing the whole job?
- **Change data capture support:** does it natively read CDC streams (Debezium, Oracle GoldenGate)?
- **Scheduling and dependency management:** can it express load dependencies (load DimCustomer before FactSales)?

---

## ETL architecture patterns

### Staging area

A transient landing zone between source and DWH. Source data is loaded here first (raw, unmodified), then transformed into the target structure. Benefits:
- Source systems are queried once and released
- Transformations can be re-run against the staged copy without re-extracting
- Failed loads don't corrupt the DWH target

### Lambda architecture (batch + streaming)

Run two parallel pipelines:
- **Batch layer:** nightly full/incremental ETL into DWH (accurate, complete, higher latency)
- **Speed layer:** real-time streaming into a fast store (low latency, may be approximate)
- BI tools merge both layers: the batch layer gives historical accuracy; the speed layer gives today's updates

Complexity of maintaining two pipelines has led most modern implementations to adopt the **Kappa architecture** instead — streaming-only, using the stream as both the real-time and historical source.

### ELT pattern (modern DWH)

With powerful cloud warehouses (Fabric, BigQuery, Snowflake), the transformation step happens *inside* the warehouse rather than in a separate ETL server:

```
Source → Extract → Raw landing (Bronze) → Transform in SQL/dbt → DWH (Silver/Gold)
```

Advantages: no separate ETL server; SQL is the transformation language; lineage is traceable through table dependencies; iterative development with dbt run.

---

## Related pages

- [Incremental Loads](../scalability/incremental-loads.md) — extraction strategies and history capture techniques
- [Data Quality](../best-practices/data-quality.md) — transformation-time quality checks
- [Keys](../data-modeling/keys.md) — surrogate key assignment during transformation
- [Slowly Changing Dimensions](../data-modeling/scd.md) — SCD2 loading pattern
