# Medallion vs Traditional DWH

## They solve the same problem differently

Both Medallion and traditional DWH methodologies aim to take raw data from source systems and turn it into trusted, analytics-ready data. The difference is in the storage technology, the modeling philosophy, and the degree of upfront schema commitment.

## Layer mapping

| Traditional DWH | Medallion layer | Notes |
|---|---|---|
| Staging area | Bronze | Raw, temporary vs raw, persistent |
| ODS / raw integration | Bronze/Silver boundary | Medallion keeps raw permanently |
| Inmon 3NF DWH | Silver (if Data Vault / 3NF used) | Optional — many Medallion implementations skip this |
| Data Vault (raw vault) | Silver | Common in enterprise Medallion setups |
| Data Vault (business vault) | Silver / Gold boundary | Enrichment and harmonization |
| Kimball data mart | Gold | Star schemas in Gold = Kimball model |

## Key differences

| | Traditional DWH | Medallion / Lakehouse |
|---|---|---|
| Storage technology | Relational RDBMS | Object storage (Delta Lake / Parquet) |
| Schema enforcement | Strict at load time | Flexible — schema-on-read in Bronze |
| Cost at scale | High (compute-bound RDBMS) | Low (cheap object storage, separate compute) |
| Raw data retention | Often discarded after staging | Bronze is permanent — full replay possible |
| Modeling upfront | Required (especially Inmon) | Progressive — add structure as needed |
| Semi-structured data | Poor support | Native (JSON, nested, arrays) |
| Streaming | Difficult | Native (Spark Structured Streaming) |

## What Medallion borrows from traditional DWH

- **Bronze ← Staging / ODS** — the concept of landing raw data before transformation
- **Silver ← 3NF / Data Vault** — clean, integrated, source-aligned data
- **Gold ← Kimball data mart** — dimensional star schemas for analytical consumption
- **SCD2** — the technique is identical; the implementation uses Delta Lake MERGE instead of SQL RDBMS triggers
- **Conformed dimensions** — still a best practice in Gold layer design

## When to choose Medallion over traditional DWH

| Factor | Medallion wins | Traditional DWH wins |
|---|---|---|
| Data volume | Very large (PB-scale) | Moderate (TB-scale) |
| Data variety | Mixed (structured + semi-structured + streaming) | Structured only |
| Cost | Need cheap storage at scale | Need strong consistency guarantees |
| Iteration speed | Need to evolve the model frequently | Model is stable and well-understood |
| Team skills | Spark / Python strong | SQL / RDBMS expertise strong |

In practice, most modern enterprise data platforms combine both: a Medallion lakehouse as the foundation, with Kimball star schemas in the Gold layer consumed via a traditional SQL Warehouse (like Microsoft Fabric or Snowflake).
