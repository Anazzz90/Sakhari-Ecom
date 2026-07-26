# Database Design Document (DDD)
## Sakhari Ecom — Saudi Arabia Quick Commerce Platform

**Version:** 1.0  
**Date:** July 24, 2026  
**Status:** Draft  
**Source of Truth:** `SRD_Sakhari_Ecom_v2.6_Saudi.md`  
**Prepared For:** Solo, AI-assisted development

---

## 0. Purpose & Scope

### 0.1 Purpose
This Database Design Document defines how the Sakhari Ecom business domain should be represented in PostgreSQL. It converts the completed SRD v2.6 into a database architecture reference that future schema, migration, repository, API, and test work can use with minimal ambiguity.

The SRD remains the authoritative business requirements document. This DDD does not reinterpret business rules, lifecycle state machines, technology choices, or MVP scope. It describes the database model needed to support those requirements.

### 0.2 Database Responsibilities
The database is responsible for preserving the authoritative state and history of:

- Customers, workers, stores, catalog, inventory, orders, payments, refunds, promotions, support, audit, and reporting data.
- Shared Customer Platform data across Customer Mobile App and Customer Web App, including authentication identity, customer profile, addresses, server-side cart, order history, notifications, promotions, and support tickets.
- Store isolation for inventory, worker assignment, orders, delivery pools, reconciliation, and reports.
- Business integrity across order creation, inventory reservation, picking, delivery, payment, refund, and reconciliation workflows.
- Historical traceability through order events, payment history, inventory ledgers, support history, audit logs, and analytics rollups.

PostgreSQL on Amazon RDS is the source of truth. Redis may coordinate locks, cache reads, and support ephemeral data flows, but Redis must never become authoritative for business records.

The database is platform-agnostic. Customer Mobile App and Customer Web App consume the same backend and share the same customer entities. Platform/device metadata may be stored for analytics, sessions, and notification capability, but it must not create separate customer business logic or duplicate customer records.

### 0.3 Scope Boundaries
This document covers:

- Logical entity catalog and ownership.
- Conceptual field groups.
- Relationships and cardinality.
- Database constraints and application-enforced constraints.
- Indexing philosophy.
- Transaction boundaries.
- Concurrency requirements.
- Soft delete, archival, audit, history, and ledger strategy.
- Store isolation model.
- Background jobs that depend on persistent data.
- Performance and future expansion considerations.

### 0.4 Intentionally Excluded
This document does not include:

- SQL statements.
- Physical table definitions.
- Migration scripts.
- ORM models or syntax.
- API endpoints.
- ER diagrams.
- Sequence diagrams.
- Redis implementation details.
- NestJS service design.
- Infrastructure deployment design.

Those belong in later implementation documents.

---

## 1. Database Design Principles

### 1.1 PostgreSQL Is the Source of Truth
All durable business data must be stored in PostgreSQL. Redis is acceptable for coordination, locks, queues, caching, and short-lived operational state, but any data required for recovery, reconciliation, finance, support, or audit must be persisted in PostgreSQL.

Rationale: Sakhari Ecom has payment, inventory, delivery, and reconciliation workflows where correctness matters more than low-latency cache convenience. PostgreSQL provides ACID behavior, referential integrity, JSONB support, and operational maturity on Amazon RDS.

### 1.2 ULID Primary Keys
Every primary business entity should use ULID identifiers unless a later implementation document justifies a narrower local identifier. Public-facing identifiers should not expose a predictable insertion count, even though ULIDs remain lexicographically sortable by creation time.

Rationale: ULIDs reduce accidental information leakage (no exposed sequence count), support distributed generation if needed later without a coordinating authority, and remain lexicographically sortable by creation time — which aids debugging, log correlation, and audit-trail readability without reintroducing the sequential-leakage risk a plain incrementing key would carry.

### 1.3 UTC Timestamps
All persisted timestamps must be stored in UTC. User-facing displays can render in the appropriate local timezone, but storage remains timezone-neutral.

Rationale: Operations happen in Saudi Arabia for MVP, but UTC avoids ambiguity during integrations, support, reporting, and possible future expansion.

### 1.4 snake_case Naming
Database names should use `snake_case` for tables, fields, constraints, indexes, enum values, and generated names.

Rationale: This is the most conventional PostgreSQL naming style and keeps later implementation consistent across hand-written database work, migrations, and generated code.

### 1.5 Foreign Key Enforcement
Durable relationships should be enforced through database-level references where feasible. Application-only references should be reserved for polymorphic audit references, external provider identifiers, or references where the parent may live outside the local database.

Rationale: No orphaned business records is a core SRD data integrity requirement.

### 1.6 Soft Deletes
Master data and user-facing entities should generally support soft delete or deactivation rather than hard deletion once business history exists. Transactional, ledger, audit, and event records must not be hard-deleted under normal operation.

Rationale: Historical orders, reports, refunds, reconciliation, and audit records must remain explainable even after users, stores, products, workers, or promotions are disabled.

### 1.7 Append-Only History
Order events, payment events, inventory ledger entries, audit logs, and other operational histories must be append-only for business meaning. Corrections are represented by new records, not by rewriting the past.

Rationale: Support, finance, and operational reconciliation require a trustworthy history of what happened.

### 1.8 Immutable Ledgers
Inventory ledger and audit log records are immutable after creation except for strictly technical metadata correction approved by policy.

Rationale: Inventory and audit data must support dispute resolution and operational accountability.

### 1.9 ACID Transactions
Workflows that change multiple business records must complete atomically or leave no partial business effect. This applies especially to order creation, stock reservation, stock deduction, payment confirmation, refund creation, and reconciliation.

Rationale: The platform cannot allow confirmed orders without stock reservations, consumed stock without packed orders, or payment records without order context.

### 1.10 JSONB Usage
JSONB is appropriate for bounded flexible content such as bilingual product text, provider payload snapshots, address/geocoding metadata, settings payloads, or audit before/after snapshots. JSONB must not replace relational modeling for core entities, relationships, status, monetary values, store scope, or operational queues.

Rationale: The SRD explicitly supports multilingual product data and PostgreSQL JSONB, but normalized relational design remains the default.

### 1.11 Monetary Storage Strategy
Monetary values should be stored as exact minor-unit or fixed-precision values and always include currency context where needed. Floating point storage is not acceptable for money.

Rationale: Orders, payments, refunds, VAT, discounts, delivery fees, packaging fees, and reconciliation must be financially reliable.

### 1.12 Unicode and Arabic Support
All text fields must support Unicode. The database must preserve Arabic product content, customer-facing text, support notes, and operational notes without transliteration or lossy conversion.

Rationale: Arabic is a first-class Customer Platform MVP requirement.

### 1.13 Data Integrity Philosophy
The database should enforce structural integrity and durable invariants. The application should enforce workflow-specific rules that depend on current actor, external provider confirmation, timing, permissions, or multi-step domain logic.

Rationale: Database constraints prevent corrupt data; application validation preserves domain intent without overloading the schema with workflow logic.

---

## 2. Naming Conventions

### 2.1 Tables
Use plural, lower-case, snake_case table names for entity collections. Use clear domain names over abbreviations.

Examples of naming style: `orders`, `order_items`, `inventory_reservations`, `support_tickets`.

### 2.2 Columns
Use lower-case, snake_case names. Column names should describe business meaning, not UI labels or implementation details.

Common names:
- `id` for primary identifier.
- `<entity>_id` for references.
- `status` for lifecycle state where a single state machine applies.
- `created_at`, `updated_at`, `deleted_at` for standard timestamps.
- `created_by_id`, `updated_by_id`, `deleted_by_id` where actor tracking matters.

### 2.3 Primary Keys
Primary keys should be named `id` and use ULIDs for major entities. Join or association entities may also use ULIDs when they carry lifecycle, audit, or metadata.

### 2.4 Foreign Keys
Foreign key columns should use the referenced singular entity name followed by `_id`. Store-scoped entities should include explicit store reference when store isolation matters.

### 2.5 Join Tables
Use descriptive names that reflect the relationship. If the association has business meaning, status, timestamps, or auditability, model it as a full entity rather than a simple join.

### 2.6 Enum Naming
Enum-like values should use lower-case snake_case values. Names must match SRD terminology closely enough that engineering, product, and operations can recognize lifecycle states.

### 2.7 Boolean Naming
Boolean fields should use positive names such as `is_active`, `is_deleted`, `is_default`, `is_verified`, or `requires_approval`. Avoid negative names that create double negatives.

### 2.8 Timestamp Naming
Use `created_at` and `updated_at` consistently. Domain timestamps should describe the business event: `confirmed_at`, `packed_at`, `delivered_at`, `closed_at`, `reconciled_at`.

### 2.9 Audit Columns
Entities that can be changed by admin, support, ops, finance, or workers should include actor/context references where useful. Append-only audit logs remain the authoritative cross-entity audit trail.

