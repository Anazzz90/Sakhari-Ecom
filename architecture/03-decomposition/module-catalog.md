# Module Catalog — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. Supersedes the aggregate-table scope of `capability-boundary-map.md`, `service-decomposition.md`, and `data-ownership-map.md` as the primary, per-module source of truth; those three documents, once authored, should summarize and cross-reference this catalog rather than duplicate it. |
| **Stability** | Evolving in detail (a module's public interface or event list can grow); stable in shape (15 modules, one deployable, per `00-Architecture-Principles.md` and `01-Architecture-Design-Specification.md`). Reassigning a module's *ownership boundary* is a high-blast-radius change and should be ADR-backed. |
| **Authority** | Subordinate to `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `02-Architecture-Decisions.md` (the ADR set), `03-System-Context.md`, and `04-Technology-Stack.md`. Never contradicts them. Where this catalog had to make an assignment those documents don't settle, it is recorded in the **Open Decisions** section (Section 3) rather than asserted as fact. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** This is the authoritative catalog of the fifteen backend modules that make up the Sakhari Ecom modular monolith. For every module, it states what the module is responsible for, where its boundary sits, what it owns, how other modules are allowed to reach it, what it must never depend on, what events it produces and consumes, and how it is expected to grow. It exists so that a question like "can the Cart module read a Product row directly?" or "does Delivery own rider location data?" has one place to be answered definitively, instead of being re-derived from the Domain Design Document (DDD) or the Software Requirements Document (SRD) each time.

**Scope.** This document covers module-level boundaries and responsibilities only. It does not cover:
- *How* modules communicate mechanically (REST conventions, internal call patterns, transaction boundaries across modules) — that is `06-Module-Communication.md`.
- Database schema, table structure, or persistence mechanics — that is `07-Data-Architecture.md`.
- Security mechanisms (JWT, OTP, RBAC internals) beyond noting which module owns them — that is `08-Security-Architecture.md`.
- Event payload structure, retry, or delivery mechanics beyond naming what's published/consumed — that is `09-Event-Architecture.md`.
- Any implementation detail: no code, no class names, no file structure. "Public Interfaces" below are named capabilities, not method signatures.

**Intended audience.** Anyone extending, reviewing, or reasoning about the backend: the project owner, an AI coding assistant generating or reviewing code against a specific module, and the author of any future Software Design Document (SDD) for a given module. A future SDD for a module should treat that module's entry here as its starting boundary, not something it renegotiates.

**Cross-references.** This document realizes `01-Architecture-Design-Specification.md` Section 10 (Module Overview) and Section 7 (High-Level Architecture) at full granularity, and is built on `02-Architecture-Decisions.md`'s ADR-0002 (Modular Monolith), ADR-0009 (Module Ownership and No Cross-Module Repository Access), ADR-0010 (Transactional Checkout and Inventory Reservation), and ADR-0011 (Event-Driven Asynchronous Side Effects). It draws its entity-ownership grounding from the DDD's Entity Catalog (Section 4) and Transaction Boundaries (Section 9), reconciled where the two disagree — see Section 3.

---

## 2. How to Read Each Module Entry

Every module below is documented with the same ten fields, in the same order:

- **Responsibilities** — what the module does, in business terms.
- **Boundaries** — what the module explicitly does *not* do, and where its edge sits relative to its nearest neighbors.
- **Ownership** — the DDD entities this module is the source of truth for (Principle 4.5).
- **Public Interfaces** — the named operations other modules and the API layer may invoke. Every module is reached only through its public interface, never through a direct data access (ADR-0009). All are exposed externally via REST (per the confirmed REST API decision) through thin controllers (Principle 4.7); the same operations are what an internal caller uses for synchronous, transactional cross-module calls (ADR-0010).
- **Dependencies** — which other modules this module is allowed to call synchronously, and why.
- **Forbidden Dependencies** — which modules this module must never call, to prevent circular dependencies and to keep the dependency graph flowing in one direction. The full graph and its enforcement is `06-Module-Communication.md`'s subject; each entry here states only this module's own rule.
- **Published Events** — domain events this module raises for others to consume asynchronously (ADR-0011), following the `<Entity><PastTenseVerb>` naming convention used consistently across this catalog (e.g. `OrderPlaced`); full event architecture is `09-Event-Architecture.md`'s subject.
- **Consumed Events** — domain events this module reacts to.
- **Data Ownership** — restates Ownership as a plain entity list for quick scanning, consistent with the eventual `data-ownership-map.md`.
- **Future Expansion** — how this module is expected to grow, and, where relevant, its path to independent-service extraction without a change to its business logic (Constitution Section 13; `01-Architecture-Design-Specification.md` Section 15).

---

## 3. Open Decisions

Before the catalog: the fifteen-module list given for this task does not fully match the DDD's nineteen-module-owner breakdown (DDD Section 4, Entity Catalog) or the ADS's nine-capability grouping (`01-Architecture-Design-Specification.md` Section 10). Per this task's own instruction, these gaps are recorded here rather than resolved by silent assumption. Every assumption below was necessary to produce a complete catalog and is used consistently throughout Section 4, but **none of it should be read as a settled architectural decision** until reconciled — ideally by a dedicated ADR that formally supersedes or amends ADR-0009's module list and updates `capability-boundary-map.md` accordingly.

1. **DDD's "Support" module has no home in the fifteen-module list.** Support Ticket and Support Ticket Comment (DDD Sections 5.34–5.35) are not assigned anywhere in this catalog. Candidates: fold into User (support is closely tied to the customer/worker relationship), fold into Order (most tickets are order-scoped), or reinstate Support as a sixteenth module. **Not assigned below — flagged as unresolved**, not defaulted.
2. **DDD's "Refund" module is assumed folded into Payment.** Refund (DDD Section 5.28) has no separate owner in the fifteen-module list; this catalog assigns it to Payment on the reasoning that a refund is a payment-reversal operation. Needs confirmation.
3. **DDD's "Picking" module is assumed folded into Delivery.** Picking Session and Picking Session Item (DDD Sections 5.20–5.21) have no separate owner; this catalog assigns them to Delivery, reasoning that the ADS's original capability grouping (`01-Architecture-Design-Specification.md` Section 10, "Dispatch & Fulfillment") already combined picking and delivery into one capability. This is the lowest-risk assumption in this list, but is still a naming/ownership choice, not a restated fact.
4. **DDD's "Workforce" module is assumed folded into User.** Worker Profile, Worker Capability, Shift, and Worker Availability (DDD Sections 5.6–5.8, and Worker Availability referenced in Section 4) have no separate owner; this catalog assigns them to User alongside Customer Profile and Address, reasoning that User is the natural home for actor-profile data generally. Needs confirmation.
5. **DDD's "Pricing" module is assumed folded into Order, not Promotion.** Price Snapshot (DDD Section 5.32) is scoped by the DDD itself as "Order/item-scoped," which this catalog reads as closer to Order's ownership than Promotion's — but the ADS's original capability grouping paired pricing with promotions ("Pricing & Promotions"). This catalog picked Order; the alternative (Promotion) is equally defensible and unresolved.
6. **The Auth/User split is assumed, not confirmed.** The DDD's Entity Catalog assigns the User entity itself (Section 5.1) to a single "Auth & Identity" module. The fifteen-module list splits this into two modules (Auth, User); this catalog assumes Auth owns the core identity/credential record (the DDD's User entity, sessions, tokens) while the User module owns profile/extension data (Customer Profile, Address, and the folded-in Workforce entities per point 4 above). This split is architecturally reasonable and common in practice, but it is this catalog's interpretation, not a documented decision.
7. **Search is an entirely new module with no DDD or ADS precedent.** It owns no primary entity — see its entry in Section 4 for how this catalog reconciles that with Principle 4.5. Its introduction as a backend module is itself a decision with no backing ADR yet.
8. **The fifteen-module list as a whole has not been reconciled against `capability-boundary-map.md`,** which remains an unauthored skeleton. Once authored, that document should either adopt this catalog's fifteen-module breakdown formally (closing points 1–7 above) or record a different resolution — but it should not be authored independently of this catalog without addressing these seven open points first.

---

## 4. The Module Catalog

### 4.1 Auth

**Responsibilities:** Owns authentication for every actor in the system — customers, workers, and admin/ops/support staff. Issues and validates JWTs, manages refresh-token lifecycle, and orchestrates OTP-based login (request code, verify code, establish session). Is the single point every other module and every client ultimately depends on, directly or indirectly, to know who is calling.

**Boundaries:** Does not own actor profile data (name, address, worker capability) — that is User's job. Does not decide *what* an authenticated actor is allowed to do beyond proving identity and carrying role/permission claims for RBAC to evaluate — permission logic itself belongs to the security model documented in `08-Security-Architecture.md`, evaluated centrally but not "owned" by any single business module. Does not send the OTP SMS itself — it requests delivery through Notification, which owns the actual SMS/OTP provider integration named in `03-System-Context.md`.

**Ownership:** The DDD's User entity (Section 5.1) — the core identity/credential record — plus session and refresh-token state, which are Auth-internal and not separately named in the DDD.

**Public Interfaces:** RequestOtp, VerifyOtp, IssueTokenPair, RefreshAccessToken, RevokeSession, ValidateToken (used internally by every other module's request-handling path, not called by clients directly).

**Dependencies:** Notification (to request OTP delivery — Auth never talks to the SMS/OTP provider directly, consistent with `03-System-Context.md`'s rule that external systems are reached only through their owning module).

**Forbidden Dependencies:** Every business-facing module (Order, Payment, Cart, Catalog, Inventory, Delivery, Promotion). Auth sits at the foundation of the dependency graph; nothing about identity or session state should ever require knowing anything about a business capability.

**Published Events:** `UserAuthenticated`, `OtpRequested`, `SessionRevoked`.

**Consumed Events:** None. Auth is a foundational module other modules depend on; it does not react to business events.

**Data Ownership:** User (identity/credential record), Session, Refresh Token.

**Future Expansion:** The most likely candidate for early independent-service extraction (Constitution Section 13) precisely because it is already the most self-contained module — every other module depends on it, it depends on almost nothing. Extraction would change nothing about its business logic, only its deployment boundary (ADR-0002's designed reversibility).

---

### 4.2 User

**Responsibilities:** Owns profile data for both customer and worker actors: customer profile and saved addresses, and worker profile, capability, shift, and availability data (per Open Decision 4). Is the source of truth for "who this person is" beyond bare authentication.

**Boundaries:** Does not authenticate anyone — that's Auth. Does not decide worker task eligibility for a specific delivery or picking assignment — that's Delivery's job, informed by data User exposes (capability, shift status) through User's public interface, not by Delivery reading User's tables directly.

**Ownership:** Customer Profile (DDD 5.2), Address (DDD 5.3), Worker Profile (DDD 5.6), Worker Capability (DDD 5.7), Shift (DDD 5.8), Worker Availability.

**Public Interfaces:** GetCustomerProfile, UpdateCustomerProfile, AddAddress, GetDefaultAddress, GetWorkerProfile, GetWorkerCapabilities, GetCurrentShift, UpdateWorkerAvailability.

**Dependencies:** Auth (to resolve the identity a profile belongs to).

**Forbidden Dependencies:** Order, Payment, Cart, Delivery, Catalog, Inventory, Promotion. User is foundational, alongside Auth; it should never need to know about a specific order, cart, or delivery to serve its own responsibilities.

**Published Events:** `CustomerProfileUpdated`, `AddressAdded`, `WorkerAvailabilityChanged`.

**Consumed Events:** `UserAuthenticated` (from Auth, to initialize a profile view on first login where relevant).

**Data Ownership:** Customer Profile, Address, Worker Profile, Worker Capability, Shift, Worker Availability.

**Future Expansion:** Likely to be split further only if worker-profile concerns grow substantially distinct from customer-profile concerns at real scale (e.g., a dedicated Workforce module re-emerging, resolving Open Decision 4 in the other direction) — not anticipated at launch scale.

---

### 4.3 Store

**Responsibilities:** Owns the definition of each dark store and its service zone — the operating units the rest of the system scopes itself against (store isolation, DDD Section 13). Determines which store serves a given address/service zone.

**Boundaries:** Does not own inventory levels (Inventory's job), workers assigned to it (User's job, referenced by store), or orders placed against it (Order's job) — Store is the reference point those other modules scope themselves to, not the owner of their data.

**Ownership:** Store (DDD 5.4), Service Zone (DDD 5.5).

**Public Interfaces:** GetStore, ListActiveStores, ResolveServingStoreForAddress, GetServiceZone.

**Dependencies:** None. Store is one of the most foundational modules in the system.

**Forbidden Dependencies:** Every other module. Store must be resolvable without knowing about catalog, inventory, orders, or any operational state — everything else scopes itself to Store, not the reverse.

**Published Events:** `StoreActivated`, `StoreDeactivated`, `ServiceZoneChanged`.

**Consumed Events:** None.

**Data Ownership:** Store, Service Zone.

**Future Expansion:** Multi-country expansion (DDD Section 17.1) is the most likely driver of change here — additional store attributes (currency, locale, regulatory region) rather than a change to Store's ownership boundary.

---

### 4.4 Catalog

**Responsibilities:** Owns product, category, and brand data — the sellable-item definitions every other commerce module (Cart, Order, Inventory, Search, Promotion) references but never owns a copy of.

**Boundaries:** Does not own stock levels (Inventory's job) or pricing/promotional rules (Promotion's job) — Catalog owns *what a product is*, not *how many are available* or *what it costs today*. Does not maintain a search index — that is Search's derived responsibility, built from Catalog data, not a Catalog responsibility.

**Ownership:** Category (DDD 5.9), Brand (DDD 5.10), Product (DDD 5.11), Product Image (DDD 5.12).

**Public Interfaces:** GetProduct, ListProductsByCategory, GetCategory, GetBrand, ListProductImages.

**Dependencies:** Store (to scope catalog visibility where store-specific catalog differences exist).

**Forbidden Dependencies:** Cart, Order, Payment, Inventory, Promotion, Delivery. Catalog is upstream of all of these; none of them should ever need to be called by Catalog for Catalog to do its job.

**Published Events:** `ProductCreated`, `ProductUpdated`, `ProductDeactivated`, `CategoryChanged`.

**Consumed Events:** None.

**Data Ownership:** Category, Brand, Product, Product Image.

**Future Expansion:** Marketplace expansion (DDD Section 17.3) is the most likely driver — multi-seller catalog ownership — which would be a significant boundary change requiring its own ADR, not an incremental extension.

---

### 4.5 Search

**Responsibilities:** Provides fast, relevance-ranked product discovery (search-by-keyword, filtering, sorting) — a read path optimized for a query pattern PostgreSQL's own indexing is not the best fit for at scale.

**Boundaries:** Owns no primary business entity. Search is explicitly the "purpose-built read model" Principle 4.5 carves out as an exception to per-module data ownership: it maintains a derived, rebuildable index projected from Catalog (and, where availability-aware search is needed, Inventory), never a second copy of record. If the search index and Catalog/Inventory's own state disagree, Catalog/Inventory are always right — Search is never a source of truth for anything, only a discovery accelerator, consistent with the same non-authoritative posture Principle 4.3 already establishes for Redis.

**Ownership:** None. Search owns a derived index only, not a DDD entity.

**Public Interfaces:** SearchProducts, SuggestSearchTerms.

**Dependencies:** Catalog (initial and ongoing index population, via consumed events below, not direct table reads — ADR-0009 applies to Search exactly as it does to any other module). Inventory (optionally, to reflect availability in search results — this dependency should be event-driven, not synchronous, to avoid coupling search latency to inventory state).

**Forbidden Dependencies:** Order, Payment, Cart, Delivery, Promotion, User. Search's entire job is discovery over Catalog/Inventory data; it has no legitimate reason to reach any commerce-transaction module.

**Published Events:** None. Search is a pure consumer/read-path module.

**Consumed Events:** `ProductCreated`, `ProductUpdated`, `ProductDeactivated` (from Catalog), `InventoryLevelChanged` (from Inventory, if availability-aware search is implemented).

**Data Ownership:** None (derived index only — see Boundaries).

**Future Expansion:** The natural home for a dedicated search technology (e.g., a specialized search engine) should evidence ever justify one — because it owns no authoritative data, introducing such a technology behind Search's interface is a self-contained change that does not touch Principle 4.2's PostgreSQL-as-system-of-record guarantee anywhere else in the system. No dedicated ADR exists yet for Search's introduction as a module at all — flagged in Open Decisions (Section 3, point 7).

---

### 4.6 Inventory

**Responsibilities:** Owns stock levels per store, inventory reservations against in-flight orders, and the append-only inventory ledger recording every stock movement and why it happened. Is the sole authority on "how many of this product are actually available at this store right now."

**Boundaries:** Does not decide whether to reserve stock for a given order — that decision, and its timing, belongs to Order's checkout orchestration (see Section 4.8). Inventory exposes the reservation *operation*; it does not initiate reservations on its own. Does not own the product definition itself (Catalog's job) — Inventory Item references a Product but does not duplicate its data.

**Ownership:** Inventory Item (DDD 5.13), Inventory Reservation (DDD 5.14), Inventory Ledger (DDD 5.15).

**Public Interfaces:** GetAvailability, ReserveStock, ReleaseReservation, ConfirmReservationConsumption (used at packing completion, DDD Section 9.4), RecordLedgerAdjustment.

**Dependencies:** Store (to scope stock to the correct store), Catalog (to validate a referenced product exists and is active).

**Forbidden Dependencies:** Order, Cart, Payment, Delivery, Promotion. Inventory must be callable without knowing anything about the order, cart, or payment that triggered a reservation — it receives a request to reserve a quantity for a reference, not a dependency on Order's internal state.

**Published Events:** `InventoryReserved`, `InventoryReservationFailed`, `InventoryReservationReleased`, `InventoryLevelChanged`, `InventoryLedgerRecorded`.

**Consumed Events:** `OrderPlaced` is explicitly **not** consumed synchronously here — per the confirmed sequencing ("Inventory Reservation after Place Order"), Order calls Inventory's `ReserveStock` interface directly and synchronously as the next step in its own checkout orchestration (see Section 4.8's Boundaries), rather than Inventory reacting to a published event after the fact. This keeps reservation failure synchronously reportable back to the checkout flow rather than discovered asynchronously. See Open Decisions for the reconciliation this implies against ADR-0010's exact wording.

**Data Ownership:** Inventory Item, Inventory Reservation, Inventory Ledger.

**Future Expansion:** A strong extraction candidate at real scale (`01-Architecture-Design-Specification.md` Section 13 names Ordering and Catalog & Inventory as the most plausible capabilities to outgrow the monolith first) precisely because its reservation-locking behavior (DDD Section 10.1) is the most concurrency-sensitive part of the system.

---

### 4.7 Cart

**Responsibilities:** Owns the customer's in-progress, pre-checkout selection — a transactional draft, not yet a committed business record.

**Boundaries:** Does not reserve inventory — a cart having an item in it reserves nothing (DDD's own framing of Cart as a "Transactional Draft" with limited history requirements reflects this). Does not calculate final, order-time pricing — that happens at checkout, owned by Order, using Promotion and Catalog data at the moment of order creation, not the cart's own possibly-stale view.

**Ownership:** Cart (DDD 5.16).

**Public Interfaces:** GetCart, AddCartItem, RemoveCartItem, UpdateCartItemQuantity, ClearCart.

**Dependencies:** Catalog (to validate items being added still exist and are active), User (to scope the cart to a customer).

**Forbidden Dependencies:** Order, Payment, Inventory, Delivery. A cart is pre-transactional by design; nothing about it should require reaching into modules that only make sense once checkout begins.

**Published Events:** `CartUpdated` (optionally, for analytics/abandonment tracking — not required for correctness anywhere else in the system).

**Consumed Events:** `ProductDeactivated` (from Catalog, to flag or remove now-invalid cart items).

**Data Ownership:** Cart.

**Future Expansion:** No structural change anticipated; Cart is expected to remain simple relative to the rest of the system by design.

---

### 4.8 Order

**Responsibilities:** Owns the order record itself and orchestrates checkout — the confirmed architectural decision that "Checkout belongs to Order Module." Order is the module that turns a Cart into a committed business transaction: validating the cart, calling Inventory to reserve stock, calling Payment to initiate/confirm payment, applying Promotion results, and recording the outcome as an Order with Order Items and an append-only Order Event history. Also owns order-time price snapshots (Open Decision 5).

**Boundaries:** Does not perform inventory reservation itself — it calls Inventory's public interface to do so, as the next step after the order record exists (see Section 4.6's Consumed Events note). Does not process payment itself — it calls Payment's public interface and reacts to Payment's results. Does not decide delivery assignment — that is Delivery's job, triggered by Order reaching a ready-for-fulfillment state. Order is an **orchestrator** of checkout, not the owner of every entity checkout touches.

**Ownership:** Order (DDD 5.17), Order Item (DDD 5.18), Order Event (DDD 5.19), Price Snapshot (DDD 5.32, per Open Decision 5).

**Public Interfaces:** PlaceOrder, GetOrder, ListOrdersForCustomer, CancelOrder, GetOrderEvents, RecordOrderEvent.

**Dependencies:** Cart (to read the customer's selection at checkout time), Inventory (to reserve stock as the step immediately following order-record creation), Payment (to initiate/confirm payment), Promotion (to apply eligible promotions and record usage), User (to resolve the ordering customer and delivery address), Store (to resolve the serving store).

**Forbidden Dependencies:** Delivery, Notification, Analytics, Search. Order must never require a synchronous response from any of these to complete a checkout — their involvement is entirely downstream and asynchronous (ADR-0011), consistent with Principle 4.1's correctness-over-latency ordering: checkout's correctness depends on Inventory and Payment responding synchronously, and on nothing else.

**Published Events:** `OrderPlaced`, `OrderCancelled`, `OrderStatusChanged`, `OrderReadyForFulfillment`.

**Consumed Events:** `PaymentAuthorized`, `PaymentFailed`, `PaymentCaptured` (from Payment), `InventoryReservationFailed` (from Inventory, to trigger the rollback/no-confirmed-order-remains expectation the DDD names in Section 9.1), `PickingCompleted`, `DeliveryCompleted` (from Delivery, to advance order status).

**Data Ownership:** Order, Order Item, Order Event, Price Snapshot.

**Future Expansion:** The other capability the ADS names as most plausible to extract first, alongside Inventory (`01-Architecture-Design-Specification.md` Section 13). Because Order already reaches every other module exclusively through public interfaces and events, extraction would preserve its orchestration logic unchanged — only the deployment boundary moves (ADR-0002/0009).

---

### 4.9 Payment

**Responsibilities:** Owns payment state and settlement across mada, card, BNPL, cash-on-delivery, and card-on-delivery, and owns refund processing (Open Decision 2). Is the Backend's sole caller of the external Payment Gateway (`03-System-Context.md`).

**Boundaries:** Does not decide whether an order is eligible for a refund on business grounds (that's a rule Order/support process evaluates before calling Payment) — Payment executes and records the financial transaction, it does not adjudicate business eligibility beyond the payment-state constraints the DDD already defines (e.g., a refund amount must use the order's price snapshot, not current catalog price). Does not initiate itself — Order calls Payment as a checkout step.

**Ownership:** Payment (DDD 5.24), Payment History (DDD 5.25), Cash Remittance (DDD 5.26), Card-on-Delivery Record (DDD 5.27), Refund (DDD 5.28, per Open Decision 2).

**Public Interfaces:** InitiatePayment, ConfirmPayment, RecordCashCollection, RecordCardOnDeliveryCollection, IssueRefund, GetPaymentStatus.

**Dependencies:** Order (to validate the order/amount context a payment or refund applies to).

**Forbidden Dependencies:** Cart, Catalog, Inventory, Delivery, Search, Promotion. Payment's job is settlement against an order it's told about; it has no legitimate reason to reach into any module upstream of Order.

**Published Events:** `PaymentAuthorized`, `PaymentCaptured`, `PaymentFailed`, `RefundIssued`, `RefundFailed`, `CashRemittanceRecorded`.

**Consumed Events:** `OrderPlaced` (informational only, e.g. for reconciliation views — payment *initiation* is a direct synchronous call from Order to `InitiatePayment`, not event-triggered; resolved in `module-communication.md` Section 7, per the DDD's Section 9.1 treatment of payment initialization as part of order-creation orchestration).

**Data Ownership:** Payment, Payment History, Cash Remittance, Card-on-Delivery Record, Refund.

**Future Expansion:** Terminal/POS settlement provider integration for card-on-delivery reconciliation remains an explicitly unresolved "Phase 1 spike" per the DDD and `03-System-Context.md`; when resolved, it becomes a new external integration owned by Payment, not a new module.

---

### 4.10 Promotion

**Responsibilities:** Owns promotional rules, promo codes, and promotion usage tracking — eligibility and discount calculation applied consistently regardless of which client initiated the order (Principle 4.6).

**Boundaries:** Does not apply itself to an order — Order calls Promotion during checkout to evaluate eligibility and compute a discount; Promotion does not reach into Order to apply anything. Does not own the frozen order-time price record (Order's Price Snapshot, per Open Decision 5) — Promotion owns the *rule and its usage record*, not the *frozen result*.

**Ownership:** Promotion (DDD 5.29), Promo Code (DDD 5.30), Promotion Usage (DDD 5.31).

**Public Interfaces:** EvaluatePromotionEligibility, ApplyPromoCode, RecordPromotionUsage, GetActivePromotions.

**Dependencies:** Catalog (to evaluate product/category-scoped promotion rules), User (to evaluate customer-scoped eligibility, e.g., first-order promotions).

**Forbidden Dependencies:** Order, Cart, Payment, Inventory, Delivery. Promotion is a rules-and-eligibility service called by Order; it must never call back into Order or any downstream checkout module.

**Published Events:** `PromotionUsageRecorded`.

**Consumed Events:** `OrderCancelled` (to reverse usage tracking where a cancelled order's promotion usage should not count against a usage limit).

**Data Ownership:** Promotion, Promo Code, Promotion Usage.

**Future Expansion:** DDD Section 17.8 ("Advanced Promotions") and Section 17.9 ("Dynamic Pricing") are the named future-expansion drivers — additional rule complexity within Promotion's existing boundary, not a boundary change.

---

### 4.11 Delivery

**Responsibilities:** Owns in-store picking (Open Decision 3) and rider-based delivery: picking session execution, delivery assignment to riders, and rider location tracking during active delivery.

**Boundaries:** Does not own the order itself — it acts on an order reaching a fulfillment-ready state (signaled by `OrderReadyForFulfillment`) and reports back via events (`PickingCompleted`, `DeliveryCompleted`) rather than writing to Order's tables. Does not own worker profile/capability/shift data — it reads that from User to determine eligible pickers/riders, through User's public interface.

**Ownership:** Picking Session (DDD 5.20, per Open Decision 3), Picking Session Item (DDD 5.21), Delivery Assignment (DDD 5.22), Rider Location (DDD 5.23).

**Public Interfaces:** StartPickingSession, CompletePickingSession, AssignRider, RecordRiderLocation, CompleteDelivery, GetActiveAssignmentForRider.

**Dependencies:** Order (to know which orders are ready for picking/delivery, and to report completion back), User (to resolve eligible/available pickers and riders), Store (to scope picking/delivery to the correct store).

**Forbidden Dependencies:** Payment, Cart, Catalog, Promotion, Search. Delivery's job is entirely about moving a physical order from shelf to customer; none of these modules have any legitimate role in that.

**Published Events:** `PickingStarted`, `PickingCompleted`, `DeliveryAssigned`, `DeliveryCompleted`, `RiderLocationUpdated`.

**Consumed Events:** `OrderReadyForFulfillment` (from Order).

**Data Ownership:** Picking Session, Picking Session Item, Delivery Assignment, Rider Location.

**Future Expansion:** DDD Section 17.4 ("Batch Delivery") is the named future-expansion driver. Rider Location's short-retention, PDPL-reviewed data (DDD Section 5.23) is a candidate for its own dedicated, time-series-optimized storage path if evidenced load ever justifies it — a `04-Technology-Stack.md`-level decision, not a module-boundary one.

---

### 4.12 Notification

**Responsibilities:** Owns outbound communication to customers and workers — SMS, push, and future channels — and is the Backend's sole caller of the external SMS/OTP Provider and Push Notification Service (`03-System-Context.md`).

**Boundaries:** Does not decide *when* a business event warrants a notification — that is expressed by the event itself (Order, Payment, Delivery, and Auth all publish events Notification consumes); Notification's job is reliable delivery and delivery-status tracking, not business judgment about what's notification-worthy. Business state must never depend on notification delivery succeeding (DDD Section 5.33's own constraint) — Notification is purely a downstream, asynchronous consumer (ADR-0011).

**Ownership:** Notification (DDD 5.33).

**Public Interfaces:** SendNotification, GetNotificationHistory, GetDeliveryStatus.

**Dependencies:** None directly required to do its core job — it is driven by consumed events, not by synchronous calls from other modules, with one exception: Auth calls Notification directly to request OTP delivery, because that specific flow needs a synchronous "was the OTP dispatched" confirmation (Section 4.1's Dependencies).

**Forbidden Dependencies:** Order, Payment, Cart, Inventory, Delivery, Catalog, Promotion. Notification only ever reacts to events or Auth's direct OTP request; it never initiates a call into any business module.

**Published Events:** `NotificationSent`, `NotificationFailed`, `NotificationDeliveryConfirmed`.

**Consumed Events:** `OtpRequested`, `OrderPlaced`, `OrderStatusChanged`, `PaymentAuthorized`, `PaymentFailed`, `DeliveryAssigned`, `DeliveryCompleted` — effectively every published event in the system with a customer- or worker-facing communication implication, per `09-Event-Architecture.md`'s eventual full list.

**Data Ownership:** Notification.

**Future Expansion:** Browser notifications for the Customer Web App and email (both noted as future channels in the DDD and in `03-System-Context.md`) extend Notification's channel set without changing its boundary.

---

### 4.13 Analytics

**Responsibilities:** Owns reporting rollups and analytics snapshots — an explicit, purpose-built read model over the rest of the system's activity (Principle 4.5's reporting exception), never a live shortcut into another module's operational tables.

**Boundaries:** Never a source of truth for anything it reports on — if a rollup and the owning module's live state disagree, the owning module is always right, and the rollup is rebuilt or corrected, never the reverse (DDD Section 4: Report Rollup and Analytics Snapshot are both marked "Rebuildable").

**Ownership:** Report Rollup (DDD 5.38), Analytics Snapshot (DDD 5.39).

**Public Interfaces:** GetReportRollup, GetAnalyticsSnapshot, TriggerRollupGeneration.

**Dependencies:** None synchronously — Analytics is built entirely from consumed events and scheduled rollup generation (DDD Section 15.2, 15.7), never a direct call into another module's data.

**Forbidden Dependencies:** Every transactional module (Order, Payment, Cart, Inventory, Delivery). Analytics must never be on any business operation's critical path — nothing should ever wait on Analytics to respond for a checkout, payment, or delivery action to complete.

**Published Events:** None. Analytics is a pure read-side consumer.

**Consumed Events:** A broad subset of system events relevant to reporting — `OrderPlaced`, `OrderCancelled`, `PaymentCaptured`, `RefundIssued`, `DeliveryCompleted`, and others as reporting needs grow; the authoritative list belongs to `09-Event-Architecture.md`.

**Data Ownership:** Report Rollup, Analytics Snapshot.

**Future Expansion:** The most likely candidate for a future dedicated analytical datastore (a warehouse or columnar store) behind its own interface, entirely decoupled from PostgreSQL's system-of-record role — this would not require a change to any other module.

---

### 4.14 Audit

**Responsibilities:** Owns the append-only, immutable audit log — the durable record of who did what, when, satisfying Principle 4.10 for every action across the system with financial, inventory, or lifecycle consequence.

**Boundaries:** Never interprets or judges the actions it records — it is a faithful, immutable recorder, not a business-rule enforcer. Never blocks a business operation from completing — audit recording failure is itself an incident to be surfaced (per the "silent failure of consequential actions" anti-pattern, `00-Architecture-Principles.md` Section 10), not a reason to fail the underlying operation, though this tradeoff and its exact mechanics belong to `08-Security-Architecture.md` and `09-Event-Architecture.md`.

**Ownership:** Audit Log (DDD 5.37).

**Public Interfaces:** RecordAuditEntry, GetAuditTrailForEntity, GetAuditTrailForActor.

**Dependencies:** None. Audit is a foundational, universally-called module, not a caller of others.

**Forbidden Dependencies:** Every other module. Audit must be callable from anywhere without itself depending on anything else, or a failure anywhere else in the system could cascade into an inability to record what happened.

**Published Events:** None.

**Consumed Events:** Every event in the system that represents a consequential action is either directly recorded via `RecordAuditEntry` calls from the acting module, or consumed here for centralized recording — the exact mechanism (push from the acting module vs. Audit subscribing broadly) is left to `09-Event-Architecture.md` to settle.

**Data Ownership:** Audit Log.

**Future Expansion:** None anticipated beyond growth in volume; append-only ledger tables are explicitly designed (DDD Section 1.8) to scale by partitioning/archival (DDD Section 16), not by a boundary change.

---

### 4.15 Settings

**Responsibilities:** Owns global and store-scoped configuration — the platform's own operational knobs, distinct from any single business module's data.

**Boundaries:** Does not own business rules themselves (a promotion rule lives in Promotion, a delivery radius lives in Store/Service Zone) — Settings owns configuration values that don't belong to any one business capability, and configuration that governs cross-cutting behavior.

**Ownership:** Settings (DDD 5.36).

**Public Interfaces:** GetSetting, UpdateSetting, ListSettingsForScope.

**Dependencies:** None.

**Forbidden Dependencies:** Every other module. Settings must be resolvable independent of any business module's state.

**Published Events:** `SettingChanged`.

**Consumed Events:** None.

**Data Ownership:** Settings.

**Future Expansion:** As the number of configurable behaviors grows, Settings may need its own versioning/rollback model (DDD Section 5.36 already notes "Version/archive" for this entity) — an internal evolution, not a boundary change.

---

## 5. Cross-Module Summary Table

A quick-reference view of the fifteen modules and their DDD-entity grounding. This table is a navigation aid only — Section 4 is authoritative where the two differ in detail.

| Module | Primary Entities Owned | Owns No Primary Data? |
|---|---|---|
| Auth | User (identity/credential record), Session, Refresh Token | No |
| User | Customer Profile, Address, Worker Profile, Worker Capability, Shift, Worker Availability | No |
| Store | Store, Service Zone | No |
| Catalog | Category, Brand, Product, Product Image | No |
| Search | — | **Yes — derived index only** |
| Inventory | Inventory Item, Inventory Reservation, Inventory Ledger | No |
| Cart | Cart | No |
| Order | Order, Order Item, Order Event, Price Snapshot | No |
| Payment | Payment, Payment History, Cash Remittance, Card-on-Delivery Record, Refund | No |
| Promotion | Promotion, Promo Code, Promotion Usage | No |
| Delivery | Picking Session, Picking Session Item, Delivery Assignment, Rider Location | No |
| Notification | Notification | No |
| Analytics | Report Rollup, Analytics Snapshot | No (but rebuildable — see 4.13) |
| Audit | Audit Log | No |
| Settings | Settings | No |

**Unassigned DDD entities:** Support Ticket, Support Ticket Comment (see Open Decision 1).
