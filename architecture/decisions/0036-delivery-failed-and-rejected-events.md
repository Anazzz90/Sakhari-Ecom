# 0036. Delivery failure and rejection events

**Decision ID:** ADR-0036
**Status:** Accepted
**Date:** 2026-07-30
**Deciders:** Principal Architect (solo-project decision authority), remediating ARR Critical Finding #2

## Context

The ARR found that Support's Consumed Events (`module-catalog.md` §4.13) already listed `DeliveryFailed`, but Delivery's own Published Events (§4.11) never included it, and Order's Consumed Events (§4.8) omitted it too — even though the SRD's Order State Machine (Section 10.1) has a "Failed Delivery" state with real business rules (BR-016 no-show handling), and the Delivery Assignment State Machine (Section 10.5) has its own "Failed" and "Rejected" states, each with a named trigger ("Rider/support failure reason," "Rider action"). A documented business state existed with no event able to drive it.

## Problem

Which events does Delivery publish when a delivery assignment fails or is rejected, and who consumes them?

## Options considered

1. **Leave failure/rejection entirely inside Delivery's own reassignment loop, never surfaced to Order.** Rejected: the SRD's Order State Machine explicitly requires Order to reach a "Failed Delivery" state with its own next-state options (Returned, Refund Pending, Cancelled) — Order cannot make that transition without ever learning the delivery failed.
2. **Publish only `DeliveryFailed`, treating rejection as an internal reassignment detail.** Rejected: the SRD's Delivery Assignment State Machine (Section 10.5) names "Rejected" as a distinct state from "Failed" with a different trigger (rider declines vs. rider/support failure reason) and a different immediate consequence (return to pool vs. potential order-level failure) — collapsing them into one event would lose information Support and Order might reasonably want to distinguish (a pattern of rejections is an operational signal; a failure is a customer-facing one).
3. **Publish both `DeliveryFailed` and `DeliveryRejected` as distinct events, each about the Delivery Assignment entity Delivery already owns.** Chosen.

## Decision

Delivery publishes two new domain events:

- **`DeliveryFailed`** — raised when a Delivery Assignment moves to the SRD's "Failed" state (Section 10.5: rider/support failure reason — customer unavailable, address unreachable, customer rejected order, rider unable to deliver, vehicle failure, safety issue, or a support-recorded equivalent). Carries: Order ID, Delivery Assignment ID, Rider ID, Failure Reason Code, and Timestamp.
- **`DeliveryRejected`** — raised when a Delivery Assignment moves to the SRD's "Rejected" state (Section 10.5: rider action, declining an offered assignment). Carries: Order ID, Delivery Assignment ID, Rider ID, and Timestamp. Unlike `DeliveryFailed`, a rejection alone does not change Order's own state — a rejected assignment returns to the same store's rider pool for reassignment (SRD Section 10.5's existing rule), and only becomes order-visible if reassignment itself is exhausted, which is reported as `DeliveryFailed` once Delivery's own retry/reassignment logic determines the order cannot proceed.

Order consumes `DeliveryFailed` (advancing the order to its own "Failed Delivery" state, per the SRD's Order State Machine, Section 10.1). Support continues to consume `DeliveryFailed`, exactly as `module-catalog.md` §4.13 already documented — this decision does not change Support's side, only supplies the event Support was already expecting. Order does not consume `DeliveryRejected` — a rejection alone is Delivery's own operational concern (reassignment) and never advances or changes Order's state, consistent with Order having no awareness of Delivery's internal assignment/reassignment mechanics elsewhere in the architecture (`module-catalog.md` §4.11's Boundaries).

## Rationale

This closes the gap using the same mechanism every other Delivery→Order signal already uses (`DeliveryAssigned`, `DeliveryCompleted`) — a published event about an entity Delivery owns, consumed by Order to drive Order's own state machine. It introduces no new dependency (Delivery already has Order as a consumer of its events) and no new module. Keeping `DeliveryRejected` un-consumed by Order (only used internally within Delivery's reassignment loop, if anyone outside Delivery ever needs it) respects Delivery's existing Boundary that reassignment/rejection handling is Delivery's own operational concern, not something Order reacts to individually per attempt.

## Consequences

- `module-catalog.md`: Delivery's Published Events gains `DeliveryFailed`, `DeliveryRejected`. Order's Consumed Events gains `DeliveryFailed`. Support's Consumed Events is unchanged (already listed `DeliveryFailed`) — no longer an orphaned reference.
- `data-ownership-map.md`, `capability-boundary-map.md`, `service-decomposition.md`: Delivery's event lists are updated to match; Order's event-handling description gains the Failed Delivery transition trigger.
- `integration-and-messaging.md`: both events follow the standard `<Entity><PastTenseVerb>` convention with no irregular naming, unlike ADR-0034's and ADR-0035's compound names.
- Order's SDD must map `DeliveryFailed` to the SRD's Order State Machine's "Failed Delivery" state transition (Section 10.1) as an explicit, tested consumer path.
- Support's existing consumption of `DeliveryFailed` (already documented) is now backed by an actual publisher — closing what was previously a dangling reference.

## Future reconsideration conditions

Revisit only if a future requirement needs Order to react to individual rejection attempts (not merely aggregate failure) — not anticipated, since the SRD's own state machine treats rejection as a Delivery-internal reassignment mechanic.

## Related

- SRD section(s): 10.1 (Order State Machine — Failed Delivery), 10.5 (Delivery Assignment State Machine — Failed, Rejected), BR-016
- Related DDD entity/data area(s): 5.22 (Delivery Assignment)
- Related architecture principle(s): Principle 4.5, Principle 4.8
- Related ADRs: 0009, 0011, 0020, 0021
