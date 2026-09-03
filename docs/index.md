# Data Engineering Knowledge Base

A practical reference for people starting out in data engineering — or anyone who wants a clear, concise explanation of the foundational concepts.

This knowledge base covers the full spectrum from **classical data warehousing** (Inmon, Kimball, Data Vault) through **modern lakehouse patterns** (Medallion architecture, Delta Lake) to **scalability and best practices**.

---

## Where to start

| I want to understand... | Go here |
|---|---|
| What data engineering actually is | [Getting Started](getting-started/index.md) |
| The classic DWH approaches (Inmon, Kimball) | [Traditional DWH](traditional-dwh/index.md) |
| Star schemas, SCDs, surrogate keys | [Data Modeling](data-modeling/index.md) |
| Bronze / Silver / Gold layers | [Medallion Architecture](medallion-architecture/index.md) |
| Cloud warehouses, lakehouses, Delta Lake | [Modern Warehousing](modern-warehousing/index.md) |
| Partitioning, Spark, incremental loads | [Scalability](scalability/index.md) |
| Naming, idempotency, data quality | [Best Practices](best-practices/index.md) |

---

## Key reference material

- **Medallion Architecture book** (code + notebooks): [github.com/pietheinstrengholt/building-medallion-architectures-book](https://github.com/pietheinstrengholt/building-medallion-architectures-book)

---

## How this is organized

```
Traditional DWH  ──►  Data Modeling  ──►  Medallion / Modern  ──►  Scalability
     (why)                (how)              (patterns)              (at scale)
```

Reading left to right gives you a logical progression — but each section also stands alone.
