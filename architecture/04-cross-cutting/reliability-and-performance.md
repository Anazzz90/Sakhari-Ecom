# Reliability & Performance

**Stability:** Evolving — targets are revisited as real traffic data becomes available.
**Status:** Skeleton — pending authoring.

## Purpose
Sets system-wide availability, latency, and resilience expectations, and the architectural mechanisms (not just aspirational numbers) used to meet them — the bridge between the NFR matrix's targets and how services are actually built to hit them.

## Relationship to other documents
- **SRD:** Availability/performance expectations implied by the "10–20 minute delivery" product promise are the business input; this document turns that into system-level targets (e.g., order-placement API latency budget).
- **06-quality-attributes/nfr-matrix.md:** That document is the measurable target list; this document is the design approach used to hit those targets.
- **SDD:** Each service's SDD should show how it meets its allocated share of the system-wide targets.
- **Implementation:** Timeout values, retry budgets, and load-shedding logic in code should trace back to a decision made here.

## Sections to be authored
1. Availability targets and how they're allocated across services
2. Latency budgets for key user journeys (order placement, dispatch, delivery tracking)
3. Failure-handling strategy (graceful degradation, what's allowed to fail loudly vs. silently)
4. Disaster recovery pointer (see 05-deployment)