### 2.10 Constraint Naming Philosophy
Constraint and index names should be deterministic, descriptive, and stable. Generated names are acceptable only if the migration tooling makes them readable and predictable.

---

## 3. Global Database Decisions

### 3.1 ULID vs Sequential IDs
Use ULIDs for primary business identifiers. Sequential IDs can leak order volume, customer count, worker count, or store operating scale if exposed. ULIDs also make future offline or distributed creation less risky, and their lexicographic sortability is treated as a benefit rather than a leak — it reveals relative creation order, not an exact count.

Human-friendly order numbers may be added separately for support and customer communication. They should not replace internal ULID identity.

### 3.2 JSONB Usage
JSONB should be used selectively:

- Bilingual product name/description content.
- Provider response snapshots for payments, SMS, push, geocoding, or POS reconciliation.
- Settings payloads where value shape differs by setting type.
- Audit before/after snapshots.
- Address/geocoding metadata that is not central to relational joins.

JSONB should not hold order items, inventory quantities, payment totals, state machine status, store ownership, worker assignment, or core relationships.

### 3.3 Composite Keys
Composite uniqueness is required where business identity is scoped by another entity. Examples include product inventory per store, worker capability per worker, promotion usage per promotion/customer/order where needed, and unique active default address per customer.

Rationale: Some business facts are unique only inside a store, customer, worker, or promotion context.

### 3.4 Foreign Keys
Foreign keys should enforce parent/child relationships for durable data. Restrictive behavior is preferred for historical parents; disabling or archiving should be used instead of deleting referenced entities.

Rationale: Historical orders must remain readable even if products, workers, stores, or customers are deactivated.

### 3.5 Soft Delete Strategy
Use soft delete for master data and profile-like records. Use deactivation when the business meaning is closer to "not currently usable" than "deleted."

Never soft delete as a substitute for lifecycle state. For example, a cancelled order is an order state, not a deleted order.

### 3.6 Append-Only Audit
Audit records must be append-only and actor-linked. Important admin/support/ops changes should record what changed, who changed it, when it changed, and the affected entity.

### 3.7 Ledger Tables
Inventory movements and financial reconciliation facts should be modeled as ledger/history records instead of only overwriting current state.

Rationale: Current state answers "what is true now"; ledgers answer "why is it true."

### 3.8 Why PostgreSQL Was Selected
PostgreSQL supports:

- ACID transactions.
- Strong referential integrity.
- JSONB for multilingual and provider payload needs.
- Readable relational modeling.
- Rollups/views for MVP analytics.
- RDS managed operations in the required AWS Saudi region.

This matches the SRD's solo-developer, low-thousands-orders-per-day, Saudi-only MVP constraints.

### 3.9 Why NoSQL Is Not Used
NoSQL is not required for the MVP domain. The platform needs relational consistency across customer, store, inventory, order, payment, refund, and worker data. NoSQL would increase operational complexity and weaken natural business relationships without solving an SRD requirement.

### 3.10 Why Redis Is Not Authoritative
Redis supports locks, cache, session-like coordination, geospatial rider data assistance, and short-lived retry state. It must not be the durable source for inventory, orders, payments, refunds, shifts, or audit logs.

Rationale: Redis state can expire, be evicted, or be rebuilt. Financial and operational records must survive process restarts, cache flushes, and incident recovery.

---

## 4. Entity Catalog

| Entity | Module Owner | Category | Mutable | Soft Delete | Audit Required | History Required | Scope |
|---|---|---|---|---|---|---|---|
| User | Auth & Identity | Master Data | Yes | Deactivate | Yes | Yes | Global |
| Customer Profile | Customer | Master Data | Yes | Soft delete/anonymize by policy | Yes | Yes | Global |
| Address | Customer | Master Data | Yes | Yes | For support changes | Snapshot on order | Customer-scoped |
| Worker Profile | Workforce | Operational Master | Yes | Deactivate | Yes | Yes | Store-scoped by assignment |
| Worker Capability | Workforce | Configuration | Yes | No; deactivate/remove with audit | Yes | Yes | Worker/store-scoped |
| Shift | Workforce | Operational | Yes | No | Yes | Yes | Store-scoped |
| Worker Availability | Workforce | Operational History | Yes | No | For admin-forced changes | Yes | Worker/store-scoped |
| Store | Store | Master Data | Yes | Deactivate | Yes | Yes | Global operating unit |
| Service Zone | Store | Configuration | Yes | No; historical orders snapshot store | Yes | Yes | Store-scoped |
| Category | Catalog | Master Data | Yes | Deactivate | Yes | Yes | Global |
| Brand | Catalog | Reference | Yes | Deactivate | Yes | Optional | Global |
| Product | Catalog | Master Data | Yes | Deactivate | Yes | Yes | Global |
| Product Image | Catalog | Master Data | Yes | Deactivate | Yes | Optional | Global |
| Inventory Item | Inventory | Operational Master | Yes | Deactivate | Yes | Yes | Store-scoped |
| Inventory Reservation | Inventory | Transactional | State-driven | No | Yes for manual changes | Yes | Store/order-scoped |
| Inventory Ledger | Inventory | Ledger | Immutable | No | Inherent | Yes | Store-scoped |
| Cart | Cart | Transactional Draft | Yes | Expire | No unless support-owned | Limited | Customer/store-scoped |
| Order | Order | Transactional | State-driven | No | Yes | Yes | Store-scoped |
| Order Item | Order | Transactional | State-driven | No | Yes | Yes | Store-scoped |
| Order Event | Order | History | Immutable | No | Inherent | Yes | Store/order-scoped |
| Picking Session | Picking | Operational | State-driven | No | Yes | Yes | Store-scoped |
| Picking Session Item | Picking | Operational | State-driven | No | Yes | Yes | Store-scoped |
| Delivery Assignment | Delivery | Operational | State-driven | No | Yes | Yes | Store-scoped |
| Rider Location | Delivery | Operational History | Append/update by policy | Expire/archive | Limited | Short-term | Worker/order-scoped |
| Payment | Payment | Transactional Finance | State-driven | No | Yes | Yes | Order-scoped |
| Payment History | Payment | History | Immutable | No | Inherent | Yes | Order-scoped |
| Cash Remittance | Payment | Finance Operational | State-driven | No | Yes | Yes | Store/shift-scoped |
| Card-on-Delivery Record | Payment | Finance Operational | State-driven | No | Yes | Yes | Store/shift-scoped |
| Refund | Refund | Transactional Finance | State-driven | No | Yes | Yes | Order/payment-scoped |
| Promotion | Promotion | Configuration | State-driven | Archive | Yes | Yes | Global/store-scoped |
| Promo Code | Promotion | Configuration | State-driven | Archive | Yes | Yes | Global/store-scoped |
| Promotion Usage | Promotion | Transactional History | Immutable | No | Inherent | Yes | Customer/order-scoped |
| Price Snapshot | Pricing | Transactional Snapshot | Immutable after finalization | No | Yes for correction | Yes | Order/item-scoped |
| Notification | Notification | Operational History | Status-driven | Archive/expire | Limited | Yes | User/order-scoped |
| Support Ticket | Support | Operational | State-driven | Close/archive | Yes | Yes | Customer/order-scoped |
| Support Ticket Comment | Support | History | Append-only | No | Inherent | Yes | Ticket-scoped |
| Settings | Administration | Configuration | Yes | Version/archive | Yes | Yes | Global/store-scoped |
| Audit Log | Audit / Compliance | Audit | Immutable | No | Inherent | Yes | Global/store-aware |
| Report Rollup | Reporting & Analytics | Analytics | Rebuildable | Archive | No unless manual correction | Source history preserved | Global/store-scoped |
| Analytics Snapshot | Reporting & Analytics | Analytics | Rebuildable/immutable by period | Archive | No unless manual correction | Source history preserved | Global/store-scoped |

---

## 5. Entity Specifications

### 5.1 User

**Purpose**  
Represents a person or staff account that can authenticate or be referenced as an actor.

**Ownership**  
Auth & Identity Module.

**Responsibilities**  
Identity, credentials/login method, role linkage, active state, and actor references for audit/support actions.

**Core Fields**  
Identifier, phone/email identity, authentication method, role flags or role references, active/deactivated state, verification state, security timestamps, optional platform/device session metadata, and standard audit timestamps.

**Relationships**  
One user may have a customer profile, worker profile, admin role context, support actions, audit actions, notifications, and sessions/tokens managed outside this DDD.

**Lifecycle**  
Created during customer/worker OTP onboarding or admin account creation. Deactivated instead of hard-deleted once business history exists.

**Constraints**  
Authentication identity must be unique within its login method. Customer Mobile App and Customer Web App use the same OTP/JWT customer identity. Tokens are platform-independent; platform metadata must not create separate customer accounts or business rules. Disabled users cannot authenticate or receive new operational tasks.

**Retention**  
Retained long-term where referenced by orders, shifts, support, audit, or finance.

