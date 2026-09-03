# Operational Data Store (ODS)

> *Primary source: W.H. Inmon, "Building the Data Warehouse, Fourth Edition" (Wiley, 2005), Chapter 16*

## What it is

The Operational Data Store is a **current, integrated, subject-oriented** view of operational data — updated in near real-time and designed for operational queries, not analytical ones.

The ODS is a companion to the data warehouse, not a replacement. Together they form the full Corporate Information Factory (CIF).

## ODS vs Data Warehouse

| | ODS | Data Warehouse |
|---|---|---|
| Data currency | Current (near real-time) | Historical (batch-loaded) |
| Updates | Yes — records updated as state changes | No — append-only |
| History? | No (or minimal) | Yes — full history |
| Query type | Operational ("what is the current state of X?") | Analytical ("what is the trend over time?") |
| Response time | Sub-second to seconds | Minutes to hours |
| Granularity | Transaction-level, current | Varies (detailed + summarized) |
| Users | Operational staff (customer service, ops) | Analysts, managers, data scientists |

## The question each answers

**ODS**: "What is the current balance on account 12345?"  
**DWH**: "What has been the average balance trend for customers in segment A over the past 3 years?"

## Four classes of ODS (Inmon)

| Class | Latency | Update mechanism |
|---|---|---|
| **Class I** | Seconds | Direct feeds from OLTP, near real-time |
| **Class II** | Minutes to hours | Light batch refresh |
| **Class III** | Overnight | Same batch window as the DWH |
| **Class IV** | DWH-derived | Built from DWH summaries, not OLTP |

Class I is the most capable but most expensive — requires change data capture or event-driven feeds. Class III is the most common in practice — loaded in the same overnight batch as the DWH but holds only the current state.

## Design: hybrid 3NF

The ODS is modeled in 3NF — same structure as the DWH — but without historical snapshots. When a customer's address changes:
- **ODS**: old address is overwritten with the new one
- **DWH**: a new snapshot row is added; the old row stays

This means the ODS always holds the **current state** and the DWH holds the **history of states**.

## Time slicing the ODS day

One common pattern: at the end of each business day, a snapshot of the ODS is captured and sent to the DWH. This snapshot becomes the DWH's source for that day's history rather than reading OLTP directly. Benefits:

- Reduces OLTP load (DWH reads from ODS snapshot, not live OLTP tables)
- Gives the DWH a consistent, already-integrated view of the operational state

## Multiple ODS

Large enterprises often have multiple ODS instances — one per domain (Customer ODS, Product ODS, Transactions ODS). These are federated via the enterprise 3NF model into a consistent view.

## ODS in modern architecture

In modern cloud lakehouse terms, the ODS concept maps to:
- A near-real-time operational layer feeding from streaming sources (Kafka, CDC)
- A Bronze/Silver layer with very low latency that answers current-state questions
- Separate from the analytical Gold layer which accumulates history

Tools like Delta Lake + Structured Streaming + MERGE INTO enable an ODS-style current-view table alongside a historical DWH table — both on the same lakehouse platform.

## Airline commission calculation (Inmon example)

An airline calculates agent commission payments that depend on **current flight status** — not historical booking data. A travel agent books a flight; the commission is calculated and paid only after the flight departs (not when booked) and is adjusted if the passenger no-shows or upgrades.

The ODS holds the **current state**: flight status (scheduled / departed / cancelled), passenger manifest, seat assignments. The commission calculation runs against the ODS in near-real-time when the flight departs — it must see today's state, not yesterday's DWH snapshot.

The DWH records historical commission totals per agent per month for trend analysis. The ODS is the source for operational commission calculation; the DWH is the source for management reporting.

## Retail personalization (Inmon example)

A retail chain's loyalty program must answer: "What is this customer's current tier and what promotions are they eligible for right now, as they check out?"

The loyalty ODS holds current-state data: total points balance (updated after each transaction), current tier (Gold/Silver/Bronze), active promotion eligibility. It is updated in real-time as transactions are processed.

The DWH holds historical point accumulation, tier changes over time, and promotion effectiveness for analytics. The two systems serve different latency requirements from the same underlying events.

## Class I ODS implementation with CDC

A Class I ODS (seconds of latency) requires Change Data Capture from the source OLTP system:

**Debezium** (open-source, Kafka-based):

```
Source OLTP (PostgreSQL/MySQL/SQL Server)
        │  (Debezium reads transaction log)
        ▼
Kafka topic (one per source table, with before/after row images)
        │  (Kafka Connect sink or custom consumer)
        ▼
ODS table (MERGE: apply the change — insert, update, or delete)
```

**Azure Data Factory CDC** (for Azure SQL / SQL Server):

ADF's CDC source connector reads the SQL Server change tracking table and delivers only changed rows — INSERT, UPDATE, or DELETE markers — to the destination. Latency: 1–5 minutes (configurable trigger interval).

**Debezium vs ADF CDC:**

| | Debezium | ADF CDC |
|---|---|---|
| Latency | Sub-second (Kafka consumer) | Minutes (pipeline trigger interval) |
| Setup | Complex (Kafka cluster + Debezium connector + consumer) | Simpler (ADF managed service) |
| Deletes | Full delete event with before-image | Delete marker delivered |
| Cloud fit | Any cloud; Kafka-native | Azure-native |

For a true Class I ODS, Debezium + Kafka is the standard pattern. For Class II (minutes), ADF CDC is simpler to operate.
