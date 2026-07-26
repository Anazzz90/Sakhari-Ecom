# Capability Boundary Map

**Stability:** Stable once written — this is the load-bearing translation layer between requirements, data ownership, and system structure; changes here ripple everywhere and should be rare and ADR-backed.
**Status:** Skeleton — pending authoring.

## Purpose
Maps each major business capability from the SRD, and each major entity ownership area from the DDD, to a concrete architectural unit (module, runtime unit, or app). This is the single document that answers "which module/app owns this responsibility?"

**Note:** `module-catalog.md` (in this folder) is now the authoritative, per-module source of truth for the fifteen backend modules, their ownership, and their boundaries — including seven open reconciliation points (its Section 3) between that module list and the DDD's entity-ownership breakdown. When this document is authored, it should summarize and cross-reference `module-catalog.md` rather than re-derive the mapping independently, and should be the place those seven open points get formally resolved.

## Relationship to other documents
- **DDD:** Direct input for entity ownership, lifecycle, and data-integrity expectations. This document does not redefine entities; it assigns the system module responsible for them.
- **SRD:** Cross-checked against the application structure table (Customer Mobile/Web, Worker App, Admin Dashboard) to confirm every context has a client surface where relevant.
- **SDD:** Each mapped module/capability becomes the subject of one or more SDD documents. One SDD scope should not span unrelated capabilities without a documented reason.
- **Implementation:** Repository/module boundaries in code should mirror this map. A code change that crosses a module/capability boundary without an update here is an architectural drift signal.

## Sections to be authored
1. Capability-to-module mapping table
2. Capability relationships (allowed dependencies, shared concerns, anti-corruption boundaries, etc., described narratively)
3. Ownership notes (which module is authoritative for which entity, cross-referenced with `data-ownership-map.md`)