**Audit Requirements**  
Role changes, activation/deactivation, credential resets, admin account creation, and permission changes must be audited.

**Notes**  
Customer privacy requests may require anonymization workflows; historical business records still need stable references.

### 5.2 Customer Profile

**Purpose**  
Represents the customer-facing profile used for ordering, support, notifications, and order history.

**Ownership**  
Customer Module.

**Responsibilities**  
Customer contact context, display identity, account status, support context, order history association, and shared Customer Platform profile state across web and mobile.

**Core Fields**  
Profile identifier, user reference, display name, preferred language, contact preferences, status, and timestamps.

**Relationships**  
Belongs to a user. Owns addresses, carts, orders, support tickets, notifications, and promotion usage.

**Lifecycle**  
Created on first customer login or checkout onboarding. Updated by customer or support. Soft-deleted/anonymized only according to privacy policy.

**Constraints**  
Customer profile, addresses, order history, favorites where implemented, notifications, promotions, support tickets, and cart state are shared across Customer Mobile App and Customer Web App. Customer orders and support history must remain accessible after account deactivation.

**Retention**  
Retained according to PDPL/privacy obligations and support/accounting needs.

**Audit Requirements**  
Support-driven profile changes and account status changes are audited.

**Notes**  
The profile is not the same as the user authentication record.

### 5.3 Address

**Purpose**  
Represents a customer delivery location and routing input.

**Ownership**  
Customer Module.

**Responsibilities**  
Saved address management, geocoding metadata, service-zone eligibility, and order-time address snapshot source.

**Core Fields**  
Customer reference, address label, contact recipient, text address, location coordinates, geocoding metadata, default flag, active state, and timestamps.

**Relationships**  
Belongs to customer. Referenced by carts and used to create immutable order address snapshots.

**Lifecycle**  
Created/updated/deleted by customer or support. Historical orders retain order-time snapshot.

**Constraints**  
Checkout requires a deliverable address that resolves to exactly one serving store after overlap rules.

**Retention**  
Active saved addresses retained until deleted/anonymized by policy. Order snapshots retained with orders.

**Audit Requirements**  
Customer self-service edits generally do not need full audit logs. Support edits require audit.

**Notes**  
Address validation depends on Google Maps/geocoding but database remains the holder of the selected result.

### 5.4 Store

**Purpose**  
Represents a physical dark store serving a neighborhood radius.

**Ownership**  
Store Module.

**Responsibilities**  
Store identity, location, operating status, service radius ownership, inventory boundary, worker home assignment, order routing, and reconciliation scope.

**Core Fields**  
Store identifier, name, location coordinates, contact/ops metadata, operating state, opening hours reference, active state, and timestamps.

**Relationships**  
Owns service zone, inventory items, orders, shifts, picking sessions, delivery assignments, cash reconciliation, store settings, and reporting scopes.

**Lifecycle**  
Created/updated by admin. Temporarily closed by ops. Deactivated instead of deleted when historical records exist.

**Constraints**  
Orders, inventory, worker queues, rider pools, and reconciliation must be store-scoped.

**Retention**  
Retained long-term because historical reports and orders depend on store identity.

**Audit Requirements**  
Store creation, location/radius changes, status changes, opening-hour changes, and deactivation are audited.

**Notes**  
Multi-store from launch is a core architecture decision.

### 5.5 Service Zone

**Purpose**  
Represents a store's delivery coverage rule.

**Ownership**  
Store Module.

**Responsibilities**  
Radius-based address eligibility and store assignment for MVP.

**Core Fields**  
Store reference, zone type, radius value, active state, effective period, and timestamps.

**Relationships**  
Belongs to store. Used by address/order routing. Historical orders retain assigned store rather than recalculating later.

**Lifecycle**  
Created/updated by admin. Current version governs new routing only.

**Constraints**  
If multiple zones match, nearest store wins. If none match, checkout is blocked.

**Retention**  
Historical zone configurations should be retained or versioned enough to explain past routing.

**Audit Requirements**  
All zone/radius changes are audited.

**Notes**  
Polygon zones are deferred.

### 5.6 Worker Profile

**Purpose**  
Represents staff using the Worker App.

**Ownership**  
Workforce Module.

**Responsibilities**  
Store assignment, task eligibility, capability linkage, shift participation, availability, and reconciliation context.

**Core Fields**  
User reference, home store reference, status, staff metadata, current availability summary, and timestamps.

**Relationships**  
Belongs to user and store. Owns capabilities, shifts, picking sessions, delivery assignments, location updates, and cash remittances.

**Lifecycle**  
Created by admin only. Updated by admin/ops. Deactivated rather than deleted once used operationally.

**Constraints**  
Worker cannot self-register or self-select store. Tasks must respect store assignment and capability.

**Retention**  
Retained long-term for operations, finance, and productivity reports.

**Audit Requirements**  
Creation, capability changes, store reassignment, activation/deactivation, and admin availability changes are audited.

**Notes**  
Picker/rider are capabilities, not separate apps.

### 5.7 Worker Capability

**Purpose**  
Defines whether a worker may operate as picker, rider, or both.

**Ownership**  
Workforce Module.

**Responsibilities**  
Mode eligibility, task filtering, and assignment rules.

**Core Fields**  
Worker reference, capability type, store context if applicable, active state, effective timestamps, and actor references.

**Relationships**  
Belongs to worker. Referenced by picking and delivery eligibility.

**Lifecycle**  
Created/updated by admin. Historical tasks retain the capability context at assignment time.

**Constraints**  
No active task conflict across modes. Capability changes do not mutate completed task history.

**Retention**  
Retained or historized for task auditability.

**Audit Requirements**  
All capability grants/removals are audited.

**Notes**  
Useful for future worker-role expansion without app split.

### 5.8 Shift

**Purpose**  
Represents a worker's operational work period.

**Ownership**  
Workforce Module.

**Responsibilities**  
Assignment eligibility, productivity reporting, cash reconciliation window, and attendance context.

**Core Fields**  
Worker reference, store reference, lifecycle state, start/end timestamps, pause/reconciliation timestamps, and summary metrics.

**Relationships**  
Belongs to worker/store. Has availability states, picking sessions, delivery assignments, cash remittances, and productivity rollups.

**Lifecycle**  
Follows SRD Shift State Machine: Created, Scheduled, Started, Active, Paused, Ended, Reconciled, Closed, Cancelled.

**Constraints**  
New assignments require active shift and eligible availability. Shift cannot close with unresolved reconciliation when cash/card-on-delivery obligations exist.

**Retention**  
Retained for productivity, finance, and operational reporting.

**Audit Requirements**  
Admin shift edits, reconciliation, closure, and forced state changes are audited.

**Notes**  
Availability is separate from shift lifecycle.

### 5.9 Category

**Purpose**  
Organizes products for browsing, reporting, and promotion eligibility.

**Ownership**  
Catalog Module.

**Responsibilities**  
Catalog hierarchy, display grouping, and category-level promotion/reporting support.

**Core Fields**  
Category identifier, multilingual name, parent category reference where needed, display order, active state, and timestamps.

**Relationships**  
Parent of products. May be referenced by promotion rules and analytics.

**Lifecycle**  
Created/updated by catalog admin. Deactivated rather than deleted when referenced.

**Constraints**  
Category changes must not alter historical order meaning.

**Retention**  
Retained long-term for product/order/report history.

**Audit Requirements**  
Admin changes are audited.

**Notes**  
Category hierarchy should remain simple for MVP.

### 5.10 Brand

**Purpose**  
Optional catalog reference for product manufacturer/brand.

**Ownership**  
Catalog Module.

**Responsibilities**  
Brand-level browsing, filtering, supplier promotion reporting, and future supplier integrations.

**Core Fields**  
Brand identifier, name, optional multilingual display data, active state, and timestamps.

**Relationships**  
Referenced by products and potentially supplier-funded promotions.

**Lifecycle**  
Created/updated by catalog admin. Deactivated rather than deleted if products reference it.

**Constraints**  
Brand is not required for every MVP product unless catalog operations decide otherwise.

**Retention**  
Retained while products or reports reference it.

**Audit Requirements**  
Admin changes are audited.

**Notes**  
Included because the DDD should support a realistic product catalog even if the SRD does not make brand mandatory for MVP.

### 5.11 Product

**Purpose**  
Represents the global sellable item definition.

**Ownership**  
Catalog Module.

**Responsibilities**  
Product identity, multilingual customer-facing content, unit, taxability, category/brand linkage, and active state.

**Core Fields**  
Product identifier, SKU/barcode where available, multilingual name/description, unit, category reference, brand reference, tax classification, active state, and timestamps.

**Relationships**  
Belongs to category/brand. Has product images, inventory items, order items, price snapshots, and promotion eligibility.

**Lifecycle**  
Created/updated by catalog admin. Disabled when no longer sold. Not hard-deleted once referenced.

**Constraints**  
Global product existence does not imply store availability. Sellability is determined by store inventory and status.

