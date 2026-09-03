# Python for Data Engineering

## Why it matters

Python has become the lingua franca of data engineering. But "Python" covers a wide surface — the way you structure a project, choose a DataFrame library, manage credentials, and test your transforms all compound into the difference between a maintainable pipeline and a liability.

This page covers the patterns that matter in production Python data engineering: project setup, type safety, secrets, the major DataFrame libraries (pandas, Polars, PySpark), pipeline design, testing, and observability.

---

## Project setup

Good project hygiene pays for itself immediately — it makes the codebase navigable to the next person and prevents whole classes of environment bugs.

### The src layout

Put source code under `src/`. This prevents the most common Python import bug: accidentally importing from the current working directory instead of the installed package.

```
my_project/
├── src/
│   └── my_package/
│       ├── __init__.py
│       ├── transforms.py
│       └── io.py
├── tests/
│   ├── conftest.py
│   ├── unit/
│   └── integration/
├── pyproject.toml          # single source of truth — no setup.py, no setup.cfg
├── .pre-commit-config.yaml
└── README.md
```

### pyproject.toml — single source of truth

All build config, tool config, and dependency declarations belong in one file. No separate `.mypy.ini`, `.flake8`, or `ruff.toml`.

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-pipeline"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["polars>=1.0", "azure-identity>=1.16"]

[project.optional-dependencies]
dev = ["pytest", "pytest-cov", "mypy", "ruff"]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "N", "B"]  # pyflakes, isort, pyupgrade, naming, bugbear

[tool.mypy]
strict = true
python_version = "3.12"

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--cov=src --cov-report=term-missing"
markers = ["unit: fast, no I/O", "integration: hits DB or network"]
```

### uv — modern package and environment management

[uv](https://docs.astral.sh/uv/) replaces pip, venv, virtualenv, and pip-tools in one Rust binary. 10–100× faster than pip, handles lockfiles, and never touches the system Python.

```bash
# Create venv and install all deps (including dev) in one step
uv sync --extra dev

# Run a command inside the venv without activating it
uv run pytest

# Add a dependency (updates pyproject.toml + uv.lock)
uv add polars

# Pin Python version per-project
uv python pin 3.12
```

> **PEP 668 note:** Modern Linux/macOS forbids `pip install` outside a venv. Always work inside a virtual environment — uv enforces this automatically.

### Ruff — linting and formatting

Ruff replaces flake8, black, isort, and pyupgrade in a single binary, roughly 100× faster than the combined toolchain.

```bash
# Lint and autofix
ruff check . --fix

# Format (replaces black)
ruff format .
```

### Pre-commit hooks

Pre-commit runs checks on every `git commit`, catching issues before they reach CI.

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.5.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.10.0
    hooks:
      - id: mypy

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: check-merge-conflict
      - id: trailing-whitespace
      - id: end-of-file-fixer
```

---

## Type safety

Type annotations are documentation that the computer checks. For data engineering code — where a wrong column type silently corrupts a dataset — they are especially valuable.

```python
from __future__ import annotations  # PEP 563: defer evaluation, enables forward refs
from collections.abc import Sequence
from typing import TypeVar

T = TypeVar("T")

# PEP 585: built-in generics (no need to import List, Dict)
# PEP 604: X | Y union syntax (no need for Optional)
def first_or_default(items: list[str], default: str | None = None) -> str | None:
    return items[0] if items else default

# Use Protocol for structural typing instead of ABC
from typing import Protocol

class DataSource(Protocol):
    def read(self, path: str) -> list[dict]: ...
```

Run `mypy --strict` in CI. Use `collections.abc` for abstract types (`Sequence`, `Mapping`, `Callable`) rather than the deprecated `typing` equivalents.

---

## Secrets management

Hardcoded credentials are the single most common cause of data breaches. The 2024 Verizon DBIR found that 60% of breaches involve credential theft, with hardcoded secrets and exposed environment variables as primary vectors. Use a layered approach.

### Layer 1 — local development: .env + python-dotenv

```python
from dotenv import load_dotenv
import os

load_dotenv()  # reads .env file, does nothing in prod where env vars are injected

db_password = os.getenv("DB_PASSWORD")
```

**Rules:**
- Add `.env` to `.gitignore` on day one — never commit it
- Never commit a real secret, even in a private repo
- Use `.env.example` with placeholder values as a template for new developers

### Layer 2 — typed config with pydantic-settings

`pydantic-settings` reads from environment variables (or `.env`) and validates them at startup — you find misconfiguration immediately, not at runtime when a job fails.

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from functools import lru_cache

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

    database_url: str
    api_key: str
    environment: str = "development"

    @property
    def is_production(self) -> bool:
        return self.environment == "production"

