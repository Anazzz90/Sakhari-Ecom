# 0003. PostgreSQL as single source of truth

**Decision ID:** ADR-0003
**Status:** Accepted
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
The platform moves real money and real inventory and must never let two stores disagree about what actually happened. `00-Architecture-Principles.md` Principle 4.2 already states, at the principle level, that durable business state lives authoritatively in PostgreSQL and nowhere else. The SRD's own version history records that an earlier direction (a managed backend-as-a-service platform combining auth and data) was superseded specifically because it could not satisfy Saudi data-residency requirements, in favor of an in-house backend on a managed PostgreSQL instance.

## Problem
Which datastore holds the authoritative record of durable business state — orders, payments, inventory, accounts, entity lifecycle — and how is the risk of two stores silently disagreeing about that state avoided?

## Options considered
1. **A managed backend-as-a-service platform** (combined auth + data store) — rejected at the requirements level already, over Saudi data-residency compliance.
2. **Polyglot persistence** — multiple specialized datastores, each authoritative for its own domain (e.g., a document store for catalog, a graph store for relationships). Offers per-workload optimization at the cost of multiple operational surfaces and reconciliation risk between stores.
3. **A single PostgreSQL instance, provisioned via a managed relational database service, as the sole system of record**, with every other store in the system (Redis, object storage) explicitly non-authoritative.

## Decision
PostgreSQL, provisioned via a managed relational database service, is the single, exclusive source of truth for all durable business state in the system.

## Rationale
Direct application of Principle 4.2. Polyglot persistence was rejected because it multiplies the operational surface a solo developer must maintain (Constitution Section 6, Section 13's "complexity is earned, not speculative") without an evidenced workload PostgreSQL cannot serve at launch scale. A managed service (over a self-managed database server) satisfies the operational-simplicity design goal by removing patching, backup, and failover from the developer's manual responsibility.

## Consequences
- One place to reason about consistency, one place to back up and restore, one place transactions (see ADR-0010) can protect.
- Every durable write must pass through PostgreSQL even where a specialized store might theoretically be faster, bounding throughput below what a distributed design could achieve — an accepted tradeoff (Constitution Section 6).
- No module or client may treat Redis (ADR-0004) or object storage (ADR-0016) as authoritative for a business fact.
- All future entity design in the DDD, and all data-ownership assignments in `03-decomposition/data-ownership-map.md`, are built on this datastore as their foundation.

## Future reconsideration conditions
A specific, evidenced workload (e.g., large-scale geospatial rider tracking, full-text search at a scale PostgreSQL's own extensions can no longer serve) proves genuinely unsuited to relational modeling even with in-database extensions. Such a case would introduce a supplementary, explicitly non-authoritative specialized store via a new ADR, without altering PostgreSQL's status as system of record.

## Related
- SRD section(s): Backend/database finalization (auth and database corrected to an in-house PostgreSQL-based approach for residency compliance)
- Related DDD entity/data area(s): all entity/data areas — the foundation every DDD entity-ownership assignment depends on
- Related architecture principle(s): 4.2, 4.9, 4.10
- Related ADRs: 0001, 0004, 0010
