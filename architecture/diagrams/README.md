# Diagram Organization

This document defines *where diagrams live and how they are named*. As of 2026-07-30, five structural diagrams exist (`generated-diagrams.md`) alongside five business-flow diagrams (`generated-business-flow-diagrams.md`) — see each for its full set with purpose, explanation, and related-document annotations per diagram, and `source/` for the authoritative Mermaid source files.

## Principles
- **Diagrams are generated from decisions, never the other way around.** A diagram is drawn once the prose document it illustrates (decomposition, context, deployment) has stabilized enough to be worth visualizing. Drawing diagrams before the decomposition is settled produces pictures that lie by the time anyone reads them.
- **Diagrams-as-code, not binary files.** Source lives as text (e.g., Mermaid or PlantUML/C4-PlantUML) under `diagrams/source/`, not as opaque `.drawio`/`.png`-only files with no diffable history. A rendered export can sit alongside the source for convenience, but the source is authoritative.
- **One diagram, one concern.** A diagram that tries to show system context *and* deployment topology *and* data flow at once is a diagram nobody trusts. Prefer several small, single-purpose diagrams over one dense one.

## Framework
Diagrams should follow the C4 model's levels, matching the granularity of the prose document they illustrate:

| C4 Level | Illustrates | Paired document |
|---|---|---|
| Level 1 — System Context | System boundary, external actors/integrations | `02-context/system-context.md` |
| Level 2 — Container | Deployable units and how they communicate | `03-decomposition/service-decomposition.md` |
| Level 3 — Component | Internal structure of one unit | corresponding SDD document (out of `/architecture` scope) |
| (Deployment variant) | Infrastructure topology | `05-deployment/infrastructure-and-release.md` |

## Naming convention
`source/<c4-level>-<topic>.<ext>`, e.g. `source/l1-system-context.mmd`, `source/l2-containers.mmd`, `source/deployment-production-topology.mmd`. Rendered exports (if kept) mirror the same basename under `diagrams/rendered/` with an image extension.

## Referencing
Prose documents link to a diagram by relative path rather than embedding it inline, so the diagram can be updated without hunting through prose for a copy-pasted image. A diagram's own file should carry a one-line header comment naming the document it illustrates and the date it was last verified against that document.

## Status
`source/` contains ten diagrams across two sets:
- **Structural** (system context, module catalog, module communication, capability boundary map, service decomposition): `l1-system-context.mmd`, `l2-containers.mmd`, `l3-module-dependencies.mmd`, `business-capability-map.mmd`, `module-communication.mmd`. Companion document: `generated-diagrams.md`.
- **Business-flow** (checkout, inventory reservation, payment, delivery lifecycle, delivery batching): `checkout-sequence.mmd`, `inventory-reservation-flow.mmd`, `payment-flow.mmd`, `delivery-lifecycle.mmd`, `delivery-batch.mmd`. Companion document: `generated-business-flow-diagrams.md`.

Each companion document provides Purpose, explanation, embedded Mermaid source, rendering notes, related ADRs, and related architecture documents per diagram; `generated-diagrams.md` additionally carries a discrepancy report (Section 0) for any requested element that did not match what the approved documents actually establish. Each `.mmd` file's header comment names the document(s) it illustrates and the date last verified against them.