@lru_cache
def get_settings() -> Settings:
    return Settings()  # reads env once, cached for the process lifetime
```

Never read `os.environ` directly in application code. All config flows through `Settings` — it stays typed, validated, and testable (override via dependency injection in tests).

### Layer 3 — production: cloud secret managers

For production, never store secrets in environment variables injected into containers — they can appear in process listings and environment dumps. Use a managed secrets service.

**Azure Key Vault (recommended for Azure/Fabric workloads):**

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

credential = DefaultAzureCredential()  # uses Managed Identity in Azure, local CLI elsewhere
client = SecretClient(
    vault_url="https://my-vault.vault.azure.net/",
    credential=credential,
)
secret_value = client.get_secret("db-password").value
```

`DefaultAzureCredential` tries a chain of auth methods — Managed Identity in Azure, then Azure CLI, then VS Code login. The same code works locally and in production without changes.

**AWS / HashiCorp Vault** equivalents follow the same pattern: a credential object resolves auth, a client reads the secret by name.

### Anti-patterns to avoid

| Anti-pattern | Why it's dangerous |
|---|---|
| `password = "hunter2"` in source code | Committed to git forever — rotation doesn't help |
| `os.environ["DB_PASS"]` scattered across modules | No validation; fails late; hard to audit |
| Secrets in notebook cells | Notebooks are often shared and outputs committed |
| `random.token_hex()` for security tokens | `random` is not cryptographically secure — use `secrets.token_urlsafe(32)` |
| String-building with secrets | Leaves memory traces — use `secrets.compare_digest()` for comparison |

---

## DataFrame libraries

Three libraries dominate Python data engineering. The right choice depends on data volume and environment.

| Library | Best for | Avoid when |
|---|---|---|
| **pandas** | < 1 GB, exploration, ad-hoc transforms, ML feature pipelines | Multi-GB datasets, distributed workloads |
| **Polars** | 1 MB – ~50 GB, batch pipelines, fast local transforms, type safety | Existing Spark cluster, Spark-native UDFs needed |
| **PySpark** | > 10 GB, distributed cluster, streaming (Structured Streaming) | Small data — Spark overhead exceeds benefit |

### pandas

pandas is powerful and easy to misuse. The two core rules: declare dtypes on read, and never iterate over rows.

```python
import pandas as pd

# Declare dtypes at read time — never let pandas guess
df = pd.read_csv("sales.csv", dtype={"customer_id": "int32", "amount": "float32"})

# Method chaining — readable, avoids intermediate variables
result = (
    df
    .query("status == 'active' and amount > 0")
    .assign(
        full_name=lambda x: x["first"] + " " + x["last"],
        amount_eur=lambda x: x["amount"] / 1.1,
    )
    .groupby("region", as_index=False)["amount_eur"]
    .sum()
    .sort_values("amount_eur", ascending=False)
)
```

**Do:**
- Use `pd.ArrowDtype` for string/datetime columns in pandas 2+ — significantly lower memory
- Prefer Parquet over CSV for all pipeline I/O — typed, compressed, column-prunable
- Profile memory with `df.info(memory_usage="deep")` before optimising

**Don't:**
- `iterrows()` / `itertuples()` — use vectorised operations; `apply()` only as a last resort
- Chained indexing `df["a"]["b"]` — use `df.loc[..., "b"]` to avoid the `SettingWithCopyWarning`
- Grow a DataFrame in a loop with `pd.concat` — collect rows in a list first, then concat once

### Polars

Polars is the modern high-performance alternative to pandas. It uses a columnar in-memory format (Apache Arrow), executes in Rust, and exposes a lazy query planner that optimises the full pipeline before running it.

**Core concepts:**

- **Eager mode** (`pl.DataFrame`) — executes immediately, like pandas. Use for quick exploration.
- **Lazy mode** (`pl.LazyFrame`) — builds a query plan; `.collect()` executes it. Use for production pipelines. The engine can reorder, push down filters, and parallelise automatically.
- **Expressions** — composable, typed column operations that stay within the Polars DSL, letting the engine optimise them.

```python
import polars as pl

# Lazy pipeline — nothing runs until .collect()
result = (
    pl.scan_parquet("sales/*.parquet")        # scan, not read — reads only needed columns
    .filter(pl.col("status") == "active")
    .filter(pl.col("amount") > 0)
    .with_columns(
        full_name=pl.concat_str(["first", "last"], separator=" "),
        amount_eur=(pl.col("amount") / 1.1).round(2),
    )
    .group_by("region")
    .agg(pl.col("amount_eur").sum().alias("total_eur"))
    .sort("total_eur", descending=True)
    .collect()                                # execute the full plan here
)
```

