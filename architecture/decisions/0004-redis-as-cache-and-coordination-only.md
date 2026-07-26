# 0004. Redis as cache/coordination only

**Decision ID:** ADR-0004
**Status:** Accepted
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
The platform needs fast reads, session handling, rate limiting, and coordination (for example, short-lived locks around inventory reservation, see ADR-0010) beyond what hitting PostgreSQL on every request would comfortably serve. `00-Architecture-Principles.md` Principle 4.3 already states, at the principle level, that Redis may accelerate and coordinate but must never become a second source of truth. ADR-0003 establishes PostgreSQL as the sole system of record this principle protects.

## Problem
What role should an in-memory store play in the system, in a way that provides real performance and coordination benefit without ever becoming a place a business fact can exist only in?

## Options considered
1. **Redis as a full secondary datastore** — hot-path entities (denormalized order/inventory views) written directly to Redis for speed, with PostgreSQL as a slower backing store. Fast, but reintroduces the exact "two authoritative stores" failure mode Principle 4.2 exists to prevent.
2. **No in-memory store at all** — every read and coordination need hits PostgreSQL directly. Simplest to reason about, but leaves no mechanism for cheap short-lived coordination (rate limits, distributed locks, idempotency keys) without overloading the relational database for that purpose.
3. **Redis strictly scoped to caching, sessions, rate limiting, and short-lived coordination**, with anything business-significant always committed to PostgreSQL as the authority.

## Decision
Redis is adopted strictly for acceleration and coordination — caching, session storage, rate limiting, distributed locks, and idempotency keys — and never as an independent store of business fact.

## Rationale
Direct application of Principle 4.3. Option 1 was rejected outright as the "cache as database" failure mode the principle names explicitly: a restart, eviction, or flush would silently lose data nobody thought to protect. Option 2 was rejected because it removes a genuinely useful, low-risk tool (fast reads, coordination primitives) for no correctness benefit, working against the delivery-appropriate-latency design goal. Option 3 gets the performance and coordination benefit while keeping ADR-0003's single-source-of-truth guarantee intact.

## Consequences
- Redis can be flushed, resized, or restarted at any time with zero business-data risk, and can be treated as fully disposable infrastructure.
- Some read paths pay the cost of a PostgreSQL round-trip, or a cache-then-verify step, that treating Redis as authoritative would avoid — an accepted tradeoff.
- Any new proposed use of Redis must be reviewed against "would losing this on a flush lose a business fact?" before being approved; if yes, it belongs in PostgreSQL instead.
- Distributed locks used for coordination (e.g., inventory reservation, see ADR-0010) are momentary aids only — the state they help produce is not considered true until committed in PostgreSQL.

## Future reconsideration conditions
None anticipated for the "never authoritative" boundary itself — Principles 4.2 and 4.3 make this a near-permanent rule. The specific scope of what is cached or coordinated through Redis is expected to expand as real load patterns emerge (ADS Section 15), without requiring reconsideration of this ADR's core decision.

## Related
- SRD section(s): Section 2.1 (Platform Architecture Overview — Redis for coordination/cache use cases)
- Related DDD entity/data area(s): n/a — Redis holds no entity of record by design
- Related architecture principle(s): 4.2, 4.3
- Related ADRs: 0003, 0010
