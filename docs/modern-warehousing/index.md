# Modern Warehousing

The shift to cloud-native data platforms in the 2010s–2020s changed the economics and architecture of data warehousing fundamentally.

## In this section

| Page | What you'll learn |
|---|---|
| [Cloud Data Warehouses](cloud-dw.md) | Snowflake, BigQuery, Redshift, Fabric — what makes them different |
| [Lakehouse Architecture](lakehouse.md) | Combining lake flexibility with warehouse query performance |
| [Delta Lake & Open Table Formats](open-table-formats.md) | Delta, Iceberg, Hudi — ACID on object storage |
| [ELT vs ETL](elt-vs-etl.md) | Why the transformation location shifted inside the warehouse |

## The fundamental shift

**Before:** compute and storage were bundled. A bigger warehouse cost more. Data was transformed *before* landing (ETL).

**After:** compute and storage are decoupled. Storage (object store) is near-free. Compute scales independently. Data lands raw first, transforms inside (ELT).
