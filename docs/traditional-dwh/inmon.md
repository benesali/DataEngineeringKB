# Inmon — Enterprise Data Warehouse (3NF)

> *Source: W.H. Inmon, "Building the Data Warehouse" (1st ed. 1992; 3rd ed. 2002; 4th ed. 2005)*

Bill Inmon coined the term "data warehouse" in the late 1980s. His approach is called **top-down**: design the enterprise data model first, build the central warehouse, then derive data marts from it.

---

## Why data warehouses exist — the spider web problem

Before data warehouses, organizations relied on a naturally evolving architecture: every team that needed data built their own **extract program** to pull it from operational systems. Extract programs multiplied into extracts of extracts. Large companies ran 45,000 extracts per day. This created the **spider web**.

The spider web produced two predictable failures:

**1. Data credibility crisis.** Two departments deliver reports to management — one says activity is up 10%, the other says it's down 15%. Management cannot trust either. Five reasons this is predictable:

- No time basis: Dept A extracted on Sunday, Dept B on Wednesday — the same database read at different times
- Algorithmic differential: Dept A analyzed "old accounts," Dept B analyzed "large accounts" — different criteria, different results
- Multiple extraction levels: each extraction level compounds the discrepancy; 8–9 levels was common
- External data: analysts brought in Wall Street Journal or Business Week data without recording its origin — it became untraceable
- No common source: Dept A started from file XYZ, Dept B from database ABC — no shared foundation

**2. Productivity collapse.** A single corporate report spanning legacy systems required 9–12 months to locate the data and 15–24 months to extract and compile it. And the work done for the first report did not reduce the cost of the second report. Organizations stopped creating reports. End users accepted: "I know the information is in my corporation, but I just can't get to it."

The **architected environment** — with a data warehouse at the center — was Inmon's solution.

---

## The four properties of a data warehouse

Inmon's definition:

> "A subject-oriented, integrated, nonvolatile, and time-variant collection of data in support of management decisions."

| Property | Meaning |
|---|---|
| **Subject-oriented** | Organized around business subjects (Customer, Product, Order) — not source applications or functional silos |
| **Integrated** | Data from many source systems unified into consistent formats, encodings, and definitions |
| **Non-volatile** | Data is never updated at the individual record level — historical records stay intact |
| **Time-variant** | Every record carries a time element; the warehouse holds a historical archive |

---

## Primitive data vs. derived data

One of Inmon's most fundamental observations: there are two fundamentally different kinds of data, and they cannot coexist in the same environment.

| Primitive data (operational) | Derived data (DSS / analytical) |
|---|---|
| Application-oriented | Subject-oriented |
| Detailed | Summarized or otherwise calculated |
| Accurate as of the moment of access | Represents values over time, snapshots |
| Serves the clerical community | Serves the managerial community |
| Can be updated | Is not updated |
| Run repetitively | Run heuristically |
| Requirements understood in advance | Requirements discovered through use |
| Performance sensitive | Performance relaxed |
| Accessed one record at a time | Accessed in sets — thousands to millions of rows |
| High probability of access | Low, modest probability of access |

> "It is a wonder that the information processing community ever thought that both primitive and derived data would fit and peacefully coexist in a single database."

The **operational environment** holds primitive data. The **data warehouse** holds derived (or historically organized primitive) data. They must be separated — and this separation is the entire point of the DWH architecture.

---

## The four architecture levels

The architected environment has four levels, each serving a different purpose:

```
  Individual (PC-based — heuristic, temporary, EIS)
      ▲
  Departmental / Data Mart (parochial, derived, shaped for one department)
      ▲
  Atomic / Data Warehouse (integrated, historical, granular — the core)
      ▲
  Operational (current-value, application-oriented, OLTP)
```

The **J Jones example**: a bank customer J Jones exists at every level, but differently:

- **Operational**: J Jones, 123 Main Street, Credit-AA — current state, updatable at any moment
- **Data warehouse**: J Jones 1986–1987 at 456 High St (Credit-B), 1987–1989 at 456 High St (Credit-A), 1989–present at 123 Main St (Credit-AA) — non-overlapping historical rows
- **Departmental**: Customers since 1982 with balances >$5,000, Jan-4101, Feb-4209... — aggregated, shaped for one department's needs
- **Individual**: "What trends are there for the customers we are analyzing?" — temporary, exploratory

> "Although it is not apparent at first glance, there is very little redundancy of data across the architected environment. It is the spider web environment that generates gross amounts of data redundancy."

---

## Integration — the most important property

Operational applications were built independently. No application designer ever considered that their data would need to integrate with other applications. The result:

