# 0009. Module ownership and no cross-module repository access

**Decision ID:** ADR-0009
**Status:** Accepted
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
ADR-0002 adopts a modular monolith: one deployable backend with no network boundary between its internal modules. `00-Architecture-Principles.md` Principle 4.5 already states, at the principle level, that each module is the only code permitted to read or write its own tables, and Section 10 names cross-module repository access and direct database access between modules as anti-patterns.

## Problem
In a single deployable backend with no network boundary between modules, what stops a capability boundary (ADR-0002) from being bypassed by a direct query into another module's tables — the exact risk a modular monolith carries that microservices avoid for free through network isolation?

## Options considered
1. **No enforced rule** — rely on developer and AI-assistant discipline alone to respect module boundaries. Cheapest to set up, weakest to hold under time pressure or across many sessions with no memory.
2. **Physically separate databases per module, even inside one deployable.** Adds real operational overhead (multiple connections, multiple backup targets) without full isolation, and conflicts with ADR-0003's single PostgreSQL system of record.
3. **One database; each module exclusively owns and exposes its own tables through its own service-layer interface**, with every other module required to go through that interface — never a direct repository, query, or shared connection into another module's tables.

## Decision
Each backend module exclusively owns its own tables. All cross-module access happens through the owning module's defined interface — never through a direct repository, query, or shared database connection into another module's tables.

## Rationale
This is the specific enforcement mechanism that makes Principle 4.5 real rather than aspirational, and the direct countermeasure to the cross-module-repository-access and direct-database-access anti-patterns named in Constitution Section 10. Option 1 was rejected because Constitution Principle 4.12 treats an unrecorded shortcut as a violation, not a smaller version of the rule — a rule with no enforcement path invites exactly the shortcuts that principle exists to prevent. Option 2 was rejected as paying real operational cost for isolation ADR-0003 already forecloses, since the system of record must remain one PostgreSQL instance. Option 3 keeps ADR-0002's modular monolith honestly reversible into independently deployed services later (Constitution Section 13, ADS Section 15) — a boundary enforced only by convention is not one a future extraction could safely rely on.

## Consequences
- A module's internal schema can change without a system-wide audit of every caller; responsibility for a table's correctness has exactly one home.
- The path to future service extraction (ADR-0002) stays genuinely open rather than theoretical.
- Cross-cutting needs — reporting, the Admin/Ops Dashboard's aggregate views — require deliberate design (an explicit read model or an owning-module API call) rather than a convenient join, per Principle 4.5's own stated exception for reporting.
- Code review, and any AI-assistant-generated code, must reject a change that queries another module's tables directly, per Principle 4.12.

## Future reconsideration conditions
None anticipated for the rule itself. The mechanism used to enforce it (code review discipline now; potentially automated boundary-checking tooling later, once the codebase is large enough to justify it) is expected to evolve as the codebase grows, without changing the underlying rule.

## Related
- SRD section(s): n/a directly — a backend structuring rule, not a product requirement
- Related DDD entity/data area(s): all entity/data-ownership areas — this ADR is the mechanism that makes `03-decomposition/data-ownership-map.md`'s per-entity ownership assignments enforceable in code
- Related architecture principle(s): 4.4, 4.5; Constitution Section 10 (anti-patterns), Section 4.12
- Related ADRs: 0002, 0003
