# 0035. Delivery-collected payment event (cash/card-on-delivery)

**Decision ID:** ADR-0035
**Status:** Accepted
**Date:** 2026-07-30
**Deciders:** Principal Architect (solo-project decision authority), remediating ARR Critical Finding #1

## Context

The Architecture Readiness Review found that no document described how a rider's cash-on-delivery or card-on-delivery collection (SRD FR-PAY-COD-001, FR-PAY-POS-003; DDD Section 9.6, "Delivery Completion") actually reaches Payment's owned tables. `module-catalog.md` places Delivery→Payment in Delivery's Forbidden Dependencies, and Payment's Consumed Events list only `OrderPlaced` (informational). With no orchestrator and no event carrying the collected-amount fact from Delivery to Payment, every cash/card-on-delivery order — the majority of orders at launch (SRD FR-PAY-001) — had no architecturally sound way to record its own payment.

## Problem

How does Payment learn that a rider collected cash or card-on-delivery payment at drop-off, without Delivery ever depending on Payment or writing to Payment's tables?

## Options considered

1. **Add a direct, synchronous Delivery→Payment dependency.** Rejected: it would add a new Forbidden-to-Allowed dependency reversal for a module (`module-catalog.md` §4.11) whose entire design principle is that it never needs to know about payment, and it would let a fulfillment-execution capability initiate a financial-settlement operation directly, which Payment's boundary (§4.9's Boundaries) exists specifically to prevent.
2. **Have Payment consume an existing event (`DeliveryCompleted`) and infer payment collection from it.** Rejected: `DeliveryCompleted` already exists to signal fulfillment completion to Order; conflating it with "and here is the payment amount, method, and reference" would make one event carry two unrelated facts (delivery completion, financial collection), violating the "ownership-scoped, one entity per event" naming and modeling discipline (`integration-and-messaging.md` Section 2–3) and forcing every existing `DeliveryCompleted` consumer to ignore fields irrelevant to it.
3. **Delivery publishes a new, dedicated event carrying the collection fact; Order consumes it and forwards to Payment through Payment's existing public interface.** Chosen.

## Decision

Delivery never owns payment data and never calls Payment. When a rider successfully collects cash-on-delivery or card-on-delivery payment at drop-off, Delivery publishes a new domain event, **`DeliveryCompletedWithPayment`**, carrying: Order ID, Delivery ID, Payment Method (cash or card-on-delivery), Amount Collected, Collection Timestamp, Driver (Rider) ID, and Reference Number (where available — e.g., a terminal transaction reference).

Order consumes `DeliveryCompletedWithPayment`. Order validates the order's own state (the order must be in a state where a delivery-collected payment is expected) and forwards payment recording through Payment's **existing** public interface — `RecordCashCollection` or `RecordCardOnDeliveryCollection` (`module-catalog.md` §4.9, already present, previously unreachable from Delivery's side of the flow). No new Payment interface is introduced; this decision closes the missing caller, not a missing capability.

Payment remains the sole owner of Payment, Payment History, Cash Remittance, Card-on-Delivery Record, Refund, and (per ADR-0037) the Payment Ledger. Delivery never writes to any Payment-owned table, directly or through any mechanism other than this event.

## Rationale

This preserves every existing boundary: Delivery's Forbidden Dependencies list (`module-catalog.md` §4.11) is unchanged — Delivery still never depends on Payment. Order already depends on both Delivery (indirectly, via `OrderReadyForFulfillment`/`DeliveryCompleted`) and Payment (`InitiatePayment`, already a Dependency), so routing the fact through Order as an event consumer and a same-existing-dependency forwarder introduces no new module-to-module dependency edge anywhere in the graph (`module-communication.md` Section 7) — it only adds a new event and a new call Order makes to an interface Payment already exposes. This is the same orchestration-with-compensation pattern (`module-communication.md` Section 8) already used for checkout: Order observes a fact from one module and acts on another module's public interface in response, never merging their transactions.

## Consequences

- `module-catalog.md`: Delivery's Published Events gains `DeliveryCompletedWithPayment`; Order's Consumed Events gains `DeliveryCompletedWithPayment`.
- `data-ownership-map.md`, `capability-boundary-map.md`, `service-decomposition.md`: Delivery's Data/Business Events Exposed sections and Order's event-handling description are updated to name the new event; no ownership, dependency, or transaction-boundary field changes.
- `integration-and-messaging.md`: the event follows the existing `<Entity><PastTenseVerb>` naming convention with one compound qualifier (`WithPayment`), the same documented pattern already used for `DriverAssignedToBatch` (ADR-0034) where a single-word verb would lose meaning.
- Order's SDD (when written) must model the state validation this event triggers (rejecting a collection fact for an order not in a state expecting one) as an explicit Order-owned business rule, not a pass-through.
- No transaction ever spans Delivery and Payment's tables — Delivery's own transaction (recording delivery completion) and Payment's own transaction (recording the collection, via `RecordCashCollection`/`RecordCardOnDeliveryCollection`) remain fully separate, consistent with ADR-0021.

## Future reconsideration conditions

Revisit only if a future payment method requires collection confirmation *before* delivery completion (which would change the event's trigger point, not this decision's ownership model) or if Order's role as forwarder becomes a proven bottleneck at a scale where direct eventing from Delivery to Payment is evidenced to be necessary — not anticipated at launch scale.

## Related

- SRD section(s): FR-PAY-COD-001–005, FR-PAY-POS-001–004, Section 10.2 (Payment State Machine)
- Related DDD entity/data area(s): 5.22 (Delivery Assignment), 5.24 (Payment), 5.26 (Cash Remittance), 5.27 (Card-on-Delivery Record), Section 9.6 (Delivery Completion)
- Related architecture principle(s): Principle 4.5 (module data ownership), Principle 4.8 (event-driven communication)
- Related ADRs: 0009, 0011, 0020, 0021, 0029, 0033
