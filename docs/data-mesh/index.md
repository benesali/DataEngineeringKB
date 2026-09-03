# Data Mesh

Data mesh is an **organizational and architectural pattern** for large-scale data platforms. Its
central claim: the bottleneck in modern data systems is not the technology — it is the
organizational model that concentrates data ownership, production, and transformation into one
central team.

Proposed by Zhamak Dehghani (ThoughtWorks) in 2019, expanded in her book *Data Mesh* (O'Reilly,
2022). Sources: *Data Engineering Podcast* ep. 90 and datamesh-architecture.com.

## In this section

| Page | What you'll learn |
|---|---|
| [The Four Principles](principles.md) | Domain ownership, data as a product, self-serve platform, federated governance |
| [Data Contracts](data-contracts.md) | How to make "data as a product" operational and testable |
| [Data Mesh vs Medallion](vs-medallion.md) | How the two patterns compose — they are not alternatives |
| [Readiness & Implementation Reality](readiness.md) | When to use data mesh, the 10-dimension checklist, real-world failure modes |

## The core shift

Traditional data platforms centralize ownership:

```
Domain A data ─┐
Domain B data ─┤──► Central data team ──► Data lake / warehouse ──► Consumers
Domain C data ─┘
```

Data mesh distributes it:

```
Domain A ──► Data product A ─┐
Domain B ──► Data product B ─┤──► Mesh (governed, discoverable) ──► Consumers
Domain C ──► Data product C ─┘
```

The central team's role shifts from **owning the data** to **owning the platform** that enables
each domain to produce and publish its own data products.

## What data mesh is not

- It is **not a technology** — no single tool or platform implements data mesh
- It is **not a replacement for Medallion** — they compose (see [Data Mesh vs Medallion](vs-medallion.md))
- It is **not for every organization** — it solves the problems of scale and domain multiplicity;
  see [Readiness & Implementation Reality](readiness.md)
