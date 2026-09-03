# Schema Evolution

> *Additional source: Claudia Imhoff et al., "Mastering Data Warehouse Design" (2003), Ch. 10–11*

## Why the DWH data model changes

Change in a data warehouse is inevitable. Reasons fall into three categories:

**Business reasons:**
- Business growth (new products, new markets, new customer segments)
- Business contraction (discontinued products, market exit)
- Regulatory requirements (new reporting mandates)
- Mergers and acquisitions (new entities, new source systems)

**Operational reasons:**
- New data sources become available
- Source system upgrades change data formats or keys
- Data quality issues require structural fixes

**Organizational reasons:**
- Reorganizations change the reporting hierarchy
- New owners request different views of the data
- Data stewardship decisions redefine entities

The data warehouse must accommodate all of these without breaking existing consumers.

---

## Data model hierarchy

Changes propagate through four levels of data model, top-down:

| Level | What it contains | Who maintains it |
|---|---|---|
| **Subject Area Model** | High-level entities and their business relationships; no attributes | Business analysts, executives |
| **Business Data Model** | All business entities, attributes, and relationships; business-defined | Business analysts + DWH architects |
| **System Data Model** | Technical entities derived from business model; surrogate keys, temporal attributes, integration keys | DWH architects |
| **Technology Data Model** | Physical DDL: tables, columns, data types, indexes, partitions | DWH developers |

A change at the business level (new business rule) must be reflected down through all four levels. A change at the technology level (data type optimization) does not require business-level review. The hierarchy determines where to start the change request.

**Synchronization implication:** if the business model and system model fall out of sync (a new attribute exists at business level but not yet in the system model), the DWH cannot be built correctly. Governance must ensure each level is updated before the next level is changed.

---

## Designing for change: assume worst case

A common mistake is designing for the stated current business rule: "A customer always has exactly one billing address." Design for the worst case instead:

- "Always" → assume it will become "sometimes" — use a one-to-many relationship from the start, not a single column
- "Never" → assume it will become "rarely" — leave the door open for the exception

This one principle eliminates the most disruptive type of schema change: upgrading a single-value attribute to a multi-value relationship (which requires table restructuring, not just column addition).

---

## Relationship generalization: designing for multiple relationship types

When a business currently has one specific relationship type (e.g. "a customer has one primary contact"), but may later need to track multiple relationship types (primary contact, billing contact, technical contact, emergency contact):

**Specific model (brittle):**
```sql
Customer (CustomerID, PrimaryContactID)
```

**Generalized model (flexible):**
```sql
Customer (CustomerID, ...)
CustomerContact (CustomerID, ContactID, ContactRoleType, CurrentFlag, InferredFlag)
```

