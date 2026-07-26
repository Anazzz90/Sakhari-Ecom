# System Context — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Stability** | Evolving — updated whenever an actor, external system, or zone-level boundary is added, removed, or re-scoped. The four-zone shape itself (Customer Platform, Operations Platform, Backend, Infrastructure) is expected to be stable; what evolves is the detail within each zone. |
| **Status** | Authored — expands `01-Architecture-Design-Specification.md` Section 6 into full detail. |
| **Authority** | Subordinate to `00-Architecture-Principles.md` and `01-Architecture-Design-Specification.md`. Where this document adds detail, it must not contradict either; where it appears to, this document is presumed wrong until reconciled. |

## Purpose

Describes Sakhari Ecom as a system with a boundary: what sits inside that boundary, what sits outside it, and exactly how the inside talks to the outside and to itself. This document is the architectural equivalent of the SRD's actor table, redrawn as a system boundary rather than a feature list, and detailed enough that a diagram — C4 Level 1 (System Context) or Level 2 (Container) — can be produced directly from it without re-deriving the underlying model. It contains no diagrams itself; see "Diagram-Ready Model" (Section 8) and `diagrams/README.md` for how this document is meant to be turned into one later.

This document does not describe deployment (regions, environments, CI/CD, scaling — see `05-deployment/`) or implementation (frameworks, libraries, code structure — see future SDD documents). It describes *what talks to what*, not *how it runs* or *how it's built*.

## Relationship to other documents

- **SRD:** Source for the actor list (Customer, Picker, Rider, Admin/Ops) and the external providers named there (payment gateway, SMS/OTP provider, cloud infrastructure). This document does not introduce an actor or external system the SRD gives no basis for.
- **DDD:** Source for which entities and data responsibilities sit at the edge of the system — i.e., which capability owns the data an external integration touches (a payment record owned by Payments, a notification log owned by Notifications). This document does not redefine those entities, only where their owning capability sits relative to the system boundary.
- **`01-Architecture-Design-Specification.md`:** This document is the full expansion of the ADS's Section 6 (System Context) and Section 5 (Platform Overview); it must stay consistent with both, and with the module groupings in ADS Section 10.
- **`03-decomposition/`:** The precise module boundaries inside the Backend zone (Section 5 below) are this document's responsibility only at the level needed to describe integration points; the authoritative, fine-grained module map is `capability-boundary-map.md` and `service-decomposition.md`.
- **SDD:** Each external integration and each internal integration named here should map to one or more SDD-level integration design documents once those exist.
- **Implementation:** Every external credential, webhook, SDK, or internal API contract in code should trace back to an entry in this document's Diagram-Ready Model (Section 8).

---

## 1. System Boundary Statement

Sakhari Ecom, as a system, consists of four zones:

1. **Customer Platform** — the customer-facing surfaces (Section 3)
2. **Operations Platform** — the internal staff-facing surfaces (Section 4)
3. **Backend** — the single modular-monolith service holding all business logic (Section 5)
4. **Infrastructure** — the datastores the Backend depends on (Section 6)

Everything else — human actors and external systems — sits **outside** the system boundary and reaches it only through one of the four zones above, never directly into an inner zone. In particular: no actor or external system talks to the Backend or Infrastructure directly except through a Platform (for actors) or through the Backend itself (for external systems) — see the Trust Boundaries in each zone's section and the consolidated model in Section 8.

This four-zone shape is deliberately coarser than the Backend's internal capability modules (Identity & Access, Catalog & Inventory, Ordering, Pricing & Promotions, Payments, Dispatch & Fulfillment, Store & Operations Management, Notifications, Admin/Reporting & Analytics — ADS Section 10). At the system-context level, those modules are treated as one zone (the Backend) because none of them is independently reachable from outside the system; their individual boundaries are the concern of `03-decomposition/`, not of this document.

---

## 2. Actors (outside the system boundary)

