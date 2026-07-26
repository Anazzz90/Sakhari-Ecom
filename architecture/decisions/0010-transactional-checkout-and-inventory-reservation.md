# 0010. Transactional checkout and inventory reservation

**Decision ID:** ADR-0010
**Status:** Accepted
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
`00-Architecture-Principles.md` Principle 4.9 already states, at the principle level, that writes with an all-or-nothing business requirement execute inside a single database transaction. The SRD establishes that cash-on-delivery is a significant share of orders, meaning payment is not always pre-confirmed at checkout time, while inventory must still be correctly and atomically reserved regardless of payment method.

## Problem
How does the system guarantee that placing an order, reserving the inventory it requires, and recording payment state (confirmed, authorized, or pending-COD) never produces a state where an order exists without correctly reserved stock, or stock is decremented with no corresponding order?

## Options considered
1. **Separate, independent writes** for order creation and inventory decrement, reconciled after the fact if they disagree. Simple to write, but creates exactly the window where oversold stock or orphaned orders can occur.
2. **Optimistic, cache-based inventory check** — check availability in Redis, decrement there, reconcile against PostgreSQL eventually. Fast, but makes Redis momentarily authoritative for a business fact, directly violating Principle 4.3 (see ADR-0004).
3. **A single PostgreSQL transaction** encompassing order creation and inventory reservation, committed atomically, with payment state (authorized, captured, or pending-COD) recorded as part of the same transactional flow.

## Decision
Order placement and its associated inventory reservation execute within a single PostgreSQL transaction; the transaction is only considered committed if both the order record and the corresponding inventory reservation succeed together. Payment state — authorized, captured, or pending-COD — is recorded as part of the same transactional order-placement flow, never as a disconnected follow-up write.

## Rationale
Direct application of Principle 4.9, guarding against the exact failure modes Principle 4.1 exists to prevent: oversold stock and orphaned orders. Option 1 was rejected as the textbook version of that failure mode. Option 2 was rejected under ADR-0004's decision that Redis may accelerate an availability check but the reservation itself is not real until committed in PostgreSQL — Redis can speed up the pre-check, but does not replace the transactional commit.

## Consequences
- The system never has to represent or explain a half-placed order; failure is a clean rollback, not a support ticket or manual reconciliation.
- The transaction is kept short and focused, per Principle 4.9's own caveat: anything not strictly required for order/inventory/payment-state atomicity (notifications, dispatch assignment, analytics) is pushed out to asynchronous events (see ADR-0011), never included inside this transaction.
- Any future checkout-adjacent feature (promotions, split payments, partial fulfillment) must be evaluated for whether it belongs inside this atomic boundary or downstream of it.
- Cash-on-delivery orders are modeled honestly as "payment pending" within the same atomic order record, not as a special-cased or delayed write.

## Future reconsideration conditions
Evidenced lock contention on high-demand inventory (for example, a flash promotion on a limited-stock item) that this transactional approach cannot absorb even with appropriate indexing and locking strategy. Such evidence would trigger a review of reservation mechanics specifically (for example, a dedicated reservation/queueing step ahead of the transaction) without abandoning the underlying transactional guarantee itself.

## Related
- SRD section(s): Section 1.2 (cash-on-delivery as a significant share of orders), Ordering/Inventory data requirements (business rules and data-integrity scope)
- Related DDD entity/data area(s): Order, OrderItem, Payment/Transaction, InventoryStock/StockLevel entity areas and their lifecycle state machines
- Related architecture principle(s): 4.1, 4.9, 4.3
- Related ADRs: 0003, 0004, 0011
