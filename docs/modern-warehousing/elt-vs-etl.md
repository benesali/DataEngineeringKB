# ELT vs ETL

## ETL — Extract, Transform, Load

The traditional pattern. Data is transformed *before* it lands in the destination.

```
Source ──► Extract ──► Transform (external tool) ──► Load (clean data)
```

**Where transformation happens:** a dedicated ETL server or tool (SSIS, Informatica, Talend, ADF Data Flow)

**When it made sense:** destination was an expensive RDBMS — you didn't want to store or process raw data there. Compute was cheap externally; storage in the DWH was expensive.

## ELT — Extract, Load, Transform

The modern pattern. Raw data lands first; transformation runs *inside* the destination.

```
Source ──► Extract ──► Load (raw data) ──► Transform (inside warehouse / lake)
```

**Where transformation happens:** inside the cloud DWH or lakehouse (SQL, Spark, dbt)

**Why it works now:** cloud object storage is near-free, so storing raw data is cheap. Cloud DWH compute (Spark, BigQuery, Snowflake, Fabric SQL) is massively scalable — transforming inside is faster than shuttling data through an external tool.

## Comparison

| | ETL | ELT |
|---|---|---|
| Transform happens | Before load | After load |
| Raw data retained? | No (usually discarded) | Yes (Bronze layer) |
| Tool for transform | External (SSIS, Informatica) | SQL / Spark / dbt inside the platform |
| Replayability | Hard (raw gone) | Easy (re-transform from Bronze) |
| Semi-structured data | Difficult | Native |
| Latency | Higher (multi-hop) | Lower (fewer hops) |
| Best for | Legacy on-prem DWH | Cloud DWH / Lakehouse |

## In Medallion terms

ELT maps directly to Medallion:
- **Extract + Load** = Bronze (raw data lands unchanged)
- **Transform** = Silver + Gold (transformation inside the platform)

## dbt — the ELT transform layer

dbt (data build tool) is the dominant open-source tool for the "T" in ELT. It writes transformations as SQL SELECT statements with Jinja templating, manages dependencies between models, and runs tests on the output.

```sql
-- models/silver/stg_customers.sql
SELECT
    customer_id,
    TRIM(customer_name) AS customer_name,
    LOWER(email)        AS email,
    CAST(signup_date AS DATE) AS signup_date
FROM {{ source('raw', 'customers') }}
WHERE customer_id IS NOT NULL
```

## dbt project structure

dbt organizes transformations into **models** — SQL files that define one output table or view. Models reference each other with `{{ ref() }}`, and dbt builds a dependency DAG automatically.

```
dbt_project/
  models/
    staging/          ← 1-to-1 source tables, light renaming + type casting
      stg_crm_customers.sql
      stg_erp_orders.sql
    intermediate/     ← business logic, multi-source joins
      int_customer_orders.sql
    marts/            ← final Gold tables: star schema dims and facts
      dim_customer.sql
      fact_sales.sql
```

Each layer builds on the previous:

```sql
-- models/marts/dim_customer.sql
SELECT
    {{ dbt_utils.surrogate_key(['CustomerBK']) }} AS CustomerKey,
    CustomerBK,
    CustomerName,
    CountryCode
FROM {{ ref('int_customer_orders') }}
```

`{{ ref('int_customer_orders') }}` tells dbt to build `int_customer_orders` first — dbt resolves the full dependency graph and runs models in the correct order.

## dbt tests

Tests run after each model build and fail the pipeline if violated:

```yaml
# models/marts/schema.yml
models:
  - name: dim_customer
    columns:
      - name: CustomerKey
        tests:
          - not_null
          - unique
      - name: CountryCode
        tests:
          - not_null
          - accepted_values:
              values: ['CZ', 'SK', 'DE', 'PL', 'HU']
      - name: CustomerBK
        tests:
          - not_null
          - relationships:
              to: ref('stg_crm_customers')
              field: CustomerBK
```

Custom tests can be written as SQL — any query that returns rows is a test failure.

## dbt on major platforms

| Platform | Adapter | Notes |
|---|---|---|
| **Snowflake** | `dbt-snowflake` | Best-supported; native `MERGE`, incremental models, clustering |
| **BigQuery** | `dbt-bigquery` | Native; uses `MERGE` for incremental; partition management via config |
| **Databricks** | `dbt-databricks` | Unity Catalog support; Delta-native; Liquid Clustering via config |
| **Microsoft Fabric** | `dbt-fabric` | Uses `dbt-synapse` adapter; Fabric Warehouse SQL dialect; growing support |
| **Redshift** | `dbt-redshift` | Mature; `DISTKEY`/`SORTKEY` config for performance |

For teams on Microsoft Fabric: the `dbt-fabric` adapter works but lags behind the Databricks and Snowflake adapters in feature completeness. ADF Data Flows or Fabric notebooks are often used alongside dbt for Fabric-specific operations.
