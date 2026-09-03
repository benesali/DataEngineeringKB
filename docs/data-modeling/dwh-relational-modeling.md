# DWH Relational Data Modeling

> *Primary source: Claudia Imhoff, Nicholas Galemmo, Jonathan Geiger — "Mastering Data Warehouse Design: Relational and Dimensional Techniques" (Wiley, 2003), Chapters 2–4*

## The core argument

A data warehouse data model starts as a 3NF business data model and is transformed through 8 defined steps into the final DW schema. This page covers the concepts, the model levels, and the transformation process.

## Four levels of data models

Every enterprise should maintain data models at four levels of abstraction, each serving a different audience and purpose:

| Level | Audience | Scope | Time to build |
|---|---|---|---|
| **Subject Area Model** | Business + IT leadership | 15–25 major data groupings for the whole enterprise | Days |
| **Business Data Model** | Data modelers + business SMEs | Full 3NF model, all entities for relevant subject areas | Months (full); weeks (project scope) |
| **System Model** | Architects + developers | Specific to one system (e.g., the DW, the billing system); technology-independent | Weeks |
| **Technology Model** | DBAs + engineers | Platform-specific — hardware, DBMS, partitioning, indexes, RAID | Days per system |

Changes flow upward: if the technology model reveals a new business rule (e.g., a source system stores something differently than assumed), that change must be reflected back up through the system model and business data model. Uncontrolled divergence between levels causes chaos over time.

## Four entity types

Every entity in a relational data model falls into one of four structural categories:

| Type | Also called | What it represents | Example |
|---|---|---|---|
| **Primary** | Fundamental | The core "things" the business cares about | Customer, Product, Employee |
| **Subtype** | Specialization | A category of a primary entity with additional attributes | SalesManager (subtype of Employee) |
| **Attributive** | Characteristic | Attributes that repeat for one instance of a primary entity | OrderLine (many per Order) |
| **Associative** | Intersection | Resolves a many-to-many relationship | Enrollment (Student ↔ Course) |

A data model that is missing attributive and associative entities usually has hidden repeating groups or unresolved many-to-many relationships — classic signs that normalization is incomplete.

## Normalization: the mantra

> "All attributes must depend on the key, the whole key, and nothing but the key."

| Normal Form | Rule | What it removes |
|---|---|---|
| **1NF** | Every entity has a PK; no repeating groups or multi-valued attributes | Arrays embedded in a row |
| **2NF** | Non-key attributes depend on the *entire* PK (not part of it) | Partial key dependencies in composite-key entities |
| **3NF** | Non-key attributes depend only on the PK (not on other non-key attributes) | Transitive dependencies |

Business data models should reach 3NF at minimum. The data warehouse system and technology models start from the 3NF business model and introduce controlled denormalization for performance reasons — but every denormalization is documented and traceable to the business model.

## Three modeling guidelines

When facing a difficult decision, fall back to these three principles:

1. **Communication tool** — the model must be readable to business users, not just technical staff. When in doubt: does this addition improve clarity or reduce it?
2. **Lowest common denominator** — model at the finest grain available. Aggregated or derived values are decomposed to their components in the business model and re-derived in the technology model for performance.
3. **Business orientation** — model what the business *wants to be*, not what it is forced to be by existing system constraints. Operational system quirks (split customer tables, embedded codes) do not belong in the business model.

## Subject area model

The subject area model is the fastest artifact to produce and provides the initial scope boundary for every DW project.

**Common cross-industry subject areas:**

| Subject Area | Definition | Examples |
|---|---|---|
| Customers | Parties that acquire the company's products | Customer, Prospect, Consumer |
| Products | Goods and services the company provides | Product, Service |
| Sales | Transactions shifting ownership to a customer | Sales Transaction, Credit Memo |
| Human Resources | Individuals performing work for the company | Employee, Contractor, Position |
| Financials | Information about money received/expended | Receivable, Payable |
| Facilities | Real estate and structures | Building, Warehouse, Store |
| Equipment | Movable machinery and tools | Vehicle, Computer, Crane |
| Suppliers | Entities providing goods/services to the company | Manufacturer, Broker |
| Locations | Geographic points and areas | Country, Region, Address |

**Industry-specific additions:**
- Retail: Stores (separate from Facilities), Items (instead of Products)
- Manufacturing: Factories (separate from Facilities), Waste
- Insurance: Policies, Premiums, Claims, Reserves
- Health: Patients (instead of Customers), multiple Supplier types (facility, physician, pharmacist)

**Development approach:** Facilitated sessions (2 sessions) are the recommended method. Session 1 brainstorms potential subject areas; participants go away and draft definitions; Session 2 refines definitions and adds major relationships. The result: 15–25 mutually exclusive, clearly defined groupings. This takes days, not weeks.

> The subject area model is used to scope iterations: if answering the first business questions requires Customers, Sales, and Products but not HR or Financials, the first iteration of the DW can exclude those subject areas entirely — reducing scope and delivering value faster.

## Business data model (6-step process)

A complete business data model can take 6–12 months. In practice, build only what is needed for the current DW iteration:

