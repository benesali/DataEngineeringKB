# Medallion Architecture

The Medallion architecture is the dominant pattern for modern data lakehouses. It organizes data into three progressive layers, each with a specific quality and purpose.

## In this section

| Page | What you'll learn |
|---|---|
| [Bronze Layer](bronze.md) | Raw ingestion — data as it arrives from sources |
| [Silver Layer](silver.md) | Cleansing, standardization, conforming |
| [Gold Layer](gold.md) | Business-ready serving — star schemas, aggregations, OBT |
| [Medallion vs Traditional DWH](vs-traditional.md) | How Medallion maps to Inmon / Kimball / Data Vault |

## The three layers

```
Sources ──► Bronze (raw) ──► Silver (clean) ──► Gold (serve)
             immutable        standardized       business-ready
```

| Layer | Aka | Purpose | Data state |
|---|---|---|---|
| Bronze | Raw, Landing | Capture everything from sources | Original format, no transformation |
| Silver | Cleansed, Conformed | Fix, standardize, deduplicate | Source-aligned but clean |
| Gold | Serving, Curated | Answer business questions | Modeled for consumption |

## Key reference

This section draws heavily from the O'Reilly book and its companion code:

> **Building Medallion Architectures** — Piethein Strengholt  
> Code: [github.com/pietheinstrengholt/building-medallion-architectures-book](https://github.com/pietheinstrengholt/building-medallion-architectures-book)