**Retention**  
Retained long-term for historical order meaning.

**Audit Requirements**  
Product content, taxability, active state, and category changes are audited.

**Notes**  
Multilingual data should use JSONB or equivalent flexible structure.

### 5.12 Product Image

**Purpose**  
Represents image assets for products.

**Ownership**  
Catalog Module.

**Responsibilities**  
Image references, display order, primary image designation, and active/inactive state.

**Core Fields**  
Product reference, storage reference, alt/display metadata, sort order, active state, and timestamps.

**Relationships**  
Belongs to product.

**Lifecycle**  
Created/updated by catalog admin. Disabled rather than deleted if historical display context is needed.

**Constraints**  
Missing image must not block operational order flow.

**Retention**  
Retained according to catalog and storage cleanup policy.

**Audit Requirements**  
Admin image changes are audited where they affect customer-facing catalog.

**Notes**  
Actual object storage is outside the database but references must be durable.

### 5.13 Inventory Item

**Purpose**  
Represents store-specific stock state for a product.

**Ownership**  
Inventory Module.

**Responsibilities**  
On-hand stock, reserved stock, available stock, low-stock threshold, shelf metadata, store sellability, and reconciliation state.

**Core Fields**  
Store reference, product reference, status, on-hand quantity, reserved quantity, threshold, shelf/aisle metadata, last sync/reconciliation context, and timestamps.

**Relationships**  
Belongs to store and product. Has reservations and ledger entries. Referenced by picking session items.

**Lifecycle**  
Created when product is stocked at a store. Updated by reservation, packing, adjustment, sync, and reconciliation workflows. Disabled rather than deleted when no longer stocked.

**Constraints**  
Inventory is unique per store/product. Available quantity cannot be negative. Store isolation is mandatory.

**Retention**  
Retained while product/store/order history requires it. Ledger preserves movement history.

**Audit Requirements**  
Manual adjustments, status changes, thresholds, and reconciliation changes are audited.

**Notes**  
Current row is optimized for reads; ledger explains movement history.

### 5.14 Inventory Reservation

**Purpose**  
Represents stock held for an order before packing.

**Ownership**  
Inventory Module.

**Responsibilities**  
Checkout oversell prevention, item-level reservation, substitution adjustment, release, consumption, and timeout handling.

**Core Fields**  
Order reference, order item reference, inventory item reference, reserved quantity, state, expiry context, and timestamps.

**Relationships**  
Belongs to order/order item/inventory item. Produces or references inventory ledger entries.

**Lifecycle**  
Follows SRD Inventory Reservation State Machine: Created, Reserved, Adjusted, Released, Consumed, Expired, Cancelled.

**Constraints**  
Cannot be consumed before packing. Cannot be released after consumption except through separate inventory restoration.

**Retention**  
Retained with order and inventory history.

**Audit Requirements**  
Manual release, adjustment, cancellation, and expiry correction are audited.

**Notes**  
Redis may coordinate reservation locks, but PostgreSQL stores the authoritative reservation.

### 5.15 Inventory Ledger

**Purpose**  
Append-only explanation of inventory movements.

**Ownership**  
Inventory Module.

**Responsibilities**  
Records reservation, release, deduction, restoration, adjustment, damaged stock, expired stock, returned stock, and sync movements.

**Core Fields**  
Inventory item reference, store/product context, movement type, quantity change, reason code, source workflow, actor reference, related order where applicable, and timestamp.

**Relationships**  
Belongs to inventory item. May reference order, order item, worker, admin/support user, or sync job context.

**Lifecycle**  
Append-only and immutable.

**Constraints**  
Every stock-changing operation must create ledger history.

**Retention**  
Retained long-term.

**Audit Requirements**  
Ledger is itself operational audit; admin-initiated changes also appear in audit log.

**Notes**  
The ledger should make current inventory explainable.

### 5.16 Cart

**Purpose**  
Represents a customer's server-side pre-order basket shared across the Customer Platform.

**Ownership**  
Cart Module.

**Responsibilities**  
Draft item selection, quantity changes, pricing preview, promotion preview, cross-client cart continuity, and checkout preparation.

**Core Fields**  
Customer reference, serving store reference, address context, cart state, item list represented relationally, pricing preview context, and expiry timestamps.

**Relationships**  
Belongs to customer/store. References products and promotions. Converts into order through checkout.

**Lifecycle**  
Created during browsing. Updated by customer. Cleared or expired after order creation.

**Constraints**  
Cart is not financially authoritative. Customer Mobile App and Customer Web App must read/write the same server-side cart for the same customer and serving store. Checkout must revalidate inventory, price, address, store, and promotion.

**Retention**  
Short-term retention; abandoned carts may be expired.

**Audit Requirements**  
Not generally audited unless support modifies it.

**Notes**  
Do not store final order truth only in cart. Do not create platform-specific cart tables unless a future requirement introduces platform-specific draft behavior.

### 5.17 Order

**Purpose**  
Represents a committed customer purchase request routed to one store.

**Ownership**  
Order Module.

**Responsibilities**  
Order lifecycle state, customer/store linkage, fulfillment coordination, payment summary, support context, and reporting source.

**Core Fields**  
Customer reference, store reference, address snapshot, lifecycle state, order number, totals summary, payment method summary, timestamps for major milestones, cancellation/failure context, and actor references where applicable.

**Relationships**  
Has order items, price snapshots, inventory reservations, picking sessions, delivery assignments, payments, refunds, support tickets, notifications, and order events.

**Lifecycle**  
Follows SRD Order State Machine from Created through Completed/Cancelled/Refunded/Returned paths.

**Constraints**  
Order cannot exist without customer, store, address snapshot, and at least one order item. State transitions must not skip required payment, inventory, picking, packing, delivery, or refund steps.

**Retention**  
Retained long-term for support, accounting, reconciliation, and analytics.

**Audit Requirements**  
Support/admin changes, cancellation, refund-triggering changes, and state corrections are audited. Normal lifecycle history is preserved through order events.

**Notes**  
Order totals must be based on snapshots, not current product/pricing data.

### 5.18 Order Item

**Purpose**  
Represents an item requested, substituted, removed, or fulfilled within an order.

**Ownership**  
Order Module.

**Responsibilities**  
Requested quantity, fulfilled quantity, substitution outcome, item-level price snapshot, refund context, and picking/inventory linkage.

**Core Fields**  
Order reference, product reference, inventory/store context, requested quantity, fulfilled quantity, item status, substitution context, price snapshot reference, and timestamps.

**Relationships**  
Belongs to order and product. Linked to inventory reservation, picking session item, and refund lines where needed.

**Lifecycle**  
Created with order. Updated during picking/substitution. Finalized when order is packed/completed/cancelled.

**Constraints**  
Historical product and price meaning must remain stable even if catalog changes.

**Retention**  
Retained with order.

**Audit Requirements**  
Substitution, removal, refund-affecting changes, and support changes are audited or recorded in order events.

**Notes**  
Use item-level snapshots to support refunds and finance.

### 5.19 Order Event

**Purpose**  
Append-only order timeline.

**Ownership**  
Order Module.

**Responsibilities**  
Lifecycle transitions, operational notes, support traceability, SLA measurement, and correction history.

**Core Fields**  
Order reference, event type, prior state, new state, actor/context, reason, note, and timestamp.

**Relationships**  
Belongs to order. May reference worker, support/admin user, payment, refund, or delivery assignment context.

**Lifecycle**  
Append-only.

**Constraints**  
Corrections append events; prior events are not erased.

**Retention**  
Retained with order.

**Audit Requirements**  
Event history is operational history; sensitive admin changes also create audit logs.

**Notes**  
Order events support reporting and support timelines.

### 5.20 Picking Session

**Purpose**  
Represents picker ownership of an order's picking/packing work.

**Ownership**  
Picking Module.

**Responsibilities**  
Claim, picking progress, substitution waiting, packing completion, expiry, and release.

**Core Fields**  
Order reference, picker worker reference, store reference, state, claim/start/complete timestamps, timeout context, and release/cancel reason.

**Relationships**  
Belongs to order, worker, and store. Has picking session items.

**Lifecycle**  
Follows SRD Picking Session State Machine.

**Constraints**  
Only one active picking session may own an order. Picker and order must belong to same store.

**Retention**  
Retained for operations and worker productivity reports.

**Audit Requirements**  
Admin release/reassignment and cancellation are audited.

**Notes**  
Timeout jobs rely on state and timestamps.

### 5.21 Picking Session Item

**Purpose**  
Represents item-level picking outcome.

**Ownership**  
Picking Module.

**Responsibilities**  
Picked quantity, unavailable marking, substitution request, customer response, and packing readiness.

**Core Fields**  
Picking session reference, order item reference, picked quantity, outcome status, substitution context, customer response state, and timestamps.

**Relationships**  
Belongs to picking session and order item. References inventory item where needed.

**Lifecycle**  
Created with picking session. Finalized before packing completion.