1. **Identify relevant subject areas** — use the subject area model to limit scope to those needed for the business questions being asked
2. **Identify entities and establish identifiers** — define entities (person, place, thing, event, or concept of interest) within the scoped subject areas; assign a business identifier to each
3. **Define relationships** — express the business rules as relationships (one-to-many, many-to-many) between entity pairs
4. **Add attributes** — add non-key attributes; source: existing reports, interviews, source system schemas
5. **Confirm model structure** — verify 3NF: every attribute depends on the key, the whole key, nothing but the key
6. **Confirm model content** — validate with business representatives; the model represents business rules, not technical constraints

**Attribute selection tip:** in the business model, use date attributes instead of derived values (e.g., `EstablishmentDate` not `StoreAge`). The derived value is computed in the technology model where the calculation algorithm can be defined and agreed.

## 8-step DW model transformation

Starting from the business data model, apply these 8 steps to arrive at the data warehouse system model:

### Step 1 — Select the data of interest

Select which attributes from the business data model actually go into the DW for this iteration. The business data model may have hundreds of attributes; the DW only needs those that answer the business questions.

**Decision rules:**

| Data type | Decision rule | Rationale |
|---|---|---|
| **Transactional data** | Include if in doubt | May be purged from source; spans years; integration is simple |
| **Reference data** | Exclude if in doubt | Can retrieve missing reference later from current records |
| **Derived-field components** | Always include | Algorithm may change; users drill down to components |

Including unnecessary data has four costs: development (each element must be defined, integrated, quality-checked), load time, storage (history multiplies the footprint), and query performance (wider rows).

### Step 2 — Add time to the key

The business data model is *point-in-time*. The DW is *over-time* (historical). Every entity that can change over time gets a time component added to its key:

```
Business model:  Dealer { DealerID, DealerName, City, State, CreditHoldIndicator }
DW model:        Dealer { DealerID, MonthYear, DealerName, City, State, CreditHoldDays }
```

Adding time to the key converts some one-to-many relationships into many-to-many — a Sales Area that moves between Sales Territories over time needs an associative entity with `EffectiveDate`.

**Five options for handling cascading foreign keys with time:**

| Option | How it works | Best when |
|---|---|---|
| Dual cascaded FK | Time component cascades into all child FKs | Few levels, stable hierarchies |
| Serial key | Generate surrogate for each time-stamped parent row | Simplifies child FKs |
| Programmatic RI | Enforce RI in code, not DBMS; date becomes non-key in parent | Recommended default — easy modeling |
| Separate history entity | Base entity holds current; history entity holds snapshots | High volatility |
| Segregate dynamic data | Base entity holds immutable attrs; attributive entity holds changing attrs | Highest volatility |

**Important distinction:** a time component can mean a *point in time* (snapshot: inventory level on Dec 31) or a *period of time* (flow: units received in December). These must be separate entities — mixing them in one key violates 1NF.

### Step 3 — Add derived data

Pre-compute derived fields in the DW so that every consumer uses the *same definition*. This is primarily a business consistency step, not a performance step.

Examples:
- **Net Sales Amount** — the definition (what is included/excluded) must be agreed once and computed once
- **Credit Hold Days** — depends on business rules about weekends, holidays, factory shutdown days; pre-computing it in the DW forces this definition to be settled
- **Seasonal indicators** — "Christmas Season Flag" in the Date table (starts day after Thanksgiving, ends Dec 24) lets every mart apply the same season definition without each team hardcoding dates

### Step 4 — Determine granularity level

Define what one row in each entity represents. The DW may retain monthly snapshots rather than daily or transaction-level rows, depending on the volume vs. precision trade-off.

### Step 5 — Summarize data

Create pre-aggregated summary tables for common query patterns. Summaries speed up data mart delivery at the cost of storage. The detailed DW rows are always retained; summaries are additive layers.

### Step 6 — Merge entities

Combine entities that are always queried together into one table to reduce join overhead. Example: merging Make, Model, and Series into one Automobile Reference entity if they are always accessed together.

### Step 7 — Create arrays

For cross-tab analysis patterns (e.g., comparing 12 months across in a single row), create array structures where each column represents one time period. This is now largely handled in the data mart or reporting layer rather than in the DW itself.

### Step 8 — Segregate data

Separate data by volatility (frequently changing vs. rarely changing attributes) and by usage pattern to minimize table widths and improve query performance.

## From DW model to data mart

The 3NF DW model is not the end-user access layer. After the 8 transformation steps, the DW system model is used as the *source* for generating dimensional data mart schemas:

```
Business Data Model (3NF, business-oriented, tech-independent)
        │  8-step transformation
        ▼
DW System Model (3NF, time-variant, data-selected)
        │  subject-specific extraction
        ▼
Data Mart (star schema / dimensional — for end-user query ease)
```

A major advantage of this approach: the same DW can feed multiple different star schemas for different user groups, without each user group's design decisions affecting the others. If a mart needs to be redesigned for a new business question, the DW layer is unchanged — only the delivery process changes.

## See also

- [Corporate Information Factory](corporate-information-factory.md) — the full CIF architecture context
- [Inmon (3NF / Enterprise DWH)](inmon.md) — Inmon's original DWH concept
- [Kimball (Dimensional Modeling)](kimball.md) — the dimensional / star schema alternative
- [Keys — Natural, Surrogate, Business](../data-modeling/keys.md) — surrogate key strategy in the DW context
- [Slowly Changing Dimensions](../data-modeling/scd.md) — how the dimensional layer handles the history the DW captures
