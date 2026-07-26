# Data Ownership Map — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. |
| **Stability** | Stable once written — changing an entity's owning module is a high-blast-radius change and must be ADR-backed (ADR-0009, ADR-0020), exactly as `module-catalog.md` and `capability-boundary-map.md` already are. |
| **Authority** | Subordinate to `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `decisions/README.md` (especially ADR-0002, ADR-0003, ADR-0004, ADR-0009, ADR-0020, ADR-0021), `03-decomposition/module-catalog.md`, `module-communication.md`, `capability-boundary-map.md`, and `04-cross-cutting/data-architecture.md`. This document introduces no entity, module, or boundary those documents don't already establish — it is the data-access lens on facts they already settle. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** Defines, for every persistent business entity in the system, exactly one owning module, and states precisely how every other module (and every external caller) is allowed to read or influence that data. `module-catalog.md` states ownership per module; `capability-boundary-map.md` states the business-decision boundary per module; this document is the third, final lens: the *data-access contract* — who may read what, who may write what, and through which mechanism — collected into one place so that "can module X touch this data, and how" has a single, checkable answer.

**Scope.** Per-module data ownership (owned tables, owned entities, aggregate roots, read/write responsibilities, public data interfaces, data exposed through events and APIs, forbidden data access, future extensibility); PostgreSQL and Redis ownership philosophy; why modules own their own data and why repositories are never shared; data consistency and transaction-ownership principles; a complete cross-system ownership matrix; allowed and forbidden access patterns; event-driven data sharing; and how reporting, analytics, and audit data are treated as a distinct category. Per this task's instruction, this document does **not** describe database schema implementation — no column lists, types, indexes, or constraints. Where it names a table, it names it the way `data-architecture.md` and the DDD's own naming conventions (Section 2) already do — a conceptual, plural, snake_case label for "the owned entity's persisted form" — never a schema definition.

**Intended audience.** Any developer or AI coding assistant about to read or write data and unsure whether it's allowed to reach a given table or entity directly; the project owner, verifying that a proposed data access pattern is either already sanctioned here or needs a new one added deliberately.

**Cross-references.** Built on `module-catalog.md` (per-module Ownership, Dependencies, Forbidden Dependencies, Published/Consumed Events — Section 4, all sixteen entries), `capability-boundary-map.md` (business-capability boundaries and Upstream/Downstream Dependencies), `module-communication.md` (Sections 2, 5, 7–9: allowed/forbidden communication, the dependency graph, transaction boundaries, circular-dependency prevention), and `04-cross-cutting/data-architecture.md` (PostgreSQL/Redis philosophy, transactions, the inventory model, soft deletes — assumed here, not repeated). Grounded in Principle 4.5 (modules own their own data), ADR-0009 (no cross-module repository access), ADR-0020 (the sixteen-module settlement), and ADR-0021 (checkout's module-local transaction model). DDD Section 6 (Relationships) is the source for aggregate-root framing, most directly its own statement that "order is the central transactional aggregate, but inventory, payment, refund, and delivery retain module ownership" (DDD §6.4).

## 2. PostgreSQL Ownership Philosophy

PostgreSQL is the exclusive system of record for every entity in this document (Principle 4.2, ADR-0003; `data-architecture.md` Section 2) — this document does not restate that rationale, only applies it consistently: every "Owned Tables" entry below is a PostgreSQL table, full stop, with no entity in this system persisted authoritatively anywhere else. Ownership in this document is inseparable from PostgreSQL ownership — to own an entity *is* to be the only module permitted to write its PostgreSQL rows.

## 3. Redis Ownership Philosophy

Redis owns nothing this document tracks. Consistent with Principle 4.3, ADR-0004, and `data-architecture.md` Section 3, Redis holds caches, sessions, rate-limit counters, and short-lived coordination state derived from or supporting PostgreSQL data — never an authoritative copy of an entity this document assigns ownership for. Where a module's own session or cache use touches data shaped like an owned entity (Auth's session state being the clearest case), that Redis-held copy is explicitly not a second source of truth — Section 12's ownership matrix lists only PostgreSQL-owned entities for exactly this reason.

## 4. Why Modules Own Data

Restating `capability-boundary-map.md` Section 2/3 and Principle 4.5 in this document's specific terms: a business capability without exclusive control over its own data is not really bounded — anyone could reach in and change what it's accountable for behind its back. Data ownership is what makes a capability boundary real rather than aspirational; it is the mechanism, not merely the intent, behind everything else in this document.

## 5. Why Repositories Are Never Shared

A shared repository (a data-access layer usable by more than one module) is the single most common way ADR-0009 gets violated in practice (`07-coding-standards/coding-standards.md` Section 13's own naming of this failure mode) — it looks like reuse, but it actually means two modules can write the same table, which is the exact condition Section 4 above says must never exist. Every module's data-access code is reachable only from within that module's own service layer (`coding-standards.md` Sections 2–3); this document's per-module "Owned Tables" and "Write Responsibilities" fields are what a shared-repository violation would be checked against.

## 6. Data Consistency Principles

- **Strong consistency within one module's own data**, always, via PostgreSQL transactions (`data-architecture.md` Section 5) — a module's own multi-record writes are atomic with respect to each other.
- **Eventual consistency across modules**, always, achieved through orchestration with compensation (Section 8 below) or asynchronous events (Section 10) — never a transaction spanning two modules' tables (ADR-0009, ADR-0021).
- **A read of another module's data is never assumed to be perfectly current** — a consumer reading Catalog data through Cart's own validation call, for example, accepts that Catalog could have changed a moment after the read; this is why optimistic revalidation (`data-architecture.md` Section 13) exists at the point of commitment, not just at the point of read.
- **Derived/read-model data (Search's index, Analytics' rollups) is never a consistency source** — if it disagrees with the owning module, the owning module is right, and the derived copy is rebuilt, never trusted over the source (Principle 4.5's reporting/read-model exception, applied identically to Search and Analytics throughout this document).

## 7. Transaction Ownership

Every transaction is owned by exactly one module, over that module's own tables only (ADR-0009, ADR-0021; `data-architecture.md` Section 5; `module-communication.md` Section 8) — this document does not repeat the orchestration-with-compensation mechanics `module-communication.md` Section 8 already covers in full. What this document adds is the ownership framing: "who may open a transaction touching this table" has exactly the same answer as "who owns this table," with no exception anywhere in the system. A module never opens a transaction that writes to a table it doesn't own, even as part of a larger orchestrated flow — Section 8's checkout example in `module-communication.md` is the canonical proof that this holds even for the system's most complex cross-module workflow.

## 8. Cross-Module Communication (Data Access Summary)

Full mechanics are `module-communication.md`'s subject; restated here strictly as it bears on data access — the only three ways a module may be affected by another module's data:

1. **Calling the owning module's public interface** and receiving data back — never a query against the owning module's tables (`module-communication.md` Section 5).
2. **Consuming a domain event** the owning module published, carrying only the data that module chose to include in the event (Section 10 below) — never a subscription to the owning module's raw data changes.
3. **Reading an explicitly-designed, purpose-built read model** (Search's index, Analytics' rollups) that the read-model owner builds and rebuilds from the source module's events — never a live join against the source module's operational tables (Principle 4.5's exception, Section 11 below).

## 9. Spotlight: Five Core Ownership Areas

The five ownership areas most load-bearing for the rest of the system, given deeper treatment here; each is fully catalogued again, alongside all sixteen modules, in Section 12.

### 9.1 Inventory Ownership

Inventory owns the only true answer to "how much stock exists and how much of it is spoken for" — three related but distinct tables (inventory items, reservations, ledger; `data-architecture.md` Section 6) that together let the system represent on-hand stock, in-flight claims against it, and the historical reason for every movement, without any of those three concerns being conflated into a single mutable number. No other module ever holds its own copy of a quantity — Cart, Order, and Search all ask Inventory (synchronously for Order's reservation step, via consumed events for Search's availability-aware index) rather than caching a number that could silently drift from the truth.

### 9.2 Order Ownership

Order is, in the DDD's own words, "the central transactional aggregate" (DDD §6.4) — but centrality of role is not the same as ownership of everything it touches. Order owns the order record, its items, its event history, and its frozen price snapshots; it does not own inventory, payment, promotion, or delivery data, each of which "retain[s] module ownership" (DDD §6.4) even while participating in the same checkout workflow Order orchestrates. This is the single most important distinction this document exists to make precise: Order being *central* to checkout does not make Order the *owner* of everything checkout touches.

### 9.3 User Ownership

User owns two related but distinct data clusters under one module boundary (ADR-0020): customer profile/address data, and worker profile/capability/shift/availability data. Both are actor-profile data in the same sense, which is why they share a module, but each forms its own aggregate (Section 12.2) — a customer's address book and a worker's shift history never reference each other, and nothing about User's ownership implies the two clusters are relationally joined at the data layer, only that they're governed by the same module boundary and the same actor-profile accountability.

### 9.4 Catalog Ownership

Catalog owns what a product *is* — globally, not per store (DDD §6.3: "Products are global") — while Inventory owns what's available *where*. This is a deliberate, load-bearing split: a product's definition does not change per store, but its availability does, and conflating the two into one owned entity would force every store-specific stock update to touch global product data, or force global product changes to somehow account for per-store state. Category and Brand are Catalog-owned reference data that Product relates to, not children of Product — they have their own lifecycle (DDD §5.9–5.10) independent of any single product referencing them.

### 9.5 Payment Ownership

Payment owns the financial truth of every order — not just successful charges, but history, cash/card-on-delivery collection records, and refunds, all under one module because they are one connected financial narrative (DDD §6.6: "payment history is immutable; manual corrections are additive"). Order never holds its own copy of payment status beyond what Payment's published events (`PaymentAuthorized`, `PaymentCaptured`, `PaymentFailed`) tell it — Order's own order record reflects *that* payment succeeded or failed as order-lifecycle state, but the financial detail (amounts, gateway references, collection method) lives only in Payment's own tables.

## 10. Event-Driven Data Sharing

An event carries only the data its publisher chooses to include — typically identifying references (which entity, which ULID) and the specific fact that changed, never a full copy of the publisher's internal record structure (`04-cross-cutting/integration-and-messaging.md` Sections 2–3). A consumer that needs more than an event's payload provides calls the publisher's public interface for the rest — it never assumes an event is a substitute for the owning module's own data. This is what keeps event payloads stable and small, and what prevents events from becoming an accidental second, looser data-ownership contract alongside the one this document defines.

## 11. Reporting, Analytics, and Audit Data

These three are grouped because they share one property none of the other fourteen modules have: **they hold data *about* the rest of the system, never data the rest of the system depends on.**

- **Reporting/Analytics data** (Analytics' Report Rollup and Analytics Snapshot) is explicitly rebuildable (DDD Entity Catalog, Section 4) — it is a purpose-built read model, built from consumed events and scheduled generation (`data-architecture.md` Section 6; `module-catalog.md` §4.14), never queried live against another module's operational tables, and never authoritative for anything it summarizes.
- **Audit data** (Audit's Audit Log) is the opposite kind of exception — immutable and never rebuilt, because it is itself the historical record, not a summary of one (`data-architecture.md` Sections 8, 15). It is written *to* by every module with a consequential action to record, but Audit itself never writes to, or is depended upon by, any other module's data.
- **Both categories are read via the owning module's public interface only** (`GetReportRollup`/`GetAnalyticsSnapshot`; `GetAuditTrailForEntity`/`GetAuditTrailForActor`) — never a direct query, even for an internal reporting need, consistent with every other ownership rule in this document.

## 12. Module Data Ownership Catalog

Each module below is documented against the ten fields this task requires. "Owned Tables" are conceptual, snake_case labels following the DDD's own naming convention (DDD Section 2.1) — not a schema. Full technical detail for Public Interfaces and Published/Consumed Events is `module-catalog.md`'s; this catalog states only the data-relevant subset.

### 12.1 Auth

| Field | Detail |
|---|---|
| **Owned Tables** | `users` (identity/credential record), `sessions`, `refresh_tokens` — the latter two Auth-internal, not separately named in the DDD (`module-catalog.md` §4.1) |
| **Owned Entities** | User (DDD §5.1) — identity/credential fields only, not profile fields |
| **Aggregate Roots** | User (root); Session and Refresh Token are dependent records scoped to a User, never independently meaningful |
| **Read Responsibilities** | Reads its own identity/session/token records to authenticate and validate every request |
| **Write Responsibilities** | Sole writer of `users` (identity fields), `sessions`, `refresh_tokens` |
| **Public Data Interfaces** | `ValidateToken` (returns identity/claims), `IssueTokenPair`, `RefreshAccessToken` — see `module-catalog.md` §4.1 |
| **Data Exposed Through Events** | `UserAuthenticated` (actor reference, timestamp), `OtpRequested` (actor reference), `SessionRevoked` (session/actor reference) — no credential material ever included |
| **Data Exposed Through APIs** | Identity/claims only, via `ValidateToken`; never raw credential or token-signing material |
| **Forbidden Data Access** | Never reads or writes Customer Profile, Address, or Worker data (User's tables) — Auth resolves identity, User resolves who that identity belongs to as a person |
| **Future Extensibility** | New authentication factors (passkeys) add fields/tables within this same ownership boundary, not a new owner |

### 12.2 User

| Field | Detail |
|---|---|
| **Owned Tables** | `customer_profiles`, `addresses`, `worker_profiles`, `worker_capabilities`, `shifts`, `worker_availability` |
| **Owned Entities** | Customer Profile (DDD §5.2), Address (§5.3), Worker Profile (§5.6), Worker Capability (§5.7), Shift (§5.8), Worker Availability |
| **Aggregate Roots** | Customer Profile (root; Address is a dependent child collection) and Worker Profile (root; Worker Capability, Shift, Worker Availability are dependent children) — two distinct aggregates under one module boundary (ADR-0020), never cross-referenced to each other |
| **Read Responsibilities** | Reads Auth's identity to resolve which profile a session belongs to |
| **Write Responsibilities** | Sole writer of all six owned tables |
| **Public Data Interfaces** | `GetCustomerProfile`, `GetDefaultAddress`, `GetWorkerCapabilities`, `GetCurrentShift` — see `module-catalog.md` §4.2 |
| **Data Exposed Through Events** | `CustomerProfileUpdated`, `AddressAdded`, `WorkerAvailabilityChanged` — actor reference and changed-field summary, not full profile dumps |
| **Data Exposed Through APIs** | Profile/address/worker fields relevant to the caller's own authorization scope (a customer sees their own profile; Delivery sees only capability/shift signals relevant to assignment) |
| **Forbidden Data Access** | Never reads or writes Order, Cart, Delivery, or Payment tables — User is read *from*, never a reader *of* transactional modules |
| **Future Extensibility** | A future dedicated Workforce module (re-splitting the worker-profile aggregate out) would require a superseding ADR to ADR-0020, not an incremental change here |

### 12.3 Store

| Field | Detail |
|---|---|
| **Owned Tables** | `stores`, `service_zones` |
| **Owned Entities** | Store (DDD §5.4), Service Zone (§5.5) |
| **Aggregate Roots** | Store (root; Service Zone is a dependent child collection) |
| **Read Responsibilities** | None from other modules — Store is one of the system's most foundational data owners |
| **Write Responsibilities** | Sole writer of `stores`, `service_zones` |
| **Public Data Interfaces** | `GetStore`, `ResolveServingStoreForAddress`, `GetServiceZone` — see `module-catalog.md` §4.3 |
| **Data Exposed Through Events** | `StoreActivated`, `StoreDeactivated`, `ServiceZoneChanged` — store/zone reference and status only |
| **Data Exposed Through APIs** | Store and service-zone definitions, read-only for every non-owning caller |
| **Forbidden Data Access** | Never reads or writes any other module's tables — store-scoped modules reference Store, Store never reaches into them |
| **Future Extensibility** | Multi-country expansion (DDD §17.1) adds fields to `stores` (currency, locale, region) within this same ownership boundary |

### 12.4 Catalog

| Field | Detail |
|---|---|
| **Owned Tables** | `categories`, `brands`, `products`, `product_images` |
| **Owned Entities** | Category (DDD §5.9), Brand (§5.10), Product (§5.11), Product Image (§5.12) |
| **Aggregate Roots** | Product (root; Product Image is a dependent child collection); Category and Brand are independent reference aggregates Product relates to, not children of Product (Section 9.4) |
| **Read Responsibilities** | Reads Store to scope catalog visibility where store-specific differences exist |
| **Write Responsibilities** | Sole writer of all four owned tables |
| **Public Data Interfaces** | `GetProduct`, `ListProductsByCategory`, `GetCategory`, `GetBrand` — see `module-catalog.md` §4.4 |
| **Data Exposed Through Events** | `ProductCreated`, `ProductUpdated`, `ProductDeactivated` (product reference and changed-field summary), `CategoryChanged` — the primary data feed Search's derived index is built from |
| **Data Exposed Through APIs** | Full product/category/brand definitions, read-only for every non-owning caller |
| **Forbidden Data Access** | Never reads or writes Inventory, Cart, Order, Payment, or Promotion tables — Catalog is purely upstream |
| **Future Extensibility** | Marketplace expansion (DDD §17.3, multi-seller ownership) is a boundary-level change requiring its own ADR, not an incremental extension |

### 12.5 Search

| Field | Detail |
|---|---|
| **Owned Tables** | None — a derived, rebuildable index, not a PostgreSQL-owned table under this document's ownership model (`module-catalog.md` §4.5) |
| **Owned Entities** | None |
| **Aggregate Roots** | None |
| **Read Responsibilities** | Reads Catalog's and Inventory's published events to build/refresh its own index; never reads their tables directly |
| **Write Responsibilities** | Writes only to its own derived index structure, which this document does not treat as an owned business entity |
| **Public Data Interfaces** | `SearchProducts`, `SuggestSearchTerms` — read-only, index-backed — see `module-catalog.md` §4.5 |
| **Data Exposed Through Events** | None — Search publishes no events (`module-catalog.md` §4.5) |
| **Data Exposed Through APIs** | Search results derived from Catalog/Inventory data, explicitly not authoritative — a caller needing authoritative data calls Catalog or Inventory directly |
| **Forbidden Data Access** | Never treated as, or queried as, a source of truth for product existence, price, or stock by any other module |
| **Future Extensibility** | A dedicated search technology behind this same interface changes nothing about any other module's data ownership |

### 12.6 Inventory

| Field | Detail |
|---|---|
| **Owned Tables** | `inventory_items`, `inventory_reservations`, `inventory_ledger` |
| **Owned Entities** | Inventory Item (DDD §5.13), Inventory Reservation (§5.14), Inventory Ledger (§5.15) |
| **Aggregate Roots** | Inventory Item (root for current stock state); Inventory Reservation and Inventory Ledger are related aggregates within Inventory's boundary, each referencing Inventory Item and (for reservations) an Order Item by reference, not nested underneath it (Section 9.1) |
| **Read Responsibilities** | Reads Store to scope stock to the correct store, Catalog to validate a referenced product |
| **Write Responsibilities** | Sole writer of all three owned tables, including every ledger entry |
| **Public Data Interfaces** | `GetAvailability`, `ReserveStock`, `ReleaseReservation`, `ConfirmReservationConsumption` — see `module-catalog.md` §4.6 |
| **Data Exposed Through Events** | `InventoryReserved`, `InventoryReservationFailed`, `InventoryReservationReleased`, `InventoryLevelChanged`, `InventoryLedgerRecorded` — quantities and references, never full row dumps |
| **Data Exposed Through APIs** | Current availability per store/product; reservation status for a specific reference |
| **Forbidden Data Access** | Never reads or writes Order, Cart, or Payment tables — it receives a reservation request by reference, never a reason tied to Order's own internal state |
| **Future Extensibility** | A strong future extraction candidate (`module-catalog.md` §4.6); ownership of its three tables does not change on extraction, only where they're hosted |

### 12.7 Cart

| Field | Detail |
|---|---|
| **Owned Tables** | `carts` |
| **Owned Entities** | Cart (DDD §5.16) |
| **Aggregate Roots** | Cart (root and, per the DDD's Entity Catalog, the sole entity in this module's boundary — no separate cart-item table is named) |
| **Read Responsibilities** | Reads Catalog to validate items still exist and are active |
| **Write Responsibilities** | Sole writer of `carts` |
| **Public Data Interfaces** | `GetCart`, `AddCartItem`, `RemoveCartItem` — see `module-catalog.md` §4.7 |
| **Data Exposed Through Events** | `CartUpdated` (optional, customer/cart reference only) |
| **Data Exposed Through APIs** | The customer's own cart contents, scoped to that customer only |
| **Forbidden Data Access** | Never reads or writes Inventory or Order tables — a cart reserves nothing and commits nothing |
| **Future Extensibility** | None anticipated; Cart is expected to remain simple by design (`module-catalog.md` §4.7) |

### 12.8 Order

| Field | Detail |
|---|---|
| **Owned Tables** | `orders`, `order_items`, `order_events`, `price_snapshots` |
| **Owned Entities** | Order (DDD §5.17), Order Item (§5.18), Order Event (§5.19), Price Snapshot (§5.32, per ADR-0020) |
| **Aggregate Roots** | Order (root; Order Item, Order Event, and Price Snapshot are all dependent children, never independently meaningful outside their owning order) — DDD's own "central transactional aggregate" framing (§6.4) |
| **Read Responsibilities** | Reads Cart at checkout time; reads User and Store to resolve customer/address/store context |
| **Write Responsibilities** | Sole writer of all four owned tables; does **not** write to Inventory, Payment, Promotion, or Delivery tables even while orchestrating checkout across them (Section 9.2) |
| **Public Data Interfaces** | `PlaceOrder`, `GetOrder`, `ListOrdersForCustomer`, `GetOrderEvents` — see `module-catalog.md` §4.8 |
| **Data Exposed Through Events** | `OrderPlaced`, `OrderCancelled`, `OrderStatusChanged`, `OrderReadyForFulfillment` — order reference, status, and the minimal context (store, customer reference) downstream consumers need |
| **Data Exposed Through APIs** | Full order detail for the owning customer or an authorized staff caller; order events for support/audit context |
| **Forbidden Data Access** | Never reads or writes Inventory Item, Payment, Promotion, or Delivery Assignment tables directly — every interaction with those capabilities goes through their public interface (Section 9.2) |
| **Future Extensibility** | Named alongside Inventory as the most plausible first extraction candidate; its four owned tables travel with it unchanged on extraction |

### 12.9 Payment

| Field | Detail |
|---|---|
| **Owned Tables** | `payments`, `payment_history`, `cash_remittances`, `card_on_delivery_records`, `refunds` |
| **Owned Entities** | Payment (DDD §5.24), Payment History (§5.25), Cash Remittance (§5.26), Card-on-Delivery Record (§5.27), Refund (§5.28, per ADR-0020) |
| **Aggregate Roots** | Payment (root; Payment History and Refund are dependent children); Cash Remittance and Card-on-Delivery Record are related aggregates scoped to a shift/store and referencing Payment, not nested underneath it |
| **Read Responsibilities** | Reads Order to validate the order/amount context a payment or refund applies to |
| **Write Responsibilities** | Sole writer of all five owned tables |
| **Public Data Interfaces** | `InitiatePayment`, `ConfirmPayment`, `IssueRefund`, `GetPaymentStatus` — see `module-catalog.md` §4.9 |
| **Data Exposed Through Events** | `PaymentAuthorized`, `PaymentCaptured`, `PaymentFailed`, `RefundIssued`, `RefundFailed`, `CashRemittanceRecorded` — status and reference only, never raw gateway payloads (`data-architecture.md` Section 12's "do not store raw card data" applies identically to event payloads) |
| **Data Exposed Through APIs** | Payment/refund status and history for the owning order, or an authorized staff/finance caller |
| **Forbidden Data Access** | Never reads or writes Order's own tables directly, Cart, Catalog, Inventory, or Promotion tables — settlement against a told-to context, never a reader of upstream modules |
| **Future Extensibility** | Terminal/POS settlement provider integration extends this same ownership boundary once resolved (Section 9.5) |

### 12.10 Promotion

| Field | Detail |
|---|---|
| **Owned Tables** | `promotions`, `promo_codes`, `promotion_usages` |
| **Owned Entities** | Promotion (DDD §5.29), Promo Code (§5.30), Promotion Usage (§5.31) |
| **Aggregate Roots** | Promotion (root; Promo Code is a dependent child); Promotion Usage is a related aggregate referencing both Promotion and the customer/order it applied to, recorded, not nested |
| **Read Responsibilities** | Reads Catalog for product/category-scoped rules, User for customer-scoped eligibility |
| **Write Responsibilities** | Sole writer of all three owned tables |
| **Public Data Interfaces** | `EvaluatePromotionEligibility`, `ApplyPromoCode`, `GetActivePromotions` — see `module-catalog.md` §4.10 |
| **Data Exposed Through Events** | `PromotionUsageRecorded` — promotion, customer, and order references only |
| **Data Exposed Through APIs** | Active promotion/eligibility results computed for a specific caller context; never a full dump of all promotion rules to an unauthorized caller |
| **Forbidden Data Access** | Never reads or writes Order, Cart, Payment, Inventory, or Delivery tables — a rules-and-eligibility service called by Order, never a caller of it |
| **Future Extensibility** | Advanced promotions and dynamic pricing (DDD §17.8–17.9) extend this same ownership boundary with additional rule complexity |

### 12.11 Delivery

| Field | Detail |
|---|---|
| **Owned Tables** | `picking_sessions`, `picking_session_items`, `delivery_assignments`, `rider_locations` |
| **Owned Entities** | Picking Session (DDD §5.20, per ADR-0020), Picking Session Item (§5.21), Delivery Assignment (§5.22), Rider Location (§5.23) |
| **Aggregate Roots** | Two distinct aggregates within one module: Picking Session (root; Picking Session Item is a dependent child) for the in-store half, and Delivery Assignment (root; Rider Location is a related, time-series tracking record referencing an assignment) for the last-mile half |
| **Read Responsibilities** | Reads Order for fulfillment-readiness signals, User for eligible pickers/riders, Store for scope |
| **Write Responsibilities** | Sole writer of all four owned tables |
| **Public Data Interfaces** | `StartPickingSession`, `AssignRider`, `RecordRiderLocation`, `GetActiveAssignmentForRider` — see `module-catalog.md` §4.11 |
| **Data Exposed Through Events** | `PickingStarted`, `PickingCompleted`, `DeliveryAssigned`, `DeliveryCompleted`, `RiderLocationUpdated` — task/assignment references and status, never full location history in one event |
| **Data Exposed Through APIs** | Active assignment detail for the assigned rider/picker; delivery status for the owning customer |
| **Forbidden Data Access** | Never reads or writes Order's own tables directly (reports back via events only), Payment, Catalog, or Promotion tables |
| **Future Extensibility** | Batch delivery (DDD §17.4) extends this same ownership boundary; Rider Location's short-retention data is a candidate for a dedicated storage path (a technology, not ownership, decision) |

### 12.12 Notification

| Field | Detail |
|---|---|
| **Owned Tables** | `notifications` |
| **Owned Entities** | Notification (DDD §5.33) |
| **Aggregate Roots** | Notification (root and sole entity) |
| **Read Responsibilities** | None from other modules synchronously — driven entirely by consumed events |
| **Write Responsibilities** | Sole writer of `notifications` |
| **Public Data Interfaces** | `SendNotification`, `GetNotificationHistory`, `GetDeliveryStatus` — see `module-catalog.md` §4.12 |
| **Data Exposed Through Events** | `NotificationSent`, `NotificationFailed`, `NotificationDeliveryConfirmed` — recipient/channel reference and status only |
| **Data Exposed Through APIs** | A user's own notification history and delivery status |
| **Forbidden Data Access** | Never reads or writes any transactional module's tables — it acts entirely on what an event or Auth's direct call already told it |
| **Future Extensibility** | Browser push and email channels extend `notifications`' own schema within this same ownership boundary |

### 12.13 Support

| Field | Detail |
|---|---|
| **Owned Tables** | `support_tickets`, `support_ticket_comments` |
| **Owned Entities** | Support Ticket (DDD §5.34), Support Ticket Comment (§5.35) |
| **Aggregate Roots** | Support Ticket (root; Support Ticket Comment is a dependent child collection) |
| **Read Responsibilities** | Reads User, Order, Payment, and Delivery for read-only context relevant to a ticket |
| **Write Responsibilities** | Sole writer of both owned tables; never writes to the tables of the entity a ticket discusses (`capability-boundary-map.md` §4.13) |
| **Public Data Interfaces** | `OpenTicket`, `AddTicketComment`, `GetTicket`, `ListTicketsForOrder` — see `module-catalog.md` §4.13 |
| **Data Exposed Through Events** | `SupportTicketOpened`, `SupportTicketAssigned`, `SupportTicketStatusChanged`, `SupportTicketCommentAdded`, `SupportTicketClosed`, `SupportTicketReopened` — ticket and linked-entity references |
| **Data Exposed Through APIs** | Ticket and comment detail for the owning customer or an authorized staff caller |
| **Forbidden Data Access** | Never writes to Order, Payment, Delivery, Inventory, or User tables — a business-state change triggered from a ticket calls the owning module's interface, exactly like any other caller |
| **Future Extensibility** | A future CRM/helpdesk integration is an external integration owned by Support, not a reason to relocate ticket ownership |

### 12.14 Analytics

| Field | Detail |
|---|---|
| **Owned Tables** | `report_rollups`, `analytics_snapshots` |
| **Owned Entities** | Report Rollup (DDD §5.38), Analytics Snapshot (§5.39) |
| **Aggregate Roots** | Report Rollup and Analytics Snapshot are each independent, self-contained aggregates — neither references the other, and neither has dependent children |
| **Read Responsibilities** | Reads no module's tables directly — built entirely from consumed events and scheduled generation |
| **Write Responsibilities** | Sole writer of both owned tables, both explicitly rebuildable (Section 11) |
| **Public Data Interfaces** | `GetReportRollup`, `GetAnalyticsSnapshot` — see `module-catalog.md` §4.14 |
| **Data Exposed Through Events** | None — Analytics publishes no events |
| **Data Exposed Through APIs** | Rollup/snapshot data for authorized reporting callers (Admin/Ops Dashboard, per RBAC) |
| **Forbidden Data Access** | Never on any business operation's critical path; never treated as authoritative for anything it summarizes |
| **Future Extensibility** | A dedicated analytical datastore behind this same interface requires no change to any other module's ownership |

### 12.15 Audit

| Field | Detail |
|---|---|
| **Owned Tables** | `audit_logs` |
| **Owned Entities** | Audit Log (DDD §5.37) |
| **Aggregate Roots** | Audit Log (root; append-only, no children — every entry is independently complete) |
| **Read Responsibilities** | None — Audit reads no other module's tables |
| **Write Responsibilities** | Sole writer of `audit_logs`; written *to* by every module with a consequential action to record, via `RecordAuditEntry` |
| **Public Data Interfaces** | `RecordAuditEntry`, `GetAuditTrailForEntity`, `GetAuditTrailForActor` — see `module-catalog.md` §4.15 |
| **Data Exposed Through Events** | None — Audit publishes no events |
| **Data Exposed Through APIs** | Audit trail entries for an authorized compliance/admin caller, scoped by entity or actor |
| **Forbidden Data Access** | Never interprets, judges, or is a precondition for the actions it records |
| **Future Extensibility** | None beyond volume growth, handled by partitioning/archival, not a boundary change |

### 12.16 Settings

| Field | Detail |
|---|---|
| **Owned Tables** | `settings` |
| **Owned Entities** | Settings (DDD §5.36) |
| **Aggregate Roots** | Settings (root and sole entity) |
| **Read Responsibilities** | None declared from other modules — see Section 15's Open Decision |
| **Write Responsibilities** | Sole writer of `settings` |
| **Public Data Interfaces** | `GetSetting`, `UpdateSetting`, `ListSettingsForScope` — see `module-catalog.md` §4.16 |
| **Data Exposed Through Events** | `SettingChanged` — setting key/scope reference only |
| **Data Exposed Through APIs** | Configuration values scoped to the caller's authorization level |
| **Forbidden Data Access** | Never holds a value with its own natural business owner elsewhere (a pricing rule, a delivery radius) |
| **Future Extensibility** | A versioning/rollback model for configuration changes extends this same ownership boundary |

## 13. Complete Ownership Matrix

Every owned entity in the system, in one table. "Who may read" and "who may modify" list modules by name; "—" means no module beyond the owner has a declared relationship. Access Method is always one of: **Interface** (synchronous public-interface call), **Event** (asynchronous consumed event), or **Derived** (a purpose-built, non-authoritative read model).

| Entity | Owner Module | Who May Read | Who May Modify | Access Method |
|---|---|---|---|---|
| User (identity/credential) | Auth | Every module (claims only, via `ValidateToken`) | Auth only | Interface |
| Session / Refresh Token | Auth | Auth only | Auth only | — (internal) |
| Customer Profile | User | Cart, Order, Delivery, Promotion, Support | User only | Interface |
| Address | User | Order, Delivery | User only | Interface |
| Worker Profile / Capability / Shift / Availability | User | Delivery, Support | User only | Interface |
| Store | Store | Catalog, Inventory, Order, Delivery | Store only | Interface |
| Service Zone | Store | Order, Delivery | Store only | Interface |
| Category / Brand | Catalog | Cart, Promotion; Search (derived) | Catalog only | Interface / Event |
| Product / Product Image | Catalog | Cart, Order, Inventory, Promotion; Search (derived) | Catalog only | Interface / Event |
| Inventory Item | Inventory | Order; Search (derived, optional) | Inventory only | Interface / Event |
| Inventory Reservation | Inventory | Order | Inventory only | Interface |
| Inventory Ledger | Inventory | — (internal history) | Inventory only | — (internal) |
| Cart | Cart | Order | Cart only | Interface |
| Order / Order Item | Order | Payment, Delivery, Support | Order only | Interface / Event |
| Order Event | Order | Support | Order only | Interface |
| Price Snapshot | Order | Payment (via order context) | Order only | Interface |
| Payment / Payment History | Payment | Order, Support | Payment only | Interface / Event |
| Cash Remittance / Card-on-Delivery Record | Payment | — (finance/reporting context via Payment) | Payment only | Interface |
| Refund | Payment | Order, Support | Payment only | Interface / Event |
| Promotion / Promo Code | Promotion | Order | Promotion only | Interface |
| Promotion Usage | Promotion | — (internal history) | Promotion only | Event (consumed: `OrderCancelled`) |
| Picking Session / Picking Session Item | Delivery | Order (via events) | Delivery only | Event |
| Delivery Assignment | Delivery | Order (via events), Support | Delivery only | Event / Interface |
| Rider Location | Delivery | — (operational only) | Delivery only | Interface |
| Notification | Notification | — (self-service history only) | Notification only | Interface |
| Support Ticket / Support Ticket Comment | Support | — (self-service/staff via Support) | Support only | Interface |
| Report Rollup / Analytics Snapshot | Analytics | Admin/Ops Dashboard callers (RBAC-gated) | Analytics only | Interface (Derived) |
| Audit Log | Audit | Admin/compliance callers (RBAC-gated) | Audit only (written to by every module) | Interface |
| Settings | Settings | Not explicitly declared by any module (Section 15) | Settings only | Interface |
| Search index | Search | — (client/API layer only) | Search only | Derived |

## 14. Allowed Access Patterns

- A module reading its own owned tables directly, inside its own transaction.
- A module calling another module's public interface to read or influence data it doesn't own (`module-communication.md` Section 5).
- A module consuming another module's published event to react to a change or maintain a derived read model (Section 10).
- Search and Analytics building and rebuilding their own derived structures from consumed events — never from a live query against the source module.
- Any module calling `RecordAuditEntry` to record a consequential action it performed — the one call every module is permitted to make into Audit's boundary without that being treated as a cross-module ownership violation, because Audit's own data (the log entry) is written by Audit itself, not by the caller reaching into `audit_logs` directly.

## 15. Forbidden Access Patterns

- Any module querying, joining against, or writing to another module's owned table, through any mechanism — an ORM relationship, a raw query, or a shared connection (ADR-0009; Section 5 above).
- A module treating a consumed event as a full substitute for the owning module's data, rather than calling the owning module's interface when it needs more than the event provides.
- A module treating Search's index or Analytics' rollups as authoritative for anything, ever (Section 11).
- A module bypassing another module's public interface "just for a read," even where the data would technically be reachable — a read-only violation is still a violation, because it still couples two modules' internal schemas together.
- Support (or any administrative caller) writing directly to the business entity a ticket or admin action concerns, instead of calling that entity's owning module's interface (`capability-boundary-map.md` Section 6).

## 16. Architecture Rules

1. Every persistent business entity has exactly one owning module, stated in Section 12 and Section 13, with no exception.
2. A module's data-access code is reachable only from within that module's own service layer (Section 5).
3. Cross-module data access happens only through an Interface call, a consumed Event, or a Derived read model (Section 13's Access Method column) — never a fourth mechanism.
4. No transaction ever spans two modules' owned tables (Section 7; ADR-0021).
5. A derived or reporting copy of data is never authoritative — the owning module's live state always wins a disagreement (Section 11).
6. Audit is written to by any module recording a consequential action, but itself depends on and writes to nothing but its own table (Section 12.15).
7. Any change to an entity's owning module is a high-blast-radius change requiring a new ADR (per ADR-0009/0020's own precedent), and must update `module-catalog.md`, `capability-boundary-map.md`, and this document together — never one without the others.

## 17. Examples of Correct and Incorrect Ownership

**Correct:** Order needs to know current stock availability before completing checkout. It calls Inventory's `GetAvailability`/`ReserveStock` interface and acts on the result — it never queries `inventory_items` directly, even though both tables live in the same PostgreSQL database and a join would be trivial to write.

**Incorrect:** A developer, under time pressure, adds a query inside Order's checkout code that reads `inventory_items` directly "just to double-check availability before calling `ReserveStock`." This is a violation even though it's read-only and even though it's "just a sanity check" — it couples Order's code to Inventory's internal schema, which is exactly what Section 15 forbids regardless of intent.

**Correct:** Search needs to reflect a product price change in its index. It consumes `ProductUpdated` and updates its own derived structure — Catalog never knows Search exists, and Search never calls Catalog synchronously to "make sure it has the latest."

**Incorrect:** Analytics, needing a same-day sales figure faster than its own scheduled rollup generation provides, is given a direct read connection to `orders` "just for this one dashboard widget." This is a violation of Section 15 and Section 11 simultaneously — it treats Analytics as something other than a purpose-built, event-derived read model, and it creates exactly the "two places order data can be read from" ambiguity Section 6 exists to prevent.

**Correct:** Support, resolving a delivery complaint, calls Payment's `IssueRefund` interface to compensate the customer — the refund appears in Payment's own tables, `RefundIssued` is published, and Order/Support both react to it as consumers, never as co-writers.

**Incorrect:** Support, to "save a step," writes a `resolved_via_refund` flag directly onto the `orders` table instead of calling Order's interface to record the outcome. Even a single flag, written directly, is still Section 15's core violation — the size of the write does not change whether it crossed an ownership boundary.

## 18. Open Decisions

- **Settings' readers are not declared anywhere.** No module's Dependencies field in `module-catalog.md`, nor its Downstream Dependencies in `capability-boundary-map.md`, names Settings — carried forward from `capability-boundary-map.md` Section 8 and restated here specifically because this document's Ownership Matrix (Section 13) would otherwise have to guess. It does not guess; the matrix records "not explicitly declared" rather than inventing a reader list.
- **Refund eligibility's decision owner** (Order vs. Support vs. both) remains open, carried from `capability-boundary-map.md` Section 8 — it does not change this document's ownership assignment (Payment owns `refunds` regardless of who decides eligibility), but it does mean Section 13's "Who May Modify" for Refund via Support is a live possibility, not a certainty, until that ambiguity is resolved.
- **Whether Auth's Session/Refresh Token state should be modeled as PostgreSQL tables or Redis-backed state** is not settled by any prior document — `module-catalog.md` calls them "Auth-internal, not separately named in the DDD," and `data-architecture.md` doesn't name them explicitly either. This document lists them as conceptual tables under Section 2's PostgreSQL-ownership philosophy for consistency, but flags that a Redis-backed implementation (consistent with Section 3's Redis-for-sessions framing) is equally plausible and not precluded — an SDD-level decision, not resolved here.
- **Cash Remittance and Card-on-Delivery Record's exact reader population** (which finance/reporting role, specifically, may read them) is not enumerated in any prior document beyond "finance reports depend on this entity" (DDD §5.26) — Section 13 leaves this general rather than inventing specific role names not established in `security-and-compliance.md`'s permission model.