The associative entity `CustomerContact` holds:
- **ContactRoleType** — the type of relationship (primary, billing, technical)
- **CurrentFlag** — is this the active relationship of this type? Allows tracking that a role exists but the specific person may change
- **InferredFlag** — was this relationship inferred/assumed (because the source didn't provide it explicitly), or confirmed from source data?

When a new contact role type emerges, add a new row — no schema change required. Without generalization, every new role would require an ALTER TABLE.

---

## Surrogate keys as change isolation

Surrogate keys insulate the DWH from natural key changes in source systems:

- Source re-platforms and reassigns CustomerID values → the DWH CustomerKey is unchanged; the business key column is updated, but all fact table FK references remain valid
- Two systems merge and must unify their customer numbering → the DWH absorbs the merge by updating the business key; surrogate keys in fact tables are untouched
- A source key that was a string becomes an integer in the new system → the DWH surrogate key is still an integer; the business key column is updated to VARCHAR

Without surrogate keys, a natural key change cascades across every fact table that used that key — a costly and error-prone operation.

---

## Integrating vs adding subject areas

**Integrating subject areas:** two previously separate parts of the data model are found to be related (e.g. "Customer" in the sales domain and "Counterparty" in the finance domain are the same entity). Requires:
- Standardizing attributes (identical names, types, and business definitions)
- Inferring roles (a person record may be both a Customer and an Employee)
- Entity integration (resolving IDs, handling duplicates, assigning the unified surrogate key)

**Adding subject areas:** a new domain is onboarded (e.g. adding an HR subject area to a sales + finance DWH). Requires:
- Identifying integration points with existing subject areas (shared entities: Dates, Employees, Products)
- Conforming shared dimensions (ensure the new area uses the same DimDate, DimEmployee surrogate keys)
- Extending the Subject Area Model with the new subject area boundary

---

## Model governance

**Steering committee:** a cross-functional body (business + IT + data stewards) that approves significant data model changes. Prevents ad-hoc changes that break consumers or introduce redundancy.

**Data stewardship program:** ongoing ownership of data definitions, business rules, and naming standards. Each subject area has a designated steward responsible for approving changes to entities within their domain.

**Collision management:** when two modelers make conflicting changes to overlapping entities:
- Roles and responsibilities define who has authority over which subject areas
- Model access controls prevent unauthorized direct edits
- A formal comparison + incorporation process merges conflicting changes, with the steering committee resolving disagreements

---

## The problem

Source systems change. A new column appears in the CRM export. A field is renamed. A data type changes from integer to string. If your pipelines are rigid, these changes break everything.

Schema evolution is the set of practices and tools that let pipelines adapt to structural changes without full rewrites.

## Types of schema changes

| Change | Risk level | Notes |
|---|---|---|
| New column added to source | Low | Usually safe to add; existing queries unaffected |
| Column removed from source | Medium | Downstream consumers may expect it |
| Column renamed | High | Must update all consumers |
| Data type changed | High | May cause cast errors downstream |
| Column order changed | Low | If using column names (not positions) |

## Delta Lake schema evolution

### Automatic schema evolution (`mergeSchema`)

When writing a DataFrame with new columns to a Delta table, Delta Lake can automatically add those columns:

```python
df.write.format("delta") \
  .option("mergeSchema", "true") \
  .mode("append") \
  .save(path)
```

What Delta does automatically:
- **New column in source:** adds column to table; existing rows get NULL
- **Column in table but not source:** keeps column; new rows get NULL
- **Same column, different type:** tries to widen (e.g. INT → LONG); fails if incompatible

### Schema enforcement (default)

Without `mergeSchema`, Delta rejects any write that doesn't match the current schema. This is the safe default — unexpected changes fail loudly rather than silently corrupting data.

### ALTER TABLE

For intentional changes, use explicit DDL:

```sql
ALTER TABLE silver.customers ADD COLUMN LoyaltyTier VARCHAR(50)
ALTER TABLE silver.customers DROP COLUMN ObsoleteField
```

Note: **renaming** and **type changes that require rewrite** (e.g. VARCHAR(50) → VARCHAR(200)) may not be supported depending on your engine. In Fabric SQL Warehouse, `ALTER COLUMN` for type changes is not supported — use DROP + ADD instead.

## Handling schema changes in pipelines

### Column mapping (Delta Lake)

Delta Lake column mapping allows renaming and dropping columns at the metadata level without rewriting all Parquet files. This makes renames cheap.

```sql
ALTER TABLE my_table SET TBLPROPERTIES ('delta.columnMapping.mode' = 'name')
ALTER TABLE my_table RENAME COLUMN OldName TO NewName
```

### Schema-on-read as a buffer

Bronze stores data in original format (JSON, CSV). This defers the schema decision. When you add a Silver transformation, you parse the raw format and define the target schema explicitly — absorbing source changes at the Bronze→Silver boundary.

### Downstream impact assessment

When a source schema changes, you need to know every pipeline and table that depends on the affected column. A **data catalog** or **lineage tool** (e.g. Microsoft Purview) makes this traceable.

## Practical checklist for a schema change

1. Identify all consumers of the changed column (lineage query or catalog search)
2. Update the target DDL (e.g. `ALTER TABLE ADD COLUMN`)
3. Update the transformation logic (ETL mapping, dbt model, Silver notebook)
4. Run a test load on dev to verify no type errors
5. Release in order: DDL first → transformation second → consumers last

## dbt approach to schema changes

dbt provides **model contracts** — explicit column-level schema declarations that fail the build if the model's output doesn't match:

```yaml
# models/marts/schema.yml
models:
  - name: dim_customer
    config:
      contract:
        enforced: true  # dbt validates column names and types on each run
    columns:
      - name: CustomerKey
        data_type: int
        constraints:
          - type: not_null
          - type: primary_key
      - name: CustomerName
        data_type: varchar
        constraints:
          - type: not_null
```

When a source adds a new column that breaks the downstream model, the dbt build fails with a clear error — rather than silently producing wrong output. This is the dbt equivalent of schema enforcement at the transformation layer.

For a column rename in the source: update the staging model (`stg_crm_customers.sql`) to map the old name to the new one, keeping the Silver/Gold column names stable.

## Schema registry for Kafka / Avro

Kafka messages are often encoded in **Avro** format — a compact binary format with an embedded schema ID. The **Confluent Schema Registry** (or Azure Event Hubs Schema Registry) stores all schema versions and ensures producers and consumers agree on the message structure.

```
Producer (CRM) → serialize message as Avro with schema ID 42 → Kafka topic
Consumer (Bronze notebook) → reads schema ID 42 from Registry → deserializes correctly
```

When the CRM adds a new field:
1. A new schema version (ID 43) is registered — the registry validates that it's **backward compatible** (consumers using schema 42 can still read messages written with schema 43)
2. Producer starts writing messages with schema ID 43
3. Consumers with schema 42 see the new field as null (backward compatibility guarantee)

Compatibility modes: `BACKWARD` (new schema reads old messages), `FORWARD` (old schema reads new messages), `FULL` (both). Setting `BACKWARD` is the safe default for Bronze ingestion.

## Automated schema drift detection

Schema drift = the source sends a column that the pipeline doesn't expect (new column added, column removed, type changed).

Detection approaches:

| Approach | How | Tooling |
|---|---|---|
| **Metadata comparison** | Compare source schema against expected schema on each run; alert on diff | Custom script + alerting (Teams, PagerDuty) |
| **Delta Lake schema enforcement** | Reject writes with unexpected columns unless `mergeSchema=true` | Delta Lake native |
| **dbt source freshness + tests** | `dbt source freshness` checks if source was updated recently; column tests catch type changes | dbt |
| **Azure Data Factory schema drift** | ADF Data Flow has built-in schema drift detection — new columns can be automatically propagated or flagged | ADF |
| **Great Expectations profiling** | Profile source data; alert when column distribution changes significantly | Great Expectations |

A minimal viable detection: compare `INFORMATION_SCHEMA.COLUMNS` for the source view against the expected schema stored in a metadata table. Run this check as the first step in the pipeline; fail fast if drift is detected.
