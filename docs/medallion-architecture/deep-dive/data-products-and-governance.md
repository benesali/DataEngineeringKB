# Medallion Architecture — Data Products, Purview Governance, and Case Studies

**Source:** *Building Medallion Architectures* by Piethein Strengholt (O'Reilly, 2025), Ch. 7 conclusion (pp. 421–459), Ch. 8 (pp. 462–480).

---

## Data Products in a Medallion Architecture

### What Is a Data Product?

[FACT] The "data as a product" concept originates from Zhamak Dehghani's *Data Mesh* work. In a Medallion context: a data product is a **reusable logical entity** — a denormalized Delta table or semantic model — designed to be consumed by business users or applications, including the metadata that describes its structure, lineage, and relationships.

[FACT] There is no universal industry standard for what counts as a data product. Organizations must establish their own guidelines. The book recommends: define a data product as a reusable logical entity referenced by denormalized Delta tables or semantic models that can be easily consumed.

[FACT] Data product suitability examples:

| Representation | Suitability | Notes |
|---|---|---|
| Raw Bronze table | Inadequate | Too tightly coupled to source systems; creates downstream fragility |
| Delta table in Gold (OBT) | Optimal | Efficient and versatile; design for reuse |
| Star schema (Gold) | Exemplary | Minimal joins; well-documented; decoupled from domain-specific workloads |
| Power BI report | Marginal | Context-specific; good for local sharing only |
| AI model | Marginal | Hard to generalize; share underlying data instead |
| Semantic model | Optimal | Broad applicability and reusability |
| Kafka/Event Hub topic | Adequate | Use state-carrying events; ensure compatibility |
| Folder of PDF files | Inadequate | Unstructured; needs processing first; share processed version with metadata |

---

### Data Product Design Guidelines

[FACT] Key areas to cover in organizational data product guidelines:

**Data modeling guidance:**
- Establish standards for granularity, reference data, data types, schema details, keys, and classification
- Require **atomic data** that correctly links data elements to business glossary terms in the catalog
- Provide key management strategies (within and across source systems) to simplify cross-domain integration
- Handle concatenated composite keys (e.g., SKU = Category_Code + "_" + Product_ID) consistently
- Use reserved column names consistently; define strategies for historical data (append/overwrite/merge/structural changes)

**Governance guidance:**
- Catalog guidance: how to register, describe, and tag data products
- Roles and responsibilities: data owners, stewards, development, onboarding, and registration processes
- Data quality: define quality standards and remediation process for source system issues
- Lifecycle management: creation → versioning → consumption request/approval → retirement
- MDM: master data strategies, including master identifier management within data products
- Access control: define request and approval workflow for accessing data products

---

### Separate Data Product Layers

[FACT] Larger organizations create **distinct layers or sublayers within the Gold layer** to separate reusable data products from use-case-specific data:
- Tag data products clearly in the catalog (e.g., `status=product`)
- Store them in a separate Lakehouse entity within Gold
- Use **data virtualization** (views or OneLake shortcuts) to create virtual product layers without physical data copies

[ANALYSIS] This separation prevents common pitfalls: point-to-point interface problems (every team rebuilding the same join), narrow use-case-specific tables that aren't reusable, and inconsistent models across teams.

---

## Microsoft Purview Data Governance

### Domain Terminology: A Source of Confusion

[FACT] The word "domain" carries different meanings in different contexts — not to be conflated:

| Context | Meaning of "domain" |
|---|---|
| **DDD (Domain-Driven Design)** | Problem space the organization addresses; encapsulates knowledge, behavior, laws, activities; segmented into subdomains |
| **Business domain** | Core activities, priorities, applications, and data of a business area |
| **Governance domain** (Purview) | Boundary encapsulating business concepts + data assets under unified ownership; for data product and glossary management |
| **Collection** (Purview) | Grouping of metadata about technical domains (scans); used for organizing technical data assets |
| **Data domain** | Boundaries within which data is collected, processed, harmonized, and distributed |
| **Fabric domain** | Organizational grouping of Workspaces in Fabric; administrative/management boundary |

[ANALYSIS] Governance domains in Purview ≠ Fabric domains ≠ DDD domains ≠ business domains. Using the same term across contexts without qualification causes misalignment between technology teams and governance teams. Always qualify which kind of "domain" is meant.

---

### Microsoft Purview: Governance Domains and Collections

[FACT] **Governance domains** (Purview Unified Catalog) are the top-level organizational boundary for data management:
- Group data products and business concepts (glossary terms, OKRs, critical data elements) under a unified ownership structure
- Span the full data lifecycle from operational systems through to data products
- Each governance domain is owned by a data owner and steward

[FACT] **Collections** organize and manage metadata about **technical** data assets (tables, pipelines, Lakehouses, columns). Collections are created by scanning data sources (Fabric Workspaces, Azure SQL, Databricks, etc.).

[FACT] Key collection constraint: a data source can only be registered **once** in a single Purview account. Shared services (Fabric, Databricks) must be registered at a **higher level in the collection hierarchy** so their metadata is distributable to child collections. Metadata flows downward (parent → child), never sideways (sibling → sibling).

[FACT] **Scoped scanning** in Purview allows choosing specific Fabric Workspaces and designating which collection their metadata should appear in. For example, a Sales Workspace scan writes its metadata into the Sales collection.

[FACT] Governance domains and collections are linked through **data products**: a data product belongs to a governance domain and maps to one or more data assets from a collection. This linking pin connects business governance to technical metadata.

---

### Microsoft Purview Data Products

[FACT] Purview data products = logical entities that group one or more data assets for a particular business purpose. Each data product has:
- Domain ownership (governance domain)
- Data owner (individual or team)
- Health actions (quality checks, freshness)
- Update frequency
- Data assets (Delta tables, files, Power BI reports, ML models)
- **Request access** workflow for authorized access
- Lineage (origin and transformation history)
- Business terms, critical data elements, OKRs

[FACT] A single data asset can be shared across multiple data products. Data products are not limited to Delta tables — they can include any Fabric item (reports, notebooks, ML models).

[ANALYSIS] Clear Purview data product standards are critical. Without guidelines, catalogs fill with technical system tables as data products, inconsistent ownership, and multiple data products for the same underlying data — the opposite of the "single source of truth" goal.

---

### Governance Models for Medallion Architectures

[FACT] Four governance topology patterns for multi-business-domain Medallion architectures:

| Model | Description | Best for |
|---|---|---|
| **Decentralized** | Each business domain manages its own source systems + separate Medallion architecture; distinct governance domains per business unit | Large orgs with independent BU data ownership |
| **Centralized engineering + distributed usage** | Single Medallion architecture; central team handles ingestion/cleaning; distributed teams do domain-specific harmonization in separate Workspaces | Platform teams serving multiple BU consumers |
| **Central management** | Central IT manages all source systems + data integration; collections tied to multiple governance domains | Large regulated enterprises with centralized IT |
| **Hybrid** | Critical data domains managed centrally (compliance/regulatory); less-critical domains managed by BU for agility | Most real-world enterprises |

[ANALYSIS] The right choice depends on team structure, not technology. Key questions: Who owns the data? Is data management moving toward decentralized (data mesh) or centralized (platform team)? Do operational and analytical teams differ? A central platform team that builds and maintains the pipeline framework while distributed business units consume the Gold outputs maps to the "centralized engineering + distributed usage" model.

---

## AP Pension Case Study (Chapter 8)

### Background

[FACT] AP Pension is a Danish pension fund (founded ~1919), ~400,000 customers, ~700 employees. Data Platform Head: Jacob Rønnow Jensen.

[FACT] AP Pension's 4 guiding principles for analytical data strategy:
1. Data must be well-protected, documented, user-friendly, and aligned with AP Pension's information model
2. Move data as little as possible — but not less
3. Use as few technologies as possible — but not fewer
4. Minimize total cost of ownership — but not lower than needed

[ANALYSIS] Principles 2–4 are optimization constraints with a lower bound: "not less / fewer / lower than needed." This prevents over-minimizing to the point of losing capability. A metadata-driven framework exemplifies this: it minimizes bespoke code (principle 3) but must retain custom stored procedures where framework abstractions are insufficient (principle 2) — the lower bound protects the team from painting itself into a corner.

---

### Bronze Layer: PII Handling and Encryption

[FACT] AP Pension's Bronze layer uses **three physical sublayers**:

| Sublayer | Purpose | Access |
|---|---|---|
| **Landing** | Receives data exactly as delivered from source (CSV, JSON, Parquet, Delta, XML); PII may be present; tables truncated after promotion | Isolated — no external access |
| **PreArchive** | Housekeeping layer; converts all formats to Delta; PII still present temporarily | Isolated |
| **Archive** | Long-term historical store; PII encrypted with individual symmetric keys per customer (keys stored in external keystore outside Fabric) | Controlled access via decryption service |

[FACT] PII is handled using **individual symmetric encryption keys per customer** stored in an external keystore (not inside Fabric). When a customer exercises their GDPR "right to be forgotten", the encryption key is deleted — the data remains in Archive but is permanently anonymized (cannot be decrypted without the key). The historical record of "a customer existed" is preserved; their personal data is not.

[ANALYSIS] This pattern cleanly separates the right-to-be-forgotten from data deletion: deleting the key is reversible at the keystore level but irreversible for the data (without key backups). This is architecturally superior to physically deleting historical records (which would break analytical timelines and audit trails).

---

### Bronze Historization: Physical Partitioning over Delta Time Travel

[FACT] AP Pension evaluated unrestricted Delta time travel in the Archive but found it too expensive in storage and too slow for query performance. Instead: **physical partitioning** in the Archive layer provides history without those costs.

[ANALYSIS] This confirms the book's earlier recommendation: physical folder partitioning (YYYY/MM/DD) is more cost-effective for long-term Bronze history than relying solely on Delta time travel (which retains full file versions). Delta time travel is appropriate for short-term rollback (7-day default); physical partitioning is the production pattern for multi-year Bronze archives.

---

### Bronze Ingestion: Versatile Framework

[FACT] AP Pension's ingestion framework handles: REST APIs, SQL databases, flat files, Delta tables, semi-structured files (JSON, XML). The approach varies by source type:
- SQL databases: more automated metadata registration and UPSERT-based ingestion
- Flat files: require manual PII flagging
- All sources: land in Landing → promoted to Archive via PreArchive

[FACT] **Mirroring** (Fabric CDC-based replication) is AP Pension's primary near-real-time ingestion method for Azure SQL, Cosmos DB, and Snowflake. Mirroring writes directly to OneLake, unlike traditional CDC which writes to tables on the source database. This minimizes impact on source database performance and reduces maintenance on the Fabric side.

[FACT] Mirrored data at AP Pension stays in Bronze (not promoted to Silver/Gold shortcuts in production yet). It is used in test environments for AI/ML teams to access near-real-time data that simulates production.

---

### Why Lakehouse over Warehouse (AP Pension's Reasoning)

[FACT] AP Pension chose Spark-based Lakehouses for all Medallion layers (Bronze, Silver, Gold) over T-SQL Warehouses for four reasons:
1. **File format diversity and row-by-row operations**: Spark handles CSV/JSON/Parquet/XML natively; Warehouse is SQL-first
2. **Python + Spark versatility**: framework logic that would require dynamic SQL stored procedures can be implemented as Python classes/wheels packaged and distributed
3. **Hybrid execution**: Spark code runs on both Databricks and Fabric simultaneously (useful for benchmarking/fallback during preview phases)
4. **Shortcut-based data distribution**: mirrored or Lakehouse data can be shared to other Workspaces via OneLake shortcuts without physical copies

[FACT] AP Pension considers Fabric Warehouse relevant for the **serving layer** only, where users join data and create tables using T-SQL — not for the Medallion transformation layers.

[ANALYSIS] This trade-off illustrates a recurring architectural fork: SQL-first teams with stored-procedure-based frameworks gravitate toward Warehouse for Gold; Spark-first teams choose Lakehouse end-to-end. Both are valid — the "right" answer depends on whether team expertise and the existing framework is SQL-first or Spark-first.

---

### Silver Layer: Base + Enriched Sublayers

[FACT] AP Pension's Silver layer has two sublayers:
- **Base layer**: First layer accessible to business users; populated via deduplication, field renaming, and JSON deserialization
- **Enriched layer**: (continues in later pages — enrichment and business rules applied on top of Base)

[FACT] **JSON deserialization at Silver**: Fabric's SQL endpoint auto-deserializes root-level JSON fields into columns. Nested arrays within JSON remain as JSON strings. AP Pension built a **custom iterative deserialization framework** for Silver that:
1. Identifies nested array columns in Bronze Delta tables
2. Creates new Delta tables for each nesting level
3. Links them using keys based on ordinal position within the original JSON document
4. Outputs fully structured columnar data usable from SQL

[ANALYSIS] This nested JSON → columnar Delta pattern applies to any team with Kafka, CosmosDB, or JSON API sources. The Fabric SQL endpoint automatically deserializes root-level JSON fields into columns; only nested arrays require an additional custom deserialization step — the same behavior seen with Kafka message body JSON parsing.

---

## Source Reference

| Concept | Book location |
|---|---|
| Dataflow Gen2, Data Wrangler no-code tools | Ch. 7, pp. 421–423 |
| Data product taxonomy + suitability table | Ch. 7, pp. 427–428 |
| Data product design guidelines | Ch. 7, pp. 428–431 |
| Governance guidance for data products | Ch. 7, pp. 430–431 |
| Purview overview (Unified Catalog) | Ch. 7, pp. 432 |
| Domain terminology disambiguation | Ch. 7, pp. 433–435 |
| Purview governance domains | Ch. 7, pp. 436–437 |
| Purview collections | Ch. 7, pp. 437–439 |
| Unity Catalog + Purview integration | Ch. 7, pp. 440 |
| Scoped scanning in Purview | Ch. 7, pp. 440–442 |
| Purview data products | Ch. 7, pp. 443–446 |
| Medallion governance topology patterns | Ch. 7, pp. 447–454 |
| AP Pension: background + data strategy | Ch. 8, pp. 462–465 |
| AP Pension: platform history | Ch. 8, pp. 465–467 |
| AP Pension: Lakehouse vs Warehouse choice | Ch. 8, pp. 468–471 |
| AP Pension: Bronze sublayers (Landing/PreArchive/Archive) | Ch. 8, pp. 471–473 |
| AP Pension: PII encryption + right to be forgotten | Ch. 8, pp. 471–472 |
| AP Pension: physical partitioning vs Delta time travel | Ch. 8, pp. 473–474 |
| AP Pension: ingestion framework + mirroring | Ch. 8, pp. 474–477 |
| AP Pension: Silver Base + Enriched layers + JSON deserialization | Ch. 8, pp. 477–480 |
