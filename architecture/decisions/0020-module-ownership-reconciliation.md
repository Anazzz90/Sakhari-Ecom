# 0020. Reconcile module ownership and add Support module

**Status:** Accepted

## Context

`module-catalog.md` documented a fifteen-module backend list but flagged unresolved mismatches against the DDD entity ownership model: Support Ticket and Support Ticket Comment had no owner; Refund, Picking, Workforce, Pricing, and Auth/User ownership were working assumptions; and Search existed as a useful derived-read module without a backing ADR.

## Problem

Implementation cannot safely begin while ownership of business entities remains unresolved. The module catalog is supposed to be the boundary contract for SDDs and AI-assisted implementation; leaving entities unassigned invites inconsistent code and cross-module leakage.

## Options Considered

1. Keep the fifteen-module list and fold Support into Order or User.
2. Remove Search and replace it with a future implementation detail.
3. Add Support as a sixteenth module and formalize the existing folding assumptions.

## Decision

Adopt a sixteen-module backend catalog:

Auth, User, Store, Catalog, Search, Inventory, Cart, Order, Payment, Promotion, Delivery, Notification, Support, Analytics, Audit, Settings.

Ownership is settled as follows:

- Support Ticket and Support Ticket Comment belong to Support.
- Refund belongs to Payment.
- Picking Session and Picking Session Item belong to Delivery.
- Worker profile/capability/shift/availability belong to User.
- Price Snapshot belongs to Order.
- Auth owns identity/session/token concerns; User owns profile/workforce concerns.
- Search is an accepted module that owns only a derived, rebuildable search index and no primary business entity.

## Rationale

Support is a real operational capability in the SRD and DDD, not merely a property of User or Order. Making it a dedicated module keeps support workflows, ticket lifecycle, support permissions, and support audit behavior from being scattered across unrelated modules.

The other folding choices follow the strongest ownership fit already present in the DDD and existing architecture: refunds reverse payment activity, picking is part of fulfillment, workforce data is actor-profile data, and price snapshots are order-time facts. Search deserves a module boundary because product discovery is already a major product concern with 50k-100k products, but it must remain explicitly non-authoritative.

## Consequences

- The module catalog, communication rules, coding standards, and AI rules must refer to sixteen modules.
- Support gets its own SDD later.
- Search may evolve internally from PostgreSQL full-text search to OpenSearch or semantic search without changing other modules, because it is already behind a module interface.
- Permission scope must include Support-specific permissions.

## Future Reconsideration Conditions

Reconsider if Support remains trivial after launch and creates meaningful overhead, or if Search becomes purely an infrastructure concern with no business-facing interface. Either change would require a superseding ADR because it changes module ownership.

## Related Documents

- Related SRD section(s): Admin Dashboard, Support Tickets, Operations, Reporting
- Related DDD entity/data area(s): Support Ticket, Support Ticket Comment, Refund, Picking Session, Worker entities, Price Snapshot, Search/read models
- Related architecture principle(s): Principle 4.4, Principle 4.5, Principle 4.12

