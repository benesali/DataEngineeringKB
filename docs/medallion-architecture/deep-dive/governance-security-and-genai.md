# Medallion Architecture — Governance Tools, Data Contracts, Security, and GenAI Integration

**Source:** *Building Medallion Architectures* by Piethein Strengholt (O'Reilly, 2025), Ch. 12 continuation (pp. 601–638), Ch. 13 (pp. 640–660).

---

## Unity Catalog: Operational Governance for Medallion

### What Unity Catalog Is

[FACT] **Unity Catalog** is Databricks's centralized catalog for managing data assets across multiple workspaces. It was created to solve the pain of the original Hive Metastore approach: each workspace required its own metastore, making cross-workspace security model management extremely cumbersome at scale.

[FACT] Unity Catalog is now **open source**. Key capabilities of the open source version:
- Supports Delta Lake, Iceberg, and Hudi table formats via **Delta Universal Format (UniForm)**: asynchronously generates Iceberg metadata so clients can read Delta tables as Iceberg or Hudi tables
- Compatible clients: Microsoft Fabric, Snowflake, DuckDB, Apache Spark, Trino, Dremio
- External access via Credential Vending API and Iceberg REST Catalog / Hive Metastore interface
- Emerging open catalog ecosystem: Unity Catalog OSS, Project Nessie, Apache Polaris, Apache Gravitino

[ANALYSIS] Unity Catalog's open-source status and cross-platform compatibility signal a convergence of catalog standards. Practically: Fabric can natively read Unity Catalog-governed Delta tables as Iceberg (via UniForm), making cross-platform data access possible without physical data copying — a significant simplification for teams running both Fabric and Databricks.

---

### Unity Catalog Namespace and Access Model

[FACT] Unity Catalog introduces a **3-level namespace**: `catalog_name.schema_name.table_name` (vs. the traditional 2-level `schema.table`). Recommended naming pattern for Medallion environments:
- Catalog per team + environment: `dev_sales`, `test_sales`, `prod_sales`
- Schema = Medallion layer: `silver_adventureworks`, `gold_adventureworks`
- Full example: `prod_sales.silver_adventureworks.clean_customer`

[FACT] Access control model: privileges assigned at catalog or schema level are **inherited** by all subordinate objects (tables, views). Privileges can also be assigned per-object. Integration with **Microsoft Entra ID** groups:
```sql
GRANT USE CATALOG ON CATALOG <catalog_name> TO <group_name>;
GRANT USE SCHEMA ON SCHEMA <catalog_name>.<schema_name> TO <group_name>;
GRANT SELECT ON <catalog_name>.<schema_name>.<table_name> TO <group_name>;
```

[FACT] Environment access model (Table 12-2 pattern):

| Principal | Dev catalog | Test catalog | Prod catalog |
|---|---|---|---|
| Developers | Read + write | None | Read-only (for debugging) |
| Test service principal | None | Read + write | None |
| Production service principal | None | None | Read + write |

[ANALYSIS] This 3-environment, 3-principal model is the standard for any dev/test/prod Fabric workspace setup. The key rule: **developers should never have write access to prod**, even for debugging. The test service principal (CI) must be isolated from both dev and prod to prevent test runs from interfering with either environment.

---

### Unity Catalog Key Features

[FACT] Unity Catalog capabilities beyond namespace management:
- **Centralized governance**: tracks data lineage, manages metadata, enforces data quality across all workspaces; metadata search with privilege-restricted access
- **Fine-grained access control**: catalog / schema / table / column / row level
- **Runtime data lineage**: captured across all Databricks languages down to the column level; includes notebooks, jobs, and dashboards
- **Data classification and tagging**: tag-based policy enforcement (e.g., `pii:ccn` tag → automatic masking rule replacing numbers with X)
- **Delta Sharing**: cross-organization/cross-platform sharing via URL + bearer token; works with Power BI, Spark, Pandas

[FACT] Unity Catalog recommendation in a broader infrastructure: pair with **Microsoft Purview** as a "catalog of catalogs." Unity Catalog = operational catalog (data engineering, security, lineage, monitoring within Databricks); Purview = referential catalog (business-facing metadata, cross-platform governance). Together they cover both technical and business governance layers.

---

## Data Contracts

### What a Data Contract Is

[FACT] A **data contract** is a formal agreement between a data provider and a data consumer that outlines terms and conditions for sharing and using data. Specifies: data format, quality, availability, security policies, and responsibilities of each party.

[FACT] Distinction: **data product** = the definition and content of the data; **data contract** = the interface (expectations and responsibilities for sharing the data product). Data contracts = computational governance replacing manual workflows.

[FACT] Industry terminology for data contracts is not standardized. Organizations must define what "data contract" means internally to avoid misunderstandings.

---

### Three Implementation Approaches

**Approach 1: Contracts within a catalog (Purview/Unity Catalog)**

[FACT] Microsoft Purview supports a data product access request workflow: user requests a data product → workflow triggered → data owner/manager approves → access granted. This is a lightweight form of a data contract. Limitation: Purview tracks requests and facilitates contracts but does not yet enforce data delivery or enforce sharing itself (needs integration with HTTP connector or ServiceNow for enforcement).

[FACT] Risk of catalog-based contracts: vendor lock-in. Difficult to extend to technologies not supported by the catalog vendor or to customize beyond predefined scope.

**Approach 2: Contracts within a metastore**

[FACT] Extending the existing metastore (e.g., Azure SQL-based schema metadata) with additional contract tables: sensitive data flags, storage location, access rules, audit logs. Example stack:
- Azure SQL as metastore + web app for visualization
- Logic App for automated approval workflows
- Azure Functions to trigger OneLake Shortcuts REST API provisioning after approval
- Microsoft Entra ID for group-based user permission management

[FACT] This approach provides more flexibility and avoids platform lock-in. Permissions are controlled through group membership (individual users) and service principals (applications).

**Approach 3: YAML files + GitOps**

[FACT] Data contracts defined in YAML files, version-controlled in Git via pull request workflows. Example structure:
```yaml
owner: piethein@oceanicairlines.internal
purpose: To analyze customer behavior and sales trends
use_case:
  description: Tailored for targeted campaigns
workspace_id: 92187CE0-B7EB-4FDF-80CE-EFF76639EED
dataset_location: https://onelake.dfs.fabric.microsoft.com/...
dataset: dim_customer
filter_sql: customer_state = 'New York'
columns:
  - name: first_name
  - name: last_name
  - name: birthdate
```

[FACT] GitOps contract workflow: create YAML → validate against platform standards in CI/CD → governance team reviews in PR → approve → release pipeline applies rights via platform REST APIs.

[FACT] Open source data contract specifications: **Open Data Contract Standard** and **Data Contract Specification** repos (technology-neutral, YAML/JSON). **Data Contract CLI**: open source command-line tool for issuing data contracts with Unity Catalog support.

[FACT] dbt's approach to data contracts: model contracts = predefined schema + column types + data constraints. If model logic or input data doesn't match the contract, **the model doesn't build**. This enforces the contract at pipeline execution time, not just at documentation level.

[ANALYSIS] The YAML/GitOps approach is the most maintainable for teams already using Git-based deployment. Data contracts become part of the same PR workflow as DDL and mapping changes, ensuring governance keeps pace with platform development — exactly the ZEISS lesson: policies must move at the same pace as platform growth.

---

## Data Security and Access Management

### Security Layers Overview

[FACT] Comprehensive data security requires defense in depth — multiple layers:

| Layer | Mechanism |
|---|---|
| **Network** | Block public access; VNET injection; private endpoints; firewall-enabled storage; workspace identities; Key Vault for secrets; safe landing zones (ADF self-hosted integration runtime for external sources) |
| **Authentication** | SSO (single sign-on) — one credential set across applications; MFA (multi-factor authentication) — additional verification factor |
| **Audit** | Diagnostic logs, Storage Analytics logs, NSG flow logs — track all user actions for compliance and records management |
| **Recovery** | Point-in-time recovery — restore data after accidental deletion, corruption, or ransomware attack |
| **Authorization (workspace)** | Built-in workspace roles (Viewer, Contributor, Admin) — for workspace-level object access |
| **Authorization (data)** | Fine-grained data permissions via Unity Catalog (catalog/schema/table/column/row) — separate from workspace roles |
| **Item-level permissions** | Per-item ACLs in Databricks or Fabric item permissions — control access to specific pipelines, notebooks, projects |
| **Sensitivity labels** | Purview + Microsoft 365 Information Protection — auto-classify and tag data (e.g., "Highly Confidential" for PII); enforce access policies based on labels |
| **Data-level access** | RLS, CLS, OLS — restrict access to rows, columns, or objects within a table |
| **Masking/encryption** | Mask or encrypt PII at ingestion; metadata must travel with data through all layers |

---

### Access Methods in Microsoft Fabric and Azure Databricks

[FACT] Fabric sharing via shortcuts: when using OneLake shortcuts to share data, permissions from both the shortcut path and the target path apply — the **most restrictive permission between the two paths wins**. This prevents shortcuts from being used as permission bypass mechanisms.

[FACT] Fine-grained data access in Medallion layers via RLS example (SQL):
```sql
CREATE SCHEMA Security;
CREATE FUNCTION Security.udf_securitypredicate(@SalesPerson nvarchar(50))
    RETURNS TABLE WITH SCHEMABINDING AS
    RETURN SELECT 1 AS result
    WHERE @SalesPerson = USER_NAME() OR IS_ROLEMEMBER('manager') = 1;

CREATE SECURITY POLICY SalesFilter
ADD FILTER PREDICATE Security.udf_securitypredicate(SalesPerson)
ON SalesLT.Customer WITH (STATE = ON);
```

[FACT] Security metadata (RLS policies, column tags, sensitivity labels) must travel with the data through all Medallion layers. If a PII classification is set in Bronze, the same classification must be enforced in Silver and Gold — otherwise the policy breaks when consumers read from upper layers.

[FACT] **ADLS ACLs** for security: not recommended as a best practice. Results in coarse-grained permissions difficult to manage precisely; provides extensive access to all underlying data.

[FACT] Power BI security options:
- **Import mode**: data refreshed using dataset owner's credentials; SSO for initial connection
- **DirectQuery to Databricks SQL**: use Microsoft Entra pass-through authentication — Databricks SQL verifies data access privileges via Unity Catalog using the end user's identity before returning results

---

### Chapter 12 Conclusion: Governance Must Grow with the Platform

[FACT] Effective data security is not just about deploying tools — it requires: defined roles, clear processes, efficient workflows, policy documents, and comprehensive user training. A holistic approach where governance, data culture, change management, and architecture all develop **in parallel and at equal pace**.

[FACT] The danger of asymmetric maturity:
- Perfect data platform without governed data → can't trust the data
- Mature governance organization without a stable platform → can't trust the data processing

[ANALYSIS] This principle applies to any platform team: change scripts, dependency checks, and a knowledge base are governance investments, not overhead. Adding new integrations without updating metadata procedures or documentation is the anti-pattern — platform and governance maturity must advance together.

---

## Chapter 13: Generative AI and Medallion Architectures

### Unstructured Data in Medallion

[FACT] Traditional Medallion architecture focuses on structured data (well-organized, queryable via Delta/Iceberg). Modern AI applications require **unstructured data** (PDFs, emails, social media, images, speech, JSON/XML documents). The key enabling pattern is **Retrieval-Augmented Generation (RAG)**.

[FACT] LLMs primarily engage with semi-structured and unstructured data formats. They efficiently handle both structured and unstructured data for: extracting insights, reorganizing data, generating new content, NLP tasks.

---

### The RAG Pattern

[FACT] **RAG (Retrieval-Augmented Generation)** = a framework that enhances LLM outputs by incorporating external knowledge at inference time (rather than relying solely on the model's training data). Steps:

1. **Capture raw documents** — extract from structured + unstructured sources; categorize by source, date, business process
2. **Generate document metadata** — creation date, titles, page numbers, URLs (can be auto-generated by SLMs)
3. **Organize and standardize** — rearrange in a standard format; ensure reusability
4. **Chunk documents** — break into smaller pieces sized for the embedding model's token limits
5. **Embed chunks** — an embedding model transforms each chunk into a numerical vector (captures semantic meaning)
6. **Index chunks in vector database** — load vectors + chunk text into a vector DB for semantic search

[FACT] At query time: user prompt → embedded into vector → vector DB retrieves most similar chunks → chunks + original prompt → LLM → contextually richer, more precise response.

[FACT] RAG can also process structured data (ERP/CRM tables), not just unstructured documents.

---

### Unstructured Data Across Medallion Layers

**Bronze layer (unstructured):**

[FACT] Same role as for structured data: capture and archive in original form (PDF, DOCX, TIFF, log files, emails). Additional activities:
- Transfer raw files to ADLS / data lake in original format
- Generate metadata at ingestion: source, format, uploader, creation date — stored in data catalog or alongside data
- Use **Small Language Models (SLMs)** for metadata generation: classification, labeling, PII identification, entity extraction, summarization (more efficient than full LLMs for these specific tasks)
- Organize into folders mirroring business processes or data origins (Teams channels, SharePoint folders)
- Partition by date for time-based versioning (similar to Bronze archive partitioning for structured data)
- Maintain references to schema definitions, parsers, and extraction scripts in each folder

[FACT] PII/sensitivity labels generated by SLMs in Bronze **carry forward** to Silver and Gold to enforce data access policies in upper layers.

**Silver layer (unstructured):**

[FACT] Emphasis shifts to refining and stabilizing unstructured data for AI-driven use cases and future fine-tuning of LLMs. Key activities:
- Partition raw data into logically organized, semantically meaningful units
- **Noise detection and duplicate identification** — filter irrelevant/erroneous content
- **LLMs as data correctors** — identify and fix errors in content
- Convert to machine-readable format: **Markdown** is the recommended standard (lightweight, AI-parseable, minimizes complex formatting errors)
- Frameworks: MarkItDown, PyMuPDF for standardizing output
- Summarize extensive documents concisely
- Break complex documents into smaller parts; extract images and tables with references (not chunking — chunking is deferred to Gold)
- Translate non-standard languages to organizational language
- Create sensitivity classifiers/labels (e.g., "Confidential" by content type, "Low Risk" by access controls)
- Classify and categorize text; perform entity recognition (party names, contract dates, obligations → structured database)
- Topic modeling and trend analysis
- Handle PII within documents via breaking into fine-grained access-controlled parts
- Store metadata in catalog, metadata store, or alongside data in lake

[FACT] Silver's unstructured data can feed upstream systems: a knowledge graph tool can consume entities and metadata from Silver without waiting for Gold processing.

[FACT] Specific chunking strategies are deferred to Gold — Silver creates stable, predictable data but does not specialize it for any particular AI application.

**Gold layer (unstructured):**

[FACT] Gold focuses on tailoring unstructured data for specific AI applications. Activities:
- **Select** most relevant documents for the specific use case (by keywords, topics, entities)
- **Data augmentation** — make data more representative, accurate, and diverse for the application
- **Chunk documents** — break into manageable pieces aligned with embedding model input limits
- **Generate embeddings** — transform chunks into vector representations using an embedding model appropriate for the use case
- **Index in vector database** — Pinecone, Azure AI Search, LanceDB, or Quokka for distributed vector search
- Manage chunking strategy + embedding model choice together (interdependent decisions)

[FACT] **Chunking strategy considerations**:
- Embedding models have token limits; exceeding them degrades performance or breaks processing
- Chunk size depends on use case: question-answering → paragraph-sized chunks; detailed analysis → smaller chunks

[FACT] **Embedding model considerations**:
- Low-dimensional models: efficient, minimal resources, suitable for real-time (chatbots)
- High-dimensional models: intricate semantic representation, suitable for detailed analysis (academic research)
- Domain-specific knowledge affects model choice (general vs. specialized)

[FACT] Future direction: Spark is expected to better support vector operations natively (approximate nearest neighbor searches, range searches), eliminating the need to transfer data to separate vector databases for most use cases.

---

### LLM Integration Scenarios in Medallion

[FACT] Three scenarios for combining LLMs with the Medallion architecture:

| Scenario | Description | LLM Role |
|---|---|---|
| **1. Data transformation, cleaning, enrichment** | LLMs parse complex/variably structured formats (XML, log files, emails) without predefined schemas; extract specific fields dynamically | Data integrator; schema-free parser |
| **2. Preprocessing and feature engineering** | Sentiment analysis, data structuring, multi-label classification, gap filling, error correction, time-series forecasting (TimeGPT-1) | Data enricher and augmenter |
| **3. RAG application serving** | Chunking, embedding, vector search, response generation — full RAG pipeline on top of the Medallion Gold layer | Consumer of prepared data; response generator |

[FACT] When developing LLM applications, treat the Medallion architecture as a **connectivity and data preparation layer** that harmonizes APIs, data products, and events into a coherent foundation for intelligent applications. This requires close collaboration between data engineers and application integration engineers.

---

### Unstructured + Structured: A Unified Architecture

[FACT] The Medallion architecture's layering concept applies equally to structured and unstructured data: Bronze (capture/archive), Silver (refine/standardize/stabilize), Gold (specialize/serve). Using the same architecture for both data types:
- Aligns diverse disciplines (data engineering, ML engineering, BI) into one coherent framework
- Enables LLMs to generate metadata that enhances both structured AND unstructured data management
- Reduces operational complexity (one platform, one governance model)

[ANALYSIS] Teams currently handling only structured/semi-structured data (relational tables, Kafka JSON, CSV files) can extend to unstructured data using the same Bronze→Silver→Gold progression: raw file store (Bronze) → standardized Markdown/metadata (Silver) → chunked+embedded for AI serving (Gold). The key new infrastructure requirement: a vector database layer at Gold.

---

## Source Reference

| Concept | Book location |
|---|---|
| Unity Catalog overview, open source, UniForm, Delta Sharing | Ch. 12, pp. 601–603 |
| Unity Catalog 3-level namespace, CI/CD naming, workspace-catalog binding | Ch. 12, pp. 605–607 |
| Unity Catalog access model (Table 12-2), group-based permissions | Ch. 12, pp. 610–612 |
| Unity Catalog key features (lineage, tagging, sharing) | Ch. 12, pp. 608–609 |
| Unity Catalog vs Purview (operational vs referential catalog) | Ch. 12, pp. 612–613 |
| Data contracts: definition, distinction from data products | Ch. 12, pp. 613–614 |
| Data contracts within Purview catalog | Ch. 12, pp. 614–616 |
| Data contracts within metastore (Azure SQL + Logic Apps + Functions) | Ch. 12, pp. 617–619 |
| Data contracts YAML + GitOps + Data Contract CLI | Ch. 12, pp. 619–622 |
| dbt model contracts | Ch. 12, pp. 621–622 |
| Data security: network, SSO/MFA, audit, recovery | Ch. 12, pp. 623–625 |
| Workspace roles, item-level permissions, sensitivity labels | Ch. 12, pp. 625–628 |
| Unity Catalog in Databricks: privileges, attribute-based security tags | Ch. 12, pp. 628–629 |
| Shortcuts in Fabric: most-restrictive permission wins | Ch. 12, pp. 629 |
| Fine-grained RLS/CLS/OLS, security metadata through layers | Ch. 12, pp. 629–637 |
| Power BI security (import vs DirectQuery, Entra pass-through) | Ch. 12, pp. 630–632 |
| Chapter 12 conclusion: parallel maturity required | Ch. 12, pp. 637–638 |
| RAG pattern overview (6 steps, vector DB) | Ch. 13, pp. 642–645 |
| Bronze layer (unstructured): capture, SLMs for metadata, partitioning | Ch. 13, pp. 645–648 |
| Silver layer (unstructured): Markdown standardization, chunking deferred | Ch. 13, pp. 649–652 |
| Gold layer (unstructured): chunking, embedding models, vector databases | Ch. 13, pp. 652–656 |
| LLM integration scenarios (3 scenarios) | Ch. 13, pp. 658–660 |
