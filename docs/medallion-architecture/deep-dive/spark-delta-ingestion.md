# Medallion Architecture — Spark, Delta Lake, Ingestion Patterns

**Source:** *Building Medallion Architectures* by Piethein Strengholt (O'Reilly, 2025), Ch. 1 continued (pp. 61–80), Ch. 2 (pp. 82–112), Ch. 3 intro (pp. 113–120).

---

## Apache Spark — What It Fixed and Why It Dominates

[FACT] MapReduce's core inefficiency: every stage reads data from disk and writes results back to disk. A machine learning algorithm needing 10 passes over a dataset requires 10 separate MapReduce jobs, each reading from scratch.

[FACT] Apache Spark was created in 2009 at UC Berkeley's AMPLab specifically to fix this. It stores intermediate data **in memory** rather than on disk between stages — enabling up to 100× speed improvement over MapReduce for iterative workloads. Spark was open-sourced via the Apache Software Foundation in 2013. Databricks was founded the same year by the Spark creators.

[FACT] Spark evolved from Shark (Hive-on-Spark, SQL over Hadoop) to **Spark SQL** — a Spark-native SQL component that retains the Hive Metastore for compatibility. Versions: 1.0 (2014), 2.0 (2016), 3.0 (2020), 4.0 (2025).

[ANALYSIS] Spark's key architectural insight for Medallion architectures: because Spark is elastic (multiple independent clusters sharing the same object storage), you can right-size compute for each layer independently. A large Bronze ingest cluster and a small Gold query cluster can coexist against the same OneLake/ADLS storage. This is impossible with a monolithic Hadoop cluster.

[FACT] Spark has a cold-start cost: data must be loaded from disk into memory before in-memory processing begins. After a cluster restart, all in-memory data is lost and must be reloaded. Sidecar files (metadata, statistics, indexes) reduce cold-start time in modern platforms.

---

## HDFS → Cloud Object Storage

[FACT] Cloud-based object storage (Azure Data Lake Storage, AWS S3, GCS) replaced HDFS in modern deployments. Key differences:

| Aspect | HDFS | Cloud Object Storage |
|---|---|---|
| Scaling | Horizontal (add nodes) | Virtually unlimited, automatic |
| Block model | Fixed-size blocks across DataNodes + NameNode metadata | Files stored as objects with metadata + unique ID |
| Cost | Commodity hardware clusters | Pay-per-use, geographic replication built-in |
| Coupling | Compute tied to storage cluster | Decoupled: compute and storage scale independently |

[FACT] Azure Data Lake Storage (ADLS) maintains HDFS interface compatibility while using cloud object storage underneath. HDFS-compatible tooling (Spark, Hive) works without modification.

[ANALYSIS] The decoupling of compute and storage is the architectural foundation that makes Medallion architecture economically viable. You pay for compute only when pipelines run, and storage costs scale linearly with data volume rather than cluster size.

---

## Open Table Formats: Hudi, Iceberg, Delta Lake

Traditional data lakes stored raw files with no transactional guarantees. Open table formats add a metadata/transaction layer on top of Parquet files to bring warehouse-grade reliability to the lake.

[FACT] Three formats emerged:

| Format | Origin | Year | Key design focus |
|---|---|---|---|
| **Apache Hudi** | Uber | 2017 | Efficient upserts, deletes, incremental processing on Hadoop-compatible filesystems; supports Parquet + ORC |
| **Apache Iceberg** | Netflix | 2018 | Performance for large-scale analytics, schema evolution, supports Parquet + ORC + Avro |
| **Delta Lake** | Databricks | 2019 | ACID transactions, unified batch + streaming, schema enforcement; **Parquet only**, Snappy compression by default |

[FACT] In 2024 Databricks acquired Tabular (Iceberg-focused) and introduced **UniForm** — writes to Delta Lake and asynchronously generates Iceberg/Hudi metadata, enabling cross-format compatibility. **Apache XTable** (from Hudi founders) provides true bidirectional conversion between all three formats.

[ANALYSIS] For this team: Microsoft Fabric uses Delta Lake as its default table format for all Lakehouse and Warehouse tables. Any Spark-based processing in Fabric writes Delta. Iceberg/Hudi knowledge is relevant for interoperability scenarios but not for Fabric-internal work.

---

## Delta Lake Internals — The DeltaLog

[FACT] Delta Lake provides ACID transactions via the **DeltaLog** — a transaction log stored as sequential JSON files in a `_delta_log/` subdirectory alongside the Parquet data files. Every write operation (insert, update, delete, optimize, alter) generates a new commit file (`000000.json`, `000001.json`, …).

[FACT] **Time travel**: because every version is preserved, you can query a table at any prior point in time — `SELECT * FROM my_table VERSION AS OF 5` or `TIMESTAMP AS OF '2024-01-01'`. This is the native Delta equivalent of a restore-from-snapshot.

[FACT] **VACUUM**: removes files not managed by Delta and data files older than a retention threshold (default: 7 days) that are no longer referenced by the current snapshot. Use `VACUUM delta_table_name RETAIN 168 HOURS`.

[FACT] **RESTORE**: rolls back a table to a prior version — `RESTORE TABLE my_table TO VERSION AS OF 10`. This is a write operation that creates a new commit pointing back to the older snapshot.

[ANALYSIS] **DeltaLog is NOT a history tracking mechanism for SCD2.** Reading the log to identify which records changed over the last month requires reading all files and comparing states — computationally expensive. The right approach for business-meaningful history is an explicit SCD2 dimension table, managed by the ETL framework. DeltaLog is for data recovery and auditing, not for tracking attribute-level business changes.

[ANALYSIS] This is why DWH teams model explicit SCD2 dimension tables managed by an ETL framework rather than relying on Delta time travel for attribute-level history queries.

---

## Lakehouse Architecture

[FACT] The lakehouse model (term coined by Databricks) combines: (1) the cost-efficiency and flexibility of cloud object storage, (2) ACID transactions from Delta/Iceberg/Hudi, and (3) the compute power of Spark. It replaces the two-tier "lake feeds warehouse" pattern with a single unified platform.

[FACT] The lakehouse distinguishes itself from pure data lakes by supporting ACID transactions on cloud object storage — enabling reliable updates, deletes, and schema enforcement that were impossible with raw HDFS files.

[FACT] Major lakehouse platforms as of 2025: Databricks, Microsoft Fabric, Azure Synapse Analytics, Azure HDInsight, Cloudera, Dremio (Trino/Apache Arrow), Starburst. AWS, GCP, and Snowflake also use "lakehouse" terminology in their offerings.

[ANALYSIS] The Medallion architecture (Bronze → Silver → Gold) was endorsed by Databricks and Microsoft as the canonical layering pattern for lakehouse platforms. It is not an evolution of warehouse or lake patterns — it is a new design pattern that assumes the lakehouse as its foundation.

---

## Landing Zones and Raw Data Mediation

[FACT] A **landing zone** is a preliminary area where raw data is ingested *before* it enters the Medallion layers. It is conceptually separate from Bronze, though some organizations include it in Bronze's definition.

[ANALYSIS] Common reasons to need a landing zone separate from Bronze:
- External SaaS vendors with strict data handoff requirements
- Source teams managing their own extraction tools and runtime components
- Security/compliance requirements for data staging before decryption or validation

[FACT] Alternative names used in practice for the pre-Bronze area: "tin layer", "pre-Bronze", "pre-refinement", "staging area", "landing area". The naming is organizational, not technical. What matters is that the design choice is explicit.

[FACT] **Raw data mediation**: "raw" does not always mean "exactly as the source emits it." Complex source systems (e.g., Temenos T24 banking: XML with just two columns — `RECID` + `XMLRECORD`) require a mediation/middleware step to convert proprietary formats into a usable structure *before* the data enters Bronze. This mediation is not considered a transformation within the Medallion architecture — it is a prerequisite.

[ANALYSIS] ERP systems with tens of thousands of specialized interconnected tables often provide semantic model extracts (rather than raw table dumps) as the extraction interface. The Bronze layer receives this already-mediated data, not the raw system tables.

---

## Batch vs Real-Time Ingestion

### Batch Processing

[FACT] Batch processing collects data over a period and processes it all at once at scheduled intervals. Still the dominant pattern for most enterprise sources in 2025.

[FACT] Key considerations for Bronze batch ingestion:
- **Integrity validation**: row counts, checksums, or hash totals to detect corruption/truncation during transfer
- **Audit metadata**: file counts, data volume, copy duration, activity run IDs, outcomes — essential for troubleshooting
- **Scheduling**: interval selection must consider source availability, downstream dependencies, and how stale data is acceptable
- **Error handling**: consider an "error layer" or "data orphanage" to catch failed records without blocking the main pipeline

[ANALYSIS] Data collection complexity is pervasive — each source system typically requires its own extraction treatment. Organizations often reuse existing batch interfaces (CSV exports, scheduled SQL extracts) rather than building new ones, because changing source-side behavior is expensive.

### Real-Time / Streaming Ingestion

[FACT] Three patterns for streaming data into a Medallion architecture:

| Pattern | Mechanism | Best for |
|---|---|---|
| **Spark Structured Streaming** | Continuous stream processing engine built into Spark; reads from Kafka, Event Hubs, IoT, log files; writes to Delta, RDBMS, NoSQL | Complex transformations on the stream; multi-sink output |
| **Change Data Feed (CDF)** | Delta feature: `delta.enableChangeDataFeed = true`; captures row-level changes to a Delta table as a stream | Propagating Silver→Gold changes incrementally; streaming only the deltas between MERGE passes |
| **Change Data Capture (CDC)** | Monitors database transaction logs (e.g., SQL Server CDC, Debezium); replicates changes in real time | Near-real-time source system replication into Bronze/landing zone |

[FACT] Real-time streaming with Spark requires an "always-on" compute cluster. To reduce cost when sub-minute latency isn't needed: trigger micro-batches every 5 minutes instead of continuously running.

[FACT] **State-carrying events** (contain a snapshot of application state at a moment in time) vs **notifications** (alerts with no state payload) — Medallion streaming architectures process state-carrying events; notifications are typically consumed by operational systems, not data platforms.

[ANALYSIS] **Parallel layer writes with streaming**: when streaming is used, Bronze can act as an archive zone while Silver and Gold are updated simultaneously from the same stream. This breaks the linear Bronze→Silver→Gold assumption. The architecture choice (linear vs parallel) depends on whether Bronze serves as a replayable source or just a raw archive.

[FACT] Microsoft Fabric's **mirroring** capability uses CDC technology to replicate data from Azure SQL into OneLake as Delta tables in near real time. Whether this goes into Bronze or a pre-Bronze layer depends on organizational intent.

---

## ETL and Orchestration Tools

[FACT] Tools relevant to Medallion architecture pipelines (as of 2025):

| Tool | Category | Key capability |
|---|---|---|
| **Apache Airflow** | Open source orchestration | Programmatic workflow authoring, scheduling, dependency management |
| **Azure Data Factory / Fabric Data Factory** | Microsoft managed | 200+ connectors, pipeline orchestration; ADF = Data Factory in Fabric |
| **Databricks Auto Loader** | Databricks ingestion | Incremental file ingestion from cloud object storage; schema evolution handling |
| **Databricks LakeFlow Connect** | Databricks connectors | Built-in connectors for Salesforce, SQL Server, etc. |
| **Databricks Workflows** | Databricks orchestration | Multi-task ETL/ML pipeline orchestration within Databricks |
| Fivetran, Informatica, Qlik, Stitch, StreamSets | Third-party ELT/ETL | Extensive connectors, often used alongside Spark-native tools |

[ANALYSIS] ETL tool choice influences the architecture: complex tool-specific loading steps may require additional preprocessing layers. Tool standardization is essential for governance, lineage collection, and consistent data quality reporting across the platform.

---

## Delta Table Optimization

Performance in a Medallion architecture depends heavily on how Delta tables are physically organized. Four main optimization techniques:

### Z-Ordering
[FACT] Z-ordering groups related data within the same Parquet files by co-locating values from multiple columns together. Delta's data-skipping statistics benefit from this: when filtering on z-ordered columns, entire files can be skipped. Result: 2–4× faster retrieval on large tables (hundreds of GB to TB+).

```sql
OPTIMIZE events
WHERE date >= current_timestamp() - INTERVAL 1 DAY
ZORDER BY (eventType)
```

[FACT] Z-ordering is beneficial only for large tables. Combining Z-ordering with partitioning is possible, but Z-order clustering only applies *within* a partition and cannot use the same field for both techniques.

### V-Ordering (Microsoft Fabric specific)
[FACT] V-ordering is a write-time Parquet optimization that sorts data using the same algorithm as Power BI's VertiPaq engine. Result: average 10% faster reads, up to 50% in some scenarios. Cost: ~15% slower write times due to the sorting step.

[FACT] V-ordering is 100% Parquet-compatible — any Spark engine reads V-ordered files as normal Parquet. It provides additional benefit specifically under Fabric compute engines (Power BI, Fabric SQL Warehouse).

```sql
CREATE TABLE person (id INT, name STRING, age INT)
USING parquet TBLPROPERTIES ("delta.parquet.vorder.enabled" = "true")
```

[ANALYSIS] Apply V-ordering selectively at layers where SQL endpoint reads dominate (Gold, Silver serving aggregated data). Avoid at Bronze ingestion-heavy layers where write cost outweighs read benefit.

### Table Partitioning
[FACT] Partitioning divides a table into physical file directories by a column's value. Queries filtering on the partition column skip entire directories. Most commonly used on date or country columns.

```sql
CREATE TABLE events USING DELTA PARTITIONED BY (date)
```

[ANALYSIS] Partitioning is most effective for tables of several hundred GB to many TB. Over-partitioning (e.g., by a high-cardinality column) creates the small-files problem it was meant to avoid.

### Liquid Clustering (Delta Lake, mid-2024)
[FACT] Liquid clustering is designed to replace both Z-ordering and partitioning. It dynamically adapts the data layout based on clustering keys, without requiring a fixed up-front layout choice. Supports incremental optimization (recluster without full table rewrite) and changing clustering columns over time.

[ANALYSIS] Liquid clustering solves the key practical problem with Z-order + partitioning: they require knowing query patterns at table-creation time. Liquid clustering is the recommended default for new tables in Delta Lake 3.x+.

### Compaction and Optimized Writes
[FACT] `OPTIMIZE delta_table_name` compacts small Parquet files (accumulated from frequent small writes) into larger files, improving read performance. The small-files problem is endemic to streaming and incremental ingestion patterns.

[FACT] `delta.autoOptimize.optimizeWrite = true` (Delta 3.1+) automatically optimizes file layout during writes. Overhead: ~5–10 seconds for 2–5M row jobs.

---

## The Three Medallion Layers — Design Principles

[FACT] The three layers are **logical**, not physical. Bronze is not "one folder" — it may span multiple physical sublayers (landing → processing → Delta storage). The logical label communicates quality and transformation status, not physical location.

### Bronze Layer
[FACT] Goal: store data from all sources in its original state, unmodified. Serves as an immutable historical record and queryable reservoir for raw data.

[FACT] Key characteristics: high volume, high variety, high veracity (source-faithful). Data is immutable (append-only; original form preserved).

[FACT] Bronze sublayers in a typical pipeline: (1) landing/staging area (checksums, decompression), (2) processing area (format conversion, basic validation, optional encryption of PII), (3) Delta storage (final Bronze table, used as source for Silver).

[FACT] Two Bronze load patterns:
- **Full loads**: entire dataset transferred at fixed intervals; accumulated in folders alongside previous deliveries; most recent delivery exposed as external table or converted to Delta for Silver
- **Incremental/delta loads**: only changed records processed; staged in preliminary layer; lightweight transformations (format change, schema alignment) before Bronze storage

[ANALYSIS] PII encryption should happen at or before Bronze — before data becomes queryable by downstream consumers. Open-source encryption frameworks (e.g., Fernet) or platform-level column encryption can be applied at the Bronze ingestion step.

### Silver Layer (preview — covered in depth in later chapters)
[FACT] Refines and standardizes Bronze data: quality checks, deduplication, standardization of names/types/formats, enrichment. Granular but consistent and query-friendly. Multi-source integration happens here.

### Gold Layer (preview — covered in depth in later chapters)
[FACT] Aggregated, summarized, enriched data optimized for business consumption. High-level reporting, KPIs, strategic analytics. Performance and scalability are the primary design constraints.

---

## Source Reference

| Concept | Book location |
|---|---|
| Hive limitations, MapReduce slowness | Ch. 1, pp. 61 |
| Apache Spark origins and evolution | Ch. 1, pp. 62–63 |
| Cloud object storage replacing HDFS | Ch. 1, pp. 66–67 |
| Open table formats (Hudi, Iceberg, Delta Lake) | Ch. 1, pp. 68–70 |
| DeltaLog, time travel, VACUUM | Ch. 1, pp. 70–71 |
| Lakehouse architecture and vendors | Ch. 1, pp. 72–75 |
| Medallion architecture origins and challenges | Ch. 1, pp. 76–79 |
| Landing zones and raw data mediation | Ch. 2, pp. 84–88 |
| Batch processing patterns | Ch. 2, pp. 88–90 |
| Spark Structured Streaming | Ch. 2, pp. 92–94 |
| Change Data Feed | Ch. 2, pp. 94–95 |
| Change Data Capture | Ch. 2, pp. 95–96 |
| ETL and orchestration tools | Ch. 2, pp. 100–102 |
| Z-ordering, V-ordering | Ch. 2, pp. 102–105 |
| Table partitioning, liquid clustering | Ch. 2, pp. 105–107 |
| Compaction, DeltaLog detail | Ch. 2, pp. 107–110 |
| Bronze layer design and sublayers | Ch. 3, pp. 117–120 |
