# Data Contracts

## What is a data contract?

"Data as a product" is aspirational until there is a mechanism that enforces it. A **data
contract** is that mechanism: a formal, versioned specification — typically YAML — that encodes
everything a consumer needs to depend on a data product.

```yaml
# example data contract (simplified)
name: payments.transactions
version: 2.1.0
owner: payments-domain-team
sla:
  freshness: 1h
  availability: 99.9%
schema:
  - name: transaction_id
    type: string
    nullable: false
  - name: amount
    type: decimal(18,2)
    nullable: false
  - name: currency_code
    type: string
    nullable: false
  - name: processed_at
    type: timestamp
    nullable: false
quality:
  completeness: 99.5%
  null_rate_transaction_id: 0%
retention: 3 years
access: payments-consumers group
```

## What a contract specifies

| Element | What it encodes |
|---|---|
| **Schema** | Field names, types, nullable constraints |
| **Quality SLAs** | Freshness, completeness thresholds, allowed null rates |
| **Retention** | How long data is kept and at what granularity |
| **Access terms** | Who can consume it and under what conditions |
| **Versioning** | Semantic version — consumers know when to expect breaking changes |
| **Owner** | Accountable team for this product |

## The contract as source of truth

The key architectural insight: the contract is the **source of truth**, not the pipeline code.
Everything derives from it:

```
Data Contract (YAML spec)
  ├──► DDL (table definition auto-generated from schema)
  ├──► Quality tests (assertions auto-generated from SLAs)
  ├──► CI/CD pipeline (validate contract on deploy)
  └──► Catalog entry (discoverability auto-populated from metadata)
```

A change to the contract is a change to the product's interface — it requires versioning and
consumer notification, the same as an API change in software.

## What contracts change

Without contracts, "data as a product" means: *we try to keep this data good and hope consumers
find it.* With contracts, a data product either passes its contract validation or it does not.
This makes quality **testable and automated**, not aspirational.

| Without contracts | With contracts |
|---|---|
| Quality is informal ("we try") | Quality is specified and validated in CI/CD |
| Schema changes are discovered by consumers breaking | Schema changes trigger versioning and consumer notification |
| Consumers don't know what to depend on | Consumers have a stable, versioned interface |
| Discoverability requires talking to the producing team | Catalog is auto-populated from contract metadata |

## Data contracts and the Gold layer

In a data mesh + medallion setup, the data contract lives on the **Gold layer output** — the
boundary where a domain publishes its data product to be consumed by other domains. See
[Data Mesh vs Medallion](vs-medallion.md) for how this fits.

## Further reading

- datamesh-architecture.com — contract-first workflow with Databricks Asset Bundles
- *Data Mesh* — Zhamak Dehghani (O'Reilly, 2022), Chapter on data product thinking
