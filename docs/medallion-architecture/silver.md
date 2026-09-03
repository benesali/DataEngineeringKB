# Silver Layer (Cleansed)

> *Source: Building Medallion Architectures — Piethein Strengholt.  
> Code examples: [chapter-06](https://github.com/pietheinstrengholt/building-medallion-architectures-book/tree/main/chapter-06)*

## Purpose

The Silver layer **refines, cleanses, and standardizes** Bronze data. It is the single clean, source-aligned version of the data — ready for further integration and analytical use, but still structured per source system.

## Key characteristics

- **Clean** — duplicates removed, nulls handled, formats standardized
- **Source-aligned** — tables still correspond to source objects (one Silver table per source entity)
- **No business integration** (generally) — data from CRM and ERP are still separate tables
- **Current records** — Silver typically holds current state; SCD2 history is built in Gold

## Cleansing activities

| Activity | Example |
|---|---|
| Remove duplicates | One customer record per CustomerID |
| Handle nulls | Replace NULL phone with empty string, or flag and quarantine |
| Standardize formats | All dates as `YYYY-MM-DD`, all countries as ISO codes |
| Trim whitespace | `'  London  '` → `'London'` |
| Correct data types | String `'2024-01-15'` → DATE `2024-01-15` |
| Fix ranges | Age cannot be negative or > 150 |
| Mask PII | Hash or tokenize personal identifiers |
| Anomaly detection | Flag sudden 10x spike in row count |

Rejected / invalid records go to a **quarantine table** (sibling to the Silver table) — never deleted.

## Data model in Silver

Silver tables generally match the Bronze structure plus added quality columns. Column names may be renamed to apply consistent conventions.

**What Silver is NOT the place for:**
- Business rules and calculations
- Joining data from multiple source systems
- Creating star schemas or aggregations

Those belong in Gold.

## Advanced Silver patterns

### 3NF or Data Vault in Silver

Some organizations use Silver for **integration** — combining sources into a normalized or vault model. This makes sense when:
- You have many sources feeding the same entity (customer in CRM + ERP)
- You need full audit history before Gold
- You want a reusable integration layer for multiple Gold consumers

The trade-off: more complexity in Silver, and denormalized Spark workloads may be slower than wide tables.

### SCD2 in Silver

Most engineers recommend putting SCD2 in Gold. Silver should hold current records.

**Exception:** if Silver serves operational reporting or ML features that need historical context in source-aligned form.

## Automation

Silver is where **metadata-driven frameworks** shine — once Bronze data is in Delta format, you can auto-generate Silver transformation jobs from a metadata repository (schema, quality rules, key columns, mapping rules).

## Code reference

- Cleaning: [`clean_data.ipynb`](https://github.com/pietheinstrengholt/building-medallion-architectures-book/blob/main/chapter-06/clean_data.ipynb)
- Quarantine: [`clean_data_quarantine.ipynb`](https://github.com/pietheinstrengholt/building-medallion-architectures-book/blob/main/chapter-06/clean_data_quarantine.ipynb)
- SCD2 historization: [`historize_data_scd2.ipynb`](https://github.com/pietheinstrengholt/building-medallion-architectures-book/blob/main/chapter-06/historize_data_scd2.ipynb)

## Column naming convention in Silver

Silver is where raw source column names are standardized to platform conventions. Apply these transforms as part of the Bronze→Silver cleansing job:

| Source field | Silver column | Rule |
|---|---|---|
| `customer_id` | `CustomerBK` | Rename to `_BK` suffix; identify as business key |
| `cust_nm` | `CustomerName` | Expand abbreviations; use PascalCase |
| `upd_dt` | `UpdatedAt` | Expand; use consistent timestamp suffix |
| `cntry_cd` | `CountryCode` | Expand abbreviations |
| `is_active` | `IsActive` | Keep `Is` prefix for booleans |

Add system columns during Silver write:
- `SysCreDT` — timestamp this row was first loaded into Silver
- `SysUpdDT` — timestamp this row was last updated
- `_source_file` — source file name (inherited from Bronze)

## Quarantine table pattern

Every Silver table has a sibling quarantine table — same schema plus a `QuarantineReason` column. Invalid rows go to quarantine instead of being silently dropped or blocking the pipeline.

```sql
-- Silver target
CREATE TABLE silver.customers (
    CustomerBK      VARCHAR(50)   NOT NULL,
    CustomerName    VARCHAR(200)  NOT NULL,
    CountryCode     CHAR(2)       NULL,
    SysCreDT        DATETIME2(6)  NOT NULL
);

-- Quarantine sibling (same columns + reason)
CREATE TABLE silver.customers_quarantine (
    CustomerBK      VARCHAR(50)   NULL,
    CustomerName    VARCHAR(200)  NULL,
    CountryCode     CHAR(2)       NULL,
    SysCreDT        DATETIME2(6)  NOT NULL,
    QuarantineReason VARCHAR(500) NOT NULL,
    QuarantineDT    DATETIME2(6)  NOT NULL
);
```

In PySpark:

```python
from pyspark.sql.functions import lit, current_timestamp

valid   = df.filter(col("CustomerBK").isNotNull() & (length(col("CountryCode")) == 2))
invalid = df.filter(~(col("CustomerBK").isNotNull() & (length(col("CountryCode")) == 2))) \
            .withColumn("QuarantineReason", lit("CustomerBK is null or CountryCode invalid")) \
            .withColumn("QuarantineDT", current_timestamp())

valid.write.format("delta").mode("append").saveAsTable("silver.customers")
invalid.write.format("delta").mode("append").saveAsTable("silver.customers_quarantine")
```

## Metadata-driven Silver

Once Bronze tables are in Delta format, a metadata repository (table, YAML, or config file) can drive Silver generation automatically:

| Metadata field | Purpose |
|---|---|
| `source_table` | Which Bronze table to read |
| `target_table` | Which Silver table to write |
| `business_key` | Column(s) identifying unique entities |
| `quality_rules` | List of column-level checks to apply |
| `column_map` | Source → Silver column rename map |
| `scd_type` | 1 = overwrite, 2 = historize |

Frameworks: **dbt** (SQL-based, model-per-table), **Delta Live Tables** (Databricks, declarative pipeline), **Azure Data Factory** (parameter-driven Copy + Data Flow).
