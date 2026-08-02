# Module Catalog — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. Supersedes the aggregate-table scope of `capability-boundary-map.md`, `service-decomposition.md`, and `data-ownership-map.md` as the primary, per-module source of truth; those three documents, once authored, should summarize and cross-reference this catalog rather than duplicate it. |
| **Stability** | Evolving in detail (a module's public interface or event list can grow); stable in shape (16 modules, one deployable, per `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, and ADR-0020). Reassigning a module's *ownership boundary* is a high-blast-radius change and should be ADR-backed. |
| **Authority** | Subordinate to `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `decisions/README.md` (the ADR set), `02-context/system-context.md`, and `04-cross-cutting/technology-decisions.md`. Never contradicts them. ADR-0020 resolves the module ownership reconciliation that earlier drafts left open. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** This is the authoritative catalog of the sixteen backend modules that make up the Sakhari Ecom modular monolith. For every module, it states what the module is responsible for, where its boundary sits, what it owns, how other modules are allowed to reach it, what it must never depend on, what events it produces and consumes, and how it is expected to grow. It exists so that a question like "can the Cart module read a Product row directly?" or "does Delivery own rider location data?" has one place to be answered definitively, instead of being re-derived from the Database Design Document (DDD) or the Software Requirements Document (SRD) each time.

**Scope.** This document covers module-level boundaries and responsibilities only. It does not cover:
- *How* modules communicate mechanically (REST conventions, internal call patterns, transaction boundaries across modules) — that is `03-decomposition/module-communication.md`.
- Database schema, table structure, or persistence mechanics — that is `04-cross-cutting/data-architecture.md`.
- Security mechanisms (JWT, OTP, RBAC internals) beyond noting which module owns them — that is `04-cross-cutting/security-and-compliance.md`.
- Event payload structure, retry, or delivery mechanics beyond naming what's published/consumed — that is `04-cross-cutting/integration-and-messaging.md`.
- Any implementation detail: no code, no class names, no file structure. "Public Interfaces" below are named capabilities, not method signatures.

**Intended audience.** Anyone extending, reviewing, or reasoning about the backend: the project owner, an AI coding assistant generating or reviewing code against a specific module, and the author of any future Software Design Document (SDD) for a given module. A future SDD for a module should treat that module's entry here as its starting boundary, not something it renegotiates.

**Cross-references.** This document realizes `01-Architecture-Design-Specification.md` Section 10 (Module Overview) and Section 7 (High-Level Architecture) at full granularity, and is built on `decisions/README.md`'s ADR-0002 (Modular Monolith), ADR-0009 (Module Ownership and No Cross-Module Repository Access), ADR-0010 (Transactional Checkout and Inventory Reservation, clarified by ADR-0021), ADR-0011 (Event-Driven Asynchronous Side Effects, made durable by ADR-0029's transactional outbox), ADR-0033 (Security and Business-Rule Closure, for refund eligibility, Settings readers, and POS/card-on-delivery ownership), ADR-0034 (Delivery Batching, resolving DDD Section 17.4), and ADR-0035–0041 (the delivery-collected-payment event, delivery failure/rejection events, the Payment Ledger, line-item-aware refunds, OTP abuse protection, step-up authentication, and store-scoped RBAC). It draws its entity-ownership grounding from the DDD's Entity Catalog (Section 4) and Transaction Boundaries (Section 9), reconciled where the two disagree — see Section 3.

---

## 2. How to Read Each Module Entry

Every module below is documented with the same ten fields, in the same order:

- **Responsibilities** — what the module does, in business terms.
- **Boundaries** — what the module explicitly does *not* do, and where its edge sits relative to its nearest neighbors.
- **Ownership** — the DDD entities this module is the source of truth for (Principle 4.5).
- **Public Interfaces** — the named operations other modules and the API layer may invoke. Every module is reached only through its public interface, never through a direct data access (ADR-0009). All are exposed externally via REST (per the confirmed REST API decision) through thin controllers (Principle 4.7); the same operations are what an internal caller uses for synchronous cross-module calls, each executing within the calling and called module's own transaction — never a transaction spanning both (ADR-0010, clarified by ADR-0021).
- **Dependencies** — which other modules this module is allowed to call synchronously, and why.
- **Forbidden Dependencies** — which modules this module must never call, to prevent circular dependencies and to keep the dependency graph flowing in one direction. The full graph and its enforcement is `03-decomposition/module-communication.md`'s subject; each entry here states only this module's own rule.
- **Published Events** — domain events this module raises for others to consume asynchronously (ADR-0011), following the `<Entity><PastTenseVerb>` naming convention used consistently across this catalog (e.g. `OrderPlaced`); full event architecture is `04-cross-cutting/integration-and-messaging.md`'s subject.
- **Consumed Events** — domain events this module reacts to.
- **Data Ownership** — restates Ownership as a plain entity list for quick scanning, consistent with the eventual `data-ownership-map.md`.
- **Future Expansion** — how this module is expected to grow, and, where relevant, its path to independent-service extraction without a change to its business logic (Constitution Section 13; `01-Architecture-Design-Specification.md` Section 15).

---

## 3. Module Ownership Reconciliation

ADR-0020 resolves the mismatch earlier drafts flagged between the DDD's entity-owner breakdown, the ADS's coarse capability grouping, and this catalog's backend module list. The settled backend catalog contains sixteen modules. The ownership decisions are:

1. **Support is a dedicated module.** Support Ticket and Support Ticket Comment belong to Support.
2. **Refund is folded into Payment.** Refund is treated as a payment-reversal operation.
3. **Picking is folded into Delivery.** Picking Session and Picking Session Item belong to the fulfillment boundary.
4. **Workforce is folded into User.** Worker Profile, Worker Capability, Shift, and Worker Availability belong to User.
5. **Pricing snapshot ownership belongs to Order.** Price Snapshot is an order-time fact, not a promotion rule.
6. **Auth/User split is settled.** Auth owns identity/session/token concerns; User owns profile/workforce concerns.
7. **Search is an accepted derived-read module.** It owns no authoritative business entity, only a rebuildable index.

`capability-boundary-map.md`, `service-decomposition.md`, and `data-ownership-map.md` should summarize this catalog and ADR-0020 when authored, not re-open these decisions.

---

## 4. The Module Catalog

### 4.1 Auth

**Responsibilities:** Owns authentication for every actor in the system — customers, workers, and admin/ops/support staff. Issues and validates JWTs, manages refresh-token lifecycle, and orchestrates OTP-based login (request code, verify code, establish session). Is the single point every other module and every client ultimately depends on, directly or indirectly, to know who is calling.

**Boundaries:** Does not own actor profile data (name, address, worker capability) — that is User's job. Does not decide *what* an authenticated actor is allowed to do beyond proving identity and carrying role/permission claims for RBAC to evaluate — permission logic itself belongs to the security model documented in `04-cross-cutting/security-and-compliance.md`, evaluated centrally but not "owned" by any single business module. Does not send the OTP SMS itself — it requests delivery through Notification, which owns the actual SMS/OTP provider integration named in `02-context/system-context.md`.

**Ownership:** The DDD's User entity (Section 5.1) — the core identity/credential record — plus session, refresh-token, and Store Scope Assignment state, all Auth-internal and not separately named in the DDD (per ADR-0041, Store Scope Assignment follows the same Auth-internal treatment already established for Session/Refresh Token).

**Public Interfaces:** RequestOtp, VerifyOtp, IssueTokenPair, RefreshAccessToken, RevokeSession, ValidateToken (used internally by every other module's request-handling path, not called by clients directly; returns identity, role/permission claims, and — per ADR-0041 — the caller's Assigned Stores for Resource Scope evaluation), InitiateStepUpChallenge, VerifyStepUpChallenge (per ADR-0040, for the four privileged roles performing a high-risk operation).

**Dependencies:** Notification (to request OTP delivery — Auth never talks to the SMS/OTP provider directly, consistent with `02-context/system-context.md`'s rule that external systems are reached only through their owning module), Settings (read-only, for cross-cutting configuration — ADR-0033, including OTP abuse-protection thresholds per ADR-0039).

**Forbidden Dependencies:** Every business-facing module (Order, Payment, Cart, Catalog, Inventory, Delivery, Promotion). Auth sits at the foundation of the dependency graph; nothing about identity or session state should ever require knowing anything about a business capability.

**Published Events:** `UserAuthenticated`, `OtpRequested`, `SessionRevoked`, `StepUpAuthenticationCompleted`, `StepUpAuthenticationFailed` (per ADR-0040, both within audit scope).

**Consumed Events:** None. Auth is a foundational module other modules depend on; it does not react to business events.

**Data Ownership:** User (identity/credential record), Session, Refresh Token, Store Scope Assignment (ADR-0041).

**Future Expansion:** The most likely candidate for early independent-service extraction (Constitution Section 13) precisely because it is already the most self-contained module — every other module depends on it, it depends on almost nothing. Extraction would change nothing about its business logic, only its deployment boundary (ADR-0002's designed reversibility).

---

### 4.2 User

**Responsibilities:** Owns profile data for both customer and worker actors: customer profile and saved addresses, and worker profile, capability, shift, and availability data (per ADR-0020). Is the source of truth for "who this person is" beyond bare authentication.

**Boundaries:** Does not authenticate anyone — that's Auth. Does not decide worker task eligibility for a specific delivery or picking assignment — that's Delivery's job, informed by data User exposes (capability, shift status) through User's public interface, not by Delivery reading User's tables directly.

**Ownership:** Customer Profile (DDD 5.2), Address (DDD 5.3), Worker Profile (DDD 5.6), Worker Capability (DDD 5.7), Shift (DDD 5.8), Worker Availability.

**Public Interfaces:** GetCustomerProfile, UpdateCustomerProfile, AddAddress, GetDefaultAddress, GetWorkerProfile, GetWorkerCapabilities, GetCurrentShift, UpdateWorkerAvailability.

**Dependencies:** Auth (to resolve the identity a profile belongs to).

**Forbidden Dependencies:** Order, Payment, Cart, Delivery, Catalog, Inventory, Promotion. User is foundational, alongside Auth; it should never need to know about a specific order, cart, or delivery to serve its own responsibilities.

**Published Events:** `CustomerProfileUpdated`, `AddressAdded`, `WorkerAvailabilityChanged`.

**Consumed Events:** `UserAuthenticated` (from Auth, to initialize a profile view on first login where relevant).

**Data Ownership:** Customer Profile, Address, Worker Profile, Worker Capability, Shift, Worker Availability.

**Future Expansion:** Likely to be split further only if worker-profile concerns grow substantially distinct from customer-profile concerns at real scale (e.g., a dedicated Workforce module re-emerging through a future ADR) — not anticipated at launch scale.

---

### 4.3 Store

**Responsibilities:** Owns the definition of each dark store and its service zone — the operating units the rest of the system scopes itself against (store isolation, DDD Section 13). Determines which store serves a given address/service zone.

**Boundaries:** Does not own inventory levels (Inventory's job), workers assigned to it (User's job, referenced by store), or orders placed against it (Order's job) — Store is the reference point those other modules scope themselves to, not the owner of their data.

**Ownership:** Store (DDD 5.4), Service Zone (DDD 5.5).

**Public Interfaces:** GetStore, ListActiveStores, ResolveServingStoreForAddress, GetServiceZone.

**Dependencies:** Settings (read-only, for cross-cutting configuration — ADR-0033). Otherwise none — Store is one of the most foundational modules in the system.

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

**Future Expansion:** The natural home for a dedicated search technology (e.g., a specialized search engine) should evidence ever justify one — because it owns no authoritative data, introducing such a technology behind Search's interface is a self-contained change that does not touch Principle 4.2's PostgreSQL-as-system-of-record guarantee anywhere else in the system. Search's existence as a module is accepted by ADR-0020.

---

### 4.6 Inventory

**Responsibilities:** Owns stock levels per store, inventory reservations against in-flight orders, and the append-only inventory ledger recording every stock movement and why it happened. Is the sole authority on "how many of this product are actually available at this store right now."

**Boundaries:** Does not decide whether to reserve stock for a given order — that decision, and its timing, belongs to Order's checkout orchestration (see Section 4.8). Inventory exposes the reservation *operation*; it does not initiate reservations on its own. Does not own the product definition itself (Catalog's job) — Inventory Item references a Product but does not duplicate its data.

**Ownership:** Inventory Item (DDD 5.13), Inventory Reservation (DDD 5.14), Inventory Ledger (DDD 5.15).

**Public Interfaces:** GetAvailability, ReserveStock, ReleaseReservation, ConfirmReservationConsumption (used at packing completion, DDD Section 9.4), RecordLedgerAdjustment.

**Dependencies:** Store (to scope stock to the correct store), Catalog (to validate a referenced product exists and is active), Settings (read-only, for cross-cutting configuration — ADR-0033).

**Forbidden Dependencies:** Order, Cart, Payment, Delivery, Promotion. Inventory must be callable without knowing anything about the order, cart, or payment that triggered a reservation — it receives a request to reserve a quantity for a reference, not a dependency on Order's internal state.

**Published Events:** `InventoryReserved`, `InventoryReservationFailed`, `InventoryReservationReleased`, `InventoryLevelChanged`, `InventoryLedgerRecorded`.

**Consumed Events:** `OrderPlaced` is explicitly **not** consumed synchronously here — per the confirmed sequencing ("Inventory Reservation after Place Order"), Order calls Inventory's `ReserveStock` interface directly and synchronously as the next step in its own checkout orchestration (see Section 4.8's Boundaries), rather than Inventory reacting to a published event after the fact. This keeps reservation failure synchronously reportable back to the checkout flow rather than discovered asynchronously. ADR-0021 clarifies the transaction-boundary wording around this flow.

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

**Responsibilities:** Owns the order record itself and orchestrates checkout — the confirmed architectural decision that "Checkout belongs to Order Module." Order is the module that turns a Cart into a committed business transaction: validating the cart, calling Inventory to reserve stock, calling Payment to initiate/confirm payment, applying Promotion results, and recording the outcome as an Order with Order Items and an append-only Order Event history. Also owns order-time price snapshots (ADR-0020).

**Boundaries:** Does not perform inventory reservation itself — it calls Inventory's public interface to do so, as the next step after the order record exists (see Section 4.6's Consumed Events note). Does not process payment itself — it calls Payment's public interface and reacts to Payment's results. Does not decide delivery assignment — that is Delivery's job, triggered by Order reaching a ready-for-fulfillment state. Order is an **orchestrator** of checkout, not the owner of every entity checkout touches.

**Ownership:** Order (DDD 5.17), Order Item (DDD 5.18), Order Event (DDD 5.19), Price Snapshot (DDD 5.32, per ADR-0020).

**Public Interfaces:** PlaceOrder, GetOrder, ListOrdersForCustomer, CancelOrder, GetOrderEvents, RecordOrderEvent.

**Dependencies:** Cart (to read the customer's selection at checkout time), Inventory (to reserve stock as the step immediately following order-record creation), Payment (to initiate/confirm payment), Promotion (to apply eligible promotions and record usage), User (to resolve the ordering customer and delivery address), Store (to resolve the serving store), Settings (read-only, for cross-cutting configuration — ADR-0033).

**Forbidden Dependencies:** Delivery, Notification, Analytics, Search. Order must never require a synchronous response from any of these to complete a checkout — their involvement is entirely downstream and asynchronous (ADR-0011), consistent with Principle 4.1's correctness-over-latency ordering: checkout's correctness depends on Inventory and Payment responding synchronously, and on nothing else.

**Published Events:** `OrderPlaced`, `OrderCancelled`, `OrderStatusChanged`, `OrderReadyForFulfillment`.

**Consumed Events:** `PaymentAuthorized`, `PaymentFailed`, `PaymentCaptured` (from Payment), `InventoryReservationFailed` (from Inventory, to trigger the rollback/no-confirmed-order-remains expectation the DDD names in Section 9.1), `PickingCompleted`, `DeliveryCompleted` (from Delivery, to advance order status), `DeliveryFailed` (per ADR-0036, to advance Order to the SRD's "Failed Delivery" state — Order does not consume `DeliveryRejected`, which is Delivery's own internal reassignment concern), `DeliveryCompletedWithPayment` (per ADR-0035 — Order validates order state and forwards the collection to Payment's `RecordCashCollection`/`RecordCardOnDeliveryCollection` interface; Order does not record the payment itself).

**Data Ownership:** Order, Order Item, Order Event, Price Snapshot.

**Future Expansion:** The other capability the ADS names as most plausible to extract first, alongside Inventory (`01-Architecture-Design-Specification.md` Section 13). Because Order already reaches every other module exclusively through public interfaces and events, extraction would preserve its orchestration logic unchanged — only the deployment boundary moves (ADR-0002/0009).

---

### 4.9 Payment

**Responsibilities:** Owns payment state and settlement across mada, card, BNPL, cash-on-delivery, and card-on-delivery, and owns refund processing (ADR-0020). Is the Backend's sole caller of the external Payment Gateway (`02-context/system-context.md`).

**Boundaries:** Does not decide whether an order is eligible for a refund on business grounds. Per ADR-0033, Order owns automatic refund-eligibility rules derived from order state, cancellation reason, delivery failure, and substitution outcome; Support may initiate a refund request with a reason; Payment validates payment/refund constraints (e.g., a refund amount must use the order's price snapshot, not current catalog price) and executes only after eligibility is approved by Order or an authorized Support approval flow. Does not initiate itself — Order calls Payment as a checkout step.

**Ownership:** Payment (DDD 5.24), Payment History (DDD 5.25), Cash Remittance (DDD 5.26), Card-on-Delivery Record (DDD 5.27), Refund (DDD 5.28, per ADR-0020, line-item aware and multi-partial-refund capable per ADR-0038), Payment Ledger (DDD 5.42, per ADR-0037 — the append-only record of every financial movement, structurally mirroring Inventory's Ledger).

**Public Interfaces:** InitiatePayment, ConfirmPayment, RecordCashCollection, RecordCardOnDeliveryCollection (per ADR-0035, now also the forwarding target for Order's `DeliveryCompletedWithPayment` handling), IssueRefund (per ADR-0038, accepts a per-line breakdown — Order Item references, quantity, and amount per line), GetPaymentStatus.

**Dependencies:** Order (to validate the order/amount context a payment or refund applies to), Settings (read-only, for cross-cutting configuration — ADR-0033).

**Forbidden Dependencies:** Cart, Catalog, Inventory, Delivery, Search, Promotion. Payment's job is settlement against an order it's told about; it has no legitimate reason to reach into any module upstream of Order. This is unaffected by ADR-0035 — Delivery still never calls Payment; Order remains the sole forwarder.

**Published Events:** `PaymentAuthorized`, `PaymentCaptured`, `PaymentFailed`, `RefundIssued`, `RefundFailed`, `CashRemittanceRecorded`, `PaymentLedgerRecorded` (per ADR-0037, mirroring `InventoryLedgerRecorded`).

**Consumed Events:** `OrderPlaced` (informational only, e.g. for reconciliation views — payment *initiation* is a direct synchronous call from Order to `InitiatePayment`, not event-triggered; resolved in `module-communication.md` Section 7, per the DDD's Section 9.1 treatment of payment initialization as part of order-creation orchestration).

**Data Ownership:** Payment, Payment History, Cash Remittance, Card-on-Delivery Record, Refund, Payment Ledger.

**Future Expansion:** Per ADR-0032/ADR-0033, MVP launch uses Payment-owned manual POS/card-on-delivery settlement import plus a discrepancy-review workflow; a terminal/POS provider settlement API becomes a new external integration owned by Payment once the terminal provider is confirmed, an optional enhancement rather than a launch blocker.

---

### 4.10 Promotion

**Responsibilities:** Owns promotional rules, promo codes, and promotion usage tracking — eligibility and discount calculation applied consistently regardless of which client initiated the order (Principle 4.6).

**Boundaries:** Does not apply itself to an order — Order calls Promotion during checkout to evaluate eligibility and compute a discount; Promotion does not reach into Order to apply anything. Does not own the frozen order-time price record (Order's Price Snapshot, per ADR-0020) — Promotion owns the *rule and its usage record*, not the *frozen result*.

**Ownership:** Promotion (DDD 5.29), Promo Code (DDD 5.30), Promotion Usage (DDD 5.31).

**Public Interfaces:** EvaluatePromotionEligibility, ApplyPromoCode, RecordPromotionUsage, GetActivePromotions.

**Dependencies:** Catalog (to evaluate product/category-scoped promotion rules), User (to evaluate customer-scoped eligibility, e.g., first-order promotions), Settings (read-only, for cross-cutting configuration — ADR-0033).

**Forbidden Dependencies:** Order, Cart, Payment, Inventory, Delivery. Promotion is a rules-and-eligibility service called by Order; it must never call back into Order or any downstream checkout module.

**Published Events:** `PromotionUsageRecorded`.

**Consumed Events:** `OrderCancelled` (to reverse usage tracking where a cancelled order's promotion usage should not count against a usage limit).

**Data Ownership:** Promotion, Promo Code, Promotion Usage.

**Future Expansion:** DDD Section 17.8 ("Advanced Promotions") and Section 17.9 ("Dynamic Pricing") are the named future-expansion drivers — additional rule complexity within Promotion's existing boundary, not a boundary change.

---

### 4.11 Delivery

**Responsibilities:** Owns in-store picking (ADR-0020) and rider-based delivery: picking session execution, delivery assignment to riders, rider location tracking during active delivery, and — per ADR-0034 — grouping up to two (MVP) independent, same-store delivery assignments into a single rider trip (delivery batching) when configurable eligibility rules are met.

**Boundaries:** Does not own the order itself — it acts on an order reaching a fulfillment-ready state (signaled by `OrderReadyForFulfillment`) and reports back via events (`PickingCompleted`, `DeliveryCompleted`) rather than writing to Order's tables. Does not own worker profile/capability/shift data — it reads that from User to determine eligible pickers/riders, through User's public interface. Delivery batching does not change any of this: a batch is a Delivery-internal grouping of assignments Delivery already owns — it never becomes a second owner of order, payment, or inventory state, and Order remains unaware a batch exists (it still only sees `DeliveryAssigned`/`DeliveryCompleted`/`DeliveryFailed` per order).

**Ownership:** Picking Session (DDD 5.20, per ADR-0020), Picking Session Item (DDD 5.21), Delivery Assignment (DDD 5.22, extended with an optional batch reference per ADR-0034), Rider Location (DDD 5.23), Delivery Batch (DDD 5.40, ADR-0034), Delivery Stop (DDD 5.41, ADR-0034).

**Public Interfaces:** StartPickingSession, CompletePickingSession, AssignRider, RecordRiderLocation, CompleteDelivery, GetActiveAssignmentForRider, EvaluateBatchEligibility, CreateDeliveryBatch, AssignBatchToRider, RecalculateBatchRoute, GetActiveBatchForRider (the last five per ADR-0034).

**COD/card-on-delivery collection (ADR-0035):** when a rider records a successful cash or card-on-delivery collection at drop-off (`CompleteDelivery`, carrying the collection fact), Delivery never calls Payment directly — it publishes `DeliveryCompletedWithPayment` (below) for Order to consume and forward through Payment's existing `RecordCashCollection`/`RecordCardOnDeliveryCollection` interface. Delivery's Forbidden Dependencies (below) are unaffected.

**Dependencies:** Order (to know which orders are ready for picking/delivery, and to report completion back), User (to resolve eligible/available pickers and riders), Store (to scope picking/delivery to the correct store), Settings (read-only, for cross-cutting configuration, including batching eligibility thresholds — ADR-0033, ADR-0034). Batching introduces no new dependency — eligibility evaluation and batch formation are Delivery-internal operations over data it already owns or already reads.

**Forbidden Dependencies:** Payment, Cart, Catalog, Promotion, Search. Delivery's job is entirely about moving a physical order from shelf to customer; none of these modules have any legitimate role in that. This is unchanged by batching — a batch never needs to call Payment, Cart, Catalog, Promotion, or Search to form or execute.

**Published Events:** `PickingStarted`, `PickingCompleted`, `DeliveryAssigned`, `DeliveryCompleted`, `RiderLocationUpdated`, `DeliveryFailed`, `DeliveryRejected` (the last two per ADR-0036), `DeliveryCompletedWithPayment` (per ADR-0035, raised alongside `DeliveryCompleted` only when the delivery included a cash/card-on-delivery collection), and — per ADR-0034 — `DeliveryBatchCreated`, `DriverAssignedToBatch`, `BatchRouteUpdated`, `BatchCompleted`, `BatchCancelled`.

**Consumed Events:** `OrderReadyForFulfillment` (from Order).

**Data Ownership:** Picking Session, Picking Session Item, Delivery Assignment, Rider Location, Delivery Batch, Delivery Stop.

**Future Expansion:** DDD Section 17.4 ("Batch Delivery") is resolved by ADR-0034 as of this revision — MVP caps a batch at two orders; a larger batch size is a Settings-configuration change, not a module-boundary change. A future routing engine (SRD FR-BATCH-010) replacing the initial stop-sequencing approach is a Delivery-internal implementation change, not an architectural one, provided it doesn't introduce a new external or cross-module dependency. Rider Location's short-retention, PDPL-reviewed data (DDD Section 5.23) remains a candidate for its own dedicated, time-series-optimized storage path if evidenced load ever justifies it — a `04-cross-cutting/technology-decisions.md`-level decision, not a module-boundary one.

---

### 4.12 Notification

**Responsibilities:** Owns outbound communication to customers and workers — SMS, push, and future channels — and is the Backend's sole caller of the external SMS/OTP Provider and Push Notification Service (`02-context/system-context.md`).

**Boundaries:** Does not decide *when* a business event warrants a notification — that is expressed by the event itself (Order, Payment, Delivery, and Auth all publish events Notification consumes); Notification's job is reliable delivery and delivery-status tracking, not business judgment about what's notification-worthy. Business state must never depend on notification delivery succeeding (DDD Section 5.33's own constraint) — Notification is purely a downstream, asynchronous consumer (ADR-0011).

**Ownership:** Notification (DDD 5.33).

**Public Interfaces:** SendNotification, GetNotificationHistory, GetDeliveryStatus.

**Dependencies:** None directly required to do its core job — it is driven by consumed events, not by synchronous calls from other modules, with one exception: Auth calls Notification directly to request OTP delivery, because that specific flow needs a synchronous "was the OTP dispatched" confirmation (Section 4.1's Dependencies). Notification also reads Settings (read-only, for template/channel configuration — ADR-0033).

**Forbidden Dependencies:** Order, Payment, Cart, Inventory, Delivery, Catalog, Promotion. Notification only ever reacts to events or Auth's direct OTP request; it never initiates a call into any business module.

**Published Events:** `NotificationSent`, `NotificationFailed`, `NotificationDeliveryConfirmed`.

**Consumed Events:** `OtpRequested`, `OrderPlaced`, `OrderStatusChanged`, `PaymentAuthorized`, `PaymentFailed`, `DeliveryAssigned`, `DeliveryCompleted` — effectively every published event in the system with a customer- or worker-facing communication implication, per `04-cross-cutting/integration-and-messaging.md`'s eventual full list.

**Data Ownership:** Notification.

**Future Expansion:** Browser notifications for the Customer Web App and email (both noted as future channels in the DDD and in `02-context/system-context.md`) extend Notification's channel set without changing its boundary.

---

### 4.13 Support

**Responsibilities:** Owns support tickets and support conversation history for customer, order, payment, refund, delivery, and worker-related issues. Provides the operational surface through which Admin/Ops/Support staff investigate and resolve customer or internal issues without taking ownership of the underlying order, payment, delivery, or inventory records.

**Boundaries:** Does not modify Order, Payment, Delivery, Inventory, or User records directly. A support action that changes business state (refund, cancellation, reassignment, profile correction) must call the owning module's public interface and must be authorized/audited like any other administrative action. Support owns the ticket and its comments; it does not own the business entity the ticket discusses.

**Ownership:** Support Ticket (DDD 5.34), Support Ticket Comment (DDD 5.35).

**Public Interfaces:** OpenTicket, AddTicketComment, AssignTicket, UpdateTicketStatus, LinkTicketToEntity, GetTicket, ListTicketsForCustomer, ListTicketsForOrder.

**Dependencies:** User (to resolve the customer/worker profile attached to a ticket), Order (for order-scoped ticket context), Payment (for payment/refund context where applicable), Delivery (for delivery-assignment context where applicable).

**Forbidden Dependencies:** Catalog, Inventory, Cart, Promotion, Search. Support may reference these indirectly through the owning business entity or through read-only context surfaced by the owning module, but it does not need direct calls into them to own a support workflow.

**Published Events:** `SupportTicketOpened`, `SupportTicketAssigned`, `SupportTicketStatusChanged`, `SupportTicketCommentAdded`, `SupportTicketClosed`, `SupportTicketReopened`.

**Consumed Events:** `OrderCancelled`, `PaymentFailed`, `RefundIssued`, `DeliveryCompleted`, `DeliveryFailed` where these events are useful for automatically linking or updating support context. Support must not be required for those workflows to complete.

**Data Ownership:** Support Ticket, Support Ticket Comment.

**Future Expansion:** A richer CRM/helpdesk integration could eventually sit behind Support's boundary. That would be an external integration owned by Support, not a reason for Order, Payment, or User to own tickets.

---

### 4.14 Analytics

**Responsibilities:** Owns reporting rollups and analytics snapshots — an explicit, purpose-built read model over the rest of the system's activity (Principle 4.5's reporting exception), never a live shortcut into another module's operational tables.

**Boundaries:** Never a source of truth for anything it reports on — if a rollup and the owning module's live state disagree, the owning module is always right, and the rollup is rebuilt or corrected, never the reverse (DDD Section 4: Report Rollup and Analytics Snapshot are both marked "Rebuildable").

**Ownership:** Report Rollup (DDD 5.38), Analytics Snapshot (DDD 5.39).

**Public Interfaces:** GetReportRollup, GetAnalyticsSnapshot, TriggerRollupGeneration.

**Dependencies:** None synchronously required for its core job — Analytics is built entirely from consumed events and scheduled rollup generation (DDD Section 15.2, 15.7), never a direct call into another module's data. Analytics also reads Settings (read-only, for cross-cutting configuration — ADR-0033).

**Forbidden Dependencies:** Every transactional module (Order, Payment, Cart, Inventory, Delivery). Analytics must never be on any business operation's critical path — nothing should ever wait on Analytics to respond for a checkout, payment, or delivery action to complete.

**Published Events:** None. Analytics is a pure read-side consumer.

**Consumed Events:** A broad subset of system events relevant to reporting — `OrderPlaced`, `OrderCancelled`, `PaymentCaptured`, `RefundIssued`, `DeliveryCompleted`, and others as reporting needs grow; the authoritative list belongs to `04-cross-cutting/integration-and-messaging.md`.

**Data Ownership:** Report Rollup, Analytics Snapshot.

**Future Expansion:** The most likely candidate for a future dedicated analytical datastore (a warehouse or columnar store) behind its own interface, entirely decoupled from PostgreSQL's system-of-record role — this would not require a change to any other module.

---

### 4.15 Audit

**Responsibilities:** Owns the append-only, immutable audit log — the durable record of who did what, when, satisfying Principle 4.10 for every action across the system with financial, inventory, or lifecycle consequence.

**Boundaries:** Never interprets or judges the actions it records — it is a faithful, immutable recorder, not a business-rule enforcer. Never blocks a business operation from completing — audit recording failure is itself an incident to be surfaced (per the "silent failure of consequential actions" anti-pattern, `00-Architecture-Principles.md` Section 10), not a reason to fail the underlying operation, though this tradeoff and its exact mechanics belong to `04-cross-cutting/security-and-compliance.md` and `04-cross-cutting/integration-and-messaging.md`.

**Ownership:** Audit Log (DDD 5.37).

**Public Interfaces:** RecordAuditEntry, GetAuditTrailForEntity, GetAuditTrailForActor.

**Dependencies:** None. Audit is a foundational, universally-called module, not a caller of others.

**Forbidden Dependencies:** Every other module. Audit must be callable from anywhere without itself depending on anything else, or a failure anywhere else in the system could cascade into an inability to record what happened.

**Published Events:** None.

**Consumed Events:** Every event in the system that represents a consequential action is either directly recorded via `RecordAuditEntry` calls from the acting module, or consumed here for centralized recording — the exact mechanism (push from the acting module vs. Audit subscribing broadly) is left to `04-cross-cutting/integration-and-messaging.md` to settle.

**Data Ownership:** Audit Log.

**Future Expansion:** None anticipated beyond growth in volume; append-only ledger tables are explicitly designed (DDD Section 1.8) to scale by partitioning/archival (DDD Section 16), not by a boundary change.

---

### 4.16 Settings

**Responsibilities:** Owns global and store-scoped configuration — the platform's own operational knobs, distinct from any single business module's data.

**Boundaries:** Does not own business rules themselves (a promotion rule lives in Promotion, a delivery radius lives in Store/Service Zone) — Settings owns configuration values that don't belong to any one business capability, and configuration that governs cross-cutting behavior.

**Ownership:** Settings (DDD 5.36).

**Public Interfaces:** GetSetting, UpdateSetting, ListSettingsForScope.

**Dependencies:** None.

**Forbidden Dependencies:** Every other module. Settings must be resolvable independent of any business module's state.

**Published Events:** `SettingChanged`.

**Consumed Events:** None.

**Readers:** Per ADR-0033, Auth, Notification, Order, Payment, Delivery, Inventory, Promotion, Store, and Analytics may read configuration through `GetSetting`/`ListSettingsForScope`; only Settings ever writes a setting (`UpdateSetting`). A reading module's own Dependencies field lists Settings; this does not make Settings dependent on them.

**Data Ownership:** Settings.

**Future Expansion:** As the number of configurable behaviors grows, Settings may need its own versioning/rollback model (DDD Section 5.36 already notes "Version/archive" for this entity) — an internal evolution, not a boundary change.

---

## 5. Cross-Module Summary Table

A quick-reference view of the sixteen modules and their DDD-entity grounding. This table is a navigation aid only — Section 4 is authoritative where the two differ in detail.

| Module | Primary Entities Owned | Owns No Primary Data? |
|---|---|---|
| Auth | User (identity/credential record), Session, Refresh Token, Store Scope Assignment | No |
| User | Customer Profile, Address, Worker Profile, Worker Capability, Shift, Worker Availability | No |
| Store | Store, Service Zone | No |
| Catalog | Category, Brand, Product, Product Image | No |
| Search | — | **Yes — derived index only** |
| Inventory | Inventory Item, Inventory Reservation, Inventory Ledger | No |
| Cart | Cart | No |
| Order | Order, Order Item, Order Event, Price Snapshot | No |
| Payment | Payment, Payment History, Cash Remittance, Card-on-Delivery Record, Refund, Payment Ledger | No |
| Promotion | Promotion, Promo Code, Promotion Usage | No |
| Delivery | Picking Session, Picking Session Item, Delivery Assignment, Rider Location, Delivery Batch, Delivery Stop | No |
| Notification | Notification | No |
| Support | Support Ticket, Support Ticket Comment | No |
| Analytics | Report Rollup, Analytics Snapshot | No (but rebuildable — see 4.13) |
| Audit | Audit Log | No |
| Settings | Settings | No |

