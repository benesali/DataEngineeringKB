# Star Schema

## What it is

The star schema is the standard data model for analytical (OLAP) workloads. It has one central **fact table** connected to multiple **dimension tables** — visually resembling a star.

It was popularized by Ralph Kimball and remains the most widely used analytical model.

## Structure

```
              DimDate
                 │
  DimCustomer ──►  FactSales ◄── DimProduct
                 │
             DimRegion
```

### Fact table

Contains one row per measurable **event or transaction**. Columns:
- **Foreign keys** to every dimension (DateKey, CustomerKey, ProductKey, RegionKey)
- **Measures** — the numbers you aggregate (SalesAmount, Quantity, DiscountAmount)

```sql
CREATE TABLE FactSales (
    SalesKey        INT          NOT NULL,  -- surrogate PK
    DateKey         INT          NOT NULL,  -- FK → DimDate
    CustomerKey     INT          NOT NULL,  -- FK → DimCustomer
    ProductKey      INT          NOT NULL,  -- FK → DimProduct
    RegionKey       INT          NOT NULL,  -- FK → DimRegion
    Quantity        INT          NOT NULL,
    SalesAmount     DECIMAL(18,2) NOT NULL,
    DiscountAmount  DECIMAL(18,2) NOT NULL
);
```

### Dimension table

Contains one row per **entity** — one per customer, one per product, one per date. Columns are descriptive attributes.

```sql
CREATE TABLE DimCustomer (
    CustomerKey     INT           NOT NULL,  -- surrogate PK
    CustomerBK      VARCHAR(50)   NOT NULL,  -- business key from source
    CustomerName    VARCHAR(200)  NOT NULL,
    Segment         VARCHAR(100)  NULL,
    Country         VARCHAR(100)  NULL,
    -- SCD2 columns (if tracking history):
    ValidFrom       DATE          NOT NULL,
    ValidTo         DATE          NULL,
    IsCurrent       BIT           NOT NULL
);
```

## Why star over 3NF for analytics

| | 3NF | Star Schema |
|---|---|---|
| Joins needed | Many (10-20+ for a simple query) | Few (one per dimension) |
| Analyst-friendliness | Low | High |
| Query performance | Poor | Good |
| Storage efficiency | High | Medium (some redundancy) |
| Update efficiency | High | Low |

## The grain

The **grain** is the single most important design decision. It defines what one row in the fact table represents.

Examples:
- "One row = one order line" (most granular)
- "One row = one order" (aggregated)
- "One row = daily sales total per store and product" (snapshot)

Every dimension and measure must be consistent with the declared grain.

## Measure types

| Type | Can SUM across... | Example | Correct aggregation |
|---|---|---|---|
| **Additive** | All dimensions | SalesAmount, Quantity | SUM |
| **Semi-additive** | Most dimensions, NOT time | Account balance, Inventory level | AVG across time; SUM across accounts |
| **Non-additive** | No dimensions | Unit price, Ratio, Percentage | MIN/MAX or calculate from additive components |

Rule of thumb: if summing across time produces a meaningless number, the measure is semi-additive. Never pre-aggregate non-additive measures — always store the components and compute the ratio in the BI tool.

## Degenerate dimensions

A dimension attribute with no corresponding dimension table — stored directly on the fact table because it has no descriptive attributes beyond itself.

**Most common example:** the transaction ID, order number, or invoice number.

```sql
CREATE TABLE FactSales (
    SalesKey     INT NOT NULL,
    DateKey      INT NOT NULL,
    CustomerKey  INT NOT NULL,
    ProductKey   INT NOT NULL,
    OrderNumber  VARCHAR(20) NOT NULL,  -- degenerate dimension
    SalesAmount  DECIMAL(18,2) NOT NULL
);
```

`OrderNumber` is used to group order lines (sub-selecting all lines for a given order) but has no descriptive attributes worth putting in a dimension table.

## Role-playing dimensions

The same physical dimension table used multiple times in one fact table, each playing a different role.

**Most common example:** DimDate as OrderDate, ShipDate, and DeliveryDate.

```sql
-- View aliases for clarity
CREATE VIEW DimOrderDate AS SELECT * FROM DimDate;
CREATE VIEW DimShipDate  AS SELECT * FROM DimDate;

-- Fact with three date foreign keys, each joining to the same DimDate
SELECT o.OrderMonth, s.ShipMonth, SUM(f.SalesAmount)
FROM FactOrderFulfillment f
JOIN DimOrderDate o ON f.OrderDateKey = o.DateKey
JOIN DimShipDate  s ON f.ShipDateKey  = s.DateKey
GROUP BY o.OrderMonth, s.ShipMonth
```

## Junk dimensions

A catch-all dimension for low-cardinality flags and codes that don't belong in any natural dimension but would clutter the fact table if stored there individually.

