# Data Quality

> *Additional source: Paulraj Ponniah, "Data Warehousing Fundamentals for IT Professionals" (2010), Ch. 13*

## Why it matters

Bad data discovered in a dashboard by a VP is ten times more expensive to fix than bad data caught at ingestion. A data quality strategy pushes validation as early in the pipeline as possible.

## Dimensions of data quality

The core six dimensions used in most modern frameworks:

| Dimension | Question | Example check |
|---|---|---|
| **Completeness** | Is required data present? | `CustomerName IS NOT NULL` |
| **Uniqueness** | Are there duplicates? | `COUNT(*) = COUNT(DISTINCT CustomerID)` |
| **Validity** | Is data in the expected format/range? | `Age BETWEEN 0 AND 150` |
| **Consistency** | Is data consistent across sources? | Same customer has same name in CRM and ERP |
| **Timeliness** | Did data arrive on time? | Max `LoadDate` is within 2 hours of expected |
| **Accuracy** | Does data reflect reality? | Requires external reference (hard to automate) |

### Extended data quality dimensions (Ponniah taxonomy)

A more granular breakdown covering additional DWH-specific aspects:

| Dimension | What it tests |
|---|---|
| **Accuracy** | Values correctly represent the real-world fact |
| **Domain Integrity** | Values fall within the allowed domain (enum, range, format) |
| **Data Type** | Values match the declared data type (no strings in numeric fields, no invalid dates) |
| **Consistency** | Same entity represented identically across tables and source systems |
| **Redundancy** | Duplicate data is managed — either eliminated or synchronized |
| **Completeness** | All expected rows and columns are present; no missing mandatory values |
| **Duplication** | No unintended duplicate records exist |
| **Conformance to Business Rules** | Values meet defined business constraints (e.g. ship date ≥ order date) |
| **Structural Definiteness** | The data structure is well-defined and documented; no ambiguous or overloaded fields |
| **Data Anomaly** | Statistical outliers or unexpected patterns are detectable |
| **Clarity** | Values are unambiguous and interpretable without tribal knowledge |
| **Timeliness** | Data is available when expected; no processing delays beyond SLA |
| **Usefulness** | Data serves the intended analytical purpose; not just technically correct but business-relevant |
| **Adherence to Data Integrity Rules** | Referential integrity holds; PK uniqueness enforced; FK constraints satisfied |

## Types of data quality problems

Problems found in source data before or during loading:

| Problem type | Description | Example |
|---|---|---|
| **Dummy values** | Placeholder values used when real data is unavailable | `9999`, `N/A`, `UNKNOWN`, `0000-00-00` |
| **Absence of values** | Required fields left null or blank | No postal code, no customer name |
| **Unofficial field use** | A field repurposed for something other than its intended use | Storing notes in the "middle initial" field |
| **Cryptic values** | Coded values with no documentation or lookup | Status = `X`, `Q`, `Z2` with no meaning table |
| **Contradicting values** | Two fields in the same record conflict | `BirthDate = 1980` but `Age = 15` |
| **Business rule violations** | Data that cannot logically be correct | `ShipDate < OrderDate` |
| **Reused primary keys** | Old records deleted; their PKs recycled for new entities | CustomerID 4421 was ACME Corp (deleted), now refers to Beta Ltd |
| **Nonunique identifiers** | The same PK appears on multiple distinct records | Two different customers both have `CustomerID = 7710` |
| **Inconsistent values** | Same entity described differently across systems | "Germany" in one system, "DE" in another, "Deutschland" in a third |
| **Incorrect values** | Values that are technically valid but factually wrong | Wrong country code assigned to a customer address |
| **Multipurpose fields** | One column serving multiple meanings depending on context | `Field27` means "discount" for orders, "adjustment" for returns |
| **Erroneous integration** | Bad joins or merges producing composite records mixing data from different entities | Customer records merged based on name match, but same name = different companies |

---

## Sources of data pollution

Where bad data comes from:

| Source | Explanation |
|---|---|
| **System conversions** | Migrating from one system to another corrupts data (encoding changes, truncation, lost mappings) |
| **Data aging** | Data becomes stale — addresses, phone numbers, prices change but records aren't updated |
| **Heterogeneous system integration** | Different systems define the same entity differently; integration introduces inconsistencies |
| **Poor database design** | Nullable columns where NOT NULL was required; missing constraints; overloaded fields |
| **Incomplete information at data entry** | Users skip optional fields that the business later discovers they need |
| **Input errors** | Typos, transpositions, wrong codes — especially in manually keyed systems |
| **Internationalization** | Different date formats, character encodings, number formats across countries |
| **Fraud** | Intentionally false data entered to manipulate outcomes |
| **Lack of policies** | No standards for naming, coding, or validation mean each user invents their own conventions |

---

## Data quality framework and purification process

A structured approach rather than ad-hoc fixes:

### Steering committee

A cross-functional committee (executive sponsor + business domain owners + IT leads) provides governance:
- Defines which data quality dimensions matter most for the organization
- Prioritizes which data quality problems to fix first
- Owns the data quality policy and enforcement mechanisms

### Priority classification

Fix high-priority issues before loading; defer low-priority issues:

| Priority | Criteria | Action |
|---|---|---|
| **High** | Directly affects business decisions or regulatory reporting | Fix in source or in ETL before loading to DWH |
| **Medium** | Affects operational reports or causes analyst confusion | Fix in ETL; quarantine affected rows; alert owner |
| **Low** | Cosmetic or edge-case issues | Log and document; fix in next maintenance window |

### Purification approach

**Preferred: clean before loading.** Identify and fix data quality issues in the staging/cleansing layer before data enters the DWH. This ensures the DWH is authoritative.

