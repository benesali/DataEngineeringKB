# Kimball — Dimensional Modeling

> *Primary source: Ralph Kimball & Margy Ross, "The Data Warehouse Toolkit, Third Edition" (Wiley, 2013)*

## Who and when

Ralph Kimball developed his dimensional modeling methodology through the 1990s–2000s. The original *Data Warehouse Toolkit* (1996) was the first practical guide to dimensional modeling. The Third Edition (2013) with Margy Ross is the definitive reference at 601 pages.

Kimball's approach is **bottom-up** — start with a specific business process and model it well, then expand.

---

## Goals of DW/BI (Ch 1)

Kimball opens with the requirements that drove data warehousing. These business management statements still hold after 30+ years:

> "We collect tons of data, but we can't access it."  
> "We need to slice and dice the data every which way."  
> "We spend entire meetings arguing about who has the right numbers rather than making decisions."

From these, Kimball derives the DW/BI system requirements:

| Requirement | Meaning |
|---|---|
| **Make information accessible** | Understandable to business users, not just developers. Simple and fast. |
| **Present information consistently** | Same metric name = same definition everywhere. Credible data. |
| **Adapt to change** | New questions or new data should not break existing applications. |
| **Present information in a timely way** | From days to hours to minutes, depending on use case. |
| **Be a secure bastion** | The warehouse contains the organization's crown jewels. |
| **Serve as the authoritative foundation for decision making** | A decision support system — the original name. |
| **Be accepted by the business community** | An elegant system nobody uses has failed. |

> "You need to bring a spectrum of skills — behave like a hybrid DBA/MBA."

### The publishing metaphor

Kimball compares the DW/BI manager to a magazine editor-in-chief: understand your readers (users), deliver compelling and accurate content (data), sustain the publication (system). The technology is a means to an end — it should not appear in your top job responsibilities.

---

## Why NOT normalized models for analytics (Ch 1)

