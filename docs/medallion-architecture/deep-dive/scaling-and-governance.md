# Medallion Architecture — Scaling, Inner Architecture Variations, and Governance

**Source:** *Building Medallion Architectures* by Piethein Strengholt (O'Reilly, 2025), Ch. 10 conclusion (pp. 541–556), Ch. 11 (pp. 559–593), Ch. 12 opening (pp. 594–600).

---

## ZEISS — Medallion in Practice (Conclusion)

*(Continues from `medallion_case_studies_amadeus_zeiss.md`, which covers ZEISS through p. 540.)*

### Medallion Layers at ZEISS

[FACT] ZEISS recommends Medallion architecture within all platform instances, but does not mandate a fixed number of layers. Typical setup: 3 layers (Bronze, Silver, Gold), sometimes split into schemas or processes within a storage account. Some instances use only 2 layers (staging + presentation).

[FACT] Ingestion patterns at ZEISS: both push (event-driven/real-time) and pull (nightly batch). Bronze layer preserves data in original format (usually JSON files) with only minor technical adjustments (e.g., sanitizing illegal column name characters). Transformations (type enforcement, denesting, schema changes) are applied in Silver and beyond.

[FACT] ZEISS is considering adding a **pre-Bronze layer for SAP data**: consolidate multiple SAP systems (S4, older versions) within SAP before exporting refined views to Azure. Rationale: SAP has its own meta models and tools for consolidation, making pre-processing faster within SAP than outside it.

[ANALYSIS] The SAP pre-processing pattern is a pragmatic application of the "move data as little as possible" principle: do consolidation where the tooling is best (inside SAP), export clean views. This is architecturally equivalent to AP Pension's Silver Base layer that stays close to the source — it defers cross-system integration to a later stage.

---

### Gold Layer and Power BI Ambiguity at ZEISS

[FACT] At ZEISS, Power BI semantic modeling is happening **after** the Gold layer — some modeling still occurs on the Power BI side. This creates ambiguity: are those Power BI models part of Gold? Answer: Gold layer boundaries can be blurry. Import mode is primary in Power BI; DirectQuery is used for real-time or very large models.

[FACT] ZEISS's catalog landscape (as of the time of interview): **three separate catalogs** with no current integration between them:
1. **Collibra** — enterprise data catalog (replacing Informatica; not yet production-ready)
2. **Microsoft Purview** — platform management tasks and data discovery within Fabric scope
3. **Unity Catalog** — Databricks-specific asset sharing across instances

Planned: **catalog of catalogs** approach — Collibra as primary, with integration to Unity Catalog, Fabric Catalog, and future catalogs.

[ANALYSIS] Multiple catalogs without integration is a common growing pain for large enterprises that adopted different technology stacks independently. The "catalog of catalogs" pattern is the realistic answer at scale: federate the catalog index rather than forcing a single tool. Teams running Fabric + Databricks concurrently will face this as each platform brings its own catalog (Purview vs Unity Catalog).

---

### ZEISS: Global vs. Local Data Products

[FACT] ZEISS differentiates two tiers of data products:
- **Global data products**: intended for company-wide sharing; created on eVA 3.0 (source-aligned + aggregate domains); require rigorous central governance since they affect everyone; direct onboarding to Fabric bypassing eVA 3.0 is not a desired pattern (except for small/infrequent sources)
- **Local data products**: used within departments or business units for internal reporting; federated governance within BU; no strict central governance required

[FACT] Consumer-aligned domains (Power BI, reporting, AI use cases) are positioned on **Fabric**, sitting on top of eVA 3.0 data products. Fabric is not recommended for source-aligned or aggregate domain activities (those stay on eVA 3.0). Data sharing from eVA 3.0 to Power BI requires a **gateway** because eVA 3.0 data products are VNET-protected.

---

### ZEISS Recommendations

[FACT] ZEISS's primary governance advice: don't dive straight into technology when adopting data mesh. Start with a holistic view, clear domain boundaries, and well-defined policies. Technology enablement follows conceptual clarity — not the other way around.

[FACT] Data contracts are identified by ZEISS as a future necessity: they will technically represent what you are allowed to do with data, what a data product is supposed to do, when to scramble or delete data, and expected quality level. Policy automation must pace with platform growth.

---

## Chapter 11: Scaling the Medallion Architecture

### Data Mesh as a Spectrum

[FACT] Data mesh is not a binary choice; it is a **spectrum** from fully centralized to fully decentralized. The federated operating model shifts responsibilities from central IT to business domains:

| Traditional central model | Federated data mesh model |
|---|---|
| Central IT manages: operational systems, ingestion, staging, cleaning, transforming, data products, sharing | Central IT manages: complex operational systems + central data management services |
| Business: data usage only | Business domains manage: ingestion, staging, cleaning, transforming, data products |

[ANALYSIS] No organization moves from fully centralized to fully decentralized in one step. The practical path is incremental federation: identify which domains have enough maturity and data ownership culture to take on more responsibility, and give them autonomy progressively. A team at the "centralized engineering" end — where the platform team manages all Medallion layers and BUs consume Gold — moves toward federation by delegating Silver-layer ownership to domain teams while maintaining Gold integration standards centrally.

---

### Medallion Mesh: Multiple Medallion Architectures

[FACT] **Medallion mesh** (term coined by Franco Patano): a network of Medallion architectures within the same organization, capable of sharing data with each other. Each team or domain operates its own Bronze/Silver/Gold stack; data products flow between them.

[FACT] In **Azure Databricks**, the recommended sharing patterns within a Medallion mesh are:
- **Pattern 1: Unity Catalog** — cross-workspace data sharing with fine-grained access control
- **Pattern 2: Delta Share** — cross-organization or cross-cloud data sharing using the open Delta Sharing protocol

[FACT] In **Microsoft Fabric**, multiple Medallion architectures share a single **OneLake** as the underlying storage layer. Each domain has its own Lakehouse (logical data lake) but all Lakehouses are stored within the same OneLake. This deviates from pure data mesh (where each domain has its own dedicated infrastructure) but provides better governance and easier sharing.

[ANALYSIS] Fabric's shared OneLake is architecturally different from Databricks's cross-workspace Unity Catalog sharing: in Fabric, data is physically co-located (not replicated); access control is at the Workspace and item level. This makes Fabric closer to a "logical mesh over shared storage" than a true physically decentralized mesh. Practically: adding new domain Lakehouses costs zero extra storage and zero data copying via shortcuts.

---

### How Many Medallion Architectures? Key Drivers

[FACT] There is no one-size-fits-all answer for how many Medallion architectures (domains/platform instances) an organization needs. Key drivers:

| Driver | Impact on count |
|---|---|
| **Organizational size** | Small org → 1 architecture; mid → central + a few BU domains; large → one per BU + shared service domains |
| **Organizational maturity** | Mature → many fine-grained specialized domains; less mature → fewer broader domains |
| **Security and compliance** | Departments with different regulatory requirements may need separate domains |
| **Cost tracking** | Separate workspaces = cleaner cost allocation by department or project |
| **R&D / prototyping** | Research projects benefit from isolation to prevent unwanted integration |
| **Regional boundaries** | Data sovereignty laws (e.g., GDPR) may require per-region domains |

[FACT] Domain count is not fixed; it must evolve organically. The guidance: **start small, grow as needed** ("progression over perfection"). A central architecture team oversees and sets standards; domain proliferation without oversight leads to cost overrun and technology fragmentation.

[ANALYSIS] The right maturity progression: start with a centralized model (single Medallion architecture managed by the framework team), then introduce separate platform instances as individual domains demonstrate data ownership capability. Premature federation before governance is in place creates the same problems ZEISS described.

---

### Tailored Medallion Architectures: 7 Domain Patterns

[FACT] Not all domains need the same Medallion layers. The book defines 7 solution patterns based on a domain's role:

| Pattern | Layer count | Description |
|---|---|---|
| **Simple data provider** | 2 (Bronze + data product) | Few sources; data not used internally; just ingest + expose |
| **Single-use complex data provider** | 3 (Bronze + Silver + Gold) | Many sources; data used only internally |
| **Multi-use complex data provider** | 4–5 | Many sources; data used for various internal projects |
| **Basic consumer** | 0–1 | Primarily reads from other domains; minimal processing |
| **Integrated consumer** | 2–3 | Integrates data from multiple source domains for own projects |
| **Distributor consumer** | 3–4 (Bronze + Silver + Gold + data product) | Integrates from multiple sources + distributes to other domains |
| **Consumer provider** | Multiple | Consumes solely to distribute; not for personal use; complex management |

[FACT] A key scenario: when domain A's Gold/data product layer feeds domain B's processing, domain B can treat domain A's output as its effective **Bronze layer** — eliminating the need to replicate the data product layer in domain B's Bronze. This creates a leaner architecture by leveraging upstream Gold as downstream Bronze.

[ANALYSIS] Recognizing these patterns explicitly prevents teams from defaulting to the same 3-layer stack regardless of role. A reporting team (basic consumer) has no reason to maintain a Bronze archive layer — they should read via shortcuts from the upstream Gold. Consuming Gold dimensional tables via a semantic model, without owning a Bronze layer, is exactly this pattern.

---

### Separate Data Product Layer

[FACT] Mature organizations introduce an **additional layer dedicated to data product design and distribution**, separate from the team's internal data consumption (Gold) layer. Pattern: split Gold or Silver into:
- **Stable "data product" layer**: carefully cataloged, promoted to other teams, always with clear ownership; generic and reusable
- **Dynamic "domain- or use case-specific" layer**: managed by the team's applications; not promoted; may change freely

[FACT] This split allows teams to independently manage their external data products while keeping internal processing flexible. Data product layers require open table formats and interoperability standards for large-scale distribution.

---

### Bronze Layer Variations: Conglomerate Pattern

[FACT] The Bronze layer can be structured as a **conglomerate of multiple Lakehouse entities**, each handling different ingestion patterns:

| Bronze sub-entity | Use case |
|---|---|
| **Shortcuts** | Data already in compatible format (Delta/Iceberg); no transformation needed; zero-copy |
| **Physical copies** | Data requiring extensive processing before Silver/Gold; source format not directly queryable |
| **Replication (mirroring/CDC)** | Real-time synchronization; always up-to-date; feeds near-real-time consumers |

[FACT] Bronze can also be split by **team security boundary**: separate Lakehouse entities within Bronze prevent teams from seeing each other's raw data while sharing the same overall domain.

[ANALYSIS] The conglomerate Bronze pattern is common in practice: some source tables arrive via CDC/mirroring (replication), others via batch notebook writes (physical copies), and some upstream Gold outputs are consumed via shortcuts. Each ingestion mechanism maps to a different Bronze sub-entity type — the architecture accommodates them all within one logical Bronze layer.

---

### Silver and Gold Layer Variations

[FACT] Silver layer: keep inner structure aligned with the Bronze layer for consistent data handling and transformation rules. Silver decouples Bronze (ingestion) from Gold (consumption) — changes in Silver do not affect source-to-domain data provision or Gold consumers directly.

[FACT] Gold layer variations:
- **Sublayers for governance**: where one engineering team manages a large domain, split Gold into sublayers by business team or data domain, each with its own governance policies and designated owner. Common in large financial institutions.
- **Enterprise-wide standards**: Gold layers should adhere to enterprise data harmonization standards (**data curation** = organizing and managing data for easier integration). Despite complexity and time cost, enterprise-wide data modeling ensures reusability across domains.

---

### Enterprise Data Models and Domain Aggregates

[FACT] In a Medallion mesh, organizations often establish **domain aggregates**: domains dedicated to enterprise-wide integration/curation. These domains:
- Pre-integrate data from multiple source domains into harmonized data products
- Distribute harmonized data products to consuming domains (similar to the "consumer provider" pattern)
- Eliminate repetitive integration efforts across consuming business units
- Are typically managed by a central team focused on data quality, governance, and compliance

[FACT] Domain aggregates are **not ready-to-use use-case-specific products** — they are pre-integrated foundational datasets. Consuming domains may further transform them for their specific needs.

[FACT] Strategy for deciding what to pre-integrate in an aggregate domain:
1. Identify the most critical, most-joined data domains (e.g., customer, product)
2. Engage stakeholders to find which datasets they frequently combine
3. Use metadata/data catalog usage patterns to find common intersections and high-demand data
4. Mine legacy data warehouses (already pre-integrated = knowledge base for what's worth harmonizing)

[ANALYSIS] Conformed dimensions (e.g. DimCustomer, DimProduct) are domain aggregates in this terminology: they pre-integrate multiple source systems (CRM, ERP, operational databases) into a single version of the truth. This is architecturally correct and scalable — one authoritative record per entity, consumed by all fact tables.

---

### Master Data Management (MDM)

[FACT] MDM establishes a **single, accurate, authoritative source of truth for master data** (customer, product, supplier, employee — entities used across systems). MDM is a data quality practice; in regulated industries (finance, healthcare) it is a regulatory requirement.

[FACT] In a Medallion mesh, MDM uses specialized domains (centralized or decentralized) to manage mastered entities. Other domains consume these via Silver/Gold layers. The MDM flow:
1. Domain data extracted from source systems → Bronze → Silver → Gold (normal Medallion flow)
2. Domain events sent to MDM system in MDM input format (real-time)
3. MDM system consolidates using match/merge algorithms into a unified view
4. MDM can also generate events back to source systems (Dynamics, Power Apps) — **coexistence pattern**
5. Mastered data products distributed to all domains as high-quality shared entities

[FACT] The **coexistence pattern**: mastered data is pushed back to source systems via events or API calls, enriching them with the enterprise master record. This ensures old and new systems share the same master.

---

### Reference Data Management

[FACT] Unlike master data (which must be compared across domains), **reference data** can be managed within specific domains. Reference data = classification/categorization data (currency codes, country codes, product codes, geographic hierarchies). Reference data is also used in access management and authorization models.

[FACT] To ensure cross-domain consistency for reference data:
- Option A: Push enterprise reference data directly to domain Lakehouses for local use in data product development
- Option B: Central MDM service manages local → enterprise alignment; a catalog helps domains map their data product columns to central reference datasets
- Both options: when a domain's data product is published to Gold, a post-processing data quality lookup job identifies and resolves inconsistencies

---

### Scaling Conclusion: Control + Flexibility

[FACT] Effective Medallion scaling is not about rigid standardization — it is about managing the **tension between standardization and flexibility**:
- Standardize where it matters (security, data quality frameworks, transformation lineage, naming conventions)
- Allow flexibility where domains have unique needs (technology stack, layer count, data model details)
- Avoid technology proliferation: independent DQ frameworks → incomparable metrics; independent transformation frameworks → untrackable lineage

[FACT] The enterprise architecture department plays a crucial role: guides the org through federated model transition while balancing centralized governance with decentralized flexibility.

---

## Chapter 12: Medallion Governance and Security (Introduction)

### Governance Framework for Medallion Architectures

[FACT] Starting points for implementing data governance across a Medallion architecture:
1. **Source system side**: identify which source systems hold authentic data; assign application owners (responsible for collection) and data owners (responsible for quality + sharing decisions)
2. **Domain boundaries**: group applications supporting the same business objective into domains; appoint a domain owner per domain
3. **Platform instance alignment**: decide if multiple application teams share a single platform instance or each gets its own Medallion architecture
4. **Maturity-aligned granularity**: mature orgs → fine-grained, smaller domains; less mature → larger coarser domains

[FACT] Governance objectives by Medallion layer (Table 12-1 in the book):

| Layer | Governance objectives |
|---|---|
| **Bronze** | Report technical validation issues; label and classify data at ingestion; optionally encrypt PII; define access controls for raw data |
| **Silver** | Report functional data quality issues; **avoid cross-source data integration** (keep source-aligned for clear ownership); conduct regular audits; apply MDM; include security metadata; define what operational usage is allowed; sign-offs for operationally aligned data products |
| **Gold** | Focus on usage nuances; ensure uniformity across datasets; transform data model into an interface model optimized for intensive consumption; sign-offs for data products |

[ANALYSIS] The key governance rule for Silver — **avoid integrating data across sources** — is the reason source-aligned staging schemas exist as a distinct layer. Cross-source integration belongs only in the Gold layer, via dimension- and fact-loading procedures. This is the correct pattern per the book: Silver is source-aligned, Gold is integration-aligned.

[FACT] Bronze-layer governance access rule: **restrict raw data within its domain**. Only certified Silver datasets or Gold data products may be shared externally to other domains. This prevents downstream consumers from taking fragile dependencies on raw source formats.

[FACT] Consumer-side governance: manage the portfolio of all consuming use cases actively. Identify overlapping requirements before provisioning new platform instances. Set up consumer-providing domains for repeated common integration needs to avoid redundant work.

---

## Cross-Study Pattern: Governance Maturity

[ANALYSIS] Across all three case studies (AP Pension, Amadeus, ZEISS), the same governance maturity pattern emerges:

| Maturity stage | Characteristics | Failure mode |
|---|---|---|
| **Early** | Technology-first; policies on paper; no enforcement mechanism | Policies agreed upon but not enforced; technical debt accumulates |
| **Intermediate** | Community-based governance; guidelines issued centrally; teams self-implement | Inconsistent adoption; difficult to audit compliance |
| **Mature (Federated)** | Central governance body + domain-local bodies; policies automated (policy-as-code); data contracts enforce contracts technically | Requires significant tooling investment |

[FACT] ZEISS explicitly identified the failure of going technology-first: they had robust paper policies but lacked technical means to enforce them — data contracts are their future mechanism. Amadeus resolved this at Governance 3.0 by creating domain governance bodies with real enforcement authority. AP Pension resolved it via privacy-by-design (encryption at ingestion, Purview policy enforcement, per-customer keys).

---

## Source Reference

| Concept | Book location |
|---|---|
| ZEISS: Medallion layers, ingestion, SAP pre-processing | Ch. 10, pp. 541–546 |
| ZEISS: Gold/Power BI ambiguity, 3-catalog challenge | Ch. 10, pp. 546–549 |
| ZEISS: Global vs local data products, eVA→Fabric roles | Ch. 10, pp. 553–556 |
| ZEISS: governance challenges and recommendations | Ch. 10, pp. 554–556 |
| Data mesh federation spectrum (central → decentralized) | Ch. 11, pp. 560–562 |
| Medallion mesh concept (Franco Patano), Unity Catalog + Delta Share, OneLake | Ch. 11, pp. 562–565 |
| Number of Medallion architectures: 6 sizing drivers | Ch. 11, pp. 566–570 |
| Inner architecture: separate data product layer | Ch. 11, pp. 571–573 |
| 7 tailored Medallion patterns (simple provider → consumer provider) | Ch. 11, pp. 576–578 |
| Bronze conglomerate (shortcuts / physical / replication) | Ch. 11, pp. 579–582 |
| Silver variations, Gold sublayers, enterprise data curation | Ch. 11, pp. 582–584 |
| Domain aggregates (enterprise harmonization domains) | Ch. 11, pp. 584–587 |
| MDM in Medallion mesh (MDM flow, coexistence pattern) | Ch. 11, pp. 587–590 |
| Reference data management (push vs MDM vs catalog alignment) | Ch. 11, pp. 590–592 |
| Scaling conclusion (standardization vs flexibility) | Ch. 11, pp. 592–593 |
| Ch. 12 intro: governance framework overview | Ch. 12, pp. 594–595 |
| Governance by layer (source system → Bronze → Silver → Gold) | Ch. 12, pp. 595–600 |
| Governance objectives table (Table 12-1) | Ch. 12, pp. 598–599 |
