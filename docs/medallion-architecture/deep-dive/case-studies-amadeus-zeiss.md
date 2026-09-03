# Medallion Architecture — Case Studies: AP Pension (Gold/Serving), Amadeus, ZEISS

**Source:** *Building Medallion Architectures* by Piethein Strengholt (O'Reilly, 2025), Ch. 8 conclusion (pp. 481–499), Ch. 9 (pp. 500–531), Ch. 10 opening (pp. 532–540).

---

## AP Pension — Gold Layer and Serving Architecture

*(Continues from `medallion_data_products_and_governance.md`, which covers Bronze through Silver Base at pp. 462–480.)*

### Silver Enriched Layer: Lightweight Dimensional Preparation

[FACT] AP Pension's Silver **Enriched layer** sits between the source-aligned Base layer and the Gold Curated layer. Its purpose: prepare data for the Gold layer dimensional framework and stage it for reuse across multiple downstream processes.

[FACT] Enriched layer responsibilities:
- Merge data from various source systems (Base data is still source-aligned)
- Pre-compute hash keys for identifying changes in source data (since Delta tables don't support primary key enforcement)
- Compute surrogate keys by hashing business keys — enables simultaneous loading of both facts and dimensions in Gold
- Handle **gap islands** — aligning mismatched timelines from different sources (gaps: holes in sequences; islands: ranges of consecutive identical values)

[ANALYSIS] The Enriched layer plays the same role as pre-load staging steps in SQL-based ETL frameworks — joining sources and computing hash keys before the final dimension/fact load. Pre-computing these joins into a temp structure is a standard pattern for avoiding expensive repeated join execution in the main transformation query.

---

### Gold Layer: Three Sublayers

[FACT] AP Pension's Gold layer is divided into three sublayers:

| Sublayer | Purpose | Model |
|---|---|---|
| **Curated** | Enterprise dimensional model; single version of truth; integrates source-oriented data from Silver | Star schema with SCDs |
| **Modeled** | Consumer-ready layer; simplifies data model for specific users; prepares data for semantic models and applications | Star schema with pre-joined views; denormalized tables for data scientists |
| **Modeled PII** | Handles decryption of personal data for authorized users in a controlled environment; tagged in Purview for data loss protection | Pseudonymized base + controlled decryption service |

[FACT] The **Curated layer** uses a framework that handles surrogate key creation and ensures full integration between dimensional model elements. It is heavily metadata-driven.

[FACT] Before adopting Fabric, AP Pension used a data vault model with PIT tables. In their current framework they have incorporated **PIT (Point-In-Time) tables** into the Gold Curated layer to manage multiple SCD2 dimensions simultaneously. This allows following different timelines separately (creation date, value date, transaction receipt date) — critical for pension data where a payment can refer to a past date and affect historical status.

[FACT] AP Pension describes their pattern as **SCD type 7 hybrid**: type 1 keys saved on the fact table, multiple type 2 keys available in PIT tables. This allows the Modeled layer to establish singular dimensional join paths for each use case or timeline.

[ANALYSIS] This PIT table pattern is a practical alternative to data vault for organizations that need multiple-timeline analysis without the full complexity of data vault's hub-satellite-link model. The key insight: move PIT table logic into the Gold framework rather than exposing it to all consumers — consumers see only the Modeled layer's simplified join paths.

---

### Modeled PII Layer: Privacy-by-Design in Gold

[FACT] AP Pension's Modeled PII layer:
- All data in the regular data platform is pseudonymized or anonymized with unique integration keys per customer — no PII stored in the main Medallion layers
- When a business need requires personal data (e.g., compliance reporting): a controlled decryption service provides access in a controlled environment
- Access to decrypted data is **tagged in Microsoft Purview** → data loss protection policies managed at the Entra ID level; access is logged and reportable
- Based on the **principle of data minimization**: identify which users have a legitimate reason to decrypt, then proceed selectively — not blanket access to all decrypted data

[ANALYSIS] This is the serving-layer implementation of AP Pension's Bronze-layer per-customer encryption (described in the previous doc). The chain: encrypted at Archive (Bronze) → pseudonymized through Silver → Purview-tagged controlled decryption in Modeled PII (Gold). The architecture enforces minimum necessary access at every layer.

---

### Serving Layer: Domain Workspaces on Demand

[FACT] AP Pension's serving layer uses **individual Fabric Workspaces per logical consumer group** (APData Function Workspaces), generated on demand from metadata and templates. Each workspace:
- Includes at least one Fabric Lakehouse + one Fabric Warehouse
- Users are **viewers** of the Workspace (read-only) and **owners** of the Warehouse (read-write for their own tables)
- Lakehouse contains **shortcuts** to Medallion layer data — no physical data movement
- From the Warehouse, users can query data from Lakehouse shortcuts using T-SQL (SSMS or Fabric GUI) and save their own tables to the Warehouse

[FACT] Workspace creation decisions are based on: functional roles, security requirements, logical data grouping, and workload isolation. Workspaces are provisioned via an automated, metadata-driven process using Azure DevOps + Fabric administrative APIs.

[ANALYSIS] The shortcut-based serving layer is what makes zero-copy distribution possible: data stays in the central Lakehouse and is accessed read-only from each domain workspace via OneLake shortcuts. Storage cost is incurred once; compute cost is incurred per workspace. This is the "move data as little as possible" principle operationalized.

---

### Capacity and Workload Isolation

[FACT] In Fabric, compute resources are shared across artifact types within a capacity, and capacities are assigned at the workspace level. AP Pension's approach:
- Heavy monthly batch workspaces get different capacity assignments than daily ad-hoc query workspaces
- Surge protection is configured; capacity usage is monitored closely
- Workspaces are created from metadata + templates to maintain consistency at scale

[FACT] AP Pension classifies users into three categories: **co-creators** (domain experts who can develop working prototypes using low/no-code tools like Data Wrangler, Dataflow), **advanced users**, and **report users**. Prototypes created by co-creators are subsequently refined and integrated by the data platform team.

---

### CI/CD: Metadata-Driven Fabric Deployment

[FACT] AP Pension's CI/CD approach for Fabric artifacts:
- A pipeline generates deployment scripts from registered metadata for each environment
- Scripts call the **Fabric administrative APIs** (REST)
- Azure DevOps manages the pipelines (mature product, handles both Fabric and non-Fabric items)
- Terraform Provider for Microsoft Fabric is being adopted to further automate infrastructure provisioning

[ANALYSIS] This "configuration as code" approach is a practical middle ground between Fabric's built-in deployment pipelines (workspace-level promotion) and full infrastructure-as-code (Terraform). Azure DevOps provides the CI gate; Fabric APIs provide the deployment primitive. Metadata-driven change script generation follows the same principle: generating deployment artifacts from a metadata repository rather than hand-authoring each object deployment.

---

## Amadeus — Data Mesh at Scale

### Background

[FACT] Amadeus: global travel technology provider, 190+ countries, 19,000 employees (10,000 in R&D). Architecture lead: Joel Singer (Head of Data Engineering, 23 years at Amadeus).

[FACT] Architecture evolution: fragmented pre-2021 (multiple Hadoop/cloud stacks, acquired company silos, limited sharing) → 2021 Microsoft strategic partnership → cloud migration to Azure Databricks → all platforms eventually migrated, forming a unified **data mesh** in the cloud.

[FACT] Amadeus's three data strategy pillars:

| Pillar | Description |
|---|---|
| **Data mesh** | Each domain (airlines, airports, travel agencies, hotels) manages its own data using domain expertise; spans both operational and analytical data |
| **Open data** | Data is valuable, quality-monitored, lineage-tracked, discoverable (data catalog), accessible (APIs/analytics), secure (audit trails), interoperable |
| **Data 360** | Use data to build knowledge → improve products → generate more data; creates a continuous improvement loop (e.g., ML-optimized flight shopping handling billions of daily transactions) |

---

### Architecture: 10 Domains, 130 Workspaces

[FACT] Amadeus manages 10 data domains mapped to **landing zones** (Azure landing zones — subscriptions with resources, per Cloud Adoption Framework). Domains are not tied to organizational structure (anticipating org changes over time). Within the 10 domains: 130 workspaces, each scoped to specific workload types and not linked to teams.

[FACT] Each workspace is deployed from a **catalog of approved, pre-secured services** (self-service portal direction). Teams choose services from the catalog, deploy them within their workspace. Standard templates exist for common patterns; flexibility for custom service selection is also supported.

---

### Medallion Layers at Amadeus

[FACT] Amadeus uses 3-layer Medallion: ingestion (Bronze), processing (Silver), exposure (Gold). All teams follow the same architecture.

**Bronze layer:**
[FACT] Treated as an **immutable archive**. Data stored in its original format (EDIFACT, XML, JSON, proprietary travel formats). Purpose: (1) compliance archive; (2) raw data access for data scientists. Primary access: external tables in Azure Databricks.

[FACT] Bronze at Amadeus is **not Delta-formatted** for querying — users who need raw data prefer to work with it using their own tools. Delta format introduced at Silver.

**Silver layer:**
[FACT] Key transformations at Silver:
- Normalize to a consistent format (JSON)
- Clean and standardize (date normalization, field standardization)
- Denormalize complex hierarchies — some records contain up to 2,000 data elements; flattening simplifies downstream processing
- Reference data integration directly at Silver (simplifies queries; maintains clear data lineage)
- Format: **Delta** on Azure Databricks

[FACT] Amadeus previously used **raw vault** at Silver but abandoned it due to cost and performance impact. Replaced with direct JSON normalization in Silver → star schema generation in Gold.

**Gold layer (JSON2Star):**
[FACT] Amadeus built an in-house library called **JSON2Star** that automates Silver → Gold transformation:
- Input: normalized JSON datasets from Silver
- Input: configuration file defining the desired star schema (single source of truth for the data model)
- Output: all scripts needed to create tables and load data; hourly execution
- Integrated with CI/CD pipelines (configuration file versioned in Git → automated script generation → automated deployment)
- Schema is a **Galaxy schema** (constellation of star schemas): multiple fact tables + multiple interconnected dimension tables

[ANALYSIS] JSON2Star is a domain-specific ETL code generator: the data model config file plays the same role as a mapping repository in a metadata-driven framework. The implementation differs — JSON2Star generates Databricks (Spark) scripts; SQL-first frameworks generate stored procedures. Both solve the same problem: metadata-driven Gold layer generation without hand-authoring each load procedure.

---

### Data Sharing and Access Control

[FACT] Cross-domain data sharing workflow at Amadeus:
1. Team requests access to a dataset from another domain
2. Collaboration between data owner and legal team to verify sharing is permissible
3. Access granted via **Microsoft Entra ID** to the data container
4. For complex, multi-entity records: **Unity Catalog** provides fine-grained row/column access based on use case or user identity
5. All access requests and data usage are tracked via audit trail

[FACT] The system supports both individual user accounts and **service principals** (representing products/applications) — e.g., Amadeus Dynamic Pricing uses data from reservation system + third-party data + reference data via SPNs.

[FACT] A **data catalog** (control plane) provides visibility into all data exchanges (big data processes, APIs, events) and tracks what was declared vs. what is actually happening in production.

---

### FinOps: From Cost Center to Virtual P&L

[FACT] Amadeus's FinOps evolution:
- **Initial model**: costs (storage + compute) allocated to data producers → disincentivizes sharing because producers pay for usage they don't benefit from
- **Consumer-centric model**: costs allocated to the end products that consume the data → shopping data used by 20 products → cost distributed proportionally among those products
- **Virtual P&L model (in progress)**: producers earn virtual revenue for their data being consumed → incentivizes higher-quality data sharing; requires mapping producer data → consumer products

[ANALYSIS] The virtual P&L model treats data as a traded asset within the organization. This is the financial mechanism that makes a data mesh's decentralized model self-sustaining: without consumption-based incentives, producers accumulate costs with no benefit, which causes sharing to stagnate. Central platform teams typically absorb costs via shared capacity billing — this works at small scale, but as domain teams become data producers, FinOps allocation becomes a real design question.

---

### Common Data Model and Data Contracts

[FACT] Amadeus's data model approach:
- **Logical model** (polyglot — not tied to any implementation technology): common data model shared across all domains
- **Physical models** derived from the logical model per interface type:
  - Events: AsyncAPI specification
  - APIs: OpenAPI specification
  - Big data: data contract
- From physical models: consumer documentation, code templates, SDKs are auto-generated via CI/CD
- Stored in a Git repository organized to mirror the data domain hierarchy

[FACT] **Data families** = centrally managed common data models that apply to most of the company (e.g., flights, travelers, pricing). Each domain can adapt by removing unnecessary elements or adding domain-specific fields. Regularly discussed and refined centrally to absorb new trends without per-team divergence.

[FACT] Amadeus defines three converging standards for data exchange:
- **OpenAPI** (for REST APIs) — mature, widely adopted
- **AsyncAPI** (for events) — emerging standard, gaining adoption
- **Data contracts** (for big data) — newest, being standardized; expected to converge with the other two

[ANALYSIS] The convergence of OpenAPI/AsyncAPI/data contracts represents an industry trend toward a unified schema-first approach to all data exchange, regardless of transport mechanism. Teams integrating Kafka (AsyncAPI) alongside REST API sources (OpenAPI) will benefit from a single schema governance workflow rather than separate specifications per transport.

---

### Governance Evolution: Three Phases

[FACT] Amadeus governance evolved through three phases:

| Phase | Era | Model | Mechanism |
|---|---|---|---|
| **Governance 1.0** | ~15 years ago | Centralized | Single R&D group; top-down standards easily enforced |
| **Governance 2.0** | After company expansion | Community-based | Central body issues guidelines; community members implement changes within their engineering groups |
| **Governance 3.0** | Current (cloud era) | **Federated** | Central governance body sets global policies, processes, rules, provides data families, manages catalog; each domain has a local governance body for day-to-day changes and external publication in compliance with central rules |

[FACT] Amadeus's recommendation for governance: **govern centrally but empower locally**. Establish a strong central body for both data governance and architecture standards; empower individual data domains to execute the strategy and make their data accessible in an easily usable format.

[ANALYSIS] Governance 3.0 is the federated data mesh governance model: central guardrails + local execution. This matches the "hybrid" Medallion governance topology (from Ch. 7): critical/regulatory data centrally managed; domain-specific data managed by BU. The pattern works at scale because local teams have real ownership, but the central body prevents fragmentation.

---

## ZEISS — Platform Evolution and Federated Model

### Background

[FACT] ZEISS: optics industry pioneer, founded 1846, ~45,000 employees, ~€10B revenue. Interviewees: Markus Morgner (head, Enterprise Data Foundation), Sascha Saumer (senior data engineer), Gert Christen (Microsoft BI platform manager, leading Fabric adoption).

[FACT] Before 2021: decentralized data initiatives in silos across segments. 2021: data strategy established + central **data and analytics team** formed, consolidating master data governance, BI, management reporting, data scientists, architects, and data governance roles.

[FACT] Current state: central organization + federated responsibility to segments. Segments use central platforms and governance processes. Model: strong core with central control + empowered segment units.

---

### Platform Evolution: eVA Generations

[FACT] ZEISS data platform history:

| Period | Platform | Description |
|---|---|---|
| 2017–2018 | Azure sandbox | Experimentation, no real use cases |
| 2018–2022 | eVA 1.0/2.0 | Real use cases, organically grown, single analytics stack + data lake; shared resource problems |
| End 2022+ | eVA 3.0 | Adopted Microsoft Cloud Adoption Framework; 30 instances, each with 3–4 Azure subscriptions; Terraform module stacks; built-in data governance and security from the start |
| Roadmap | eVA 4.0 | eVA 3.0 + Fabric integration; Fabric used in the consumer-aligned layer on top of eVA 3.0 data mesh |

[FACT] eVA 3.0 structure: central hub for data management, security, monitoring, metadata extraction (data catalog); **data landing zones (DLZs)** per domain (using Cloud Adoption Framework landing zone definition = subscription + resources).

---

### Three Instance Patterns

[FACT] ZEISS's three deployment patterns for platform instances:

| Pattern | Description | Use case |
|---|---|---|
| **Central instances** | Provide data products used company-wide (e.g., finance) | Cross-segment common data — no need to reinvent per BU |
| **Decentralized DLZs** | Unique to specific business domains or processes; restrictive data, minimal communication to other zones | Sensitive domain-specific data |
| **Hybrid instances** | ZEISS provides infrastructure + security monitoring; BU handles development (or collaborates with ZEISS depending on maturity) | BUs with variable data engineering capability |

[ANALYSIS] These three patterns mirror the governance topology table from Ch. 7 — central, decentralized, and hybrid — applied to infrastructure provisioning. The same governance principle appears as an infrastructure deployment pattern: adjust centralization/decentralization based on data sensitivity and BU capability, not a single uniform model.

---

### Technology Stack: Organic Migration

[FACT] ZEISS does not prescribe which technology domain teams must use. Teams choose what fits their needs; ZEISS provides guidance. Supported technologies include Azure Synapse Analytics, Azure Databricks, and Microsoft Fabric.

[FACT] Observed technology migration pattern (organic, not mandated):
- Initial heavy Synapse usage in data lakehouses → organic shift toward **Azure Databricks** for complex scenarios requiring additional flexibility
- Synapse SQL **serverless pools** still in use due to ease-of-use for SQL queries
- Unity Catalog adoption simplified the Synapse → Databricks transition
- No technology is being phased out by policy

[FACT] eVA 4.0 roadmap: **Fabric** will serve the consumer-aligned layer (on top of eVA 3.0 data products). This positions Fabric as the semantic/serving tier (Power BI Direct Lake, semantic models) while Databricks/Synapse remain for the Medallion transformation layers.

---

## Cross-Case Design Patterns

[ANALYSIS] Patterns appearing consistently across AP Pension, Amadeus, and ZEISS:

| Pattern | AP Pension | Amadeus | ZEISS |
|---|---|---|---|
| **Federated governance** | APData central + serving workspaces | Governance 3.0 (central body + domain local bodies) | Central core + empowered segments |
| **Metadata-driven deployment** | Azure DevOps + Fabric REST APIs + metadata | CI/CD pipelines + single source of truth config | Terraform modules + metadata |
| **No universal technology mandate** | Spark-first for Medallion (vs SQL for serving) | Databricks but no prohibition on other stacks | Teams choose; guidance not mandates |
| **Data products at Gold or above** | Modeled layer (consumer-facing) | Gold (star schemas via JSON2Star) | Consumer-aligned Fabric layer in eVA 4.0 |
| **Privacy/PII: minimize exposure** | Modeled PII sublayer + Purview tagging | Access approval workflow + audit trail | (Security via DLZ isolation) |
| **Serving via shortcuts / virtual access** | OneLake shortcuts in domain workspaces | External tables in Databricks (Bronze) | (Future: Fabric shortcuts from eVA 3.0) |

[ANALYSIS] The recurring theme: no organization uses a single, uniform architecture for all data. All three use tiered models where central teams manage shared infrastructure and common data products, while domain teams manage their specific workloads within guardrails. This is the operationalization of "federated computational governance" from data mesh principles.

---

## Source Reference

| Concept | Book location |
|---|---|
| AP Pension: Silver Enriched layer, gap islands | Ch. 8, pp. 481–482 |
| AP Pension: Gold Curated layer, PIT tables, SCD type 7 | Ch. 8, pp. 483–485 |
| AP Pension: Modeled layer (consumer-ready, denormalized) | Ch. 8, pp. 484–487 |
| AP Pension: Modeled PII layer, Purview tagging | Ch. 8, pp. 487–488 |
| AP Pension: Serving layer (domain workspaces, shortcuts) | Ch. 8, pp. 488–492 |
| AP Pension: Capacity and workload isolation | Ch. 8, pp. 491–492 |
| AP Pension: CI/CD (Azure DevOps + Fabric APIs) | Ch. 8, pp. 494–495 |
| AP Pension: Data contracts and Purview DQ | Ch. 8, pp. 495–497 |
| Amadeus: background, data strategy pillars | Ch. 9, pp. 500–504 |
| Amadeus: data mesh (operational + analytical) | Ch. 9, pp. 504–506 |
| Amadeus: 10 domains, 130 workspaces, workspace blueprint | Ch. 9, pp. 506–509 |
| Amadeus: Bronze (archive), Silver (normalize/denormalize), Gold (JSON2Star, Galaxy schema) | Ch. 9, pp. 510–516 |
| Amadeus: cross-domain sharing, Unity Catalog, audit trail | Ch. 9, pp. 516–518 |
| Amadeus: data product definition | Ch. 9, pp. 518–520 |
| Amadeus: FinOps, consumer-centric costs, virtual P&L | Ch. 9, pp. 520–522 |
| Amadeus: common data model, data families, data contracts | Ch. 9, pp. 522–527 |
| Amadeus: governance evolution (1.0 → 2.0 → 3.0) | Ch. 9, pp. 528–530 |
| ZEISS: background, data strategy 2021 | Ch. 10, pp. 532–535 |
| ZEISS: eVA platform evolution (1.0 → 3.0), 3 instance patterns | Ch. 10, pp. 535–538 |
| ZEISS: technology stack (Synapse/Databricks/Fabric), eVA 4.0 roadmap | Ch. 10, pp. 539–540 |
