# Medallion Architecture — Layer Design Patterns (Bronze, Silver, Gold)

**Source:** *Building Medallion Architectures* by Piethein Strengholt (O'Reilly, 2025), Ch. 3 (pp. 120–180).

---

## Bronze Layer: Load Patterns and Schema Management

### Append vs Merge for Incremental Loads

[FACT] Two write patterns for integrating incoming data into a Bronze Delta table:

| Pattern | When to use | What it does |
|---|---|---|
| **Append mode** | Streaming data, event logs, insert-only sources | Adds new records without touching existing data; does not handle updates or deletes |
| **Merge (upsert)** | Sources with updates (e.g., CRM, ERP) | Inserts new records AND updates existing records based on a key condition in one atomic operation |

```python
# Append mode
df.write.format("delta").mode("append").save(tablePath)

# Merge mode (PySpark)
deltaTable.alias("target").merge(
    source_df.alias("source"),
    "target.key = source.key"
).whenMatched(...).whenNotMatched(...).execute()
```

[FACT] For incremental loading to work, the source table must have unique incremental identifiers or `updated_at` columns that mark new/changed records. If source records older than the last fetched increment can be updated (backdated corrections), standard incremental loading **will miss those changes** — CDC is needed instead.

[FACT] For streaming incremental loads, use `startingVersion` from the Delta/Iceberg transaction log to resume from exactly where the last run left off, without reading from the beginning.

[ANALYSIS] Maintaining a **metadata control table** that tracks the last processed watermark (max `updated_at` or transaction version) is the reliable pattern for stateful incremental loading. It avoids full-table scans and is resilient to cluster restarts.

---

### Data Historization in Bronze

[FACT] Bronze historization = accumulating raw data deliveries in time-partitioned folders (YYYY/MM/DD format). Each delivery is preserved as a snapshot alongside all previous deliveries. This enables reprocessing from any historical point if downstream data is corrupted.

[FACT] Bronze data is **immutable and read-only** — you append or merge to grow it, but you do not update or reprocess within it. Bronze is NOT an SCD2 store; it is an archive of raw deliveries.

[ANALYSIS] The difference between Delta time travel and Bronze historization: Delta time travel provides recovery/auditing of specific table versions (file-level rollback). Bronze folder historization provides business-day-aligned snapshots of exactly what the source delivered on each day. Use folder historization for "what did the source send on 2024-03-15?"; use Delta time travel for "restore the table to its state at version 42."

---

### Schema Evolution and Management

[FACT] Bronze typically starts with **schema-on-read** (in the landing/pre-Bronze zone — store files without enforcing schema). When data is promoted to a queryable Delta table, switch to **schema-on-write** (schema enforced at write time).

[FACT] Delta Lake schema evolution options:

| Feature | Behavior |
|---|---|
| `mergeSchema = true` | Adds new source columns to the target table (existing rows get NULL); keeps old target columns (new rows get NULL); fails if a column exists with incompatible type |
| Schema enforcement (default ON) | Rejects any write where source schema doesn't match target schema; prevents schema drift from silently corrupting tables |
| `ALTER COLUMN` (SQL) | Manual schema change; requires coordination with source teams; check into version control |
| Automated schema evolution | Scripts detect source schema changes and generate/apply ALTER statements |
| Metadata-driven framework | Maintain schema mappings in a repository; auto-generate ALTER statements; CI/CD-integrated |

[ANALYSIS] The recommended approach is a metadata-driven framework: maintain source-to-target schema mappings in a repository. This scales to many source systems and avoids the fragility of manual ALTER scripts.

[FACT] For disruptive schema changes incompatible with mergeSchema, create a **new version of the pipeline** landing in a new target location. Maintain the old and new schemas in parallel, letting consumers choose which schema version they need.

---

### Technical Validation at Bronze

[FACT] Bronze is the **primary checkpoint for technical validation** — format checks, schema checks, completeness checks (row counts, checksums). Catching errors here prevents them from propagating to Silver and Gold.

[FACT] Two philosophies on handling validation failures:

| Approach | Behavior | Use when |
|---|---|---|
| **Intrusive** | Halt the pipeline on failures; store bad records in a separate error folder/table; prevent them from entering Bronze | Downstream processes cannot tolerate any bad data |
| **Non-intrusive** | Continue processing; bad records co-exist in Bronze (flagged); fix in Silver | Tolerance for imperfect data; prefer throughput over gatekeeping |

[FACT] Data validation tools for Bronze:
- **Delta schema validation** — built-in; rejects incompatible writes
- **Delta Live Tables (DLT)** — Databricks; combines transformation, orchestration, DQ monitoring
- **Great Expectations** — open source; assertion-based data quality framework
- **dbt** — SQL-first transformation + tests
- **Ataccama, Monte Carlo Data** — enterprise DQ platforms
- **Azure Data Factory schema drift detection** — ADF-native; flags upstream schema changes without breaking the pipeline

[ANALYSIS] Responsibility for fixing technical validation failures rests with **source system owners**, not the data platform team. The data platform team defines the standards; the source team must meet them. This boundary is critical for governance.

---

### Bronze Governance

[FACT] Bronze data is immutable — it should not be altered, deleted, or corrected once ingested (with narrow exceptions for PII encryption or technical mediation metadata). Access controls must prevent modification.

[FACT] Required Bronze observability: ingestion timestamps, source identifiers, any system interaction logs, anomaly alerts on data size/format/arrival time changes, incident response plan for incorrect deliveries.

---

## Silver Layer: Cleansing and Standardization

### What Silver Is and Is Not

[FACT] Silver layer purpose: clean, standardize, and slightly enhance Bronze data **within each source system's scope**. The data stays source-oriented — it is not yet integrated with data from other sources.

[ANALYSIS] The most important thing Silver does NOT do (by default): merge data from different source systems. Cross-source integration is a Gold layer responsibility. Premature cross-source joins in Silver create implicit coupling — a failure or schema change in one source system propagates to tables that logically belong to a different domain.

[FACT] After Silver cleaning, the table structures are generally **the same as Bronze** — same tables, same granularity, just cleaner. Rejected/incorrect data is not deleted; it is flagged and stored in a **sibling quarantine table** within Silver.

---

### Data Cleaning Activities

[FACT] Standard Silver cleaning operations:

| Activity | What it covers |
|---|---|
| Remove noise / inauthentic data | Drop irrelevant columns/rows; remove data that doesn't reflect the true source |
| Handle missing values | Remove, substitute with default, or impute based on surrounding data |
| Remove duplicates | Unless retention compliance requires them |
| Trim spaces | Leading/trailing whitespace in strings |
| Error corrections | Typos, incorrect capitalization, wrong units, outlier detection |
| Consistency checks | Standardize abbreviations (NL not NETHERLANDS), terminology, units of measure |
| Standardize formats | Dates: YYYY-MM-DD; handle locale variations |
| Correct types | Cast to proper numeric/date/float types |
| Fix ranges | Validate values fall within acceptable bounds |
| Fix uniqueness | Enforce uniqueness where required |
| Fix constraints | Verify referential integrity (child → parent) |
| Mask PII | Conceal clear-text personally identifiable information before exposure |
| Anomaly detection | Sudden spikes (e.g., sales volume) as DQ signals |
| Master data management | Apply MDM rules for consistency of shared critical data |
| Standardize reference data | Addresses, phone numbers, country codes |
| Conform data | Apply a common data model / naming convention |

---

### Silver Data Model Design Choices

#### Column Renaming and Standardization
[FACT] Renaming cryptic source column names to business-readable names is a **best practice in Silver**. Using SQL `COMMENT` on columns and tables improves navigability.

[ANALYSIS] Correct data types in Silver prevent implicit type conversions at query time (performance cost) and reduce storage. Use the smallest type that fits the data.

#### Denormalization
[FACT] Denormalization in Silver (consolidating multiple tables into fewer, wider tables) is optional. Use it when the Silver layer will be heavily read and frequently reloaded. Denormalization reduces join overhead at query time but increases storage volume and maintenance complexity.

#### SCDs in Silver vs Gold
[ANALYSIS] There is no universal rule; the choice depends on use case:

| Scenario | Recommendation |
|---|---|
| General reporting | Keep Silver as current-only; build SCD2 in Gold |
| ML models needing historical context in source form | Build SCD2 in Silver — historical data must stay aligned with source context |
| Operational reporting replacing an ODS | Build SCD2 in Silver — eliminates the need for a separate operational data store |

[ANALYSIS] Building SCD2 in the Gold layer via a dimension-loading procedure, while keeping Silver-equivalent staging tables as current-only, matches the "current in Silver, historical in Gold" recommendation cleanly.

[FACT] SCD2 implementation requires: `start_date`, `end_date`, `is_current` (or `is_valid`) columns, plus business keys and hashes for change detection.

#### Surrogate Keys
[ANALYSIS] Surrogate keys (system-generated meaningless identifiers) do NOT belong in Silver. They should first appear in Gold, where data from multiple sources is joined using natural/business keys to look up or generate surrogate keys. Silver tables identify records by business/natural keys.

[FACT] Exception: if Silver does build SCD2 tables, surrogate keys may appear there as lookup keys for Gold. This is viable but requires careful consistency management.

#### Harmonization: When to Cross-Join Sources
[ANALYSIS] The default recommendation is to **delay cross-source integration to Gold**. If Silver does integrate multiple sources, use traditional data modeling techniques (3NF or Data Vault) rather than ad-hoc merges.

#### 3NF and Data Vault in Silver

[FACT] **Data Vault** structure:
- **Hubs**: unique business keys across sources
- **Links**: relationships/connections between hubs
- **Satellite tables**: descriptive attributes and history about hubs/links

[FACT] Data Vault in Silver contains two sub-areas: **raw vault** (structured normalized representation of raw data, integrated via business keys, tracks history) and **business vault** (harmonization rules, intermediate transformations, PIT/bridge tables for performance).

[ANALYSIS] When to prefer Data Vault or 3NF over denormalized Silver:
- High integration complexity across many disparate source systems
- Rapid schema change / schema drift expected
- Need to manage multiple active timelines (creation time, processing time, load time)

[ANALYSIS] When to prefer wide denormalized Silver tables (the common choice in cloud lakehouses):
- Spark performance: denormalized tables avoid expensive shuffle operations across compute nodes from joining many normalized tables
- Cloud storage is cheap; denormalization trade-off on storage is acceptable
- Lower development complexity

---

### Silver Enrichment and Business Rules

[ANALYSIS] Keep enrichments minimal in Silver. Enrichment for operational reporting can start in Silver. Enrichment for ML feature engineering typically requires a separate sandbox or ML layer beyond the three main layers.

[ANALYSIS] Business rules belong in Gold (where they can be customized per use case). Silver should focus on cleansing and standardization, not business logic. Exception: basic conforming rules (calculating a derived field that is universally applicable) can go in Silver for reuse.

---

### Automation in the Medallion Layers

[FACT] Automation difficulty differs by layer transition:

| Transition | Automation difficulty | Reason |
|---|---|---|
| Source → Bronze | Hardest | Diverse source technologies, proprietary APIs, varied formats |
| Bronze → Silver | Easiest | Predictable, parameterizable transformations: rename, filter, fix types, apply lookups |
| Silver → Gold | Harder | Complex business logic, integration rules not easily captured in metadata |

[FACT] Bronze → Silver is the most suitable for **metadata-driven frameworks**: define schema mappings, DQ rules, natural keys, and transformation rules in a metadata repository; auto-generate transformation code. Tools: custom metadata frameworks, dbt, Delta Live Tables (DLT).

[ANALYSIS] This is the pattern behind metadata-driven ETL frameworks: a mapping repository declaratively defines the ETL logic; a registration procedure ingests it; framework procedures execute the generated transformations without hand-authoring per-table SQL. The Bronze → Silver automation challenge remains: each new source integration still requires bespoke extraction code before the metadata layer takes over.

---

## Gold Layer: Integration, Modeling, and Serving

### What Gold Is

[FACT] Gold is the most complex layer — it integrates data from multiple Silver sources, applies business rules, aggregates/summarizes, and produces outputs directly used for reporting and decision-making. Gold is shaped by business requirements, not source system structure.

[FACT] Gold is often divided into multiple sublayers or stages because the variety of consumer requirements (flat reports, star schemas, OLAP cubes, API feeds, semantic models) cannot be served by a single unified structure.

---

### Star Schema Design

[FACT] Star schema is the dominant Gold layer design:
- **Fact table**: central table, stores quantitative measurements (metrics), compact and fast
- **Dimension table**: stores descriptive context (attributes), surrounds the fact table

[FACT] Star schema design sequence:
1. Capture business requirements from stakeholders
2. **Declare granularity** — the level of detail of the fact table (each row represents one transaction, one day's summary, one event, etc.)
3. Identify dimensions (who, what, when, where — e.g., customer, product, time, location)
4. Identify facts (quantitative measurements at the declared grain)

[ANALYSIS] The star schema is a **public interface** (like an API). Its job is not just performance — it must align with how business users naturally think about and slice the data. A technically optimal model that doesn't match users' mental model will not be used.

---

### Loading Dimension Tables

[FACT] Loading a dimension requires comparing incoming data with the existing table using the business key to determine: new record (insert + assign new surrogate key) or changed record (update existing, insert new version for SCD2).

[FACT] Harmonization before dimension load: transform source values into a common format (decode codes, split multi-part attributes, replace NULLs with required default values, standardize reference data).

[FACT] **Early arriving fact**: a fact record arrives referencing a dimension key that doesn't yet exist in the dimension table. Resolution: insert a **placeholder record** in the dimension for the unknown key; update it when the real dimension data arrives. This maintains referential integrity without blocking fact loading.

[FACT] **Dimension tables must be loaded before fact tables** — facts reference dimension surrogate keys; without the dimension rows, surrogate key lookup fails.

---

### Loading Fact Tables

[FACT] Fact table loading replaces **business keys** (natural identifiers from source) with **surrogate keys** from the corresponding dimension tables. Each row in the fact table holds foreign key references (surrogate keys) to each dimension.

[FACT] Administrative optimization columns: `type1_hash` and `type2_hash` (MD5/SHA hash of SCD1 and SCD2 attribute sets) speed up change detection during incremental ETL by avoiding column-by-column comparison. `creation_date` and `update_date` columns identify newly added vs modified records.

---

### Alternative Gold Designs

#### Curated / Semantic / Platinum Layers
[FACT] Organizations with many data marts sometimes add a **conformed/curation layer** between Silver and the final data marts — housing enterprise-wide conformed dimensions and reference tables reusable across multiple star schemas. These extra layers are sometimes called "Platinum."

[ANALYSIS] This is the pattern that justifies large, centrally managed conformed dimension tables — they live in a curated zone, loaded before any domain-specific fact or data mart is built, and reused across all downstream star schemas.

#### One-Big-Table (OBT) Design
[FACT] OBT stores all relevant data for a use case in a single wide, flat table with nested arrays for related child data. Avoids joins entirely.

| OBT advantage | OBT disadvantage |
|---|---|
| Simpler for non-specialists to manage | Nested data complicates aggregation queries |
| Faster for queries that don't need aggregation across dimensions | Data duplication across rows (inflates memory usage, hurts Spark performance) |
| Flexible schema — easier to add fields | Schema changes (new fields) require recreating the entire table |
| Preferred by data scientists and ML workflows | Poor reusability; hard to build shared analytics on top |
| Good for time-series analysis | |

[ANALYSIS] OBT is a pragmatic choice for specialized ML feature stores or API-serving tables where the consumer is a single pipeline and schema flexibility matters more than analytical slicing. It is not a substitute for a properly modeled star schema for BI/reporting use cases.

#### Serving Layer (beyond the lakehouse)
[FACT] Gold layer data is sometimes replicated from Delta Lake into other database types (RDBMS, NoSQL, search indexes) to meet specific serving requirements (low-latency API responses, full-text search, graph queries). The Gold layer stays authoritative; the serving layer is a read replica.

---

## Source Reference

| Concept | Book location |
|---|---|
| Append vs merge mode | Ch. 3, pp. 121–122 |
| Incremental load requirements and CDC | Ch. 3, pp. 123–124 |
| Bronze historization (folder partitioning) | Ch. 3, pp. 125–126 |
| Schema-on-read → schema-on-write transition | Ch. 3, pp. 127–130 |
| mergeSchema and schema enforcement | Ch. 3, pp. 131–133 |
| Technical validation checks, intrusive vs non-intrusive | Ch. 3, pp. 134–137 |
| Bronze governance and usage | Ch. 3, pp. 138–140 |
| Silver cleaning activities | Ch. 3, pp. 141–145 |
| Silver data model: columns, denormalization, SCDs, surrogate keys | Ch. 3, pp. 146–154 |
| Harmonization: when to cross-join sources | Ch. 3, pp. 154–155 |
| 3NF and Data Vault in Silver | Ch. 3, pp. 155–159 |
| Silver enrichments and business rules | Ch. 3, pp. 160–162 |
| Automation difficulty by layer | Ch. 3, pp. 162–165 |
| Silver in practice | Ch. 3, pp. 165–167 |
| Gold layer overview | Ch. 3, pp. 167–168 |
| Star schema design | Ch. 3, pp. 169–175 |
| Curated/Semantic/Platinum layers | Ch. 3, pp. 175–176 |
| One-big-table design | Ch. 3, pp. 177–180 |
