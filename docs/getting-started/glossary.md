# Core Concepts Glossary

Quick definitions — each term links to its full page where one exists.

| Term | Definition |
|---|---|
| **ETL** | Extract-Transform-Load. Data is transformed *before* being loaded into the warehouse. |
| **ELT** | Extract-Load-Transform. Raw data lands first, transformations run *inside* the warehouse. Modern cloud approach. |
| **Data Warehouse (DWH)** | A central repository of integrated, historical data structured for analytical queries. |
| **Data Lake** | Storage for raw data in its native format (files, JSON, Parquet) at low cost. |
| **Lakehouse** | Combines the flexibility of a data lake with the structure and performance of a warehouse. |
| **Medallion Architecture** | Bronze (raw) → Silver (clean) → Gold (serving). A layered lakehouse pattern. |
| **Fact table** | Stores measurable events or transactions (e.g. a sale, a click). Usually large. |
| **Dimension table** | Stores descriptive context for facts (e.g. customer, product, date). |
| **Star schema** | One fact table surrounded by dimension tables. The standard analytical model (Kimball). |
| **SCD** | Slowly Changing Dimension. A strategy for tracking how dimension attributes change over time. |
| **Surrogate key** | A system-generated unique ID with no business meaning. Used to join fact → dimension. |
| **Natural / business key** | The identifier from the source system (e.g. CustomerID from CRM). |
| **3NF** | Third Normal Form. Relational modeling that eliminates redundancy. Basis of Inmon DWH. |
| **Data Vault** | A modeling technique built on Hubs (keys), Links (relationships), Satellites (attributes). Flexible and auditable. |
| **Partitioning** | Splitting a large table into physical segments by a column (e.g. date) to speed up queries. |
| **Schema evolution** | How a system handles changes to table structure (new columns, type changes) without breaking pipelines. |
| **Idempotency** | A pipeline or script that produces the same result whether run once or many times. |
| **Orchestration** | Scheduling and managing dependencies between pipeline steps. |
| **Delta Lake** | Open-source storage format (on Parquet) that adds ACID transactions and schema enforcement to a data lake. |
| **SCD Type 1** | Overwrite old value — no history kept. |
| **SCD Type 2** | Add a new row for each change — full history kept. |
