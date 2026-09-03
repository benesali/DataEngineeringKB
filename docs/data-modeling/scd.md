# Slowly Changing Dimensions (SCD)

> *Primary source: Ralph Kimball & Margy Ross, "The Data Warehouse Toolkit, Third Edition" (Wiley, 2013), Chapter 5 — Procurement*

## The problem

Dimension attributes change over time. A customer moves cities. A product is re-categorized. An employee changes department. When this happens, what do you do with the old value?

SCD types define the answer. Kimball defines 7 types (Types 0–6, plus Type 7).

---

## SCD Type 0 — Retain Original

The value is set once and never changed, regardless of what the source says.

**Use for**: immutable attributes — original signup date, date of birth, a customer's first-ever assigned segment.

```sql
-- If source sends a new value, it is ignored
CustomerKey=1, SignupDate='2020-01-15'
-- Source corrects to 2020-01-14 → still 2020-01-15 in the DWH
```

---

## SCD Type 1 — Overwrite

Simply update the row. No history is kept.

```
Before: CustomerKey=1, Name='Alice', City='London'
Source says: Alice moved to Manchester
After:  CustomerKey=1, Name='Alice', City='Manchester'
```

**Use for**: data corrections (the old value was simply wrong), or attributes where history genuinely has no analytical value.

**Risk**: all historical fact rows for Alice now show 'Manchester' — including sales from when she lived in London. Geographic analysis of past sales is now incorrect.

---

## SCD Type 2 — Add New Row (full history)

When a tracked attribute changes, close the current row and insert a new one with a new surrogate key. The historical row stays intact.

```sql
-- Before: Alice in London
CustomerKey=1,  CustomerBK='C001', Name='Alice', City='London',
    ValidFrom='2020-01-01', ValidTo=NULL, IsCurrent=1

-- Alice moves to Manchester on 2024-07-01:
CustomerKey=1,  CustomerBK='C001', Name='Alice', City='London',
    ValidFrom='2020-01-01', ValidTo='2024-06-30', IsCurrent=0  -- closed

CustomerKey=99, CustomerBK='C001', Name='Alice', City='Manchester',
    ValidFrom='2024-07-01', ValidTo=NULL,         IsCurrent=1  -- new current
```

A fact row for a sale in March 2022 still points to `CustomerKey=1` (London) — historically correct.

**Use for**: any attribute where the historical value of the dimension matters for past analysis — customer geography, employee department, product category.

**Cost**: dimension table grows with every change. Queries for current state must filter `IsCurrent=1`; point-in-time queries join on a date range.

### Change detection with HashDiff

Computing whether any tracked attribute changed before inserting a Type 2 row is expensive if done column by column. The standard optimization: compute a hash of all tracked attributes and store it:

```sql
-- On each ETL load, compute:
HashDiff = HASHBYTES('SHA2_256', CustomerName + '|' + City + '|' + Segment)
-- Compare incoming HashDiff to stored HashDiff
-- If different → new Type 2 row; if same → no change
```

---

## SCD Type 3 — Add New Attribute (one previous value)

Add a `Previous_X` column to store only the most recent prior value.

```sql
CustomerKey=1, Name='Alice', City='Manchester', PreviousCity='London'
```

**Use for**: only when you need to compare current vs. one prior value — "what was the old region?" for a specific analytical use case.

**Hard limit**: if Alice moves a third time, London is lost. Only one historical step is preserved.

---

## SCD Type 4 — Add Mini-Dimension

When certain attributes change so frequently (daily or weekly — e.g., customer age band, income bracket, credit score) that adding a Type 2 row for every change would explode the dimension table, extract those attributes into a separate small **mini-dimension**.

```
DimCustomer          DimCustomerProfile (mini-dimension)
─────────────        ──────────────────────────────────
CustomerKey (PK)     ProfileKey (PK)
CustomerBK           AgeBand          ('25-34', '35-44', ...)
Name                 IncomeLevel      ('Low', 'Medium', 'High')
Address              CreditScoreBand  ('A', 'B', 'C')
...

FactSales
──────────
CustomerKey (FK → DimCustomer)
ProfileKey  (FK → DimCustomerProfile)  ← current at time of sale
```

The mini-dimension rows are reused whenever the profile combination repeats — no new rows needed if the combination already exists. The fact table carries the `ProfileKey` at the time of the transaction.

---