**Expressions — composable and optimisable:**

```python
# Define reusable expressions
active = pl.col("status") == "active"
eur_amount = (pl.col("amount") / 1.1).round(2).alias("amount_eur")

# Compose them — the engine sees the full plan and can optimise
result = df.filter(active).with_columns(eur_amount)
```

**Type safety:**

Polars schemas are strict — column types are declared, not inferred at collect time. A type mismatch raises immediately.

```python
schema = {
    "customer_id": pl.Int32,
    "amount": pl.Float64,
    "event_date": pl.Date,
}

df = pl.read_parquet("data.parquet", schema_overrides=schema)
```

**Do:**
- Use `scan_*` functions (`scan_parquet`, `scan_csv`) instead of `read_*` in lazy pipelines — they enable predicate and projection pushdown
- Use `.pipe()` to break long chains into named, testable steps
- Use `pl.concat()` to union DataFrames — it is zero-copy on Arrow buffers
- Use `group_by(...).agg(...)` rather than `apply` for aggregations — stays in the native engine

**Don't:**
- Call `.collect()` inside a loop — defeats the lazy planner entirely
- Mix Polars and pandas in the same pipeline — `df.to_pandas()` copies all data
- Use `.apply()` (row-wise Python function) unless truly unavoidable — it leaves the native engine

### PySpark

PySpark is for distributed workloads on a cluster. Always define schemas explicitly — schema inference reads the entire dataset and is expensive and unreliable.

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

schema = StructType([
    StructField("customer_id", IntegerType(), nullable=False),
    StructField("name",        StringType(),  nullable=True),
    StructField("amount",      IntegerType(),  nullable=True),
])

spark = SparkSession.builder.appName("pipeline").getOrCreate()

df = (
    spark.read.schema(schema).parquet("/data/raw/")
    .filter(F.col("amount") > 0)
    .withColumn("amount_eur", F.col("amount") / 1.1)
    .groupBy("name")
    .agg(F.sum("amount_eur").alias("total_eur"))
)

# Write partitioned by date — enables partition pruning on downstream reads
df.write.mode("overwrite").partitionBy("date").parquet("/data/processed/sales/")
```

**Do:**
- Always define explicit schemas — never rely on inference in production
- Broadcast small lookup DataFrames: `F.broadcast(small_df)` — avoids a shuffle join
- Cache a DataFrame only when it is used in 2+ separate actions
- Partition writes by a filter column (usually date) for downstream read performance

**Don't:**
- `.collect()` on large DataFrames — pulls all data to the driver node
- Call `.count()` inside a loop — each call triggers a full Spark action
- Use SQL string interpolation — use `F.col()` API to prevent injection

---

## Pipeline design

### Idempotency

Every pipeline run must produce the same result regardless of how many times it runs. See the dedicated [Idempotency](idempotency.md) page for SQL patterns and the watermark / Spark Structured Streaming cases.

**The Python-side rule:** overwrite a partition, never append blindly.

```python
# Idempotent — re-running overwrites the same partition with the same data
(
    df.write.mode("overwrite")
    .partitionBy("date")
    .parquet("/data/processed/sales/")
)

# Not idempotent — re-running duplicates every row
df.write.mode("append").parquet("/data/processed/sales/")
```

### Write-Audit-Publish (WAP)

The WAP pattern separates writing from exposing. Write to a staging area, run validation, then atomically swap. Delta Lake / Iceberg support this natively. The simplified pattern in batch pipelines:

1. **Write** to a staging table or partition
2. **Audit** — run data quality assertions
3. **Publish** — overwrite the production table only if assertions pass

```python
def run_pipeline(spark: SparkSession, date: str) -> None:
    df = extract(spark, date)
    df_transformed = transform(df)

    # Audit before publish
    assert df_transformed.filter(F.col("customer_id").isNull()).count() == 0, \
        "NULL customer_ids in output — aborting publish"
    assert df_transformed.count() > 0, "Empty output — aborting publish"

    # Publish only after assertions pass
    (
        df_transformed.write
        .mode("overwrite")
        .partitionBy("date")
        .parquet("/data/processed/sales/")
    )
```

### Error handling and retry

Transient failures (network timeouts, API rate limits) should be retried. Structural failures (schema mismatch, bad data) should fail fast.

```python
import time
import logging
from collections.abc import Callable
from typing import TypeVar

T = TypeVar("T")
logger = logging.getLogger(__name__)

