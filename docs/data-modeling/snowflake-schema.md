# Snowflake Schema

## What it is

A snowflake schema is a normalized variant of the [star schema](star-schema.md). Dimension tables are broken out into sub-dimensions, reducing redundancy at the cost of more joins.

## Star vs snowflake

**Star** — DimProduct contains everything:

```
FactSales → DimProduct (ProductName, Category, SubCategory, Brand)
```

**Snowflake** — DimProduct references sub-tables:

```
FactSales → DimProduct → DimCategory → DimBrand
```

## When snowflake makes sense

- Very high cardinality dimensions where redundancy is costly
- When a sub-dimension is shared across multiple parent dimensions
- Strict normalization requirements from governance or tooling

### The subdimension pattern: a legitimate snowflake use case

A **subdimension** is a set of attributes split off from a larger dimension into its own table — not because normalization requires it, but because the separated attributes have genuinely different operational characteristics:

**Different load timing:** demographic attributes for a customer (city classification, income band, lifestyle segment) may be loaded from a third-party data provider on a monthly schedule, while the core customer attributes (name, address, account status) are loaded nightly from the CRM. Separating them into a `CustomerDemographic` subdimension allows each to be loaded on its own schedule without blocking the other.

**Different granularity:** a city classification subdimension maps entire cities to quality-of-life scores, population ranges, and commerce indices. Many customers share the same city classification record. Storing the classification once in a subdimension (joined via a city key on the customer dimension) avoids repeating the classification columns for every customer in the same city.

**Different browsing patterns:** if users frequently filter on demographic attributes but rarely filter on core attributes in the same query, separating them allows the BI tool to browse demographics independently, with a smaller table to scan.

```
DimCustomer (CustomerKey, CustomerName, Address, State, CityClassKey)
    └─→ DimCityClassification (CityClassKey, CityCode, PopulationRange, CostOfLiving, QualityOfLife)
```

This is not the same as normalization-for-normalization's sake — it reflects a real operational difference in how the two sets of attributes are sourced, refreshed, and used.

## When to prefer star

Most modern analytical environments (cloud DWH, Spark) benefit more from fewer joins than from storage savings. **Star schema is almost always the better default** for performance and analyst usability.

## Storage vs join cost

With 1M product rows, 5 categories, and 20 subcategories:

| Approach | Product rows | Extra tables | Join depth | Storage |
|---|---|---|---|---|
| Star (denormalized) | 1,000,000 | 0 | 1 | Higher (CategoryName repeated 1M times) |
| Snowflake (normalized) | 1,000,000 | 2 (DimCategory + DimSubCategory) | 3 | Lower (CategoryName stored once per category) |

On modern columnar storage (Parquet, Delta Lake), columnar compression makes the redundancy in the star schema nearly free — `CategoryName VARCHAR(100)` repeated 1M times compresses to kilobytes. The extra joins in a snowflake schema are the real cost — they require shuffles in Spark and more complex query plans.

**Conclusion:** for most cloud/lakehouse workloads, the storage savings of snowflaking are negligible, and the join overhead is real. Default to star schema unless there's a specific reason to normalize.

## OLAP cube implications

Traditional OLAP cubes (SQL Server Analysis Services, SSAS) were built on top of star or snowflake schemas. Snowflake schemas required the cube tool to define relationships across multiple dimension tables — adding configuration overhead.

Modern semantic models (Power BI, Fabric semantic model, Tableau) handle snowflake relationships natively but still add query complexity. The simpler the physical model, the simpler the semantic model built on top.

For Fabric Direct Lake mode: fewer tables = fewer relationships = faster query rewriting by the engine. Another argument for preferring star over snowflake.
