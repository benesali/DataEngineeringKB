# Data Vault

## What it is

Data Vault is a modeling methodology developed by Dan Linstedt in the early 2000s. It sits between Inmon (fully normalized) and Kimball (fully denormalized), designed specifically for **enterprise-scale DWH** environments with many source systems, frequent schema changes, and strong auditability requirements.

## The three building blocks

### Hub
Stores **unique business keys** — one row per unique entity, regardless of source.

```sql
-- Example: Hub_Customer
Hub_Customer_SK   (surrogate key)
Customer_BK       (business key, e.g. CRM CustomerID)
LoadDate
RecordSource
```

### Link
Stores **relationships between hubs** — the connections between entities.

```sql
-- Example: Link_CustomerOrder
Link_SK
Hub_Customer_SK   (FK to Hub_Customer)
Hub_Order_SK      (FK to Hub_Order)
LoadDate
RecordSource
```

### Satellite
Stores **descriptive attributes** about a hub or link, with full history.

```sql
-- Example: Sat_Customer_CRM
Hub_Customer_SK
LoadDate          (start of validity)
LoadEndDate       (end of validity, NULL = current)
CustomerName
Email
Phone
HashDiff          (hash of all attributes — used to detect changes)
RecordSource
```

## Why the separation

| Structure | Changes when... | Risk |
|---|---|---|
| Hub | A new source introduces the same entity | Very low — just add a new row |
| Link | A new relationship is discovered | Low — add a new link table |
| Satellite | An attribute is added or changes | Low — add a column or new satellite |

Because hubs and links never change structure, schema evolution is isolated to satellites.

## Raw Vault vs Business Vault

- **Raw Vault** — direct, uninterpreted load from sources. No business rules applied. One satellite per source system.
- **Business Vault** — derived or calculated attributes, harmonized values, applied business rules. Lives alongside the raw vault.

## When Data Vault fits

- Many source systems feeding the same entities (customer exists in CRM, ERP, billing)
- Regulatory environments requiring a full audit trail of every change
- Agile DWH projects where the model evolves frequently
- When combining Inmon's integration discipline with Kimball's performance at the mart layer

## Relation to Medallion architecture

In a Medallion/Lakehouse setup, Data Vault typically occupies the **Silver layer** — raw vault in silver, business vault in silver/gold boundary, dimensional star schemas in Gold.

See [Silver Layer](../medallion-architecture/silver.md) and [Medallion vs Traditional DWH](../medallion-architecture/vs-traditional.md).

## Full worked example: Hub + Link + Satellite

A customer placing an order. Three entities: Customer, Order, Product.

```sql
-- Hub: unique customers by business key
CREATE TABLE Hub_Customer (
    Hub_Customer_SK  INT          NOT NULL PRIMARY KEY,
    Customer_BK      VARCHAR(50)  NOT NULL UNIQUE,  -- CRM CustomerID
    LoadDate         DATETIME2    NOT NULL,
    RecordSource     VARCHAR(100) NOT NULL
);

-- Hub: unique orders
CREATE TABLE Hub_Order (
    Hub_Order_SK  INT         NOT NULL PRIMARY KEY,
    Order_BK      VARCHAR(50) NOT NULL UNIQUE,
    LoadDate      DATETIME2   NOT NULL,
    RecordSource  VARCHAR(100) NOT NULL
);

-- Link: customer placed order (relationship)
CREATE TABLE Link_CustomerOrder (
    Link_SK          INT NOT NULL PRIMARY KEY,
    Hub_Customer_SK  INT NOT NULL REFERENCES Hub_Customer,
    Hub_Order_SK     INT NOT NULL REFERENCES Hub_Order,
    LoadDate         DATETIME2    NOT NULL,
    RecordSource     VARCHAR(100) NOT NULL
);

-- Satellite: customer attributes from CRM (versioned)
CREATE TABLE Sat_Customer_CRM (
    Hub_Customer_SK  INT          NOT NULL REFERENCES Hub_Customer,
    LoadDate         DATETIME2    NOT NULL,
    LoadEndDate      DATETIME2    NULL,      -- NULL = current
    CustomerName     VARCHAR(200) NOT NULL,
    Email            VARCHAR(200) NULL,
    CountryCode      CHAR(2)      NULL,
    HashDiff         CHAR(64)     NOT NULL,  -- SHA-256 of all attributes
    RecordSource     VARCHAR(100) NOT NULL,
    PRIMARY KEY (Hub_Customer_SK, LoadDate)
);
```

## Point-in-Time (PIT) tables

When a Hub has multiple satellites (from different sources or for different attribute groups), querying current state requires joining each satellite with `WHERE LoadEndDate IS NULL`. PIT tables pre-compute these joins for a set of snapshot dates.

```sql
-- PIT table: for each customer and snapshot date, which satellite row was current?
CREATE TABLE PIT_Customer (
    SnapshotDate     DATE         NOT NULL,
    Hub_Customer_SK  INT          NOT NULL,
    Sat_CRM_LoadDate DATETIME2    NULL,      -- NULL if no CRM row existed yet
    Sat_ERP_LoadDate DATETIME2    NULL,
    PRIMARY KEY (SnapshotDate, Hub_Customer_SK)
);
```

PIT tables make point-in-time reporting fast: join Hub → PIT → Satellites using the snapshot date to find the right satellite version without window functions on every query.

**Bridge tables** solve a similar problem for many-to-many relationships over time — e.g. which accounts did a customer own on a given date, when account membership changes.

## Data Vault 2.0 additions

| Feature | DV 1.0 | DV 2.0 |
|---|---|---|
| Business key hashing | Not required | `HASHBYTES('SHA2_256', BK)` as surrogate key — eliminates sequence-based PK lookups |
| Multi-active satellites | Not defined | Satellite with multiple current rows (e.g. multiple addresses per customer) — uses sequence or type discriminator |
| Effectivity satellites | Not defined | Explicitly tracks when a link relationship started/ended |
| Business vault | Informal | Formal layer for derived and calculated attributes alongside Raw Vault |
| NoSQL/Big Data sources | Not addressed | Explicit patterns for JSON, events, unstructured sources |

Hash-based PKs in DV 2.0 eliminate the need for a sequence lookup when loading — a new customer's Hub_Customer_SK is computed from `HASHBYTES('SHA2_256', Customer_BK)` deterministically, so parallel loads can assign the same key independently.

## Automation frameworks

| Tool | Notes |
|---|---|
| **VaultSpeed** | SaaS; generates Data Vault DDL and ETL code from business key definitions; integrates with Databricks, Snowflake, Synapse |
| **WhereScape** | Full DWH automation platform; supports DV patterns; generates code for most major platforms |
| **dbt + DV packages** | `dbt_datavault` / `dbtvault` open-source packages; SQL-macro-based Hub/Link/Satellite templates for dbt-supported platforms |
| **Matillion** | Low-code ELT; DV patterns via templates |

For teams starting with Data Vault, an automation framework typically pays for itself within weeks — hand-coding Hub/Link/Satellite DDL + ETL for 50+ entities is error-prone and slow.
