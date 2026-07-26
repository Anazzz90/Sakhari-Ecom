# 0013. ULIDs for entity identifiers

**Decision ID:** ADR-0013
**Status:** Superseded by [0018](0018-uuid-primary-keys.md)
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
Every entity in the system (Order, Payment, Product, Store, User, and others defined in the DDD) requires a primary identifier. The platform is multi-store, event-driven (ADR-0011), and exposes identifiers across four clients and external providers (payment gateway, SMS/OTP provider).

## Problem
What identifier scheme should entities use, given the need for identifiers that can be generated before or independent of a database round-trip, that remain sortable for debugging and pagination, and that do not leak sequential business volume (such as total order count) to anyone who happens to see one?

## Options considered
1. **Auto-incrementing integer primary keys.** Simple and compact, but sequential integers leak business volume (a competitor or customer could infer total order counts from a visible order number) and complicate any future extraction of a module into its own service (ADR-0002/0009), since ID generation would need central coordination.
2. **Random UUIDv4.** Avoids the leakage problem and requires no central coordination, but is not sortable, which hurts debugging, log correlation, and the readability of audit trails (Principle 4.10) at launch scale.
3. **ULIDs (Universally Unique Lexicographically Sortable Identifiers)** — timestamp-prefixed, sortable, collision-resistant identifiers, generatable by any module or client without a coordinating authority.

## Decision
Use ULIDs as the primary identifier for business entities across the system.

## Rationale
Option 1 was rejected for the business-volume leakage risk and for working against ADR-0002's future-extractability goal. Option 2 was rejected because it forfeits sortability for no benefit ULIDs don't already provide. ULIDs combine non-sequential, non-guessable identifiers with lexicographic sortability by creation time, and any module or client can generate its own ULIDs without a central sequence — consistent with the modular monolith's designed reversibility (Constitution Section 13) and with Principle 4.10's need for a readable audit trail.

## Consequences
- Identifiers sort naturally by creation time in logs, audit trails, and database indexes, aiding debugging and correlation.
- Any module can generate its own identifiers without a central coordinating sequence, which matters if a module is later extracted into its own service (ADR-0002).
- ULIDs are longer than integer keys, with a marginal storage and index-size cost, accepted as a minor tradeoff.
- All new entity tables use a ULID primary key convention rather than mixing identifier schemes across the schema.

## Future reconsideration conditions
None anticipated. This is a low-level, inexpensive-to-maintain convention; the one-way-door risk lies in retroactively changing established identifiers once real data exists, which is precisely why the convention is adopted uniformly from the outset rather than deferred.

## Related
- SRD section(s): n/a directly — an implementation-adjacent data convention, not a business requirement
- Related DDD entity/data area(s): all entity areas — the identifier convention every DDD entity's primary key follows
- Related architecture principle(s): 4.1, 4.10
- Related ADRs: 0003, 0009
