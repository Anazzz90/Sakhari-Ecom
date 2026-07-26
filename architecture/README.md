# Sakhari Ecom â€” Architecture Documentation

This folder is the **architecture layer** of the Sakhari Ecom documentation set. It sits between the **Database Design Document (DDD)** and **implementation**, and above module-level **Software Design Documents (SDD)**:

```
SRD  â†’  DDD  â†’  /architecture  â†’  SDD  â†’  Implementation
(what & why)  (data model)     (system shape)  (module detail)       (code)
```

- **SRD** (`SRD_Sakhari_Ecom_v2.6_Saudi.md`) says what the product must do and under what business/regulatory constraints.
- **DDD** describes the database design: entities, ownership, relationships, retention, integrity rules, and logical data architecture.
- **`/architecture` (this folder)** decides how the domain becomes a running system: service boundaries, data ownership, cross-cutting rules (security, data, integration, observability, reliability), deployment topology, and the decisions behind all of it.
- **SDD** (module-level, not yet created) takes each backend module/capability defined in `03-decomposition/service-decomposition.md` and specifies its internal design: responsibilities, schemas, endpoints, events, and tests.
- **Implementation** is code that satisfies its SDD, which in turn must be consistent with this folder.

If code and this folder disagree, one of them is wrong â€” that disagreement should be resolved explicitly (update the doc via an ADR, or fix the code), never left silent.

Within `/architecture` itself there is a further internal hierarchy, from foundational to detailed:

```
00-Architecture-Principles.md  (the Constitution â€” durable values)
        â†“
01-Architecture-Design-Specification.md  (the ADS â€” the master description of the architecture)
        â†“
02-context/ â€¦ 06-quality-attributes/  (topic-specific detail, each owned by the ADS's summary of it)
        â†“
decisions/  (the record of individual significant choices, referenced from anywhere above)
```

---

## 1. Folder structure

```
architecture/
â”œâ”€â”€ README.md                              you are here â€” index & system explanation
â”œâ”€â”€ CHANGELOG.md                           structural changes to this documentation set
â”œâ”€â”€ 00-Architecture-Principles.md          the Constitution â€” binding engineering philosophy & core principles
â”œâ”€â”€ 01-Architecture-Design-Specification.md  the ADS â€” master description of the architecture; single source of truth
â”œâ”€â”€ 02-context/
â”‚   â”œâ”€â”€ system-context.md                  system boundary: actors, external integrations
â”‚   â””â”€â”€ glossary.md                        architecture-specific vocabulary (links to SRD/DDD terminology)
â”œâ”€â”€ 03-decomposition/
â”‚   â”œâ”€â”€ module-catalog.md                  the 15 backend modules in full: responsibilities, boundaries, ownership, interfaces, events
â”‚   â”œâ”€â”€ module-communication.md            allowed/forbidden communication, dependency graph, transaction boundaries
â”‚   â”œâ”€â”€ capability-boundary-map.md             business capability â†’ owning module/app
â”‚   â”œâ”€â”€ service-decomposition.md           deployable units and their responsibilities
â”‚   â””â”€â”€ data-ownership-map.md              entity â†’ source-of-truth module, access rules
â”œâ”€â”€ 04-cross-cutting/
â”‚   â”œâ”€â”€ security-and-compliance.md         authN/authZ, data protection, SAMA/PDPL
â”‚   â”œâ”€â”€ data-architecture.md               persistence, caching, consistency, retention
â”‚   â”œâ”€â”€ integration-and-messaging.md       event architecture: naming, publish/consume, retry, idempotency, versioning, future event bus
â”‚   â”œâ”€â”€ observability-and-operations.md    logging, metrics, alerting, incident response
â”‚   â”œâ”€â”€ reliability-and-performance.md     availability/latency targets and how they're met
â”‚   â””â”€â”€ technology-decisions.md            per-technology purpose, alternatives, tradeoffs, replacement strategy
â”œâ”€â”€ 05-deployment/
â”‚   â”œâ”€â”€ environment-strategy.md            dev/staging/prod and what varies between them
â”‚   â””â”€â”€ infrastructure-and-release.md      topology, CI/CD, release & rollback, DR
â”œâ”€â”€ 06-quality-attributes/
â”‚   â””â”€â”€ nfr-matrix.md                      measurable NFR targets, traced to the SRD
â”œâ”€â”€ decisions/                             Architecture Decision Records (ADRs)
â”‚   â”œâ”€â”€ README.md                          ADR process, lifecycle, index
â”‚   â”œâ”€â”€ template.md
â”‚   â””â”€â”€ 0001-record-architecture-decisions.md
â””â”€â”€ diagrams/
    â”œâ”€â”€ README.md                          diagram conventions (C4 levels, naming, diagrams-as-code)
    â””â”€â”€ source/                            diagram source files (empty until decomposition stabilizes)
```

