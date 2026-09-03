# Readiness & Implementation Reality

## When data mesh is the right approach

Data mesh is designed for the problems of scale: many business domains, many data consumers,
and a central data team that has already become the bottleneck. Signs you may be there:

- New data requests take weeks because every one goes through one team
- Domain teams complain that the central team doesn't understand their data
- Data quality problems are discovered by consumers rather than producers
- The data lake has datasets nobody owns and nobody cleans up

## When data mesh is the wrong approach

| Situation | Why data mesh does not help |
|---|---|
| **Small organization** with one or two data domains | Coordination overhead exceeds the benefit; a well-run central team is faster |
| **No domain ownership culture** | The technical pattern will not survive an org model that doesn't support it |
| **Greenfield platform** | The central model has not yet become a bottleneck — premature decentralization adds complexity |
| **Active org transformation underway** | Layering data mesh onto a simultaneous monolith-to-microservices migration multiplies failure risk |
| **Domains lack technical capability** | Self-serve platform cannot fully abstract infrastructure complexity; domains need baseline DevOps fluency |

## Readiness checklist — 10 dimensions

Assess honestly before starting. A "no" on the first three is a hard blocker; the rest are
risks to manage.

| Dimension | Hard blocker? | What to check |
|---|---|---|
| **Team structure** | Yes | Domain-aligned teams exist or are actively forming — not purely function-aligned |
| **DevOps culture** | Yes | Domain teams have CI/CD fluency; they deploy software, not just write it |
| **Product thinking** | Yes | Teams treat their outputs as products with SLAs — not internal artifacts that "someone else handles" |
| Cloud maturity | Risk | Domain teams can provision and operate cloud infrastructure without a central ops ticket |
| Domain-driven design familiarity | Risk | Teams understand bounded contexts; they can draw a domain boundary and defend it |
| Data quality culture | Risk | Teams already care about the quality of data they produce; enforcement is not entirely new |
| Executive sponsorship | Risk | Org restructuring requires authority above the data team — without it, data mesh stays a pilot |
| Tolerance for transition cost | Risk | Initial data products take 3+ sprints; leadership must accept short-term slowdown |
| Trust culture | Risk | Federated governance requires cross-domain agreement without a central enforcer — requires a culture of collaboration |
| Timing | Risk | No other major transformation (platform migration, monolith breakup) is running simultaneously |

## Implementation reality

From documented real-world implementations (source: datamesh-architecture.com):

### Cognitive overload is real

Domain engineers taking on data responsibility on top of frontend, backend, and operations need
a **genuinely capable self-serve platform** — not just a policy that says "you own your data now."
If the platform doesn't absorb the infrastructure complexity, teams get responsibility without
capability, which fails.

### Data quality equals trust

A low-quality data product erodes adoption faster than having no product at all. Other domain
teams stop trusting and stop consuming — then build workarounds. Monitoring and SLAs are not
nice-to-haves; they are the core of what makes a domain's data trustworthy across the mesh.

### Adoption does not follow demand

A data product that was requested by consumers can still go unused after delivery — the same
failure mode as feature software that nobody ends up using. Domain teams need to validate that
consumers actually use products, not just that products technically exist and pass CI.

### Initial velocity is slow

First data products take 3+ sprints as teams learn new tooling, cloud billing, file format
optimization, and operational patterns they have never owned before. Plan for this — it is not a
sign the approach is wrong, it is the expected onboarding cost.

### Not every domain needs full product maturity

Some domains are better served by direct SQL access to their operational database, or by
commercial SaaS products that already expose clean data APIs. Forcing every domain through the
full Gold-layer data product pattern over-engineers domains that do not need it. Match the
investment to the domain's external data demand.

### Timing is a hard constraint

Organizations that ran data mesh simultaneously with a monolith-to-microservices migration
report high failure rates. Dedicated organizational capacity for the data mesh transition is a
prerequisite, not a preference. If bandwidth is split, the organizational change stalls.

## Summary

| | |
|---|---|
| **Best fit** | Large organizations, many domains, central team already a bottleneck, DevOps culture in place |
| **Poor fit** | Small teams, greenfield, no product culture, concurrent major transformation |
| **Starting point** | Pick one domain with clear boundaries and high data demand — deliver one data product end-to-end before scaling |
