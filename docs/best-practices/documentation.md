# Documentation Standards

## What to document

| Artifact | Minimum documentation |
|---|---|
| Table | Purpose, grain (what one row represents), refresh schedule, owner |
| Column | Definition, data type, nullable, any business rules applied |
| Pipeline | Source, destination, trigger, dependencies, expected run time |
| Data mart / star schema | Business process, conformed dimensions used, grain |
| Data quality rule | What it checks, threshold, action on failure |

## The grain statement

Every fact table must have an explicit grain statement:

> "One row in `FactSales` represents one order line, identified by `OrderID` + `LineNumber`."

Without a grain statement, the table is ambiguous — future engineers will add measures that violate the grain without realizing it.

## Where documentation lives

| Type | Where |
|---|---|
| Table/column definitions | Data catalog (Purview, Alation, DataHub, dbt docs) |
| Pipeline logic | Code comments in transformation SQL / notebooks |
| Architecture decisions | Confluence / wiki — record *why*, not just *what* |
| Data model diagrams | ERD in the catalog or wiki |
| Runbooks / playbooks | Ops wiki — what to do when the pipeline fails |

## What NOT to document in code comments

- What the code does — well-named identifiers already say that
- Which ticket required this — that belongs in the PR description and the git log
- When it was added — git blame answers that

**Only comment the WHY:** a non-obvious constraint, a workaround for a specific bug, behavior that would surprise a future reader.

## Data catalog integration

A data catalog auto-discovers tables and columns but relies on humans to fill in descriptions, grain, ownership, and quality rules. Without active curation, the catalog becomes a useless list of table names.

Minimum curation per table:
1. Description (one sentence)
2. Grain statement
3. Owner (team or person)
4. Refresh schedule
5. Source systems

## dbt docs

dbt auto-generates documentation from model SQL files, YAML descriptions, and test definitions. Every column description, model purpose, and test result is compiled into a searchable static site.

```bash
# Generate the docs site
dbt docs generate

# Serve locally at http://localhost:8080
dbt docs serve
```

To add descriptions (they appear in the docs site and data catalog integrations):

```yaml
# models/marts/schema.yml
models:
  - name: dim_customer
    description: "One row per unique customer. SCD2 — history preserved via ValidFrom/ValidTo."
    columns:
      - name: CustomerKey
        description: "Surrogate key. SHA-256 hash of CustomerBK."
      - name: CustomerBK
        description: "Business key from source CRM (Salesforce AccountID)."
      - name: IsCurrent
        description: "1 = this is the current active row for this customer."
```

dbt docs integrate with **dbt Cloud**, **Atlan**, and **Microsoft Purview** — column descriptions written once in YAML propagate automatically to the catalog.

## Microsoft Purview — glossary terms and asset annotations

Purview has two documentation mechanisms:

**Glossary terms** — business vocabulary definitions, independent of any specific asset:

```
Term: "Customer Lifetime Value"
Definition: "Total revenue attributed to a customer from first to last transaction."
Steward: Data Platform team
Related assets: gold.DimCustomer, gold.FactSales
```

**Asset annotations** — applied directly to a table or column in the catalog:

| Annotation | Purpose |
|---|---|
| Description | What this asset is (mirrors the grain statement) |
| Owner | Person or team responsible |
| Sensitivity label | Classification (Public, Internal, Confidential, Highly Confidential) |
| Certified | This asset is trusted and governed |
| Glossary term assignment | Links a column to a business term |

In Fabric, the SQL Warehouse auto-registers tables and columns in Purview via native scanning. Descriptions added via `COMMENT ON TABLE` or through the Fabric portal propagate to Purview without manual entry.

## Data lineage as living documentation

Manual documentation drifts. Lineage derived from pipeline execution or code parsing stays current automatically.

**Purview native lineage:** ADF pipelines, Fabric notebooks, and SQL stored procedures registered in Purview create column-level lineage automatically — showing that `gold.DimCustomer.CustomerName` flows from `bronze.crm_account.name` through `silver.crm_account.CustomerName`.

**dbt lineage:** `dbt docs generate` builds a full DAG of model dependencies. Every `{{ ref('stg_crm_customers') }}` reference is a lineage edge — source → staging → intermediate → mart, all visualized in the docs site.

**Practical impact:** when a source column is renamed or removed, lineage tells you exactly which downstream models, reports, and consumers are affected — before the pipeline breaks.
