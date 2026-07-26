# Non-Functional Requirements Matrix

**Stability:** Evolving — revisited every time scale, market, or SLA commitments change.
**Status:** Skeleton — pending authoring.

## Purpose
The single, measurable source of truth for non-functional targets (availability, latency, throughput, security posture, compliance deadlines) that every cross-cutting and module-level design is judged against. Where `reliability-and-performance.md` explains the *approach*, this document is the *scorecard*.

## Relationship to other documents
- **SRD:** Every row here must trace back to an SRD statement (explicit or reasonably inferred from the launch-scale/product-promise constraints) — this document does not invent NFRs the SRD gives no basis for.
- **DDD:** Entity volume, ownership, relationship, and retention expectations inform throughput and retention-related targets.
- **04-cross-cutting/*.md:** Each cross-cutting document should reference the specific NFR row(s) it exists to satisfy.
- **SDD / Implementation:** SDD documents allocate a share of each system-wide target to their service; test/monitoring thresholds in implementation should be traceable to a row here.

## Sections to be authored
1. NFR table (category, target, measurement method, SRD source reference)
2. Traceability notes (which architecture document is responsible for each NFR)
