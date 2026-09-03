# Traditional Data Warehousing

Before cloud lakehouses and medallion layers, data warehouses were built on three foundational methodologies — each with a different philosophy about how to model and organize data.

## In this section

| Page | What you'll learn |
|---|---|
| [Inmon (3NF / Enterprise DWH)](inmon.md) | Top-down, normalized, single source of truth |
| [Kimball (Dimensional Modeling)](kimball.md) | Bottom-up, star schemas, business-process focus |
| [Data Vault](data-vault.md) | Hybrid — normalized core + historized satellites |
| [Inmon vs Kimball vs Data Vault](comparison.md) | When to use which, trade-offs at a glance |

## The key tension

Inmon says: build a normalized enterprise model first, derive data marts from it.  
Kimball says: build dimensional models per business process, iterate fast.  
Data Vault says: make the core auditable and flexible, denormalize only at the end.

All three are still in use today — often in combination.
