# Data Mesh vs Medallion Architecture

## They are not alternatives

Data mesh and Medallion are frequently described as competing "modern data architecture"
patterns. They are not — they address **different dimensions** of a data platform and compose
naturally.

| | Medallion | Data mesh |
|---|---|---|
| **What it solves** | How to organize data transformation layers | Who owns and is responsible for data |
| **Unit of concern** | Bronze / Silver / Gold layers | Domain + data product |
| **Owned by** | Typically one central platform team | Each business domain |
| **Prescribes** | Technical layering and quality progression | Organizational structure and ownership model |
| **Can exist without the other** | Yes — most Medallion platforms are centrally owned | Yes — a data mesh can use any internal storage pattern |

## How they compose

In a data mesh, **each domain implements its own medallion internally**. The data product is
what gets published at the Gold layer. Other domains consume that Gold output as their own
Bronze input.

```
Domain: Payments
  Bronze  ◄── raw events ingested from the operational system
  Silver  ◄── validated, deduplicated, conformed to domain schema
  Gold    ◄── data product published to the mesh
               │  (governed by a data contract)
               ▼
Domain: Orders
  Bronze  ◄── Payments Gold is Orders' Bronze input
  Silver  ◄── Orders-internal validation and enrichment
  Gold    ◄── Orders data product published to the mesh
```

This creates a **fractal medallion pattern**: Bronze → Silver → Gold repeats at the domain
level, and data flows between domains through published Gold-layer products rather than through
shared Silver tables that multiple teams touch simultaneously.

## The key difference from traditional (centralized) Medallion

In a traditional Medallion platform, a **central team owns all three layers for all domains**:

```
Central data platform team owns:
  Payments Bronze → Silver → Gold
  CRM Bronze      → Silver → Gold
  Fleet Bronze    → Silver → Gold
  (all in one pipeline, one team, one bottleneck)
```

This is precisely the centralized ownership model data mesh identifies as the bottleneck at
scale. Data mesh does not discard the Bronze/Silver/Gold pattern — it distributes ownership
of the layers to domain teams and shifts the platform team's role to providing the
infrastructure that each domain uses to run its own layers.

## Where the seam between domains is

The boundary between domains is the **Gold → Bronze handoff**: one domain's Gold layer is
another domain's Bronze input. The [data contract](data-contracts.md) governs that seam — it
defines what the consuming domain can depend on from the producing domain.

```
Payments Gold (data product)
  │  governed by data contract v2.1
  ▼
Orders Bronze (input from Payments)
```

## Layer mapping

| Medallion layer | Role in a centralized platform | Role in a data mesh |
|---|---|---|
| **Bronze** | Central team ingests raw data from all sources | Each domain team ingests raw data for its own sources |
| **Silver** | Central team cleanses and conforms all domains | Each domain team cleanses its own data |
| **Gold** | Central team builds marts for consumers | **The published data product** — the domain's interface to the mesh |

## Migration implication

If your platform uses Medallion today with centralized ownership, a migration toward data mesh
does not discard the Bronze/Silver/Gold structure. It **distributes ownership of the layers** to
domain teams. The layering stays; the org model changes.

The safe incremental path: identify domains with clear ownership boundaries, stand up their own
Bronze/Silver/Gold pipelines, define a Gold-layer data contract, and route other domains to
consume the contract rather than shared Silver tables. Repeat per domain.
