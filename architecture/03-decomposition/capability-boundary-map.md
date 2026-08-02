# Capability Boundary Map — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. |
| **Stability** | Stable once written — this is the load-bearing translation layer between business capability and system structure; changes here ripple everywhere and must be ADR-backed (ADR-0009, ADR-0020). |
| **Authority** | Subordinate to `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `decisions/README.md` (especially ADR-0009, ADR-0020, ADR-0021), `02-context/system-context.md`, and `03-decomposition/module-catalog.md`. This document does not redesign, merge, split, or add to the sixteen-module catalog those documents already settle — it explains the business boundary each module already has, in business terms. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** `module-catalog.md` answers "what does this module do, technically, and what is it allowed to call." This document answers a narrower but easily-confused question: **what business capability does each module own, and what business decisions must never be made anywhere else?** It exists so that a developer or an AI assistant deciding *where a new piece of business logic belongs* has an unambiguous answer in business terms — not just a technical dependency table, but a statement of which module is accountable, in the business, for a given kind of decision.

**Scope.** For each of the sixteen backend modules: its business capability, responsibilities and explicit non-responsibilities stated in business language, the business processes it owns end to end, the business events it owns, its public interfaces and upstream/downstream dependencies (summarized, not re-derived — `module-catalog.md` and `module-communication.md` are authoritative for the technical detail), how it collaborates with other modules in real business workflows, what it is explicitly forbidden from ever deciding, and where its capability is expected to grow. Also: the principles behind why capabilities are separated at all, worked examples of correct and incorrect interactions, and guidance for adding a capability in the future. This document does **not** redesign the architecture, introduce a module, merge or split one, or assign a responsibility no prior document already assigns — per this task's own instruction, anything that looked like a gap or an inconsistency during authoring is recorded in Section 8 (Open Decisions), not resolved by invention.

**Intended audience.** Any developer or AI coding assistant about to write business logic and unsure which module it belongs in; the project owner, reviewing whether a proposed feature respects existing boundaries or genuinely needs a new one (Section 7).

**Cross-references.** Built entirely on `module-catalog.md` (Section 4, all sixteen entries) and `module-communication.md` (Section 7's dependency layers), both authoritative for anything this document summarizes. Grounded in Principle 4.4 (architecture follows business capabilities), Principle 4.5 (modules own their own data), ADR-0009 (module ownership, no cross-module repository access), and ADR-0020 (the sixteen-module settlement, including Support and the Auth/User split). Provider-specific facts (Moyasar, Unifonic) are drawn from ADR-0022 and ADR-0023 respectively, referenced only where they clarify a capability's real-world boundary.

---

## 2. Capability Boundary Principles

1. **One capability, one accountable module.** Every business capability in this system has exactly one module accountable for it — never two modules jointly responsible for the same kind of decision, and never a capability with no owner (Principle 4.5, ADR-0009).
2. **A capability boundary is a decision boundary, not just a data boundary.** `module-catalog.md`'s "Ownership" field says which tables a module writes to; this document's "Business Capability" and "Forbidden Responsibilities" fields say which *decisions* a module is allowed to make. The two are related but not identical — a module can be forbidden from a decision even where it technically could compute the answer, because deciding is a different capability's job (Section 6's violations are almost all decision-boundary violations, not data-boundary ones).
3. **Boundaries follow the business, not the codebase's convenience.** A module's boundary matches how the SRD and DDD describe the business (Order handles ordering, Payment handles settlement, Delivery handles fulfillment) — never a technical grouping ("everything that touches money" is not one module; Payment and Promotion are deliberately separate because pricing rules and settlement are different business concerns that happen to both involve currency).
4. **Boundaries are stable; capability *detail* evolves.** The sixteen-module list (ADR-0020) is not expected to change casually — a new capability is added only through the process in Section 7, never by quietly expanding an existing module's job.
5. **A capability's boundary must survive its own extraction.** Every boundary in this document is drawn so that the module behind it could become an independently deployed service without changing what it decides or owns (ADR-0002/0009; `01-Architecture-Design-Specification.md` Section 15) — a boundary that only works because everything is in one process is not a real capability boundary, it's a shortcut.

## 3. Why Business Capabilities Are Separated

- **Accountability.** When a defect or a dispute traces back to "why was this order allowed to ship without payment confirmation," there is exactly one place to look (Order's checkout orchestration, Payment's confirmation state) — not a search across every module that happened to touch the order.
- **Independent change.** A change to how promotions are evaluated (Promotion) should never require touching how orders are placed (Order), how stock is reserved (Inventory), or how deliveries are assigned (Delivery) — each capability evolves on its own schedule because nothing outside it depends on its internals, only its public interface.
- **Consistent business behavior across every client.** Four different client applications reach the same sixteen capabilities; a capability boundary is what guarantees a promotion rule, a refund rule, or an authorization check behaves identically regardless of which client triggered it (Principle 4.6) — scattering business logic across modules (or into controllers) is exactly what would make that guarantee impossible to keep.
- **Cognitive load, for a solo developer and a memoryless AI assistant.** Sixteen bounded capabilities, each answerable from one document section, is tractable in a way "figure out where this logic lives by reading everything" is not — this is the direct, practical payoff of Principle 4.4 applied at the scale this project actually operates at.
- **A capability boundary drawn around the business, not the database, survives the business changing its mind about implementation** — a capability can move from PostgreSQL tables to a different persistence mechanism, or from in-process to its own service, without the business meaning of "what Payment is accountable for" changing at all.

## 4. The Capability Catalog

Each entry below summarizes `module-catalog.md`'s technical detail (Public Interfaces, Upstream/Downstream Dependencies) and adds the business-capability framing that document does not carry. Where a field says "see `module-catalog.md` §4.N," that section is authoritative for the exhaustive list; this document's own text is the business rationale, not a competing source of truth.

### 4.1 Auth — Identity & Session Authentication

**Purpose:** Establishes and proves who is acting, for every actor type, before any other capability is allowed to act on their behalf.

**Business Capability:** Identity & Session Authentication.

**Responsibilities:** Verifying a claimed identity via OTP; issuing and validating session credentials; managing session lifetime and revocation.

**Explicit Non-Responsibilities:** Does not decide what an authenticated actor is allowed to *do* (that's the RBAC/permission model, evaluated centrally but not owned by Auth as a business capability — `04-cross-cutting/security-and-compliance.md`). Does not hold a person's name, address, or work schedule (User's capability). Does not send SMS itself (Notification's capability, via Unifonic — ADR-0023).

**Owned Business Processes:** Account Login (OTP request/verify), Session Renewal, Session/Account Revocation, Step-Up Authentication (ADR-0040, for the four privileged roles before a high-risk operation), Store Scope Assignment (ADR-0041, which stores an operational user is authorized to act against).

**Business Events it Owns:** A login was completed (`UserAuthenticated`); an OTP was requested (`OtpRequested`); a session was revoked (`SessionRevoked`); a step-up challenge was completed or failed (`StepUpAuthenticationCompleted`, `StepUpAuthenticationFailed` — ADR-0040).

**Public Interfaces:** See `module-catalog.md` §4.1 — RequestOtp, VerifyOtp, IssueTokenPair, RefreshAccessToken, RevokeSession, ValidateToken.

**Upstream Dependencies:** Notification (to dispatch the OTP).

**Downstream Dependencies:** User (to resolve whose profile a session belongs to); every other module, indirectly, for token validation on every request — a cross-cutting dependency `module-catalog.md` notes explicitly rather than listing per module.

**Collaboration with Other Modules:** Auth is the first capability every request touches; every other module trusts Auth's identity determination completely and never re-derives it independently.

**Forbidden Responsibilities:** Deciding what a role or permission allows (that decision belongs to the centrally-evaluated RBAC model, not to Auth as a capability owner — Auth issues the claims, it does not interpret them for authorization purposes). Storing or reasoning about a customer's or worker's profile data.

**Future Expansion Opportunities:** A future passkey or app-based authenticator would extend this capability without changing its boundary — it still only ever answers "who is this and are they still who they claim to be" (`module-catalog.md` §4.1's extraction candidacy).

---

### 4.2 User — Customer & Workforce Profile Management

**Purpose:** Owns who a person *is*, beyond the bare fact that they're authenticated — their profile, addresses, and (for workers) capability, shift, and availability.

**Business Capability:** Customer & Workforce Profile Management.

**Responsibilities:** Maintaining customer profile and saved-address data; maintaining worker profile, capability, shift, and availability data (ADR-0020).

**Explicit Non-Responsibilities:** Does not authenticate anyone. Does not decide whether a specific worker is *eligible for a specific task right now* — that's a Delivery-capability decision informed by, but not made by, User's exposed capability/shift data.

**Owned Business Processes:** Customer Profile Maintenance, Address Book Management, Worker Availability & Shift Status Maintenance.

**Business Events it Owns:** A profile changed (`CustomerProfileUpdated`); an address was added (`AddressAdded`); a worker's availability changed (`WorkerAvailabilityChanged`).

**Public Interfaces:** See `module-catalog.md` §4.2 — GetCustomerProfile, UpdateCustomerProfile, AddAddress, GetDefaultAddress, GetWorkerProfile, GetWorkerCapabilities, GetCurrentShift, UpdateWorkerAvailability.

**Upstream Dependencies:** Auth (to resolve the identity a profile belongs to).

**Downstream Dependencies:** Cart, Order, Delivery, Promotion, Support (each reads profile/worker data as context for their own capability).

**Collaboration with Other Modules:** User is a reference capability other modules read from but never write to — Delivery reads worker eligibility signals; Order reads customer/delivery-address context; none of them ever hold their own copy of profile truth.

**Forbidden Responsibilities:** Deciding whether a worker is assigned to a specific delivery or picking task (Delivery's capability). Deciding order eligibility, cart contents, or any transactional business outcome.

**Future Expansion Opportunities:** A dedicated Workforce capability could re-emerge if worker-profile concerns grow substantially distinct from customer-profile ones — only through a future ADR superseding ADR-0020, not an incremental change here.

---

### 4.3 Store — Store & Service Area Management

**Purpose:** Defines the operating units — dark stores and the geographic areas they serve — that every store-scoped capability in the system anchors itself to.

**Business Capability:** Store & Service Area Management.

**Responsibilities:** Maintaining store definitions and their service zones; resolving which store serves a given address.

**Explicit Non-Responsibilities:** Does not track what's in stock at a store (Inventory's capability). Does not know who works at a store or what orders are active there — Store is the reference point, not the owner of operational activity within it.

**Owned Business Processes:** Store Activation/Deactivation, Service Zone Definition & Maintenance.

**Business Events it Owns:** A store was activated or deactivated (`StoreActivated`, `StoreDeactivated`); a service zone changed (`ServiceZoneChanged`).

**Public Interfaces:** See `module-catalog.md` §4.3 — GetStore, ListActiveStores, ResolveServingStoreForAddress, GetServiceZone.

**Upstream Dependencies:** None — one of the system's most foundational capabilities.

**Downstream Dependencies:** Catalog, Inventory, Order, Delivery — every store-scoped capability resolves its store context through Store rather than duplicating store data.

**Collaboration with Other Modules:** Store is consulted, never directed — other capabilities ask "which store serves this address" or "is this store active," and Store never initiates a call into any other capability.

**Forbidden Responsibilities:** Deciding stock levels, worker assignment, or order routing beyond resolving which store an address belongs to.

**Future Expansion Opportunities:** Multi-country expansion (DDD §17.1) would add store attributes (currency, locale, regulatory region) within this same capability, not a new one.

---

### 4.4 Catalog — Product Catalog Management

**Purpose:** Defines what is sellable — the product, category, and brand data every commerce capability references but never owns a copy of.

**Business Capability:** Product Catalog Management.

**Responsibilities:** Maintaining product, category, and brand definitions and imagery.

**Explicit Non-Responsibilities:** Does not track stock levels (Inventory's capability). Does not set or calculate price — Catalog holds a product's *definition*, not what it costs today or what discount applies (Promotion's and Order's capabilities respectively). Does not maintain a search index (Search's derived capability, built from Catalog's own published events).

**Owned Business Processes:** Product Lifecycle Management (create, update, deactivate); Category & Brand Maintenance.

**Business Events it Owns:** A product was created, updated, or deactivated (`ProductCreated`, `ProductUpdated`, `ProductDeactivated`); a category changed (`CategoryChanged`).

**Public Interfaces:** See `module-catalog.md` §4.4 — GetProduct, ListProductsByCategory, GetCategory, GetBrand, ListProductImages.

**Upstream Dependencies:** Store (to scope catalog visibility where store-specific differences exist).

**Downstream Dependencies:** Search (via consumed events), Inventory, Cart, Promotion — every capability that needs to know what a product *is* reads from Catalog.

**Collaboration with Other Modules:** Catalog is purely upstream — it never calls into Cart, Order, Inventory, or Promotion; they call into it, or react to its published events.

**Forbidden Responsibilities:** Calculating a price, applying a discount, or reporting stock availability — each belongs to a different, specifically accountable capability.

**Future Expansion Opportunities:** Marketplace expansion (DDD §17.3, multi-seller catalog ownership) would be a significant boundary change requiring its own ADR, not an incremental extension of this capability.

---

### 4.5 Search — Product Discovery

**Purpose:** Makes the catalog findable — relevance-ranked, filtered, sorted discovery — without ever becoming a second source of truth for what a product is.

**Business Capability:** Product Discovery.

**Responsibilities:** Serving search and discovery queries over product data.

**Explicit Non-Responsibilities:** Owns no primary business fact about a product — if Search's index and Catalog ever disagree, Catalog is right (Principle 4.5's reporting/read-model exception, applied here to discovery rather than reporting).

**Owned Business Processes:** Search Index Maintenance (a supporting process, not a customer-facing business process in its own right — the customer-facing process is "finding a product," which this capability serves but does not itself constitute a business record of anything).

**Business Events it Owns:** None — Search is a pure consumer/read-path capability (`module-catalog.md` §4.5).

**Public Interfaces:** See `module-catalog.md` §4.5 — SearchProducts, SuggestSearchTerms.

**Upstream Dependencies:** Catalog (via consumed events); Inventory (optionally, event-driven, for availability-aware results).

**Downstream Dependencies:** None within the backend — Search is called directly by the client-facing API layer, not by another module.

**Collaboration with Other Modules:** Search never initiates a call into Catalog or Inventory — it reacts to their published events and stays current asynchronously, which is precisely what keeps a slow or degraded Search from ever being able to slow down Catalog or Inventory in return.

**Forbidden Responsibilities:** Ever being treated as authoritative for a product's existence, price, or stock — those questions always go back to Catalog, Promotion/Order, and Inventory respectively.

**Future Expansion Opportunities:** A dedicated search technology behind Search's own interface, should evidence justify one, changes nothing about any other capability's boundary (`module-catalog.md` §4.5; accepted as a capability by ADR-0020).

---

### 4.6 Inventory — Inventory & Stock Management

**Purpose:** Is the single, authoritative answer to "how much of this product is actually available at this store right now," and protects that answer under concurrent demand.

**Business Capability:** Inventory & Stock Management.

**Responsibilities:** Tracking stock levels per store; reserving stock against in-flight orders; recording every stock movement and its reason.

**Explicit Non-Responsibilities:** Does not decide *whether* or *when* to reserve stock for an order — that decision belongs to Order's checkout orchestration; Inventory only executes the reservation operation once asked.

**Owned Business Processes:** Stock Level Maintenance, Inventory Reservation, Inventory Adjustment & Reconciliation.

**Business Events it Owns:** Stock was reserved, or reservation failed or was released (`InventoryReserved`, `InventoryReservationFailed`, `InventoryReservationReleased`); a stock level changed (`InventoryLevelChanged`); a ledger entry was recorded (`InventoryLedgerRecorded`).

**Public Interfaces:** See `module-catalog.md` §4.6 — GetAvailability, ReserveStock, ReleaseReservation, ConfirmReservationConsumption, RecordLedgerAdjustment.

**Upstream Dependencies:** Store, Catalog.

**Downstream Dependencies:** Order (the primary caller, synchronously, during checkout); Search (event-driven, optional, for availability-aware discovery).

**Collaboration with Other Modules:** Inventory is called *by* Order, never the reverse — it has no awareness of why a reservation was requested, only what was requested, which is what lets it stay callable without knowing anything about orders, carts, or payments.

**Forbidden Responsibilities:** Deciding to place, cancel, or modify an order. Initiating a reservation on its own judgment rather than in response to an explicit request.

**Future Expansion Opportunities:** A strong future extraction candidate (`module-catalog.md` §4.6) precisely because reservation locking is the system's most concurrency-sensitive capability — extraction would not change what Inventory is accountable for, only where it runs.

---

### 4.7 Cart — Shopping Cart Management

**Purpose:** Holds a customer's in-progress selection before it becomes a real business commitment.

**Business Capability:** Shopping Cart Management.

**Responsibilities:** Maintaining the customer's pre-checkout item selection.

**Explicit Non-Responsibilities:** Does not reserve inventory — an item sitting in a cart reserves nothing. Does not compute final, order-time pricing — that's Order's job at checkout, using live Catalog and Promotion data rather than the cart's own possibly-stale view.

**Owned Business Processes:** Cart Building (add/remove/update items, clear cart).

**Business Events it Owns:** The cart changed (`CartUpdated`) — optional, for analytics/abandonment tracking, never required for correctness elsewhere.

**Public Interfaces:** See `module-catalog.md` §4.7 — GetCart, AddCartItem, RemoveCartItem, UpdateCartItemQuantity, ClearCart.

**Upstream Dependencies:** Catalog (to validate items still exist and are active), User (to scope the cart to a customer).

**Downstream Dependencies:** Order (reads the cart at checkout time).

**Collaboration with Other Modules:** Cart is deliberately isolated from every capability that only matters once checkout begins — it is the one capability in the system explicitly designed to be pre-transactional.

**Forbidden Responsibilities:** Reserving stock, applying a promotion result, or creating an order — a cart is a draft, never itself a business commitment.

**Future Expansion Opportunities:** None anticipated — Cart is expected to remain simple relative to the rest of the system by design (`module-catalog.md` §4.7).

---

### 4.8 Order — Order Management & Checkout Orchestration

**Purpose:** Turns a customer's cart into a committed business transaction, and is accountable for the correctness of that transition end to end.

**Business Capability:** Order Management & Checkout Orchestration.

**Responsibilities:** Owning the order record and its lifecycle; orchestrating checkout — validating the cart, reserving inventory, initiating payment, applying promotions, and recording the outcome; owning order-time price snapshots (ADR-0020).

**Explicit Non-Responsibilities:** Does not reserve inventory itself, process payment itself, or assign a delivery itself — it calls the capability accountable for each and reacts to the result. Order *orchestrates* checkout; it does not *own* every capability checkout touches.

**Owned Business Processes:** Checkout (Place Order), Order Cancellation, Order Status Progression.

**Business Events it Owns:** An order was placed, cancelled, changed status, or became ready for fulfillment (`OrderPlaced`, `OrderCancelled`, `OrderStatusChanged`, `OrderReadyForFulfillment`).

**Public Interfaces:** See `module-catalog.md` §4.8 — PlaceOrder, GetOrder, ListOrdersForCustomer, CancelOrder, GetOrderEvents, RecordOrderEvent.

**Upstream Dependencies:** Cart, Inventory, Payment, Promotion, User, Store.

**Downstream Dependencies:** Payment (Order is the only caller of `InitiatePayment`), Delivery (acts once Order signals fulfillment-readiness), Support (references order context for tickets).

**Collaboration with Other Modules:** Order is the system's primary orchestrator — per ADR-0021, each step it takes against another capability is that capability's own module-local transaction, with Order performing an explicit compensating action (such as cancelling a pending order) if a later step fails, rather than any single transaction spanning capabilities. Per ADR-0035/ADR-0036, Order also acts as the forwarding point for two Delivery-originated facts it does not itself own: a delivery-collected payment (forwarded to Payment's interface) and a failed delivery (advancing Order's own state machine) — in both cases Order reacts to a Delivery event and acts through the owning capability's interface, never by writing to Delivery's or Payment's data directly.

**Forbidden Responsibilities:** Deciding stock availability, executing a payment, assigning a rider, or resolving a support dispute — Order coordinates these outcomes, it does not perform them.

**Future Expansion Opportunities:** Named alongside Inventory as the most plausible first extraction candidate (`01-Architecture-Design-Specification.md` §13) — because it already only ever reaches other capabilities through their public interfaces and events, extraction would preserve its orchestration logic unchanged.

---

### 4.9 Payment — Payment Settlement & Refunds

**Purpose:** Is accountable for the financial truth of every order — what was charged, collected, or refunded, and through which channel.

**Business Capability:** Payment Settlement & Refunds.

**Responsibilities:** Settlement across mada, card, Apple Pay, and BNPL via Moyasar (ADR-0022), plus cash-on-delivery and card-on-delivery collection; refund processing (ADR-0020).

**Explicit Non-Responsibilities:** Does not decide whether an order is *eligible* for a refund on business grounds. Per ADR-0033, Order owns automatic refund-eligibility rules from order state, cancellation reason, delivery failure, and substitution outcome; Support may initiate a refund request with a reason; Payment executes and records a refund once eligibility is approved by Order or an authorized Support approval flow. Does not initiate itself — it acts only when Order calls it as a checkout step.

**Owned Business Processes:** Payment Authorization & Capture, Cash/Card-on-Delivery Collection & Reconciliation, Refund Processing, Financial Ledger Recording (ADR-0037 — every financial movement, including delivery-collected payments and line-item refunds, produces an immutable ledger entry).

**Business Events it Owns:** A payment was authorized, captured, or failed (`PaymentAuthorized`, `PaymentCaptured`, `PaymentFailed`); a refund was issued or failed (`RefundIssued`, `RefundFailed`); cash was remitted (`CashRemittanceRecorded`); a ledger entry was recorded (`PaymentLedgerRecorded`, ADR-0037).

**Public Interfaces:** See `module-catalog.md` §4.9 — InitiatePayment, ConfirmPayment, RecordCashCollection, RecordCardOnDeliveryCollection, IssueRefund, GetPaymentStatus.

**Upstream Dependencies:** Order (the sole source of the order/amount context a payment or refund applies to).

**Downstream Dependencies:** Support (may trigger `IssueRefund` on a customer's behalf during a support workflow).

**Collaboration with Other Modules:** Payment is the Backend's sole caller of the external Moyasar gateway (ADR-0022, `02-context/system-context.md`) — no client, and no other module, ever calls Moyasar directly or holds its credentials. Per ADR-0035, Delivery never calls Payment even for a delivery-collected cash/card-on-delivery payment — Order consumes Delivery's `DeliveryCompletedWithPayment` event and forwards the collection to Payment's existing interface, so Payment's own upstream caller set (Order, Support) is unchanged.

**Forbidden Responsibilities:** Deciding whether a refund is warranted as a business matter, changing order state directly, or being called by any capability other than Order or Support.

**Future Expansion Opportunities:** Per ADR-0032/ADR-0033, MVP launch uses Payment-owned manual POS/card-on-delivery settlement reconciliation; a terminal/POS provider settlement API integration (`module-catalog.md` §4.9) extends this same capability once the terminal provider is confirmed — an enhancement, not a new capability.

---

### 4.10 Promotion — Promotions & Discount Management

**Purpose:** Owns the rules that determine what a customer is eligible to pay less for, applied identically regardless of which client placed the order.

**Business Capability:** Promotions & Discount Management.

**Responsibilities:** Maintaining promotion and promo-code rules; evaluating eligibility and computing discounts; recording usage.

**Explicit Non-Responsibilities:** Does not apply a promotion to an order itself — Order calls Promotion during checkout and applies the result. Does not own the frozen, order-time price record (Order's Price Snapshot, per ADR-0020) — Promotion owns the *rule and its usage history*, never the *frozen outcome* of applying it.

**Owned Business Processes:** Promotion Eligibility Evaluation, Promo Code Redemption, Promotion Usage Tracking.

**Business Events it Owns:** Promotion usage was recorded (`PromotionUsageRecorded`).

**Public Interfaces:** See `module-catalog.md` §4.10 — EvaluatePromotionEligibility, ApplyPromoCode, RecordPromotionUsage, GetActivePromotions.

**Upstream Dependencies:** Catalog (product/category-scoped rules), User (customer-scoped eligibility).

**Downstream Dependencies:** Order (the sole caller during checkout).

**Collaboration with Other Modules:** Promotion is a rules-and-eligibility service called by Order — it never calls back into Order or any other checkout capability, keeping discount logic reusable across every current and future path that needs it.

**Forbidden Responsibilities:** Creating, cancelling, or modifying an order; deciding final order-time price on its own authority (it computes a discount; Order records the resulting price).

**Future Expansion Opportunities:** Advanced promotions and dynamic pricing (DDD §17.8–17.9) are additional rule complexity within this capability's existing boundary, not a boundary change.

---

### 4.11 Delivery — Fulfillment (Picking & Delivery)

**Purpose:** Moves a confirmed order from a store shelf into a customer's hands, owning both the in-store and the last-mile halves of that journey.

**Business Capability:** Fulfillment (Picking & Delivery).

**Responsibilities:** In-store picking (ADR-0020); rider assignment and delivery execution; rider location tracking during active delivery; grouping up to two (MVP) eligible, same-store deliveries into one rider trip — delivery batching (ADR-0034), a purely operational optimization within this same capability, not a new one.

**Explicit Non-Responsibilities:** Does not own the order itself — it acts once Order signals fulfillment-readiness and reports back via events, never by writing to Order's own records. Does not own worker profile or shift data — it reads eligibility signals from User's capability rather than duplicating them. Batching does not change any of this: a Delivery Batch groups Delivery's own Delivery Assignments only — it is never a second owner of order, payment, or inventory state, and grants Delivery no new authority over anything outside its existing boundary. Does not decide batching eligibility from hardcoded values — thresholds (proximity, readiness-time tolerance, capacity, SLA headroom) are Settings-owned configuration (ADR-0034), evaluated by Delivery, not invented ad hoc.

**Owned Business Processes:** Picking, Rider Assignment, Delivery Execution, Delivery Batching (ADR-0034).

**Business Events it Owns:** Picking started or completed (`PickingStarted`, `PickingCompleted`); a rider was assigned, completed, or failed to complete a delivery, or rejected an offered one (`DeliveryAssigned`, `DeliveryCompleted`, `DeliveryFailed`, `DeliveryRejected` — the last two per ADR-0036); a delivery completed with a cash/card-on-delivery collection (`DeliveryCompletedWithPayment`, ADR-0035 — a fact Delivery announces but never records financially itself); a rider's location updated (`RiderLocationUpdated`); a batch was created, assigned, re-routed, completed, or cancelled (`DeliveryBatchCreated`, `DriverAssignedToBatch`, `BatchRouteUpdated`, `BatchCompleted`, `BatchCancelled` — ADR-0034).

**Public Interfaces:** See `module-catalog.md` §4.11 — StartPickingSession, CompletePickingSession, AssignRider, RecordRiderLocation, CompleteDelivery, GetActiveAssignmentForRider, EvaluateBatchEligibility, CreateDeliveryBatch, AssignBatchToRider, RecalculateBatchRoute, GetActiveBatchForRider.

**Upstream Dependencies:** Order (fulfillment-readiness signal and completion reporting), User (eligible pickers/riders), Store (scope).

**Downstream Dependencies:** Support (references delivery-assignment context for tickets).

**Collaboration with Other Modules:** Delivery never reaches into Order's own tables to update status — it reports outcomes as events (`PickingCompleted`, `DeliveryCompleted`) that Order consumes and reacts to on its own terms.

**Forbidden Responsibilities:** Deciding order cancellation, payment collection eligibility, or anything about catalog, promotion, or search — fulfillment is a physical-execution capability, not a commerce-decision one.

**Future Expansion Opportunities:** Batch delivery (DDD §17.4) is the named growth driver within this same capability boundary.

---

### 4.12 Notification — Customer & Worker Communication

**Purpose:** Delivers every outbound message the business needs to send, and is the only capability that talks to Unifonic or the push notification channel.

**Business Capability:** Customer & Worker Communication.

**Responsibilities:** Dispatching SMS (via Unifonic, ADR-0023) and push notifications; tracking delivery status.

**Explicit Non-Responsibilities:** Does not decide *when* a notification is warranted — that judgment is expressed entirely by which events exist and what they mean; Notification's job is reliable delivery, never business judgment about what's notification-worthy. Business state never depends on a notification actually being delivered.

**Owned Business Processes:** Notification Dispatch & Delivery Tracking.

**Business Events it Owns:** A notification was sent, failed, or its delivery confirmed (`NotificationSent`, `NotificationFailed`, `NotificationDeliveryConfirmed`).

**Public Interfaces:** See `module-catalog.md` §4.12 — SendNotification, GetNotificationHistory, GetDeliveryStatus.

**Upstream Dependencies:** None for its core job (event-driven); Auth calls it directly and synchronously only for OTP dispatch confirmation.

**Downstream Dependencies:** None — nothing in the system waits on Notification to complete anything.

**Collaboration with Other Modules:** Notification is the system's clearest example of a pure, decoupled consumer capability — it reacts to what nearly every other capability publishes, and nothing reacts to it in return.

**Forbidden Responsibilities:** Ever being a required step for a business operation (order placement, payment, delivery) to complete successfully.

**Future Expansion Opportunities:** Browser push and email (both named as future channels) extend this capability's channel set without changing its boundary.

---

### 4.13 Support — Customer & Operational Support

**Purpose:** Is the operational surface through which staff investigate and resolve customer or internal issues, without taking ownership of whatever business record the issue is about.

**Business Capability:** Customer & Operational Support.

**Responsibilities:** Owning support tickets and their comment history across customer, order, payment, refund, delivery, and worker-related issues (ADR-0020).

**Explicit Non-Responsibilities:** Does not modify Order, Payment, Delivery, Inventory, or User records directly — a support action that changes business state (a refund, a cancellation, a reassignment, a profile correction) calls the owning capability's interface and is authorized/audited exactly like any other administrative action. Support owns the ticket and its conversation, never the underlying business record it discusses.

**Owned Business Processes:** Support Ticket Lifecycle (open, assign, comment, resolve, reopen).

**Business Events it Owns:** A ticket was opened, assigned, changed status, commented on, closed, or reopened (`SupportTicketOpened`, `SupportTicketAssigned`, `SupportTicketStatusChanged`, `SupportTicketCommentAdded`, `SupportTicketClosed`, `SupportTicketReopened`).

**Public Interfaces:** See `module-catalog.md` §4.13 — OpenTicket, AddTicketComment, AssignTicket, UpdateTicketStatus, LinkTicketToEntity, GetTicket, ListTicketsForCustomer, ListTicketsForOrder.

**Upstream Dependencies:** User, Order, Payment, Delivery (each for read-only context relevant to a ticket).

**Downstream Dependencies:** None — no other capability depends on Support to complete its own work, consistent with support being a response to a problem, never a precondition for normal operation.

**Collaboration with Other Modules:** Support consumes completion/failure events (`OrderCancelled`, `PaymentFailed`, `RefundIssued`, `DeliveryCompleted`, `DeliveryFailed`) to automatically link or update ticket context, and calls into Payment (for a refund) or another capability's interface exactly as any other administrative caller would — never through a shortcut unavailable to other callers.

**Forbidden Responsibilities:** Directly writing to any entity it doesn't own — an order status, a payment record, a delivery assignment, or a profile field are all changed only through their owning capability's interface, with the same authorization and audit requirements as any other administrative change.

**Future Expansion Opportunities:** A richer CRM/helpdesk integration could sit behind this capability's boundary as an external integration Support owns — never a reason for Order, Payment, or User to take on ticket ownership themselves.

---

### 4.14 Analytics — Reporting & Business Analytics

**Purpose:** Turns the rest of the system's activity into reporting the business can act on, without ever becoming a shortcut into anyone else's live data.

**Business Capability:** Reporting & Business Analytics.

**Responsibilities:** Building and serving rebuildable reporting rollups and analytics snapshots.

**Explicit Non-Responsibilities:** Never a source of truth for anything it reports on — if a rollup and the owning capability's live state disagree, the owning capability is always right.

**Owned Business Processes:** Rollup & Snapshot Generation.

**Business Events it Owns:** None — Analytics is a pure read-side consumer.

**Public Interfaces:** See `module-catalog.md` §4.14 — GetReportRollup, GetAnalyticsSnapshot, TriggerRollupGeneration.

**Upstream Dependencies:** None synchronously — built entirely from consumed events and scheduled generation.

**Downstream Dependencies:** None — nothing in the system waits on Analytics for anything.

**Collaboration with Other Modules:** Analytics listens broadly (`OrderPlaced`, `OrderCancelled`, `PaymentCaptured`, `RefundIssued`, `DeliveryCompleted`, and more as reporting needs grow) and never calls into any capability it reports on.

**Forbidden Responsibilities:** Sitting on any business operation's critical path — nothing should ever wait on Analytics to respond for a checkout, payment, or delivery action to complete.

**Future Expansion Opportunities:** A dedicated analytical datastore behind this same capability's interface, entirely decoupled from PostgreSQL's system-of-record role, requires no change to any other capability.

---

### 4.15 Audit — Compliance & Audit Trail

**Purpose:** Is the durable, trustworthy record of who did what and when, for every action across the business with financial, inventory, or lifecycle consequence.

**Business Capability:** Compliance & Audit Trail.

**Responsibilities:** Recording and serving the append-only audit log.

**Explicit Non-Responsibilities:** Never interprets or judges the actions it records — it is a faithful recorder, not a business-rule enforcer, and never blocks a business operation from completing.

**Owned Business Processes:** Audit Recording.

**Business Events it Owns:** None.

**Public Interfaces:** See `module-catalog.md` §4.15 — RecordAuditEntry, GetAuditTrailForEntity, GetAuditTrailForActor.

**Upstream Dependencies:** None.

**Downstream Dependencies:** None formally declared, though — like Auth's token validation — Audit is called across essentially every capability with a consequential action to record; `module-catalog.md` notes this as cross-cutting rather than enumerating it per module (see Section 8).

**Collaboration with Other Modules:** Audit is called, never a caller — every other capability's consequential action reaches it, and it reaches nothing in return.

**Forbidden Responsibilities:** Ever being a precondition for a business operation to succeed — an audit-recording failure is itself an incident to surface, never a reason to fail the action it was recording.

**Future Expansion Opportunities:** None beyond growth in volume, handled by partitioning/archival rather than a boundary change.

---

### 4.16 Settings — Platform Configuration Management

**Purpose:** Owns the platform's own operational configuration — the knobs that govern cross-cutting behavior but don't belong to any single business capability.

**Business Capability:** Platform Configuration Management.

**Responsibilities:** Maintaining global and store-scoped configuration values.

**Explicit Non-Responsibilities:** Does not own business rules themselves — a promotion rule lives in Promotion, a delivery radius lives in Store/Service Zone; Settings owns configuration that doesn't belong to any one business capability.

**Owned Business Processes:** Configuration Change Management.

**Business Events it Owns:** A setting changed (`SettingChanged`).

**Public Interfaces:** See `module-catalog.md` §4.16 — GetSetting, UpdateSetting, ListSettingsForScope.

**Upstream Dependencies:** None.

**Downstream Dependencies:** Per ADR-0033, Auth, Notification, Order, Payment, Delivery, Inventory, Promotion, Store, and Analytics read configuration from Settings (`module-catalog.md` §4.16's Readers field); Settings itself depends on none of them.

**Collaboration with Other Modules:** Settings is queried, never directed — it has no awareness of who reads a given configuration value or why.

**Forbidden Responsibilities:** Holding a business rule that has its own accountable capability elsewhere (pricing, promotion eligibility, delivery radius) — if a value has a natural business owner, it belongs there, not in generic Settings.

**Future Expansion Opportunities:** A versioning/rollback model for configuration changes, as the number of configurable behaviors grows — an internal evolution, not a boundary change.

---

## 5. Examples of Correct Interactions

- **Checkout.** Order calls Inventory's `ReserveStock` and Payment's `InitiatePayment` as explicit, synchronous steps it orchestrates — each capability decides only what it's accountable for (does the stock exist, can the payment be authorized), and Order alone decides what the combined outcome means for the order record. This is correct because no capability had to know about another's internals to do its job.
- **Search staying current.** Search never asks Catalog "has anything changed" — it consumes `ProductCreated`/`ProductUpdated`/`ProductDeactivated` and updates its own index. This is correct because Catalog's capability boundary (owning product truth) is never compromised by Search's need to stay fresh.
- **A support-initiated refund.** A Support agent, investigating a ticket, triggers a refund by calling Payment's `IssueRefund` — the same interface any other authorized caller would use — rather than Support writing directly to a payment record. This is correct because Support's non-responsibility for payment data is respected even while Support is the one *initiating* the business action.
- **Promotion evaluated but not applied by Promotion itself.** Order calls `EvaluatePromotionEligibility` and applies the result to the order it's building; Promotion never reaches back into Order to apply anything. This is correct because it keeps discount rules reusable across every current and future path that needs a discount computed, without Promotion needing to know what "applying" means in every context that might call it.

## 6. Examples of Architecture Violations

- **Catalog calculating a discounted price.** Pricing and promotion eligibility are Promotion's and Order's capabilities; a product's definition (Catalog) having any awareness of "today's price after discount" blurs a boundary that exists specifically so pricing logic has one home.
- **Inventory deciding to cancel an order because stock ran out.** Inventory's capability is to report and reserve stock accurately — deciding what happens to an order as a consequence is Order's accountability. Inventory publishing `InventoryReservationFailed` and Order deciding to cancel is correct; Inventory cancelling the order itself is not.
- **Support updating an order's status directly.** Even though Support legitimately needs to change order-related outcomes as part of resolving a ticket, doing so by writing to Order's data directly — rather than calling Order's public interface — violates both this document's non-responsibility statement for Support and ADR-0009's ownership rule.
- **Delivery reading a worker's shift data directly from User's tables** instead of through User's public interface, "because it's faster." This is the cross-module repository access anti-pattern (Constitution Section 10) applied to a business-capability boundary specifically — the fact that it's business-adjacent, not a purely technical shortcut, does not make it acceptable.
- **A new "loyalty points" feature added inside Promotion** without going through Section 7's process, on the reasoning that "it's discount-adjacent." Loyalty is a distinct business capability (accrual, redemption, expiry rules distinct from promotion eligibility) and deserves its own boundary decision, not an assumption that an existing capability's boundary quietly stretches to cover it.

## 7. Guidelines for Adding Future Capabilities

A new business capability (and the module that would own it) is added only when all of the following hold:

1. **It is not already covered.** Check every entry in Section 4 — including each "Explicit Non-Responsibilities" and "Forbidden Responsibilities" field — before concluding a capability is missing. A capability that looks new is often an existing one's boundary being misunderstood.
2. **It has exactly one natural accountable owner.** If two existing capabilities could plausibly own it, that is a sign the new capability's own boundary needs more thought before it's added, not a reason to split ownership.
3. **It fits the layered dependency model.** A new capability must have a coherent position in `module-communication.md` §7's layered graph — able to state clearly what it may depend on and what must never depend on it.
4. **It is proposed and recorded as an ADR**, following ADR-0020's own pattern — never added by simply writing code or updating this document unilaterally (Constitution Principle 4.12).
5. **It is named in the business's own vocabulary**, consistent with the SRD and DDD, not a technical grouping invented for implementation convenience.
6. **Its data ownership is exclusive.** Per Principle 4.5, the new capability owns entities no other module already owns — an ADR that would strip ownership from an existing module needs to say so explicitly and update `module-catalog.md`, `data-ownership-map.md`, and this document together.

## 8. Open Decisions

- **Audit and Auth's cross-cutting call pattern is acknowledged but not formally modeled.** Both `module-catalog.md` entries describe being "universally called" without listing every calling module explicitly. This document follows that same treatment rather than inventing a full enumeration neither source document provides.
- **Loyalty, marketplace, and multi-country expansion** (referenced under several modules' Future Expansion Opportunities) are named as plausible future capability boundaries but are explicitly not decided here — each would require its own ADR under Section 7's process before this document is updated to include it.
- Refund eligibility's decision owner and Settings' downstream dependents — previously open here — are resolved by ADR-0033 (Sections above, and `module-catalog.md` §4.9, §4.16) and must not be treated as open going forward.
- The delivery-collected-payment path, failed/rejected delivery events, the Payment Ledger, line-item refunds, OTP abuse protection, step-up authentication, and store-scoped authorization — all identified as gaps by the 2026-07-30 Architecture Readiness Review — are resolved by ADR-0035 through ADR-0041 respectively and must not be treated as open going forward.
