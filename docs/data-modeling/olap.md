# OLAP — Multidimensional Analysis

> *Source: Paulraj Ponniah, "Data Warehousing Fundamentals" (2001), Ch. 15*

## Why traditional tools fail for complex analysis

A data warehouse stores historical, integrated data optimized for analysis — but the tools that suffice for OLTP and basic reporting break down when users need serious multidimensional analysis:

**Report writers** — can generate and format SQL-based reports, but cannot drill down interactively, cannot rotate rows and columns, and cannot aggregate on the fly. Once formatted, the result is static.

**Spreadsheets** — support "what if" analysis and can show rows, columns, and pages, approximating three dimensions. But building four-dimensional analysis with multiple hierarchy levels in a spreadsheet requires enormous manual effort. When a user wants to change the roll-up or drill-down path, they must rebuild the analysis.

**SQL** — too verbose for interactive analysis. Complex multidimensional queries translate into multiple SQL statements with full table scans, multiple joins, aggregations, groupings, and sorts. Even a trained analyst cannot keep up the necessary pace. SQL is notably weak at time-series calculations and cross-dimensional comparisons.

The core problem is the **interactive chain of thought**. A real analysis session looks like:

1. Query: overall sales dip in last 5 months → result shows profitability is down, not revenue
2. Query: break down by worldwide region → European region stands out
3. Query: break down European sales by country → EU countries show sharp decline
4. Query: by country, month, and product → costs are stable but indirect costs are up
5. Query: identify the indirect cost → additional tax levies on EU products

Each query is informed by the previous result. The user cannot lose momentum — slow response breaks the chain of thought. SQL returns acceptable results on the first query; by the third query with joins across multiple levels of aggregation, it returns too slowly or too verbosely to sustain the analysis.

---

## The definition of OLAP

The term **OLAP (Online Analytical Processing)** was introduced by Dr. E. F. Codd (author of the relational model) in a 1993 paper "Providing On-Line Analytical Processing to User Analysts."

The OLAP Council's definition:

> "OLAP is a category of software technology that enables analysts, managers and executives to gain insight into data through fast, consistent, interactive access in a wide variety of possible views of information that has been transformed from raw data to reflect the real dimensionality of the enterprise as understood by the user."

Four key ingredients: **speed, consistency, interactive access, multiple dimensional views**.

---

## Codd's 12 rules for an OLAP system

1. **Multidimensional conceptual view** — the data model must be intuitively multidimensional; it must match how business users think about their enterprise
2. **Transparency** — the underlying data repository, computing architecture, and source data diversity must be hidden from users
3. **Accessibility** — present a single, coherent view; map logical dimensions to heterogeneous physical stores transparently
4. **Consistent reporting performance** — response time must not degrade as the number of dimensions or database size increases
5. **Client/server architecture** — multiple clients must attach with minimal integration work
6. **Generic dimensionality** — every dimension must have equivalent structure and operational capabilities; no dimension is treated as special
7. **Dynamic sparse matrix handling** — the system must detect sparse data and optimize storage/access dynamically
8. **Multi-user support** — concurrent access with data integrity and security
9. **Unrestricted cross-dimensional operations** — calculations and manipulations across any number of dimensions, with automatic roll-up/drill-down
10. **Intuitive data manipulation** — pivoting, drill-down, and roll-up via point-and-click directly on cells; not through menus
11. **Flexible reporting** — columns, rows, and cells freely rearranged; any dimension displayable with equal ease
12. **Unlimited dimensions and aggregation levels** — at least 15–20 dimensions; practically unlimited aggregation levels within any dimension

Additional rules (1995): drill-through to detail level, treatment of nonnormalized data, incremental database refresh, SQL interface.

---

## The cube metaphor

Dimensional data can be visualized geometrically. For **three dimensions** (product, time, store), the data forms a physical cube:

- X-axis: products
- Y-axis: months
- Z-axis: stores
- Each cell at the intersection: a metric (e.g., sale units)

A **slice** is a 2D plane through the cube — for example, all products and months for one specific store. Displaying that slice on a screen gives you a page of rows (months) × columns (products).

For **more than three dimensions**, a physical cube cannot represent the model. The concept extends to a **hypercube**: an abstract representation of n-dimensional data. The Multidimensional Domain Structure (MDS) diagram represents each dimension as a straight line, allowing 4, 6, or 15 dimensions to be represented and navigated — even though they cannot be drawn as a three-dimensional solid.

---

## Core OLAP operations

### Drill-down and roll-up

Dimensions have hierarchies. In a product dimension, the hierarchy is:

```
Department → Product Line → Category → Subcategory → Product
```

- **Roll-up**: navigate to a higher, more aggregated level. Show sales by Product Line instead of by individual Product.
- **Drill-down**: navigate to a lower, more detailed level. Expand a Product Line into its Categories.
- **Drill-across**: navigate to another OLAP instance or dimension to bring a different analytical view into the session.
- **Drill-through**: leave the OLAP system and access the underlying raw data in the DWH at the lowest grain.

### Slice-and-dice (rotation)

A cube can be rotated so that a different face becomes the visible slice. In the three-dimension example (products × months × stores):

- Initial view: page = one store; rows = months; columns = products
- Rotate 1 (pivot by store): page = one product; rows = months; columns = stores
- Rotate 2 (pivot by product): page = one month; rows = stores; columns = products

Each rotation shows the same data from a different angle, enabling users to spot patterns that are invisible in one orientation but obvious in another.

The operation of rotating and re-slicing is called **slice-and-dice**. It is the core interactive capability that distinguishes OLAP from static reporting.

---

## MOLAP vs. ROLAP

