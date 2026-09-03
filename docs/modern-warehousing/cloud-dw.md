# Cloud Data Warehouses

## What changed vs on-premise DWH

| | On-premise DWH | Cloud DWH |
|---|---|---|
| Storage | Attached to compute nodes | Decoupled (object storage) |
| Scaling | Buy more hardware | Spin up more compute, pay per use |
| Maintenance | DBA-managed | Fully managed (SaaS) |
| Startup cost | High (hardware) | Low (pay as you go) |
| Semi-structured data | Poor | Native JSON, nested columns |

## Major platforms

| Platform | Provider | Key trait |
|---|---|---|
| **Snowflake** | Multi-cloud | Virtual warehouses, time travel, data sharing |
| **BigQuery** | Google Cloud | Serverless, pay per query, nested RECORD types |
| **Redshift** | AWS | Columnar, RA3 nodes with S3-decoupled storage |
| **Microsoft Fabric** | Microsoft | Unified lakehouse + DWH + BI in one platform |
| **Databricks SQL Warehouse** | Multi-cloud | Spark-backed SQL on Delta Lake |
| **Azure Synapse** | Microsoft | Dedicated pools (old) + serverless SQL |

## Separation of storage and compute

The defining architectural feature of modern cloud DWH. Storage (S3, ADLS, GCS) is cheap and unlimited. Compute clusters are spun up on demand and released when done.

This means you can have multiple compute clusters querying the same data simultaneously — one for ETL, one for ad-hoc queries, one for dashboards — without contention.

## Snowflake virtual warehouse sizing

Snowflake compute is sold in T-shirt sizes (XS, S, M, L, XL, ...). Each size doubles the credit consumption per hour:

| Size | Credits/hour | Typical use |
|---|---|---|
| XS | 1 | Dev/test, light queries |
| S | 2 | Ad-hoc analyst queries |
| M | 4 | ETL jobs, moderate concurrency |
| L | 8 | Heavy transformations, data loads |
| XL | 16 | Large batch ETL, high concurrency |
| 4XL | 128 | Extreme batch, data science |

Auto-suspend (idle timeout) and auto-resume are critical for cost control — set `AUTO_SUSPEND = 60` seconds on dev warehouses. Multi-cluster warehouses auto-add compute during concurrency spikes.

## BigQuery pricing models

| Model | How it works | Best for |
|---|---|---|
| **On-demand** | Pay per TB of data scanned ($6.25/TB as of 2024) | Infrequent ad-hoc queries |
| **Capacity (slot-based)** | Purchase fixed slots (100 slot minimum); flat monthly cost | Predictable, high-volume workloads |
| **BigQuery Editions** | Standard/Enterprise/Enterprise+ — include autoscaling slots | Teams wanting managed slot pools |

Key cost control: `SELECT *` scans the entire table and is expensive. Always select only needed columns; partition and cluster tables to enable pruning.

## Microsoft Fabric capacity

Fabric uses **Capacity Units (CUs)** — a unified compute currency across all Fabric workloads (Spark, SQL Warehouse, Power BI, pipelines).

| SKU | CUs | Approx. use case |
|---|---|---|
| F2 | 2 | Dev/test only |
| F4 | 4 | Small team, light analytics |
| F8 | 8 | Small team + ETL |
| F32 | 32 | Mid-size team, regular pipeline loads |
| F64 | 64 | Large team, concurrent workloads |
| F128+ | 128+ | Enterprise, Power BI Premium-equivalent |

Fabric charges burst CUs against a smoothed capacity pool — short spikes are absorbed; sustained overuse causes throttling. Monitor via the Fabric Capacity Metrics app.

## SQL dialect comparison

| Feature | Fabric SQL Warehouse | Snowflake | BigQuery | Databricks SQL |
|---|---|---|---|---|
| String type | `VARCHAR(n)` (no nvarchar) | `VARCHAR` / `STRING` | `STRING` | `STRING` |
| Timestamp precision | `datetime2(6)` max | `TIMESTAMP_TZ` | `TIMESTAMP` | `TIMESTAMP` |
| MERGE | Yes | Yes | Yes | Yes (Delta) |
| CREATE TABLE AS SELECT | Yes (CTAS) | Yes | Yes | Yes |
| Temp tables | `#table` syntax | Supported | Not supported (use CTEs) | Supported |
| Time travel | Via Delta (Lakehouse) | `AT (TIMESTAMP =>...)` | `FOR SYSTEM_TIME AS OF` | `VERSION AS OF` |
| Array/JSON | Limited JSON support | VARIANT type | ARRAY/STRUCT native | ARRAY native |