The two root-level documents (`00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`) are the only ones written and named as capital-letter titles â€” they are the foundational pair every other document defers to. Detail documents under `02-context/` through `06-quality-attributes/` remain **skeletons**: a stated purpose, relationship to SRD/DDD/SDD/implementation, stability classification, and a list of sections still to be authored. The ADS summarizes what each of them will eventually contain in full; filling them in is the next phase of work, in the order set out in Section 4 below.

---

## 2. Purpose of every document

| Document | Purpose |
|---|---|
| `00-Architecture-Principles.md` | The Constitution â€” engineering philosophy, the twelve core architectural principles, design goals/tradeoffs/constraints, coding and AI-development philosophy, anti-patterns, decision-making guidelines, and the amendment process. Binding on every other document in this folder. |
| `01-Architecture-Design-Specification.md` | The ADS â€” the master, single-source-of-truth description of what the architecture actually *is*: business context, system vision, platform overview, system context, high-level architecture, key decisions, technology choices, module overview, and the data/security/scalability/reliability philosophies. Realizes the Constitution's principles into an actual system shape without re-explaining them. |
| `02-context/system-context.md` | The system as a black box, in full â€” who and what it talks to, where the boundary sits. The ADS's System Context section summarizes this; this document is the authoritative detail. |
| `02-context/glossary.md` | Architecture-specific terms only; explicitly does not redefine SRD business terminology or DDD data terminology. |
| `03-decomposition/module-catalog.md` | The 15 backend modules (Auth, User, Store, Catalog, Search, Inventory, Cart, Order, Payment, Promotion, Delivery, Notification, Analytics, Audit, Settings) documented in full — responsibilities, boundaries, ownership, public interfaces, dependencies, forbidden dependencies, published/consumed events, data ownership, future expansion. The primary per-module source of truth; includes an Open Decisions section reconciling this module list against the DDD's entity-ownership breakdown. |
| `03-decomposition/module-communication.md` | How the 15 modules are allowed and forbidden to reach each other — REST vs. in-process interface calls vs. domain events, the full dependency graph, transaction-boundary and orchestration-with-compensation rules for cross-module operations, and circular-dependency prevention. |
| `03-decomposition/capability-boundary-map.md` | Answers "which module/app owns this business capability?" â€” the translation of SRD capabilities and DDD entities into system units. |
| `03-decomposition/service-decomposition.md` | The backend modules/runtime units and what each is responsible for inside the modular monolith. |
| `03-decomposition/data-ownership-map.md` | Per-entity source of truth and the rules for reading it from elsewhere. |
| `04-cross-cutting/security-and-compliance.md` | Security architecture: identity, authentication (OTP), JWT, refresh-token rotation, RBAC/permission model, API security, audit's security role, secrets, and SAMA/PDPL compliance mapping. |
| `04-cross-cutting/data-architecture.md` | System-wide persistence/caching/consistency/retention rules: PostgreSQL and Redis philosophy, data ownership, transactions, the inventory model and reservation lifecycle, the inventory ledger, soft deletes, ULIDs, UTC, money representation, concurrency strategy, and data-level recovery. |
| `04-cross-cutting/integration-and-messaging.md` | Event Architecture: domain event naming/publish/consume rules, background processing, the full notification flow, retry strategy, idempotency, event versioning, and the future event-bus strategy for module extraction. Synchronous API conventions and external-integration retry patterns remain open (see its own Open Decisions). |
| `04-cross-cutting/observability-and-operations.md` | How the system is watched and operated by a single developer. |
| `04-cross-cutting/reliability-and-performance.md` | Performance, Scalability & Reliability: performance philosophy, availability/failure-handling, caching, search, read/write optimization, queue usage, horizontal and database scaling, future microservice extraction, and capacity planning principles. |
| `04-cross-cutting/technology-decisions.md` | Every technology in use (React Native, Next.js, React/Vite, NestJS, PostgreSQL, Redis, S3, JWT, OTP, Cloud Infrastructure, CDN), each with purpose, why chosen, alternatives considered, benefits, drawbacks, and future replacement strategy. Expands ADS Section 9 into full detail; complements rather than duplicates the technologies that already have a dedicated ADR. |
| `05-deployment/environment-strategy.md` | The environment list (Development/Staging/Production), what varies between them, and the promotion path. |
| `05-deployment/infrastructure-and-release.md` | Deployment Architecture: infrastructure components, networking, storage, cache, logging, monitoring, backup, disaster recovery, CI/CD, and a scaling-strategy overview (deep dive is `11-Performance-Scalability.md`, not yet authored). |
| `06-quality-attributes/nfr-matrix.md` | The measurable scorecard every cross-cutting and module-level design is checked against. |
| `decisions/*` | The record of *why* â€” one immutable file per significant, hard-to-reverse choice. |
| `diagrams/*` | Where and how diagrams will be organized once there's enough stable content to draw. |

