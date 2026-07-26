# 0011. Event-driven asynchronous side effects

**Decision ID:** ADR-0011
**Status:** Accepted
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
`00-Architecture-Principles.md` Principle 4.8 already states, at the principle level, that downstream reactions to a business event travel as events the originating action does not wait on. A single order placement (ADR-0010) fans out to concerns — customer notification, dispatch assignment, analytics — none of which the customer needs to wait on for their checkout response.

## Problem
How should the system trigger downstream reactions to a completed business operation without making the customer-facing request as slow and fragile as its slowest downstream dependency?

## Options considered
1. **Synchronous chained calls** — the order-placement request itself calls the notification system, then the dispatch system, waiting on each in turn. Simple to trace, but makes checkout latency and reliability hostage to every downstream system's health.
2. **Fire-and-forget calls with no delivery guarantee.** Keeps the originating request fast, but a dropped notification or missed dispatch trigger becomes a silent failure — directly contrary to Principle 4.10 and the Section 10 anti-pattern "silent failure of consequential actions."
3. **Asynchronous, durable events** raised as a byproduct of the originating transaction, consumed independently by each downstream concern, with retry and idempotency handling on the consumer side.

## Decision
Side effects of a completed business operation are raised as asynchronous events after the originating transaction commits, and consumed independently by each downstream concern (notifications, dispatch assignment, analytics), with retry and idempotency handling on the consumer side.

## Rationale
Direct application of Principle 4.8. Option 1 was rejected because it makes the customer-facing path only as fast and as reliable as its weakest downstream link, contrary to the delivery-appropriate-latency design goal. Option 2 was rejected because a silently dropped event is exactly the consequential-action failure Principle 4.10 and Constitution Section 10 forbid. Option 3 keeps the transactional core (ADR-0010) fast and insulated from downstream system health while still guaranteeing every side effect is eventually and traceably handled.

## Consequences
- The originating request (checkout, order status change) stays fast regardless of downstream system health.
- New consumers of an existing event (for example, a future analytics pipeline) can be added without touching the code that produces the event.
- Introduces eventual consistency for everything downstream of the event, and requires deliberate idempotency design (Constitution Section 11) so that a retried event does not double-notify a customer or double-assign a dispatch task.
- Every event consumer must handle at-least-once delivery safely, and any event representing a consequential action must itself be auditable (Principle 4.10) if its outcome carries financial or lifecycle weight.

## Future reconsideration conditions
Evidenced need for stronger delivery or ordering guarantees than the initially adopted event mechanism provides — for example, strict ordering across a single customer's events under real concurrent load. Such evidence would prompt review of the specific event transport and delivery mechanics, not of the asynchronous-by-default decision itself.

## Related
- SRD section(s): Order lifecycle, notification, and dispatch/operations-workflow requirements
- Related DDD entity/data area(s): Order lifecycle state machine, Notification, Dispatch/Assignment entity areas
- Related architecture principle(s): 4.8, 4.10; Constitution Section 11 (idempotency)
- Related ADRs: 0010
