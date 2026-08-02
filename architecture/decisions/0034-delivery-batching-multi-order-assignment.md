# 0034. Delivery batching (multi-order assignment)

**Decision ID:** ADR-0034
**Status:** Accepted
**Date:** 2026-07-29
**Deciders:** Principal Architect (solo-project decision authority)

## Context

DDD Section 17.4 ("Batch Delivery") and SRD BR-017 both already anticipated this: assignment cardinality was deliberately left open ("one active delivery per rider **unless batching is introduced later**"), and the DDD warned that "current design should avoid assuming delivery assignment can never relate to future route grouping." That evidence now exists — operational efficiency requires assigning more than one nearby order to a single rider trip. This ADR is the "later" both documents were written to expect, not a new architectural direction.

## Problem

How does a rider deliver more than one customer order in a single trip while every other principle already established for Order, Payment, Inventory, and Audit (independence, no cross-module transaction, no shared aggregate) remains true without exception?

## Options considered

1. **Merge multiple orders into one combined delivery/order record.** Rejected outright: it would violate ADR-0009/0021's module-ownership and transaction-independence rules, make cancellation of one order affect the other, and contradict Principle 4.5 (Order remains Order's own aggregate).
2. **Introduce a new "Dispatch" bounded context/module to own batching.** Rejected: batching is a fulfillment-execution concern with no new business capability, entity ownership, or actor — introducing a module for it would violate Principle 4.4 (capability-aligned decomposition) by splitting one capability (fulfillment) across two modules for no data-ownership reason.
3. **Add a new, Delivery-owned grouping entity that references existing per-order Delivery Assignments without altering their independence, ownership, or state machine.** Chosen.

## Decision

Delivery Batching is a capability of the existing Delivery module (DDD Section 17.4, resolved). Two new Delivery-owned entities are introduced:

- **Delivery Batch** — a rider-scoped, store-scoped grouping of up to two (MVP) active Delivery Assignments, with its own batch-level status, creation/assignment/completion timestamps.
- **Delivery Stop** — an ordered pointer from a Delivery Batch to one Delivery Assignment (and, through it, one order), carrying only a sequence number. A stop has no status of its own — its completion is read from the Delivery Assignment it points to (Section 10.5 of the SRD, unchanged).

The existing **Delivery Assignment** entity (DDD 5.22) is extended with two optional fields — `batch_id` and `sequence_in_batch` — both nullable, so a non-batched assignment (the common case today, and always for orders that don't clear batching eligibility) is unaffected in shape or behavior. Nothing about Order, Payment, Inventory, or Audit's ownership, transaction boundary, or state machine changes. A batch is a Delivery-internal grouping concern; it has no representation in Order, Payment, or Inventory's own tables or transactions, and no other module is ever aware a batch exists — Order still only ever sees `DeliveryAssigned`/`DeliveryCompleted`/`DeliveryFailed` for its own order, exactly as before.

MVP batch size is capped at two active orders per batch. Batching eligibility thresholds (proximity, readiness-time tolerance, driver capacity, SLA headroom) are Settings-owned configuration values (ADR-0033's existing Settings-reader pattern), not hardcoded constants — this is what lets a future larger batch size or tighter/looser threshold ship as a configuration change, not a new ADR.

## Rationale

This preserves every principle the task and the prior architecture both require: Orders, Payments, Inventory reservations, order status, and audit history all remain exactly as independent as they are today, because none of their owning modules' tables, transactions, or state machines are touched. Delivery Batch and Delivery Stop are purely Delivery-internal bookkeeping — an operational optimization layer sitting on top of unchanged per-order Delivery Assignments, not a new transactional unit spanning orders. Cancellation of one order cancels only its own Delivery Assignment (unchanged state machine, Cancelled is already a terminal/reachable state); the batch and its other stop are simply updated to reflect one fewer active stop, exactly the way removing one item from any list doesn't affect the other items. This also keeps the extraction story intact (ADR-0002/0009): Delivery could still be extracted into its own service without redesign, since batching never created a dependency on any other module.

## Consequences

- `module-catalog.md`, `capability-boundary-map.md`, `data-ownership-map.md`, and `service-decomposition.md`'s Delivery entries gain two owned entities, new public interface operations, and new published events — no new Dependencies or Forbidden Dependencies (Order, User, Store, Settings remain the complete set).
- Order's own consumed-event list, state machine, and transaction boundary are unchanged — Order continues to react only to `DeliveryAssigned`/`DeliveryCompleted`/`DeliveryFailed` for its own order and has no awareness of batching.
- Delivery's SDD (when written) must model batch formation as a Delivery-internal Application Service operation (evaluating eligibility, creating the batch, sequencing stops), never as a distributed operation touching another module's transaction.
- Rider capacity ("one active delivery per rider," SRD BR-017) is superseded by "one active delivery **or one active batch of up to two**, per rider" — BR-017 is updated in place (not a new BR) since the SRD itself flagged this as the expected resolution, not a reversal of an otherwise-final rule.
- Reassignment, rejection, and failure handling for a batched stop reuse the existing Delivery Assignment state machine (SRD Section 10.5) unchanged; batching adds only the bookkeeping of which other stop(s), if any, share the trip.

## Future reconsideration conditions

Revisit only if: batch size needs to exceed two (a configuration change first — Settings — before this ADR needs revisiting; this ADR itself is only implicated if a larger batch changes the *ownership or independence* model, e.g., a shared-capacity vehicle-routing concern that no longer fits "one rider, one batch, N independent stops"); a future routing engine requires Delivery to depend on a new external or internal capability not already in its Dependencies list; or evidence shows batching needs to be evaluated across store boundaries (explicitly out of scope here, consistent with SRD BR-021's store-scoped rider pool rule).

## Related

- SRD section(s): 5.5 (Rider Mode), 10.5 (Delivery Assignment State Machine), BR-017, BR-020, BR-021
- Related DDD entity/data area(s): 5.22 (Delivery Assignment, extended), 17.4 (Batch Delivery, resolved); new entities Delivery Batch, Delivery Stop
- Related architecture principle(s): Principle 4.4 (capability-aligned decomposition — no new module), Principle 4.5 (module data ownership), Principle 4.9 (transaction boundaries), Principle 4.10 (auditability)
- Related ADRs: 0002, 0009, 0011, 0017, 0020, 0021, 0029, 0033