**Constraints**  
Packing cannot complete until every item is resolved.

**Retention**  
Retained with picking session and order.

**Audit Requirements**  
Substitution and support/admin corrections are audited or evented.

**Notes**  
Useful for picker productivity and out-of-stock analytics.

### 5.22 Delivery Assignment

**Purpose**  
Represents rider responsibility for a packed order.

**Ownership**  
Delivery Module.

**Responsibilities**  
Assignment, acceptance, rejection, expiry, pickup, delivery, failed delivery, and cancellation.

**Core Fields**  
Order reference, rider worker reference, store reference, state, assignment/accept/pickup/delivery timestamps, failure reason, OTP/proof status, and reassignment context.

**Relationships**  
Belongs to order, rider, and store. References payment collection and rider location history.

**Lifecycle**  
Follows SRD Delivery Assignment State Machine.

**Constraints**  
Assignment cannot exist before order is packed. Rider pool is store-scoped for MVP. One active delivery per rider unless batching is introduced later.

**Retention**  
Retained for SLA, support, failed delivery, and productivity reports.

**Audit Requirements**  
Manual reassignment, cancellation, failed delivery support corrections, and admin overrides are audited.

**Notes**  
Rejected/expired assignments return to same store pool.

### 5.23 Rider Location

**Purpose**  
Represents rider position while online or active on delivery.

**Ownership**  
Delivery Module.

**Responsibilities**  
Live tracking, ETA support, operational dispute review, and active delivery context.

**Core Fields**  
Rider reference, delivery assignment/order reference where applicable, location coordinates, accuracy/metadata, captured timestamp, and retention marker.

**Relationships**  
Belongs to rider and optionally delivery assignment/order.

**Lifecycle**  
Created while rider is online/active. Retained short-term.

**Constraints**  
Location collection is limited to online/active states and PDPL review.

**Retention**  
Short-term only unless needed for dispute/safety/legal retention.

**Audit Requirements**  
Normal location updates are not admin audit logs. Access/export policies may require audit later.

**Notes**  
Do not use location history as a long-term analytics substitute without privacy review.

### 5.24 Payment

**Purpose**  
Represents the financial obligation and settlement status for an order.

**Ownership**  
Payment Module.

**Responsibilities**  
Payment method, amount due/paid, status, gateway/POS/COD evidence, reconciliation linkage, and refund basis.

**Core Fields**  
Order reference, payment method, state, amount summary, currency, provider reference, verification metadata snapshot, paid/failed/cancelled timestamps, and reconciliation context.

**Relationships**  
Belongs to order. Has payment history and refunds. May connect to cash remittance or card-on-delivery record.

**Lifecycle**  
Follows SRD Payment State Machine.

**Constraints**  
Paid state requires verified gateway confirmation, rider cash collection, or card-on-delivery record. Client-supplied payment status is not authoritative.

**Retention**  
Retained long-term for finance, support, disputes, and accounting.

**Audit Requirements**  
Payment state corrections, reconciliation changes, and manual finance actions are audited.

**Notes**  
Payment provider payload snapshots can use JSONB.

### 5.25 Payment History

**Purpose**  
Append-only history of payment state changes and provider results.

**Ownership**  
Payment Module.

**Responsibilities**  
Payment timeline, gateway response evidence, rider collection evidence, and finance traceability.

**Core Fields**  
Payment reference, event type, prior/new state, provider metadata, actor/context, amount where relevant, and timestamp.

**Relationships**  
Belongs to payment/order.

**Lifecycle**  
Append-only.

**Constraints**  
Payment corrections append history.

**Retention**  
Retained with payment.

**Audit Requirements**  
Inherent finance history; manual corrections also audited.

**Notes**  
Do not store raw card data.

### 5.26 Cash Remittance

**Purpose**  
Represents rider cash handover and discrepancy resolution.

**Ownership**  
Payment Module.

**Responsibilities**  
Expected cash, remitted cash, discrepancy, reason, acknowledgement, and shift/store reporting.

**Core Fields**  
Rider reference, shift reference, store reference, expected amount, remitted amount, discrepancy amount, status, reason, acknowledged by, and timestamps.

**Relationships**  
Belongs to rider/shift/store. Summarizes COD payments.

**Lifecycle**  
Created during reconciliation. Updated until acknowledged/closed.

**Constraints**  
Cash and card-on-delivery are separate reconciliation streams.

**Retention**  
Retained long-term for finance.

**Audit Requirements**  
All remittance creation, changes, discrepancy acknowledgement, and closure actions are audited.

**Notes**  
Finance reports depend on this entity.

### 5.27 Card-on-Delivery Record

**Purpose**  
Represents rider-recorded POS terminal payment at delivery.

**Ownership**  
Payment Module.

**Responsibilities**  
Expected amount, recorded amount, terminal/provider reference where available, rider confirmation, and reconciliation.

**Core Fields**  
Payment/order reference, rider/shift/store reference, recorded amount, terminal metadata, status, discrepancy context, and timestamps.

**Relationships**  
Belongs to payment/order and shift/store. May be reconciled against terminal provider reports.

**Lifecycle**  
Created when customer selected Card-on-Delivery or rider records terminal payment. Reconciled during shift/day close.

**Constraints**  
Must not be conflated with cash.

**Retention**  
Retained long-term for finance and dispute handling.

**Audit Requirements**  
Manual edits and reconciliation actions are audited.

**Notes**  
Terminal settlement API support is still a Phase 1 spike.

### 5.28 Refund

**Purpose**  
Represents money or manual compensation owed back to customer.

**Ownership**  
Refund Module.

**Responsibilities**  
Eligibility, amount, reason, approval, processing, settlement, and failure tracking.

**Core Fields**  
Order/payment reference, refund amount, currency, reason, state, approval actor, settlement context, failure reason, and timestamps.

**Relationships**  
Belongs to order/payment. May relate to order items and support ticket.

**Lifecycle**  
Follows SRD Refund State Machine.

**Constraints**  
Refund amount uses order price snapshots, not current catalog price. Completed refunds are immutable except corrective refund records.

**Retention**  
Retained long-term for finance/accounting.

**Audit Requirements**  
Request, approval, rejection, cancellation, failure, and completion are audited.

**Notes**  
COD/manual refunds need explicit finance resolution context.

### 5.29 Promotion

**Purpose**  
Represents a discount campaign or rule family.

**Ownership**  
Promotion Module.

**Responsibilities**  
Promotion lifecycle, eligibility scope, funding context, type, validity, and reporting classification.

**Core Fields**  
Promotion name, type, lifecycle state, eligibility scope, date/time validity, funding/source context, limits, and timestamps.

**Relationships**  
May own promo codes, promotion rules, usage records, store/product/category eligibility references, and order price snapshots.

**Lifecycle**  
Follows SRD Promotion Lifecycle: Draft, Scheduled, Active, Paused, Expired, Archived.

**Constraints**  
Paused, expired, and archived promotions cannot apply to new orders. Applied promotion data remains attached to historical orders.

**Retention**  
Archived for reporting/history.

**Audit Requirements**  
All lifecycle and rule changes are audited.

**Notes**  
MVP may expose promo codes only, but design supports future promotion types.

### 5.30 Promo Code

**Purpose**  
Represents customer-entered MVP discount code.

**Ownership**  
Promotion Module.

**Responsibilities**  
Code value, eligibility, usage limits, validity, and link to promotion benefit.

**Core Fields**  
Promotion reference, code, state, validity window, per-customer usage limit, total usage limit, minimum spend, store/customer eligibility context, and timestamps.

**Relationships**  
Belongs to promotion. Has promotion usage records. Applied to carts/orders.

**Lifecycle**  
Uses promotion lifecycle or a compatible code lifecycle.

**Constraints**  
Code must be unique within active eligibility context. Must be revalidated at order creation.

**Retention**  
Archived after expiry/deactivation.

**Audit Requirements**  
Creation, edits, pauses, expiry overrides, and archiving are audited.

**Notes**  
Usage history must survive code archive.

### 5.31 Promotion Usage

**Purpose**  
Records that a promotion or promo code was applied to an order/customer.

**Ownership**  
Promotion Module.

**Responsibilities**  
Usage limit enforcement, reporting, customer eligibility, and finance traceability.

**Core Fields**  
Promotion/promo code reference, customer reference, order reference, discount amount, eligibility snapshot, usage timestamp.

**Relationships**  
Belongs to promotion/promo code, customer, and order.

**Lifecycle**  
Created after successful order creation. Immutable.

**Constraints**  
Must not be created for failed or abandoned checkout.

**Retention**  
Retained with order/promotion history.

**Audit Requirements**  
Inherent transactional history; manual corrections audited.

**Notes**  
Supports first-order and usage-limit rules.

### 5.32 Price Snapshot

**Purpose**  
Captures final price facts used for orders, items, payments, and refunds.

**Ownership**  
Pricing Module.