Normalized 3NF structures (like Inmon's DWH model) are essential for operational processing — updates touch one place. But for analytical queries:

> "Users can't understand, navigate, or remember normalized models that resemble a map of the Los Angeles freeway system."

The complexity of unpredictable BI queries overwhelms relational database optimizers on normalized schemas, resulting in disastrous query performance.

> A dimensional model contains the same information as a normalized model, but packages the data in a format that delivers **user understandability, query performance, and resilience to change**.

---

## Core vocabulary

### Grain

The grain is **what one row in the fact table represents**. It is the single most important design decision — declared before choosing dimensions or facts.

Kimball: "One of the core tenets of dimensional modeling is that all the measurement rows in a fact table must be at the same grain."

Examples:
- "One row per product scanned at checkout" (most granular — Kimball recommends this)
- "One row per order line item"
- "One row per daily inventory position per product per store"

### Fact table

Stores the performance measurements from business process events. Each row is at the declared grain.

- **Additive measures**: can be summed across all dimensions (SalesAmount, Quantity)
- **Semi-additive measures**: can be summed across some dimensions but not all — typically not across time (account balance, inventory level)
- **Non-additive measures**: cannot be summed at all (ratios, percentages, unit prices)

### Dimension table

Describes the context of facts — who, what, where, when, how, why.

- Low row count but many columns (wide and flat)
- Denormalized — attributes are NOT normalized into sub-tables
- Contains both codes and their text descriptions (both `ProductCode` and `ProductDescription` in one row)
- Business users navigate and filter by dimension attributes

---

## The Four-Step Dimensional Design Process (Ch 2, Ch 3)

Every dimensional model starts with four steps, in this exact order:

### Step 1: Select the business process

A **business process** is a measured activity in the organization — retail sales, inventory levels, procurement, orders. One business process = one (or a small family of) dimensional model(s).

The grain and dimensions flow naturally from the business process. Trying to model across processes in one fact table is the most common mistake.

### Step 2: Declare the grain

Commit to exactly what one fact row represents **before** identifying dimensions or facts.

> "The grain must be declared before choosing dimensions or facts. The grain is the fundamental commitment of the dimensional model."

Grain drives everything else: if you declare "one row per order line," then every dimension must apply at that level, and every measure must be valid at that level.

**Always go as granular as possible.** Granular data can always be aggregated up; summary data cannot be disaggregated down.

### Step 3: Identify the dimensions

With the grain declared, ask: "How does the business describe the events resulting from this process?"

Dimensions answer the who, what, where, when, how, why of each measurement event. Standard dimensions: Date, Product, Customer, Employee, Location.

### Step 4: Identify the facts

Ask: "What is the process measuring?"

Facts are the numeric measurements that result from the business process event: quantity sold, revenue, cost, count. Every fact must be true at the grain declared in Step 2.

---

## Retail Sales case study (Ch 3)

Kimball's canonical worked example:

**Process**: Retail store point-of-sale transactions  
**Grain**: One row per product scanned at checkout  
**Dimensions**: Date, Product, Store, Promotion, Cashier (optional), Payment Method (optional)  
**Facts**: Quantity Sold, Sales Amount (extended), Cost Amount, Gross Profit

```
Date Dimension
      │
Product Dimension ──► FactRetailSales ◄── Store Dimension
                              │
                       Promotion Dimension
```

### Date dimension

The Date dimension is pre-populated for years — one row per calendar day — and is the richest, most reused dimension. It contains every conceivable calendar attribute:

- `DateKey` (INT YYYYMMDD — a smart key)
- `FullDateDescription` ('January 15, 2024')
- `DayOfWeekNumber`, `DayOfWeekName`
- `MonthNumber`, `MonthName`, `MonthYearLabel`
- `CalendarQuarterNumber`, `CalendarYearNumber`
- `FiscalWeekNumber`, `FiscalMonthNumber`, `FiscalYearNumber`
- `IsWeekend` (flag), `IsHoliday` (flag)

The Date dimension is always pre-built and reused across all fact tables — the canonical conformed dimension.

### Product dimension

Denormalized: contains product, sub-category, category, department, brand — all in one row. **Do not normalize into sub-tables** (that creates snowflaking — more joins, slower queries, harder for users).

```
Product Dimension (one row per product):
ProductKey (PK), ProductBK, ProductDescription,
SubcategoryName, CategoryName, DepartmentName,
BrandName, PackageType, PackageSize, DietType, IsLowFat, IsLowSodium, ...
```

---

## Conformed dimensions — the bus architecture (Ch 2, Ch 4)

**Conformed dimensions** are the foundation of Kimball's enterprise integration strategy.

A conformed dimension is one that means exactly the same thing across multiple fact tables and data marts. DimDate used in FactSales, FactInventory, and FactMarketing is the same DimDate — same keys, same definitions.

Without conformed dimensions, you cannot **drill across** fact tables: a Sales and a Returns dashboard cannot be linked by Customer if the two fact tables use different Customer dimensions.

### Enterprise Data Warehouse Bus Matrix

The Bus Matrix maps business processes (rows) against dimensions (columns). A tick means this dimension applies to this process. It is the planning document for a Kimball enterprise DWH.

| | Date | Customer | Product | Store | Promotion |
|---|---|---|---|---|---|
| Retail Sales | ✓ | ✓ | ✓ | ✓ | ✓ |
| Inventory | ✓ |   | ✓ | ✓ |   |
| Procurement | ✓ |   | ✓ |   |   |
| Orders | ✓ | ✓ | ✓ |   |   |

Each row is a separate fact table. Columns that appear in multiple rows are conformed dimensions.

---

## Slowly Changing Dimensions (SCD) — all 7 types (Ch 5)

See the dedicated [SCD page](../data-modeling/scd.md) for full detail. Quick summary:

| Type | Name | What happens on change | History? |
|---|---|---|---|
| 0 | Retain Original | Ignore the change | None |
| 1 | Overwrite | Update the row | None |
| 2 | Add New Row | New row with new surrogate key | Full |
| 3 | Add New Attribute | New `Previous_X` column | One step |
| 4 | Add Mini-Dimension | Move rapidly changing attributes to a separate small dimension | Full (in mini-dim) |
| 5 | Mini-Dim + Type 1 Outrigger | Type 4 + current mini-dim profile on main dim | Full |
| 6 | Type 1 on Type 2 | Add Type 1 current value columns to a Type 2 dimension | Full + current |
| 7 | Dual Type 1 + Type 2 | Two surrogate keys on fact — one current, one historical | Full + current lookup |

---

## Fact table types in depth (Ch 4)

### Transaction fact table

One row per discrete business event. Most common.

- `FactRetailSales` — one row per product scanned
- `FactOrderLines` — one row per order line
- Grows continuously; rows never updated

### Periodic snapshot fact table

One row per period per entity — regardless of whether anything happened.

- `FactDailyInventory` — one row per product per store per day
- Captures level/balance measures (inventory on hand, account balance)
- **Semi-additive facts**: summing across products and stores is valid; summing across dates is NOT — use average instead

Kimball's checking account example: you cannot add Monday's $50 balance to Tuesday's $50 balance to Wednesday's $100 balance and declare $200 is your "total balance" — that is meaningless. Average ($80) is the correct aggregation across time.

### Accumulating snapshot fact table

One row per instance of a multi-stage business process. Updated as stages complete.

- `FactOrderFulfillment` — one row per order with columns for each stage date
- Multiple date foreign keys (OrderDate, PickDate, ShipDate, DeliveryDate, ReturnDate)
- Rows are **updated** (not appended) as stages complete
- Enables lag/duration calculations between stages

```
FactOrderFulfillment:
  OrderKey          | ProductKey    | CustomerKey
  DateOrderedKey    | DatePickedKey | DateShippedKey | DateDeliveredKey
  QuantityOrdered   | QuantityShipped
  OrderToPickLag    | PickToShipLag | ShipToDeliverLag
```

---

## Kimball's top 10 dimensional modeling mistakes (Ch 16)

(In reverse order of severity — #1 is the worst)

| Rank | Mistake |
|---|---|
| 10 | Place text attributes in the fact table |
| 9 | Limit verbose descriptors to save space |
| 8 | Split hierarchies into multiple dimensions |
| 7 | Ignore the need to track dimension attribute changes |
| 6 | Solve all performance problems with more hardware |
| 5 | Use operational keys to join dimensions and facts |
| 4 | Neglect to declare and comply with the fact grain |
| 3 | Use a report to design the dimensional model |
| 2 | Expect users to query normalized atomic data |
| **1** | **Fail to conform facts and dimensions** |

---

## The Kimball Lifecycle (Ch 17)

Kimball's full project lifecycle has three parallel tracks:

```
Program/Project Planning ──────────────────────────────────────────────────►
Business Requirements Definition

Technology Track:     Technical Architecture ──► Product Selection
Data Track:           Dimensional Model ──► Physical Design ──► ETL Development
BI Applications Track: BI Specification ──► BI Development

                                    └─► Deployment ──► Maintenance & Growth
```

Key principle: **requirements-driven**. Business requirements definition feeds all three tracks. You don't design the data model until you know what the business needs.

---

## Kimball's DW/BI architecture (Ch 1)

```
Operational       Extract,           Presentation Area        Business
Source Systems ──► Transform, ──────► (Star Schemas /  ──────► Intelligence
                  Load (ETL)          OLAP Cubes)              Applications
```

- **Operational sources**: ERPs, CRMs, web logs, external feeds
- **ETL system**: extracts, cleans, conforms, and loads data — the "back room"
- **Presentation area**: the star schemas business users query — the "front room"
- **BI applications**: reports, dashboards, ad-hoc query tools

Kimball's key architectural tenet: **atomic, granular data belongs in the presentation area** (as star schemas), not hidden behind aggregation layers. OLAP cubes are built on top of the star schemas, not instead of them.

---

## Kimball book chapter map

| Chapter | Topic |
|---|---|
| 1 | DW/BI primer — goals, vocabulary, Kimball architecture |
| 2 | Complete dimensional modeling techniques reference (all 34 technique types) |
| 3 | Retail Sales — the canonical worked example and four-step process |
| 4 | Inventory — transaction, periodic snapshot, accumulating snapshot compared |
| 5 | Procurement — **all 7 SCD types** with full examples |
| 6 | Order Management — role-playing dims, junk dims, multi-currency, accumulating snapshot |
| 7 | Accounting — fixed/ragged hierarchies, consolidated facts, OLAP |
| 8 | CRM — bridge tables, multivalued dims, behavior step dimensions |
| 9 | HR — recursive hierarchies, survey data |
| 10 | Financial Services — mini-dimensions, household dimensions, dynamic value banding |
| 11 | Telecommunications — design review guidelines |
| 12–16 | Transportation, Education, Healthcare, eCommerce, Insurance |
| 17 | Kimball DW/BI Lifecycle — roadmap, program management |
| 18 | Dimensional modeling process and tasks |
| 19 | The 34 ETL subsystems |
| 20 | ETL system design and development |
| 21 | Big Data Analytics — Hadoop, MapReduce, best practices |

---

## When Kimball fits

- Business teams that need fast, self-service analytics now
- Organizations building incrementally — one business process at a time
- Environments where analyst-friendliness is paramount
- Most BI and reporting workloads

## Ch 2 — Complete 34-technique reference (summary)

Kimball's Chapter 2 is the complete reference for all dimensional modeling techniques. Key groups:

| Group | Techniques |
|---|---|
| **Fact table** | Transaction, periodic snapshot, accumulating snapshot, factless, aggregated, consolidated |
| **Dimension** | Conformed, degenerate, role-playing, junk, rapidly changing, large flat, date/time, SCD 0–7, mini-dimension |
| **Drill-across** | Conformed facts, shrunken dimensions |
| **Special topics** | Enterprise bus matrix, header/line fact, heterogeneous products, bridge tables for multi-valued dims, behavior study groups |

## Ch 7 — Ragged hierarchies

A ragged hierarchy has a variable number of levels — e.g. a corporate org chart where some branches have 3 levels and others have 7.

**Pathstring attribute approach:** denormalize the full path into a single column.

```
Employee.OrgPath = "CEO|COO|VP Sales|Regional Dir|District Mgr|John Smith"
```

Filtering by `OrgPath LIKE '%VP Sales%'` returns all employees under VP Sales at any depth.

**Bridge table approach:** a separate table records every ancestor-descendant pair with a depth column.

```sql
CREATE TABLE Bridge_OrgHierarchy (
    EmployeeKey   INT NOT NULL,
    AncestorKey   INT NOT NULL,
    Depth         INT NOT NULL,  -- 0 = self, 1 = parent, 2 = grandparent, ...
    IsSelf        BIT NOT NULL
);
```

Query: all employees under VP Sales + their subtotals = join FactPayroll → Bridge where `AncestorKey = [VPSalesKey]`.

## Ch 19 — The 34 ETL subsystems

Kimball defines 34 ETL subsystems grouped into 4 categories:

**Extracting (1–7):** Data profiling, Change data capture, Extract system, Table-driven diff (CDC without log access), Audit dimension, Surrogate key pipeline, Flat file reader

**Cleaning and conforming (8–17):** Data cleansing, Error event schema, Audit dimension, Conforming, Slowly changing dimension pipeline, Surrogate key lookup, Hierarchy manager, Special dimension loader, Facts table builder, Surrogate key pipeline

**Delivering (18–26):** Slowly changing dimension manager, Aggregate builder, OLAP cube builder, Data mart bus, Fact table loader, Surrogate key pipeline (fact), Late-arriving data handler, Error handler, Aggregate navigator

**Managing (27–34):** Job scheduler, Backup/recovery, Recovery/restart, Version control, Version migration, Lineage/dependency documentation, Problem escalation, Parallel/pipelined jobs

The 34 subsystems are not a software checklist — they are categories of ETL logic every warehouse needs. Different tools implement them differently.

## Ch 21 — Big Data and the modern lakehouse

Kimball's 2013 Big Data observations (Ch 21) aligned with what became the lakehouse pattern:

- Hadoop/HDFS enabled storing massive raw datasets cheaply — same principle as Bronze on object storage
- MapReduce enabled distributed transformation — same principle as Spark on a Delta lake
- The recommendation was to apply dimensional modeling on top of Hadoop outputs, not to abandon Kimball — the data model discipline does not change with the storage technology

In modern terms: Kimball's star schema design principles apply identically to Gold Delta tables on a Fabric Lakehouse or Databricks Delta Lake. The technology changed; the modeling discipline did not.
