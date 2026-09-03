# The Four Principles of Data Mesh

Data mesh rests on four principles. The first three describe what to build; the fourth describes
how to coordinate it.

## Why centralized data architectures fail first

Before the principles make sense, it helps to understand the failure mode they address.

The dominant pattern — a central data engineering team owning a data lake or warehouse on behalf
of all business domains — degrades predictably as the organization grows:

| Failure mode | What happens |
|---|---|
| **Bottleneck ownership** | Every new data need flows through one team; delivery slows as domains multiply |
| **Context loss** | A central team cannot understand Payments, CRM, and Logistics as deeply as the domain teams living inside them — data becomes technically present but semantically wrong |
| **"Big Ball of Mud"** | Data lakes intended to simplify integration accumulate unmanaged, undiscoverable, ungoverned datasets |
| **Governance decay** | No single owner enforces quality, definitions, or access policies across a lake owned by everyone and no one |

The anti-pattern is not a technology failure — it is a mismatch between a centralized
organizational model and a decentralized business.

---

## Principle 1 — Domain Ownership

**Each business domain owns the data it produces.**

This mirrors the shift microservices made for application services: instead of a central
infrastructure team owning all services, each product team owns its own.

- A domain team (e.g. Payments, Customer, Fleet) is responsible for producing, maintaining, and
  exposing the data that originates in its bounded context
- The team must hold both the **business competency** (what the data means) and the **technical
  competency** (how to produce and serve it reliably)
- Domain boundaries are a judgment call — there is no universal formula

The organizational consequence: domain teams can no longer hand off raw data to a central team
and stop caring. Data production is a first-class responsibility alongside feature delivery.

---

## Principle 2 — Data as a Product

**Domain data is a product, not a by-product.**

Domain ownership alone is not enough. A domain producing data for its own internal use is
different from a domain producing data that other domains depend on. This principle reframes
the output.

A data product has:

| Attribute | Meaning |
|---|---|
| **Stable interface** | Schema, SLA, versioning that downstream consumers can depend on |
| **Accountable owner** | A named team responsible for quality and reliability |
| **Discoverability** | Findable and self-describing without contacting the producing team |
| **Access policies** | Enforced at the product level, not managed ad hoc |

The contrast with the status quo: in most organizations, data is a by-product of operational
systems — available if you know to look and can figure out the schema, but not designed for
external consumption.

---

## Principle 3 — Self-Serve Data Platform

**The organization provides shared infrastructure that makes it cheap and fast for domain teams
to build and operate data products.**

If every domain must produce data products, you cannot also require every domain team to become
experts in data infrastructure. The self-serve platform is the platform engineering layer applied
to data.

What it provides:

- Storage and compute a domain team can provision without a central team ticket
- Standardized tooling for schema registration, cataloguing, quality checks, and lineage
- Automated pipelines for partitioning, access control, and monitoring
- CI/CD templates for deploying and validating data products

Without this layer, decentralization redistributes work but not capability — domain teams get
the responsibility without the means.

---

## Principle 4 — Federated Computational Governance

**A set of global policies that all domains participate in defining and that are enforced by the
platform, not by manual process.**

Distributed ownership creates a coordination problem: if each domain evolves its data
independently, the platform becomes a collection of islands. This principle resolves the tension
without reintroducing a central authority.

Three governance concerns that must stay consistent across domain boundaries:

| Concern | Why it matters |
|---|---|
| **Discoverability** | Data products must be findable and self-describing — consumers should not need to call the producing team to evaluate a dataset |
| **Access control** | Security policies must be enforced consistently, even across independently managed domains |
| **Semantic consistency** | Core concepts (Customer, Transaction, Product) must mean the same thing across domain schemas |

Federated governance is the hardest principle to implement: it requires cross-domain agreement
without a central authority that can impose it. The word "computational" signals that policy
enforcement is automated by the platform — not done manually by a governance committee.

---

## How the principles relate

```
Self-serve platform
  │  (provides the infrastructure so that...)
  ▼
Domain teams can own their data
  │  (and treat it as...)
  ▼
Data products
  │  (coordinated across domains by...)
  ▼
Federated governance
```

The principles depend on each other. Domain ownership without a self-serve platform creates
unfunded mandates. Data products without federated governance create silos. Governance without
domain ownership creates a central bottleneck by another name.