Each document's own file repeats this purpose alongside its specific relationships â€” this table is the map, not a replacement for reading the document.

---

## 3. How each document relates to SRD, DDD, SDD, and Implementation

The relationship is consistent across the whole folder and worth stating once instead of per-file:

- **To the SRD:** every architecture document must be traceable to something the SRD actually says (a constraint, an actor, a compliance obligation, an application). Architecture documents translate business requirements into system rules; they never introduce business requirements the SRD doesn't support. The ADS's Business Context section makes this traceability explicit at the top level; `06-quality-attributes/nfr-matrix.md` does it for measurable targets specifically.
- **To the DDD:** `03-decomposition/*` is the direct bridge between the logical database design and the running system â€” entities, relationships, ownership, and lifecycle expectations from the DDD become module ownership and data-access rules here. No architecture document redefines entity meaning; they only decide *where* data responsibilities live and *how* they're technically realized.
- **To the SDD (future, module-level):** this folder is upstream of SDD. Each module/capability in `service-decomposition.md` (summarized in the ADS's Module Overview) gets one or more SDD documents; each cross-cutting document and each ADS philosophy section sets rules that every SDD document must individually satisfy and cite. Architecture answers "what exists and what rules apply everywhere"; SDD answers "how this one module is built inside."
- **To implementation:** implementation should never contradict an `Accepted` ADR, a cross-cutting rule, or a philosophy stated in the ADS without that change first being reflected back here (new ADR, updated document). Architecture documents are the thing code is checked against â€” not documentation of what the code happened to do.

---

## 4. Order of authorship

Write in this order â€” each phase depends on the one before it:

1. **The Constitution** (`00-Architecture-Principles.md`) â€” establishes scope, engineering philosophy, and the criteria everything else is judged by. Derived from the SRD's constraints section. Written once, amended rarely and only through the explicit process in its own Section 13.
2. **The ADS** (`01-Architecture-Design-Specification.md`) â€” the master description of the architecture, realizing the Constitution's principles into an actual system shape. Written second because every detail document that follows exists to elaborate a section the ADS already summarizes.
3. **Context & glossary** (`02-context/`) â€” establishes the system boundary before anything inside it is designed in detail; expands the ADS's System Context section.
4. **Decomposition** (`03-decomposition/`) â€” capability/module map first, then runtime decomposition, then data ownership. This is the highest-leverage, highest-blast-radius phase; take the most care here. Expands the ADS's Module Overview and High-Level Architecture sections.
5. **Cross-cutting concerns** (`04-cross-cutting/`) â€” informed by the decomposition (you can't write a security model for services that don't exist yet) and by the SRD's NFR-adjacent language. Expands the ADS's Data, Security, and Reliability Philosophy sections.
6. **Deployment** (`05-deployment/`) â€” depends on the decomposition (what needs deploying) and cross-cutting rules (what infrastructure needs to enforce). Expands the ADS's Scalability and Reliability Philosophy sections.
7. **Quality attributes** (`06-quality-attributes/nfr-matrix.md`) â€” written last among the numbered folders because it's a synthesis: it cites specific rows from cross-cutting and deployment documents as the mechanism for each target.
8. **ADRs** â€” not a phase, a continuous practice. The first real ADRs should retroactively capture decisions the SRD and ADS already show were made (e.g., "why NestJS + RDS over Supabase," "why two customer clients, one worker app," "why a modular monolith over microservices at launch") so that reasoning is preserved in addressable, linkable form rather than buried in prose. New ADRs are added the moment a future significant decision is made, not batched.
9. **Diagrams** â€” deliberately last. Diagrams drawn before decomposition and cross-cutting concerns stabilize become misleading almost immediately; see `diagrams/README.md`.

As each numbered detail folder is authored in full, the corresponding ADS section is revisited and tightened into a true summary (with the detail document referenced, not duplicated) rather than left as the fuller placeholder text it starts as.

---

## 5. Stability classification

Not every document changes at the same rate. Treating them uniformly either makes stable documents feel falsely provisional or lets volatile documents ossify past their usefulness.

| Stable â€” change rarely, ADR-backed when it happens | Evolving â€” expected to change as the system grows |
|---|---|
| `00-Architecture-Principles.md` (the Constitution â€” see its own Section 13 for the amendment process) | `04-cross-cutting/data-architecture.md` |
| `01-Architecture-Design-Specification.md` (high-level shape; kept in sync as detail is authored â€” see Section 4 above) | `04-cross-cutting/integration-and-messaging.md` |
| `03-decomposition/capability-boundary-map.md` | `04-cross-cutting/observability-and-operations.md` |
| `03-decomposition/data-ownership-map.md` | `04-cross-cutting/reliability-and-performance.md` |
| `04-cross-cutting/security-and-compliance.md` (baseline; detail evolves) | `03-decomposition/service-decomposition.md` |
| `05-deployment/environment-strategy.md` | `05-deployment/infrastructure-and-release.md` |
| Accepted ADRs (individually immutable) | `06-quality-attributes/nfr-matrix.md` |
| | `02-context/system-context.md` / `glossary.md` (grow, don't restructure) |

The ADS sits in an intentional middle position: its overall shape and section list are stable, but its content is expected to be *tightened* (not contradicted) as each detail document below it reaches full authorship â€” a summary getting more precise is not the same kind of change as a principle being overturned, and does not require an ADR unless the tightening reveals the original summary was substantively wrong.

A document moving from "evolving" to "requires an ADR to change" is itself a signal the system has matured â€” revisit this table periodically rather than treating it as permanent.

---

## 6. Naming conventions

- **Files:** `kebab-case.md`, descriptive nouns, no dates or version numbers in filenames (git history is the version record) â€” except the two root-level `Title-Case` documents (the Constitution and the ADS), whose capitalization marks them as the foundational pair every other document is subordinate to.
- **Folders:** two-digit numeric prefix (`00-`, `01-`, â€¦) enforcing reading order within `/architecture`; the prefix is about sequence, not importance.
- **ADRs:** `NNNN-short-kebab-title.md`, four-digit zero-padded, strictly sequential, never renumbered or reused â€” see `decisions/README.md`.
- **Diagrams:** `<c4-level>-<topic>.<ext>` under `diagrams/source/` â€” see `diagrams/README.md`.
- **Headings inside documents:** every detail document opens with `Purpose`, then `Relationship to other documents`, so any document can be understood in isolation without reading this README first. The Constitution and the ADS instead open with a document-control table (version, status, authority) appropriate to their higher standing.
- **No content in filenames or paths that would need renaming on a decision reversal** â€” e.g. `data-architecture.md`, not `postgres-architecture.md`, so a future datastore change doesn't force a rename (and broken links) on top of the ADR that records it.

---

## 7. Diagram organization

Full convention lives in `diagrams/README.md`; summary:

- Diagrams are **diagrams-as-code** (Mermaid / PlantUML / C4-PlantUML), stored as text under `diagrams/source/`, not opaque binary-only files â€” so they diff and review like any other document.
- Organized by **C4 level** (System Context / Container / Component / Deployment), each level paired with the prose document it illustrates.
- **One diagram, one concern** â€” no diagram tries to show decomposition and deployment and data flow simultaneously.
- Diagrams are drawn **after** the document they illustrate has stabilized, not before â€” see Section 4's ordering. No diagrams exist yet in this project; that's correct for the current phase, and it is why neither the Constitution nor the ADS contains one.

---

## 8. ADR management

Full process lives in `decisions/README.md`; summary:

- One immutable file per significant, hard-to-reverse decision â€” not a running log, not inline notes in other documents.
- Lifecycle: `Proposed â†’ Accepted â†’ Superseded by NNNN` (or `Deprecated`). Accepted ADRs are never edited to change the decision; a changed decision is always a new ADR.
- Recorded the moment a qualifying decision is made â€” continuous practice, not a phase to batch at the end.
- Indexed in `decisions/README.md`, which is the fastest way (for a human or an AI assistant) to check whether a given structural choice is already settled before proposing it again.
- The ADS's "Architectural Decisions Summary" section names the significant decisions already implicit in the finalized SRD/architecture and flags which of them still need to be formally backfilled as ADRs â€” that summary is a work list, not a substitute for the records themselves.

---

## 9. How AI coding assistants should use these documents

This project is built by a solo developer working with AI coding assistants across many sessions with no persistent memory of past conversations. This folder is what replaces that memory. When an assistant is about to generate code, or propose a structural change, it should:

1. **Check `decisions/README.md` first** for any `Accepted` ADR that already settles the question. An accepted ADR is binding; contradicting it silently is a bug, not a judgment call.
2. **Check `00-Architecture-Principles.md`** before introducing anything structural (new dependency, new service, new datastore, new external integration) â€” the Constitution's twelve core principles (Section 4) are the fastest filter for rejecting an off-architecture suggestion before it's written, and Section 12 gives the precedence order when principles appear to conflict.
3. **Check `01-Architecture-Design-Specification.md`** for the actual shape of the system â€” platform overview, module boundaries, technology decisions â€” before assuming or inventing structure the ADS already settles.
4. **Check the relevant `03-decomposition/` document** before writing code that crosses a module/capability boundary â€” code should not casually reach into another module's owned data (see `data-ownership-map.md`) or blur a boundary defined in `capability-boundary-map.md`.
5. **Check the relevant `04-cross-cutting/` document** before implementing anything touching auth, persistence, external integration, logging, or resilience â€” these are the system-wide rules a single service's SDD or code must not reinvent or contradict.
6. **Never edit an Accepted ADR's Decision section, and never restructure `/architecture` itself, without the user's explicit direction** â€” this folder is a governance artifact; an assistant should propose a new ADR or flag a conflict rather than quietly resolving it by rewriting history.
7. **When a document here is still a skeleton** (as all detail documents under `02-context/` through `06-quality-attributes/` are as of this writing), treat its "Sections to be authored" list as the scope of what still needs deciding, and prefer asking or proposing an ADR over silently inventing the missing architecture.
8. **Do not treat this folder as optional context.** For any task that touches system structure â€” as opposed to a local bug fix or a UI tweak fully inside one already-defined unit â€” these documents are the constraints layer the SRD and DDD alone cannot provide.

---

## Current status

`00-Architecture-Principles.md` (the Constitution), `01-Architecture-Design-Specification.md` (the ADS), `02-context/system-context.md`, `03-decomposition/module-catalog.md`, `03-decomposition/module-communication.md`, `04-cross-cutting/data-architecture.md`, `04-cross-cutting/security-and-compliance.md`, `04-cross-cutting/integration-and-messaging.md`, `04-cross-cutting/technology-decisions.md`, `04-cross-cutting/reliability-and-performance.md`, `05-deployment/environment-strategy.md`, `05-deployment/infrastructure-and-release.md`, and the full `decisions/` set (0001–0019) are fully authored. Every remaining detail document remains a skeleton: purpose, cross-document relationships, and stability classification are defined; full architectural content is not yet authored. Each subsequent document must be consistent with the Constitution's principles and the ADS's system description, or must justify a deviation through the ADR process. Several authored documents (module-catalog.md, module-communication.md, data-architecture.md) carry their own Open Decisions sections — see each document directly rather than assuming full closure. See `CHANGELOG.md` for the history of this folder's structure, and Section 4 above for what gets authored next.

