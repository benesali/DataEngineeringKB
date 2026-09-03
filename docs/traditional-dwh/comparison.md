# Inmon vs Kimball vs Data Vault

> *Sources: W.H. Inmon "Building the Data Warehouse" (4th ed.); Ralph Kimball & Margy Ross "The Data Warehouse Toolkit" (3rd ed.); Claudia Imhoff, Nicholas Galemmo, Jonathan Geiger "Mastering Data Warehouse Design" (2003), Ch. 13*

## At a glance

| | Inmon / CIF | Kimball / MD | Data Vault |
|---|---|---|---|
| **Philosophy** | Top-down, enterprise-first | Bottom-up, business-unit-first | Hybrid, audit-first |
| **Core model** | 3NF at DW; star at mart | Star schema throughout | Hubs, Links, Satellites |
| **Starting point** | Enterprise data model | A single business process | Business keys |
| **Iteration speed** | Slower first project; faster subsequent | Fast first delivery; retrofitting adds cost later | Medium |
| **Query performance** | Poor on DW directly; good via marts | Good (few joins in star) | Poor without mart layer |
| **Schema flexibility** | High (3NF absorbs new questions) | Medium (new dimensions may require new stars) | Very high (satellites absorb change) |
| **Auditability** | Medium | Low | Very high (every load timestamped) |
| **Best for** | Large enterprises, regulated industries, future unknown needs | Analyst-facing marts, self-service, time-to-value | Multi-source integration, agile DWH, compliance |
| **Mart layer needed?** | Yes (derived from 3NF DWH) | IS the mart layer | Yes (typically Kimball-style Gold) |
| **Physical central DWH?** | Yes — a separate, distinct physical store | No — the staging area + atomic data marts IS the warehouse | No — vault is the integration layer; marts are separate |

---

## Enterprise scope vs. business unit scope — the real difference

The deepest difference between Inmon/CIF and Kimball/MD is not the data model but the **scope priority**:

- **CIF** places a higher priority on **enterprise scope**: the DW model is designed for the entire organization before any mart is built. IT pushes data from operational systems to where it's needed.
- **MD** places a higher priority on **business unit scope**: the star schema is designed optimally for a specific department or user group. Business units pull the data they need.

Neither architecture ignores the other scope entirely. Kimball explicitly requires **conformed dimensions** to achieve cross-mart consistency — which is an enterprise concern. CIF iterates incrementally, never requiring the full enterprise model before delivering value.

> "The MD architectural approach subordinates data management to business requirements because its reason for being is to satisfy a business unit within the enterprise. On the other hand, the CIF architectural approach manages data to the subordination of the business requirements because its reason for being is to serve the entire enterprise." — Imhoff et al., Ch. 13

---

## CIF vs. MD: six dimensions compared

### 1. Data flow — push vs. pull

| CIF (Inmon) | MD (Kimball) |
|---|---|
| **Push**: IT integrates enterprise data and makes it available; users are served from the DW | **Pull**: business units pull the data they need from operational systems into star schemas |
| First CIF project: higher overhead (enterprise modelling) | First MD project: lower overhead (single mart scope) |
| Subsequent CIF projects: faster — existing subject areas already modelled | Subsequent MD projects: nontrivial changes to conformed dimensions; retrofitting accumulates cost |

### 2. Volatility — stability under business change

| CIF (Inmon) | MD (Kimball) |
|---|---|
| The DW model is **process-free**: not designed for any specific query. New questions don't require rebuilding. | The star schema is optimized for known queries. When business processes change, the fact table may need to be **reshuffled or rebuilt**. |
| Atomic fact tables with billions of rows can be rebuilt from the DW — the DW itself remains stable | A large atomic fact table (billions of rows) is too expensive to rebuild; a new (partially redundant) star schema is often created instead |

### 3. Flexibility — what types of analysis are supported