def retry(fn: Callable[[], T], attempts: int = 3, backoff_seconds: float = 2.0) -> T:
    """Retry fn up to attempts times with exponential backoff."""
    last_exc: Exception | None = None
    for attempt in range(1, attempts + 1):
        try:
            return fn()
        except Exception as exc:
            last_exc = exc
            if attempt < attempts:
                wait = backoff_seconds * (2 ** (attempt - 1))
                logger.warning("attempt %d failed, retrying in %.1fs: %s", attempt, wait, exc)
                time.sleep(wait)
    raise last_exc  # type: ignore[misc]
```

---

## Testing

Separate unit tests from integration tests — they have different speed, dependencies, and failure modes.

```python
# tests/unit/test_transforms.py
import pytest
import polars as pl
from my_package.transforms import compute_totals

@pytest.mark.unit
@pytest.mark.parametrize("amounts,expected_total", [
    ([10.0, 20.0], 30.0),
    ([],            0.0),
    ([-5.0, 5.0],   0.0),
])
def test_compute_totals(amounts: list[float], expected_total: float) -> None:
    df = pl.DataFrame({"amount": amounts})
    result = compute_totals(df)
    assert result["total"][0] == expected_total
```

```python
# tests/integration/test_pipeline.py  — allowed to hit disk/DB/network
import pytest
from my_package.pipeline import run_full_pipeline

@pytest.mark.integration
def test_pipeline_produces_output(tmp_path):
    run_full_pipeline(source="tests/fixtures/sample.parquet", dest=str(tmp_path))
    output = list(tmp_path.glob("*.parquet"))
    assert len(output) > 0
```

**Rules:**
- Unit tests: zero I/O, zero network — they run in milliseconds
- Mock only at the external boundary (HTTP client, filesystem, DB connection) — never mock internals
- Use `pytest-xdist` (`-n auto`) to parallelise the integration suite
- Run unit tests first in CI — fast feedback before the slower integration suite
- A test that doesn't assert anything is not a test

---

## Notebooks

Notebooks are for exploration and communication. Production logic belongs in `src/`.

**Do:**
- Start every notebook with `%autoreload 2` — changes to `src/` reload automatically without kernel restart
- Clear all outputs before committing: `jupyter nbconvert --clear-output --inplace notebook.ipynb`
- Structure cells with section headers: `# Section: Load`, `# Section: Transform`, `# Section: Verify`
- Reference functions from `src/` rather than duplicating logic in cells

**Don't:**
- Load credentials from environment variables directly in a cell — use the `Settings` class from your config layer
- Run notebooks as CI tests — extract the logic into pytest functions
- Leave large output tables or plots in committed notebooks — they bloat the git history and create merge conflicts

---

## Observability and logging

Logging in a data pipeline should answer: how much data moved, how long it took, and where it broke.

```python
import logging

# Per-module logger — never use the root logger directly
logger = logging.getLogger(__name__)

def process_batch(batch_id: str, df) -> None:
    input_rows = len(df)
    logger.info("batch started", extra={"batch_id": batch_id, "input_rows": input_rows})

    result = transform(df)

    output_rows = len(result)
    logger.info(
        "batch complete",
        extra={
            "batch_id": batch_id,
            "input_rows": input_rows,
            "output_rows": output_rows,
            "drop_rate": round(1 - output_rows / max(input_rows, 1), 4),
        },
    )
```

**Rules:**
- Log record counts at each stage: input → filtered → output
- Use `extra={}` for structured key-value pairs — they are queryable in log aggregation tools (Application Insights, Datadog, ELK)
- Never use f-strings in log calls: `logger.info("id=%s", id)` not `logger.info(f"id={id}")` — lazy formatting avoids string building when the log level is disabled
- Never log sensitive data: passwords, tokens, PII — even at DEBUG
- Call `logging.basicConfig()` only at the application entry point, never inside a library module
- Emit a metric (duration, row count, error count) — logs tell you what happened, metrics tell you if it's trending wrong

---

## Key PEPs for data engineers

| PEP | What it changes |
|---|---|
| [PEP 585](https://peps.python.org/pep-0585/) | Use `list[int]`, `dict[str, int]` — not `List`, `Dict` from `typing` (3.9+) |
| [PEP 604](https://peps.python.org/pep-0604/) | Use `str \| None` — not `Optional[str]` (3.10+) |
| [PEP 544](https://peps.python.org/pep-0544/) | Protocols — structural subtyping without ABC inheritance |
| [PEP 668](https://peps.python.org/pep-0668/) | Why `pip install` outside a venv now fails — always use `uv` |
| [PEP 636](https://peps.python.org/pep-0636/) | `match/case` — clean dispatch on message types, config variants |
