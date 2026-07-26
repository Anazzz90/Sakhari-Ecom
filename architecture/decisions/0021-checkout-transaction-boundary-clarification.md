# 0021. Clarify checkout transaction boundaries

**Status:** Accepted

## Context

ADR-0010 established transactional checkout and inventory reservation. Later architecture work clarified that each module owns its own data and no transaction spans multiple modules' tables. `module-communication.md` therefore describes checkout as orchestration with module-local transactions and explicit compensation, not as one PostgreSQL transaction across Order, Inventory, Payment, and Promotion.

## Problem

ADR-0010's wording could be read as requiring one database transaction spanning multiple modules. That contradicts ADR-0009's module ownership rule and the future service-extraction path.

## Options Considered

1. Keep ADR-0010 wording as-is and let `module-communication.md` override it informally.
2. Edit ADR-0010 directly.
3. Add a clarifying ADR that preserves history and states the operative rule.

## Decision

Checkout is implemented as orchestration by the Order module using module-local transactions:

- Order creates/updates its own order records in an Order-owned transaction.
- Inventory reserves stock in an Inventory-owned transaction.
- Payment initiation/confirmation uses Payment-owned transactions.
- Promotion evaluation and usage are invoked through Promotion's interface.
- If a later step fails, Order performs explicit compensating actions such as cancelling the pending order and instructing Inventory to release reservations.

No single database transaction may span two modules' owned tables.

## Rationale

This preserves strict module ownership while still satisfying the business requirement: no confirmed order should remain if inventory or payment fails. It also keeps the modular monolith extractable later because the workflow already behaves like a sequence of owned operations instead of a hidden shared transaction.

## Consequences

- SDDs must model checkout states explicitly enough to tolerate short-lived pending/intermediate states.
- Compensation logic is part of the Order module's checkout orchestration and must be tested.
- ADR-0010 remains historical; this ADR is the current clarification where wording differs.

## Future Reconsideration Conditions

Reconsider only if the architecture abandons strict module-owned persistence or introduces a distributed transaction pattern, both of which would require major ADRs.

## Related Documents

- Related SRD section(s): Order lifecycle, Inventory Management, Business Rules
- Related DDD entity/data area(s): Order, Order Item, Inventory Reservation, Payment, Promotion Usage
- Related architecture principle(s): Principle 4.5, Principle 4.9, Principle 4.12

