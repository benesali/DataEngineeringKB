# Partitioning & Clustering

## Partitioning

Partitioning splits a large table into smaller physical segments based on the values in one or more columns. A query with a filter on the partition column only reads the relevant partition — skipping the rest entirely.

### Example: date partitioning

A table with 3 years of sales data partitioned by `SalesDate`:

```
sales/
  SalesDate=2024-01-01/  ← only these files are read for a Jan 2024 query
  SalesDate=2024-01-02/
  ...
  SalesDate=2026-08-06/
```

Query: `WHERE SalesDate = '2024-01-01'` reads 1/1000th of the data.

### Choosing a partition column

Good partition columns:
- **High query filter frequency** — always filtered in WHERE clauses
- **Low-to-medium cardinality** — date (365/year) is good; CustomerID (millions) is too many partitions

Poor choices:
- Boolean columns → only 2 partitions, uneven distribution
- High-cardinality strings → millions of tiny files (the "small file problem")

### Common partition strategies

| Strategy | Column | Use case |
|---|---|---|
| Date | `LoadDate`, `EventDate` | Time-series data, historical tables |
| Year-Month | `YEAR(date)`, `MONTH(date)` | Reduces partition count for long histories |
| Region / Country | `CountryCode` | Global datasets where queries filter by region |

## Clustering (Z-ordering / Liquid Clustering)

Partitioning is coarse-grained (one directory per value). **Clustering** (Z-ordering in Delta Lake) is fine-grained — it co-locates related rows *within* a partition based on one or more column values.

```sql
-- Delta Lake Z-order by CustomerID and ProductID
OPTIMIZE my_table ZORDER BY (CustomerID, ProductID)
```

After Z-ordering, a query filtering by `CustomerID = 12345` reads far fewer files within each partition — statistics per file show the min/max values, so Spark can skip files that don't contain CustomerID=12345.

**Liquid Clustering** (Delta Lake 3.0+) is the successor — automatic, incremental, no need to choose a static partition column.

## Small file problem

Too many partitions + frequent small writes → thousands of tiny Parquet files → slow reads (metadata overhead).

Fix: `OPTIMIZE` compacts small files into larger ones. Run it regularly on high-write tables.

```sql
OPTIMIZE my_table WHERE LoadDate >= date_sub(current_date(), 7)
```

## Partition elimination vs data skipping

Two complementary techniques for avoiding unnecessary data reads:

| Technique | Granularity | How it works |
|---|---|---|
| **Partition elimination** | Partition level (directory) | Query planner skips entire partitions based on the filter; no files read at all |
| **Data skipping** | File level (within a partition) | Delta Lake reads file-level min/max statistics; skips files where the filter value is outside the min-max range |

Example: table partitioned by `LoadDate`, Z-ordered by `CustomerID`:

- `WHERE LoadDate = '2026-08-01'` → partition elimination skips all other date directories
- `WHERE CustomerID = 12345` → data skipping reads file statistics and skips files where `CustomerID` min-max range excludes 12345

Both work together — partition elimination narrows to the right directory, data skipping narrows to the right files within that directory.

## Auto Optimize (Databricks)

Databricks has two features that automatically manage file sizing:

**Optimized writes** — Databricks automatically coalesces small write tasks into larger files before committing. Avoids the small file problem without a separate OPTIMIZE job.

```sql
-- Enable at table level
ALTER TABLE silver.orders
SET TBLPROPERTIES (delta.autoOptimize.optimizeWrite = true);

-- Or at session level
SET spark.databricks.delta.optimizeWrite.enabled = true;
```

**Auto compaction** — after each write, if enough small files have accumulated, Databricks automatically triggers a compact operation in the background.

```sql
ALTER TABLE silver.orders
SET TBLPROPERTIES (delta.autoOptimize.autoCompact = true);
```

These are Databricks Runtime-specific features. In vanilla Spark or Microsoft Fabric, schedule a periodic `OPTIMIZE` job manually.

## Horizontal partitioning: manageability vs performance

Two distinct goals drive horizontal partitioning — choose the strategy that matches the goal:

### Date-based partitioning for manageability

