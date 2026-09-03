# Naming Conventions

Consistent naming makes a data platform navigable without documentation. Inconsistent naming forces everyone to memorize exceptions.

## Tables

| Pattern | Example | When |
|---|---|---|
| `Fact` prefix | `FactSales`, `FactOrders` | Kimball fact tables |
| `Dim` prefix | `DimCustomer`, `DimProduct` | Kimball dimension tables |
| `stg` prefix | `stgSalesforce`, `stgErp` | Staging / source-aligned tables |
| `TD_` prefix | `TD_Product` | Dimension in enterprise DWH (Inmon style) |
| `TF_` prefix | `TF_Sales` | Fact in enterprise DWH |
| `TS_` prefix | `TS_CustomerHistory` | Snapshot / history tables |

## Columns

| Pattern | Example | When |
|---|---|---|
| `_SK` suffix | `CustomerKey`, `Customer_SK` | Surrogate key |
| `_BK` suffix | `CustomerBK` | Business / natural key |
| `_ID` suffix | `CustomerID` | Source system identifier |
| `SysCreDT` | `SysCreDT` | System: row creation timestamp |
| `SysUpdDT` | `SysUpdDT` | System: last update timestamp |
| `SysDelFlag` | `SysDelFlag` | System: soft delete flag |
| `ValidFrom`, `ValidTo` | `ValidFrom` | SCD2 validity period |
| `IsCurrent` | `IsCurrent` | SCD2 current-row flag |

## General column rules

- Use `PascalCase` for column names (consistent capitalization)
- Avoid abbreviations unless universally understood (`BK` = business key, `SK` = surrogate key)
- Boolean columns: `IsActive`, `IsCurrent`, `HasChildren` — always `Is`/`Has` prefix
- Date columns: `OrderDate` (not `order_dt`, not `dtOrder`)
- Timestamp columns: `CreatedAt`, `UpdatedAt` — or system prefix `SysCreDT` for framework columns

## Schemas

Use schemas to separate concerns:

| Schema | Purpose |
|---|---|
| `bronze` / `raw` / `stg*` | Source-aligned, unprocessed data |
| `silver` / `cleansed` | Cleaned, conformed data |
| `gold` / `dbo` / `Report` | Business-ready, served data |
| `backup` | Point-in-time backup tables |
| `System` / `SystemLog` | Framework and audit tables |

## Pipelines and processes

- **Top-level pipelines** carry a shared prefix that distinguishes them from sub-pipelines: `PL_`, `PIPELINE_`, or similar — pick one and apply it everywhere
- **Framework-managed procedures** use a consistent prefix that indicates the operation type: a load procedure prefix differs from a utility or audit prefix
- **Procedure names encode the target object**: the table or entity being written is part of the name, not buried in a comment — `LoadDim_Customer` is navigable; `LoadData_001` is not
- **Sub-pipelines and helper procedures** use a different prefix or suffix from top-level ones — the name should reveal the hierarchy at a glance

## File naming conventions

| File type | Pattern | Example |
|---|---|---|
| Raw data file (CSV/JSON) | `{source}_{entity}_{YYYYMMDD}.csv` | `crm_account_20260807.csv` |
| Raw data file (streaming batch) | `{source}_{entity}_{YYYYMMDD_HHMMSS}.json` | `kafka_order_20260807_143022.json` |
| Change/release script | `{TICKET-ID}_{description}.py` or `.sql` | `DWH-1042_dim_salesrep_fix.py` |
| dbt model file | `{layer}_{source}_{entity}.sql` | `stg_crm_account.sql`, `dim_customer.sql` |
| Notebook | `{layer}_{action}_{entity}.ipynb` | `silver_clean_crm_customers.ipynb` |
| Backup table | `{Schema}_{Table}_{USERABBR}_{YYYYMMDD}` | `Analytics_DimCustomer_JDO_20260807` |

## dbt model naming

dbt convention: prefix tells you which layer and what kind of model it is.

| Prefix | Layer | Purpose |
|---|---|---|
| `stg_` | Staging (Bronze→Silver) | 1-to-1 source tables; rename, cast, minimal logic |
| `int_` | Intermediate | Multi-source joins; business logic; not directly consumed |
| `dim_` | Mart dimension | Final dimension table in the star schema |
| `fact_` | Mart fact | Final fact table |
| `agg_` | Aggregate | Pre-computed summaries on top of facts |
| `rpt_` | Report | Report-specific views; often one-off |

Naming within a layer:

```
stg_crm_account       ← source=crm, entity=account
stg_erp_order_line    ← source=erp, entity=order_line
int_customer_lifetime ← intermediate calculation
dim_customer
fact_sales
agg_daily_sales_by_region
```

## Case style comparison

| Style | Example | Common in |
|---|---|---|
| `PascalCase` | `CustomerName`, `SalesAmount` | SQL Server, SSAS, .NET DWH stacks |
| `snake_case` | `customer_name`, `sales_amount` | Python, dbt, Databricks, BigQuery, Snowflake |
| `camelCase` | `customerName`, `salesAmount` | REST APIs, JSON schemas, JavaScript |
| `UPPER_SNAKE_CASE` | `CUSTOMER_NAME`, `SALES_AMOUNT` | Legacy Oracle, some ETL tools |

**Recommendation:** pick one style per layer and enforce it consistently. Mixing styles within a layer (some columns PascalCase, some snake_case) is the worst outcome — it forces every consumer to memorize exceptions.

In practice: use `snake_case` for Python/Spark code and dbt models; use `PascalCase` for SQL Server / Fabric Warehouse DDL if your team has a PascalCase history. The critical rule is consistency within a platform, not conformity across platforms.
