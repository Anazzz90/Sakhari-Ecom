# Data Ownership Map

**Stability:** Stable once written — changing an entity's owning module is a high-blast-radius change and should be ADR-backed.
**Status:** Skeleton — pending authoring.

## Purpose
States, for every major entity in the platform, which module is the source of truth, which read models/caches are allowed, and how other consumers are expected to access it (module interface, API call, event, or approved reporting read model).

**Note:** `module-catalog.md` (in this folder) already states per-module Data Ownership and includes a Cross-Module Summary Table (its Section 5). ADR-0020 resolves the earlier unassigned Support Ticket and Support Ticket Comment ownership gap by assigning them to the Support module. When this document is authored, it should expand the catalog ownership table with read-access rules and cross-module consistency expectations rather than re-deriving entity ownership independently.

## Relationship to other documents
- **DDD:** Entities, relationships, retention, and integrity expectations are defined there; this document does not redefine entity meaning, only assigns module ownership and access rules.
- **SRD:** Cross-checked against the SRD's data requirements and retention sections (v2.5 scope) to ensure ownership assignment satisfies stated integrity and retention expectations.
- **SDD:** Each module's SDD persistence section must be consistent with what this document says that module owns.
- **Implementation:** A migration or schema change owned by a module not listed as the owner here is a red flag.

## Sections to be authored
1. Entity → owning module table
2. Read-access rules for non-owning consumers (API-only vs. replicated read model)
3. Cross-service consistency expectations (strong vs. eventual, and where)