**Responsibilities**  
Product price, store-specific price, VAT, fees, discounts, substitutions, final paid amount, and refund basis.

**Core Fields**  
Order/order item reference, price components, currency, VAT context, promotion/coupon deductions, delivery/packaging fee context, and timestamp.

**Relationships**  
Belongs to order/order item. Referenced by payment, refund, reporting, and support.

**Lifecycle**  
Created at order placement and adjusted only through explicit substitution/refund events.

**Constraints**  
Historical totals cannot change when current catalog or promotion rules change.

**Retention**  
Retained with order long-term.

**Audit Requirements**  
Corrections are audited and evented.

**Notes**  
This is essential for finance correctness.

### 5.33 Notification

**Purpose**  
Represents attempted or delivered customer/worker/admin message.

**Ownership**  
Notification Module.

**Responsibilities**  
Push, SMS, in-app/in-platform history, delivery status, retry context, platform capability context, and user-visible notification history.

**Core Fields**  
Recipient user, channel, message type, related entity, status, provider metadata, retry count/context, platform capability metadata where needed, sent/delivered/failed timestamps.

**Relationships**  
Belongs to user and may reference order, assignment, support ticket, or auth workflow.

**Lifecycle**  
Created when notification is requested. Updated with send/delivery/failure state. Archived/expired by policy.

**Constraints**  
Business notification events are platform-independent. Delivery channel depends on capability: mobile push for Customer Mobile App, browser notifications as a future Customer Web App channel, SMS for OTP/delivery confirmation, and email as a future channel. Business state must not depend only on notification delivery.

**Retention**  
Short-term for troubleshooting and in-app history.

**Audit Requirements**  
Sensitive/admin notification actions may be audited; routine lifecycle status does not require audit log.

**Notes**  
Provider payloads can be stored as bounded JSONB metadata.

### 5.34 Support Ticket

**Purpose**  
Represents customer, worker, order, payment, refund, delivery, or inventory issue.

**Ownership**  
Support Module.

**Responsibilities**  
Issue state, owner, priority, reason, notes, escalation, resolution, and linked business actions.

**Core Fields**  
Requester/customer/worker reference, related order/payment/refund/delivery references, state, reason, priority, assigned user, resolution context, and timestamps.

**Relationships**  
May link to customer, worker, order, payment, refund, delivery assignment, and audit logs. Has comments/history.

**Lifecycle**  
Follows SRD Support Ticket State Machine.

**Constraints**  
Support cannot directly mutate owned entities without using owning module workflows.

**Retention**  
Retained for support history and dispute resolution; closed tickets may archive.

**Audit Requirements**  
Business-impacting support actions are audited.

**Notes**  
Support history should be readable from customer and order views.

### 5.35 Support Ticket Comment

**Purpose**  
Append-only support conversation or internal note history.

**Ownership**  
Support Module.

**Responsibilities**  
Customer replies, support notes, store responses, internal notes, and resolution evidence.

**Core Fields**  
Ticket reference, author/actor, comment type, body, visibility, attachments metadata if needed, and timestamp.

**Relationships**  
Belongs to support ticket.

**Lifecycle**  
Append-only, with limited redaction/anonymization policy if required.

**Constraints**  
Internal notes must not be exposed to customers.

**Retention**  
Retained with ticket.

**Audit Requirements**  
Comment creation is history; redaction requires audit.

**Notes**  
Attachments storage is outside the database.

### 5.36 Settings

**Purpose**  
Represents operational configuration.

**Ownership**  
Administration / relevant domain module.

**Responsibilities**  
Service radius, delivery fees, minimum order amount, free-delivery threshold, timeout values, reason codes, opening hours, and operational toggles.

**Core Fields**  
Setting key, scope, value, effective period, active state, description, actor context, and timestamps.

**Relationships**  
May be global or store-scoped. Referenced by operational workflows and reports.

**Lifecycle**  
Created/updated by admin. Versioned or historized where changes affect financial/operational results.

**Constraints**  
Settings changes must not retroactively mutate historical orders.

**Retention**  
Retain history for settings that affect pricing, delivery, service eligibility, or operational timeouts.

**Audit Requirements**  
All setting changes are audited.

**Notes**  
Flexible setting values may use JSONB, but important settings should remain queryable where reports depend on them.

### 5.37 Audit Log

**Purpose**  
Cross-entity administrative and sensitive operational audit trail.

**Ownership**  
Audit / Compliance Module.

**Responsibilities**  
Actor tracking, affected entity, action type, before/after values, reason/context, store context, and timestamp.

**Core Fields**  
Actor reference, action, entity type/reference, store reference where applicable, before/after snapshots, reason, request context, and timestamp.

**Relationships**  
May reference many entity types. Some references may be logical/polymorphic because audit spans modules.

**Lifecycle**  
Append-only and immutable.

**Constraints**  
Audit logs cannot be edited for business meaning or deleted under normal workflows.

**Retention**  
Long-term.

**Audit Requirements**  
Audit log is the audit record.

**Notes**  
Audit logs are distinct from ledgers and history tables.

### 5.38 Report Rollup

**Purpose**  
Stores derived reporting summaries for Admin Dashboard.

**Ownership**  
Reporting & Analytics Module.

**Responsibilities**  
Daily/weekly/monthly sales, operations, inventory, finance, and worker productivity summaries.

**Core Fields**  
Rollup type, reporting period, store scope, metric values, generation timestamp, source range, and rebuild status.

**Relationships**  
Derived from orders, payments, refunds, inventory ledger, shifts, and delivery assignments.

**Lifecycle**  
Created/refreshed by rollup jobs. Rebuildable from source records.

**Constraints**  
Rollups are not authoritative over source data.

**Retention**  
Retained as useful for trend reporting; may archive older periods.

**Audit Requirements**  
Manual corrections are audited.

**Notes**  
ClickHouse remains deferred.

### 5.39 Analytics Snapshot

**Purpose**  
Captures point-in-time analytics aggregates or operational dashboard snapshots.

**Ownership**  
Reporting & Analytics Module.

**Responsibilities**  
Operational trends, SLA summaries, cancellation rates, inventory exceptions, and worker metrics.

**Core Fields**  
Snapshot type, period, store scope, metric payload, generation timestamp, and source range.

**Relationships**  
Derived from operational source records.

**Lifecycle**  
Generated periodically; rebuilt or archived by policy.

**Constraints**  
Must be traceable to source data range.

**Retention**  
Retained according to reporting usefulness.

**Audit Requirements**  
No routine audit; manual correction requires audit.

**Notes**  
May use JSONB for metric payloads if metric shape varies.

---

## 6. Relationships

### 6.1 Customer Relationships
- A user may have one customer profile.
- A customer may have many addresses.
- A customer may have active or historical carts.
- A customer may place many orders.
- A customer may have many support tickets, notifications, refunds through orders, and promotion usages.
- Customer Mobile App and Customer Web App share these same customer relationships through the backend; the database does not duplicate customer entities by platform.

Dependency philosophy: customer profile deactivation must not delete orders, payments, refunds, or support history.

### 6.2 Store Relationships
- A store owns one or more service zone configurations over time.
- A store owns inventory items for products stocked there.
- A store receives orders routed by delivery address.
- A store scopes workers, shifts, picking queues, rider pools, reconciliation, and reports.

Dependency philosophy: stores are deactivated rather than deleted once referenced.

### 6.3 Catalog and Inventory Relationships
- Categories organize products.
- Brands optionally classify products.
- Products are global.
- Inventory items connect products to stores and determine store-specific availability.
- Inventory reservations connect order items to inventory items.
- Inventory ledger entries explain stock movement.

Dependency philosophy: product deletion must not break historical orders; store sellability is controlled through inventory status.

### 6.4 Order Fulfillment Relationships
- Order has many order items.
- Order has many order events.
- Order may have inventory reservations.
- Order may have one active picking session at a time.
- Order may have delivery assignments over time due to rejection/expiry/reassignment.
- Order has payment and may have refunds/support tickets.

Dependency philosophy: order is the central transactional aggregate, but inventory, payment, refund, and delivery retain module ownership.

### 6.5 Workforce Relationships
- User can have worker profile.
- Worker has capabilities.
- Worker belongs to a home store.
- Worker has shifts and availability history.
- Picker capability links to picking sessions.
- Rider capability links to delivery assignments, rider locations, and cash/card reconciliation.

Dependency philosophy: worker deactivation preserves task history.

### 6.6 Payment and Refund Relationships
- Order has payment records.
- Payment has payment history.
- Payment may produce refunds.
- COD payments link to cash remittance.
- Card-on-Delivery payments link to terminal records and reconciliation.

Dependency philosophy: payment history is immutable; manual corrections are additive.

### 6.7 Support and Audit Relationships
- Support tickets can relate to customer, order, worker, payment, refund, delivery assignment, or store.
- Support comments belong to tickets.
- Audit logs reference actors and affected entities across modules.

Dependency philosophy: support and audit preserve what happened, not only the final state.