**Example:** `IsOnlineOrder` (Y/N), `PaymentMethod` (Card/Cash/Voucher), `IsGift` (Y/N), `PromotionApplied` (Y/N).

Instead of 4 columns on the fact, combine all permutations into a small junk dimension:

```sql
CREATE TABLE DimTransactionFlags (
    TransactionFlagsKey   INT NOT NULL,  -- surrogate key
    IsOnlineOrder         BIT NOT NULL,
    PaymentMethod         VARCHAR(20) NOT NULL,
    IsGift                BIT NOT NULL,
    PromotionApplied      BIT NOT NULL
    -- total rows: 2 × 3 × 2 × 2 = 24 permutations max
);
```

The fact table gets one `TransactionFlagsKey` instead of four separate columns.

## Large dimensions

A dimension is "large" in two senses:

**Very deep** — very many rows. Customer and product dimensions are the most common large dimensions:
- Customer dimensions in national retail can reach 20–100 million rows (one per household)
- Product dimensions in large retailers can reach 100,000+ product variations
- Telecommunications and travel industries also produce multi-million row customer dimensions

**Very wide** — very many attributes. Customer dimensions can have 100–150 attributes; product dimensions 100+. Both can also have **multiple hierarchies** (e.g. a marketing hierarchy and a finance hierarchy for the same product dimension).

**Performance challenges with large dimensions:**
- Initial population is slow (100M rows to ETL)
- Browsing a large unconstrained dimension is slow (selecting all distinct values of a low-cardinality attribute still requires scanning all rows unless indexed)
- Type 2 SCD changes create additional rows on every change, compounding the size problem
- Fact table joins slow down when the dimension is large

**Design responses:** choose proper indexes; consider minidimensions for rapidly changing attributes (see below); build aggregate fact tables at the grain that avoids traversal of the full large dimension.

---

## Rapidly changing dimensions

A dimension attribute that changes frequently (e.g. credit rating, income level, customer tier) on a very large dimension causes a "Type 2 row explosion": millions of new rows are inserted each time the attribute changes, even for customers with no sales activity. The dimension becomes unmanageable.

**Solution: minidimension (behavior dimension)**

Split the large dimension into:
- **Core dimension** — stable attributes that change infrequently (CustomerName, Address, State, CustomerCode)
- **Minidimension** — rapidly changing attributes grouped by type (CreditRating, IncomeLevel, MaritalStatus, LifeStyle, HomeOwnership, PurchasesRange)

```
Customer (core):          Behavior (mini):
CustomerKey (PK)          BehaviorKey (PK)
CustomerName              CustomerType
Address                   CreditRating
State                     MaritalStatus
Zip                       IncomeLevel
                          HomeOwnership
```

The fact table gets two foreign keys: `CustomerKey` (to the stable core) and `BehaviorKey` (to the current minidimension snapshot). When behavioral attributes change, only a new minidimension row is created — the core dimension row is unchanged, and the fact table just references a different `BehaviorKey`.

The minidimension contains all *combinations* of its attributes (e.g. all valid combinations of CreditRating × IncomeLevel × MaritalStatus). It has far fewer rows than the full customer dimension, so it's cheap to scan and browse.

---

## STARjoin and STARindex

Two performance techniques enabled by the star schema arrangement:

**STARjoin:** a high-speed, single-pass, parallelizable multi-table join. Standard SQL joins are pairwise (A JOIN B JOIN C JOIN D = three separate join operations). STARjoin can join more than two tables in a single operation — the query processor uses the star topology (each dimension table connects only to the fact table, not to each other) to execute all dimension joins together in one scan of the fact table.

**STARindex:** a specialized index built on one or more foreign key columns of the fact table. When a query filters on dimension attributes (e.g. `ProductCategory = 'Electronics' AND Region = 'APAC'`), the database first resolves which keys satisfy the filter from each dimension table, then uses the STARindex to quickly identify the matching fact table rows — without a full fact table scan.

Both techniques are available in major analytical databases (Oracle, SQL Server) and are a primary reason the star schema's simple topology translates to efficient execution plans.

---

## Conformed dimensions

A conformed dimension is shared — with the **same keys, same attribute names, same definitions** — across multiple fact tables. This enables **drill-across**: combining metrics from two separate fact tables in one report.

```
DimDate ──► FactSales
DimDate ──► FactInventory
DimDate ──► FactOrders

-- Drill-across query: sales vs inventory by month
SELECT d.MonthName,
       SUM(s.SalesAmount) AS TotalSales,
       SUM(i.InventoryLevel) AS AvgInventory
FROM DimDate d
LEFT JOIN FactSales      s ON d.DateKey = s.DateKey
LEFT JOIN FactInventory  i ON d.DateKey = i.DateKey
GROUP BY d.MonthName
```

If different subject areas use different Date dimensions (different keys, different fiscal year definitions), the drill-across query breaks — you cannot join them. Conforming prevents this.
