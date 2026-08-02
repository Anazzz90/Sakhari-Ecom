# 0029. PostgreSQL transactional outbox for domain events

**Status:** Accepted

## Context

ADR-0011 requires asynchronous side effects to be durable and retryable. Later event architecture correctly described the current backend as a modular monolith, but left unresolved whether in-process events are backed by a durable queue or are purely memory-resident.

## Problem

A purely in-memory dispatcher can lose committed facts if the backend process crashes after a database commit but before all consumers run. That is unacceptable for order, payment, inventory, delivery, audit, notification, and analytics workflows.

## Options Considered

1. Pure in-memory event dispatch.
2. Redis-backed queues for domain events.
3. PostgreSQL transactional outbox, with asynchronous workers dispatching and retrying events.
4. Dedicated message broker from day one.

## Decision

Use a PostgreSQL-backed transactional outbox for all domain events.

When a module commits a business fact that must publish an event, it writes the owned entity changes and an outbox record in the same module-local PostgreSQL transaction. A background dispatcher reads pending outbox records, invokes subscribed consumers, records delivery attempts, retries with backoff, and marks events as processed only after successful handling.

Consumers remain idempotent. Event dispatch may still happen in-process at runtime, but the source of pending work is durable PostgreSQL state, not process memory.

## Rationale

The transactional outbox preserves ADR-0011's durability requirement without introducing a separate message broker before scale requires one. PostgreSQL is already the system of record, already backed up, and already inside the Saudi residency boundary. Redis remains coordination/cache only and does not become authoritative for domain events.

## Consequences

- Every event-producing module needs an outbox write path.
- Event consumers need idempotency keys or processed-event records.
- A dispatcher/background worker becomes part of the backend runtime.
- Horizontal backend scaling is safe because instances coordinate through PostgreSQL, not memory.
- A future message broker can be introduced behind the dispatcher without changing event names or consumers.

## Future Reconsideration Conditions

Reconsider only if outbox throughput or operational characteristics become a measured bottleneck, or when a module is extracted into an independently deployed service.

## Related Documents

- Related architecture principle(s): Principle 4.2, Principle 4.8, Principle 4.10
- Related ADRs: 0003, 0004, 0011, 0021

