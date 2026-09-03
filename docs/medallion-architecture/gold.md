# Gold Layer (Serving)

> *Source: Building Medallion Architectures — Piethein Strengholt.  
> Code examples: [chapter-07](https://github.com/pietheinstrengholt/building-medallion-architectures-book/tree/main/chapter-07)*

## Purpose

The Gold layer delivers **business-ready data** — optimized, aggregated, and structured specifically for consumption by analysts, dashboards, and applications. This is where business rules are applied and where the model is designed around how the business thinks, not how the source stores data.

## Key characteristics

- **Business-oriented** — organized around business processes and KPIs
- **Optimized for queries** — denormalized, pre-aggregated, few joins
- **Curated** — governed, documented, trusted
- **Multiple consumers** — the same Gold table may feed BI tools, APIs, and ML features

## Common Gold models

### Star schema (Kimball)

The most common Gold pattern — fact tables with dimension tables.

```
DimCustomer
     │
DimDate ──► FactSales ◄── DimProduct
     │
DimRegion
```

See [Star Schema](../data-modeling/star-schema.md) for full detail.

### One-Big-Table (OBT)

All relevant attributes in a single wide table. No joins needed.

**When it fits:**
- Data scientists who need flat input for ML models
- Fast ad-hoc analysis on a single subject area
- High-read, low-model-complexity use cases

**Trade-off:** redundant storage, harder to maintain consistency.

### Aggregated / summary tables

Pre-computed metrics for dashboard performance.

```sql
-- Example: daily sales by region
SELECT SalesDate, RegionName, SUM(SalesAmount) AS TotalSales
FROM FactSales JOIN DimRegion ...
GROUP BY SalesDate, RegionName
```

## Loading the Gold layer

### Loading dimensions first

Dimension tables must be loaded before fact tables, because facts need the surrogate keys from dimensions.

```
Bronze/Silver ──► Dimensions (with SCD2 if needed)
                       │
              ──►  Fact (lookup surrogates from dims)
```

### Surrogate key lookup

The fact load replaces natural/business keys from Silver with surrogate keys from dimension tables.

```sql
-- Replace CustomerBK with CustomerKey surrogate
SELECT d.CustomerKey, f.SalesAmount, ...
FROM SilverSales f
JOIN DimCustomer d ON f.CustomerBK = d.CustomerBK AND d.IsCurrent = 1
```

### Early-arriving facts

When a fact arrives before its dimension record exists, create a **placeholder dimension row** (unknown member) to maintain referential integrity. Update it when the dimension arrives.

## Governance at Gold

- Document every Gold table with purpose, grain, refresh schedule, and owner
- Apply row-level security where different teams should see different data
- Catalog Gold tables in a data catalog (Purview, Alation, DataHub)

## Code reference

- Gold tables: [`create_gold_tables.ipynb`](https://github.com/pietheinstrengholt/building-medallion-architectures-book/blob/main/chapter-07/create_gold_tables.ipynb)
- Dimension: customer [`dimension_customer.ipynb`](https://github.com/pietheinstrengholt/building-medallion-architectures-book/blob/main/chapter-07/dimension_customer.ipynb), product [`dimension_product.ipynb`](https://github.com/pietheinstrengholt/building-medallion-architectures-book/blob/main/chapter-07/dimension_product.ipynb)
- Fact: [`fact_sales.ipynb`](https://github.com/pietheinstrengholt/building-medallion-architectures-book/blob/main/chapter-07/fact_sales.ipynb)

## SCD2 dimension load with MERGE INTO (Delta Lake)

The standard SCD2 pattern in Delta Lake: detect changed rows by comparing a hash of tracked attributes, then expire the old row and insert a new one.

```python
from delta.tables import DeltaTable
from pyspark.sql.functions import sha2, concat_ws, col, lit, current_timestamp

# Step 1: compute hash of tracked attributes
source = spark.table("silver.crm_customers") \
    .withColumn("HashDiff", sha2(concat_ws("||", col("CustomerName"), col("CountryCode"), col("Segment")), 256))

# Step 2: MERGE into Gold dimension
gold_dim = DeltaTable.forName(spark, "gold.DimCustomer")

gold_dim.alias("tgt").merge(
    source.alias("src"),
    "tgt.CustomerBK = src.CustomerBK AND tgt.IsCurrent = true"
).whenMatchedUpdate(
    condition="tgt.HashDiff <> src.HashDiff",
    set={"tgt.ValidTo": current_timestamp(), "tgt.IsCurrent": lit(False)}
).whenNotMatchedInsert(
    values={
        "CustomerBK":    "src.CustomerBK",
        "CustomerName":  "src.CustomerName",
        "CountryCode":   "src.CountryCode",
        "Segment":       "src.Segment",
        "HashDiff":      "src.HashDiff",
        "ValidFrom":     current_timestamp(),
        "ValidTo":       lit(None),
        "IsCurrent":     lit(True)
    }
).execute()
```

After the MERGE, any row where `ValidTo` was set (expired) needs a corresponding new INSERT of the changed row. In practice, this is done in two passes: the MERGE expires old rows and inserts unchanged-or-new rows, then a second INSERT adds the new version of changed rows (where source hash ≠ target hash and target was `IsCurrent = true`).

## Hash-based change detection

Instead of comparing every column individually (`WHERE old.Col1 <> new.Col1 OR old.Col2 <> new.Col2 ...`), compute a single hash of all tracked attributes. One column comparison per row, regardless of how many attributes are tracked.

```python
from pyspark.sql.functions import sha2, concat_ws

df = df.withColumn(
    "HashDiff",
    sha2(
        concat_ws("||",
            col("CustomerName"),
            col("CountryCode"),
            col("Segment"),
            col("Email")
        ),
        256  # SHA-256, 64-char hex string
    )
)
```

`concat_ws("||", ...)` uses `||` as the separator to reduce collision risk between values like `("AB", "CD")` vs `("A", "BCD")`.

Store `HashDiff` as a column on the target dimension table. The MERGE condition is then simply `tgt.HashDiff <> src.HashDiff`.

## Semantic layer (Power BI / Fabric)

The Gold layer feeds a **semantic model** — the business-facing layer that hides SQL complexity behind user-friendly measures and hierarchies.

In Microsoft Fabric, the semantic model sits directly on top of Gold Delta tables (Direct Lake mode — no data copy, sub-second refresh):

```
Gold Delta Tables (DimCustomer, DimProduct, FactSales)
        │
        ▼ (Direct Lake connection)
Fabric Semantic Model
  ├── Measures: Total Sales = SUM(FactSales[SalesAmount])
  ├── Hierarchies: Date → Year → Quarter → Month → Day
  └── Row-level security: filter by Region based on user email
        │
        ▼
Power BI Reports / Excel / Copilot queries
```

Key semantic model additions over the raw Gold tables:
- **Calculated measures** (DAX) — KPIs, ratios, period-over-period growth
- **Hierarchies** — drill-down paths (Geography → Country → City)
- **Perspectives** — subset of the model exposed to specific audiences
- **Row-level security** — filter rows at query time based on user identity
