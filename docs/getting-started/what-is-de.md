# What is Data Engineering?

## The core job

Data engineering is the discipline of **designing and building systems that collect, store, transform, and serve data** reliably at scale.

Where a data scientist asks "what does the data tell us?", a data engineer asks "how do we get the data there in the first place — reliably, at the right quality, and fast enough?"

## What data engineers build

- **Pipelines** — automated workflows that move data from source systems into storage
- **Data warehouses / lakehouses** — structured storage optimized for analytical queries
- **Data models** — how tables relate to each other (star schemas, vaults, etc.)
- **Transformations** — cleaning, conforming, aggregating raw data into usable form
- **Orchestration** — scheduling and dependency management between pipeline steps
- **Monitoring & alerting** — knowing when data is late, wrong, or missing

## How it fits with other roles

```
Source systems
      │
      ▼
 Data Engineer  ──── builds ────►  Warehouse / Lakehouse
                                         │
                              ┌──────────┴──────────┐
                              ▼                     ▼
                        Data Analyst          Data Scientist
                    (dashboards, reports)  (ML models, forecasts)
```

## Real-world example: CRM → warehouse pipeline

A customer record is created in Salesforce. Here is how it travels to a Gold dimension table:

```
1. Salesforce CRM (OLTP)
        │  (Dataverse Synapse Link / CDC / API extract)
        ▼
2. Bronze table (raw, immutable)
   bronze.crm_account — raw JSON or Parquet, one row per Salesforce record version
        │  (Silver notebook: clean, rename, type-cast)
        ▼
3. Silver table (clean, source-aligned)
   silver.crm_account — CustomerBK, CustomerName, CountryCode, Email, HashDiff
        │  (Gold notebook: SCD2 MERGE, surrogate key assignment)
        ▼
4. Gold dimension table (business-ready)
   gold.DimCustomer — CustomerKey, CustomerBK, CustomerName, ..., ValidFrom, ValidTo, IsCurrent
        │  (Power BI DirectLake semantic model)
        ▼
5. Dashboard / report
   "Customer count by country"
```

Each step is a separate scheduled pipeline. A failure at step 3 does not corrupt Bronze — data can be reprocessed from the raw layer.

## Common tools stack

| Tool | Role | When to use |
|---|---|---|
| **Azure Data Factory** | Orchestration + copy activity | Moving data from source to Bronze; triggering notebooks |
| **Apache Spark / Databricks** | Distributed transformation | Silver and Gold transformations on large datasets |
| **dbt** | SQL-based transformation | Silver/Gold layer when team prefers SQL over Python |
| **Apache Airflow** | Orchestration | Python-first teams; complex DAG dependency management |
| **Delta Lake** | Table format | ACID transactions + time travel on object storage |
| **Microsoft Fabric** | Unified platform | All-in-one: Lakehouse + SQL + Spark + Power BI |
| **Kafka / Event Hubs** | Streaming ingestion | Real-time event streams into Bronze |

There is no universal stack — the right combination depends on team skills, cloud platform, and data volume. Most modern platforms (Fabric, Databricks) bundle orchestration + compute + storage.

## On-prem vs cloud data engineering

| | On-premise | Cloud |
|---|---|---|
| Storage | SAN/NAS, fixed capacity | Object storage (S3, ADLS, GCS) — elastic, cheap |
| Compute | Dedicated servers, sized at peak | Elastic clusters — pay for what you use |
| Maintenance | DBA team manages hardware + OS + DB | Managed services (PaaS/SaaS) — vendor manages infra |
| Data sharing | Expensive and complex | Zero-copy sharing (Delta Sharing, OneLake Shortcuts) |
| Semi-structured data | Poor (RDBMS) | Native (JSON columns, Parquet, Avro) |
| Scaling time | Months (hardware procurement) | Minutes (spin up cluster) |
| Cost model | CapEx (large upfront) | OpEx (pay-as-you-go) |

The shift to cloud eliminated most of the operational burden of on-premise DE, but introduced new challenges: cost control (runaway compute bills), data governance across many cloud services, and organizational skills gaps in cloud-native tools.