Partition by date (year, month, quarter) to enable independent lifecycle management per partition:

- **Tiered storage:** recent partitions (last 3 months = "hot") on fast SSD storage; older partitions (last 3 years = "warm") on standard storage; archive partitions (>3 years = "cold") on cheap blob/object storage. Move partitions between tiers by metadata change, not data rewrite.
- **Incremental backups:** back up only the partitions that changed since the last backup (current + recent). Historical partitions are static — back them up once.
- **Archival:** when a partition ages out of the retention window, detach it, compress it, move to archive. No full-table operations.
- **Rolling delete:** delete the oldest partition directly (partition drop, not DELETE WHERE) — instant and cheap.

### Hash-based partitioning for parallel query performance

Partition by hash of a key column to distribute data evenly across database nodes or processing units:

- Each query that filters by the hash key is routed to the correct partition; no cross-partition fan-out
- Works well for data marts with known, repetitive query patterns (e.g. always filtered by CustomerID)
- Even distribution requires the key to have many distinct values and no hot spots

---

## Indexing strategies for DWH

### B-tree indexes

B-tree (balanced tree) indexes organize values in a sorted tree structure. Best for:

- Columns with **high selectivity** (many distinct values — e.g. CustomerID, OrderDate)
- Range queries (`WHERE OrderDate BETWEEN '2026-01-01' AND '2026-03-31'`)
- Equality lookups on FK columns (joining fact to dimension)

**Key limitations in analytical DWH:**
- You cannot combine multiple independent B-tree indexes in a single query execution — the optimizer picks one
- A compound (multi-column) B-tree index is efficient only when the leading column appears in the WHERE clause; queries that filter only on trailing columns get no benefit
- High maintenance overhead during bulk loads (each insert updates the index)

B-tree indexes are appropriate for **controlled environments**: queries are known and predictable (e.g. a dimension lookup always joins on CustomerKey), and bulk loads can disable and rebuild the index.

### Bitmap indexes

Bitmap indexes create one compressed bit vector per distinct value in the column. Each bit represents one row; the bit is 1 if that row has that value.

Analytical queries with multiple filter conditions use Boolean AND/OR/NOT operations on the bit vectors — extremely fast for combining several low-selectivity filters.

Best for:
- Low-cardinality columns (Status, Region, ProductCategory — few distinct values)
- **Ad hoc analytical queries** where many different filter combinations are issued

**Why bitmap indexes are problematic for frequently-updated tables:**
- Every UPDATE or INSERT in the target table causes fragmentation in every affected bit vector
- Under concurrent writes, bitmap index maintenance creates locking contention
- Must be **rebuilt after each bulk load** to defragment; for a large DWH table this can be expensive

Bitmap indexes are appropriate for **read-mostly data marts** — load once, query many times, rebuild at load time.

| | B-tree | Bitmap |
|---|---|---|
| Best for | High-selectivity FK/range columns | Low-cardinality filter columns |
| Query pattern | Known/predictable | Ad hoc |
| Update overhead | Medium (one tree update per row) | High (bit vector fragmentation) |
| Combine multiple? | Not effectively | Yes — AND/OR/NOT on bit vectors |
| After bulk load | Disable + rebuild | Rebuild mandatory |

### Local vs global indexes on partitioned tables

When a table is horizontally partitioned, index scope matters:

**Local index** — one index partition per data partition. The index is stored in the same partition as the data. When a partition is added, dropped, or moved, only its local index is affected — no other partitions' indexes change. This is why local indexes are strongly preferred for DWH tables: operational tasks (partition archival, partition load) have limited blast radius.

**Global index** — a single index that spans all partitions. Provides full flexibility for queries that don't filter on the partition key. But every partition-level operation (drop, split, merge) invalidates the global index — the entire index must be rebuilt. High maintenance cost at DWH scale.

**Recommendation:** prefer local indexes. Use global indexes only when specific query patterns make them unavoidable, and build in a scheduled index rebuild after each partition operation.

---

## Vertical partitioning

Horizontal partitioning splits rows across partitions. **Vertical partitioning** splits *columns* across tables — each table holds a subset of the original table's columns. All vertical partitions share the same primary key.