| CIF (Inmon) | MD (Kimball) |
|---|---|
| **Supports any analysis type**: multidimensional (star schema marts), data mining, statistical analysis, exploration, memory-resident BI, bitmap index queries | **Multidimensional by design**: all components (except staging) must be star schema; data mining and statistical analysis on star schemas is limited because only *known* relationships are modelled |
| "Everything is a nail" problem avoided | "Everything is a hammer" — if you design everything as dimensional, all you can do is dimensional analysis |

### 4. Complexity — starting simple vs. staying simple

| CIF (Inmon) | MD (Kimball) |
|---|---|
| Starts with a **complex, enterprise data model** — hard upfront; data marts derived from it are simpler | Starts with a **simple, business-unit model** — easy upfront; enterprise integration through conformed dimensions adds complexity over time |
| Risk of over-engineering on day 1 | Risk of inconsistency and retrofit cost by day 100 |

### 5. Functionality — what can you do with it

| CIF (Inmon) | MD (Kimball) |
|---|---|
| Supports multidimensional marts (OLAP), exploration warehouse (ad-hoc detail), data mining warehouse, operational marts | Optimized for relational multidimensional processing — slice, dice, drill-up, drill-down — all dimensions are symmetric |
| Broader set of BI technologies supported from a single foundation | Excellent multidimensional performance; data mining on star schemas is constrained |

### 6. Ongoing maintenance — pay now or pay more later

CIF incurs higher upfront cost on the first project to establish the enterprise foundation. But with each subsequent iteration, building new marts is cheaper because the DW already holds the integrated data — only the delivery process changes.

MD delivers quickly per mart but accumulates technical debt in the form of:
- Conformed dimension governance overhead (every new dimension must achieve enterprise consensus)
- Potential for conflicting definitions between marts built at different times
- Costly retrofits when an existing mart's design must change to accommodate new business questions

> "Just as a sound foundation for a house takes forethought and is absolutely necessary for the longevity of the structure, regardless of the changes that occur over the years, a well-designed data warehouse data model will serve your enterprise for the long haul." — Imhoff et al., Ch. 13

---

## The practical consensus

The authors of *Mastering Data Warehouse Design* (all experienced practitioners of both approaches) conclude:

> "A combination of the data-modeling techniques found in the two architectural approaches works best — ERD or normalization techniques for the data warehouse and the star schema data model for multidimensional data marts."

This **is** the CIF in practice. A CIF with only a 3NF warehouse and no dimensional marts is useless — users can't query it. A purely dimensional environment (no 3NF integration layer) risks data mart chaos and limits non-multidimensional analysis.

---

## How they combine in practice

These approaches are not mutually exclusive:

```
Sources
  │
  ▼
Raw layer (Bronze — land exactly as received)
  │
  ▼
Integration layer (Silver — 3NF/CIF or Data Vault — integrated, historical, non-volatile)
  │
  ▼
Mart layer (Gold — Kimball star schemas — optimized for end-user queries)
```

In a **Medallion architecture**:
- Bronze = raw / staging
- Silver = Data Vault or 3NF integration (CIF Data Warehouse)
- Gold = Kimball dimensional marts

---

## Choosing between them

**Pick Inmon/CIF if:** you need one authoritative model across many departments, must support analysis types beyond multidimensional (data mining, statistics), or work in a regulated industry where a full audit trail from source to result is required.

**Pick Kimball/MD if:** you need to deliver analytical value quickly to a specific business team, your data integration needs are manageable, and your primary output is OLAP / slice-and-dice dashboards.

**Pick Data Vault if:** you have many heterogeneous source systems, frequent schema changes, strict auditability, and the team has the technical maturity to implement it.

**Use all three in layers if:** you are building a modern lakehouse at enterprise scale — Bronze ingests, Silver integrates (3NF or Vault), Gold serves (Kimball dimensional).

---

## Related pages

- [Inmon — Enterprise Data Warehouse (3NF)](inmon.md)
- [Kimball — Dimensional Modeling](kimball.md)
- [Corporate Information Factory (CIF)](corporate-information-factory.md)
- [Data Vault](data-vault.md)
- [OLAP — Multidimensional Analysis](../data-modeling/olap.md)
