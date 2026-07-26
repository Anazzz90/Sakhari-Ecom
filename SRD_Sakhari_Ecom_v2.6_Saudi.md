# Software Requirements Document (SRD) — v2.6
## Sakhari Ecom — Saudi Arabia Quick Commerce Platform

**Version:** 2.6
**Date:** July 24, 2026
**Status:** Draft
**Prepared For:** Solo, AI-assisted development
**Supersedes:** `SRD_QuickCommerce.md` (v1.1, Flutter single-app), `SRS_Sakhari_ReactNative_QuickCommerce.md` (v1.0, RN three-app draft), and prior single-store/Twilio-assumption drafts of this document (v2.0–2.2)

---

## 0. Document History & Why This Version Exists

| Version | Direction | Status |
|---|---|---|
| v1.0 (April 2026) | Flutter, single app, all roles (customer/rider/picker) via role-based routing | Superseded — kept as reference prototype |
| v1.1 (June 2026, SRS) | React Native rebuild, customer-facing app plus combined Worker App + Admin dashboard | Direction confirmed, this doc finalizes it |
| v2.0 (July 2026) | Two-app structure, scoped for solo/AI-assisted build, Gulf (UAE+KSA) market, low-thousands-orders-per-day target | Superseded — market narrowed |
| v2.1 (July 2026) | Saudi Arabia only, payment gateway/cloud region corrected for KSA, single-dark-store MVP assumption | Superseded — store model corrected |
| v2.2 (July 2026) | Multi-store from launch across Sakaka neighborhoods, store-scoped worker assignment, card-on-delivery payment added, both BNPL providers confirmed | Superseded — auth/SMS/DB provider corrected |
| v2.3 | Auth built in-house on NestJS+RDS (not Supabase — residency conflict), SMS/OTP switched to Unifonic (Twilio can't register Saudi domestic Sender IDs), DB explicitly RDS not raw EC2 | Superseded — operational/product specs expanded |
| v2.4 | Expands inventory, admin dashboard, operations workflow, promotions, pricing, reporting, analytics, and numbered business rules while preserving the finalized v2.3 architecture | Superseded — data requirements expanded |
| v2.5 | Expands data requirements, entity ownership, lifecycle ownership, logical relationships, integrity expectations, and retention requirements while keeping schema design out of the SRD | Superseded — lifecycle state machines added |
| **v2.6 (this doc)** | **Adds entity lifecycle state machines and valid state transitions** while preserving finalized architecture, business rules, and requirements scope | Active |

The original Flutter prototype (`rapiddash/`) remains as a historical reference under its legacy project name. The production platform is now named **Sakhari Ecom**.

---

## 1. Project Overview

### 1.1 Purpose
A quick-commerce platform delivering groceries and daily essentials in 10–20 minutes, launching in the Gulf (UAE/Saudi first), built and maintained by a solo developer using AI-assisted workflows.

### 1.2 Design Constraints (drives every decision below)
- **One developer.** Every extra service, language, or codebase is maintenance debt paid by one person.
- **Launch scale:** few cities, low thousands of orders/day — not Blinkit/Zepto scale. Infrastructure should be boring and cheap until traffic proves otherwise.
- **Saudi Arabia only at launch:** currency in SAR, Arabic is a first-class requirement (not deferred), payment gateway must support mada (Saudi's national debit network) and comply with SAMA/PDPL, COD expected to be a significant share of orders.

### 1.3 Target Users
| Actor | Role |
|---|---|
| Customer | Browses, orders, tracks deliveries |
| Picker | Picks and packs orders inside a dark store |
| Rider | Delivers packed orders to customers |
| Admin / Ops | Manages catalog, orders, workers, stores, dispatch, analytics |

---

## 2. Application Structure (Finalized)

| Application | Platform | Users | Notes |
|---|---|---|---|
| **Customer Mobile App** | iOS + Android (React Native) | Customers | Public App Store / Play Store listing. Lean bundle, only customer-relevant permissions. |
| **Customer Web App** | Web (Next.js) | Customers | Customer-facing website for browse, cart, checkout, order history, profile, addresses, support, and promotions. |
| **Worker App** | iOS + Android (React Native) | Pickers + Riders | Single app, mode selection after login. Distributed to onboarded staff (internal/TestFlight track initially, not publicly marketed). |
| **Admin/Ops Dashboard** | Web (React + Vite) | Admins, Ops, Support | Staff-only, browser-based. |

**Customer Platform definition:** The customer-facing product is the **Customer Platform**, consisting of the Customer Mobile App and Customer Web App. Both clients use the same backend APIs, OTP/JWT authentication flow, PostgreSQL database, order lifecycle, payment flows, inventory logic, promotion engine, support system, and business rules. There must never be separate business logic for web and mobile; platform differences are limited to presentation, analytics/device context, and client capabilities such as mobile push notifications.

**Decision rationale (recap):** Customer-facing clients are separated from worker/admin surfaces for bundle size, permission trust, attack-surface isolation, and independent release cadence. Picker and rider stay combined because they share nearly all non-task infrastructure (auth, profile, notifications, shift state) and splitting them buys nothing at this scale — revisit only once dedicated single-capability riders are recruited publicly at real volume.

### 2.1 Platform Architecture Overview

```
Customer Mobile App (React Native)
Customer Web App (Next.js)
Worker Mobile App (React Native)
Admin Dashboard (React + Vite)
        ↓
Backend API (NestJS)
        ↓
PostgreSQL (Amazon RDS) + Redis + Object Storage
```

The Backend API remains the single source of business logic. PostgreSQL remains the source of truth. Redis supports coordination/cache use cases, and object storage holds product/media assets.

### 2.2 Worker App Mode Rules
- AMR-001: Customer functionality is never present in the Worker App.
- AMR-002: Picker and rider modes share app shell, auth, profile, notifications, settings.
- AMR-003: Picker and rider remain separate task modes with separate screens/flows.
- AMR-004: A worker cannot switch modes while an active task is unresolved.
- AMR-005: Admin controls which modes a given worker account can access.
- AMR-006: Admin controls which store a worker account is assigned to; a worker cannot self-select or switch their own store.
- If a worker has only one capability, the app opens directly into that mode (no selection screen).

---

## 3. Market & Localization

### 3.1 Launch Market
**Saudi Arabia only.** UAE is explicitly out of scope for MVP — revisit as a Phase 2 expansion once KSA operations are validated. Currency: SAR.

### 3.1.1 Launch City: Sakaka, Al Jouf — Multi-Store
Population ~226,000 (2026 est.), compact ~100 km² urban core, density ~2,000/km². **Launch is multi-store, not single-store** — dark stores are placed across distinct neighborhoods (e.g. Qara, Lakhayath, Khatib, and others as ops requires) so each store's delivery radius stays tight enough for the 10–20 min SLA rather than one store trying to cover the whole city.

This is a real architecture decision, not a config tweak — see §3.1.2 for the dispatch model this requires.

- **Worker pool is small** relative to Riyadh/Jeddah. This reinforces (doesn't just permit) the combined picker+rider Worker App decision in §2 — a small gig-worker pool makes flexing one person between both roles more valuable, not less, since dedicated single-role workers may not be available early on. Multi-store makes this sharper: a worker pool split across several small stores means each store individually has even fewer dedicated riders, so flexibility matters more, not less.
- Google Maps coverage/routing quality in Saudi's northern cities is generally solid; confirmed to spot-check Sakaka geocoding/routing accuracy per-neighborhood during Phase 1 rather than assuming parity with Riyadh (open item, §15).
- Local vendor/supplier sourcing for the initial catalog: confirmed available.

### 3.1.2 Multi-Store Dispatch Model
- **Store as a first-class entity.** Each dark store has a name, location (lat/lng), and a service zone. For MVP, service zones are defined as a **simple radius from the store's coordinates** (not a hand-drawn polygon) — enough to route orders correctly without the extra engineering effort of geofence polygon management. Revisit polygon zones only if radius-based zones start overlapping in ways that misroute orders between neighborhoods.
- **Customer → store routing:** when a customer enters or selects a delivery address, the system determines which store's service zone contains that address and routes the order there. If the address falls in more than one store's zone (overlap), the nearer store wins. If it falls in none, the customer sees the existing "outside service zone" message (FR-CUST-008).
- **Worker → store assignment:** every worker (picker and/or rider capability) is assigned to exactly one home store by admin. A picker's order queue only ever shows orders for their assigned store. A rider's assignment pool draws from packed orders at their assigned store.
- **Cross-store fallback:** not built for MVP. If a store is short on available riders, that's a manual ops call (admin reassigns a worker's home store, or an order sits slightly longer) rather than automated cross-store dispatch logic. Automating this is a reasonable Phase 2 item once real store-count and volume data exists to design it against.

### 3.2 Payment Gateway — Correction from Earlier Draft
**Stripe is not usable here.** Saudi Arabia is not on Stripe's supported-country list — a Saudi-domiciled business cannot open a standard Stripe merchant account. Workarounds exist (incorporating abroad to get a foreign Stripe account) but that's an unnecessary complication for a business actually operating in KSA, and it wouldn't support mada, the network most Saudi debit cards run on.

**Decided: Moyasar**, based on a cost/fit comparison against the other three standard KSA options:

| Gateway | Mada fee | Monthly fee | Settlement | Best fit |
|---|---|---|---|---|
| **Moyasar (chosen)** | ~1.5–2.5% + 1 SAR | None | T+1 | Saudi-only startups, simplest integration, SAMA-licensed |
| HyperPay | ~1.5% + 1 SAR | ~250 SAR | Standard | Enterprise scale, multi-country (SA/UAE/Egypt/Jordan) |
| PayTabs | Variable | $49 or none | Standard | Full-GCC coverage incl. SADAD |
| Tap Payments | Lower than card | Variable | Standard | GCC-wide, Shopify-heavy merchants |

Moyasar wins for this project specifically because it has zero monthly fee, the fastest/simplest developer integration of the four, and — since the launch is Saudi-only — none of the other gateways' multi-country coverage is worth paying a premium for. Revisit only if expanding beyond KSA later.

Note: 15% Saudi VAT applies to gateway fees; confirm with a VAT advisor how Moyasar invoices processing fees for accurate cost modeling.

**BNPL (Phase 2, not MVP):** confirmed both **Tabby and Tamara** are used in this market — integrate both rather than picking one. The earlier research noted a second BNPL integration is a small marginal lift (~2 engineer-days) once the first is done, and offering both rather than one measurably improves conversion. Still sequenced after MVP core payment flow is stable, not in initial launch scope.

**Card-on-Delivery via rider-carried POS terminal:** since physical card machines are already available and will be issued to riders, Card-on-Delivery is a third payment method alongside online (Moyasar) and Cash on Delivery:
- FR-PAY-POS-001: Customers may select "Card on Delivery" at checkout; the rider completes the card charge on their handheld terminal at drop-off.
- FR-PAY-POS-002: The terminal transaction settles directly with the acquiring bank (outside the app's payment flow) — the app does not process this charge, it only records that it happened.
- FR-PAY-POS-003: At delivery completion, the rider marks the payment method actually used (cash, card, or already paid online) in the Worker App.
- FR-PAY-POS-004: Card-on-delivery transactions are included in the same daily reconciliation report as cash (§ COD reconciliation below), as a separate line so cash-in-hand and card settlement aren't conflated.
- Note: confirm with the existing terminal's provider (commonly Geidea in this market, ~75% POS share in KSA) whether it exposes a settlement API/report — if so, FR-PAY-POS-004 can cross-check terminal settlement data against app-recorded transactions automatically rather than relying on manual rider entry alone. Flagged as a Phase 1 technical spike, not assumed.

**Cash & Card-on-Delivery reconciliation:** since COD (cash) share is expected to be significant and card-on-delivery is now a third method, the platform needs a real reconciliation flow, not just "mark as paid on delivery":
- FR-PAY-COD-001: Riders record cash collected per delivery at the point of drop-off.
- FR-PAY-COD-002: The system tracks a running cash-in-hand balance per rider across their shift.
- FR-PAY-COD-003: Riders remit collected cash at shift end (or per configured threshold); admin/ops marks remittance received.
- FR-PAY-COD-004: Discrepancies between expected vs. remitted cash are flagged for ops review, not silently accepted.
- FR-PAY-COD-005: Daily reconciliation report available in the Admin Dashboard, broken down **per store and per rider**, covering cash and card-on-delivery separately.

### 3.3 Language — Assumption Flagged
The prior React Native SRS deferred "full multilingual UI/content" out of MVP scope. **For a Gulf launch, that assumption no longer holds** — Arabic is an expected baseline in this market, not a nice-to-have. This version elevates it:
- **MVP requires English + Arabic (RTL)** for the Customer Platform at minimum.
- Worker App and Admin Dashboard can launch English-only initially (internal tools, staff can be hired bilingual) and get Arabic in a fast-follow — this keeps solo-dev scope realistic while not compromising the customer-facing experience where it matters most.
- Localization approach carried over from the original Flutter SRD (still sound): static ARB/JSON files for UI strings, JSONB columns for multilingual product data, `Accept-Language` header with English fallback.

*(Flag this back to me if you'd rather launch English-only everywhere for MVP and add Arabic fast-follow — it's a real scope/timeline lever.)*

---

## 4. Finalized Tech Stack

### 4.1 Customer Platform
- **Customer Mobile App:** React Native + TypeScript, Expo (Expo Router) unless a specific native module forces a bare workflow
- **Customer Web App:** Next.js + TypeScript
- **Shared customer logic:** both clients consume the same backend APIs and must not fork business rules, checkout rules, payment rules, inventory logic, promotion logic, support logic, or order lifecycle behavior.
- **Server state:** TanStack Query
- **Local state:** Zustand
- **Forms/validation:** React Hook Form + Zod
- **Shared packages (monorepo):** `api client`, `domain types`, `ui`, `config` — shared across Customer Mobile App, Customer Web App, and Worker App where practical to cut duplicate work for the solo dev

### 4.2 Backend
- **Framework:** NestJS + TypeScript (reused/refactored from the existing legacy `rapiddash-api` codebase into the Sakhari Ecom API, not rewritten in Java/Go)
- **Real-time:** Socket.IO, in-process with the main API (no separate real-time service until proven necessary)
- **Auth:** Built directly in NestJS — phone OTP (customers, workers) via **Unifonic** for SMS delivery (not Twilio/Firebase, see §4.7 note); short-lived access + 7-day refresh JWTs issued and validated server-side; email/password + bcrypt/Argon2 + JWT for admin. Customer Mobile App and Customer Web App use the same OTP/JWT flow. Tokens are platform-independent; the backend distinguishes mobile and web only where analytics, session metadata, or device capabilities require it. Auth is not delegated to a third-party auth platform (e.g. Supabase) — see rationale in §4.4.

### 4.3 Admin Dashboard
- **Framework:** React + Vite + TailwindCSS

### 4.4 Data Layer
- **Primary DB:** **Amazon RDS for PostgreSQL**, in `me-central-2` (Riyadh) — managed, not self-hosted Postgres on a bare EC2 box. For a solo dev, RDS's automated backups/patching/failover are worth the modest extra cost versus the ops risk of hand-managing crash recovery alone. Single-AZ to start (cheaper); upgrade to Multi-AZ once real traffic justifies it. ACID guarantees + JSONB for multilingual content, as already decided.
- **Why not Supabase/BaaS instead of NestJS+RDS:** Supabase bundles Postgres+Auth+Storage into one platform, but its cloud regions are US/EU/Asia-Pacific only — nearest is Frankfurt, which reopens the exact PDPL cross-border residency problem `me-central-2` was chosen to avoid. It would also mean partially replacing the NestJS backend already decided on, not adding to it. Worth reconsidering only if a future region announcement changes this.
- **Cache / locks / geospatial:** Redis (session cache, inventory locks, GEO commands for rider location — one system covers all three, no ScyllaDB needed)
- **Analytics:** Postgres rollup tables/views for MVP; revisit ClickHouse only once query volume actually hurts

### 4.5 Infrastructure
- **Cloud / region:** AWS, region **`me-central-2` (Riyadh)** — AWS's native Saudi Arabia region, live since January 2026. This matters for two concrete reasons, not just "use AWS because the old doc said so":
  - **Latency:** an in-Kingdom region beats DigitalOcean's nearest options (Frankfurt/Bangalore) for the 300ms API / 5s tracking targets in §6.
  - **PDPL data residency:** Saudi's Personal Data Protection Law defaults to keeping residents' personal data in-Kingdom unless specific cross-border transfer conditions are met. Hosting in `me-central-2` avoids having to build and maintain that compliance case at all.
  - **DigitalOcean for local dev/prototyping only** — genuinely fine and cheaper for that, just not for anything touching real customer data at launch.
- **Compute:** EC2 or Fargate + Docker Compose — **no Kubernetes/EKS** until multiple services genuinely need independent scaling
- **CI/CD:** GitHub Actions
- **CDN:** CloudFront, WebP images
- **Monitoring:** Sentry (crashes/errors) + Grafana Cloud free tier (metrics) — no self-hosted Prometheus/Loki stack for a solo dev to babysit

### 4.6 Explicitly Cut From the Blinkit-Scale Proposal
Kafka, Debezium, ClickHouse (for now), ScyllaDB, Kubernetes/EKS, native Kotlin rider app, separate Java/Go/Python backend services, dual maps providers, self-hosted observability stack. Rationale for each is in the earlier analysis — all are real patterns, just for a scale and team size this project isn't at yet.

### 4.7 Third-Party Integrations
- **Payments:** Moyasar (§3.2) for card/mada/Apple Pay, plus Cash on Delivery and Card on Delivery with reconciliation flow
- **Maps/Routing/Geocoding/ETA:** Google Maps API only
- **Push:** Firebase Cloud Messaging
- **SMS/OTP:** **Unifonic**, not Twilio — correction from earlier drafts. Twilio cannot register Sender IDs for domestic Saudi brands (CITC regulations prohibit reselling domestic SMS traffic), meaning unregistered messages get blocked at the carrier level. Unifonic is Saudi-headquartered, handles local Sender ID registration directly (~299 SAR/year), and is dramatically cheaper at volume than Twilio would be even if it worked (rough comparisons put 100k OTPs/month at ~$700 with a local provider vs. ~$19,000+ with Twilio).
- **Image storage:** Firebase Storage or S3

---

## 5. Functional Requirements

### 5.1 Authentication & Authorization
- FR-AUTH-001: Phone OTP login for customers and workers.
- FR-AUTH-002: Email/password login for admin.
- FR-AUTH-003: Access tokens (short-lived) + refresh tokens issued on login.
- FR-AUTH-004: Server-side RBAC for customer, picker, rider, admin, ops, support.
- FR-AUTH-005: Worker accounts carry one or both capabilities: picker, rider.
- FR-AUTH-006: Single-capability workers skip the mode-selection screen.
- FR-AUTH-007: Dual-capability workers see mode selection after login.
- FR-AUTH-008: Worker accounts are created by admin only for MVP (no self-registration — reduces fraud/vetting risk while the ops process is still manual).
- FR-AUTH-009: Customer Mobile App and Customer Web App authenticate through the same customer OTP/JWT flow and share the same customer identity.
- FR-AUTH-010: Customer tokens are platform-independent; platform/device metadata may be recorded for analytics or notification capability but must not create separate customer business rules.

### 5.1.1 Store & Dispatch (Multi-Store)
- FR-STORE-001: Admin can create/edit dark stores — name, location (lat/lng), service radius.
- FR-STORE-002: Each customer order is routed to exactly one serving store, determined by matching the delivery address against store service zones; nearer store wins on overlap.
- FR-STORE-003: If a delivery address falls outside every store's service zone, the customer sees "outside service area" (ties to FR-CUST-008).
- FR-STORE-004: Every worker (picker and/or rider capability) is assigned to exactly one home store by admin.
- FR-STORE-005: A picker's order queue shows only orders for their assigned store.
- FR-STORE-006: A rider's assignment pool draws only from packed orders at their assigned store for MVP; cross-store reassignment during shortages is a manual admin action, not automated (§3.1.2).
- FR-STORE-007: Admin dashboard shows per-store views: active orders, worker roster, inventory, and the reconciliation report (FR-PAY-COD-005).
- FR-STORE-008: Store temporary closure can be toggled by admin/ops; affected customers see unavailable status before checkout.

### 5.2 Customer Platform
- FR-CUST-000: Customer Platform consists of Customer Mobile App and Customer Web App. Both use the same backend APIs, authentication flow, customer profile, addresses, server-side cart, order history, favorites where implemented, notifications, promotions, support tickets, payment flows, inventory rules, and order lifecycle.
- FR-CUST-001: System determines and shows the serving store (or "unavailable") based on the customer's address — not a manual store picker.
- FR-CUST-002–004: Browse categories, search, view product detail (image, price, unit, stock, bilingual description) — scoped to the customer's serving store's inventory.
- FR-CUST-005–006: Server-side cart management with live subtotal/delivery fee/discount/total recalculation. A customer can add products on the website, open the mobile app, and continue the same cart; mobile changes must also be reflected on the website.
- FR-CUST-007–008: Manage saved addresses; warn if outside every store's service zone.
- FR-CUST-009: Place order via Moyasar (mada/card/Apple Pay), Cash on Delivery, or Card on Delivery.
- FR-CUST-010–013: Order confirmation, live status, rider tracking post-assignment, order history.
- FR-CUST-014: Contact support for active/past orders.
- FR-CUST-015: Approve/reject substitution requests for out-of-stock items.
- FR-CUST-016: Full Customer Platform UI available in English and Arabic (RTL) at launch.
- FR-CUST-017: Order history, favorites where implemented, addresses, notification history, promotions, support tickets, and customer profile data are synchronized across Customer Mobile App and Customer Web App.

### 5.3 Worker App — Shared
- FR-WORK-001–007: Login; show only available modes; mode switching only outside active tasks; profile/shift/availability/support visible; push notifications for assignments; clear online/offline/busy states; no cross-mode task conflicts.
- FR-WORK-008: Worker App displays the worker's assigned home store and does not show tasks from other stores.
- FR-WORK-009: Worker App requires explicit shift start before assignments appear.
- FR-WORK-010: Worker App records task timestamps for assignment, claim, pick start, packing complete, rider accept, pickup, arrival, delivery completion, and failure.

### 5.4 Picker Mode
- FR-PICK-001–012: SLA-sorted order queue scoped to the picker's assigned store (FR-STORE-005); claim order (no double-claim); item details with aisle/shelf where available; per-item confirmation; out-of-stock marking triggers substitution flow; picker may continue picking other items while awaiting substitution approval; packing completion requires all items resolved; completing packing sets order to "packed" and eligible for rider assignment; SLA timers/warnings visible.
- FR-PICK-013: Picker claim creates a picking session linked to the order, picker, and store.
- FR-PICK-014: If a picker does not begin picking within the configured timeout, the claim expires and the order returns to the store queue.
- FR-PICK-015: If an active picker session stalls beyond the timeout, ops receives an alert and can release/reassign the order.
- Barcode scanning: deferred, consistent with prior docs — manual confirmation for MVP.

### 5.5 Rider Mode
- FR-RIDE-001–00X: Online/offline toggle; receive assignments from the rider's assigned store (FR-STORE-006); accept/reject within a timeout (auto-expire and reassign); navigate via deep link to Google Maps (no embedded navigation for MVP); background location sharing while online/on active delivery; OTP-based proof of delivery; card-on-delivery charge via handheld terminal where selected (FR-PAY-POS-001–004); earnings dashboard.
- FR-RIDE-010: Rider arrival and delivery completion timestamps are captured for SLA reporting.
- FR-RIDE-011: Rider can mark failed delivery only with a reason code and optional support note.

### 5.6 Inventory Management

Inventory is store-scoped from launch. A product can exist globally in the catalog while available quantity, reserved quantity, low-stock threshold, price override, shelf location, and availability status vary by store. This is required by the multi-store architecture; a single global stock value would mislead customers and create impossible picker queues.

#### 5.6.1 Inventory Data Model
- FR-INV-001: Inventory is represented by `inventory_item` or equivalent rows keyed by `store_id + product_id`.
- FR-INV-002: Each inventory row tracks at minimum: on-hand quantity, reserved quantity, available quantity, low-stock threshold, shelf/aisle metadata, stock status, last sync timestamp, and last adjusted by.
- FR-INV-003: Available quantity is derived as `on_hand - reserved`, not manually entered.
- FR-INV-004: Inventory changes are append-only in an `inventory_ledger` / audit log, with the current row updated transactionally for fast reads.
- FR-INV-005: Inventory rows support status values: in stock, low stock, out of stock, disabled, and pending reconciliation.

#### 5.6.2 Stock Reservation Workflow
- FR-INV-006: During checkout, the backend validates every cart line against the serving store's available inventory.
- FR-INV-007: Order creation reserves stock immediately after payment intent validation for online payments, or immediately at order creation for COD/Card-on-Delivery.
- FR-INV-008: Reservation increases `reserved_quantity` but does not reduce `on_hand_quantity`.
- FR-INV-009: Reservation is linked to the order and expires automatically if order creation or payment confirmation does not complete.
- FR-INV-010: Checkout cannot succeed if any required item cannot be reserved at the requested quantity.

Decision rationale: reserving before picking prevents overselling during customer checkout without prematurely reducing physical stock. Deduction waits until the picker confirms what was actually packed.

#### 5.6.3 Stock Deduction Timing
- FR-INV-011: Stock is deducted from `on_hand_quantity` when packing is completed, not when the customer first adds an item to cart.
- FR-INV-012: For each packed item, the system reduces both on-hand and reserved quantity by the packed quantity.
- FR-INV-013: If a picker marks a lower quantity, unavailable item, or approved substitute, the reservation is adjusted before packing completes.
- FR-INV-014: Cancelled orders release reservations if not yet packed; if already packed, ops must confirm whether stock returns to sellable inventory.

#### 5.6.4 Inventory Locking & Race Condition Handling
- FR-INV-015: Checkout stock validation/reservation uses Redis locks keyed by `store_id:product_id`.
- FR-INV-016: Lock TTL must be short and bounded; failure to acquire a lock returns a retryable checkout error rather than risking oversell.
- FR-INV-017: The database update that adjusts reserved quantity must run inside a PostgreSQL transaction and re-check available quantity before commit.
- FR-INV-018: Redis locks are used as a coordination guard; PostgreSQL remains the source of truth.
- FR-INV-019: Idempotency keys are required for order creation so customer retries cannot duplicate stock reservations.

Decision rationale: Redis is already part of the finalized stack for locks/session/geospatial support. Using it here avoids adding a new queue or distributed transaction system while still covering the real race condition.

#### 5.6.5 Picker Timeout & Reservation Release
- FR-INV-020: If a picker claim expires before picking starts, existing order reservations remain intact and the order returns to the same store queue.
- FR-INV-021: If an order is cancelled before packing, all reservations are released immediately.
- FR-INV-022: If substitution approval times out, the affected item is removed or refunded according to the order policy; unrelated item reservations remain.
- FR-INV-023: Admin can manually release a stuck reservation only through an audited support action.

#### 5.6.6 Damaged, Expired, Returned, and Cancelled Inventory
- FR-INV-024: Damaged stock is removed from sellable on-hand through a manual adjustment reason code: damaged.
- FR-INV-025: Expired stock is removed through reason code: expired; expiry tracking is MVP-manual unless supplier/batch tracking is later added.
- FR-INV-026: Returned items are not automatically restored to sellable inventory. Ops must classify them as sellable, damaged, expired, or discard.
- FR-INV-027: Cancelled pre-pick orders restore reserved stock automatically.
- FR-INV-028: Cancelled post-pack orders require ops confirmation because the product has physically left normal shelf flow.

#### 5.6.7 Manual Stock Adjustments & Reconciliation
- FR-INV-029: Admin/ops can perform manual stock adjustments by store/product with mandatory reason, note, and user attribution.
- FR-INV-030: Bulk inventory upload supports CSV/XLSX for initial setup and periodic corrections.
- FR-INV-031: Daily store closing includes inventory reconciliation for selected fast-moving and exception items.
- FR-INV-032: Inventory mismatch creates an audit entry and optionally sets the item to pending reconciliation.
- FR-INV-033: Low-stock thresholds are store-specific and trigger dashboard alerts.
- FR-INV-034: Out-of-stock items are hidden or marked unavailable in the Customer Platform for the affected store only.

#### 5.6.8 Store-Specific Synchronization
- FR-INV-035: Store inventory sync imports supplier/POS counts into store-scoped rows without changing the global product catalog.
- FR-INV-036: Sync jobs must produce a success/failure report per store.
- FR-INV-037: Sync conflicts do not overwrite active reservations; they update on-hand counts and recalculate availability.
- FR-INV-038: Every sync result is visible in Admin under Inventory Management with timestamp, source, and changed item count.

### 5.7 Out-of-Stock & Substitution Flow
- Picker suggests substitute or flags item as unavailable → push notification to customer for approve/reject → backend recalculates total and processes partial refund if needed.
- FR-SUB-001: Substitution requires customer approval unless a future explicit customer preference says otherwise.
- FR-SUB-002: Substitute items are priced according to §5.11.11.
- FR-SUB-003: If the customer does not respond within the configured timeout, the unavailable item is removed and refund/total adjustment follows payment-method rules.

### 5.8 Admin/Ops Dashboard Product Specification

The Admin/Ops Dashboard is the operational control plane for the business. It must be complete enough for daily store operations, support, and reconciliation without requiring direct database edits. It stays a React + Vite + TailwindCSS web app; this section expands scope, not technology.

#### 5.8.1 Dashboard Overview
- Shows active orders, delayed orders, open support tickets, online workers, low-stock alerts, store status, payment exceptions, and reconciliation exceptions.
- Supports global view and per-store filtered view.
- Uses operational status cards and tables, not marketing-style analytics.

#### 5.8.2 Order Management
- View, search, filter, and inspect orders by status, store, customer, rider, picker, payment method, and time window.
- Manual actions: cancel eligible order, trigger refund workflow, reassign picker/rider where allowed, mark support escalation, and view event timeline.
- Order timeline must show customer placement, store assignment, reservation, picking, packing, rider assignment, pickup, delivery, payment, and refund events.

#### 5.8.3 Inventory Management
- Store-scoped inventory list with on-hand, reserved, available, low-stock threshold, status, and shelf metadata.
- Manual adjustments with reason codes and audit logging.
- Bulk import/export for store inventory.
- Low-stock and out-of-stock exception views.
- Reconciliation workflow for damaged, expired, returned, and mismatched inventory.

#### 5.8.4 Product Catalog
- Manage global product data: name, Arabic/English descriptions, images, barcode/SKU where available, category, unit, taxability, active status.
- Product catalog changes do not automatically imply store availability; store inventory controls sellability.
- Image uploads target S3/Firebase Storage as already allowed in §4.7.

#### 5.8.5 Promotions
- MVP: create and manage promo codes with usage limits, date ranges, minimum spend, store eligibility, and customer eligibility.
- Future-ready admin model should not block percentage, fixed, category, bundle, free-delivery, first-order, and supplier-funded promotion types described in §5.10.

#### 5.8.6 Store Management
- Create/edit dark stores, radius, operating hours, temporary closure state, contact details, and active worker roster.
- View per-store order load, inventory exceptions, delayed orders, and reconciliation status.

#### 5.8.7 Worker Management
- Create worker accounts, assign home store, grant picker/rider capabilities, activate/deactivate accounts, and review shift history.
- Worker capability changes are audited.

#### 5.8.8 Rider Management
- View rider online/offline status, active delivery, current store, cash-in-hand balance, card-on-delivery records, failed deliveries, and productivity metrics.
- Manual reassignment remains an admin action for MVP; automated cross-store logic is deferred.

#### 5.8.9 Customer Management
- Search customers by phone, name, order history, saved addresses, and support tickets.
- Support actions must avoid exposing more personal data than needed.

#### 5.8.10 Support Tickets
- Create/view tickets linked to customer, order, rider, or store.
- Track status, priority, reason code, owner, and resolution notes.
- Support ticket activity is included in the order/customer timeline.

#### 5.8.11 Refund Management
- View refund eligibility, refund reason, original payment method, calculated amount, approval status, and gateway result.
- Online refunds use Moyasar APIs where supported; COD/Card-on-Delivery refunds are recorded as manual cash/card resolution tasks.

#### 5.8.12 Reports
- Generate daily/weekly/monthly reports for sales, operations, inventory, finance, and worker productivity.
- Reports use Postgres rollups/views for MVP, not ClickHouse.

#### 5.8.13 Analytics
- Operational analytics show trends and exceptions: peak order times, SLA compliance, cancellations, failed deliveries, payment failures, low-stock frequency, and store comparison.
- Analytics are decision-support for ops, not a separate data platform.

#### 5.8.14 Cash & COD Reconciliation
- Per-rider shift balances for cash collected, card-on-delivery recorded, remitted cash, expected balance, and discrepancy.
- Per-store daily reconciliation summary.
- Discrepancies require reason code and admin acknowledgement.

#### 5.8.15 System Settings
- Manage service radius values, delivery fee, minimum order amount, free-delivery threshold, picker/rider timeout values, support reason codes, and operational opening hours.
- Settings changes are audited and effective timestamps are stored.

#### 5.8.16 Roles & Permissions
- RBAC distinguishes admin, ops manager, store manager, inventory manager, support, finance/reconciliation, and read-only roles.
- Permission checks must be enforced server-side, not only hidden in UI.

#### 5.8.17 Audit Logs
- All admin changes to stores, workers, inventory, prices, promotions, refunds, orders, and system settings are logged.
- Audit logs are filterable by user, entity, action, store, and time window.

### 5.9 Payments
- FR-PAY-001: COD supported and expected to be the majority payment method at launch — see §3.2 for the reconciliation flow (FR-PAY-COD-001–005).
- FR-PAY-002: Online payment via Moyasar (or equivalent KSA-licensed gateway), supporting mada, cards, and Apple Pay.
- FR-PAY-003: No raw card data stored (PCI-compliant handling only).
- FR-PAY-004: Payment verification via gateway signature/receipt, never client-supplied order IDs alone.
- FR-PAY-005–006: Retry on failure; refund workflow from admin/support tooling.
- FR-PAY-007: Payment status changes are append-only events linked to order and payment records.
- FR-PAY-008: Payment gateway outage keeps COD/Card-on-Delivery available if store operations are otherwise online.

### 5.10 Promotions

MVP supports promo codes only. The data model should still avoid boxing the product into a promo-code-only design because grocery/quick-commerce promotions commonly become store, category, bundle, and supplier-funded campaigns.

#### 5.10.1 MVP Promo Codes
- FR-PROMO-001: Admin can create promo codes with code, discount type, value, validity window, usage limit, per-customer limit, minimum spend, active status, and optional store eligibility.
- FR-PROMO-002: Promo code validation runs server-side at cart/checkout and again at order placement.
- FR-PROMO-003: Promo usage is recorded only after successful order creation.
- FR-PROMO-004: Promo code discounts must appear as a separate line in the order price breakdown.

#### 5.10.2 Future Promotion Types
- Percentage discounts: reduce eligible item subtotal by a configured percentage.
- Fixed discounts: subtract a fixed SAR amount from eligible subtotal.
- Buy X Get Y: add or discount qualifying item(s) when quantity rules are met.
- Bundle pricing: apply a fixed bundle price when all configured products are present.
- Category discounts: apply to products in selected categories.
- Flash sales: time-limited promotion with stricter start/end validation.
- Time-based promotions: scheduled by day/time, useful for demand shaping.
- First-order discounts: customer eligibility based on completed order history.
- Delivery fee discounts: reduce or waive delivery fee separately from product subtotal.
- Minimum spend offers: require subtotal threshold before discount applies.
- Supplier-funded promotions: track funding source for reporting without changing checkout flow.

Decision rationale: these promotion types can be modeled as rules evaluated against cart context. They do not require Kafka, a separate pricing service, or a new database in MVP; NestJS + PostgreSQL are sufficient.

### 5.11 Pricing Rules

Pricing must be deterministic and explainable to customers, support, and finance. The backend is the source of truth; clients only display calculated values.

- FR-PRICE-001: Product base price is stored on the product/store price record and displayed in SAR.
- FR-PRICE-002: Store-specific pricing is allowed because supplier cost, availability, and local promotions can vary by store.
- FR-PRICE-003: Delivery fee is calculated from configured store/customer/order rules; MVP may use a flat fee per store or global setting.
- FR-PRICE-004: Packaging fee, if enabled, appears as a separate line item.
- FR-PRICE-005: VAT is calculated according to Saudi VAT requirements and shown in order totals. Confirm product/category VAT treatment with accounting advice before launch.
- FR-PRICE-006: Coupon deductions apply after eligible subtotal validation and before final total.
- FR-PRICE-007: Promotional discounts are represented separately from promo-code discounts where needed for reporting.
- FR-PRICE-008: Free delivery threshold applies to eligible subtotal after excluded items are removed and before delivery fee is added.
- FR-PRICE-009: Minimum order amount is checked before payment; customer sees the remaining amount needed.
- FR-PRICE-010: Refund calculation uses the final paid/owed line-item totals, not current catalog prices.
- FR-PRICE-011: Substitution pricing: if substitute is cheaper, customer pays lower amount or receives partial refund; if substitute is more expensive, customer must approve the higher price before it is packed.
- FR-PRICE-012: Price snapshots are stored on order items so later catalog price changes do not mutate historical orders.

### 5.12 Notifications
- Business notification events are platform-independent; delivery channel depends on recipient and client capability.
- Mobile push (FCM) for Customer Mobile App order lifecycle events and Worker App task assignment.
- Browser notifications are a future Customer Web App channel, not required for MVP unless explicitly prioritized later.
- SMS (Unifonic) for OTP and delivery confirmation.
- Email is a future channel.
- In-app/in-platform notification history is shared across the Customer Platform where applicable.
- Notification send failures are logged and retried where safe; order state must not depend solely on a push, browser, SMS, or email notification being delivered.

### 5.13 Location & Tracking
- Rider location shared while online/on active delivery, updated at a throttled interval (5s, matching prior cost-optimization decision).
- ETA via Google Maps.
- Geocoding results cached to control API cost.
- Location history retention should be limited to operational needs and PDPL review.

---

## 6. Non-Functional Requirements

| Category | Requirement |
|---|---|
| Performance | Catalog APIs < 300ms p95; order placement < 2s p95 excl. gateway; worker task screens < 1s p95; tracking updates visible within 5s |
| Reliability | Idempotent order creation; concurrency-safe stock reservation/deduction; no double picker-claim or double rider-assignment; graceful degradation if maps/payment/notification providers are down |
| Security | Auth required on all non-public endpoints; server-side RBAC; admin 2FA available; no secrets in source control; payment gateway signature verification; phone numbers not exposed unnecessarily |
| Data Protection (PDPL) | Personal data of Saudi residents hosted in-Kingdom (`me-central-2`) by default; any cross-border transfer (e.g., third-party analytics/SaaS tools) reviewed against PDPL's cross-border transfer conditions before adoption; data breach notification process capable of meeting the 72-hour SDAIA reporting window; this is a general-business PDPL obligation (via SDAIA), distinct from the stricter in-Kingdom hosting mandate that applies specifically to SAMA-licensed financial institutions — Sakhari Ecom is not itself a financial institution, but should still default to in-Kingdom hosting given the customer PII and order data involved |
| Usability | Minimal-step checkout across Customer Mobile App and Customer Web App; large tap targets in picker mode; one-handed, navigation-first rider mode; visible success/failure states; retry-friendly on poor connectivity |
| Compatibility | Recent iOS/Android per Expo support window; current Chrome/Edge/Safari/Firefox for dashboard; versioned backend APIs |
| Observability | Key business events logged; Sentry for crashes; core metrics (order volume, delay rate, assignment failures, payment failures, inventory exceptions, payment failures) visible; admin actions audited |
| Operations | Store opening/closing, shift state, cash reconciliation, inventory reconciliation, and outage states must be visible in Admin without direct DB access |

---

## 7. Operations Workflow

This chapter describes how Sakhari Ecom operates during a normal day. It does not introduce new architecture; it clarifies the workflows the apps and Admin Dashboard must support.

### 7.1 Store Opening Workflow
1. Store manager/admin opens the store in Admin Dashboard.
2. Staff log into the Worker App.
3. Workers start their shift and set availability.
4. Store inventory sync/reconciliation runs for priority items.
5. Low-stock/out-of-stock exceptions are reviewed before accepting orders.
6. Cash drawer or rider cash float is prepared if the store uses one.
7. Store status changes from closed/preparing to open.

Expected system behaviour:
- Customers cannot place orders for a closed store.
- Workers cannot receive assignments before shift start.
- Admin sees store readiness: staff online, inventory sync status, cash setup, and open exceptions.

### 7.2 Order Lifecycle Workflow
1. Customer enters/selects address.
2. Backend assigns serving store by radius and nearest-store overlap rule.
3. Customer browses store-scoped catalog and inventory.
4. Customer checks out; backend reserves inventory and creates order.
5. Store picker queue receives the order.
6. Picker claims, picks, resolves substitutions, and packs.
7. Order becomes packed and eligible for rider assignment.
8. Rider accepts, picks up, navigates through Google Maps deep link, and delivers.
9. Delivery is validated by OTP.
10. Payment is confirmed online, collected as cash, or recorded as card-on-delivery.
11. Order completion feeds reconciliation, inventory, SLA, and reporting records.

### 7.3 Store Closing Workflow
1. Admin/manager stops new order intake by closing the store.
2. Active orders are completed, cancelled, or explicitly handed off by ops.
3. Riders complete remittance for cash collected.
4. Card-on-delivery records are reviewed against terminal totals where available.
5. Inventory reconciliation is performed for exception/fast-moving items.
6. Worker shifts are completed.
7. Daily reports are generated per store.

Expected system behaviour:
- Closing a store prevents new checkout but does not hide active operational queues.
- Reconciliation exceptions remain open until acknowledged.
- Daily reports include cash, card-on-delivery, online payment, refund, inventory, and SLA summaries.

### 7.4 Operational Failure Scenarios

| Scenario | Expected system behaviour |
|---|---|
| Internet outage at store | Worker/Admin actions may fail safely; no local order mutation is considered final until backend confirms. Store can be temporarily closed by ops if outage persists. |
| Payment gateway outage | Online payment is disabled or shows retry; COD/Card-on-Delivery may remain available if store operations are healthy. |
| Worker absent | Admin marks worker unavailable and reassigns tasks within same store. Cross-store help remains manual for MVP. |
| Rider unavailable | Packed orders wait in store pool; admin can reassign rider home store manually or temporarily pause intake. |
| POS terminal failure | Card-on-Delivery can be switched to cash where customer agrees; otherwise support cancels or reschedules according to business rules. |
| Store temporarily closed | Address/store routing returns unavailable for that store; existing active orders continue unless manually cancelled. |
| Inventory mismatch | Item is flagged pending reconciliation; customer-facing availability is reduced or disabled until corrected. |

---

## 8. Reporting & Analytics

Reporting stays inside PostgreSQL rollup tables/views for MVP. This is sufficient for low-thousands-orders-per-day and avoids a separate analytics database before the business needs it.

### 8.1 Sales Reports
- Daily sales by store, payment method, category, and order status.
- Weekly and monthly sales rollups.
- Store comparison for revenue, order count, average order value, and cancellation rate.

### 8.2 Operations Reports
- Average picking time by store, picker, and time window.
- Average delivery time by rider/store.
- SLA compliance and delay buckets.
- Cancelled orders by reason.
- Failed deliveries by reason and rider/store.

### 8.3 Inventory Reports
- Low-stock report by store.
- Out-of-stock frequency by product/store.
- Fast-moving products.
- Dead stock / low-turnover products.
- Inventory valuation using current cost/price assumptions where available.
- Damaged/expired/returned inventory adjustments.

### 8.4 Finance Reports
- Cash reconciliation by rider/store/day.
- Card-on-delivery reconciliation by rider/store/day.
- Online payment reconciliation against Moyasar status.
- Refund reports by method/reason/status.
- Revenue, discounts, delivery fees, packaging fees, VAT, and net payable summary.

### 8.5 Worker Reports
- Orders completed by picker/rider.
- Average picking speed.
- Rider productivity and delivery completion rate.
- Shift statistics: online time, assigned tasks, accepted/rejected tasks, late tasks.

### 8.6 Analytics Implementation Notes
- Rollup jobs can run on a schedule from the NestJS backend or a simple worker process inside the same deployment model.
- Reports should allow CSV export for accounting/ops use.
- Raw event tables remain the source of truth; rollups are rebuildable.
- ClickHouse remains deferred until query volume or retention requirements justify it.

---

## 9. Business Rules

Business rules are numbered because they must be testable and visible in support/admin workflows.

- BR-001: Orders may be cancelled by the customer only before picking has started, unless support/admin overrides.
- BR-002: Orders cancelled before packing release reserved inventory automatically.
- BR-003: Orders cancelled after packing require ops review before inventory is restored.
- BR-004: Negative available inventory is not allowed for sellable stock.
- BR-005: Checkout must fail if any cart item cannot be reserved from the serving store.
- BR-006: A customer address outside all service radii cannot place an order.
- BR-007: If multiple store radii cover an address, the nearest store serves the order.
- BR-008: Maximum delivery radius is configured per store and enforced server-side.
- BR-009: Store closed state blocks new checkout for that store.
- BR-010: Online payment failure does not create a confirmed order unless a valid payment confirmation is received.
- BR-011: Payment gateway signatures/receipts are required for online payment confirmation.
- BR-012: COD and Card-on-Delivery orders are payable at delivery and reconciled at shift/day close.
- BR-013: Rider wait time at customer location is capped by a configurable threshold; after that, rider contacts support or marks failed delivery with reason.
- BR-014: OTP is required for normal delivery completion.
- BR-015: Failed OTP validation requires support action or approved exception reason.
- BR-016: Customer no-show creates a failed-delivery record and support follow-up.
- BR-017: A rider cannot hold more than one active delivery unless batching is explicitly introduced later.
- BR-018: A picker cannot claim the same order twice or claim an order already claimed by another active picker session.
- BR-019: Picker claim timeout returns the order to the same store queue.
- BR-020: Rider assignment timeout returns the packed order to the same store rider pool.
- BR-021: Automated cross-store rider reassignment is not allowed in MVP.
- BR-022: Manual cross-store reassignment requires admin action and audit log.
- BR-023: Worker must have an active shift before receiving picker/rider tasks.
- BR-024: Worker cannot switch app mode while an active task is unresolved.
- BR-025: Worker availability controls assignment eligibility; offline workers cannot receive new assignments.
- BR-026: Refund eligibility depends on order status, payment method, cancellation reason, delivery failure reason, and substitution outcome.
- BR-027: Refund amount must be calculated from the order price snapshot, not current product prices.
- BR-028: Substitution that increases price requires customer approval.
- BR-029: Substitution that decreases price triggers total reduction or partial refund.
- BR-030: If customer does not respond to substitution request before timeout, unavailable item is removed unless support overrides.
- BR-031: Cash collected by rider must equal expected COD amount minus acknowledged exceptions.
- BR-032: Cash discrepancy requires admin acknowledgement and reason code.
- BR-033: Card-on-Delivery amount recorded by rider must be reconciled separately from cash.
- BR-034: Admin changes to inventory, prices, promotions, store settings, worker permissions, refunds, and order status require audit logs.
- BR-035: Product availability is store-specific; out-of-stock in one store does not remove the product from another store.
- BR-036: Promotion eligibility is validated at checkout and again at order creation.
- BR-037: Minimum order amount is enforced before payment.
- BR-038: Free delivery threshold is calculated according to configured subtotal rules.
- BR-039: Support/admin overrides must preserve the original event history rather than mutating past events.
- BR-040: Any operational failure mode must fail closed for payment, inventory, and assignment integrity.

---

## 10. Entity Lifecycle State Machines

This chapter defines the allowed lifecycle states for major business entities. It is requirements-level only: it defines valid state movement, ownership, validation, auditability, and history expectations. It does not define APIs, database design, event infrastructure, or implementation mechanics.

### 10.1 Order State Machine

The Order Module owns order state. Picking, Delivery, Payment, Refund, Inventory, and Support modules may request transitions only through business workflows that preserve the order timeline.

| State | Purpose | Moved by / trigger | Allowed next states | Forbidden transitions |
|---|---|---|---|---|
| Created | Order intent exists after checkout validation starts. | Checkout flow after cart, address, pricing, and store validation. | Payment Pending, Confirmed, Cancelled | Picking, Packed, Delivered, Completed |
| Payment Pending | Online payment is awaiting confirmation. | Payment Module after online payment selection. | Payment Failed, Confirmed, Cancelled | Inventory Reserved, Picking, Delivered |
| Payment Failed | Online payment attempt failed or expired. | Payment Module after gateway failure/timeout. | Payment Pending, Cancelled | Confirmed without valid payment, Picking, Delivered |
| Confirmed | Order is accepted for fulfillment. | Payment confirmation, COD confirmation, or Card-on-Delivery confirmation. | Inventory Reserved, Cancelled | Delivered, Completed, Refunded |
| Inventory Reserved | Store stock has been reserved for the order. | Inventory Module after successful reservation. | Picking, Cancelled, Awaiting Customer Response | Delivered, Completed |
| Picking | Picker is actively resolving items. | Picking Module after picker claim/start. | Awaiting Customer Response, Packing, Cancelled | Awaiting Rider, Out for Delivery, Completed |
| Awaiting Customer Response | Customer decision is needed for substitution/unavailable item. | Picking Module after picker marks item unavailable or substitute proposed. | Picking, Packing, Cancelled | Packed, Delivered, Completed |
| Packing | All items are resolved and being prepared for handoff. | Picking Module after picking completion. | Packed, Cancelled | Picking without explicit correction, Delivered |
| Packed | Order is packed and ready for rider assignment. | Picking Module after packing completion and inventory deduction. | Awaiting Rider, Cancelled | Picking, Delivered, Completed |
| Awaiting Rider | Packed order is waiting for rider acceptance. | Delivery Module after packed order enters rider pool. | Out for Delivery, Cancelled | Picking, Completed |
| Out for Delivery | Rider has accepted/picked up and is delivering. | Delivery Module after rider pickup. | Delivered, Failed Delivery, Cancelled | Picking, Packed, Awaiting Customer Response |
| Delivered | Rider has completed drop-off and proof is accepted. | Delivery Module after OTP/approved delivery completion. | Completed, Refund Pending | Picking, Cancelled, Returned without support review |
| Completed | Fulfillment and payment/reconciliation requirements are closed. | Order/Payment modules after delivery and payment completion. | Refund Pending | Picking, Packed, Cancelled |
| Cancelled | Order will not be fulfilled. | Customer, support, admin, payment failure, timeout, or ops action where allowed. | Refund Pending, Refunded | Picking, Packed, Delivered, Completed without support correction |
| Refund Pending | Refund is required or under review. | Refund Module after cancellation, substitution, failed delivery, or support decision. | Refunded, Completed | Picking, Packed, Delivered as normal flow |
| Refunded | Refund has been completed or manual refund resolution recorded. | Refund Module after settlement/finance action. | Completed | Picking, Packed, Out for Delivery |
| Failed Delivery | Delivery could not be completed. | Delivery Module after rider failure reason and support path. | Returned, Refund Pending, Cancelled | Completed without resolution |
| Returned | Items have returned to store or require store review. | Delivery/Store Ops after failed delivery or cancellation after pickup. | Refund Pending, Completed | Picking, Delivered without support correction |

Terminal states: Completed and Cancelled are terminal for normal fulfillment. Refunded is terminal for refund workflow but may be followed by Completed for order closure. Returned is not terminal until support/refund/store review is complete.

Business rules governing transitions:
- Order state must follow fulfillment order: confirmation before reservation, reservation before picking, packing before rider assignment, delivery before completion.
- Online orders cannot move from Payment Pending to Confirmed without verified payment confirmation.
- COD/Card-on-Delivery orders may move to Confirmed without online capture but remain subject to delivery payment reconciliation.
- Orders cannot move to Picking unless inventory is reserved and the order belongs to the picker's store.
- Orders cannot move to Out for Delivery unless packed and accepted by an eligible rider.
- Completed orders cannot be cancelled; post-completion customer issues use refund/support flows.
- Support/admin corrections must add history and audit entries rather than erase prior state.

### 10.2 Payment State Machine

The Payment Module owns payment state. Order, Refund, Delivery, and Reconciliation workflows may reference payment state but do not own it.

| State | Purpose | Moved by / trigger | Allowed next states | Forbidden transitions |
|---|---|---|---|---|
| Pending | Payment record exists but final method outcome is not known. | Checkout/payment flow. | Authorized, COD Pending, Card-on-Delivery Pending, Failed, Cancelled | Paid, Refunded |
| Authorized | Online payment is authorized where the gateway flow supports authorization before capture. | Payment Module after verified gateway response. | Captured, Failed, Cancelled | COD Pending, Card-on-Delivery Pending |
| Captured | Online amount has been captured. | Payment Module after verified gateway capture/confirmation. | Paid, Refund Pending | COD Pending, Failed without corrective record |
| COD Pending | Cash is expected at delivery. | Checkout flow for COD order. | Paid, Failed, Cancelled, Refund Pending | Authorized, Captured |
| Card-on-Delivery Pending | Card terminal payment is expected at delivery. | Checkout flow for Card-on-Delivery order. | Paid, Failed, Cancelled, Refund Pending | Authorized, Captured |
| Paid | Payment obligation is satisfied. | Gateway confirmation, rider cash collection, or card terminal confirmation recorded. | Refund Pending | Pending, Failed, Cancelled |
| Failed | Payment attempt failed or expected delivery payment was not collected. | Payment/Delivery/Support workflow. | Pending, Cancelled, Refund Pending | Paid without valid payment evidence |
| Refund Pending | A full/partial refund is required. | Refund Module after approved refund request. | Refunded, Paid | Pending, Failed |
| Refunded | Approved refund has been completed or manual resolution recorded. | Refund Module/finance workflow. | Paid only if partial refund leaves paid balance | Pending, Captured |
| Cancelled | Payment obligation was cancelled before fulfillment or due to failed order creation. | Payment/Order workflow. | Pending only through new payment attempt | Paid, Refunded |

Terminal states: Paid, Refunded, and Cancelled are terminal for normal payment movement. Failed may be terminal if the order is cancelled.

Business rules governing transitions:
- Payment state cannot be marked Paid from customer/client input alone.
- Online payment confirmation requires verified gateway evidence.
- COD and Card-on-Delivery must remain distinguishable for reconciliation.
- Refund transitions require an approved refund reason and order/payment context.
- Payment corrections require audit history.

### 10.3 Inventory Reservation State Machine

The Inventory Module owns reservation state. Reservations protect checkout integrity without becoming the final stock deduction.

| State | Purpose | Moved by / trigger | Allowed next states | Forbidden transitions |
|---|---|---|---|---|
| Created | Reservation request exists for an order item. | Checkout/order creation flow. | Reserved, Cancelled, Expired | Consumed, Adjusted |
| Reserved | Stock is held for the order. | Inventory Module after successful availability validation. | Adjusted, Released, Consumed, Expired, Cancelled | Created, Delivered |
| Adjusted | Reserved quantity changed due to substitution, partial availability, or customer response. | Picking/Inventory workflow. | Reserved, Released, Consumed, Cancelled | Created |
| Released | Reserved stock is returned to availability. | Cancellation, item removal, or support action before consumption. | Terminal | Consumed, Adjusted |
| Consumed | Reserved stock has been deducted during packing. | Packing completion. | Terminal | Released, Expired |
| Expired | Reservation timed out before order confirmation or valid fulfillment path. | Timeout/ops rule. | Released, Cancelled | Consumed |
| Cancelled | Reservation is void because order/payment flow did not proceed. | Order/payment cancellation. | Terminal | Consumed |

Terminal states: Released, Consumed, and Cancelled. Expired must resolve to Released or Cancelled.

Business rules governing transitions:
- Reservation cannot be consumed unless the order is in packing/packed flow.
- Reservation cannot be released after consumption without a separate inventory restoration/return action.
- Adjustments must preserve item-level history.
- Reservation changes must keep store-specific available inventory non-negative.

### 10.4 Picking Session State Machine

The Picking Module owns picking session state. A picking session represents operational ownership of picking work, not the full order lifecycle.

| State | Purpose | Moved by / trigger | Allowed next states | Forbidden transitions |
|---|---|---|---|---|
| Created | Picking work is available for claim. | Order enters store picker queue. | Claimed, Cancelled, Expired | Picking, Completed |
| Claimed | Picker has taken responsibility but may not have started item work. | Picker claim. | Picking, Released, Expired, Cancelled | Completed |
| Picking | Picker is actively resolving order items. | Picker starts item workflow. | Awaiting Substitution, Packing, Released, Cancelled | Completed without item resolution |
| Awaiting Substitution | Customer response is needed for one or more items. | Picker marks item unavailable/substitute proposed. | Picking, Packing, Released, Cancelled, Expired | Completed |
| Packing | Items are resolved and being packed. | Picker completes item resolution. | Completed, Released, Cancelled | Awaiting Substitution unless item issue is reopened |
| Completed | Packing is complete and order is ready for rider assignment. | Picker completes packing. | Terminal | Picking, Released |
| Expired | Claim or response window timed out. | Timeout rule. | Released, Cancelled | Completed |
| Released | Session is released back to queue or ops control. | Timeout/admin action. | Claimed, Cancelled | Completed |
| Cancelled | Picking session stopped because order was cancelled or invalidated. | Order/support/ops action. | Terminal | Picking, Completed |

Terminal states: Completed and Cancelled. Expired should be resolved to Released or Cancelled.

Business rules governing transitions:
- Only one active picking session may own an order.
- Picker must belong to the same store as the order.
- Picker claim timeout returns the order to the store queue.
- Packing cannot complete until every item is picked, substituted, removed, or otherwise resolved.
- Release/reassignment actions require audit history.

### 10.5 Delivery Assignment State Machine

The Delivery Module owns delivery assignment state. Assignment is store-scoped for MVP.

| State | Purpose | Moved by / trigger | Allowed next states | Forbidden transitions |
|---|---|---|---|---|
| Created | Delivery work exists for a packed order. | Order becomes packed. | Assigned, Cancelled | Accepted, Delivered |
| Assigned | Rider has been offered the delivery. | Dispatch/admin assignment. | Accepted, Rejected, Expired, Cancelled | Picked Up, Delivered |
| Accepted | Rider accepted within timeout. | Rider action. | Picked Up, Cancelled | Rejected, Expired |
| Rejected | Rider declined assignment. | Rider action. | Assigned, Cancelled | Accepted, Delivered |
| Expired | Rider did not respond within timeout. | Timeout rule. | Assigned, Cancelled | Accepted, Delivered |
| Picked Up | Rider collected order from store. | Rider/store handoff confirmation. | Out for Delivery, Cancelled | Assigned, Rejected |
| Out for Delivery | Rider is en route to customer. | Rider departure / delivery start. | Delivered, Failed, Cancelled | Assigned, Picked Up without correction |
| Delivered | Delivery proof accepted. | Rider OTP/proof completion. | Terminal | Failed, Cancelled |
| Failed | Delivery could not be completed. | Rider/support failure reason. | Assigned, Cancelled | Delivered without support correction |
| Cancelled | Assignment is no longer valid. | Order cancellation, admin action, or operational failure. | Terminal | Delivered |

Terminal states: Delivered and Cancelled. Failed may be reassigned or cancelled based on support decision.

Business rules governing transitions:
- Assignment cannot be created until order is packed.
- Rider must be active, eligible, and store-assigned unless admin manually changes store assignment.
- Rejected/expired assignments return to the same store rider pool for MVP.
- Delivery cannot be completed without OTP or approved exception.
- Failed delivery requires reason code and support visibility.

### 10.6 Shift State Machine

The Workforce Module owns shift state. Shifts define assignment eligibility and reconciliation windows.

| State | Purpose | Moved by / trigger | Allowed next states | Forbidden transitions |
|---|---|---|---|---|
| Created | Shift record exists but is not yet scheduled/started. | Admin/worker shift setup. | Scheduled, Started, Cancelled | Active, Reconciled |
| Scheduled | Shift is planned for a worker/store. | Admin/ops planning. | Started, Cancelled | Active without start |
| Started | Worker has begun shift. | Worker shift start. | Active, Paused, Ended | Reconciled, Closed |
| Active | Worker is eligible for assignments subject to availability. | Worker availability/ops confirmation. | Paused, Ended | Closed |
| Paused | Worker is temporarily not assignment-eligible. | Break, ops pause, worker action where allowed. | Active, Ended | Reconciled |
| Ended | Worker has ended shift; no new assignments. | Worker/admin shift end. | Reconciled, Closed | Active without reopening rule |
| Reconciled | Cash/task reconciliation is complete. | Ops/finance reconciliation. | Closed | Active, Paused |
| Closed | Shift is administratively closed. | Ops/admin closure. | Terminal | Active, Reconciled changes without correction |
| Cancelled | Planned shift will not occur. | Admin/ops action. | Terminal | Started, Active |

Terminal states: Closed and Cancelled.

Business rules governing transitions:
- Worker cannot receive new assignments unless shift is Started/Active and availability permits.
- Shift cannot close with unresolved cash reconciliation where COD/Card-on-Delivery obligations exist.
- Ended shifts cannot receive new tasks.
- Reconciliation and closure require audit history.

### 10.7 Support Ticket State Machine

The Support Module owns support ticket state. Tickets may request actions from Order, Payment, Refund, Inventory, Delivery, or Workforce modules but do not own those entities.

| State | Purpose | Moved by / trigger | Allowed next states | Forbidden transitions |
|---|---|---|---|---|
| Open | Ticket has been created and awaits triage. | Customer/support/admin creation. | Assigned, In Progress, Closed | Resolved without review where action is required |
| Assigned | Ticket has an owner. | Support/admin assignment. | In Progress, Waiting for Customer, Waiting for Store, Resolved, Closed | Reopened |
| In Progress | Support is actively investigating or resolving. | Support action. | Waiting for Customer, Waiting for Store, Resolved, Closed | Open |
| Waiting for Customer | Customer response is required. | Support request to customer. | In Progress, Resolved, Closed | Assigned without response/action |
| Waiting for Store | Store/ops input is required. | Support request to store/ops. | In Progress, Resolved, Closed | Assigned without response/action |
| Resolved | Support believes issue is resolved. | Support/admin resolution. | Closed, Reopened | Open |
| Closed | Ticket is closed. | Support/admin closure or auto-close after resolution window. | Reopened | In Progress without reopen |
| Reopened | Closed/resolved ticket requires more work. | Customer/support/admin reopen. | Assigned, In Progress | Closed without review |

Terminal states: Closed, unless reopened by allowed policy.

Business rules governing transitions:
- Tickets affecting money, inventory, order state, or worker status must invoke the owning module.
- Resolution must include reason/outcome where operational or financial action occurred.
- Reopened tickets preserve prior ticket history.
- Support actions with business impact require audit history.

### 10.8 Refund State Machine

The Refund Module owns refund state. Refund state must remain tied to the original order/payment context.

| State | Purpose | Moved by / trigger | Allowed next states | Forbidden transitions |
|---|---|---|---|---|
| Requested | Refund need has been raised. | Customer/support/system rule. | Pending Approval, Cancelled | Processing, Completed |
| Pending Approval | Refund awaits review. | Refund/support workflow. | Approved, Rejected, Cancelled | Completed |
| Approved | Refund is approved and ready to process. | Admin/support/automated eligibility rule. | Processing, Cancelled | Rejected |
| Rejected | Refund request is denied. | Support/admin decision. | Terminal | Processing, Completed |
| Processing | Refund is being settled or manually resolved. | Refund/finance workflow. | Completed, Failed | Approved without corrective record |
| Completed | Refund has settled or manual resolution is recorded. | Refund/finance completion. | Terminal | Rejected, Cancelled |
| Failed | Refund processing failed. | Gateway/manual finance failure. | Processing, Cancelled | Completed without resolution |
| Cancelled | Refund request is withdrawn or no longer valid. | Support/admin/system rule. | Terminal | Processing, Completed |

Terminal states: Rejected, Completed, and Cancelled. Failed is not terminal until retried or cancelled.

Business rules governing transitions:
- Refund cannot be processed without order/payment context and eligible amount.
- Completed refunds are immutable except through corrective refund records.
- Refund amount must use order price snapshots.
- Approval, rejection, cancellation, and completion require audit history.

### 10.9 Promotion Lifecycle

The Promotion Module owns promotion lifecycle. MVP uses promo codes, but the same lifecycle applies to future promotion rules where relevant.

| State | Purpose | Moved by / trigger | Allowed next states | Forbidden transitions |
|---|---|---|---|---|
| Draft | Promotion is being prepared and is not customer-eligible. | Admin creation. | Scheduled, Active, Archived | Expired |
| Scheduled | Promotion is configured for future activation. | Admin scheduling. | Active, Paused, Archived | Expired before active window unless schedule passes |
| Active | Promotion can be evaluated for eligible carts/orders. | Date/time rule or admin activation. | Paused, Expired, Archived | Draft |
| Paused | Promotion is temporarily unavailable. | Admin/ops action. | Active, Archived, Expired | Draft |
| Expired | Promotion validity window has ended. | Time rule/admin closure. | Archived | Active unless admin creates a new eligible version |
| Archived | Promotion is retained for history/reporting and no longer editable for active use. | Admin/archive action. | Terminal | Active, Draft |

Terminal state: Archived.

Business rules governing transitions:
- Active promotions must be revalidated at order creation.
- Paused, expired, and archived promotions cannot apply to new orders.
- Historical orders retain applied promotion details.
- Promotion state changes require audit history.

### 10.10 Worker Availability State

Worker availability is operational state and is separate from shift lifecycle. Shift determines whether the worker is on duty; availability determines whether the worker can receive work.

| State | Purpose | Moved by / trigger | Assignment eligibility | Allowed next states |
|---|---|---|---|---|
| Offline | Worker is not connected or not available. | Worker logout/app offline/shift not active. | Not eligible | Online, Unavailable |
| Online | Worker is connected but not necessarily available. | Worker login/app online. | Not eligible until available and on shift | Available, Break, Unavailable, Offline |
| Available | Worker can receive eligible tasks. | Worker/ops availability action during active shift. | Eligible | Busy, Picking, Delivering, Break, Unavailable, Offline |
| Busy | Worker is occupied but not specifically in picker/rider task state. | Ops/system state. | Not eligible | Available, Break, Unavailable |
| Picking | Worker is actively in picker task mode. | Picking session ownership. | Not eligible for conflicting tasks | Available, Busy, Unavailable |
| Delivering | Worker is actively in rider delivery flow. | Delivery assignment accepted/picked up. | Not eligible for another delivery in MVP | Available, Busy, Unavailable |
| Break | Worker is on break. | Worker/ops action. | Not eligible | Available, Unavailable, Offline |
| Unavailable | Worker is administratively unavailable. | Ops/admin action or operational issue. | Not eligible | Online, Available, Offline |

Business rules governing availability:
- Worker must have an active shift and required capability before becoming assignment-eligible.
- Offline, Break, Busy, Picking, Delivering, and Unavailable workers cannot receive new assignments.
- Picking and Delivering are mutually exclusive active task states for MVP.
- Availability changes that affect assignments or reconciliation require history preservation; admin-forced changes require audit history.

### 10.11 Transition Requirements

- Allowed transitions: Each entity may move only through the states listed in its lifecycle table unless support/admin executes an approved corrective workflow.
- Disallowed transitions: Direct jumps that skip payment verification, inventory reservation, picking resolution, packing completion, rider acceptance, delivery proof, refund approval, or reconciliation are invalid.
- Required business validation: Transitions must validate ownership, store scope, role/capability, current state, terminal-state rules, payment status, inventory status, and customer/support eligibility where relevant.
- Audit requirements: Admin/support overrides, financial changes, inventory changes, worker assignment changes, promotion state changes, and reconciliation actions require audit history.
- History preservation: State history must be preserved. Corrections append new history; they do not rewrite prior lifecycle events.
- Terminal states: Terminal states cannot return to active workflow except through explicitly allowed reopen/correction paths.

### 10.12 Lifecycle Summary Table

| Entity | Initial state | Terminal states | Can reopen? | Requires audit? | History preserved? |
|---|---|---|---|---|---|
| Order | Created | Completed, Cancelled | Limited support correction only | Yes for admin/support changes | Yes |
| Payment | Pending | Paid, Refunded, Cancelled | No; corrective records only | Yes | Yes |
| Inventory Reservation | Created | Released, Consumed, Cancelled | No | Yes for manual/admin actions | Yes |
| Picking Session | Created | Completed, Cancelled | Released sessions can be reclaimed | Yes for release/reassignment | Yes |
| Delivery Assignment | Created | Delivered, Cancelled | Failed assignments can be reassigned | Yes for admin/support changes | Yes |
| Shift | Created | Closed, Cancelled | No normal reopen after closure | Yes for reconciliation/closure | Yes |
| Support Ticket | Open | Closed | Yes | Yes if business action occurs | Yes |
| Refund | Requested | Rejected, Completed, Cancelled | Failed can retry; completed cannot reopen | Yes | Yes |
| Promotion | Draft | Archived | No; create new version instead | Yes | Yes |
| Worker Availability | Offline | Not terminal | Yes | Yes for admin-forced changes | Yes |

---

## 11. Data Requirements

This section defines the logical data requirements needed for implementation and for the follow-on Database Design Document. It intentionally does not define tables, columns, indexes, migrations, SQL, or physical database constraints. The DDD should translate these requirements into a concrete PostgreSQL design.

Existing Postgres schema from the prior backend (orders, delivery agents, picking sessions) is a valid starting reference — not a rebuild from zero.

### 10.1 Entity Ownership Summary

| Entity | Owning module |
|---|---|
| User / Customer profile / Worker profile | Auth & Identity Module |
| Address | Customer Module |
| Store / Service zone | Store Module |
| Category / Product / Product media | Catalog Module |
| Inventory record / Inventory reservation / Inventory ledger | Inventory Module |
| Cart | Cart Module |
| Order / Order item / Order event | Order Module |
| Picking session / Picking session item | Picking Module |
| Delivery assignment / Rider location | Delivery Module |
| Payment / Cash remittance / Card-on-Delivery record | Payment Module |
| Refund | Refund Module |
| Promo code / Promotion rule | Promotion Module |
| Price snapshot | Pricing Module |
| Notification | Notification Module |
| Support ticket | Support Module |
| Shift | Workforce Module |
| Analytics rollup | Reporting & Analytics Module |
| Audit log | Audit / Compliance Module |

Ownership means the module is responsible for lifecycle rules, validation, state transitions, and write APIs. Other modules may reference an entity but should not mutate it directly except through the owning module's application service.

### 10.2 Core Entity Requirements

#### 10.2.1 User
- Purpose: Represents an authenticated person or staff account in the platform.
- Responsibilities: Identity, login method, security state, and role linkage.
- Primary relationships: Customer profile, worker profile, admin role, support actions, audit logs.
- Ownership: Auth & Identity Module.
- Lifecycle: Created through phone OTP onboarding for customers/workers or admin account creation for staff; updated by profile/admin workflows; not hard-deleted after business activity exists.
- Business constraints: A user cannot perform actions outside assigned roles; inactive users cannot authenticate or receive new operational tasks.

#### 10.2.2 Customer Profile
- Purpose: Represents the customer-facing identity used for browsing, checkout, support, and order history.
- Responsibilities: Customer display data, contact linkage, order history access, and support context.
- Primary relationships: User, addresses, carts, orders, support tickets, notifications.
- Ownership: Customer Module, with authentication owned by Auth & Identity.
- Lifecycle: Created on first customer login or checkout onboarding; updated by customer/profile support actions; soft-deleted or anonymized only according to retention/privacy policy.
- Business constraints: Customer order history must remain available for support, refunds, reconciliation, and reporting even if the customer account is later disabled.

#### 10.2.3 Worker Profile
- Purpose: Represents operational staff using the Worker App.
- Responsibilities: Store assignment, worker status, capability access, task eligibility, and shift participation.
- Primary relationships: User, worker capabilities, store, shifts, picking sessions, delivery assignments, cash remittances.
- Ownership: Workforce Module.
- Lifecycle: Created by admin only for MVP; updated by admin/ops; deactivated rather than deleted once assigned to tasks.
- Business constraints: Worker cannot self-register, self-assign store, or receive tasks outside assigned capabilities and active shift state.

#### 10.2.4 Worker Capability
- Purpose: Defines whether a worker can act as picker, rider, or both.
- Responsibilities: Mode eligibility and assignment routing.
- Primary relationships: Worker profile, store, shifts, picking sessions, delivery assignments.
- Ownership: Workforce Module.
- Lifecycle: Created/updated by admin; changes are audited; historical tasks retain the capability context that existed when assigned.
- Business constraints: A worker cannot switch mode while an active task is unresolved; capability changes must not mutate completed task history.

#### 10.2.5 Address
- Purpose: Represents a customer's delivery location.
- Responsibilities: Delivery eligibility, store routing, map/geocoding context, and saved address reuse.
- Primary relationships: Customer profile, order, store/service zone.
- Ownership: Customer Module.
- Lifecycle: Created/updated/deleted by customer or support; historical orders keep the address snapshot used at order time.
- Business constraints: Checkout requires an address that resolves into one serving store; changing a saved address must not alter past order records.

#### 10.2.6 Store / Dark Store
- Purpose: Represents a physical operating location serving a neighborhood radius.
- Responsibilities: Service zone origin, inventory ownership boundary, worker home assignment, order routing, operating state, and reconciliation scope.
- Primary relationships: Service zone, workers, inventory records, orders, shifts, cash reconciliation, reports.
- Ownership: Store Module.
- Lifecycle: Created/updated by admin; temporarily closed by ops; not hard-deleted while referenced by orders, inventory history, shifts, or reports.
- Business constraints: Each order must belong to exactly one store; store isolation applies to inventory, worker queues, rider pools, and reconciliation.

#### 10.2.7 Service Zone
- Purpose: Represents the geographic delivery coverage of a store.
- Responsibilities: Address eligibility and store assignment.
- Primary relationships: Store, address, order.
- Ownership: Store Module.
- Lifecycle: Derived from store coordinate + radius for MVP; updated by admin; historical orders retain assigned store even if radius changes later.
- Business constraints: Radius-based zones are MVP; if multiple zones match, nearest store wins; if none match, checkout is blocked.

#### 10.2.8 Category
- Purpose: Organizes products for browsing, search, promotions, and reports.
- Responsibilities: Catalog hierarchy and customer-facing navigation.
- Primary relationships: Product, promotion rule, analytics rollup.
- Ownership: Catalog Module.
- Lifecycle: Created/updated by admin; disabled rather than deleted if products or historical reports reference it.
- Business constraints: Category changes must not alter historical order item meaning.

#### 10.2.9 Product
- Purpose: Represents the global sellable item definition.
- Responsibilities: Bilingual product content, image references, unit, taxability, category membership, and active status.
- Primary relationships: Category, inventory record, order item, promotion rule, price snapshot.
- Ownership: Catalog Module.
- Lifecycle: Created/updated by catalog admin; disabled when no longer sold; not hard-deleted once ordered or referenced by inventory history.
- Business constraints: Product definition is global; sellability is store-specific through inventory and store pricing.

#### 10.2.10 Product Media
- Purpose: Represents product image assets used by the Customer Platform and Admin Dashboard.
- Responsibilities: Image reference, display ordering, and active/inactive state.
- Primary relationships: Product.
- Ownership: Catalog Module.
- Lifecycle: Created/updated by catalog admin; old media may be disabled while historical product/order context remains intact.
- Business constraints: Missing or inactive media must not block operational order flow.

#### 10.2.11 Inventory Record
- Purpose: Represents store-specific sellable stock state for a product.
- Responsibilities: On-hand quantity, reserved quantity, availability, low-stock status, shelf metadata, and reconciliation state.
- Primary relationships: Store, product, inventory reservation, inventory ledger, order item.
- Ownership: Inventory Module.
- Lifecycle: Created when a product is stocked at a store; updated by reservation, packing, sync, reconciliation, and manual adjustment workflows; disabled rather than deleted while history exists.
- Business constraints: Available inventory cannot go negative; inventory is isolated by store; global product availability must never be inferred from one store's stock.

#### 10.2.12 Inventory Reservation
- Purpose: Represents stock held for an order before packing/deduction.
- Responsibilities: Prevent oversell between checkout and picker completion.
- Primary relationships: Order, order item, inventory record, store.
- Ownership: Inventory Module.
- Lifecycle: Created during checkout/order creation; adjusted during substitution; released on eligible cancellation; consumed when packing completes; history preserved through inventory events.
- Business constraints: Reservation must be idempotent per order attempt; stale reservations must be releasable only through system timeout or audited admin action.

#### 10.2.13 Inventory Ledger
- Purpose: Represents the history of inventory changes.
- Responsibilities: Auditability for reservations, deductions, restorations, adjustments, damaged stock, expired stock, returns, and sync changes.
- Primary relationships: Inventory record, store, product, order, worker/admin user.
- Ownership: Inventory Module.
- Lifecycle: Append-only; created by inventory-changing operations; never updated for business meaning; never hard-deleted under normal operation.
- Business constraints: Current inventory state must be explainable from ledger history and approved operational actions.

#### 10.2.14 Cart
- Purpose: Represents the customer's pre-order basket for a serving store.
- Responsibilities: Selected items, quantities, pricing preview, promo validation preview, and checkout preparation.
- Primary relationships: Customer profile, address/serving store, product, promotion rule.
- Ownership: Cart Module.
- Lifecycle: Created during browsing; updated by customer actions; expires or is cleared after successful order creation.
- Business constraints: Cart values are not final financial records; checkout must revalidate price, promotion, address, payment, and inventory.

#### 10.2.15 Order
- Purpose: Represents a committed customer purchase request routed to one store.
- Responsibilities: Order status, customer/store linkage, fulfillment lifecycle, payment state summary, support context, and reporting source.
- Primary relationships: Customer, address snapshot, store, order items, inventory reservations, picking session, delivery assignment, payment, refund, support ticket, order events.
- Ownership: Order Module.
- Lifecycle: Created after checkout validation; updated through controlled state transitions; cancelled/completed/failure states are terminal except approved support correction; not hard-deleted.
- Business constraints: Order cannot exist without customer, serving store, address snapshot, and at least one order item; order history must be preserved for support, refunds, reconciliation, and analytics.

#### 10.2.16 Order Item
- Purpose: Represents a product requested, substituted, removed, or fulfilled within an order.
- Responsibilities: Item-level quantity, final fulfillment outcome, price snapshot reference, substitution/refund context, and inventory linkage.
- Primary relationships: Order, product, inventory reservation, picking session item, refund.
- Ownership: Order Module, with stock effects owned by Inventory Module.
- Lifecycle: Created with order; updated during picking/substitution; final item outcome preserved after order completion.
- Business constraints: Historical item price and product meaning must not change if catalog data changes later.

#### 10.2.17 Order Event
- Purpose: Represents important lifecycle transitions and operational actions on an order.
- Responsibilities: Timeline, support traceability, SLA measurement, and audit context.
- Primary relationships: Order, customer, worker, admin/support user, payment/refund where applicable.
- Ownership: Order Module.
- Lifecycle: Append-only; created whenever order state or meaningful operational context changes.
- Business constraints: Support/admin overrides must add corrective events rather than erase prior events.

#### 10.2.18 Picking Session
- Purpose: Represents a picker's claimed work on an order.
- Responsibilities: Claim ownership, picking start/end timestamps, picker timeout behaviour, and packing completion.
- Primary relationships: Order, picker worker profile, store, picking session items.
- Ownership: Picking Module.
- Lifecycle: Created when a picker claims an order; updated as items are picked/resolved; completed, expired, or released; history preserved.
- Business constraints: Only one active picking session may own an order at a time; picker and order must belong to the same store.

#### 10.2.19 Picking Session Item
- Purpose: Represents item-level picking outcome.
- Responsibilities: Picked quantity, unavailable status, substitution request/approval outcome, and packing readiness.
- Primary relationships: Picking session, order item, inventory record.
- Ownership: Picking Module.
- Lifecycle: Created with picking session; updated by picker actions and customer substitution response; finalized at packing completion.
- Business constraints: Packing cannot complete until every item is resolved.

#### 10.2.20 Delivery Assignment
- Purpose: Represents rider responsibility for a packed order.
- Responsibilities: Assignment, accept/reject/timeout, pickup, delivery, failed delivery, and OTP completion.
- Primary relationships: Order, rider worker profile, store, payment collection, rider location events.
- Ownership: Delivery Module.
- Lifecycle: Created when a packed order is assigned; updated by rider/admin actions; completed, expired, rejected, or failed; history preserved.
- Business constraints: Rider assignment pool is store-scoped for MVP; automated cross-store assignment is not allowed.

#### 10.2.21 Rider Location
- Purpose: Represents operational rider position while online or on active delivery.
- Responsibilities: Live tracking, ETA support, and delivery operations.
- Primary relationships: Rider worker profile, delivery assignment, order.
- Ownership: Delivery Module.
- Lifecycle: Created while rider is online/active; updated at throttled intervals; retained only as long as operationally needed.
- Business constraints: Location collection must be limited to worker online/active delivery states and PDPL review.

#### 10.2.22 Payment
- Purpose: Represents the financial obligation and settlement status for an order.
- Responsibilities: Online payment verification, COD/Card-on-Delivery expected amount, payment status, reconciliation linkage, and refund basis.
- Primary relationships: Order, customer, Moyasar transaction where applicable, cash remittance, refund.
- Ownership: Payment Module.
- Lifecycle: Created with or immediately after order creation; updated only through verified gateway events, rider collection, or admin reconciliation actions; history preserved.
- Business constraints: Payment confirmation cannot rely on client-provided status; payment state must remain explainable for finance and support.

#### 10.2.23 Cash Remittance
- Purpose: Represents rider cash handover and reconciliation.
- Responsibilities: Expected cash, remitted cash, discrepancy, reason code, and admin acknowledgement.
- Primary relationships: Rider worker profile, shift, store, payment records, admin user.
- Ownership: Payment Module.
- Lifecycle: Created during shift/day reconciliation; updated by finance/ops acknowledgement; not hard-deleted.
- Business constraints: Cash and card-on-delivery must be reconciled separately.

#### 10.2.24 Refund
- Purpose: Represents money or manual compensation owed back to a customer.
- Responsibilities: Eligibility, amount, reason, approval, payment-method handling, and settlement status.
- Primary relationships: Order, order item, payment, support ticket, admin/support user.
- Ownership: Refund Module.
- Lifecycle: Created by support/admin or automatic substitution/cancellation flow; updated through approval and settlement; immutable once settled except corrective refund records.
- Business constraints: Refund amount must use order price snapshots, not current catalog price.

#### 10.2.25 Promo Code
- Purpose: Represents the MVP customer-entered discount mechanism.
- Responsibilities: Code eligibility, usage limits, validity, customer/store constraints, and reporting.
- Primary relationships: Customer, cart, order, promotion rule.
- Ownership: Promotion Module.
- Lifecycle: Created/updated/disabled by admin; usage history preserved after deactivation.
- Business constraints: Promo eligibility must be revalidated at order creation; historical promo usage must remain auditable.

#### 10.2.26 Promotion Rule
- Purpose: Represents future-ready discount logic beyond MVP promo codes.
- Responsibilities: Eligibility, benefit definition, funding/source context, date/time validity, and reporting classification.
- Primary relationships: Promo code, product/category, store, cart, order.
- Ownership: Promotion Module.
- Lifecycle: Created/updated/disabled by admin; historical orders retain applied promotion snapshot.
- Business constraints: Promotion rules must not require a separate pricing service for MVP.

#### 10.2.27 Price Snapshot
- Purpose: Represents the final price facts used for an order or order item.
- Responsibilities: Product price, store price, VAT, fees, discounts, substitution price, and refund basis.
- Primary relationships: Order, order item, payment, refund, promotion rule.
- Ownership: Pricing Module.
- Lifecycle: Created at order placement and adjusted only through explicit substitution/refund events; preserved permanently with order history.
- Business constraints: Historical order totals must not change when catalog, fee, VAT, or promotion rules change later.

#### 10.2.28 Notification
- Purpose: Represents an attempted or delivered customer/worker/admin message.
- Responsibilities: Push, SMS, in-app notification history, delivery status, retry context, and user visibility.
- Primary relationships: User, order, support ticket, worker assignment.
- Ownership: Notification Module.
- Lifecycle: Created when a notification is requested; updated with send/delivery/failure state; expired or archived according to retention needs.
- Business constraints: Order/payment/inventory state must not depend only on notification delivery.

#### 10.2.29 Support Ticket
- Purpose: Represents a customer, worker, order, payment, refund, or delivery issue.
- Responsibilities: Support workflow, issue classification, ownership, resolution notes, and escalation context.
- Primary relationships: Customer, order, payment, refund, delivery assignment, support/admin user.
- Ownership: Support Module.
- Lifecycle: Created by customer/support/admin; updated until resolved; reopened where policy allows; preserved for support history and quality review.
- Business constraints: Support actions that affect order/payment/inventory must call the owning module and produce audit history.

#### 10.2.30 Shift
- Purpose: Represents worker availability for a defined work period.
- Responsibilities: Assignment eligibility, productivity reporting, cash reconciliation window, and attendance context.
- Primary relationships: Worker profile, store, picking sessions, delivery assignments, cash remittance.
- Ownership: Workforce Module.
- Lifecycle: Created at shift start; updated by availability/task events; closed at shift end; preserved for reports and reconciliation.
- Business constraints: Worker must have an active shift before receiving tasks.

#### 10.2.31 Analytics Rollup
- Purpose: Represents derived reporting summaries used by Admin Dashboard.
- Responsibilities: Daily/weekly/monthly sales, operations, inventory, finance, and worker metrics.
- Primary relationships: Orders, payments, refunds, inventory ledger, shifts, delivery assignments.
- Ownership: Reporting & Analytics Module.
- Lifecycle: Created/rebuilt by scheduled rollup jobs; corrected when source data is corrected; source event history remains authoritative.
- Business constraints: Rollups must be rebuildable from source records and must not replace operational source of truth.

#### 10.2.32 Audit Log
- Purpose: Represents administrative and sensitive operational change history.
- Responsibilities: Accountability for changes to stores, workers, inventory, prices, promotions, refunds, orders, settings, and permissions.
- Primary relationships: Acting user, affected entity, store where relevant.
- Ownership: Audit / Compliance Module.
- Lifecycle: Append-only; created by audited actions; never edited for business meaning; never hard-deleted under normal operation.
- Business constraints: Audit history must survive entity deactivation and must be available for support, finance, and compliance review.

### 10.3 Logical Entity Relationships

These relationship descriptions are requirements-level guidance for the future ERD, not the ERD itself.

Customer flow:
```
User
  ↓
Customer Profile
  ↓
Address
  ↓
Cart
  ↓
Order
  ↓
Order Items
  ↓
Inventory Reservations
```

Store/catalog/inventory flow:
```
Store
  ↓
Service Zone
  ↓
Orders

Store
  ↓
Inventory Records
  ↓
Products
  ↓
Categories

Inventory Records
  ↓
Inventory Reservations
  ↓
Inventory Ledger
```

Worker fulfillment flow:
```
Worker Profile
  ↓
Worker Capabilities
  ↓
Shift
  ↓
Picking Sessions / Delivery Assignments
  ↓
Order Events
```

Payment/refund flow:
```
Order
  ↓
Payment
  ↓
Refunds

Rider Shift
  ↓
COD / Card-on-Delivery Records
  ↓
Cash Remittance
```

Support flow:
```
Customer / Worker / Order
  ↓
Support Tickets
  ↓
Refunds or Operational Corrections
  ↓
Audit Logs
```

Reporting flow:
```
Orders / Payments / Refunds / Inventory Ledger / Shifts
  ↓
Analytics Rollups
  ↓
Admin Reports
```

### 10.4 Lifecycle Ownership Requirements

- Orders: Created by checkout; updated by Order/Picking/Delivery/Support flows through controlled state transitions; never hard-deleted; history must be preserved.
- Payments: Created with order payment flow; updated by verified gateway events, rider collection, or reconciliation; never hard-deleted; settlement history preserved.
- Refunds: Created by support/admin or automatic cancellation/substitution rules; updated through approval/settlement; settled refunds are immutable except through corrective records.
- Inventory records: Created when a product is stocked at a store; updated by Inventory Module only; disabled rather than deleted when no longer sellable.
- Inventory ledger: Created by every stock-changing operation; append-only; never deleted under normal operation.
- Audit logs: Created by sensitive/admin actions; append-only; never modified for business meaning.
- Notifications: Created by notification requests; updated with delivery/failure status; may be archived/expired after operational usefulness ends.
- Support tickets: Created by customer/support/admin; updated by support workflow; soft-closed/resolved rather than deleted.
- Shifts: Created by worker shift start; updated by task/availability events; closed at shift end; preserved for productivity and reconciliation.
- Rider locations: Created while online/active; updated by tracking events; retained only for operational and dispute-resolution needs.
- Analytics rollups: Created and refreshed from source records; rebuildable; not authoritative over source records.

### 10.5 Data Integrity Requirements

- Foreign reference consistency: business records must not reference missing customers, stores, products, workers, orders, payments, or inventory records.
- Transaction boundaries: order creation, payment confirmation, inventory reservation, inventory deduction, cancellation, refund creation, and reconciliation must leave the system in a complete business state.
- Idempotent operations: checkout/order creation, payment webhooks, refund requests, notification sends, and inventory reservation/release must tolerate retries without duplicate business effects.
- Immutable history: audit logs, inventory ledger entries, payment events, order events, and settled refund records must preserve history rather than being overwritten.
- No orphaned records: order items, payments, refunds, reservations, picking sessions, delivery assignments, and support tickets must remain attached to their parent business context.
- Store isolation: inventory, worker queues, rider pools, orders, reconciliation, and reports must be scoped to the correct store unless an audited admin action explicitly changes operational ownership.
- Inventory consistency: available inventory must not become negative; reservations must be released/consumed according to order lifecycle; inventory mismatches must be visible for reconciliation.
- Referential integrity: disabling or deactivating a user, worker, product, category, store, promotion, or setting must not break historical orders, reports, refunds, or audit logs.
- Append-only events: operational timelines must record new events for corrections rather than mutating prior events.
- Source of truth: operational source records remain authoritative; analytics rollups and dashboard summaries are derived views.

### 10.6 Data Retention Requirements

Retention values should be finalized with legal/accounting advice before production launch. These are business expectations for the DDD and implementation planning.

- Orders and order items: Retain long-term for customer support, tax/accounting, reconciliation, and analytics.
- Payments and refunds: Retain long-term for finance, dispute handling, settlement reconciliation, and accounting.
- Audit logs: Retain long-term; deletion should not be part of normal application workflows.
- Inventory ledger: Retain long-term so stock movements remain explainable.
- Support tickets: Retain for support history, dispute resolution, and quality review; closed tickets may be archived.
- Notifications: Retain short-term for operational troubleshooting and in-app history; SMS/push provider metadata should not be kept longer than useful.
- Worker locations: Retain short-term only for live tracking, delivery disputes, SLA review, and safety/ops needs.
- Shifts and task assignments: Retain for payroll/productivity/reconciliation and operational history.
- Analytics rollups: Retain as long as useful for business trend reporting; must be rebuildable from source records.
- Customer profile/address data: Retain according to PDPL privacy obligations and business support needs; historical orders keep necessary order-time snapshots.

---

## 12. Recommended Workspace Structure

```
Sakhariapp/
  rapiddash/                 # Legacy Flutter reference prototype (historical name, untouched)
  rapiddash-api/             # Legacy NestJS backend codebase, reused/refactored into Sakhari Ecom API
  sakhari-mobile/
    apps/
      customer/
      worker/
    packages/
      api/
      types/
      ui/
      config/
  sakhari-web/
    apps/
      customer-web/
  sakhari-admin/              # React + Vite dashboard
```

---

## 13. MVP Scope

**In:**
- Customer Platform: phone login, browse, search, server-side cart, checkout (Moyasar + Cash on Delivery + Card on Delivery), order tracking, EN+AR UI, automatic store routing by address across Customer Mobile App and Customer Web App
- Worker: login, mode selection, picker queue/pick/pack (scoped to assigned store), rider availability/assignment/accept-reject/navigate/OTP delivery/card terminal charge (scoped to assigned store)
- Admin: multi-store management, catalog, orders, worker accounts + store assignment, promo codes, inventory operations, support tickets, refund management, per-store reconciliation, reports, basic analytics
- Push + SMS notifications for key events; browser notifications and email are future channels

**Deferred (Phase 2+):**
- Barcode scanning
- Embedded in-app navigation (relying on Google Maps deep link)
- Automated cross-store dispatch/reassignment (manual admin action for MVP, §3.1.2)
- Polygon-based service zones (radius-based for MVP)
- AI-based dispatch (simple proximity/zone assignment for MVP)
- Wallet / loyalty program
- Demand prediction / dynamic pricing
- Arabic in Worker App and Admin Dashboard (English-only at launch, fast-follow)
- BNPL — Tabby and Tamara, both (confirmed as fast-follow, not MVP)
- Kafka/streaming, Kubernetes, ClickHouse, multi-region — revisit only when real usage data justifies them

---

## 14. Assumptions Made in This Draft (please confirm/correct)

1. **Brand name:** The production project name is now **Sakhari Ecom**. Legacy folder names such as `rapiddash/` and `rapiddash-api/` may remain in historical code references until the repositories are formally renamed.
2. **Arabic elevated to MVP for the Customer Platform only** (not Worker/Admin) — see §3.3. This is a real scope addition versus the prior RN SRS; flag if you'd rather push all Arabic to fast-follow.
3. **Worker accounts admin-created only**, no self-registration for MVP (fraud/vetting reasons) — resolves prior open question OQ-004.
4. **Substitution flow is customer-approved** (not auto-approved or picker-controlled) — resolves prior open question OQ-006, carried over from existing docs.
5. **Barcode scanning and photo proof-of-delivery remain deferred**, OTP-only for delivery proof — resolves OQ-007/OQ-008, carried over.
6. **Service zones are radius-based, not polygon-based**, for MVP (§3.1.2) — simplest to implement correctly with a solo dev; revisit only if radius zones actually misroute orders in practice.
7. **No automated cross-store rider reassignment for MVP** — a manual admin action if a store runs short-staffed. Flag if this needs to be automated sooner than expected.
8. **Payment gateway defaulted to Moyasar**, decided in §3.2 based on fee/settlement/integration comparison.

---

## 15. Resolved Questions (this round)

- ~~OQ-A: UAE and Saudi simultaneously, or one first?~~ → **Saudi only** for MVP; UAE deferred to a future expansion phase.
- ~~OQ-B: Stripe alone, or stronger COD reconciliation needed?~~ → **COD (cash) expected to be significant**; dedicated reconciliation flow added, now also covering card-on-delivery (§3.2). Also surfaced that Stripe isn't usable for KSA merchants at all — switched to Moyasar.
- ~~OQ-C: Data residency requirements affecting cloud region?~~ → **Yes, PDPL applies**; AWS region switched to `me-central-2` (Riyadh), DigitalOcean scoped to dev/prototyping only.
- ~~OQ-D: Payment gateway preference among Moyasar/HyperPay/PayTabs/Tap?~~ → **Moyasar**, based on fee/settlement/integration comparison in §3.2.
- ~~OQ-E: Which city launches first?~~ → **Sakaka, Al Jouf**, multi-store across neighborhoods (Qara, Lakhayath, Khatib, etc.) — architecture in §3.1.1–3.1.2.
- ~~OQ-F: Single store or multi-store?~~ → **Multi-store from launch**, not single-store as originally assumed — full dispatch model in §3.1.2, FR-STORE-001–007.
- ~~OQ-G: BNPL provider — one or both?~~ → **Both Tabby and Tamara**, confirmed as fast-follow.
- ~~OQ-H: Card-on-delivery via existing POS terminals?~~ → **Yes**, added as a third payment method (FR-PAY-POS-001–004).
- ~~OQ-I: Supabase+Twilio for auth, or another option?~~ → **Neither** — auth built directly in NestJS+RDS to preserve PDPL residency (Supabase has no Middle East region); SMS/OTP switched to **Unifonic** since Twilio can't register Saudi domestic Sender IDs at all (§4.7).
- ~~OQ-J: Database hosting — EC2 or another option?~~ → **Amazon RDS for PostgreSQL** in `me-central-2`, not self-managed Postgres on EC2 (§4.4).

## 16. Remaining Open Questions

None blocking at this point. Items to settle during Phase 1 execution rather than in this SRD:
- Spot-check Google Maps geocoding/routing accuracy per-neighborhood in Sakaka before committing to the 10–20 min SLA publicly.
- Confirm existing POS terminal provider/model (commonly Geidea in this market) and whether it exposes a settlement API for automated reconciliation (FR-PAY-POS-004) versus manual rider entry only.
- Finalize which neighborhoods get a store at launch vs. fast-follow (Qara, Lakhayath, Khatib were named as examples — confirm the full initial list and each store's approximate coverage radius).

---

*Document prepared for internal development use. Supersedes prior SRD/SRS drafts listed in §0.*