Three main use cases:

### For query performance

Separate columns by access frequency:
- **Frequently accessed columns** (Name, Status, Key) — primary table, always in cache
- **Seldom accessed columns** (secondary attributes queried occasionally) — secondary table
- **Never-updated columns** (historical/archival data) — archive table

A query that only needs the frequently-accessed columns never touches the secondary or archive tables — less I/O, smaller memory footprint, faster response.

### For change history

Separate current-state attributes from historical attributes:
- **Current table** — one row per entity, only the current values
- **History table** — one row per change event per entity, with effective/expiration dates

This avoids the need to scan a single large SCD2 table for the current row. The current table is small and fast; the history table is large but only queried when historical analysis is needed.

### For large columns

Isolate columns with large payloads (BLOBs, CLOBs, long VARCHAR) into a separate table:
- Analytical queries that don't need the large column never read it — no I/O wasted
- The main table stays compact and fits more rows per database page

When the relationship between the main table and the large-column table is many-to-many (e.g. a document that can be attached to multiple orders), use an **associative entity** to link them.

---

## Denormalization

In a normalized model, a customer's country is stored in a City table, which links to a Country table. A query joining Customer → Order → City → Country requires four joins.

**Denormalization** collapses some of these tables together — the Country column is added directly to Customer, eliminating the join chain. This trades write efficiency (redundant storage, multi-table updates) for read efficiency (fewer joins, smaller result sets).

Common DWH denormalization patterns:
- **Rolled-up hierarchies** — flatten a multi-level hierarchy into columns on the dimension row (Region, District, Territory all on DimCustomer) — eliminates hierarchy join traversal
- **Pre-computed derived columns** — store `FullYearRevenue` on DimCustomer even though it could be computed from FactSales; avoids an aggregation join on every query
- **Collapsed subtypes** — see subtype clusters below

Denormalization is appropriate in the **data delivery layer** (data marts, Gold layer). In the **DWH integration layer**, keep normalization to maintain flexibility and correctness.

## Subtype clusters

Some entities have a common set of attributes (the supertype) and a specialized set that only apply to certain subtypes:

```
Vehicle (supertype): VehicleID, Make, Model, Year
  ├── Truck (subtype): PayloadCapacity, AxleCount
  └── Car (subtype): DoorCount, BodyStyle
```

A normalized model uses three tables (Vehicle, Truck, Car) with FK relationships. For analytical DWH delivery, combine them into a **subtype cluster** — a single wide table with all columns, with NULL in subtype columns that don't apply:

```sql
CREATE TABLE DimVehicle (
    VehicleKey       INT NOT NULL,
    Make             VARCHAR(100),
    Model            VARCHAR(100),
    Year             INT,
    VehicleType      VARCHAR(20),  -- 'Truck' or 'Car'
    -- Truck columns:
    PayloadCapacity  DECIMAL(10,2) NULL,
    AxleCount        INT NULL,
    -- Car columns:
    DoorCount        INT NULL,
    BodyStyle        VARCHAR(50) NULL
);
```

Eliminates subtype joins; queries filtering on VehicleType only need this one table. Appropriate when subtype tables are small and queries frequently span supertypes.

---

## Iceberg hidden partitioning vs Delta explicit partitioning

| | Delta Lake | Apache Iceberg |
|---|---|---|
| Partition definition | Must specify `PARTITIONED BY (LoadDate)` — query must also filter on `LoadDate` for pruning | Hidden transforms: `PARTITIONED BY (days(LoadDate))` — Iceberg rewrites the predicate automatically |
| User experience | Analyst must know the partition column and use it in WHERE | Analyst doesn't need to know the partition — `WHERE LoadDate BETWEEN ...` auto-prunes |
| Partition evolution | Changing partition scheme rewrites the table | Iceberg supports partition evolution without rewriting existing files |
| Small files on high cardinality | Risk if `PARTITIONED BY (CustomerID)` — millions of directories | Hidden bucket transforms (`bucket(100, CustomerID)`) avoid file explosion |

Iceberg's hidden partitioning is one of its most significant advantages for large, complex tables — it makes partition pruning automatic and transparent to end users.
