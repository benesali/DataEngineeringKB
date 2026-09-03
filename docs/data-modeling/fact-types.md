# Fact Table Types

## Transaction fact table

One row per discrete event. The most common type.

- One row = one order line, one payment, one web click
- Grows continuously as events occur
- Supports the most granular analysis

Example: FactSales, FactPayments, FactWebEvents

## Periodic snapshot fact table

One row per period per entity — capturing state at a fixed point in time.

- One row = inventory level per product per day
- Created by a scheduled job, regardless of whether anything changed
- Used for trend analysis and period-over-period comparison

Example: FactDailyInventory, FactMonthlyAccountBalance

## Accumulating snapshot fact table

One row per instance of a multi-step process, updated as the process moves through stages.

- One row = one order, updated across fulfillment stages
- Contains multiple date keys (OrderDate, PickDate, ShipDate, DeliveryDate)
- The row is modified (not appended) as stages complete

Example: FactOrderFulfillment, FactLoanApplication

## Choosing the right type

| | Transaction | Periodic Snapshot | Accumulating Snapshot |
|---|---|---|---|
| Row = | An event | A period-end state | One process instance |
| Row changes? | Never | Never (new row each period) | Yes, as stages complete |
| Best for | Event analysis | Trend / period comparison | Process / pipeline analysis |
| Date keys | One (event date) | One (period date) | Many (one per stage) |

## Factless fact table

A fact table with no measures — it records that an event *occurred* or that a coverage *existed*, with no numeric value attached.

**Use case 1 — event coverage:** which products were on promotion on which dates in which stores?

```sql
CREATE TABLE FactPromotionCoverage (
    DateKey        INT NOT NULL,
    ProductKey     INT NOT NULL,
    StoreKey       INT NOT NULL,
    PromotionKey   INT NOT NULL
    -- no measures: the row itself is the fact
);
```

To answer "how many products were promoted on 2026-08-01?": `COUNT(*)` on `FactPromotionCoverage WHERE DateKey = 20260801`.

**Use case 2 — absence detection:** recording that a student *attended* a class enables detecting who *did not* attend by joining against DimStudent.

## Aggregate fact tables

An aggregate fact table stores pre-computed summaries at a *higher grain* than the base (atomic) fact table.

**Motivation:** the base fact table may hold billions of rows at the transaction line level. A query asking "total monthly sales by region" still scans all rows even though the answer requires only 12 × N rows. An aggregate fact table pre-computes this summary once.

### Types by dimension count

| Type | What it aggregates | Example |
|---|---|---|
| **One-way aggregate** | Across one dimension only | Monthly totals (collapses the date dimension from day to month) |
| **Two-way aggregate** | Across two dimensions | Monthly totals by region (collapses date + collapses geography) |
| **Three-way aggregate** | Across three dimensions | Monthly totals by region by product category |

Each type corresponds to a specific query pattern. For a DWH with frequent monthly/region/category queries, a three-way aggregate delivers sub-second response where the base table query would take minutes.

### Sparsity

At the atomic grain, most combinations of dimension keys have no fact rows (most customers don't buy most products on most days). Sparsity at the base level is normal.

As you aggregate up (from daily to monthly, from product to category), sparsity *decreases* — summaries cover more combinations. Aggregate tables are denser than base tables relative to their size.

### Aggregate strategy

Two approaches:

**Materialized aggregates (pre-computed):** build and maintain a physical aggregate fact table. Query tools are directed to the aggregate table when the query pattern matches. Requires:
- Aggregate navigation (OLAP tool or BI layer must know which aggregate to use for which query grain)
- Refresh after each base load (aggregates become stale when base data changes)

**On-the-fly aggregation:** no pre-built table; queries aggregate at runtime from the base table. Works when the BI engine or columnar storage (e.g. Parquet + Fabric) is fast enough that pre-computation provides no meaningful advantage.

Modern columnar engines (Fabric SQL Warehouse, Databricks Photon, BigQuery BI Engine) often make materialized aggregates unnecessary for moderately sized datasets. At very large scales (100B+ rows), they remain valuable.

---

## Families of stars

A **family of stars** is a group of related fact tables designed to work together — sharing conformed dimensions and answering complementary questions about the same business process.

### Snapshot + transaction pair

Two fact tables covering the same process from different angles:

- **Transaction fact table** — one row per event; records what happened (FactSales: each order line)
- **Periodic snapshot fact table** — one row per entity per period; records state at a point in time (FactDailyInventory: inventory level per product per day)

The snapshot answers "what was the state?" The transaction answers "what changed, and when?" Together they give a complete picture: current inventory + the individual sales transactions that drove it to that level.

### Core + custom pair

When different user groups need different cuts of the same process:

- **Core fact table** — the shared base (FactSales with standard measures: quantity, amount, discount)
- **Custom fact tables** — extensions for specific domains (FactSales_Finance with margin data; FactSales_Logistics with weight and volume)

The core table is maintained centrally. Custom tables are owned by the domain team and join to the core via the grain key. All custom tables share the same conformed dimensions as the core.

### Value chain / value circle

When multiple processes in a business flow sequentially, each producing a fact table:

**Value chain:** linear flow — procurement → inventory → sales → delivery

```
FactPurchaseOrder → FactInventory → FactSales → FactDelivery
```

Each table covers one stage. Conformed dimensions (DimProduct, DimDate, DimLocation) connect them — a drill-across query can compare purchase cost vs sales revenue vs inventory level for the same product.

**Value circle:** a circular flow where the output of the last stage feeds back into the first (e.g. returns → reprocessing → resale). Model as a chain that loops back rather than literally circular tables.

---

## Early-arriving facts and late-arriving dimensions

**Early-arriving fact**: a fact row arrives before its dimension record exists. Example: an order line for a new customer whose CustomerBK hasn't been processed through the Customer dimension load yet.

**Solution — placeholder / unknown member row:**

```sql
-- DimCustomer always has a catch-all unknown row
INSERT INTO gold.DimCustomer (CustomerBK, CustomerName, ..., IsCurrent)
VALUES (-1, 'Unknown', ..., 1);
```

The fact load joins to CustomerKey = -1 when the customer is not found. When the real customer dimension row arrives later, a correction job re-links the fact rows to the correct CustomerKey.

**Late-arriving dimension**: the dimension row arrives after the fact, but with a ValidFrom date in the past. The SCD2 history must be back-patched to cover the period when the fact already existed. This is one of the harder ETL problems — it requires re-evaluating which historical dimension row was valid at the time of each fact row.
