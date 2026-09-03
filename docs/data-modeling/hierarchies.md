# Hierarchies in Data Warehousing

> *Primary source: Claudia Imhoff, Nicholas Galemmo, Jonathan Geiger, "Mastering Data Warehouse Design" (2003), Ch. 7*

Hierarchies are central to business data. They define organizational chains of command, product classifications, geographic territories, customer ownership structures, and anything else that can be broken into levels of detail. Without hierarchies, analytical systems return a morass of undifferentiated detail — you cannot "see the forest for the trees."

---

## Hierarchy terminology

A hierarchy is a special type of parent-child relationship where a child represents a lower level of detail (granularity) of its parent. The standard vocabulary:

- **Node** — any member in the hierarchy
- **Root node** — the topmost node; has no parent
- **Leaf node** — the bottommost nodes; have no children
- **Parent node** — a node that has children
- **Child node** — a node that has a parent

A parent (except the root) may also be a child. A child (except a leaf) may also be a parent.

---

## Types of hierarchies

### By depth

**Known depth (fixed-level)** — The most common in business. Every path from root to leaf has the same number of levels, and each level has a name: Region → District → Territory → Customer. Sales hierarchies, product hierarchies, and organizational hierarchies are typically known-depth.

**Unknown (variable) depth** — No fixed level count; the same level structure (e.g., "assembly → subassembly") repeats. A bill of materials (Boeing 777: airframe → wing → tip light → bulb...) may exceed 40 levels. Levels cannot be individually named because the depth varies.

### By parentage (simple vs. complex)

**Simple hierarchy** — every child has one and only one parent. This is the desirable structure for reporting: each child is counted exactly once in any aggregation.

**Complex hierarchy** — a child may belong to multiple parents. Example: a company partially owned by two parent companies. Or a standard product shared across multiple brands. Complex hierarchies require allocation factors to distribute contribution across parents.

### By texture (balanced vs. ragged)

**Balanced (smooth)** — all leaves exist at the lowest level, and every parent is exactly one level above its child. Every region has districts, every district has territories, every territory has customers.

**Ragged (sparse)** — depth varies. A very large customer may report directly to a region (skipping district and territory), while smaller customers are three levels down. The hierarchy has a maximum number of levels and named levels, but individual paths may skip intermediate levels.

---

## Hierarchy history

Two types of historical change can affect a hierarchy:

1. **Entity change** — an attribute of a node changes (e.g., Sales District Name changes)
2. **Relationship change** — a node is reassigned to a different parent (e.g., Sales District moves from one Region to another)

The data warehouse must track both. Relationship changes require effective/expiration dates on the parent-child links, not just on the entity rows themselves.

---

## Implementation: two physical structures

### Flattened (non-recursive) tree

Arranges hierarchy levels horizontally in different columns. Used for **known-depth hierarchies only**.

```
Sale → Product (FK) → Department (FK) → Region (FK) → Corporation
```

- Simple to query with standard SQL — each level is a directly accessible column
- Cannot handle ragged hierarchies well (requires placeholder rows for missing levels)
- Schema change required when a level is added or removed — all standard reports must change

Best for data mart delivery of known-depth hierarchies.

### Recursive tree

A table with a parent foreign key and a child foreign key — both pointing to the same table. Works for **any type of hierarchy**, including unknown depth and ragged.

```sql
-- All hierarchy levels (HQ, Planning Group, Sold-To, Ship-To) in one table
CREATE TABLE Customer (
    CustomerKey   INT NOT NULL,    -- surrogate PK
    CustomerBK    VARCHAR(50) NOT NULL,
    CustomerType  VARCHAR(10) NOT NULL,  -- role: HQ, PG, SO, SH
    CustomerName  VARCHAR(200) NOT NULL
);

CREATE TABLE CustomerHierarchy (
    ParentKey   INT NOT NULL,   -- FK → Customer
    ChildKey    INT NOT NULL,   -- FK → Customer
    EffDt       DATE NOT NULL,
    ExpDt       DATE NOT NULL
);
```

**Key rule:** all hierarchy levels must live in the same table. If Customer HQ, Planning Group, Sold-To, and Ship-To are in separate tables, the recursive tree requires 8 foreign keys (parent + child to each table) — an unmaintainable structure. Fold them all into one Customer table using a row-type column.

**Querying a recursive tree** requires recursive traversal — a routine that calls itself. SQL extensions (Oracle's CONNECT BY, SQL Server's recursive CTE) can traverse the tree, but sorting and extracting ancestor relationships still require code.

---

## Complex hierarchies: retaining ancestry

When a hierarchy is received as a denormalized flat column (e.g., a product code with embedded division + brand + product group), a product group code may appear under multiple brands — the same code ENTR meaning "Entrée" under both Lazy Guy and Like-A-Chef brands.

**Solution: prefix each child code with its ancestor codes.** This makes each node's identity within the context of its lineage explicit:

