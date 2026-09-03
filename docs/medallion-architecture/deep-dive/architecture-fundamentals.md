# Medallion Architecture — Fundamentals

**Source:** *Building Medallion Architectures* by Piethein Strengholt (O'Reilly, 2025), Chapters 1 (pp. 26–51) and partial Ch. 3 intro (pp. 30–31). Pages 1–60 (parts 01–03).

---

## Why Separate Operational from Analytical Workloads

[FACT] OLTP (online transaction processing) systems are designed for operational workloads: short, predictable, low-volume queries — read a record, update a record, delete a record. They are normalized (typically 3NF) to minimize redundancy and maintain referential integrity, and must satisfy ACID properties (atomicity, consistency, isolation, durability) to protect live business transactions.

[ANALYSIS] Running complex analytical queries directly on OLTP systems creates two problems: (1) the joins required for analytical work — across many normalized tables — are resource-intensive and can degrade the operational system's performance, putting the business at risk; (2) OLTP systems are designed to hold only current data efficiently, so historical data is often purged, making point-in-time analysis impossible without a separate store.

[FACT] OLAP (online analytical processing) systems separate the analytical load. Because they serve repeated reads with few writes, data is **denormalized** — large, flattened, sparse tables — to minimize joins at query time. Data integrity requirements are less strict (analytical results are offline and delay-tolerant), which makes lower-cost infrastructure viable.

**Practical implication for this team:** our stg/dwh layer split mirrors this same separation — stg holds source-faithful copies (OLTP-shaped), dwh layers hold integrated, denormalized analytical structures. The OLTP → OLAP boundary is why the ETL step exists at all.

---

## Data Layering as a Design Principle

[FACT] Every data architecture — regardless of era or technology — has three fundamental layers: (1) data providers (sources), (2) a distribution platform (middle tier), and (3) data consumers (reporting, ML, apps). An overarching metadata and governance layer manages the whole stack.

[ANALYSIS] Layering assigns contrasting responsibilities to each stage. The staging/ingestion layer separates sources from processing. The integration/transformation layer harmonizes data from multiple sources (unified names, types, structures, relations, history). The presentation layer reshapes the integrated data for specific consumption patterns. The number of intermediate layers is a trade-off, not a fixed rule — some organizations add an extra layer for auditability, others split staging into a cheap file store plus a validated relational DB.

[FACT] This three-layer principle appears in traditional data warehouses (staging → integration → data marts), in data lakes (raw → curated → serving), and is directly encoded in Medallion architecture (Bronze → Silver → Gold).

---

## Normalization vs Denormalization: the Read/Write Trade-off

[FACT] **Normalization** (in the context of 3NF) eliminates data redundancy by storing each attribute once and enforcing referential integrity. It is the right design for write-heavy, mutation-intensive workloads because updates touch one place.

[FACT] **Denormalization** reintroduces redundancy to reduce the number of joins at read time. It is the right design for read-heavy analytical workloads where query speed matters more than storage efficiency.

[ANALYSIS] The persistent data engineering debate between Inmon (normalize the integration layer) and Kimball (denormalize from the start) is essentially a debate about where to pay the cost: at write time (ETL complexity, double-load) vs. at read time (more joins, slower queries). Cloud storage economics have shifted this trade-off — storage is now cheap, so the Kimball argument (avoid double-loading, pay storage, win on query speed) has become dominant.

[INFERRED] In Hadoop/data lake environments, denormalization is practically forced: the HDFS is optimized for large files, and joining many small normalized tables across a distributed file system is extremely inefficient. This is one reason the lake era reinforced dimensional modeling rather than 3NF designs.

---

## Staging Areas: Why You Always Need One

[FACT] The staging area (also called landing area or staging layer) sits between source systems and the integration layer. It receives raw extracts from sources before any transformation is applied.

[FACT] Staging serves two distinct purposes: (1) it decouples the extraction cadence from source systems, relieving operational systems of repeated analytical queries; (2) it holds historical deliveries so that the warehouse can be rebuilt from scratch if corrupted ("reprocessing scenarios").

[ANALYSIS] Whether staging is persistent (all deliveries retained) or ephemeral (cleared after successful processing) is a governance decision, not a technical one. Audit requirements may mandate keeping years of deliveries; cost management may mandate purging after N days. Both are valid — the key is that the choice is made explicitly and documented.

[ANALYSIS] The staging area is the place where technical heterogeneity lives: different source file formats, API shapes, encoding issues, and delivery schedules all arrive here before being reconciled downstream. Keeping this messiness isolated in staging protects the cleaner integration layer.

---

## Inmon vs Kimball: Two Philosophies, One Goal

Both methodologies organize a data warehouse with staging → integration → presentation layers. They differ in *how* the integration layer is modeled.

### Inmon (top-down)

[FACT] Inmon's approach (early 1990s) builds a centralized enterprise data warehouse (EDW) modeled in **third normal form (3NF)** as the integration layer. Data marts (dimensional, star-schema) are then built on top of the EDW for specific user groups.

[ANALYSIS] Strengths: single source of truth with no redundancy in the integration layer; easy to add a new data mart without restructuring the core. Weaknesses: ETL must run twice (source → 3NF EDW, then 3NF EDW → data mart), development is slower, and new data mart requirements always block on the EDW team first.

### Kimball (bottom-up)

[FACT] Kimball's approach (1996) builds **dimensional models (star schemas)** directly as the integration layer. Data marts are either subsets or virtual views built on those dimension and fact tables. The integration layer is already denormalized and optimized for reading.

[FACT] Key Kimball concepts:
- **Conformed dimensions** — dimension tables shared across multiple subject areas/user groups, enabling consistent cross-domain reporting.
- **Star schema** — a fact table surrounded by dimension tables; minimizes joins for analytical queries.
- **Data marts** — can be physical (persisted aggregations) or virtual (views over the base dimensional model).

[ANALYSIS] Kimball wins on speed-to-delivery and query performance; Inmon wins on long-term flexibility when the subject area scope is genuinely enterprise-wide and unpredictable. The two are not mutually exclusive — many mature warehouses use 3NF at the EDW core and dimensional models for presentation, which is essentially the Inmon pattern.

---

## Slowly Changing Dimensions (SCDs)

[FACT] An SCD is a dimension table that tracks how attribute values change over time. Three types are standard:

| Type | Alias | Mechanism | Use case |
|---|---|---|---|
| SCD1 | Overwrite | Update the existing row; old value is lost | History doesn't matter; only current state needed |
| SCD2 | Add new row | Insert a new row for each change; original row retained with a separate PK | Full history required; point-in-time queries needed |
| SCD3 | Add new attribute | Add a column for the "previous" value; only one prior state is tracked | Partial history — typically just current vs. previous |

[ANALYSIS] SCD2 is the most powerful and the most expensive to implement and query. In HDFS/Hadoop environments, implementing SCD2 requires physically recreating the entire dimension table (a new CTAS with all history merged in) because HDFS blocks are immutable — you cannot update individual rows in place. This makes SCD2 in data lakes significantly more compute-intensive than in relational warehouses.

[ANALYSIS] SQL-based DWH frameworks commonly implement SCD2 via a stored procedure whose name reflects the original SCD1 intent but whose behavior is determined by an attribute-type parameter — allowing one codebase to handle multiple SCD strategies without branching.

---

## The Medallion Architecture

[FACT] A Medallion architecture is a data design pattern that organizes data into three layers — Bronze, Silver, Gold — progressively improving data structure and quality as data flows through each layer.

| Layer | Also called | What it holds | Purpose |
|---|---|---|---|
| **Bronze** | Raw | Data from sources in native format | Historical record; reliable initial store; no transformation |
| **Silver** | Curated / Refined | Cleaned, standardized, deduplicated data | Complex analytics; granular but consistent; multiple sources harmonized |
| **Gold** | Serving / Presentation | Aggregated, summarized, enriched data | High-level reporting; business metrics; optimized for query performance |

[ANALYSIS] The Bronze → Silver → Gold naming is intentionally business-friendly. It replaces the warehouse vocabulary (staging → integration → presentation) with terms that communicate quality progression to non-technical stakeholders. The underlying design principle is identical to the traditional three-layer warehouse model.

[ANALYSIS] The layers are not rigid checkboxes. Organizations deviate legitimately — adding an extra layer between Bronze and Silver for schema validation, or splitting Gold into domain-specific sub-marts. The guiding principle is: data consumers should be able to tell when data has been cleaned and when it is ready for consumption. The layer count is secondary to that clarity.

[FACT] Medallion architectures are specifically associated with **lakehouse** platforms (Delta Lake, Iceberg, Hudi) and compute engines like Apache Spark. The open table formats solve a key Hadoop-era limitation: they provide ACID-like semantics (atomic commits, row-level updates/deletes) on top of distributed file stores, which makes SCD2 and incremental processing practical without full table rewrites.

---

## The Data Lake Era: Horizontal Scale and Schema Flexibility

[FACT] Data lakes emerged in the mid-2000s alongside the rise of open source software. Unlike traditional warehouses, they use commodity hardware (horizontal scaling), not specialized appliances (vertical scaling). This separation of compute from storage was the architectural breakthrough.

[FACT] First-generation data lakes were built on **Hadoop**:
- **HDFS (Hadoop Distributed File System)** — distributes data across nodes in 128 MB or 256 MB immutable blocks, replicated 3× for fault tolerance. Optimized for large sequential reads; inefficient for many small files.
- **MapReduce** — the processing engine: Map (parallelize input), Shuffle (sort/redistribute output), Reduce (aggregate). Concepts underpin modern Spark processing even though MapReduce itself is largely obsolete.
- **Apache Hive** — SQL-like interface (HiveQL) over HDFS, translating queries into MapReduce jobs. Enabled SQL users to work with lake data without writing Java.

[FACT] HDFS blocks are **immutable** — you can insert and append, but not update individual records. Mutations are written to a write-ahead log and applied asynchronously. This is why traditional SCD2 processing (update a row in place) requires full table rewrites in Hadoop.

---

## Schema-on-Read vs Schema-on-Write

[FACT] Traditional relational databases enforce **schema-on-write**: data must conform to a predefined schema at the moment of ingestion. Invalid data is rejected.

[FACT] Hadoop/Hive introduced **schema-on-read**: data is stored without a fixed structure; the schema is applied dynamically at query time. A CSV or Parquet file can be mounted as an external Hive table and queried without prior schema registration.

[ANALYSIS] Schema-on-read is not a license to skip data modeling. It enables fast, cheap raw ingestion (Bronze layer), but downstream Silver and Gold layers still require explicit, intentional data models. Without them: data quality is low, cross-source integration is inconsistent, and query performance degrades. The schema-on-read vs schema-on-write choice is about *when* to enforce structure, not *whether* to enforce it.

[ANALYSIS] Schema-on-read at the source (e.g. Synapse Link / Dataverse) does not preclude schema-on-write downstream. A common pattern: the source delivers data schema-flexibly, but the Bronze-to-Silver boundary enforces a registered DDL — the pipeline fails fast if the incoming schema diverges, rather than silently propagating unexpected columns.

---

## Columnar Storage Formats

[FACT] Hive internal (managed) tables use columnar formats like **ORC (Optimized Row Columnar)** and **Parquet**. Modern Medallion architectures use **Parquet** and **Delta Lake** (Parquet + transaction log).

[ANALYSIS] Columnar formats store each column's values contiguously on disk. For analytical queries that aggregate or filter on a small subset of columns, this means only the relevant columns are read from disk — dramatically reducing I/O and memory. Columnar formats also compress better than row-based formats because similar values (e.g., all values in a `CountryCode` column) compress more efficiently than interleaved mixed-type rows.

[ANALYSIS] Delta Lake adds a transaction log (JSON-based) on top of Parquet files. This log records every write operation atomically, enabling: (1) ACID transactions on a file store, (2) time travel (query data as of a past snapshot), (3) schema evolution, (4) row-level updates and deletes without full rewrites. These capabilities are what make the Silver layer's SCD2 and incremental merge patterns practical in modern Medallion architectures.

---

## Hive Metastore (and Why It Still Matters)

[FACT] The **Hive Metastore** is a central catalog that stores metadata about tables, columns, partitions, data locations, and schemas within a Hadoop cluster. It survives as a concept in nearly all modern lakehouse platforms — Microsoft Fabric's data catalog, Databricks Unity Catalog, and AWS Glue Catalog are all conceptual descendants.

[FACT] **External tables** in Hive link to files managed outside Hive (CSV, Parquet on HDFS). Dropping an external table removes only the metadata; the underlying files are untouched. **Internal (managed) tables** are fully Hive-controlled — dropping them removes both metadata and data files.

[ANALYSIS] The external/internal distinction maps directly to modern lakehouse patterns: Bronze layer tables are typically external (raw files own the data; the catalog references them), while Silver and Gold tables are managed (the platform controls lifecycle). This separation protects raw data from accidental deletion during catalog operations.

---

## Source Reference

| Concept | Book location |
|---|---|
| Three-layered architecture | Ch. 1, p. 26–27 |
| What is Medallion architecture | Ch. 1, p. 30–31 |
| OLTP systems and ACID | Ch. 1, p. 34–38 |
| Staging areas | Ch. 1, p. 39–41 |
| Inmon methodology | Ch. 1, p. 42–44 |
| Kimball methodology | Ch. 1, p. 45–46 |
| SCD types 1/2/3 | Ch. 1, p. 48–49 |
| Key takeaways from data warehouses | Ch. 1, p. 50–51 |
| Data lakes and Hadoop | Ch. 1, p. 52–54 |
| HDFS and MapReduce | Ch. 1, p. 54–57 |
| Apache Hive, external/internal tables | Ch. 1, p. 57–59 |
| Schema-on-read vs schema-on-write | Ch. 1, p. 60 |