**Alternative: clean as you go.** Fix issues in the BI layer (calculated fields, report-time corrections). Simpler upfront, but spreads correction logic across many reports and creates divergent "versions of the truth."

### Tools

- **Specialized DQ tools:** Informatica Data Quality, IBM InfoSphere QualityStage, Talend Data Quality — profile data, detect issues, apply standardization rules
- **In-house programs:** custom SQL/Python scripts targeting specific known problems; simpler but must be maintained manually
- **Metadata-driven correction:** store correction rules in a metadata table; apply them programmatically during ETL

---

## Where to check

| Layer | What to check | Action on failure |
|---|---|---|
| Bronze | Technical validity (file parseable, columns present, row count in range) | Quarantine file / alert |
| Silver | Business validity (nulls, ranges, duplicates, referential integrity) | Quarantine rows / flag |
| Gold | Aggregation plausibility (revenue not zero, KPIs in expected range) | Alert / block report refresh |

## Quarantine pattern

Separate valid and invalid records at Silver — invalid rows go to a sibling quarantine table.

```
silver.customers          ← valid rows only
silver.customers_quarantine ← failed rows + reason column
```

This lets the pipeline continue without blocking on bad data, while keeping a full audit trail of what was rejected.

## dbt tests

dbt provides built-in data quality tests that run as part of every build:

```yaml
models:
  - name: dim_customer
    columns:
      - name: CustomerKey
        tests:
          - not_null
          - unique
      - name: Country
        tests:
          - accepted_values:
              values: ['CZ', 'SK', 'DE', 'PL']
      - name: CustomerBK
        tests:
          - not_null
          - relationships:
              to: ref('stg_crm_customers')
              field: CustomerBK
```

## Data quality frameworks

| Tool | Notes |
|---|---|
| **dbt tests** | SQL-based; runs at transform time; open-source |
| **Great Expectations** | Python library; rich expectation catalog; can run at any layer |
| **Delta Lake constraints** | Database-level `CHECK` constraints enforced on every write |
| **Soda** | SaaS + open-source; integrates with dbt, Airflow |

## Delta Lake CHECK constraints

Delta Lake (3.0+) supports database-level CHECK constraints that are enforced on every write:

```sql
-- Add constraint to Silver table
ALTER TABLE silver.customers
ADD CONSTRAINT valid_country CHECK (CountryCode IN ('CZ', 'SK', 'DE', 'PL', 'HU', 'AT'));

ALTER TABLE silver.orders
ADD CONSTRAINT positive_amount CHECK (SalesAmount >= 0);

-- List constraints
SHOW TBLPROPERTIES silver.customers;
-- delta.constraints.valid_country = CountryCode IN (...)
```

Any INSERT or UPDATE that violates the constraint raises an error and rolls back the write. This is the strictest form of DQ enforcement — the bad data never reaches the table.

Use constraints for absolute invariants (a negative SalesAmount is always wrong). Use quarantine tables for business rules that might have legitimate exceptions.

## Great Expectations quickstart

Great Expectations (GX) is a Python library for defining and running data quality assertions ("expectations") on DataFrames or tables.

```python
import great_expectations as gx

context = gx.get_context()

# Define a batch of data to validate
batch = context.sources.pandas_default.read_dataframe(df)

# Define expectations
batch.expect_column_to_exist("CustomerBK")
batch.expect_column_values_to_not_be_null("CustomerBK")
batch.expect_column_values_to_be_unique("CustomerBK")
batch.expect_column_values_to_be_in_set("CountryCode", ["CZ", "SK", "DE"])
batch.expect_table_row_count_to_be_between(min_value=1000, max_value=10_000_000)

# Run all expectations
results = batch.validate()
print(results.success)  # True if all expectations passed
```

GX integrates with Airflow, dbt, and Spark — run validations as a step in the pipeline before writing to the target.

## Alerting on DQ failures

DQ failures are only useful if someone is notified quickly. Standard alerting patterns:

```python
def send_teams_alert(message: str, webhook_url: str):
    import requests
    requests.post(webhook_url, json={"text": message})

# In pipeline after DQ check:
if quarantine_count > 0:
    send_teams_alert(
        f"⚠️ DQ Alert: {quarantine_count} rows quarantined in silver.customers "
        f"at {datetime.now()}. Check silver.customers_quarantine.",
        webhook_url=os.getenv("TEAMS_DQ_WEBHOOK")
    )
```

For operational pipelines: integrate DQ alerts with the same channel as pipeline failure alerts. A DQ failure that silently quarantines 80% of rows is as serious as a pipeline crash.

Fabric / Databricks: use Data Activator (Fabric) or Databricks Alerts on a monitoring query for threshold-based alerting without custom code.

## Row count anomaly detection

Statistical bounds catch unexpected changes in data volume — useful when you don't know the exact expected count but know the historical pattern.

```python
import pandas as pd
from scipy import stats

# Load historical row counts for this load
history = spark.sql("""
    SELECT LoadDate, RowCount
    FROM SystemLog.TS_LoadStats
    WHERE TableName = 'silver.orders'
    ORDER BY LoadDate DESC
    LIMIT 30
""").toPandas()

mean = history["RowCount"].mean()
std  = history["RowCount"].std()
today_count = df.count()

z_score = (today_count - mean) / std
if abs(z_score) > 3:  # 3 standard deviations = ~0.3% chance if normal
    send_alert(f"Row count anomaly: {today_count} rows (expected ~{mean:.0f} ± {std:.0f})")
```

This catches both sudden drops (source feed failed, partial extract) and sudden spikes (duplicate load, incorrect filter removed).