| Level | Without ancestry | With ancestry |
|---|---|---|
| Division | `01` (Frozen) | `01` |
| Brand | `035` (Lazy Guy) | `01035` |
| Product Group | `ENTR` | `01035ENTR` |
| Product Group | `ENTR` | `01160ENTR` |

Now "01035ENTR" and "01160ENTR" are unambiguous even though both describe an "Entrée" product group. This approach makes the child a **dependent entity** — uniquely identified within the context of its ancestry.

---

## Bridge tables: connecting levels

A bridge table contains key pairs that link detail-level data to any ancestor level. It enables rolling up detailed data (sales at SKU level) to any higher planning level (product group, brand, division) without recursive traversal.

**Structure:**

| Column | Purpose |
|---|---|
| ParentKey | Surrogate key of any ancestor |
| ChildKey | Surrogate key of the leaf (or any descendant) |
| Level | Level of the parent (1 = root) |
| Distance | Number of levels between parent and child (0 = identity row) |
| Bottom | True if child is a leaf node |
| Sequence | Sort order for proper hierarchical report sequence |
| EffDt / ExpDt | Effective and expiration dates of the relationship |

**Identity rows:** each node has a bridge row with `ParentKey = ChildKey` and `Distance = 0`. This covers the case where a plan exists at the same level as the detail.

**All-ancestors pattern:** for each leaf (SKU), insert one row for every ancestor: SKU→Division, SKU→Brand, SKU→ProductGroup, SKU→StandardProduct, SKU→SKU. A sales plan at any level can then be joined to actual sales at the SKU level through this table.

This same bridge structure is what's produced by **exploding** a recursive tree.

---

## Exploding a recursive tree for data mart delivery

Most reporting tools cannot process recursive tree structures. The solution is to **explode** the tree: traverse it once and generate a nonrecursive table of every possible ancestor-descendant relationship pair.

The explosion result is identical in structure to the bridge table described above. It eliminates recursive queries — all hierarchical reports can be generated with simple, single-pass SQL.

**Updating the exploded tree:** when a node moves to a new parent:
1. The direct parent-child relationship date changes
2. All ancestor-descendant relationships derived through that link must have their date ranges updated to reflect the intersection of the parent's and child's effective dates

This is a time-sensitive intersection calculation that requires procedural code (ETL/3GL), not plain SQL.

---

## Allocation factors in complex hierarchies

When a child has multiple parents, simple aggregation is not correct — the child would be counted fully under each parent. Instead, each parent-child relationship carries a **factor** (a fractional value):

```
Revenue contribution of C to A = Factor(B→A) × Factor(C→B)
e.g., 0.5 × 0.3 = 0.15  →  C contributes 15% of its revenue to A
```

When creating the physical schema, allocate sufficient decimal precision to factor columns — factors are multiplied (not summed) across levels, and precision errors compound.

---

## Multiple hierarchies per entity

A single entity often participates in multiple hierarchies simultaneously:

- A product has a **marketing hierarchy** (for promotional analysis) and a **finance hierarchy** (for cost analysis)
- A customer has a **sales hierarchy** (sold-to buyer) and a **distribution hierarchy** (ship-to location)
- An employee has a **reporting hierarchy** (org chart) and a **commission hierarchy** (territory assignments)

Each hierarchy is independent. A fact table record (a sale) is connected to all relevant hierarchies through the appropriate dimension keys. The bridge table or exploded tree must be built separately for each hierarchy.

---

## Ragged hierarchies: modeling choices

For a ragged hierarchy (varying depth, named levels):

**Option A — recursive tree:** schema never changes when levels are added or removed; queries require traversal code; ideal for the data warehouse integration layer.

**Option B — flattened tree with placeholder rows:** simpler SQL but requires artificial "filler" nodes for missing levels; hard to maintain.

**Best practice:** use recursive tree in the data warehouse; explode to a flattened structure when delivering to data marts where known-depth queries are required.

---

## Modern equivalents

| Classic hierarchy concept | Modern equivalent |
|---|---|
| Recursive tree in 3NF DWH | Parent-child table in Fabric SQL Warehouse; Power BI parent-child DAX hierarchy functions |
| Flattened hierarchy for data mart | Flat dimension table in star schema Gold layer |
| Bridge table for multi-level rollup | Bridge table pattern — same structure in modern DWH |
| Ragged hierarchy | Power BI / DAX `PATH()`, `PATHITEM()` for ragged parent-child navigation |
| Exploded tree | Pre-computed ancestor table via recursive CTE during ETL |
| Unknown-depth BOM | Recursive CTE in Fabric SQL Warehouse; `CONNECT BY` in Oracle; graph DB for very complex cases |

---

## Related pages

- [Keys in data warehousing](keys.md)
- [Slowly Changing Dimensions (SCD)](scd.md)
- [Star Schema](star-schema.md)
- [OLAP — Multidimensional Analysis](olap.md)
