# Diagram Organization

No diagrams exist yet in this project — this document defines *where they will live and how they will be named* once the architecture documents above are far enough along to draw from, not the diagrams themselves.

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
`source/` currently contains no files. The first diagrams should be drawn only after `02-context/system-context.md` and `03-decomposition/service-decomposition.md` have real content — see the writing order in the top-level `architecture/README.md`.