| Attribute | App A | App B | App C | App D | DWH standard |
|---|---|---|---|---|---|
| Gender | m/f | 1/0 | x/y | male/female | m/f |
| Pipeline | cm | inches | mcf | yds | cm |
| Balance field | balance | bal | currbal | balcurr | bal |
| Key structure | char(10) | dec fixed(9,2) | pic '9999999' | char(12) | char(12) |

As data passes into the warehouse, every inconsistency is resolved. Inmon identifies this as the most important property — without it, the DWH cannot provide a corporate view of data.

---

## Granularity — the single most important design decision

**Granularity** is the level of detail at which data is stored in the DWH. It is the most important design decision because it determines both how much data you must store and what questions you can answer.

The **phone call example** makes the trade-off concrete:

**Fine granularity** — one row per phone call:
```
activityrec: date, time, to_whom, op_assisted, call_completed,
             long_distance, cellular, special_rate, ...
→ 200 records per customer per month = 40,000 bytes
```

**Coarse granularity** — one row per customer per month:
```
activityrec: month, cumcalls, avglength, cumlongdistance, cuminterrupted
→ 1 record per customer per month = 200 bytes
```

A **200× reduction** in storage. But the trade-off:

> "Did Cass Squire call his girlfriend in Boston last week?"

- With fine granularity: answerable — search 175,000,000 records, 45,000,000 I/Os
- With coarse granularity: **not answerable** — the detail no longer exists

For collective queries ("On average, how many long-distance calls did people from Washington make last month?") coarse granularity is 100× more efficient. And collective queries are far more common in DSS.

> "The level of granularity has a profound effect both on what questions can be answered and on what resources are required to answer a question."

There is no single correct answer — it depends on the business. But coarser data that cannot answer an important question is worse than finer data that is expensive to query.

---

## Dual levels of granularity

The practical solution for most organizations: **store two levels simultaneously**.

```
Lightly summarized  →  95%+ of DSS queries (compact, fast, disk)
True archival detail  →  <5% of queries (expensive, bulk storage)
```

The telephone company example:
- Operational: 30 days of per-call detail
- DWH lightly summarized: monthly customer summaries (10 years, on disk)
- DWH true archival: full per-call detail (indefinitely, on bulk storage)

> "By creating two levels of granularity in the data warehouse, the DSS architect has killed two birds with one stone — most DSS processing is performed against the lightly summarized data, where the data is compact and efficiently accessed. When some greater level of detail is required — 5% of the time or less — there is the true archival level."

The **banking DDL** at each level:

*Operational (60 days):*
```
account, activity_date
  amount, teller, location, to_whom
  identification, account_balance, instrument_number
```

*DWH — lightly summarized (10 years, monthly):*
```
account, month
  number_of_transactions, withdrawals, deposits
  beg_balance, end_balance, account_high, account_low
  average_account_balance
```

*DWH — true archival (bulk storage, indefinite):*
```
account, activity_date
  amount, to_whom, identification
  account_balance, instrument_number
  (fields not needed for legal/informational purposes are purged)
```

Customer demographic history uses a **continuous file** — non-overlapping from/to dated rows:
```
customer_id, from_date, to_date
  name, address, credit_rating, monthly_income, occupation
```

---

## Sizing the DWH — raw estimates and row count thresholds

Before building, estimate size: for each table, multiply estimated row size by projected row count at the 1-year and 5-year horizons, then add index space.

> "Estimates projecting the size of the data warehouse almost always are low. Growth rate is usually faster than the projection."

Row count thresholds drive design decisions:

| 1-year row count | Design and overflow decision |
|---|---|
| < 100,000 | Any design; no overflow needed |
| < 1,000,000 | Design carefully; overflow unlikely |
| < 10,000,000 | Design carefully; some data may go to overflow |
| < 100,000,000 | Very careful design; majority on disk, some in overflow |
| > 1,000,000,000 | Majority in overflow; granularity choices are critical |

---

## Partitioning

Partitioning breaks large tables into physically separate, independently managed units. In a DWH, partitioning is not optional — it is mandatory at large volumes.

**Benefits:**
- Restructure, reindex, or archive one partition without touching others
- Recovery of a failed partition is isolated
- Old partitions can migrate to cheaper storage tiers

**Criteria:** almost always by **date** (year or month), sometimes also by line of business or geography.

**Acid test:** "Can an index be added to a partition with no discernible interruption to other operations?" If yes — partition size is appropriate. If no — partition more finely.

**Application-level vs system-level partitioning:** Inmon recommends application-level. The DBMS enforces a single schema definition, but a 10-year DWH accumulates data across many schema versions. Application-level partitioning allows 2000's definition and 2010's definition to coexist as separate physical units.

---

## Loading the DWH — five ETL detection techniques

The hardest part of DWH loading is **incremental refresh**: how do you know which operational records changed since the last load without scanning everything?

