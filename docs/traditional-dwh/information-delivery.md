# Information Delivery & User Classes

> *Primary source: Paulraj Ponniah, "Data Warehousing Fundamentals for IT Professionals" (2010), Ch. 14*

A data warehouse serves a heterogeneous user community. Different users have completely different information needs, computing proficiencies, and usage patterns. Designing an information delivery system that serves every user requires understanding these differences.

---

## Five classes of DWH users

### Tourists — Senior executives

A tourist arrives with broad awareness of the territory but no time for detailed exploration. Senior executives visit the data warehouse to check specific key performance indicators. If a KPI is out of line, they drill in further; otherwise they move on quickly.

**What they need:**
- Status of key indicators at routine intervals (daily, weekly)
- Ability to identify items of interest without difficulty or navigation complexity
- Fast response — seconds, not minutes
- Capability to drill down when something demands attention
- Pre-built dashboards with pre-defined expectations (traffic-light views)

**Tools:** executive dashboards, mobile BI apps, simple drill-down reports

---

### Operators — Departmental and line managers

Operators care about *now*. They monitor current performance, not historical patterns. A line manager wants to know what is happening today in the warehouse, in the distribution system, or on the production floor. Operators are experienced OLTP system users and expect fast response to detailed queries.

**What they need:**
- Current state of performance metrics — daily or more frequent updates
- Immediate answers based on reliable current data
- Quick access to very detailed records
- Simple and straightforward interface
- Rapid analysis of the most current data (not deep historical analysis)

**Tools:** operational reporting tools, alerts, near-real-time data mart access

---

### Farmers — Routine analysts

Farmers know the terrain intimately. Technical analysts, marketing analysts, and financial analysts who run the same types of reports month after month are farmers. Their requirements are standard, predictable, and repeatable.

**What they need:**
- Quality data properly integrated from source systems
- Ability to run predictable queries easily and quickly
- Routine reports at predictable intervals (monthly profit by product, quarterly sales by territory)
- Mostly current data with simple comparisons to historical reference points
- Precise, smaller result sets — not exploratory dumps

**Tools:** report builders, parametric query templates, scheduled report distribution

---

### Explorers — Researchers and advanced analysts

Explorers go where few venture. They have no pre-set queries and no predictable usage pattern. They may spend several days in intensive analysis and then disappear for months. The occasional breakthrough from an explorer — a pattern or insight nobody anticipated — is worth the long periods of silence.

**What they need:**
- Totally unpredictable and intensely ad hoc queries
- Ability to retrieve large volumes of detailed data for analysis
- Capability to perform complex analysis across many dimensions
- Support for unstructured, completely new and innovative queries
- Long, protracted analysis sessions — not quick lookups

**Tools:** SQL workbenches, ad hoc OLAP tools, Python/R notebooks, self-service analytics platforms

---

### Miners — Data scientists and knowledge workers

Miners prove or disprove hypotheses, or discover unsuspected patterns entirely. They are a special breed — statisticians, machine learning practitioners, domain experts with analytical training. Many enterprises bring in outside consultants for data mining projects.

**What they need:**
- Access to mountains of historical data going back many years
- Ability to wade through large volumes to extract meaningful correlations
- Capability to export data into formats suitable for specialized mining techniques (flat files, matrices, graph representations)
- Two modes of operation: (1) hypothesis-driven (prove/disprove a known question); (2) discovery-driven (find patterns without preconceptions)

**Tools:** Python/R, specialized ML platforms, statistical packages, graph databases

---

## Summary of user classes

| Class | Organizational level | Usage frequency | Data depth | Query type |
|---|---|---|---|---|
| **Tourist** | C-suite, VP | Regular, brief | Aggregated KPIs | Pre-built, drill if needed |
| **Operator** | Dept. manager, line manager | Daily, fast | Current detail | Structured, fast-response |
| **Farmer** | Analyst (marketing/finance/ops) | Scheduled, routine | Current + some history | Predictable, parametric |
| **Explorer** | Researcher, advanced analyst | Bursty, unpredictable | Large volumes of detail | Ad hoc, open-ended |
| **Miner** | Data scientist, statistician | Project-based | Deep history | Algorithmic, statistical |

---

## Four information delivery methods

Regardless of user class, information is delivered through four mechanisms:

| Method | Description | User class |
|---|---|---|
| **Queries** | User-driven ad hoc questions against the DWH or mart; SQL or GUI-based | Farmers, Explorers |
| **Reports** | Pre-formatted outputs produced at scheduled intervals; canned or parametric | Tourists, Operators, Farmers |
| **Analysis** | Interactive exploration — slicing, dicing, drill-down, drill-across; OLAP tools | Tourists (light), Farmers, Explorers |
| **Applications** | Purpose-built apps with embedded analytics (CRM with embedded DWH metrics, pricing tools) | Operators, specialized Farmers |

---

## Tool selection considerations

Matching tools to user classes requires evaluating:

- **Response time requirements** — Tourists and Operators need seconds; Explorers and Miners accept minutes
- **Metadata interface** — Tourists need business-term search; technical users can navigate schemas directly
- **Query construction** — Operators and Tourists need GUI-driven forms; Explorers and Miners need SQL access
- **Data volume** — Explorers and Miners need bulk extract capability; Tourists and Operators need pre-aggregated views
- **Access patterns** — scheduled (Farmers), on-demand (Operators), interactive (Explorers), algorithmic (Miners)

---

## Related pages

- [Corporate Information Factory (CIF)](corporate-information-factory.md)
- [OLAP — Multidimensional Analysis](../data-modeling/olap.md)
- [Star Schema](../data-modeling/star-schema.md)
- [Fact Table Types](../data-modeling/fact-types.md)
