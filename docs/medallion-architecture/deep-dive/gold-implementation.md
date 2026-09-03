# Medallion Architecture — Gold Layer Implementation and Semantic Model

**Source:** *Building Medallion Architectures* by Piethein Strengholt (O'Reilly, 2025), Ch. 6 conclusion (pp. 361–372), Ch. 7 (pp. 373–420).

---

## Silver Layer Data Products and Engineering Best Practices

### Silver Layer Best Practices (Engineering Culture)

[FACT] Recommended Silver pipeline engineering practices:
- **DevOps mindset**: CI/CD pipelines, code reviews, version control (Git) — notebooks, scripts, and tests all in a repository with folders like `notebooks/`, `src/`, `tests/`
- **ADF + Airflow split**: use ADF for data extraction/movement; use Airflow for complex workflow orchestration and dependency management
- **Logging over print**: use a logging module rather than print statements; unify logging across ADF and notebooks for end-to-end debugging
- **Key Vault for credentials**: never store passwords or secrets in notebook code; use Azure Key Vault
- **Function size discipline**: notebooks under 300 lines; functions under 100 lines; break up large functions
- **Pylint + PEP 8**: enforce consistent coding style via static analysis
- **Views not physical tables** for intermediate results: reduces storage cost
- **Cleanup**: drop temp tables and release Spark compute after each notebook run
- **pytest**: test Silver logic with automated unit/integration tests
- **IDE-first for production**: write production code as `.py` files in VS Code with Git integration, not Jupyter `.ipynb` notebooks

[ANALYSIS] These practices form the engineering discipline backbone of any production Medallion pipeline: git-versioned notebooks and scripts, Key Vault for credentials, and an automated testing framework for validation.

---

### Silver Layer Data as a Product

[FACT] **Data as a Product** (concept originating from Zhamak Dehghani's "Data Mesh" paper): treating data with the same care as a product — defined ownership, active maintenance, backward-compatible design, and metadata-described for discoverability.

[FACT] In Medallion architectures, data products are logical representations in a data catalog, encompassing tables, files, and reports, with metadata describing structure, lineage, and relationships.

[FACT] Two data product categories in Medallion:

| Category | Layer | Characteristics |
|---|---|---|
| **Operational data products** | Silver | Source-aligned; suitable for operational reporting and ML (digital feedback loops); domain-scoped |
| **Analytical data products** | Gold | Cross-domain integrated; suitable for broader analytics and reporting |

[ANALYSIS] Silver data products have a design constraint: **backward compatibility**. If business teams consume Silver tables directly, schema changes (renames, type changes, column removals) break consumers. The solution is a stable virtual sublayer (views) within Silver that acts as the stable API surface, with the underlying tables free to evolve.

[ANALYSIS] Silver data products should **stay within their domain** — cross-domain joins on Silver data belong in Gold. This boundary prevents Silver from becoming an accidental integration layer that inherits all the complexity of cross-source schema differences.

---

## Gold Layer: Star Schema Implementation

### Design Principles

[FACT] The Gold layer = the final stage of Medallion refinement, optimized for decision making and reporting. It integrates data from multiple Silver sources, harmonizes it, and presents it in a business-oriented structure.

[FACT] Star schema design steps for Gold:
1. Identify business entities (subjects) → these become **dimension tables** (Product, Customer, Date, Address)
2. Identify business events with measurements → these become **fact tables** (Sales)
3. Declare **grain** (level of detail) — e.g., one row per sales order detail line
4. Establish foreign key relationships: fact table holds surrogate key references to each dimension

[FACT] Star schema loading order is non-negotiable: **dimensions must be loaded before fact tables**. Fact table rows reference dimension surrogate keys — without the dimension rows, surrogate key lookup fails.

---

### Creating Gold Tables (Schema-First Pattern)

[FACT] Gold tables are created with predefined schemas using the DataFrame API (`StructType` + `StructField`), then written as empty Delta tables:

```python
from pyspark.sql.types import *
spark.sql('CREATE SCHEMA IF NOT EXISTS adventureworks')

schemaAddress = StructType([
    StructField("ID", StringType()),
    StructField("AddressID", IntegerType()),
    StructField("City", StringType()),
    StructField("current_flag", BooleanType()),
    StructField("current_date", DateType()),
    StructField("end_date", DateType())
])
dfAddress = spark.createDataFrame([], schemaAddress)
dfAddress.write.mode("append").saveAsTable("adventureworks.dimension_address")
```

[ANALYSIS] Creating empty tables with predefined schemas first = schema-first (schema-on-write) approach. This gives full control over column types and order before any data is written, prevents type inference surprises, and makes the DDL version-controllable.

---

### V-Order Writing for Gold (Fabric Optimization)

[FACT] Gold layer notebooks should enable **V-order writing** at session start:

```python
spark.conf.set("spark.sql.parquet.vorder.enabled", "true")
spark.conf.set("spark.microsoft.delta.optimizeWrite.enabled", "true")
spark.conf.set("spark.microsoft.delta.optimizeWrite.binSize", "1073741824")  # 1 GB
```

[FACT] V-order produces Parquet files with a column ordering optimized for VertiPaq (Power BI's in-memory engine), yielding 10–50% read performance improvements in Power BI and Fabric analytical workloads. `optimizeWrite` reduces small-file proliferation by targeting ~1 GB file sizes.

[ANALYSIS] V-order is write-time optimization — it does not require re-reading or re-writing existing data. Enabling it on Gold tables (the layer that feeds BI tools) gives the biggest benefit-to-cost ratio.

---

### Surrogate Keys via SHA-2 Hash

[FACT] Fabric Lakehouse does **not support identity columns** at the time of writing (expected in Delta Lake 3.3.0). Surrogate keys in Gold are generated using SHA-2 hashing across all business key columns:

```python
# Generate surrogate key as SHA-256 hash of all source columns
dimension_address = address.withColumn("ID", sha2(concat_ws("||", *address.columns), 256))
```

[FACT] Hash-based surrogate keys are stable across pipeline re-runs — the same input columns always produce the same hash. This makes the Gold layer **idempotent**: running the pipeline multiple times with unchanged data produces unchanged dimension and fact table contents.

[ANALYSIS] Hash surrogates trade physical uniqueness guarantees (which identity columns provide at database level) for portability and idempotency. The trade-off is acceptable in a Medallion Gold layer because: (1) the source business keys are already unique at Silver; (2) hash collisions at SHA-256 are astronomically unlikely. SQL Warehouse teams using identity columns do not need this workaround — SHA-256 is the Lakehouse-specific alternative for environments without identity column support.

---

### Dimension Loading Pattern (DeltaTable MERGE with SCD2)

[FACT] Dimension tables use a three-clause MERGE operation for SCD2:

```python
from delta.tables import *
deltaTable = DeltaTable.forPath(spark, 'Tables/adventureworks/dimension_address')

deltaTable.alias('gold').merge(
    dimension_address.alias('updates'),
    'gold.ID = updates.ID'
).whenMatchedUpdate(set={
    "current_flag": lit("1"),
    "current_date": current_date(),
    "end_date": "to_date('9999-12-31', 'yyyy-MM-dd')"
}).whenNotMatchedInsert(values={
    "ID": "updates.ID",
    "AddressID": "updates.AddressID",
    # ... all columns ...
    "current_flag": lit("1"),
    "current_date": current_date(),
    "end_date": "to_date('9999-12-31', 'yyyy-MM-dd')"
}).whenNotMatchedBySourceUpdate(set={
    "current_flag": lit("0"),
    "end_date": current_date()
}).execute()
```

[FACT] The three MERGE clauses map to three SCD2 outcomes:
- `whenMatchedUpdate`: record exists and hash matches → update timestamps only (no data change)
- `whenNotMatchedInsert`: new record in source → insert with `current_flag=1`, `end_date=9999-12-31`
- `whenNotMatchedBySourceUpdate`: record exists in Gold but not in source → expire it (`current_flag=0`, `end_date=today`)

[ANALYSIS] `whenNotMatchedBySource` handles **deletes from the source** — records that have been physically removed. Without this clause, deleted source records stay in Gold as stale `current_flag=1` rows indefinitely.

---

### Date Dimension: No SCD2 Needed

[FACT] The date dimension does **not** use SCD2 — dates have no mutable attributes (January 1, 2024 will always be in January, 2024, Q1). The MERGE only uses `whenNotMatchedInsert` (insert new dates as they appear in data; never update or expire existing date rows).

[ANALYSIS] This is a common optimization: if a dimension is **reference data** (values never change once established), skip the SCD2 machinery. Applying SCD2 to a date dimension adds overhead with zero analytical benefit.

---

### Fact Table: Surrogate Key Substitution

[FACT] Fact table loading requires **replacing business keys with surrogate keys** from each dimension. Pattern:

```python
# Read current (only) records from each dimension
dimension_address = spark.read.table("adventureworks.dimension_address").where(col("current_flag") == True)
dimension_customer = spark.read.table("adventureworks.dimension_customer").where(col("current_flag") == True)

# Join fact data against dimensions on business keys, select surrogate keys
fact_sales = sales.join(dimension_address, sales.BillToAddressID == dimension_address.AddressID, "left") \
    .join(dimension_customer, sales.CustomerID == dimension_customer.CustomerID, "left") \
    .select(
        col("dimension_address.ID").alias("AddressKey"),
        col("dimension_customer.ID").alias("CustomerKey"),
        col("SalesKey"), col("Revenue"), ...
    )
```

[FACT] The MERGE for the fact table uses a **composite key** combining all foreign surrogate keys + the natural sales key — because a single sales transaction is uniquely identified by the combination of its order ID and all dimension references.

[ANALYSIS] LEFT JOINs to dimensions (not INNER) preserve fact records even when a dimension match is missing (early-arriving facts). Missing dimension references produce NULL surrogate keys, which the placeholder record pattern resolves (see enhancement area below).

---

### Pipeline Topology for Gold

[FACT] Correct Gold pipeline execution topology:
```
Silver historization (last Silver step)
    ↓
Create Gold tables (once, if not yet exist)
    ↓
Dimension notebooks (run in PARALLEL — no inter-dimension dependencies)
├── dimension_address
├── dimension_customer
├── dimension_date
└── dimension_product
    ↓
Fact notebook (fact_sales — AFTER all dimensions complete)
```

[ANALYSIS] Dimension notebooks are independent of each other and can execute concurrently. The fact notebook has a hard dependency on all four dimension notebooks (it reads their current surrogate keys). Any ETL framework's dependency mechanism — whether a process list, Airflow DAG, or ADF pipeline chaining — exists for exactly this reason: enforcing that dimensions are fully loaded before fact surrogate key substitution begins.

---

### Idempotency: SCD2 as the Only Truly Idempotent Load Pattern

[FACT] SCD2 (type 2) is the **only SCD method that guarantees idempotence**. Running the pipeline with unchanged source data: no new dimension rows inserted, no records expired, fact table unchanged. Running with changed data: new version inserted, old version expired — deterministically.

[ANALYSIS] Idempotency is critical for pipeline reliability: a failed run can be safely retried without corrupting historical data. Patterns that append without deduplication (append-only without a `current_flag` check) are not idempotent — retrying after failure doubles records.

---

## Semantic Model and Power BI

### What a Semantic Model Is

[FACT] A **semantic model** in Fabric translates the technical data model (with database column names and surrogate keys) into a business-friendly view with renamed tables/columns, defined relationships, and pre-computed measures. It is the data foundation for Power BI reports, Excel, mobile apps, and Power Platform.

[FACT] Semantic model setup steps:
1. In the Gold Lakehouse → "New semantic model"
2. Select tables to include (dimension_address, dimension_customer, dimension_date, dimension_product, fact_sales)
3. Define relationships: e.g., `fact_sales.CustomerKey → dimension_customer.ID` (one-to-many)
4. Define cardinality for each relationship
5. Rename tables and hide technical columns (`ID`, `CustomerID`, `current_flag`, `end_date`)
6. Create DAX measures: e.g., `TotalRevenue = SUM(sales[Revenue])`

---

### SCD2 in Gold + Direct Lake: The Trade-Off

[FACT] **Direct Lake** mode queries Delta tables directly in Fabric without copying data. Its constraint: it cannot apply `WHERE` clauses or row filters at the semantic model layer — it queries the full table.

[FACT] When Gold tables contain SCD2 history (both `current_flag=1` and `current_flag=0` rows), Direct Lake exposes all rows to Power BI. Four options to resolve:

| Option | Mechanism | Performance | Complexity |
|---|---|---|---|
| **Views filtering `current_flag=1`** | CREATE VIEW in Gold; bind semantic model to view | DirectQuery mode (slower than Direct Lake) | Low |
| **Power BI Import mode** | Copy filtered data into Power BI's in-memory VertiPaq | Fast (in-memory), but no real-time | Data duplication, refresh scheduling |
| **Report-level filters** | Apply `current_flag=1` filter in every PBI report | Slowest — all history downloaded first | Fragile (filter must be added to every report) |
| **Redesign Gold (current-only)** | Rewrite transformation to only keep current records; or add dedicated "current view" sublayer | Direct Lake (fastest) | Additional sublayer complexity |

[ANALYSIS] Best practice trade-off: **Import mode** for stable, frequently queried reports where near-real-time isn't needed. **Direct Lake with current-only redesign** for real-time or large datasets where duplication cost is unacceptable. Warehouse-based Gold (T-SQL) avoids the Direct Lake SCD2 problem entirely — a view with `WHERE IsCurrent = 1` (or equivalent) is straightforward in T-SQL and does not trigger the full-table scan that limits Direct Lake filtering.

---

### Power BI DAX Performance Guidance

[FACT] DAX (Data Analysis Expressions) is the measure language in Power BI. Complex DAX calculations can be computationally expensive.

[FACT] DAX performance guidelines:
- Pre-calculate derived values in the data source (Gold SQL/Spark) rather than in DAX where possible
- Minimize **context transitions** (CALCULATE, SUMX over large tables) in DAX
- Use **correct star schema relationships** with proper cardinality — many-to-many relationships force expansion and are expensive
- Filter at source, not in DAX

---

### Task Flows (Fabric Workspace Documentation)

[FACT] Fabric **task flows** are a visual tool in the Workspace that documents how items (Lakehouses, notebooks, pipelines, semantic models) relate to and depend on each other. They serve as living documentation of the end-to-end pipeline without requiring external diagramming tools.

---

## Gold Layer Enhancement Areas

[FACT] Key Gold layer improvements beyond the baseline tutorial implementation:

| Enhancement | Description |
|---|---|
| **OBT / user-friendly sublayer** | Wide single table for non-SQL users or data scientists; semantic model with renamed columns |
| **DQ step in Gold** | Additional validation after Silver → Gold join to catch integration errors |
| **Placeholder dimension records** | Insert a 0-keyed placeholder record in each dimension for early-arriving fact rows; prevents NULL foreign keys in fact table |
| **Partitioning** | Partition fact tables on high-cardinality filter columns (e.g., `Year`, `Month`) to reduce scan volume |
| **Auditing columns** | Add `created_at`, `updated_at`, `source_system` to all tables for lineage and troubleshooting |
| **Row-level security** | Restrict row access in Power BI / semantic model based on user role/department |
| **Domain sublayers** | Separate sublayers within Gold for different business units or use cases; conformed common integration layer + specialized aggregation layers |
| **Physical serving layer** | Copy Gold → Azure SQL, Cosmos DB, or time-series DB for latency-sensitive API or IoT consumption |

[ANALYSIS] No single database service optimally handles all serving requirements simultaneously. A realistic enterprise architecture combines: a columnar store for fast batch reporting, a relational DB for ad-hoc SQL queries, a time-series store for IoT/streaming, and a serverless SQL endpoint for exploratory analysis. The Gold layer is the source of truth; serving layers are read-optimized replicas.

---

## Source Reference

| Concept | Book location |
|---|---|
| Silver engineering best practices | Ch. 6, pp. 362–366 |
| Silver data as a product, data product categories | Ch. 6, pp. 367–372 |
| Gold layer star schema design | Ch. 7, pp. 374–377 |
| Schema-first Gold table creation (StructType) | Ch. 7, pp. 378–383 |
| V-order writing configuration | Ch. 7, pp. 383–384 |
| SHA-2 hash surrogate keys + identity column limitation | Ch. 7, pp. 384–385 |
| DeltaTable MERGE (dimension SCD2 pattern) | Ch. 7, pp. 385–388 |
| Date dimension (no SCD2) | Ch. 7, pp. 390–392 |
| Product dimension (join + MERGE) | Ch. 7, pp. 392–395 |
| Fact table surrogate key substitution | Ch. 7, pp. 396–401 |
| Pipeline topology + parallel dimensions | Ch. 7, pp. 401–403 |
| Idempotency and SCD2 | Ch. 7, pp. 403 |
| Semantic model setup (relationships, DAX measures) | Ch. 7, pp. 403–409 |
| SCD2 + Direct Lake trade-offs (4 options) | Ch. 7, pp. 404–407 |
| Power BI report creation | Ch. 7, pp. 412–414 |
| Task flows | Ch. 7, pp. 414–416 |
| Gold layer enhancement areas | Ch. 7, pp. 417–420 |