Five techniques, in order of desirability:

1. **Timestamp on the source record**: the application stamps the last-update time. Scan reads only records changed since the last DWH load. Efficient — but few legacy applications were ever timestamped.
2. **Delta file**: a separate file containing only the changes. Very efficient — but rare.
3. **Log / audit file**: the DBMS transaction log contains the same data. Challenges: operations protects log files for recovery; internal format is system-oriented; log may contain unwanted data.
4. **Modify application code**: embed extract logic into the source application. Never popular — old application code is fragile.
5. **Before/after image comparison**: take a snapshot, later take another, compare them. "A hideous option, mentioned primarily to convince people there must be a better way." Last resort only.

---

## Data modeling — three levels

Inmon prescribes a rigorous top-down design with three levels:

1. **High-level model (ERD)**: entities and relationships only — no attributes. Defines the major subjects (Customer, Account, Product) and how they relate. Covers the entire enterprise.
2. **Mid-level model (DIS — Data Item Set)**: for each entity, adds actual data attributes. Four constructs: primary grouping (exists once per subject), secondary grouping (repeating attributes), connectors (foreign-key relationships between subjects), "type of" data (subtypes).
3. **Physical model**: adds storage details — data types, lengths, indexes, partitioning.

> "If the corporate data model is Adam, the operational data model is Cain, and the data warehouse data model is Abel — all from the same lineage, but each different."

### Stability analysis

Before finalizing physical design, group attributes by rate of change. A manufacturing `Part` table example:

| Seldom changes | Sometimes changes | Frequently changes |
|---|---|---|
| description | safety stock | qty on hand |
| order unit | primary supplier | last order date |
| lead time | expediter | last order amount |

Three separate physical tables result — each optimized for its actual change frequency. Inmon calls this **stability analysis**.

---

## The 3NF data model — why it matters

The DWH itself is in **Third Normal Form (3NF)**. Data marts derived from it are denormalized star schemas.

**Why 3NF at the warehouse level?**
- No redundancy — one place to correct an error, one definition of every entity across the enterprise
- Stable under change — adding a column doesn't require restructuring the model
- Flexible — can answer future unknown questions because all granular detail is retained

**Why not star schema at the warehouse level?**
- Star schemas are optimized for a specific set of known query patterns. The DWH will be asked questions nobody anticipated when it was built. A 3NF model answers any question; a star schema efficiently answers only questions that fit its pre-defined dimensional structure.

> "The relational model, because of its flexibility, is the correct choice for the data warehouse. The multidimensional model, because of its performance, is the correct choice for the data mart."

---

## Snapshots and profile records

### Snapshots

Every record in the DWH is a **snapshot** — the state of something at one moment in time. Snapshot components:

- **Key** (composite, includes a time element)
- **Unit of time** (when the event occurred)
- **Primary data** (attributes of the subject)
- **Secondary data** (optional — incidental context captured at the moment, called an *artifact*)

Example: a loan approval snapshot captures the interest rate prevailing at the moment of approval as secondary data. Even if that rate changes tomorrow, the historical loan record carries the rate that was true when it was approved.

Triggers: activity-generated (a sale, a claim, a phone call) or time-generated (end of day, end of month).

### Profile records

When detail volume is very large, individual activity records are aggregated into **profile records** — a single record representing many events:

A phone company aggregates each customer's 200 calls per month into one monthly record:
- Number of calls, average length, number long-distance, number operator-assisted, number incomplete

Result: **2–3 orders of magnitude** reduction in data volume. Profile records are pre-computed summaries that make analyst access fast. The raw detail is retained on bulk storage for the rare case it's needed.

---

## The operational window of opportunity

Every industry has a window during which operational source systems hold current data before purging it. Outside that window, only the DWH can answer historical questions.

| Industry | Operational window |
|---|---|
| Retail banking account activity | 30 days |
| Telephone customer usage | 30–60 days |
| Retailing SKU activity | 1–14 days |
| Vendor activity | 1 week to 1 year |
| Insurance | 2–3 years |
| Loans | 2–5 years |
| Bank trust processing | 2–5 years |

---

## The wrinkle of time

At least **24 hours** must pass between when a change is known to the operational environment and when it is reflected in the DWH. This "wrinkle of time":
- Allows data to settle and be corrected operationally before entering the warehouse
- Prevents the temptation to do operational processing in the DWH
- Keeps the two environments cleanly separated

If the wrinkle is reduced to 4 hours, the boundary between environments blurs. Both environments suffer.

---

## The Operational Data Store (ODS)

The ODS is a complementary structure — not a replacement for the DWH. It answers "What is the state of customer X right now?" while the DWH answers "What was the trend in customer behavior over the last 3 years?"