## SCD Type 5 — Mini-Dimension + Type 1 Outrigger

Type 4 allows point-in-time analysis (via the fact table's `ProfileKey`). But filtering the current state of all customers by profile requires joining through the fact table — inconvenient for many reports.

Type 5 adds a **Type 1 outrigger key** to the main dimension pointing to the *current* mini-dimension profile:

```
DimCustomer
─────────────
CustomerKey (PK)
CustomerBK
Name
CurrentProfileKey (FK → DimCustomerProfile)  ← Type 1: always updated to current
...
```

Now you can filter customers by their current profile directly from `DimCustomer`, while the fact table's `ProfileKey` still gives the historical profile at time of sale.

---

## SCD Type 6 — Add Type 1 Attributes to Type 2 Dimension

Also called the "Hybrid" type. The dimension is Type 2 (full history via new rows) but also carries Type 1 current-value columns alongside the historical values.

```sql
CustomerKey=1,  CustomerBK='C001', City='London',  CurrentCity='Manchester',
    ValidFrom='2020-01-01', ValidTo='2024-06-30', IsCurrent=0

CustomerKey=99, CustomerBK='C001', City='Manchester',    CurrentCity='Manchester',
    ValidFrom='2024-07-01', ValidTo=NULL,           IsCurrent=1
```

Both the old and new rows carry `CurrentCity='Manchester'` — updated with each Type 1 overwrite.

**Benefit**: a single join to the dimension gives you both the historical value at time of sale (`City`) and the current value (`CurrentCity`) — enabling "what is the current region of customers who bought in 2022?"

---

## SCD Type 7 — Dual Type 1 and Type 2 Dimensions

The fact table carries two foreign keys to the dimension:

- `CustomerKey` → the surrogate key at the time of the event (historical)
- `CurrentCustomerKey` → always updated to the current row's key (Type 1)

This allows queries to choose: join on `CustomerKey` for historical analysis, or join on `CurrentCustomerKey` for current-state analysis of past facts — without adding extra columns to the dimension itself.

---

## Choosing the right SCD type

| | SCD 0 | SCD 1 | SCD 2 | SCD 3 | SCD 4 | SCD 5 | SCD 6 | SCD 7 |
|---|---|---|---|---|---|---|---|---|
| History kept | None | None | Full | One step | In mini-dim | In mini-dim | Full + current | Full + current |
| Dim size growth | None | None | High | None | Moderate | Moderate | High | High |
| Current-state query | Simple | Simple | Filter IsCurrent=1 | Simple | Requires join | Simple | Simple | Two FK choice |
| Historical query | N/A | N/A | Date range join | Limited | Via fact | Via fact | Via City column | Via CustomerKey |
| Common? | Rare | Very common | Very common | Rare | Moderate | Rare | Moderate | Rare |

**Type 1 and Type 2 together** are the overwhelming standard in practice. Types 3–7 address specific edge cases.

---

## SCD implementation in Kimball's ETL (Ch 5, Ch 19)

The standard ETL pattern for Type 2 in Kimball:

1. Extract incoming rows from source
2. Compute `HashDiff` for each row
3. Match to existing dimension rows by business key (`CustomerBK`)
4. **New key** (not in dimension) → INSERT as new Type 2 row (`IsCurrent=1`)
5. **Same hash** (no change) → no action
6. **Different hash** (changed) → UPDATE existing row (`ValidTo=today, IsCurrent=0`) + INSERT new row (`IsCurrent=1`)

This is implemented as a single **MERGE** statement in modern SQL engines:

```sql
MERGE INTO DimCustomer AS tgt
USING incoming AS src ON tgt.CustomerBK = src.CustomerBK AND tgt.IsCurrent = 1
WHEN MATCHED AND tgt.HashDiff <> src.HashDiff THEN
    UPDATE SET tgt.ValidTo = GETDATE(), tgt.IsCurrent = 0
;
INSERT INTO DimCustomer (CustomerBK, Name, City, ValidFrom, ValidTo, IsCurrent, HashDiff)
SELECT src.CustomerBK, src.Name, src.City, GETDATE(), NULL, 1, src.HashDiff
FROM incoming src
WHERE NOT EXISTS (SELECT 1 FROM DimCustomer WHERE CustomerBK = src.CustomerBK AND IsCurrent = 1 AND HashDiff = src.HashDiff)
;
```
