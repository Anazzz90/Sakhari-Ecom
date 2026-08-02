# Service Decomposition — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. |
| **Stability** | Stable in the layering pattern (every module follows the same internal shape); evolving in which specific services exist as modules grow in complexity. Reassigning a module's boundary remains a high-blast-radius, ADR-backed change (ADR-0009, ADR-0020) — this document changes what's *inside* a boundary, never the boundary itself. |
| **Authority** | Subordinate to `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `decisions/README.md` (especially ADR-0002, ADR-0009, ADR-0020, ADR-0021), `03-decomposition/module-catalog.md`, `module-communication.md`, `capability-boundary-map.md`, `data-ownership-map.md`, and `07-coding-standards/coding-standards.md` Section 3. This document introduces no new module, responsibility, dependency, or boundary — it describes the internal service layering already implied by those documents' rules, made explicit. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** `module-catalog.md` established sixteen module boundaries; `module-communication.md` established how those boundaries talk to each other; `data-ownership-map.md` established who owns what data. This document is the last piece: how each module is decomposed *internally* — into public-facing services, application-level orchestration, domain-level business rules, and repositories — while preserving every rule the other three documents already establish. It describes the runtime shape `01-Architecture-Design-Specification.md`'s Section 7 (High-Level Architecture) and `coding-standards.md` Section 3 (Module Structure) already imply, at the granularity a future SDD needs to start from.

**Scope.** Per module: Purpose, Public Services, Internal Services, Application Services, Domain Services, Repositories, Published/Consumed Events, Transaction Boundaries, Dependencies, and Forbidden Dependencies. System-wide: why services exist, how they communicate, how they expose functionality, how they enforce module boundaries, and how future microservice extraction would work; a set of architectural rules; a full service dependency matrix; and worked legal/illegal interaction examples. Per this task's instruction, this document contains **no code** (no class names, no method signatures, no file paths) and **no diagrams** — every service named below is a conceptual label for a layer of responsibility, exactly the way `module-catalog.md`'s "Public Interfaces" are named operations rather than code.

**Intended audience.** The project owner and any AI coding assistant about to write code inside a module, needing to know which conceptual layer a piece of logic belongs in; the author of any future module SDD, for whom this document's per-module breakdown is the internal structure to detail further, not renegotiate.

**Cross-references.** Realizes `coding-standards.md` Section 3's module structure (public interface, service layer, thin controller, data-access code, event publishing/consuming) at the granularity of distinguishing *application* orchestration from *domain* business rules within the "service layer" that document already named. Every Public Service, Dependency, Forbidden Dependency, and Published/Consumed Event below is drawn directly from `module-catalog.md` §4 and `module-communication.md` §7 — this document does not re-derive them, only re-expresses them through the internal-service lens. Transaction Boundaries apply `data-architecture.md` §5 and `module-communication.md` §8 (and ADR-0021) per module.

## 2. Why Services Exist

A module's business logic is never one undifferentiated blob (Principle 4.6) — it is layered so that three different concerns never get tangled into one piece of code:

- **What can be called from outside the module** (Public Services) versus **what exists only to support that** (Internal Services) — this is what keeps a module's internal refactoring from ever becoming a breaking change for its callers, as long as the public layer's behavior is preserved.
- **Coordinating a business operation** (Application Services — calling other modules, sequencing steps, managing the module-local transaction) versus **deciding a business rule** (Domain Services — pure logic, no orchestration, no awareness of other modules) — this split is what makes business rules independently testable and reusable across more than one orchestration path, without a rule's logic being entangled with the mechanics of calling Inventory or Payment.
- **Deciding and doing** (Application/Domain Services) versus **persisting** (Repositories) — a Repository never contains a business decision, and a service never contains a raw query; this is the internal expression of `coding-standards.md` Section 3's "data-access code, reachable only from within the module's own service layer."

Without this layering, "business logic lives only in a module's service layer" (Principle 4.6) would be true only at the module granularity — this document makes it true at every granularity inside a module too.

## 3. How Services Communicate

- **Within a module:** an Application Service may call the module's own Domain Services and Repositories directly (in-process, no boundary to cross) — this is ordinary internal composition, not subject to `module-communication.md`'s cross-module rules at all.
- **Across modules:** only a Public Service may be called, and only by another module's Application Service (never by a Domain Service or a Repository, which have no awareness that other modules exist) — exactly `module-communication.md` Section 5's "internal service interfaces" rule, restated at the layer that actually issues the call.
- **Domain Services never call another module**, directly or indirectly — a Domain Service's entire job is deciding something from the data and rules it's given, which is what keeps business rules portable and independently testable.
- **Repositories never call anything except the database** — no Repository calls a Public Service, an Application Service, or another module's Repository, ever.

## 4. How Services Expose Functionality

Only a module's **Public Services** are reachable from outside the module — by a REST controller translating an external request (`module-communication.md` Section 4), or by another module's Application Service making a direct, in-process, synchronous call (`module-communication.md` Section 5). Internal Services, Application Services (of another module), Domain Services, and Repositories are never directly reachable from outside their own module, under any circumstance — this is not a convention layered on top of the architecture, it is the architecture: `module-catalog.md`'s "Public Interfaces" field *is* the enumeration of each module's Public Services, and nothing not listed there is externally callable.

## 5. How Services Enforce Module Boundaries

Two structural facts make ADR-0009's "no cross-module repository access" rule true by construction rather than by discipline alone:

1. **A Repository is never part of a module's Public Services.** It is only reachable from that module's own Application and Domain Services. There is no path from another module into a Repository that doesn't first pass through a Public Service.
2. **A Public Service is the only layer with awareness that other modules exist.** Domain Services and Repositories are written as if they were the only code in the system — they take the data they're given and either decide something (Domain) or persist something (Repository), with zero knowledge of module boundaries, because they never need to cross one.

This is what `coding-standards.md` Section 2 means by "the structural, on-disk expression of 'no cross-module repository access'" — the layering in this document is that structure made explicit at the service level, not just the folder level.

## 6. How Future Microservice Extraction Would Work

When a module is extracted into its own independently deployed service (ADR-0002/0009's designed reversibility; `01-Architecture-Design-Specification.md` Section 15; `04-cross-cutting/reliability-and-performance.md` Section 10), this document's layering is exactly what makes the extraction mechanical rather than a redesign:

- **Public Services become the extracted service's network-facing API** — the same operations, the same names, the same meaning, now reached over a network call instead of an in-process one (`module-communication.md` Section 11).
- **Internal Services, Application Services, Domain Services, and Repositories move together, unchanged** — none of them ever had awareness of other modules to begin with, so none of them need to change when the module they live inside gets a new deployment boundary.
- **Published/Consumed Events need only a transport change**, not a meaning change (`04-cross-cutting/integration-and-messaging.md` Section 11) — an in-process event dispatcher becomes a message broker, but the event's name, ownership, and payload are untouched.
- **Transaction Boundaries do not change at all** — they were already module-local before extraction (Section 8 below; ADR-0021), so a module becoming its own service doesn't newly discover a transaction it needs to break apart.

This document names no extraction plan or timeline — Section 12 lists candidates already named elsewhere, without committing to when or whether extraction happens.

## 7. Architectural Rules

- **Thin Controllers.** A controller translates a request into a call on a Public Service and the result back into a response — it never calls an Application Service, Domain Service, or Repository directly, and it contains no business decision (Principle 4.7; `coding-standards.md` Section 3).
- **Business Logic Only in Services.** Every business decision lives in an Application or Domain Service — never in a controller, never in a Repository, never in event-handling code that isn't itself calling into the service layer (Principle 4.6).
- **No Repository Sharing.** A Repository is used only by its own module's Application and Domain Services — never imported, called, or referenced by any other module, under any circumstance (ADR-0009; Section 5 above).
- **No Circular Dependencies.** The service dependency matrix (Section 13) is a directed acyclic graph — a Public Service may depend on another module's Public Service only where Section 13 marks it Allowed, and never in a way that would create a cycle (`module-communication.md` Section 9).
- **Public Interfaces Only.** Cross-module calls reach only a Public Service — never an Internal Service, Application Service, Domain Service, or Repository belonging to another module (Section 4 above).
- **Domain Events.** A module's events are published only by its own Application Services, about entities that module owns, after the transaction that made the fact true has committed (ADR-0011; `integration-and-messaging.md` Section 4).
- **Transaction Ownership.** A transaction is opened and committed by an Application Service, scoped to that module's own Repositories only — never spanning two modules' Repositories, and never opened by a Domain Service or a controller (`data-architecture.md` Section 5; ADR-0021).
- **Validation Boundaries.** Transport-shape validation (is this request well-formed) happens at the controller; business-rule validation (is this action allowed) happens in the Application or Domain Service layer — a Repository never validates anything beyond what the database itself enforces structurally (Constitution Section 8; `coding-standards.md` Section 6).
- **Error Propagation.** An error raised in a Domain Service propagates up through the Application Service that called it, which decides how to surface it (a business-rule rejection returned clearly, or an unexpected failure surfaced and never swallowed — `coding-standards.md` Section 7) — a Repository-level failure is never silently caught and hidden by the service layer above it.

## 8. Module Service Catalog

Public Services restate `module-catalog.md`'s Public Interfaces grouped into named service concepts; Dependencies/Forbidden Dependencies/Published/Consumed Events restate that document's fields concisely for continuity — `module-catalog.md` remains authoritative for their exhaustive detail.

### 8.1 Auth

| Field | Detail |
|---|---|
| **Purpose** | Establish and prove identity for every actor and request. |
| **Public Services** | AuthenticationService (RequestOtp, VerifyOtp), SessionService (IssueTokenPair, RefreshAccessToken, RevokeSession, ValidateToken), StepUpAuthenticationService (InitiateStepUpChallenge, VerifyStepUpChallenge — ADR-0040) |
| **Internal Services** | TokenSigningService — signs/verifies tokens on behalf of SessionService, never called directly by anything outside Auth |
| **Application Services** | LoginOrchestrationService — coordinates OTP verification, Notification dispatch, and token issuance as one business operation; StepUpChallengeOrchestrationService (ADR-0040) |
| **Domain Services** | OtpExpiryPolicy, TokenClaimRules, OtpAbuseProtectionRules (ADR-0039 — request/attempt/lockout thresholds), StoreScopeEvaluationRules (ADR-0041 — pure rule with no awareness of any other module, evaluated after permission but before an operation proceeds) — pure rules with no awareness of Notification or any other module |
| **Repositories** | UserCredentialRepository, SessionRepository, StoreScopeAssignmentRepository (ADR-0041) |
| **Published Events** | `UserAuthenticated`, `OtpRequested`, `SessionRevoked`, `StepUpAuthenticationCompleted`, `StepUpAuthenticationFailed` (ADR-0040) |
| **Consumed Events** | None |
| **Transaction Boundaries** | Each login/session operation (OTP verification plus token issuance) commits within one Auth-local transaction, over `users`/`sessions`/`refresh_tokens` only |
| **Dependencies** | Notification (OTP dispatch) |
| **Forbidden Dependencies** | Order, Payment, Cart, Catalog, Inventory, Delivery, Promotion |

### 8.2 User

| Field | Detail |
|---|---|
| **Purpose** | Own customer and worker profile data. |
| **Public Services** | CustomerProfileService, WorkerProfileService |
| **Internal Services** | AddressBookHelperService — supports CustomerProfileService's address operations, not independently exposed |
| **Application Services** | ProfileUpdateOrchestrationService, WorkerAvailabilityUpdateService |
| **Domain Services** | DefaultAddressSelectionRules, WorkerEligibilitySignalRules |
| **Repositories** | CustomerProfileRepository, WorkerProfileRepository |
| **Published Events** | `CustomerProfileUpdated`, `AddressAdded`, `WorkerAvailabilityChanged` |
| **Consumed Events** | `UserAuthenticated` |
| **Transaction Boundaries** | Profile and worker-data updates commit within one User-local transaction each, never spanning both clusters unnecessarily |
| **Dependencies** | Auth |
| **Forbidden Dependencies** | Order, Payment, Cart, Delivery, Catalog, Inventory, Promotion |

### 8.3 Store

| Field | Detail |
|---|---|
| **Purpose** | Define stores and the service zones they cover. |
| **Public Services** | StoreDirectoryService, ServiceZoneService |
| **Internal Services** | None notable |
| **Application Services** | StoreResolutionOrchestrationService (ResolveServingStoreForAddress) |
| **Domain Services** | ServiceZoneMatchingRules |
| **Repositories** | StoreRepository, ServiceZoneRepository |
| **Published Events** | `StoreActivated`, `StoreDeactivated`, `ServiceZoneChanged` |
| **Consumed Events** | None |
| **Transaction Boundaries** | Store/zone changes commit within one Store-local transaction |
| **Dependencies** | None |
| **Forbidden Dependencies** | Every other module |

### 8.4 Catalog

| Field | Detail |
|---|---|
| **Purpose** | Own product, category, and brand definitions. |
| **Public Services** | ProductCatalogService, CategoryBrandService |
| **Internal Services** | None notable |
| **Application Services** | ProductLifecycleOrchestrationService (create/update/deactivate) |
| **Domain Services** | ProductVisibilityRules |
| **Repositories** | ProductRepository, CategoryRepository, BrandRepository |
| **Published Events** | `ProductCreated`, `ProductUpdated`, `ProductDeactivated`, `CategoryChanged` |
| **Consumed Events** | None |
| **Transaction Boundaries** | A product/category/brand change commits within one Catalog-local transaction |
| **Dependencies** | Store |
| **Forbidden Dependencies** | Cart, Order, Payment, Inventory, Promotion, Delivery |

### 8.5 Search

| Field | Detail |
|---|---|
| **Purpose** | Provide relevance-ranked product discovery over a derived index. |
| **Public Services** | SearchService |
| **Internal Services** | IndexBuildingService — maintains the derived index from consumed events, never externally callable |
| **Application Services** | IndexSyncOrchestrationService |
| **Domain Services** | RelevanceRankingRules |
| **Repositories** | None — Search owns no PostgreSQL-persisted entity (`data-ownership-map.md` §12.5); its index is maintained by IndexBuildingService, not a Repository over an owned table |
| **Published Events** | None |
| **Consumed Events** | `ProductCreated`, `ProductUpdated`, `ProductDeactivated`, `InventoryLevelChanged` (optional) |
| **Transaction Boundaries** | None — Search performs no PostgreSQL writes of record; index updates are not transactional business writes |
| **Dependencies** | Catalog, Inventory (event-driven) |
| **Forbidden Dependencies** | Order, Payment, Cart, Delivery, Promotion, User |

### 8.6 Inventory

| Field | Detail |
|---|---|
| **Purpose** | Track stock and protect it under concurrent demand. |
| **Public Services** | InventoryAvailabilityService, ReservationService |
| **Internal Services** | LedgerRecordingService — records every stock movement on behalf of ReservationService, never called externally |
| **Application Services** | ReservationOrchestrationService (ReserveStock, ReleaseReservation, ConfirmReservationConsumption) |
| **Domain Services** | StockAvailabilityRules, ReservationLockingRules |
| **Repositories** | InventoryItemRepository, ReservationRepository, LedgerRepository |
| **Published Events** | `InventoryReserved`, `InventoryReservationFailed`, `InventoryReservationReleased`, `InventoryLevelChanged`, `InventoryLedgerRecorded` |
| **Consumed Events** | None — reservation is a direct synchronous call from Order, not event-triggered (`module-catalog.md` §4.6) |
| **Transaction Boundaries** | Each reservation, release, or consumption is its own Inventory-local transaction — never combined with Order's own transaction (ADR-0021) |
| **Dependencies** | Store, Catalog |
| **Forbidden Dependencies** | Order, Cart, Payment, Delivery, Promotion |

### 8.7 Cart

| Field | Detail |
|---|---|
| **Purpose** | Hold a customer's pre-checkout selection. |
| **Public Services** | CartService |
| **Internal Services** | None notable |
| **Application Services** | CartMutationOrchestrationService |
| **Domain Services** | CartItemValidationRules |
| **Repositories** | CartRepository |
| **Published Events** | `CartUpdated` (optional) |
| **Consumed Events** | `ProductDeactivated` |
| **Transaction Boundaries** | Each cart mutation commits within one Cart-local transaction |
| **Dependencies** | Catalog, User |
| **Forbidden Dependencies** | Order, Payment, Inventory, Delivery |

### 8.8 Order

| Field | Detail |
|---|---|
| **Purpose** | Own the order record and orchestrate checkout end to end. |
| **Public Services** | OrderService, OrderHistoryService |
| **Internal Services** | CompensationHandlerService — executes Order's compensating actions on a failed checkout step, never externally callable |
| **Application Services** | CheckoutOrchestrationService (the system's most significant Application Service — sequences Cart, Inventory, Payment, and Promotion per `module-communication.md` §10), OrderCancellationOrchestrationService |
| **Domain Services** | OrderEligibilityRules, PriceSnapshotRules |
| **Repositories** | OrderRepository, OrderItemRepository, OrderEventRepository, PriceSnapshotRepository |
| **Published Events** | `OrderPlaced`, `OrderCancelled`, `OrderStatusChanged`, `OrderReadyForFulfillment` |
| **Consumed Events** | `PaymentAuthorized`, `PaymentFailed`, `PaymentCaptured`, `InventoryReservationFailed`, `PickingCompleted`, `DeliveryCompleted`, `DeliveryFailed` (ADR-0036), `DeliveryCompletedWithPayment` (ADR-0035 — forwarded to Payment's PaymentService, never recorded by Order itself) |
| **Transaction Boundaries** | Order-record creation is its own transaction; each downstream step (Inventory reservation, Payment initiation) is that module's own separate transaction, coordinated by CheckoutOrchestrationService with explicit compensation on failure — never one transaction spanning modules (ADR-0021) |
| **Dependencies** | Cart, Inventory, Payment, Promotion, User, Store |
| **Forbidden Dependencies** | Delivery, Notification, Analytics, Search |

### 8.9 Payment

| Field | Detail |
|---|---|
| **Purpose** | Own settlement and refunds across every payment method. |
| **Public Services** | PaymentService, RefundService |
| **Internal Services** | GatewayIntegrationService — wraps the Moyasar integration (ADR-0022), never externally callable, never used by any other module; LedgerRecordingService (ADR-0037) — records every financial movement on behalf of PaymentService/RefundService, never called externally, mirroring Inventory's own LedgerRecordingService shape |
| **Application Services** | PaymentInitiationOrchestrationService, RefundOrchestrationService (now computing a per-line refund breakdown against Order's Price Snapshot — ADR-0038), DeliveryCollectionRecordingService (ADR-0035 — receives Order's forwarded `DeliveryCompletedWithPayment` handling, distinct from PaymentInitiationOrchestrationService since it never talks to Moyasar) |
| **Domain Services** | PaymentStateTransitionRules, RefundConstraintValidationRules (validates the price-snapshot/order-context constraints already established, not the business eligibility decision itself — per ADR-0033, eligibility is Order's rule, executed by Payment once approved by Order or an authorized Support flow), RefundLineItemAllocationRules (ADR-0038 — proportional vs. full-line reversal against a promotion-adjusted Price Snapshot; never re-evaluates promotion eligibility itself) |
| **Repositories** | PaymentRepository, PaymentHistoryRepository, RefundRepository, CashRemittanceRepository, CardOnDeliveryRepository, PaymentLedgerRepository (ADR-0037) |
| **Published Events** | `PaymentAuthorized`, `PaymentCaptured`, `PaymentFailed`, `RefundIssued`, `RefundFailed`, `CashRemittanceRecorded`, `PaymentLedgerRecorded` (ADR-0037) |
| **Consumed Events** | `OrderPlaced` (informational) |
| **Transaction Boundaries** | Each payment or refund operation commits within one Payment-local transaction, over Payment's own five tables only |
| **Dependencies** | Order |
| **Forbidden Dependencies** | Cart, Catalog, Inventory, Delivery, Search, Promotion |

### 8.10 Promotion

| Field | Detail |
|---|---|
| **Purpose** | Own promotion rules, eligibility, and usage tracking. |
| **Public Services** | PromotionEligibilityService, PromoCodeService |
| **Internal Services** | None notable |
| **Application Services** | PromotionEvaluationOrchestrationService |
| **Domain Services** | EligibilityRuleEngine, DiscountCalculationRules |
| **Repositories** | PromotionRepository, PromoCodeRepository, PromotionUsageRepository |
| **Published Events** | `PromotionUsageRecorded` |
| **Consumed Events** | `OrderCancelled` |
| **Transaction Boundaries** | Eligibility evaluation and usage recording each commit within one Promotion-local transaction |
| **Dependencies** | Catalog, User |
| **Forbidden Dependencies** | Order, Cart, Payment, Inventory, Delivery |

### 8.11 Delivery

| Field | Detail |
|---|---|
| **Purpose** | Own in-store picking, last-mile delivery, and — per ADR-0034 — grouping eligible deliveries into rider batches. |
| **Public Services** | PickingService, DeliveryAssignmentService, DeliveryBatchingService (ADR-0034) |
| **Internal Services** | None notable |
| **Application Services** | PickingOrchestrationService, DeliveryOrchestrationService, BatchFormationOrchestrationService (ADR-0034 — evaluates eligibility and forms a batch as one Delivery-local operation) |
| **Domain Services** | RiderAssignmentRules, DeliveryCompletionValidationRules, BatchEligibilityRules (proximity/readiness-time/capacity/SLA checks against Settings-owned thresholds — ADR-0034), RouteSequencingRules (stop ordering within a batch; deliberately a pure rule with no routing-engine implementation committed, per ADR-0034's future-replacement note) |
| **Repositories** | PickingSessionRepository, DeliveryAssignmentRepository, RiderLocationRepository, DeliveryBatchRepository, DeliveryStopRepository (ADR-0034) |
| **Published Events** | `PickingStarted`, `PickingCompleted`, `DeliveryAssigned`, `DeliveryCompleted`, `RiderLocationUpdated`, `DeliveryFailed`, `DeliveryRejected` (ADR-0036), `DeliveryCompletedWithPayment` (ADR-0035), `DeliveryBatchCreated`, `DriverAssignedToBatch`, `BatchRouteUpdated`, `BatchCompleted`, `BatchCancelled` |
| **Consumed Events** | `OrderReadyForFulfillment` |
| **Transaction Boundaries** | Each picking, delivery-assignment, or batch-formation/update operation commits within one Delivery-local transaction, over Delivery's own tables only — batch formation never opens a transaction spanning Order, Payment, or Inventory's tables (ADR-0034; ADR-0021's rule applied identically) |
| **Dependencies** | Order, User, Store, Settings (batching-eligibility thresholds) |
| **Forbidden Dependencies** | Payment, Cart, Catalog, Promotion, Search |

### 8.12 Notification

| Field | Detail |
|---|---|
| **Purpose** | Dispatch outbound customer/worker communication. |
| **Public Services** | NotificationService |
| **Internal Services** | ChannelDispatchService — routes to Unifonic SMS or push per channel (ADR-0023), never externally callable |
| **Application Services** | NotificationDispatchOrchestrationService |
| **Domain Services** | ChannelSelectionRules |
| **Repositories** | NotificationRepository |
| **Published Events** | `NotificationSent`, `NotificationFailed`, `NotificationDeliveryConfirmed` |
| **Consumed Events** | `OtpRequested`, `OrderPlaced`, `OrderStatusChanged`, `PaymentAuthorized`, `PaymentFailed`, `DeliveryAssigned`, `DeliveryCompleted` |
| **Transaction Boundaries** | Each dispatch attempt commits within one Notification-local transaction |
| **Dependencies** | None (event-driven); Auth calls it directly for OTP dispatch |
| **Forbidden Dependencies** | Order, Payment, Cart, Inventory, Delivery, Catalog, Promotion |

### 8.13 Support

| Field | Detail |
|---|---|
| **Purpose** | Own support tickets and their resolution workflow. |
| **Public Services** | TicketService |
| **Internal Services** | None notable |
| **Application Services** | TicketLifecycleOrchestrationService |
| **Domain Services** | TicketLinkingRules |
| **Repositories** | SupportTicketRepository, SupportTicketCommentRepository |
| **Published Events** | `SupportTicketOpened`, `SupportTicketAssigned`, `SupportTicketStatusChanged`, `SupportTicketCommentAdded`, `SupportTicketClosed`, `SupportTicketReopened` |
| **Consumed Events** | `OrderCancelled`, `PaymentFailed`, `RefundIssued`, `DeliveryCompleted`, `DeliveryFailed` |
| **Transaction Boundaries** | Each ticket operation commits within one Support-local transaction |
| **Dependencies** | User, Order, Payment, Delivery |
| **Forbidden Dependencies** | Catalog, Inventory, Cart, Promotion, Search |

### 8.14 Analytics

| Field | Detail |
|---|---|
| **Purpose** | Build rebuildable reporting rollups and analytics snapshots. |
| **Public Services** | ReportingService |
| **Internal Services** | RollupGenerationService — scheduled, never externally callable except via TriggerRollupGeneration on ReportingService |
| **Application Services** | RollupOrchestrationService |
| **Domain Services** | AggregationRules |
| **Repositories** | ReportRollupRepository, AnalyticsSnapshotRepository |
| **Published Events** | None |
| **Consumed Events** | `OrderPlaced`, `OrderCancelled`, `PaymentCaptured`, `RefundIssued`, `DeliveryCompleted`, and other reporting-relevant events as they're added |
| **Transaction Boundaries** | Rollup/snapshot generation commits within one Analytics-local transaction |
| **Dependencies** | None synchronously |
| **Forbidden Dependencies** | Order, Payment, Cart, Inventory, Delivery |

### 8.15 Audit

| Field | Detail |
|---|---|
| **Purpose** | Record the durable, immutable audit trail. |
| **Public Services** | AuditTrailService |
| **Internal Services** | None |
| **Application Services** | AuditRecordingOrchestrationService |
| **Domain Services** | None — deliberately absent, consistent with Audit's Boundary of never interpreting or judging the actions it records (`module-catalog.md` §4.15) |
| **Repositories** | AuditLogRepository |
| **Published Events** | None |
| **Consumed Events** | Every module's consequential actions reach Audit via direct call or consumed event, per `integration-and-messaging.md` Section 5: consumed event where one already exists for the underlying action, a direct synchronous `RecordAuditEntry` call otherwise |
| **Transaction Boundaries** | Each audit entry is recorded within its own Audit-local transaction, independent of the transaction of the action being recorded |
| **Dependencies** | None |
| **Forbidden Dependencies** | Every other module |

### 8.16 Settings

| Field | Detail |
|---|---|
| **Purpose** | Own platform and store-scoped configuration. |
| **Public Services** | SettingsService |
| **Internal Services** | None |
| **Application Services** | ConfigurationChangeOrchestrationService |
| **Domain Services** | SettingScopeResolutionRules |
| **Repositories** | SettingsRepository |
| **Published Events** | `SettingChanged` |
| **Consumed Events** | None |
| **Transaction Boundaries** | Each configuration change commits within one Settings-local transaction |
| **Dependencies** | None |
| **Forbidden Dependencies** | Every other module |

## 9. Legal Service Interaction Examples

- Order's CheckoutOrchestrationService (Application Service) calls Inventory's ReservationService (Public Service) — a cross-module call through the only door Inventory has for outsiders.
- Within Order, CheckoutOrchestrationService calls OrderEligibilityRules (its own Domain Service) before calling any other module — an in-process, same-module call needing no cross-module sanction at all.
- Search's IndexSyncOrchestrationService (Application Service) reacts to a consumed `ProductUpdated` event and calls its own IndexBuildingService (Internal Service) — never a call back into Catalog.
- Support's TicketLifecycleOrchestrationService calls Payment's RefundService (Public Service) to issue a refund tied to a ticket — an administrative action still going through the only sanctioned door.
- Auth's LoginOrchestrationService calls Notification's NotificationService (Public Service) to dispatch an OTP — the one named synchronous exception in the whole system (`module-communication.md` Section 3), still going through a Public Service, never Notification's internals.

## 10. Illegal Service Interaction Examples

- Order's CheckoutOrchestrationService calling Inventory's InventoryItemRepository directly "to save a call" instead of going through ReservationService — a Repository is never reachable from outside its own module, regardless of how convenient the shortcut looks.
- Catalog's ProductLifecycleOrchestrationService calling Promotion's DiscountCalculationRules (a Domain Service) to "preview" a discounted price — Domain Services are never reachable across a module boundary; only Public Services are.
- Analytics' RollupGenerationService opening a transaction against Order's OrderRepository to compute same-day figures faster than event-driven rollups allow — a Repository is module-private with no exception for urgency or convenience.
- Delivery's PickingOrchestrationService calling Search's SearchService synchronously to "check if the product is still listed" before starting a picking session — Search is Forbidden as a Delivery dependency (`module-catalog.md` §4.11), and even where it weren't, a Domain-relevant decision like this belongs to Catalog or Inventory, not Search.
- A controller in any module calling a Domain Service directly, bypassing the Application Service layer, "because it's a simple rule" — controllers call only Public Services (Section 7's Thin Controllers rule), never anything beneath that layer, no matter how simple the underlying logic is.

## 11. Validation and Error Propagation in Practice

Restating Section 7's rules with a worked path: a malformed checkout request is rejected by Order's controller before CheckoutOrchestrationService is ever invoked (transport-shape validation). Once inside CheckoutOrchestrationService, a business-rule rejection — insufficient stock, reported by Inventory's ReservationService as a normal, expected outcome — is not treated as a system failure; CheckoutOrchestrationService receives it, performs its compensating action (Section 8.8), and returns a clear, specific outcome to the controller, which translates it into a response. An unexpected failure (a database connection lost mid-transaction inside Inventory's own ReservationRepository) propagates up through ReservationService as a surfaced, logged failure — never silently caught and hidden by CheckoutOrchestrationService as if it were an ordinary business-rule rejection. The distinction between these two paths (`coding-standards.md` Section 7) is what a Domain Service's return type and an Application Service's error handling exist to preserve.

## 12. Service Dependency Matrix

Rows are the calling module ("From"); columns are the called module ("To"). **A** = Allowed (an explicit Dependency in `module-catalog.md`); **F** = Forbidden (an explicit Forbidden Dependency); **N** = Not sanctioned — no dependency is documented in either direction, and per ADR-0009's closed-world posture (a dependency must be explicitly documented to be legal), this is treated as forbidden by default, not merely undecided. `–` marks a module against itself.

| From \ To | Auth | User | Store | Catalog | Search | Inventory | Cart | Order | Payment | Promotion | Delivery | Notification | Support | Analytics | Audit | Settings |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **Auth** | – | N | N | F | N | F | F | F | F | F | F | A | N | N | N | A |
| **User** | A | – | N | F | N | F | F | F | F | F | F | N | N | N | N | N |
| **Store** | F | F | – | F | F | F | F | F | F | F | F | F | F | F | F | A |
| **Catalog** | N | N | A | – | N | F | F | F | F | F | F | N | N | N | N | N |
| **Search** | N | F | N | A | – | A | F | F | F | F | F | N | N | N | N | N |
| **Inventory** | N | N | A | A | N | – | F | F | F | F | F | N | N | N | N | A |
| **Cart** | N | A | N | A | N | F | – | F | F | N | F | N | N | N | N | N |
| **Order** | N | A | A | N | F | A | A | – | A | A | F | F | N | F | N | A |
| **Payment** | N | N | N | F | F | F | F | A | – | F | F | N | N | N | N | A |
| **Promotion** | N | A | N | A | N | F | F | F | F | – | F | N | N | N | N | A |
| **Delivery** | N | A | A | F | F | N | N | A | F | F | – | N | N | N | N | A |
| **Notification** | N | N | N | F | N | F | F | F | F | F | F | – | N | N | N | A |
| **Support** | N | A | N | F | F | F | F | A | A | F | A | N | – | N | N | N |
| **Analytics** | N | N | N | N | N | F | F | F | F | N | F | N | N | – | N | A |
| **Audit** | F | F | F | F | F | F | F | F | F | F | F | F | F | F | – | F |
| **Settings** | F | F | F | F | F | F | F | F | F | F | F | F | F | F | F | – |

## 13. Future Extraction Candidates

Named without committing to a timeline or changing anything about the current architecture (Section 6):

- **Order and Inventory** — already named together by `01-Architecture-Design-Specification.md` Section 13 as the most plausible first candidates: Order for its orchestration centrality, Inventory for its concurrency-sensitive reservation logic. Both already reach every other module exclusively through Public Services and events, so their Application/Domain/Repository layers would move unchanged.
- **Auth** — the most self-contained module in the system (`module-catalog.md` §4.1's own framing: "every other module depends on it, it depends on almost nothing"), making it a low-risk extraction candidate independent of Order/Inventory's timeline.
- **Search** — already architecturally isolated (owns no primary data, consumed-event-driven only), so extracting it — or replacing its internal indexing technology entirely — touches nothing about any other module's Public Services or Repositories.
- **Analytics** — similarly isolated (pure event consumer, no synchronous callers), a natural candidate if a dedicated analytical datastore is ever introduced behind its existing Public Service.

None of these candidates changes this document's Public Services, Dependencies, or Forbidden Dependencies — extraction is a deployment-boundary decision, not an architectural one, and remains gated behind evidence per Constitution Section 13.

## 14. Open Decisions

- The internal Application/Domain/Internal Service names in Section 8 are this document's own conceptual labels, consistent with `coding-standards.md` Section 3's layering but not previously named anywhere — a future SDD for any module may refine or rename them without that being an architectural change, as long as the module's Public Services, Dependencies, and Forbidden Dependencies remain exactly as `module-catalog.md` and this document state them.
- Whether Search and Analytics require any transactional guarantee at all around their own derived-data writes (Sections 8.5, 8.14) is left to their eventual SDDs — this document states they are not authoritative and therefore not held to the same transaction-ownership rigor as the other fourteen modules, but does not prescribe a specific mechanism.
- The Open Decisions previously carried through `module-catalog.md`, `capability-boundary-map.md`, and `data-ownership-map.md` (Settings' dependents, refund eligibility's decision owner, Auth's session/token storage mechanism) are resolved by ADR-0033 and must not be treated as open going forward.
- The internal service names introduced here for ADR-0035–0041 (`DeliveryCollectionRecordingService`, `LedgerRecordingService`, `RefundLineItemAllocationRules`, `StepUpAuthenticationService`, `StoreScopeEvaluationRules`, and related) are, like every other name in Section 8, this document's own conceptual labels — a future SDD may refine or rename them without that being an architectural change, as long as each module's Public Services, Dependencies, and Forbidden Dependencies remain exactly as `module-catalog.md` and this document state them.
