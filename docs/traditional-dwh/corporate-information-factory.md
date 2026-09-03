# Corporate Information Factory (CIF)

> *Primary source: Claudia Imhoff, Nicholas Galemmo, Jonathan Geiger — "Mastering Data Warehouse Design: Relational and Dimensional Techniques" (Wiley, 2003), Chapters 1–2*

## What it is

The Corporate Information Factory is an architecture framework that places the data warehouse inside a broader enterprise information ecosystem. It describes not just the database but the full pipeline of acquiring, storing, and delivering information — and the management layer that keeps it coherent.

The CIF was developed by Imhoff (building on Inmon's DWH concept) as a response to the "data mart chaos" problem: organizations that had built dozens of independent, inconsistent data marts and ended up with multiple conflicting versions of the same fact.

## The five data stores

```
Operational            Data               ODS         Data Marts      Operational
Systems           Warehouse                              (OLAP,        Marts
(OLTP/ERP/CRM)       (3NF)           (near real-time)   Exploration,  (tactical,
                                                         Mining,       short history)
    │                  ▲                  ▲             Analytical)       ▲
    │                  │                  │                ▲              │
    └──── Data Acquisition ──────────────┴────────────────┴──────────────┘
                                  Data Delivery
```

| Store | Purpose | Data currency | Design |
|---|---|---|---|
| **Operational Systems** | Run the business — OLTP, ERP, CRM, mainframe | Current only | Normalized for write throughput |
| **Data Warehouse** | Integrated historical enterprise record | Years of history | 3NF, nonredundant |
| **ODS** | Current-state integrated view | Near real-time | 3NF, updated in place (see ODS page) |
| **Data Marts** | Subject-specific analytics | Derived from DW | Dimensional (star/snowflake) |
| **Operational Marts** | Tactical analysis, short history | Rolling window | Dimensional, smaller scope |

## Four types of data marts

| Type | Purpose | Who uses it |
|---|---|---|
| **OLAP** | Multidimensional slice-and-dice, pre-aggregated cubes | Business analysts, managers |
| **Exploration warehouse** | Large-scale ad-hoc queries, detailed unpredictable access | Power analysts |
| **Data mining / statistical** | Algorithm and statistical analysis on historical data | Data scientists |
| **Customizable analytical applications** | Specialized applications with embedded analytics | Domain experts |

Each type has different latency, storage, and query patterns. The CIF framework keeps them all fed from a single integrated DW source so the numbers they produce remain consistent.

## Three process layers

**Data Acquisition** — ETL from operational systems into the DW and ODS. The critical integration point: data is reconciled, cleansed, and conformed here. Conflicts between source systems (homonyms, different codes for the same concept) are resolved once and stored correctly.

**Data Delivery** — moving data from the DW outward to data marts and end-user tools. This layer can range from batch mart loads to real-time query federation.

**Metadata Management** — a cross-cutting discipline covering three types of metadata:
- *Technical metadata*: table structures, column types, ETL job definitions
- *Business metadata*: definitions, ownership, lineage, data quality rules
- *Process metadata*: job run logs, record counts, timestamps

## Four characteristics of the data warehouse itself

Imhoff (following Inmon) defines the DW by four design properties:

| Property | Meaning |
|---|---|
| **Nonredundant** | Each fact is stored exactly once; no copies across subject areas |
| **Stable** | Historical data never changes; only new rows are added per time period |
| **Consistent** | Single definitions for shared concepts (Customer, Product, Date) across all marts |
| **Flexible** | Model can accommodate new questions without rebuilding existing marts |

The "flexible" property is the most practically important: a well-designed 3NF DW can answer questions that were never anticipated at design time, because the model reflects the business relationships rather than any specific report.

## Why 3NF for the DW?

This is the central design argument in Imhoff's book. The data mart layer (dimensional / star schema) is designed for end-user query ease. The DW layer below it is designed for **integration stability and flexibility** — goals that 3NF serves better than dimensional modeling:

- **Integration**: 3NF forces resolution of homonyms and synonyms at load time. Two source systems with different "customer" definitions must reconcile before data enters the DW.
- **Stability**: Adding a new question doesn't require restructuring the DW — the data is already there at the right granularity.
- **Data mart generation**: A 3NF DW can feed multiple different star schemas for different mart audiences, because it retains the full relational structure. A star schema DW cannot easily feed a different dimensional structure.

The trade-off: 3NF requires more joins for user queries, which is why the data mart layer (dimensional) exists — it pre-joins and denormalizes for the specific access patterns of each user group.

## CIF vs. data mart chaos

The problem the CIF was designed to solve:

| Data Mart Chaos | CIF |
|---|---|
| Each mart built independently from source systems | All marts built from the DW |
| Each mart defines its own version of "Customer" | One definition, stored in the DW, shared by all marts |
| Conflicting numbers between marts | Same numbers everywhere — reconciled at the DW layer |
| Mart schema changes require rebuilding source extracts | DW absorbs source changes; marts derived from DW are insulated |

## Other CIF components

**Information Feedback** — insights and analysis results flowing back into operational systems (e.g., a propensity score computed from the DW fed back into the CRM for sales reps to act on).

**Information Workshop** — the end-user access layer: reports, dashboards, OLAP tools, spreadsheet extracts, self-service analytics.

**Operations & Administration** — infrastructure, scheduling, monitoring, access management for the full environment.

## CIF in modern architectures

The CIF's ideas map directly into modern lakehouse terminology:

| CIF concept | Modern lakehouse equivalent |
|---|---|
| Data Warehouse (3NF) | Silver layer — integrated, cleansed, source-aligned |
| Data Marts (dimensional) | Gold layer — business-ready, aggregated for specific use cases |
| ODS | Low-latency Silver table with CDC streaming + MERGE INTO |
| Data Acquisition (ETL) | Ingestion pipelines (ADF, Databricks, dbt) |
| Metadata Management | Microsoft Purview, DataHub, dbt docs |

The key CIF insight — separate the integration layer (DWH / Silver) from the presentation layer (marts / Gold) — is the same principle behind the medallion architecture's Bronze → Silver → Gold separation.

## Further reading

The 8-step methodology for transforming the 3NF business data model into the DW system model is covered in the [DWH Data Model Development](dwh-relational-modeling.md) page.
