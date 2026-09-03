# Medallion Architecture — Microsoft Fabric Setup and Bronze Pipeline

**Source:** *Building Medallion Architectures* by Piethein Strengholt (O'Reilly, 2025), Ch. 3 conclusion (pp. 181–188), Ch. 4 (pp. 191–228), Ch. 5 intro (pp. 229–240).

---

## Gold Layer Serving: Beyond the Lakehouse

[FACT] Gold layer data is often replicated from Delta tables in the lakehouse to specialized database services to meet specific consumer needs:

| Serving target | Use case |
|---|---|
| Azure SQL Database | Business units already running applications/reports on Azure SQL; team-managed "data mart" pattern |
| Azure Data Explorer (Kusto) | Time-series and IoT analytics; KQL query language |
| Graph database | Relationship/graph traversal queries not suited for tabular storage |
| Power BI Import mode | Replicates Delta data into Power BI's in-memory VertiPaq engine; best for consistent performance and fine-grained row/column security |

[FACT] Power BI can connect directly to Delta tables in a lakehouse (DirectQuery/DirectLake). Most organizations choose **Import mode** for predictable performance on large datasets — VertiPaq replicates data from OneLake into its in-memory columnar store.

[ANALYSIS] The serving layer is NOT a replacement for the Gold layer; it is a read-optimized copy for a specific consumer. The Gold layer Delta tables remain the authoritative source; the serving targets are refreshed from them.

---

## Medallion Layer Summary (Table)

[FACT] Official layer characteristics from the book:

| Layer | Purpose | Data model | Transformations |
|---|---|---|---|
| **Landing** | Preliminary zone before Bronze | Raw as-is from source | None |
| **Bronze** | Validated raw data in standardized format | Source system schema | Minimal: filters, metadata columns |
| **Silver** | Cleaned, historized, read-optimized | Mirrors Bronze / subject-oriented / 3NF / Data Vault | SCD2, lightweight transformations, reference data alignment |
| **Gold** | Optimized for value creation | Kimball dimensional / OBT | Harmonized, aggregated, complex business logic |

---

## Microsoft Fabric: Architecture Concepts

### Hierarchy: Tenant → Capacity → Domain → Workspace → Items

[FACT] Microsoft Fabric is a **SaaS analytics platform** built on Apache Spark and Delta Lake. All data is stored in **OneLake** (Azure Data Lake Storage Gen2, Delta format by default).

[FACT] The organizational hierarchy in Fabric:

```
Azure Tenant
  └── Fabric Capacity (compute pool, licensed by SKU: F2, F4, F16, F64...)
        └── Domain (logical grouping by business area, e.g. Sales, Finance)
              └── Workspace (dev / test / prod environments, CI/CD boundary)
                    └── Items: Lakehouse, Warehouse, Notebook, Data Pipeline, KQL database...
```

[FACT] **Domains** are the highest abstraction layer in Fabric. They organize data by business capability (department, business unit). They are for **administrative/management boundaries only** — they do NOT control data access permissions. Subdomains are supported (e.g. "Sales" → "Sales Consumers", "Sales Businesses").

[FACT] **Capacities** are licensed compute pools, measured in Capacity Units. Capacity SKUs: F2, F4, F16, F64, etc. Two billing models: reserved (predictable cost) and pay-as-you-go (on-demand, pause when idle). Multiple Workspaces can share a single Capacity, or each Workspace can have its own.

[FACT] **Workspaces** are collaborative hubs where data practitioners create and share items (Lakehouses, Warehouses, notebooks, reports). Each Workspace connects to one domain and one capacity/region. Workspaces integrate with version control (Azure DevOps, GitHub) and serve as **security boundaries** — Workspace roles: Admin, Member, Contributor, Viewer.

[ANALYSIS] Workspace = the primary unit for CI/CD separation (dev/test/prod each get their own Workspace). Storing all Medallion architecture items (Bronze, Silver, Gold Lakehouses) in a single Workspace is the recommended starting point — splitting across Workspaces complicates CI/CD artifact management.

---

### OneLake

[FACT] OneLake is the shared storage layer for all Fabric items. It uses ADLS Gen2 underneath and stores data in Delta format by default. All compute engines (Spark, SQL Warehouse, Power BI) access the same OneLake data without copying.

[FACT] Two connectivity patterns in OneLake:

| Pattern | Mechanism | Best for |
|---|---|---|
| **Shortcuts** | Virtual pointer to data in another OneLake location or external storage (AWS S3, ADLS) | Open formats (Delta, Iceberg); data already in compatible format; zero-copy access |
| **Mirroring** | CDC-based real-time replication of database snapshots into OneLake as Delta tables | Proprietary database formats (Azure SQL, Cosmos DB); need for near-real-time Delta copy |

[ANALYSIS] Shortcuts = zero-ETL access; the data stays where it lives. Mirroring = near-real-time Bronze-equivalent copy for databases that can't be directly queried via shortcuts.

---

### Lakehouse vs Warehouse

[FACT] Both Lakehouse and Warehouse entities store data in OneLake using Delta format. The difference is the processing engine:

| Aspect | Lakehouse | Warehouse |
|---|---|---|
| Processing engine | Apache Spark (Spark SQL) | Distributed T-SQL engine |
| Language | PySpark, Spark SQL | T-SQL |
| Transactions | Spark-based Delta | Multi-table T-SQL transactions |
| Performance optimization | Spark partitioning / liquid clustering | V-order-optimized tables (VertiPaq-aligned) |
| Best for | Data engineering, ML, open-standard interoperability | SQL-native teams, complex T-SQL queries, BI tools expecting SQL Server semantics |

[ANALYSIS] For a Medallion architecture: use Lakehouse for Bronze and Silver (Spark-based ingestion and transformation); use Lakehouse OR Warehouse for Gold depending on whether the consuming team is more comfortable with Spark SQL or T-SQL. SQL-first teams with stored-procedure-based frameworks typically choose Warehouse for Gold; Spark-first teams choose Lakehouse end-to-end.

[FACT] **Zero-ETL approach**: because all Lakehouse and Warehouse items share OneLake storage, data created in a Bronze Lakehouse is immediately accessible to a Silver Lakehouse or Gold Warehouse without a copy step.

---

### Other Fabric Workload Types

[FACT] Fabric workload types beyond Lakehouse and Warehouse:
- **Data Factory**: orchestration and automation of data movement / transformation (ADF-equivalent; 200+ connectors)
- **Real-Time Intelligence**: pulls data from Azure Event Hubs or IoT Hub; ingests into Lakehouse, Warehouse, or KQL (Kusto Query Language) database for real-time / time-series analytics
- **Data Science**: notebooks with Data Wrangler (auto-generates Python data prep code)
- **Power BI**: reporting and visualization integrated with OneLake data

---

## Fabric Setup for Medallion Architecture

### Recommended Foundation (Oceanic Airlines example)

[FACT] Minimum Fabric setup for a standard Medallion architecture:
1. **Capacity**: F4 SKU for simple batch ETL (one shared dev capacity, separate production capacity for performance isolation)
2. **Domain**: one per business unit / department (e.g., "Sales")
3. **Workspaces**: 3 per project — development, testing, production; all linked to the domain
4. **Lakehouse entities**: 3 per Workspace (Bronze, Silver, Gold) → 9 total Lakehouses for DTP environments

[FACT] When creating a Fabric Lakehouse, enable **"Lakehouse schemas"** — this allows creating named schemas (like SQL Server schemas) to organize tables by source, layer, or domain within one Lakehouse entity. A default `dbo` schema is created and cannot be removed.

[ANALYSIS] Multiple schemas within a single Lakehouse vs separate Lakehouses: separate Lakehouses provide separate SQL endpoints, allowing finer-grained access control (e.g., read-only access to Gold endpoint only). Multiple schemas within one Lakehouse are simpler to manage but share one SQL endpoint. The book recommends **separate Lakehouses per layer** for production setups.

---

### Capacity Design Considerations

[FACT] For mixed workload organizations: consider two capacity types:
- **Reserved capacity**: for everyday batch ETL jobs (predictable, steady cost)
- **Pay-as-you-go capacity**: for ad hoc / light / bursty jobs (pause when idle, lower steady-state cost)

[ANALYSIS] Heavy Silver-to-Gold transformation jobs and interactive Power BI queries compete for compute on a shared capacity. Isolating heavy ETL jobs to a separate reserved capacity from interactive reporting capacity improves both reliability and cost predictability.

---

### Storage Account Strategy (Azure Databricks / non-Fabric Spark)

[FACT] For Spark platforms using ADLS directly (Azure Databricks, Synapse Analytics, HDInsight), three storage strategies:

| Strategy | Structure | When to use |
|---|---|---|
| **Single account, 3 containers** | One ADLS account: Bronze/, Silver/, Gold/ | Small projects, single team, simple governance |
| **Multiple accounts per domain** | Each domain has its own ADLS account | Multi-domain, strict data isolation, different governance policies per domain |
| **Hybrid (local + central)** | Local ADLS per domain for internal processing; central shared ADLS for distributing finalized data products | Multiple domain teams, need to share Gold-layer outputs across domains while keeping Bronze/Silver isolated |

[ANALYSIS] Reasons to use multiple storage accounts: organizational structure (dept ownership), multi-regional compliance (data residency laws), different Azure policies per lake, cost tracking by subscription, sensitive data isolation, environment segregation (dev/test/prod), disaster recovery (geo-distribution).

---

## Bronze Pipeline Pattern (Data Factory)

### Standard Pipeline Structure

[FACT] The canonical Data Factory pipeline for Bronze ingestion uses four activity types in sequence:

```
Lookup → ForEach → CopyTable → Notebook
```

| Activity | Purpose |
|---|---|
| **Lookup** | Queries INFORMATION_SCHEMA to get the list of source tables dynamically |
| **ForEach** | Iterates over the table list from Lookup; executes CopyTable for each table |
| **CopyTable** | Copies each source table from Azure SQL to the Bronze Lakehouse, partitioned by load date |
| **Notebook** | Runs Spark job to create Delta tables from the raw files in Bronze |

[FACT] INFORMATION_SCHEMA query used to retrieve target tables:
```sql
SELECT * FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_TYPE = 'BASE TABLE' AND TABLE_SCHEMA NOT IN ('sys', 'information_schema', ...)
```

[FACT] ForEach is configured with `@activity('LookupTables').output.value` — the input name must **exactly match** the Lookup activity's name. This pattern dynamically copies all tables without hardcoding table names.

[FACT] **Service principal authentication** is the recommended authentication method for Data Factory connections to Azure services — more secure and scalable than user-based credentials.

[ANALYSIS] This Lookup → ForEach → CopyTable pattern is **metadata-driven ingestion**: source table inventory is discovered at runtime, not hardcoded. The same principle applies broadly: store the table list in a control table and have the pipeline iterate over it dynamically, rather than encoding table names in pipeline logic.

---

### Source Reference

| Concept | Book location |
|---|---|
| Gold serving layer options, Power BI VertiPaq | Ch. 3, pp. 181–182 |
| Medallion layer overview table | Ch. 3, pp. 185–186 |
| Microsoft Fabric intro and hierarchy | Ch. 4, pp. 195–200 |
| OneLake, shortcuts, mirroring | Ch. 4, pp. 199–201 |
| Data Engineering (Lakehouse) workload | Ch. 4, pp. 202–205 |
| Data Warehousing (Warehouse) workload | Ch. 4, pp. 206–207 |
| Other Fabric workload types, zero-ETL | Ch. 4, pp. 207–208 |
| Fabric setup steps (capacity, domain, workspace, lakehouse) | Ch. 4, pp. 208–214 |
| Capacity considerations | Ch. 4, pp. 215 |
| Domain considerations | Ch. 4, pp. 216–217 |
| Workspace considerations (CI/CD, security) | Ch. 4, pp. 217–220 |
| Lakehouse entity considerations | Ch. 4, pp. 220–223 |
| Storage account strategies | Ch. 4, pp. 223–226 |
| Bronze pipeline (Lookup → ForEach → CopyTable → Notebook) | Ch. 5, pp. 230–240 |
