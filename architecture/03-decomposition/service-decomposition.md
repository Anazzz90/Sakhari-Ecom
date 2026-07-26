# Service Decomposition

**Stability:** Evolving — expected to change as modules mature, background workers are introduced, or a future scale threshold justifies extracting a module into a separately deployed service.
**Status:** Skeleton — pending authoring.

## Purpose
Defines the actual backend runtime shape: the modular monolith, its internal modules, background workers/queues, and the responsibility of each. Where the SRD describes client applications, this document describes what sits behind them.

**Note:** `module-catalog.md` (in this folder) already documents the sixteen backend modules' responsibilities, dependencies, and public interfaces in full. When this document is authored, its job is the runtime-shape view on top of that catalog — background workers/queues and splitting criteria — not a re-statement of per-module responsibility.

## Relationship to other documents
- **SRD:** Must respect the SRD's stated backend shape and constraints (single-developer maintainability, launch-scale sizing) — this document does not introduce services the SRD's constraints wouldn't support.
- **DDD / capability-boundary-map.md:** Each decomposed unit must trace to one or more SRD capabilities and DDD entity-ownership areas via the capability boundary map; decomposition never invents business concepts.
- **SDD:** Each module/runtime unit named here gets its own SDD document describing internal module structure, API contracts, and persistence.
- **Implementation:** This is the canonical list of "what should exist in the backend runtime" — new modules, workers, or deployed services in code that aren't listed here are a signal the doc is stale or the addition wasn't reviewed.

## Sections to be authored
1. Unit list (name, responsibility, owning capability area, dependencies)
2. Synchronous vs. asynchronous responsibilities per unit
3. Splitting criteria — what would trigger decomposing a unit further

