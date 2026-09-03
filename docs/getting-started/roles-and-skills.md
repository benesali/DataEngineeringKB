# Key Roles & Skills

## Core skill areas

| Skill | Why it matters |
|---|---|
| **SQL** | Most analytical transformations are still SQL; essential for data modeling |
| **Python** | Pipeline logic, orchestration scripts, data quality checks |
| **Cloud platform** | Azure / AWS / GCP — storage, compute, managed services |
| **Data modeling** | Knowing *how* to structure data (star schema, vault, normalized) |
| **Pipeline orchestration** | ADF, Airflow, Prefect — scheduling and dependencies |
| **Distributed compute** | Spark for large-scale transformations |
| **Version control (git)** | All infrastructure and transformation code lives in git |

## Adjacent roles

| Role | Overlap with DE |
|---|---|
| Analytics Engineer | Focuses on the transform layer (dbt); less infra, more modeling |
| Data Architect | Designs the overall data platform strategy; less hands-on coding |
| MLOps Engineer | Specializes in serving ML models; uses DE pipelines as input |
| Data Analyst | Consumes what DE builds; SQL-heavy, reporting-focused |

## Learning path

Recommended sequence for a new data engineer:

1. **SQL** — master SELECT, JOIN, GROUP BY, window functions, CTEs. Everything in data engineering comes back to SQL.
2. **Data modeling basics** — star schema, fact vs dimension, SCD types. Understand *why* data is structured the way it is before building pipelines.
3. **Python** — scripting, pandas for small-data manipulation, basic OOP. Not necessary to be expert before starting.
4. **One cloud platform** — pick one (Azure/AWS/GCP) and learn its storage (blob/S3/GCS), compute, and managed DWH offering. Go deep on one before sampling others.
5. **Apache Spark / PySpark** — how distributed compute works; DataFrame API; partitions and shuffles; integration with Delta Lake.
6. **Pipeline orchestration** — Airflow, ADF, or Fabric pipelines. DAG dependencies, scheduling, retry logic.
7. **dbt** — if your team uses it; SQL transformations, testing, documentation.
8. **Streaming** — Kafka, Spark Structured Streaming, Event Hubs. Learn after batch is solid.

## Orchestration tool comparison

| Tool | Type | Strengths | Weaknesses |
|---|---|---|---|
| **Apache Airflow** | Open-source, Python | Maximum flexibility; Python DAGs; huge community; cloud-managed (MWAA, Cloud Composer, Astronomer) | Complex to self-host; Python-only; UI is functional but dated |
| **Azure Data Factory** | Managed SaaS (Azure) | Visual UI; native Azure integrations; built-in connectors; no infra to manage | Azure-only; limited Python; complex JSON pipeline definitions |
| **Prefect** | Open-source + SaaS | Modern Python API; easy local dev; Prefect Cloud for managed scheduling | Smaller community than Airflow; SaaS cost |
| **Dagster** | Open-source + SaaS | Asset-based model (tracks data assets, not just tasks); strong observability | Steeper learning curve; smaller community |
| **Microsoft Fabric** | Managed SaaS | Integrated with Lakehouse/Warehouse/Spark; no separate orchestration infra | Azure/Fabric-only; less flexible than Airflow for complex logic |

For Azure-first teams: ADF for data movement + triggering; Airflow or Fabric pipelines for complex orchestration logic.

## Certifications

| Cloud | Certification | Level |
|---|---|---|
| **Microsoft Azure** | DP-203: Azure Data Engineer Associate | Associate |
| **Microsoft Azure** | DP-900: Azure Data Fundamentals | Fundamentals |
| **Databricks** | Databricks Certified Associate Developer for Apache Spark | Associate |
| **Databricks** | Databricks Certified Data Engineer Associate | Associate |
| **AWS** | AWS Certified Data Engineer – Associate (DEA-C01) | Associate |
| **Google Cloud** | Professional Data Engineer | Professional |
| **dbt** | dbt Certified Developer | Tool-specific |

For Microsoft Fabric specifically: DP-600 (Microsoft Fabric Analytics Engineer) is the current relevant certification.