### 6.8 Cascade Philosophy
Hard cascade deletes should be avoided for business data. Cascades may be acceptable only for temporary draft data or implementation-private child records that have no independent business meaning. For historical and financial records, restrict deletion and use deactivation/archive/state transitions instead.

---

## 7. Constraints

### 7.1 Unique Constraints
Important uniqueness expectations:

- User login identity unique by login method.
- One active customer profile per customer user.
- One worker profile per worker user.
- One active inventory item per store/product.
- One active worker capability per worker/capability/store context.
- Promo codes unique within active promotion eligibility context.
- One active default address per customer where supported.
- Human-readable order number unique if introduced.

### 7.2 Composite Uniqueness
Composite uniqueness should protect scoped business identity:

- Store/product inventory.
- Customer/address default.
- Promotion/customer/order usage where usage limits require it.
- Active assignment/session ownership where only one active record is allowed.

### 7.3 Foreign Key Rules
Foreign keys should preserve:

- Orders to customers/stores.
- Order items to orders/products.
- Inventory items to stores/products.
- Reservations to order items/inventory items.
- Picking/delivery records to orders/workers/stores.
- Payments/refunds to orders.
- Support tickets to related business entities where direct reference is appropriate.

### 7.4 Check Constraints
Database-level checks should protect simple invariants:

- Quantities cannot be negative where business state does not allow it.
- Monetary amounts cannot use invalid signs except explicit adjustment/ledger contexts.
- State values must be within defined lifecycle values.
- Timestamps should be logically compatible where simple checks are safe.

### 7.5 Business Integrity Rules
Some business rules require application validation in addition to database constraints:

- Valid lifecycle transitions.
- Permission and role checks.
- Store radius routing decisions.
- Payment gateway verification.
- Redis lock ownership.
- Customer substitution response rules.
- Rider OTP/proof validation.
- Refund eligibility.
- Promotion eligibility and stacking behavior.

### 7.6 Store Isolation Rules
Store-scoped entities must carry store context directly or through an unambiguous parent. This applies to orders, inventory, workers, shifts, picking, delivery, reconciliation, and store reports.

### 7.7 Inventory Integrity
The database must prevent durable negative available stock and orphaned reservations. Application workflow must validate reservation, adjustment, release, and consumption rules against current order state.

### 7.8 Worker Assignment Integrity
The database should prevent multiple active ownership records where business requires exclusivity. The application must validate capability, shift, availability, mode conflict, store assignment, and timeout rules.

### 7.9 Payment Integrity
Payment records must remain attached to orders. Paid state must be backed by verified gateway evidence or operational collection evidence. Refunds must remain linked to original payment/order context.

---

## 8. Indexing Strategy

### 8.1 Indexing Philosophy
Indexes should support the system's real access patterns: customer lookup, store operations, worker queues, order timelines, inventory availability, reconciliation, support, and reporting. Avoid indexing every field by default; each index should support a known query, queue, uniqueness rule, or report.

### 8.2 Customer and Identity
- Lookup by phone/email identity for login.
- Lookup customer profile by user.
- Lookup addresses by customer and active/default status.

### 8.3 Store and Catalog
- Lookup stores by active/open status and location-related routing support.
- Lookup products by category, active status, searchable text, and SKU/barcode where available.
- Lookup inventory by store/product and store/status for customer catalog availability.

### 8.4 Order and Fulfillment
- Queue indexes by store, order status, SLA timestamps, and created time.
- Customer order history lookup by customer and created time.
- Order timeline lookup by order.
- Picking queue lookup by store and order state.
- Delivery pool lookup by store and assignment state.

### 8.5 Inventory
- Inventory availability lookup by store/product/status.
- Reservation lookup by order/order item and expiry/state.
- Ledger lookup by inventory item, store/product, reason, and time period.

### 8.6 Payment and Reconciliation
- Payment lookup by order and state.
- Gateway/provider reference lookup for idempotent webhook handling.
- Cash/card reconciliation lookup by rider, shift, store, state, and date.
- Refund lookup by order/payment/state.

### 8.7 Support and Audit
- Support tickets by customer, order, state, priority, assignee, and created time.
- Audit logs by actor, entity, store, action, and time.

### 8.8 Reporting
- Rollup lookup by period, store, metric type, and generation state.
- Large source tables should have time-based indexes that support scheduled rollups.

### 8.9 Search
Product search should support name/category/SKU use cases. Full-text or trigram-style strategy can be selected later, but the database design must preserve searchable fields and language-aware content.

---

## 9. Transaction Boundaries

### 9.1 Order Creation
Participating entities: cart, customer, address snapshot, store, order, order items, price snapshots, inventory reservations, payment initialization, promotion usage where applicable.

Atomicity: confirmed order creation must not leave an order without items, store, address snapshot, price snapshot, or required reservation/payment context.

Rollback expectation: if reservation or required payment initialization fails, no confirmed order should remain.

### 9.2 Inventory Reservation
Participating entities: inventory item, inventory reservation, order item, inventory ledger where required.

Atomicity: reservation state and reserved quantity must change together.

Rollback expectation: failed reservation must leave inventory availability unchanged.

### 9.3 Payment Confirmation
Participating entities: payment, payment history, order, order events, refund eligibility if needed.

Atomicity: verified payment result and order/payment state must remain consistent.

Rollback expectation: failed confirmation processing must be retryable without duplicate paid effects.

### 9.4 Packing Completion
Participating entities: picking session, picking session items, order/order items, inventory reservations, inventory items, inventory ledger, order events.

Atomicity: packing completion, reservation consumption, stock deduction, and order state movement must succeed together.

Rollback expectation: failed packing completion must not partially deduct stock.

### 9.5 Rider Assignment
Participating entities: delivery assignment, order, worker availability/shift context, order events.

Atomicity: assignment creation and order delivery-pool state must remain consistent.

Rollback expectation: failed assignment must return order to awaiting-rider state.

### 9.6 Delivery Completion
Participating entities: delivery assignment, order, payment/COD/card-on-delivery record, order events, payment history.

Atomicity: delivered state, proof/payment collection, and order/payment updates must remain consistent.

Rollback expectation: failed completion must not mark order delivered without proof/payment context.

### 9.7 Refund
Participating entities: refund, payment, payment history, order/order item, support ticket where applicable, audit log.

Atomicity: refund state and payment/order state must remain consistent.

Rollback expectation: failed processing should allow retry or explicit failure state without duplicate refund completion.

### 9.8 Inventory Adjustment
Participating entities: inventory item, inventory ledger, audit log, reconciliation record where applicable.

Atomicity: current inventory and ledger/audit record must change together.

Rollback expectation: failed adjustment must leave stock unchanged.

### 9.9 Shift Reconciliation
Participating entities: shift, cash remittance, card-on-delivery records, payment records, audit log.

Atomicity: reconciliation closure must not occur without preserving discrepancy context.

Rollback expectation: failed reconciliation closure leaves shift ended but unreconciled.

---

## 10. Concurrency Strategy

### 10.1 Redis Locking
Redis may coordinate short-lived locks for inventory reservation and other race-prone operations. Lock ownership and timeout behavior are implementation details for the System Design Document. PostgreSQL remains authoritative for final state.

### 10.2 PostgreSQL Transactions
Every workflow that updates multiple authoritative records must use database transactions. This is required for order creation, reservation, packing, payment confirmation, refund, adjustment, and reconciliation.

### 10.3 Optimistic Validation
Before committing a workflow, the application should revalidate current state: order state, inventory availability, payment state, worker eligibility, promotion eligibility, and store status.

### 10.4 Idempotency
The following workflows require idempotency:

- Order creation.
- Payment confirmation/webhooks.
- Inventory reservation/release.
- Notification send attempts.
- Refund processing.
- Rider assignment timeout/reassignment.

Rationale: external retries, mobile network instability, and provider callbacks must not duplicate business effects.

### 10.5 Retry Handling
Retries should be safe only where idempotency exists. Failed workflows should move to explicit failure/pending states rather than silently repeating irreversible actions.

### 10.6 Race Condition Prevention
Key race conditions:

- Two customers trying to reserve the same last item.
- Two pickers claiming the same order.
- Multiple riders receiving the same packed order.
- Duplicate payment confirmation callbacks.
- Refund retry after partial provider failure.

Database uniqueness, transactional state checks, and application-level locks together prevent these races.

### 10.7 Deadlock Avoidance
Implementation should use a consistent entity update order for multi-entity transactions. The DDD does not define that order, but later implementation must document it for inventory/order/payment workflows.

---

## 11. Soft Delete & Archival Strategy

### 11.1 Soft Delete Entities
Likely soft delete/deactivate:

- Customer profile.
- Address.
- Worker profile.
- Store.
- Category.
- Brand.
- Product.
- Product image.
- Inventory item.
- Promotions and promo codes through archive/deactivation.
- Settings through version/archive.

### 11.2 Immutable Entities
Immutable or append-only:

- Inventory ledger.
- Audit log.
- Order events.
- Payment history.
- Promotion usage.
- Support ticket comments.
- Finalized price snapshots.

### 11.3 State-Driven Entities
State-driven entities should not use deletion to represent workflow:

- Orders.
- Payments.
- Refunds.
- Picking sessions.
- Delivery assignments.
- Shifts.
- Support tickets.

### 11.4 Archival Philosophy
Archival means moving old records out of active operational views while preserving history and reporting meaning. Archive strategy should be defined after usage volume is known.

### 11.5 Purging Rules
Purging is limited to:

- Expired draft data.
- Abandoned carts after retention window.
- Short-term notification/provider metadata.
- Worker location history after operational retention.
- Data required to be removed/anonymized by privacy policy, while preserving business records as legally required.

### 11.6 Regulatory Considerations
Orders, payments, refunds, tax-related price facts, audit logs, and inventory ledgers should be retained long-term pending legal/accounting review.

---

## 12. Audit Strategy

### 12.1 Audit Ownership
Audit / Compliance Module owns the audit log. Each domain module is responsible for emitting audit-worthy context when it performs sensitive changes.

### 12.2 Audit Granularity
Audit logs should capture:

- Actor.
- Action.
- Affected entity type/reference.
- Store context where applicable.
- Before/after values for meaningful changes.
- Reason code or support/admin note where required.
- Timestamp.

### 12.3 Before/After Values
Before/after values are required for admin/support changes to inventory, worker permissions, store configuration, promotions, refunds, prices/settings, and order/payment corrections.

### 12.4 Actor Tracking
Actor may be customer, worker, admin, support user, ops user, finance user, or system job. System jobs should still identify the job source.

### 12.5 Store Context
Audit records affecting store-scoped data must include store context for filtering and accountability.

### 12.6 Support/Admin Actions
Support/admin actions that affect money, inventory, order state, worker assignment, permissions, or customer account state require audit.

### 12.7 Audit vs History vs Ledger
- Audit logs record who changed sensitive data and why.
- History tables record lifecycle/timeline movement.
- Ledger tables record quantity/financial movement and explain current balances.

These are related but not interchangeable.

---

## 13. Store Isolation Model

### 13.1 Global Entities
Global entities:

- Users.
- Customer profiles.
- Categories.
- Brands.
- Products.
- Product images.
- Some promotions.
- Global settings.

These are not owned by a single store, though they may have store-specific availability or eligibility.

Customer Platform entities are also client-agnostic. A customer profile, address, cart, order, notification history, support ticket, and promotion usage belongs to the customer/business workflow, not to mobile or web separately.

### 13.2 Store-Scoped Entities
Store-scoped entities:

- Service zones.
- Inventory items.
- Inventory reservations.
- Inventory ledger entries.
- Orders.
- Picking sessions.
- Delivery assignments.
- Worker home assignments.
- Shifts.
- Cash remittances.
- Card-on-delivery reconciliation.
- Store reports/rollups.

These must include direct or derived store context.

### 13.3 Shared Entities
Shared entities:

- Payments and refunds are order-scoped and therefore indirectly store-scoped.
- Support tickets may be customer-scoped, order-scoped, or store-scoped depending on issue.
- Notifications may be user-scoped and optionally order/store-linked.

### 13.4 Configuration Entities
Configuration may be global or store-specific:

- Delivery fee.
- Minimum order amount.
- Free delivery threshold.
- Service radius.
- Timeouts.
- Opening hours.
- Reason codes.

Configuration design must preserve the setting effective at order time where it affects financial or operational outcomes.

### 13.5 Rationale
Store isolation is required because Sakhari Ecom launches multi-store. Inventory, worker queues, rider pools, reconciliation, and SLA reporting cannot be globally blended without breaking operational correctness.

Platform isolation is not a database boundary for the customer experience. Web and mobile are presentation clients over the same backend and shared business records.

---

## 14. History & Ledger Model

### 14.1 Order History
Order events preserve lifecycle and support timeline. They are append-only and tied to order state machine transitions.

### 14.2 Payment History
Payment history preserves gateway confirmations, COD/card collection, failures, corrections, and refund-impacting payment transitions.

### 14.3 Inventory Ledger
Inventory ledger preserves every stock movement and reason. It explains current inventory and supports damaged/expired/returned/reconciled stock handling.

### 14.4 Shift History
Shift state, availability changes, task assignment, productivity, and reconciliation outcomes are retained for workforce reporting and finance.

### 14.5 Notification History
Notification records preserve send attempts, delivery/failure state, provider context, and retries for operational troubleshooting.

### 14.6 Support History
Support tickets and comments preserve issue lifecycle, customer/store responses, decisions, and linked corrective actions.

### 14.7 Immutability
History and ledger records should be immutable for business meaning. Corrections are additive.

---

## 15. Background Jobs Supporting the Database

### 15.1 Reservation Expiry
Find reservations that exceeded allowed checkout/substitution windows and move them to expired/released/cancelled states according to SRD lifecycle rules.

### 15.2 Report Generation
Create or refresh rollups for sales, operations, inventory, finance, and worker metrics. Source records remain authoritative.

### 15.3 Notification Retry
Retry failed notification sends where safe. Preserve retry history and final delivery/failure status.

### 15.4 Inventory Reconciliation
Support daily closing workflows, mismatch review, pending reconciliation flags, and ledger entries for approved adjustments.

### 15.5 Shift Reconciliation
Identify ended shifts requiring cash/card reconciliation and prevent closure until required finance checks are complete.

### 15.6 Cleanup
Expire abandoned carts, temporary draft records, old notification metadata, and worker location records according to retention policy.

### 15.7 Rollup Generation
Generate analytics snapshots for Admin Dashboard and reports. Rollups must be rebuildable.

---

## 16. Performance Strategy

### 16.1 Expected Growth
MVP is expected to support a few cities and low thousands of orders per day. The database should be normalized and operationally simple first, with clear paths for scale later.

### 16.2 Large Tables
Likely high-growth tables:

- Orders.
- Order events.
- Order items.
- Inventory ledger.
- Notifications.
- Rider locations.
- Audit logs.
- Payment history.
- Support comments.
- Analytics snapshots/rollups.

These need indexing and retention/archival attention.

### 16.3 Future Partitioning
Partitioning is not required at MVP unless load proves otherwise. Candidate future partition keys are time period and store context for large append-only tables.

### 16.4 Archive Strategy
Archive old operational records out of active views while preserving long-term finance/support/audit access.

### 16.5 Read/Write Expectations
High-write workflows: order creation, inventory ledger, rider location, notification history, audit logs. High-read workflows: catalog browsing, inventory availability, worker queues, order tracking, admin dashboards.

### 16.6 Rollup Philosophy
Use PostgreSQL rollups/views for MVP analytics. Rollups reduce dashboard query cost without introducing ClickHouse prematurely.

### 16.7 Vacuum Considerations
High-update tables such as inventory items, carts, notifications, and availability summaries require operational monitoring in RDS. Later implementation should consider update patterns and archiving to avoid table bloat.

### 16.8 Query Optimization Philosophy
Optimize around known workflows: checkout, store catalog, picker queue, rider queue, admin order search, reconciliation, and reporting. Avoid premature enterprise-scale denormalization.

---

## 17. Future Expansion

### 17.1 Multi-Country
Current SRD is Saudi-only. Future multi-country support would require country, currency, tax, payment provider, localization, and data residency expansion. Current design should avoid hard-coding Saudi assumptions into identifiers.

### 17.2 Warehouses
Dark stores can evolve into store/warehouse entities with different fulfillment types if needed. Inventory remains location-scoped.

### 17.3 Marketplace
Marketplace support would introduce sellers/suppliers, seller-owned catalog, settlement, and multi-party order accounting. Current product/category/brand and supplier-funded promotion concepts leave room for this but do not implement it.

### 17.4 Batch Delivery
Batch delivery would change delivery assignment cardinality from one order per rider delivery to route/batch groupings. Current design should avoid assuming delivery assignment can never relate to future route grouping.

### 17.5 Loyalty
Loyalty would add customer points/wallet ledgers. It should follow ledger principles, not mutable balances only.

### 17.6 BNPL
Tabby and Tamara are deferred fast-follow. Payment design should allow multiple provider types and provider payload snapshots without raw card data.

### 17.7 Supplier Integrations
Supplier integrations may affect inventory sync, cost/valuation, purchase orders, and supplier-funded promotions. Current inventory sync and brand/supplier-ready promotion concepts should not block this.

### 17.8 Advanced Promotions
Future promotion types can reuse promotion/rule/usage/snapshot concepts. Historical orders must preserve applied promotion details.

### 17.9 Dynamic Pricing
Dynamic pricing is deferred. If introduced, price snapshots remain mandatory so historical orders and refunds remain stable.

---

*Document prepared for database architecture planning. The SRD v2.6 remains the authoritative business requirements source.*
