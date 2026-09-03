# Distributed Processing (Spark)

## Why distributed?

A single machine has limits — memory, CPU, disk I/O. When data exceeds those limits, you need to distribute the work across many machines working in parallel.

Apache Spark is the dominant distributed processing engine for data engineering. It runs in-memory, distributes computation across a cluster, and integrates natively with Delta Lake.

## Core concepts

### Driver and executors

```
Driver (coordinator)
    │
    ├── Executor 1 (worker node)
    ├── Executor 2 (worker node)
    └── Executor 3 (worker node)
```

- **Driver** — orchestrates the job: parses the query, builds an execution plan, schedules tasks
- **Executors** — do the actual work: read data, apply transformations, write results
- Each executor processes a **partition** of the data independently

### Partitions (data parallelism)

A Spark DataFrame is split into partitions. Each partition is processed by one task on one executor — in parallel.

```
DataFrame
├── Partition 0  ← Executor 1
├── Partition 1  ← Executor 2
└── Partition 2  ← Executor 3
```

More partitions = more parallelism = faster (up to the cluster size limit).

### Lazy evaluation

Spark builds a **logical plan** of transformations without executing them. Execution only happens when an **action** is called (`.write`, `.count`, `.collect`).

This allows Spark to optimize the full query plan before running anything.

## Shuffles — the expensive operation

A **shuffle** happens when data must be redistributed across executors (e.g., a `GROUP BY` or a `JOIN`). All data for a given key must land on the same executor.

Shuffles involve writing data to disk and sending it across the network — the most expensive operation in Spark.

**How to minimize:**
- Filter data early (push predicates down to the read)
- Use broadcast joins for small tables (one executor sends the small table to all others)
- Partition data on join keys to avoid reshuffling

## DataFrame API vs SQL

Both are equivalent — Spark compiles both to the same execution plan.

```python
# DataFrame API
df = spark.read.table("silver.customers")
  .filter(col("Country") == "CZ")
  .groupBy("Segment")
  .agg(count("*").alias("CustomerCount"))

# SQL
spark.sql("""
  SELECT Segment, COUNT(*) AS CustomerCount
  FROM silver.customers
  WHERE Country = 'CZ'
  GROUP BY Segment
""")
```

## Spark in the Medallion context

| Layer | Spark usage |
|---|---|
| Bronze | Spark Structured Streaming for real-time; batch reads for file ingestion |
| Silver | Spark batch transformations + MERGE for SCD2 |
| Gold | Spark batch aggregations + dimension/fact loads |

## Adaptive Query Execution (AQE)

AQE (Spark 3.0+) re-optimizes query plans at runtime based on actual statistics collected during execution. It activates after shuffle operations, where the actual data distribution is known.

Three key AQE features:

| Feature | What it does |
|---|---|
| **Coalesce post-shuffle partitions** | Reduces the number of shuffle partitions when actual data is smaller than expected (avoids thousands of tiny tasks) |
| **Convert sort-merge join to broadcast join** | If one side of a join turns out to be small after filtering, switch to a cheaper broadcast join at runtime |
| **Skew join optimization** | Splits oversized skewed partitions into smaller tasks to avoid one slow task blocking the entire stage |

AQE is enabled by default in Spark 3.2+. No code changes needed — it activates automatically.

## Memory tuning

Key Spark memory knobs:

```python
spark = SparkSession.builder \
    .config("spark.executor.memory", "8g") \
    .config("spark.executor.cores", "4") \
    .config("spark.sql.shuffle.partitions", "200") \  # default 200 — reduce for small data
    .config("spark.memory.fraction", "0.75") \       # fraction of heap for Spark (rest = user)
    .getOrCreate()
```

Symptoms and fixes:

| Symptom | Likely cause | Fix |
|---|---|---|
| OOM on executors | Too little `executor.memory` or too-large partitions | Increase memory or reduce partition size |
| Thousands of tiny tasks | `shuffle.partitions` too high for data size | Reduce `shuffle.partitions` or enable AQE coalescing |
| One task runs 10x longer | Data skew on join/groupBy key | Enable AQE skew join; salt the key |
| Disk spill | Not enough memory for shuffle data | Increase executor memory or reduce parallelism |

Rule of thumb: start with 4 cores per executor, 4–8 GB memory per executor. Tune from there based on observed behavior.

## Structured Streaming

Spark Structured Streaming processes continuous data as an unbounded DataFrame. Two processing modes:

| Mode | How it works | Latency | Use case |
|---|---|---|---|
| **Micro-batch** (default) | Processes accumulated data in small batches on a trigger interval | Seconds | Most streaming ETL workloads |
| **Continuous** (experimental) | Sub-millisecond latency; very limited operations supported | Milliseconds | Low-latency alerting |

Micro-batch example reading from Kafka:

```python
stream = (
    spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker:9092")
    .option("subscribe", "crm.customer.updates")
    .load()
    .selectExpr("CAST(value AS STRING) AS message", "timestamp")
)

stream.writeStream \
    .format("delta") \
    .option("checkpointLocation", "/mnt/checkpoints/crm_updates") \
    .trigger(processingTime="30 seconds") \
    .table("bronze.crm_updates")
```

The **checkpoint** persists offsets and state — restarting the stream resumes exactly where it left off.

## Databricks cluster types

| Type | When to use | Cost model |
|---|---|---|
| **Job cluster** | Automated pipeline runs (notebooks, Spark jobs) | Created fresh per job run; terminated when done — most cost-efficient |
| **All-purpose cluster** | Interactive development in notebooks | Stays running until manually stopped or after idle timeout; expensive if left on |
| **SQL warehouse** | BI queries, dbt, SQL notebooks | Serverless or provisioned; auto-suspends on idle |

Best practice: **never run production pipelines on all-purpose clusters**. Use job clusters (or serverless compute in Databricks workflows) — they spin up fresh, run the job, and terminate automatically.