| | ODS | Data Warehouse |
|---|---|---|
| Data currency | Current (near real-time) | Historical (batch-loaded) |
| Updates | Yes — records updated as state changes | No — append-only |
| Query type | Single customer, current state | Trends, aggregations, history |
| Response time | Sub-second to seconds | Minutes to hours |
| Users | Customer service, operations | Analysts, managers, data scientists |

### Four ODS classes

| Class | Latency | Method |
|---|---|---|
| I | Seconds | Direct feeds from OLTP |
| II | Minutes to hours | Periodic refresh |
| III | Overnight batch | Same batch window as DWH |
| IV | DWH-derived | Built from the DWH, not from OLTP |

A common DWH loading pattern: **time slicing the ODS** — take a snapshot of the ODS at end-of-day and feed it to the DWH. This avoids competing with OLTP load times.

---

## Storage tiers for large warehouses

As data ages, its access probability drops. Tiered storage matches cost to usage:

| Tier | Speed | Cost | What lives here |
|---|---|---|---|
| Primary disk (SSD / HDD) | Milliseconds | High | Recent, active data |
| Near-line (optical, slower disk) | Seconds to minutes | Medium | Older data, occasional queries |
| Archival (robotic tape silos) | Minutes to hours | Very low | Regulatory retention, rarely queried |

**CMSM (Continuous Monitoring and Storage Management)** automates the migration:
- **Activity monitor**: tracks which data is accessed, when, by whom
- **Cross-media storage manager**: moves data between tiers based on activity — older unused data moves to near-line; data accessed for an audit moves back to disk

---

## The four user archetypes

Different users need different things from the DWH. Designing for only one archetype underserves the others.

| Archetype | How they work | What they need |
|---|---|---|
| **Farmer** | Regular, predictable queries — same reports every week | Pre-aggregated summaries, data marts, fast response |
| **Explorer** | Ad-hoc, irregular — probing for unknown patterns | Fine-grained detail, full 3NF DWH, drill-down capability |
| **Miner** | Statistical analysis, predictive modeling | Large volumes of granular history, living sample database |
| **Tourist** | Occasional visitor, simple needs | Pre-built reports, dashboards, semantic layer |

Farmers and Tourists are well-served by data marts. Explorers and Miners need the granular DWH directly. This is Inmon's architectural justification for keeping both detailed and summarized data: each archetype requires one or the other.

---

## The independent data mart trap

Inmon's sharpest warning: **never build data marts directly from operational sources**, without a DWH foundation.

Each mart makes its own transformation decisions. When two marts report different "total customers," neither is wrong by its own logic — but the organization cannot trust either. This recreates the spider web at the mart level.

> "You build one mart, it delivers value, you build another, the numbers don't match the first, and you've created the credibility problem you were trying to solve."

The solution: always derive data marts from a common DWH foundation.

---

## The Corporate Information Factory (CIF)

Inmon situates the DWH inside a larger architecture:

```
Source Systems (OLTP)
      │
      ▼
  Staging / ODS  ◄──────────────────┐
      │                              │
      ▼                              │
  Enterprise DWH (3NF) ─────────────┘
      │
      ├──► Data Marts (departmental DSS)
      ├──► Exploration Warehouse (data mining, statistics)
      └──► EIS / BI Reporting
```

> "The data warehouse is an architecture, not a technology. Just as Santa Fe's architecture is defined by adobe bricks — but adobe bricks alone don't make Santa Fe — data warehousing requires specific technology but is not itself that technology."

---

## When Inmon fits

- Large enterprises needing a single authoritative model across many departments
- Regulatory environments (banking, healthcare, insurance) — SOX, HIPAA, Basel II require full audit trail
- When data marts must agree with each other (all derived from one DWH guarantees consistency)
- When future analytical needs are unknown — 3NF flexibility over star schema efficiency
- When both current operational queries (ODS) and historical analytical queries (DWH) are needed

---

## Book chapter reference

| Chapter | What to find there |
|---|---|
| 1 | DSS evolution, spider web, primitive vs derived data, four architecture levels, CLDS |
| 2 | Four DW properties, Day 1→n, granularity, dual levels, living sample, partitioning, operational window |
| 3 | 3-level data model, stability analysis, ETL techniques, snapshots, metadata, profile records, wrinkle of time |
| 4 | Row count thresholds, dual granularity across banking/manufacturing/insurance, overflow storage |
| 5 | Technology requirements for the DWH |
| 12 | Near-line/archival storage tiers, CMSM, usage monitoring |
| 13 | 3NF vs star schema, future unknown needs, independent data mart trap |
| 16 | ODS four classes, ODS vs DWH, time slicing |
| 18 | Farmer / Explorer / Miner / Tourist archetypes |