The OLAP concept defines the user experience. The storage model is a separate concern:

### MOLAP — Multidimensional Online Analytical Processing

- Data is extracted from the DWH and loaded into a **proprietary multidimensional database (MDDB)**
- The OLAP engine pre-builds and pre-aggregates all dimension combinations as **multidimensional arrays**
- At query time, the engine reads from these pre-built cubes — no SQL is generated
- For 3 dimensions, the engine pre-computes: the base 3D array, all 2D cross-sections, and all 1D summaries

**Result**: very fast query response; consistent performance regardless of query complexity

**Trade-offs**:
- Proprietary storage format — vendor lock-in
- Data must be extracted, pre-aggregated, and loaded — expensive build process; most MOLAP systems refresh **monthly**, not daily
- Handles summarized data well; handles large volumes of detail data poorly
- Sparse matrix handling required (stores with zero sales on Sundays leave empty cells)

### ROLAP — Relational Online Analytical Processing

- Data stays in the relational DWH; no separate physical cube is pre-built
- An **analytical server** intercepts multidimensional user requests and translates them to **complex SQL** against the DWH
- Cubes are created **dynamically on the fly** per query
- A metadata layer maps dimensions to relational tables

**Result**: works on larger data volumes; closer to the DWH; SQL-based

**Trade-offs**:
- Slower than MOLAP for complex aggregations (no pre-computation)
- Cross-dimensional drill-across can be difficult
- Drill-through to detail is easier (the detail is already in the same relational database)

### DOLAP — Desktop OLAP

A variant of ROLAP: a small multidimensional dataset is extracted to the user's desktop machine, where analysis runs locally. Provides portability but limited to the subset downloaded.

### ROLAP vs. MOLAP — the key tradeoff

| | ROLAP | MOLAP |
|---|---|---|
| Data storage | Relational tables in DWH | Proprietary multidimensional arrays |
| Data volumes | Large (detailed + light summary) | Moderate (summary-focused) |
| Query speed | Moderate (SQL generated dynamically) | Fast (pre-built cubes) |
| Complex calculations | Limited library | Extensive library |
| Refresh frequency | Can be frequent (daily) | Infrequent (monthly typical) |
| Drill-through | Easier (data already in RDB) | Harder (must cross-system boundary) |
| Sparse matrix handling | Implicit (nulls in relational tables) | Explicit sparse matrix handling needed |

The practical rule: MOLAP for intensive, complex queries where response time is critical; ROLAP for larger data volumes and more frequent refresh needs. As technology advanced, hybrid HOLAP engines appeared — storing aggregations multidimensionally but detailed data relationally.

---

## Why OLAP cannot bypass the data warehouse

A tempting shortcut is to build the OLAP system on top of operational source systems directly. This does not work:

1. **OLAP needs integrated data.** Operational systems have inconsistent formats, naming conventions, and representations for the same business concept. Integration must happen before OLAP.

2. **OLAP needs historical data.** Operational systems retain limited history. OLAP analysis typically spans 5–10 years. Historical data must be accumulated and consolidated first.

3. **OLAP needs multi-level summarizations.** Creating aggregations at every possible dimensional level combination from multiple operational systems simultaneously is computationally untenable. Consolidation in a DWH must precede summarization.

4. **Multiple OLAP instances multiply the problem.** Marketing OLAP, Finance OLAP, Inventory OLAP — each would need its own separate extraction interface from source systems. A single DWH as the consolidated source is far more maintainable.

The correct data flow is always: Operational systems → Data Warehouse → OLAP system.

---

## OLAP characteristics summary

An OLAP system:
- presents a **multidimensional logical view** of the data to users
- enables **interactive query and complex analysis** in a speed-of-thought cycle
- allows **drill-down, roll-up, drill-across, and drill-through** along any dimension or hierarchy
- supports **intricate calculations**: moving averages, year-over-year comparisons, share of total, growth percentages, trend analysis
- presents results in **multiple forms**: tables, charts, graphs, pivot views
- maintains **consistent response times** regardless of query complexity or database size

---

## Modern equivalents

The 2001 landscape of separate MOLAP servers and ROLAP engines has largely been absorbed into integrated platforms:

| Classic OLAP concept | Modern equivalent |
|---|---|
| MOLAP (pre-built proprietary cubes) | Power BI / Fabric Semantic Model (VertiPaq in-memory columnar engine — pre-aggregated, compressed) |
| ROLAP (dynamic SQL translation) | DirectQuery mode in Power BI; Tableau live connections; Analysis Services DirectQuery |
| OLAP calculation language | DAX (Data Analysis Expressions) — Power BI/Fabric's equivalent of OLAP formula libraries |
| Hypercube navigation (drill-down/up) | PBI matrix visuals with drill hierarchy; Fabric notebook multidimensional analysis |
| OLAP separate from DWH | Semantic model sits on top of Fabric SQL Warehouse — same Fabric workspace, no separate platform |
| Drill-through to detail | Power BI drill-through pages; direct queries to the Fabric warehouse for row-level detail |

In the modern Fabric architecture, the **Fabric SQL Warehouse** is the DWH (integrated, historical, non-volatile), and the **Semantic Model** is the OLAP layer. The semantic model is built on top of the warehouse, conforming to the principle that OLAP must never bypass the DWH — it always reads from the integrated, consolidated warehouse, not directly from operational sources.

---

## Related pages

- [Dimensional modeling — Star schema](dimensional-modeling.md)
- [Kimball — Dimensional Modeling](../traditional-dwh/kimball.md)
- [Metadata in Data Warehousing](../traditional-dwh/metadata.md)
- [Inmon vs Kimball vs Data Vault](../traditional-dwh/comparison.md)