| Actor | Description | Reaches the system through |
|---|---|---|
| **Customer** | Browses, orders, pays, tracks delivery. | Customer Platform |
| **Picker** | Picks and packs orders inside a dark store. | Operations Platform (Worker App) |
| **Rider** | Delivers packed orders between a dark store and a customer. | Operations Platform (Worker App) |
| **Admin / Ops / Support** | Manages catalog, inventory, workers, stores, dispatch, promotions, and reporting; the SRD's single "Admin/Ops" actor category, refined internally by RBAC role (ADR-0012). | Operations Platform (Admin/Ops Dashboard) |

No actor is trusted by anything other than the Platform built for them. A Customer has no reachable path to the Operations Platform, and a Picker or Rider has no reachable path to Admin/Ops Dashboard functionality — see each zone's Trust Boundaries below.

---

## 3. Customer Platform

**Composition:** Customer Mobile App + Customer Web App. Together these form the Customer Platform (ADS Section 5) — two presentations of one product, never diverging in business behavior (Principle 4.6).

### Responsibilities
- Present the customer-facing product: browse catalog, manage cart, checkout, track orders, manage profile/addresses, view order history, engage support, view promotions.
- Enforce presentation-level restrictions for usability (e.g., graying out an unavailable action) — never the actual authority on whether an action is allowed.
- Handle device- or browser-specific concerns (push notification display, responsive layout, offline/poor-connectivity handling) that are presentation, not business, concerns (Principle 4.6's exception).

### Boundaries
- Contains no business logic, no direct data access, and no independent copy of business rules — every business decision (pricing, availability, order validity) is delegated to the Backend (Principle 4.6, ADR-0007's rationale for Next.js server-side code not becoming a second home for logic).
- The Customer Mobile App and Customer Web App are two separate applications but one platform: they must never present different prices, different availability, or different eligibility for the same customer at the same time.
- Carries only customer-relevant capability — no picker, rider, or admin functionality is present in either client, mirroring the Worker App's own AMR-001 rule in reverse.

### Trust Boundaries
- The Customer Platform is **not trusted** to self-report identity, permissions, or the correctness of any value it sends (price, quantity, discount). Every request is re-validated and re-authorized by the Backend regardless of what the client believes to be true (ADS Security Philosophy, Section 12).
- Crossing from Customer Platform into the Backend is where authentication (proving who the caller is) and authorization (RBAC — what a Customer role may do, ADR-0012) are enforced. Nothing before that crossing is trusted.
- A compromise of the Customer Mobile App or Customer Web App grants an attacker nothing toward Operations Platform capability — the two platforms share no trust, only a common backend that independently authorizes each (ADS Security Philosophy).

### External Integrations
- **None directly, by default.** The Customer Platform does not talk to the payment gateway, the SMS/OTP provider, or the push notification service directly — all such interaction is mediated exclusively through the Backend (Principle 4.6; ADS Section 12). This is a deliberate architectural choice, not an omission: it keeps every external credential and every business-consequential external call in exactly one place.
- **One open exception, not yet settled:** whether address-entry UX calls a client-embedded maps/geocoding SDK directly (common industry practice for autocomplete responsiveness) or goes through the Backend (this document's default assumption — see Section 5). Until settled by `04-cross-cutting/integration-and-messaging.md` or an ADR, this document treats the Customer Platform's external integrations as none, with this one point flagged rather than silently decided.
- **Inbound only, not initiated by the platform:** the Push Notification Service delivers directly to the installed Customer Mobile App (Section 5's Trust Boundaries note) — this is a delivery path into the platform, not a call the platform makes out.

### Internal Integrations
- **Customer Platform → Backend:** the platform's only integration path. All requests (browse, cart, checkout, order tracking, profile, support, promotions) go through the Backend's versioned API (ADR-0017), authenticated and authorized on every call.

---

## 4. Operations Platform

**Composition:** Worker App (Picker and Rider modes) + Admin/Ops Dashboard. This document names these two applications collectively as the **Operations Platform** — the internal-facing counterpart to the Customer Platform, introduced here for architectural symmetry with ADS Section 5's Customer Platform grouping and to give the two internal-facing clients one name when describing system-level boundaries. It is not a single deployable or a single codebase; like the Customer Platform, it is two separate applications sharing a trust model and a backend, never a merged trust surface.

### Responsibilities
- **Worker App (Picker mode):** receive and fulfill picking tasks inside a dark store.
- **Worker App (Rider mode):** receive and fulfill delivery tasks between a dark store and a customer.
- **Admin/Ops Dashboard:** manage catalog, inventory, orders, workers, stores, dispatch, promotions, and reporting/analytics, scoped by the acting user's RBAC role (Admin, Ops, or Support — ADR-0012).

### Boundaries
- Like the Customer Platform, contains no independent business logic — every business decision is delegated to the Backend (Principle 4.6).
- The Worker App carries no customer functionality under any circumstance (SRD rule AMR-001); Picker and Rider modes share app shell, authentication, profile, and notifications, but not task-level capability (SRD rule AMR-002).
- The Admin/Ops Dashboard is staff-only and never publicly distributed; it is not merely "more privileged" than the Worker App, it is a categorically different application with a different actor population.
- Internally, the Operations Platform is not one trust level: Admin, Ops, and Support see different capability inside the same Dashboard application, and Picker/Rider see different task types inside the same Worker App — both differentiated by RBAC (ADR-0012), not by the client.

### Trust Boundaries
- The Operations Platform is **not trusted** to self-report identity, role, or permissions, exactly as the Customer Platform is not (ADS Security Philosophy). Every action — including ones the client's own UI already hid based on a cached role — is re-authorized by the Backend's RBAC evaluation on every request.
- A compromise of the Worker App grants an attacker nothing toward Admin/Ops Dashboard capability, and vice versa; a compromise of either grants nothing toward Customer Platform capability. The three client surfaces (Customer Platform, Worker App, Admin/Ops Dashboard) share no client-side trust with one another — only a common backend that independently authorizes each.
- Within the Admin/Ops Dashboard, an Ops or Support session is not trusted with Admin-only capability (e.g., permission management) even though it is the same application — this distinction is enforced by RBAC at the Backend, not by anything the Dashboard client withholds on its own authority.

### External Integrations
- **None directly.** As with the Customer Platform, no external system (payment gateway, SMS/OTP provider, geocoding provider) is reachable directly from the Worker App or the Admin/Ops Dashboard. Every external effect an operations action might trigger (for example, an Admin issuing a refund) is executed by the Backend, never by the client calling out itself.
- **Possible inbound push, unconfirmed:** if the Worker App is later confirmed to use the same Push Notification Service as the Customer Mobile App (Section 5), that would be an inbound delivery path into the Worker App only, not an integration the platform initiates — consistent with the Customer Platform's own treatment of push in Section 3.

### Internal Integrations
- **Operations Platform → Backend:** the platform's only integration path, for both the Worker App and the Admin/Ops Dashboard, through the Backend's versioned API (ADR-0017), authenticated and RBAC-authorized on every call.

---

## 5. Backend

**Composition:** a single modular-monolith service (ADR-0002), internally organized into capability-aligned modules (ADS Section 10): Identity & Access, Catalog & Inventory, Ordering, Pricing & Promotions, Payments, Dispatch & Fulfillment, Store & Operations Management, Notifications, and Admin/Reporting & Analytics. This document treats the Backend as one zone at the system-context level; the modules' individual internal boundaries and interfaces are the subject of `03-decomposition/`.

The DDD's per-entity "Module Owner" column (Section 4, Entity Catalog) is more granular than this nine-capability grouping — it names distinct owners such as Auth & Identity, Customer, Workforce, Store, Catalog, Inventory, Cart, Order, Picking, Delivery, Payment, Refund, Promotion, Pricing, Notification, Support, Administration, Audit/Compliance, and Reporting & Analytics. Reconciling that finer data-ownership granularity with the ADS's nine-capability grouping is `03-decomposition/capability-boundary-map.md`'s job, not this document's; at the system-context level, all of it remains inside the single `zone.backend` node.

### Responsibilities
- Hold all business logic for the entire system (Principle 4.6) — the Backend is the only party in the system that decides whether an action is permitted, what something costs, or what state a change produces.
- Authenticate and authorize every request from both Platforms, centrally, via the Identity & Access capability and RBAC (ADR-0012).
- Own and protect all durable business state through Infrastructure (Section 6), including transactional guarantees for atomic operations (ADR-0010) and an audit trail for consequential actions (Principle 4.10).
- Mediate every external interaction the system has — payment processing, SMS/OTP delivery — so that no client ever holds an external credential or calls an external system directly.
- Raise asynchronous events for side effects downstream of a completed operation (ADR-0011), consumed by the appropriate module (most visibly Notifications).

### Boundaries
- The Backend is a single deployable unit (ADR-0002); at the system-context level it presents one boundary to both Platforms, even though it is internally organized into modules.
- Each internal module exclusively owns its own data; no module may access another module's tables directly (ADR-0009) — this is an internal-to-Backend boundary, invisible from outside the Backend but load-bearing for how the Backend is described in `03-decomposition/`.
- The Backend's API surface is versioned (ADR-0017); both Platforms consume the same versioned contract, with no platform-specific business behavior.
- Entry points into the Backend (controllers/handlers) are deliberately thin — translation only, never a decision point (Principle 4.7). The actual boundary of trust and logic sits at the service layer just behind that entry point, not at the entry point itself.

### Trust Boundaries
- The Backend is the **first fully trusted zone** in the system: everything crossing into it from a Platform is treated as untrusted input requiring validation and authorization (Constitution Section 8), and everything the Backend itself produces and commits is treated as true.
- The Backend does not trust external systems' responses uncontested either — a payment gateway's callback or an SMS provider's delivery status is validated (signature/webhook verification, expected-shape checks) before being treated as fact, consistent with the same "validate at the boundary" discipline applied to client input.
- Internally, one module does not trust another module's data directly — it trusts only that module's exposed interface (ADR-0009); this is a trust boundary in miniature, repeated at every module-to-module edge inside the Backend.
- The Backend fully trusts Infrastructure (Section 6) as its own committed state once a write is acknowledged — there is no additional verification step between the Backend and PostgreSQL, since PostgreSQL's transactional guarantee is itself what the Backend's own correctness guarantees (ADR-0010) are built on.
- A push notification delivered by the Push Notification Service directly to a client (Section 3/4) is never treated by a Platform as authoritative data in its own right — it is a wake-up/display trigger only. Whatever it announces (a new order status, a new dispatch assignment) is re-fetched through the normal, authenticated Platform → Backend path before being acted on, preserving the same "the Backend decides what's true" trust model even for the one integration that reaches a Platform without an intervening Backend request.

### External Integrations
- **Moyasar Payment Gateway** — authorization and capture across mada, card, Apple Pay, and BNPL where supported; owned by the Payments capability (ADR-0022). Cash and card-on-delivery (collected by the Rider through the Worker App, per the DDD's Cash Remittance and Card-on-Delivery Record entities) are reconciled inside the Backend/Infrastructure and do not require this integration to complete an order (ADR-0010). Card-on-delivery terminal/POS settlement reconciliation with a payment terminal provider is explicitly unresolved (the DDD marks it "a Phase 1 spike") and is not treated as a committed external integration by this document.
- **Unifonic SMS/OTP Provider** — authentication OTPs, rider delivery-confirmation OTPs, and SMS delivery/status notifications; owned by the Identity & Access capability (OTP) and the Notifications capability (delivery/status messages, delivery-confirmation OTP) respectively (ADR-0023).
- **Push Notification Service** — outbound push delivery to installed mobile clients, dispatched by the Notifications capability. Confirmed for the Customer Mobile App by the DDD's Notification entity; the Worker App's use of the same channel is architecturally plausible (it is also a mobile client) but not yet explicitly confirmed by the SRD or DDD, and should not be assumed settled until it is. Browser notifications for the Customer Web App and email are both noted in the DDD as future channels, not current ones.
- **Geocoding / Maps Provider** — address validation and geocoding metadata for customer delivery addresses (DDD Section 5.3, "Address validation depends on Google Maps/geocoding"). This document defaults to treating the integration as Backend-mediated, consistent with every other external integration (Principle 4.6); the DDD does not state whether address-entry UX instead calls a client-embedded maps SDK directly from the Customer Platform, which would be a narrow, deliberate exception to that default. This is flagged as an open point for `04-cross-cutting/integration-and-messaging.md` or a future ADR to settle, not asserted either way here.

All four are called exclusively by the Backend; neither Platform ever calls any of them directly, with the geocoding/maps caveat above noted explicitly as unresolved rather than silently assumed (Section 3, Section 4).

### Internal Integrations
- **Customer Platform → Backend** and **Operations Platform → Backend:** inbound, synchronous, versioned API calls (ADR-0017), authenticated and authorized on every call.
- **Module → module, within the Backend:** two patterns only, per Principles 4.8/4.9 and ADR-0009/0010/0011 —
  - *Synchronous, transactional*: used only where an all-or-nothing outcome is required across modules within one request (e.g., Ordering + Catalog & Inventory during checkout — ADR-0010), always through the owning module's interface, never a direct table access.
  - *Asynchronous, event-driven*: used for everything downstream of a completed operation that the caller does not need to wait on (e.g., Ordering emits an event that Notifications and Dispatch & Fulfillment consume independently — ADR-0011).
- **Backend → Infrastructure:** described in Section 6.

---

## 6. Infrastructure

**Composition:** PostgreSQL (the system of record, ADR-0003), Redis (acceleration and coordination only, ADR-0004), and Object Storage (media and file assets, ADR-0016). This document treats these three datastores as one zone because all three are reachable only from the Backend, never from a Platform or an external system directly.

### Responsibilities
- **PostgreSQL:** hold every durable business fact — orders, inventory, payments, accounts, entity lifecycle state — as the sole system of record (Principle 4.2).
- **Redis:** accelerate reads, hold sessions, enforce rate limits, and provide short-lived coordination (locks, idempotency keys) — never as an independent record of a business fact (Principle 4.3).
- **Object Storage:** hold product imagery and other file/media assets referenced by, but not stored inside, PostgreSQL rows (ADR-0016).

### Boundaries
- Infrastructure holds no business logic of any kind — no stored procedures or triggers acting as a second home for business rules (Principle 4.6's requirement that services, not the database, own logic).
- PostgreSQL's authority is absolute for anything it holds; Redis and Object Storage are both explicitly non-authoritative and fully disposable/reconstructable from PostgreSQL's perspective (Principles 4.2, 4.3).
- Infrastructure is reachable **only** from the Backend. No Platform, no external system, and no actor ever connects to PostgreSQL, Redis, or Object Storage directly.

### Trust Boundaries
- Infrastructure is fully trusted by the Backend, and the Backend is the only party Infrastructure trusts — there is no independent authorization layer inside PostgreSQL, Redis, or Object Storage; all access control is enforced by the Backend before a request ever reaches this zone.
- Because Infrastructure has no reachable path from outside the Backend, it has no trust boundary of its own to defend beyond "only the Backend may write here" — this is what makes Principle 4.2/4.3's guarantees enforceable in practice, not just in policy.

### External Integrations
- Infrastructure runs on managed services provided by a cloud infrastructure provider (compliant with Saudi data-residency requirements). At the system-context level, this is noted only as an external dependency for hosting, backup, and operational continuity of the datastores themselves — the specific provisioning, regions, and topology are deployment concerns and out of scope for this document (see `05-deployment/infrastructure-and-release.md`).

### Internal Integrations
- **Backend → PostgreSQL:** transactional reads and writes, module-scoped (ADR-0009) — every module writes only to the tables it owns.
- **Backend → Redis:** caching, session, rate-limit, and coordination operations, scoped per module use case, never as a substitute for a PostgreSQL write of record.
- **Backend → Object Storage:** upload/retrieve operations for media assets, with PostgreSQL holding only the reference (key/URL) to each object (ADR-0016).

---

## 7. External Systems (consolidated)

| External system | Used by | Purpose | Trust treatment |
|---|---|---|---|
| **Moyasar Payment Gateway** (mada, card, Apple Pay, BNPL where supported) | Backend (Payments capability) only | Payment authorization and capture | Backend validates every response/callback before treating it as fact; never called by a Platform directly |
| **Unifonic SMS/OTP Provider** | Backend (Identity & Access, Notifications capabilities) only | Authentication OTPs, rider delivery-confirmation OTPs, SMS delivery/status notifications | Backend validates delivery/status callbacks; never called by a Platform directly |
| **Push Notification Service** | Backend (Notifications capability) outbound; delivers inbound directly to the Customer Mobile App (confirmed) and possibly the Worker App (unconfirmed) | Push delivery to installed mobile clients | Backend-initiated send is validated as any other external call; the inbound delivery to a client is never treated as authoritative data (see Section 5's Trust Boundaries note) |
| **Geocoding / Maps Provider** | Backend by default (open point — see Section 3, Section 5) | Address validation and geocoding metadata for customer addresses | Backend validates/normalizes results before persisting; direct client-embedded use is a flagged, unresolved possibility, not a decision made by this document |
| **AWS `me-central-2` Cloud Infrastructure** | Infrastructure zone only | Managed hosting substrate for PostgreSQL/RDS, Redis, and Object Storage/S3 in Riyadh | Accepted production region/provider per ADR-0024 — see `05-deployment/` |

With the one flagged exception (geocoding/maps, if later confirmed client-embedded) and the one inbound-only exception (push delivery), no external system in this table is ever reachable from the Customer Platform or the Operations Platform. Every external edge in the system originates from the Backend or, for hosting only, from Infrastructure. A payment terminal/POS settlement provider for card-on-delivery reconciliation is explicitly **not** included here — the DDD marks that integration "a Phase 1 spike," i.e. unresolved, and this document does not treat it as committed.

---

## 8. Diagram-Ready Model

This section exists specifically so a future diagram — C4 Level 1 (System Context) or Level 2 (Container), per `diagrams/README.md` — can be produced by translating the tables below directly into diagram elements, without re-deriving the model from prose.

### Nodes

| ID | Name | Type | Notes |
|---|---|---|---|
| `actor.customer` | Customer | External actor | Reaches the system via `zone.customer-platform` |
| `actor.picker` | Picker | External actor | Reaches the system via `zone.operations-platform` (Worker App) |
| `actor.rider` | Rider | External actor | Reaches the system via `zone.operations-platform` (Worker App) |
| `actor.admin-ops` | Admin / Ops / Support | External actor | Reaches the system via `zone.operations-platform` (Admin/Ops Dashboard); RBAC-differentiated internally |
| `zone.customer-platform` | Customer Platform | System zone (container group) | Composed of `app.customer-mobile` + `app.customer-web` |
| `app.customer-mobile` | Customer Mobile App | Container | Part of `zone.customer-platform` |
| `app.customer-web` | Customer Web App | Container | Part of `zone.customer-platform` |
| `zone.operations-platform` | Operations Platform | System zone (container group) | Composed of `app.worker-app` + `app.admin-dashboard` |
| `app.worker-app` | Worker App (Picker/Rider modes) | Container | Part of `zone.operations-platform` |
| `app.admin-dashboard` | Admin/Ops Dashboard | Container | Part of `zone.operations-platform` |
| `zone.backend` | Backend | System zone (container) | Single deployable; internally modular per ADS Section 10 |
| `zone.infrastructure` | Infrastructure | System zone (container group) | Composed of `store.postgresql` + `store.redis` + `store.object-storage` |
| `store.postgresql` | PostgreSQL | Container (data store) | System of record (ADR-0003) |
| `store.redis` | Redis | Container (data store) | Cache/coordination only (ADR-0004) |
| `store.object-storage` | Object Storage | Container (data store) | Media/file assets (ADR-0016) |
| `ext.payment-gateway` | Moyasar Payment Gateway | External system | mada, card, BNPL |
| `ext.sms-otp-provider` | Unifonic SMS/OTP Provider | External system | OTP (auth + rider delivery-confirmation) + SMS notification delivery |
| `ext.push-notification-service` | Push Notification Service | External system | Outbound from Backend; inbound delivery to `app.customer-mobile` (confirmed) and possibly `app.worker-app` (unconfirmed) |
| `ext.geocoding-provider` | Geocoding / Maps Provider | External system | Address validation/geocoding; Backend-mediated by default, client-embedded status unresolved |
| `ext.cloud-provider` | AWS `me-central-2` Cloud Infrastructure | External system | Hosting substrate for `zone.infrastructure` only |

### Edges

| ID | Source | Target | Direction | Interaction type | Crosses a trust boundary? |
|---|---|---|---|---|---|
| `e1` | `actor.customer` | `zone.customer-platform` | Customer → Platform | Human interaction | Yes — actor to client |
| `e2` | `actor.picker` | `zone.operations-platform` | Actor → Platform | Human interaction | Yes — actor to client |
| `e3` | `actor.rider` | `zone.operations-platform` | Actor → Platform | Human interaction | Yes — actor to client |
| `e4` | `actor.admin-ops` | `zone.operations-platform` | Actor → Platform | Human interaction | Yes — actor to client |
| `e5` | `zone.customer-platform` | `zone.backend` | Platform → Backend | Synchronous, versioned API call | Yes — untrusted client to trusted backend; authenticated + authorized |
| `e6` | `zone.operations-platform` | `zone.backend` | Platform → Backend | Synchronous, versioned API call | Yes — untrusted client to trusted backend; authenticated + RBAC-authorized |
| `e7` | `zone.backend` | `ext.payment-gateway` | Backend → External | Synchronous API call (authorization/capture) | Yes — trusted backend to external, response validated |
| `e8` | `zone.backend` | `ext.sms-otp-provider` | Backend → External | Synchronous API call (send) + inbound callback (status) | Yes — trusted backend to external, callback validated |
| `e9` | `zone.backend` | `store.postgresql` | Backend → Infrastructure | Transactional read/write, module-scoped | No — Infrastructure is fully trusted by the Backend |
| `e10` | `zone.backend` | `store.redis` | Backend → Infrastructure | Cache/session/rate-limit/coordination read-write | No — non-authoritative, disposable |
| `e11` | `zone.backend` | `store.object-storage` | Backend → Infrastructure | Upload/retrieve media, reference stored in PostgreSQL | No — non-authoritative for business fact |
| `e12` | `zone.infrastructure` | `ext.cloud-provider` | Infrastructure → External | Managed hosting dependency (out of scope beyond this note) | Deployment-level, not detailed here |
| `e13` | `zone.backend` | `ext.push-notification-service` | Backend → External | Synchronous API call (send) | Yes — trusted backend to external |
| `e14` | `ext.push-notification-service` | `app.customer-mobile` | External → Client (inbound) | Push delivery, confirmed | Yes — but treated as a display trigger only, never authoritative data (Section 5 Trust Boundaries) |
| `e15` | `ext.push-notification-service` | `app.worker-app` | External → Client (inbound) | Push delivery, **unconfirmed** — included for diagram completeness, not to be rendered as settled until confirmed | Yes, if confirmed — same non-authoritative treatment as `e14` |
| `e16` | `zone.backend` | `ext.geocoding-provider` | Backend → External | Synchronous API call (address validation/geocoding) | Yes — default assumption; see Section 3/5 open-point note. An alternative `zone.customer-platform` → `ext.geocoding-provider` edge is explicitly *not* asserted here pending that decision |

Edges `e13`–`e16` should be rendered distinctly (e.g., dashed) in any generated diagram until the two flagged open points — Worker App push (Section 4) and geocoding client-vs-backend mediation (Section 3/5) — are settled by a future document or ADR; every other edge in this model is a settled, confirmed fact as of this document's authoring.

Internal-to-Backend edges (module-to-module, per ADR-0009/0010/0011) are intentionally **not** enumerated at this system-context level — they belong to `03-decomposition/service-decomposition.md`, which operates at the granularity where individual modules are distinguishable nodes. At the level of this document, `zone.backend` is a single node, and a Level 2 (Container) diagram produced from this section should render it as such; a future Level 3 (Component) diagram, sourced from `03-decomposition/`, is where module-to-module edges belong.

