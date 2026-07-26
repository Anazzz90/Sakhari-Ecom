# 0002. Modular monolith over microservices

**Decision ID:** ADR-0002
**Status:** Accepted
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
The backend must serve four client applications (Customer Mobile, Customer Web, Worker App, Admin/Ops Dashboard) and is built and operated by a single developer with AI assistance, at a launch scale of low thousands of orders per day across a handful of stores. `00-Architecture-Principles.md` already commits, at the principle level, to decomposing the system around business capabilities with strict data ownership (Principles 4.4, 4.5) and names the modular-monolith-over-microservices tradeoff explicitly in its Section 6. `01-Architecture-Design-Specification.md` Section 7 describes this shape as the system's high-level architecture and flags it, in Section 8, as a decision awaiting formal ADR backfill.

## Problem
Should the backend be deployed as a single unit internally organized by capability, or as multiple independently deployed services (microservices) — one deployable per capability — given the team size, scale, and evolvability requirements this project actually has?

## Options considered
1. **Microservices from day one** — one independently deployed service per business capability (Identity, Catalog & Inventory, Ordering, Payments, Dispatch, etc.). Offers independent scaling and deployment per capability, at the cost of distributed transactions, network-partition handling, service discovery, and per-service operational overhead.
2. **Modular monolith** — a single deployable backend, internally organized into capability-aligned modules with enforced data ownership, but no network boundary between them.
3. **Undifferentiated monolith** — a single deployable with no enforced internal structure, business logic and data access organized ad hoc.

## Decision
Adopt a modular monolith: one deployable backend service, internally organized into modules aligned to business capabilities (Principle 4.4), each exclusively owning its own persisted data (Principle 4.5, and see ADR-0009).

## Rationale
Microservices' benefits — independent scaling, independent deployment, fault isolation between capabilities — are not worth their cost at this project's actual scale and team size: one developer cannot safely operate, debug, and coordinate deployment across many independently versioned services, and the business does not yet generate load that a single well-resourced instance cannot serve (Constitution Section 6, Design Goal: operational simplicity). An undifferentiated monolith was rejected because it would erode exactly the capability boundaries Principle 4.4 requires, with nothing to enforce them as the system grows. A modular monolith gets the organizational benefit of capability boundaries without paying the distributed-systems cost until evidence says it's needed (Constitution Section 13).

## Consequences
- Single deployment pipeline, single runtime to monitor and debug, no distributed transactions or network-partition handling to design around.
- Internal module boundaries must be maintained by discipline and review (ADR-0009), not by a network boundary — this is a real, accepted cost of the choice.
- All new backend capabilities are added as a new or existing capability-aligned module inside the one deployable, never as a new standalone service, unless a future ADR revisits this decision.
- Keeps the door open to extracting an individual capability into its own service later without redesigning the boundary itself (Constitution Section 13, ADS Section 15).

## Future reconsideration conditions
Order volume or store count grows beyond what a single, well-resourced backend instance can serve via vertical scaling; a specific capability (most plausibly Ordering or Catalog & Inventory per ADS Section 13/15) shows evidenced, sustained load or team-scaling pressure a modular monolith cannot absorb; or the team grows beyond one developer such that independent deployment cadence per capability becomes worth its coordination cost.

## Related
- SRD section(s): Section 1.2 (Design Constraints — one developer, launch scale), Section 2 (Application Structure)
- Related DDD entity/data area(s): n/a directly — a runtime/deployment decision, not a data-ownership one; must stay consistent with `03-decomposition/data-ownership-map.md` once authored
- Related architecture principle(s): 4.4, 4.5; Constitution Section 6, Section 13
- Related ADRs: 0009, 0011
