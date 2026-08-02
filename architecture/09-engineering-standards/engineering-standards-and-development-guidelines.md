# Engineering Standards & Development Guidelines (ESDG) — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. New document — no prior skeleton existed for this scope. |
| **Stability** | Stable in principle (every normative rule here is a direct, concrete expression of an already-Accepted ADR or an already-authored architecture document); evolving in tool-specific detail (exact linter/test-framework/package names) as those below-architecture choices are made. |
| **Authority** | Subordinate to `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `decisions/` (the ADR set), and every document in `02-context/` through `06-quality-attributes/`. Sibling to, and the superset of, `07-coding-standards/coding-standards.md` and `08-ai-development/ai-development-rules.md` — this document does not repeat their content; where a rule already exists there in full, this document cites it and adds the surrounding implementation detail (API shape, DTO rules, git workflow, CI/CD, and the rest) that a Software Design Document actually needs and those two documents deliberately left unwritten. **This is the canonical implementation standard every future SDD must cite and comply with.** |

**Normative language.** This document uses RFC 2119 terminology throughout: **MUST**/**MUST NOT** denote an absolute requirement or prohibition; **SHOULD**/**SHOULD NOT** denote a strong default that may be deviated from only with a documented, reviewed reason; **MAY** denotes a genuinely open choice. Where a rule traces to an Accepted ADR, deviating from it is not a style disagreement — it requires a new, superseding ADR (`decisions/README.md`'s lifecycle rules), exactly as any other contradiction of an Accepted ADR would.

---

## 1. Purpose

**Why this document exists.** The Architecture Phase is complete: the SRD states what the product must do, the DDD models the data, the ADRs and `/architecture` documents decide the system's shape, and the Architecture Readiness Review has been remediated. What does not yet exist is the layer between "the architecture is decided" and "code is written": a single, normative reference an engineer (human or AI) opens before writing a controller, a migration, a DTO, or a test, and finds the concrete convention to follow — not the reasoning behind it again, but the rule itself, checkable in a code review without re-deriving it from first principles. That is this document's entire purpose. It invents no new architecture, no new module, no new business rule, and no new ADR-level decision; every normative statement here either restates a decision already made elsewhere in citable form, or fills a genuinely below-architecture gap (a DTO's exact validation shape, a git branch's exact name format) those documents correctly declined to specify.

**Relationship to the SRD.** The ESDG never introduces a requirement the SRD does not support. Where a standard here has a business reason (why refunds are line-item aware, why OTP is the only credential), that reason lives in the SRD and the ADR that operationalized it; this document states only the resulting implementation rule.

**Relationship to the DDD.** The DDD is authoritative for entity meaning, relationships, and lifecycle. This document's Database Standards (Section 11) and Domain Model Standards (Section 13) describe *how* an entity the DDD already defines is implemented in PostgreSQL and in code — never a second definition of what an entity *is*.

**Relationship to the ADRs.** Every ADR referenced below is cited, not re-argued. An engineer who finds this document and an ADR in apparent conflict follows the ADR and flags the ESDG as needing correction — this document is wrong until fixed, never the other way around (the same authority rule `08-ai-development/ai-development-rules.md` already states for itself).

**Relationship to the Architecture Documents.** `02-context/` through `06-quality-attributes/`, `07-coding-standards/`, and `08-ai-development/` remain the authoritative source for module boundaries, data ownership, cross-cutting philosophy, and quality targets. This document sits one layer below them: where they say *what must be true*, the ESDG says *the concrete shape that makes it true* — a DTO's validation rule, an endpoint's URI pattern, a migration's naming format, a commit message's structure.

**Relationship to SDDs.** Every future per-module Software Design Document (`03-decomposition/service-decomposition.md`'s module list) MUST cite this document as its implementation-standard baseline and MUST NOT restate a rule this document already makes — an SDD deviating from a MUST-level rule here without a documented, approved reason is a defect in the SDD, not a valid module-specific variation. An SDD's own content is the module's *internal* design (schemas, endpoints, events, tests) built *on top of* these standards, not a redefinition of them.

**Relationship to Implementation.** Code that satisfies its SDD but violates this document is not "done" (Section 22). This document, together with the Code Review and Architecture Compliance checklists it inherits (`07-coding-standards/coding-standards.md` Sections 10–11) and extends (Section 23 below), is what a pull request is checked against before merge.

---

## 2. Technology Stack Standards

Every technology named below already carries a dedicated ADR and, in most cases, a full entry in `04-cross-cutting/technology-decisions.md`; this section is the flat, implementation-facing list — engineers and AI assistants MUST treat it as closed, not as a menu.

**Backend**
- **NestJS** on **Node.js LTS**, **TypeScript**-first, as the single modular-monolith deployable (ADR-0002, ADR-0025). All backend code MUST be TypeScript, `strict` mode MUST be enabled at the compiler level, and `any` MUST NOT be used to bypass a type error it would otherwise catch — an untyped external payload (a webhook body, a provider response) is typed at its boundary with an explicit interface or schema, never left as `any` past that boundary.
- No second backend language or runtime MAY be introduced without a superseding ADR (ADR-0005's TypeScript-centered stack is a closed decision, not a per-service choice).

**Client applications** (referenced for completeness; this document's normative weight is heaviest on the Backend, since clients are thin presentation layers, Principle 4.6): React Native (Customer Mobile App, Worker App — ADR-0006), Next.js (Customer Web App — ADR-0007), React + Vite (Admin/Ops Dashboard — ADR-0008). All four clients share the TypeScript ecosystem; a client change MUST NOT introduce business logic that belongs in the Backend's service layer (Section 13).

**Database**
- **PostgreSQL** via Amazon RDS is the sole, exclusive system of record (ADR-0003). No module MAY introduce a second authoritative datastore without a superseding ADR.
- **Redis** (via ElastiCache) is acceleration/coordination only — cache, sessions, rate limiting, short-lived locks, idempotency keys (ADR-0004). Redis MUST NOT hold anything whose loss on a flush would lose a business fact (`data-architecture.md` Section 3's own test).

**Storage**
- **Amazon S3** for product imagery and file/media assets, referenced by key/URL from PostgreSQL rows, never embedded as a BLOB (ADR-0016).
- A **CDN** (CloudFront, per ADR-0030's topology) fronts public, cacheable content — product imagery and Customer Web App static assets (ADR-0028).

**API**
- **REST** is the only client-facing protocol (`module-communication.md` Section 4); modules never call each other over REST internally (Section 5 below).
- Every endpoint is **versioned** from its first release (ADR-0017) — see Section 5.

**Authentication**
- **JWT** access tokens (ADR-0026) plus rotating refresh tokens (ADR-0033) — no server-side session-store lookup on the common request path.
- **OTP** (one-time password) is the *only* credential mechanism for every actor; there are no passwords anywhere in this system (ADR-0027).

**Infrastructure**
- **Docker**-packaged builds, deployed as **AWS ECS Fargate** services behind an Application Load Balancer (ADR-0030).
- **GitHub** for source control; **GitHub Actions** for CI/CD (ADR-0030) — see Section 20.
- **AWS `me-central-2`** as the sole production region (ADR-0024).
- **AWS Secrets Manager** for production secrets (ADR-0030; Section 10 below).

**MUST NOT introduce, without a superseding ADR or a new ADR:** a second database technology, a message broker (the current in-process/outbox event model is deliberate at this scale — `integration-and-messaging.md` Section 11), a different cloud provider or region, a different authentication mechanism (password-based auth, a third-party identity platform), or any technology category not named in `04-cross-cutting/technology-decisions.md`.

---

## 3. Project Structure Standards

**Module layout.** Every one of the sixteen backend modules (Auth, User, Store, Catalog, Search, Inventory, Cart, Order, Payment, Promotion, Delivery, Notification, Support, Analytics, Audit, Settings — `module-catalog.md`) MUST have an identical internal shape, per `07-coding-standards/coding-standards.md` Sections 2–3:

```
<module>/
  controller/        thin HTTP entry points — request-shape validation only, no decisions
  service/            business logic — the only layer permitted to make decisions
  application-service/ orchestrates across the module's own domain services/repositories
                        for a single use case (Section 13); the layer a controller calls
  domain/              entities, value objects, domain services, domain events (Section 13)
  repository/          data-access code — reachable only from this module's own service layer
  infrastructure/      module-owned adapters: outbox publisher, external-integration client
                        (Section 15), module-scoped background job handlers
  dto/                 request/response DTOs (Section 6)
  validators/          custom validation rules beyond declarative DTO decorators (Section 7)
  events/              published/consumed event definitions and handlers (Section 14)
  constants/           module-scoped enums, permission names, fixed lookup values
  config/              module-scoped configuration surface (Section 3's Configuration below)
  test/                unit, integration, and (where applicable) contract tests (Section 16)
```

A module's `repository/`, `domain/`, and internal `service/` code MUST NOT be imported from outside the module's own directory — only its declared public interface (`controller/` plus any explicitly exported application-service methods named in `module-catalog.md`) is reachable from elsewhere (ADR-0009; `07-coding-standards/coding-standards.md` Section 2).

**Configuration.** Every module's runtime configuration (feature toggles it owns, its own Settings-sourced values per `module-catalog.md`'s Settings dependency) is read through a single, typed configuration surface per module — never `process.env` read ad hoc from business logic. A configuration value that is a business-tunable default (OTP lockout duration, batching eligibility window) is owned by the Settings module (ADR-0033) and fetched through Settings' interface, not hardcoded or duplicated as a local environment variable.

**Shared libraries.** Code shared across modules (a validation helper, a money-formatting utility, a ULID generator wrapper) MUST be genuinely cross-cutting utility — never a place business logic or another module's data model leaks into. A shared library:
- MUST NOT depend on any single module's domain types or repositories.
- MUST NOT become an escape hatch for cross-module data access (`08-ai-development/ai-development-rules.md` Section 4's forbidden patterns apply identically to "put it in shared" as to a direct cross-module query).
- SHOULD be small and boring — a shared library that grows business rules inside it is a sign those rules belong in a specific module instead.

**Repository topology.** Whether the Backend and the four client applications live in one repository or five remains an Open Decision (`07-coding-standards/coding-standards.md` Section 14) — this document does not resolve it. Regardless of the answer, the module-directory shape above is what the Backend's own source tree follows.

---

## 4. Naming Conventions

| Element | Convention | Example |
|---|---|---|
| Files | `kebab-case`, suffixed by role | `order.controller.ts`, `reserve-stock.service.ts`, `order-placed.event.ts` |
| Folders | `kebab-case`, matching Section 3's module layout | `inventory/repository/` |
| Classes | `PascalCase`, suffixed by role | `OrderController`, `InventoryReservationService`, `ReserveStockDto` |
| Interfaces | `PascalCase`, no `I` prefix (TypeScript idiom; the type system, not a naming tag, distinguishes an interface) | `OrderRepository`, not `IOrderRepository` |
| Enums | `PascalCase` type name, `PascalCase` or `UPPER_SNAKE_CASE` members, consistently within one enum | `enum OrderStatus { Pending, Reserved, Confirmed }` |
| DTOs | `PascalCase` + `Dto` suffix, verb-object for requests | `CreateOrderDto`, `OrderResponseDto` |
| Repositories | `PascalCase` + `Repository` suffix, one per aggregate root the module owns | `InventoryItemRepository` |
| Controllers | `PascalCase` + `Controller` suffix, one per resource (Section 5) | `OrdersController` |
| Services | `PascalCase` + `Service` suffix (domain/application distinction is by folder, Section 3, not by name collision) | `OrderService`, `CheckoutApplicationService` |
| Variables/functions | `camelCase`, full words — no abbreviation that obscures a term already defined in the DDD or `module-catalog.md` (`07-coding-standards/coding-standards.md` Section 4) | `inventoryReservation`, not `invRes` |
| Constants | `UPPER_SNAKE_CASE` for true compile-time constants; a business-tunable default is a Settings value (Section 3), never a hardcoded constant | `MAX_OTP_REQUESTS_PER_HOUR` only as a fallback default, not the source of truth |
| Database tables | `snake_case`, plural, matching the DDD's own convention (`data-architecture.md` Section 10; DDD Section 2) | `inventory_reservations`, `payment_ledger` |
| Database columns | `snake_case`; primary key `id`; foreign key `<entity>_id`; timestamps `created_at`/`updated_at`/domain-specific (`placed_at`, `reserved_at`) | `order_id`, `reserved_at` |
| Indexes | `idx_<table>_<column(s)>` | `idx_orders_store_id_status` |
| Constraints | `<type>_<table>_<column(s)>` (`pk_`, `fk_`, `uq_`, `chk_`) | `fk_order_items_order_id`, `chk_inventory_items_on_hand_non_negative` |
| Events | `<Entity><PastTenseVerb>` exactly, per `integration-and-messaging.md` Section 3 | `OrderPlaced`, `InventoryReserved`, `PaymentFailed` |
| Queues / outbox rows | Named after the owning module's outbox table, `<module>_outbox` (`data-architecture.md` Section 16); no shared, cross-module queue | `payment_outbox` |
| Environment variables | `UPPER_SNAKE_CASE`, prefixed by concern, never containing a secret's actual value in any committed file | `DATABASE_URL`, `JWT_SIGNING_KEY_ARN` (a reference, never the key itself) |
| Git branches | `<type>/<short-kebab-description>` — see Section 19 | `feat/inventory-reservation-locking`, `fix/otp-lockout-window` |

**No abbreviation rule.** If the DDD or `module-catalog.md` spells a term out in full ("Inventory Reservation," "Delivery Assignment"), code, logs, and events MUST use that same full term — never a private shorthand a future reader or AI session would have to learn to map back (`07-coding-standards/coding-standards.md` Section 4, restated here as a naming-table rule rather than prose alone).

---

## 5. API Standards

Builds on ADR-0017 (versioning) and `module-communication.md` Section 4 (REST as the sole client-facing protocol); this section is where those conventions become concrete and checkable.

**URI naming.**
- Resources are **plural nouns**: `/orders`, `/inventory-items`, not `/order`, `/getOrder`.
- Nesting reflects genuine ownership, not convenience: `/stores/{storeId}/inventory-items`, not a flat list requiring a separate filter for the same relationship the URI could express.
- No verbs in a URI for a standard CRUD-shaped operation — the HTTP method carries the verb. An operation that is not naturally CRUD-shaped (e.g., "cancel an order," "assign a rider") is expressed as a sub-resource action: `POST /orders/{orderId}/cancel`, `POST /delivery-assignments/{id}/reassign` — never a bare verb URI at the collection root.

**HTTP methods.**
| Method | Use |
|---|---|
| `GET` | Read, never mutates state, safe to retry/cache |
| `POST` | Create a resource, or invoke a non-CRUD action (above) |
| `PUT` | Full replace of a resource's representation |
| `PATCH` | Partial update |
| `DELETE` | Remove — for a soft-deleted or lifecycle-state entity (`data-architecture.md` Section 9), this MUST map to a state transition, never a hard delete of a record that must be retained |

**Status codes.** `200` (success with body), `201` (resource created, `Location` header set), `202` (accepted, processing continues asynchronously — used sparingly, and only where the client genuinely cannot get a synchronous result), `204` (success, no body), `400` (malformed request — validation failure, Section 7), `401` (unauthenticated), `403` (authenticated but not authorized — RBAC/store-scope denial, `security-and-compliance.md` Section 7), `404` (resource does not exist, or exists but the caller has no visibility into it — never used to leak existence to an unauthorized caller), `409` (a business-rule conflict — insufficient stock, an already-cancelled order), `422` (well-formed but semantically invalid), `429` (rate-limited), `5xx` (unexpected system failure, Section 8) — never used for an expected business-rule rejection.

**Pagination.** Every collection endpoint MUST paginate — cursor-based for high-volume, frequently-changing collections (orders, inventory ledger entries) and offset-based acceptable for small, stable collections (categories, brands). Response envelopes carry `data` and a `pagination` block (`nextCursor`/`hasMore`, or `page`/`pageSize`/`totalCount` for offset-based) — one consistent shape per pagination style, applied identically across every module, never a per-endpoint variant.

**Filtering and sorting.** Query parameters, never a request body, for `GET` filtering (`?status=Pending&storeId=...`) and sorting (`?sort=-createdAt`, leading `-` for descending). A module exposes only the filter/sort fields its own read model actually supports efficiently — an unindexed or expensive filter is not added to a public endpoint without a caching or read-model plan (Section 17).

**Versioning.** Every endpoint carries an explicit version in its path (`/v1/orders`), per ADR-0017. A breaking change (removed field, changed type, changed required-ness, changed status-code semantics) MUST ship as a new version, with the old version continuing to serve existing clients until every client using it has migrated — never a silent change to a live version. An additive, backward-compatible change (a new optional field) does not require a version bump.

**Headers.**
- `Authorization: Bearer <access token>` on every authenticated request.
- `X-Request-ID` — set by the client or generated by the Backend if absent — carried through every log line and downstream call for that request (`observability-and-operations.md` Section 3).
- `X-Correlation-ID` — set once at the origin of a business transaction (e.g., checkout) and propagated through every synchronous call and asynchronous event it triggers, distinct from a single request's `X-Request-ID` when a transaction spans multiple requests.
- `Idempotency-Key` — REQUIRED on every request that creates or mutates state in a way `data-architecture.md` Section 13 or `integration-and-messaging.md` Section 9 names as idempotency-required (order creation, payment confirmation, refund issuance) — the server MUST return the original result for a repeated key rather than re-executing the operation.

**OpenAPI.** Every endpoint MUST be documented in an OpenAPI (Swagger) specification generated from the same decorators/types that define the DTOs (Section 6) — never a hand-maintained document that can drift from the actual contract. A new or changed endpoint that is not reflected in the generated spec is not "done" (Section 22).

---

## 6. DTO Standards

- **Request DTOs** describe exactly the shape a client is allowed to send — never the same class as a domain entity (Section 13) or a database row shape. A request DTO is validated at the controller boundary (Section 7) before a service ever sees it.
- **Response DTOs** describe exactly the shape returned to a client — MUST NOT leak an internal field (an audit reason code meant for Admin, an internal state enum value with no client-facing meaning) to a caller not authorized to see it. Different roles reading the "same" resource MAY receive different response DTO shapes where RBAC (`security-and-compliance.md` Section 7) scopes what they can see.
- **Validation** is declarative wherever possible (class-validator-style decorators or an equivalent schema), with custom validators (Section 3's `validators/` folder) reserved for rules a declarative decorator cannot express. A DTO field's validation lives with the DTO, not scattered across the controller.
- **Serialization/transformation.** A response DTO is produced by an explicit mapping from the domain/entity shape — never the entity itself serialized directly — so a future entity field addition does not silently become API surface without a deliberate decision to expose it.
- **Version compatibility.** A DTO tied to a specific API version (Section 5) is named or namespaced so it's unambiguous which version it belongs to when two versions' shapes diverge (`CreateOrderDtoV1`, or a `v1/` DTO folder per version) — never one mutable DTO class silently reused and reshaped across versions.
- Money fields on a DTO are integer halala (ADR-0015, Section 11) — never a float or a pre-formatted currency string; formatting to SAR display happens client-side or at a dedicated presentation boundary, never inside a DTO meant for API transport.
- Timestamp fields are ISO-8601 UTC (ADR-0014, Section 11) on the wire; conversion to Arabia Standard Time happens only in the client.
- Identifier fields are ULIDs (ADR-0019, Section 11), typed as strings on the wire.

---

## 7. Validation Standards

Builds directly on `07-coding-standards/coding-standards.md` Section 6 and `security-and-compliance.md` Section 8; this section makes the boundary concrete.

- **Input validation** (transport-shape correctness — required fields present, correct type, string length/format) happens once, at the controller, against the request DTO (Section 6), before the service layer is invoked. A malformed request MUST be rejected with `400` and never reach business logic.
- **Business validation** (is this order cancellable, is this stock available, is this promotion eligible) happens in the service layer, as part of the operation's own logic — never duplicated as a second pass in the controller, and never skipped because "the DTO already validated the shape" (shape correctness and business correctness are different questions, Section 6's own distinction).
- **Database validation** (a `NOT NULL`, a `CHECK`, a `UNIQUE` constraint, a foreign key) is the last line of defense, not the primary mechanism — application-level validation MUST catch a violation before it reaches the database in the ordinary case, but the schema constraint still exists so a bug in application-level validation cannot silently corrupt data (Section 11's constraint-naming convention applies to every such constraint).
- **Cross-module validation** — a business rule that depends on another module's state (does this promotion apply to this store, is this worker eligible for this shift) is validated by calling that module's public interface (`module-communication.md` Section 5), never by reaching into its tables directly, and never by trusting a value passed from another module without revalidating it against current state where `data-architecture.md` Section 13's optimistic-revalidation rule applies.
- **Error messages** returned to a client MUST be specific enough to be actionable (which field, what's wrong) without leaking internal implementation detail (a raw database error, a stack trace, an internal entity name not part of the public contract) — see Section 8's error format.
- **Sanitization.** Any input rendered back to a client or stored for later display (a Support ticket comment, a product review, a delivery note) is treated as untrusted and sanitized against injection (SQL injection is prevented structurally by using a parameterized query layer/ORM, never string-concatenated SQL; XSS-relevant output is escaped at the point of rendering, a client-side responsibility the Backend supports by never assuming a stored string is safe to render unescaped).

---

## 8. Error Handling Standards

**Standard error format.** Every error response returned to a client MUST use one consistent envelope across all sixteen modules:

```
{
  "error": {
    "code": "<STABLE_MACHINE_READABLE_CODE>",
    "message": "<human-readable, safe-to-display message>",
    "requestId": "<X-Request-ID>",
    "details": [ { "field": "...", "issue": "..." } ]   // present for validation errors
  }
}
```

`code` is a stable string a client can branch on (`INSUFFICIENT_STOCK`, `ORDER_NOT_CANCELLABLE`, `INVALID_OTP`) — never a code that changes when the human-readable `message` is reworded. `requestId` MUST always be present and MUST match the `X-Request-ID` used in server-side logs for that request (Section 9), so a reported error is traceable back to its exact log lines.

**Categories**, each mapped to an HTTP status range (Section 5) and a handling rule:

| Category | Example | Status | Rule |
|---|---|---|---|
| Validation errors | missing required field, wrong type | 400 | Rejected at the controller boundary (Section 7); never reaches the service layer |
| Business errors | insufficient stock, ineligible promotion | 409/422 | Returned as a specific, named outcome the caller can act on — never thrown as an unexpected system failure (`07-coding-standards/coding-standards.md` Section 7) |
| Authentication errors | missing/expired/invalid token | 401 | Never distinguishes "wrong password" from "unknown user" in the message (there are no passwords, ADR-0027, but the same non-enumeration discipline applies to phone-number/OTP flows) |
| Authorization errors | valid session, insufficient permission or wrong store scope | 403 | Never reveals what the caller *would* be allowed to see if they had the permission — a denial looks the same whether the resource exists or not, where that distinction itself would leak information |
| Infrastructure errors | database connection lost, Redis unavailable | 500/503 | Surfaced, logged with full context, never silently swallowed (Constitution Section 10); a workflow moves to an explicit failure/pending state, never retried blindly |
| External service errors | Moyasar timeout, Unifonic failure | 502/503/504, per ADR-0032's per-provider table (`integration-and-messaging.md` Section 12) | Handled per that provider's documented timeout/retry/circuit-breaker rule — never a generic catch-all retry |
| Retryable errors | a transient infrastructure or external-service failure | varies | MUST be distinguished from a non-retryable one (a business rejection, a validation failure) in both the error code and any internal retry logic — retrying a business rejection is a bug, not resilience |

**Logging requirement.** Every error at 500-level, and every business error the owning module considers significant enough to track (a recurring `INSUFFICIENT_STOCK` on one SKU, say), is logged with the correlation ID (Section 9) — an error a client sees and an error a developer can find in logs MUST be joinable by `requestId`/correlation ID, always.

---

## 9. Logging Standards

Builds on `infrastructure-and-release.md` Section 7 and `observability-and-operations.md` Section 3 (the system-wide baseline) and `07-coding-standards/coding-standards.md` Section 5 (the module-level discipline); this section is the concrete field contract.

**Log levels.** `error` (an unexpected failure requiring attention), `warn` (a degraded-but-recovered condition, a retried operation, an approaching rate limit), `info` (a normal, significant business or operational event — order placed, payment confirmed, a background job completed), `debug` (verbose, development/troubleshooting detail — MUST NOT be enabled by default in Production, and MUST NOT itself carry sensitive data even when enabled, per the masking rule below).

**Structured logging — required minimum field set on every log entry:**

| Field | Requirement |
|---|---|
| `timestamp` | UTC, ISO-8601 |
| `level` | one of the four levels above |
| `module` | the owning module name, matching `module-catalog.md` exactly (Section 4's no-abbreviation rule) |
| `operation` | the specific operation/method that produced the entry |
| `outcome` | success/failure/partial, where applicable |
| `requestId` | present on every request-scoped log line (Section 5, 8) |
| `correlationId` | present wherever the log line is part of a multi-step business transaction |
| `actorId` | the authenticated caller's identity, where applicable — never the raw phone number (masking rule below) |

**Sensitive information masking.** A phone number, an address, a payment instrument detail, an OTP code, or any other PII/secret MUST NOT appear in a log in raw form, at any log level, including `debug` — masked (`+9665****1234`) or omitted entirely, consistent with `security-and-compliance.md` Section 11's PDPL obligations extending to logs, not just PostgreSQL (`infrastructure-and-release.md` Section 7). A secret (Section 10) MUST NOT appear in a log under any circumstance.

**Correlation IDs.** Generated (or accepted from the client) at the edge of the system, propagated through every synchronous call and attached to every outbox/event record a request produces, so a single business transaction's full path — across modules, across the sync/async boundary — is reconstructable from logs alone (`observability-and-operations.md` Section 3).

**Audit logging is not application logging.** A log entry, however complete, is never a substitute for an Audit Log entry where `security-and-compliance.md` Section 9 / DDD Section 12.6 requires one — the two are written by different code paths, even when triggered by the same event (`07-coding-standards/coding-standards.md` Section 5).

**Performance logging.** Every request logs its own duration; a request exceeding its module's expected latency profile (Section 17) is logged at `warn` even if it ultimately succeeded, so a creeping regression is visible before it breaches an alert threshold (`nfr-matrix.md` Section 4).

---

## 10. Security Standards

This section operationalizes `04-cross-cutting/security-and-compliance.md` in full; it does not restate that document's rationale, only the concrete rule a change must satisfy.

**Authentication.** OTP is the only credential mechanism (ADR-0027) — a change MUST NOT introduce password-based login, a client-side-stored long-lived credential, or a second parallel authentication path. Every request is authenticated fresh by the Backend; a client's own belief about its identity or role is never trusted (`security-and-compliance.md` Section 2).

**Authorization / RBAC / Store-Scoped RBAC.** Every module operation maps to a named permission from the authoritative catalogue (`security-and-compliance.md` Section 7, `<resource>:<operation>`), checked centrally before a controller delegates to its service layer — never re-implemented per module. Every store-scoped operation additionally requires the resource's store to be present in the caller's Assigned Stores (ADR-0041) — Permission AND Store Scope, evaluated in the order Section 7 of that document specifies. A new operation gets a new named permission; an existing permission's meaning is never silently broadened.

**Password policy.** Not applicable — there are no passwords anywhere in this system (ADR-0027); a change MUST NOT introduce one.

**OTP.** 6-digit codes, 5-minute expiry, 5 requests/phone/hour, 5 verification attempts/code, 60-second resend cooldown, 30-minute lockout on ceiling breach, independent device/IP throttling, progressive backoff (ADR-0039) — all configurable via Settings, none hardcoded as a business rule (only as a documented fallback default, Section 4).

**JWT / Refresh Tokens.** Access tokens: 15-minute TTL. Refresh tokens: 7-day idle / 30-day absolute TTL, rotated on every use, with reuse detection invalidating the full token family (ADR-0033). PostgreSQL is authoritative for session/revocation state; Redis is an accelerator only (`security-and-compliance.md` Section 6).

**Step-up authentication.** Refund approval, permission changes, settings changes, financial ledger adjustments, and sensitive audit access by Super Admin, Operations Manager, Finance, or Support Lead REQUIRE a fresh step-up challenge (TOTP or an approved enterprise MFA mechanism) in addition to the standing OTP session and the RBAC/store-scope check (ADR-0040) — a change touching any of these five operations that does not call Auth's step-up interfaces is incomplete, not merely risky.

**Secrets management.** Every secret (JWT signing keys, Payment/Notification/Maps provider credentials) is held in AWS Secrets Manager (ADR-0030), never in code, never in a committed file, never logged (Section 9), and scoped to only the single module that owns the integration it authenticates (`security-and-compliance.md` Section 10). A pull request introducing a literal credential, key, or token — even in a test fixture — MUST be rejected, not merged with a follow-up removal.

**Encryption.** Data in transit is TLS-terminated at the load balancer (`infrastructure-and-release.md` Section 4's networking topology); data at rest relies on the managed services' own encryption (RDS, S3, ElastiCache) rather than a custom application-level scheme, consistent with the "boring technology" conviction (Constitution Section 3) — a module MUST NOT invent its own encryption for data already covered by the managed store's at-rest encryption.

**PII handling / Saudi PDPL.** Every obligation in `security-and-compliance.md` Section 11's compliance-mapping table applies without exception: Saudi data residency (no durable data or backup leaves `me-central-2`), data minimization/retention per entity category, accountability for administrative access to sensitive data (via Audit), and payment-handling integrity (state changes only on verified provider confirmation, never client-supplied status). A field newly identified as PII MUST be added to the masking rule (Section 9) and the retention/purge category it belongs to (Section 11) as part of the same change that introduces it — not a follow-up.

---

## 11. Database Standards

Operationalizes `04-cross-cutting/data-architecture.md`; naming already appears in Section 4's table above and is not repeated here.

**Migration strategy.** Every schema change ships as a versioned, forward-only migration, reviewed and run through the same CI/CD pipeline as code (Section 20) — never a manual, undocumented schema change against any environment above Development. A migration that changes a column's meaning or a constraint in a backward-incompatible way is coordinated with the API version (Section 5) that depends on it, so a rollback of the deployable does not leave the schema in a state the previous version cannot serve.

**Primary keys.** Every table's primary key is a ULID, stored as `id` (ADR-0019; `data-architecture.md` Section 10) — no auto-increment integer, no UUID, with no exception anywhere in the schema.

**Foreign keys.** Every foreign key column is named `<referenced_entity>_id` and carries a real database-level `FOREIGN KEY` constraint (Section 7's "database validation as the last line of defense") — a foreign key relationship is never enforced only at the application layer where the database can enforce it directly, unless a specific, documented performance reason (and an equivalent application-level guarantee) justifies the exception.

**Indexes.** Every foreign key column, every column used in a `WHERE`/`ORDER BY` on a query path this document's Performance Standards (Section 17) or an SDD names as latency-sensitive, and every column enforcing a uniqueness business rule (a store's SKU, a phone number) is indexed — named per Section 4's `idx_<table>_<column(s)>` convention.

**Transactions.** Every workflow updating more than one authoritative record within a single module's boundary runs inside a database transaction (`data-architecture.md` Section 5) — never spanning two modules' tables; a multi-module operation is orchestrated with compensation instead (`module-communication.md` Section 8).

**Optimistic locking.** The default concurrency strategy everywhere a workflow's correctness depends on state that could change between read and commit — order state transitions, promotion eligibility, worker eligibility — implemented as an explicit state-check inside the transaction, never a held lock across a request's full round-trip (`data-architecture.md` Section 13).

**Pessimistic locking.** Reserved for the one workflow where the business cost of losing a race outweighs a short lock wait — inventory reservation's Redis-coordinated, then PostgreSQL-committed claim on an Inventory Item (`data-architecture.md` Sections 7, 13). A module introducing a new pessimistic lock elsewhere MUST justify it against this same cost/benefit test, not add one by default.

**Lock ordering.** Any module whose own transaction touches more than one of its own tables acquires row locks in the fixed order `data-architecture.md` Section 13 already specifies per module (Inventory: Item → Reservation → Ledger; Order: Order → Order Items ascending by PK; Payment: Payment → History → Refund → Ledger; Delivery: Assignment → Picking Session) — a new multi-table workflow within a module MUST extend that module's existing fixed order, never introduce an ad hoc one.

**Soft delete vs. hard delete vs. lifecycle state.** Applied exactly per `data-architecture.md` Section 9's three categories: master/profile data is soft-deleted; append-only/immutable records (ledgers, audit logs, order events) are never deleted at all; state-driven entities (Order, Payment, Refund, Shift) use lifecycle state, never deletion, to represent status. A migration or a service method that hard-deletes a row outside the narrow "genuinely disposable" purge category (Section 9's own list) is a defect.

**Audit fields.** Every table carries `created_at` and `updated_at` at minimum (UTC, below); a table whose entity has a meaningful domain timestamp (`placed_at`, `reserved_at`, `completed_at`) carries that explicitly rather than inferring it from `created_at`/`updated_at`.

**Timestamps.** Every timestamp column is UTC, with zero exceptions; conversion to Arabia Standard Time happens only at client presentation (ADR-0014).

**Money handling.** Every monetary column is an integer halala column — `bigint` or an equivalent exact integer type, never `float`/`double`/an ambiguous `decimal` scale that could silently round — across every module that stores money (Order, Payment, Promotion, Refund, Cash Remittance, Payment Ledger). No monetary arithmetic anywhere (application code, a database query, a reconciliation job) is performed in a non-integer type (ADR-0015).

**ULIDs/UUIDs.** ULID, only, everywhere (ADR-0019) — see Primary keys above; this is restated here because it is the single most common hallucination this project's own history has produced (`08-ai-development/ai-development-rules.md` Section 12) and cannot be over-stated.

---

## 12. Repository Standards

- **Responsibilities.** A repository is data-access only — translating between a module's domain model (Section 13) and its own PostgreSQL tables. It answers "how do I persist/retrieve this entity," never "should I."
- **Forbidden business logic.** A repository MUST NOT contain a validation rule, an eligibility check, a calculation, or any conditional that represents a business decision — those belong to the service layer (Section 13) that calls the repository. A repository method named `findEligibleForRefund` (a business judgment baked into a query name) is a violation; `findByOrderIdAndStatus` (a mechanical query) is not.
- **Query organization.** Queries live with the repository for the aggregate root they concern, not scattered across services as raw ORM calls — a service asks its module's repository for what it needs; it does not construct its own query against another repository's tables, and never against another module's tables at all (ADR-0009).
- **Transactions.** A repository method that must run inside a transaction accepts (or is invoked within) the transaction context its calling service established (Section 11) — a repository does not silently open its own, separate transaction for an operation that is supposed to be atomic with sibling writes in the same use case.
- **Read/write separation.** Where a module's read load and write load genuinely diverge (a reporting query against Order, for example), a read-oriented repository method or a dedicated read model MAY be introduced — but it still reads from the same module-owned tables (or an owned, explicitly rebuildable projection, per `data-architecture.md` Section 4) never a table owned by another module, and never a second authoritative copy of data this module already owns.

---

## 13. Domain Model Standards

- **Entities** have identity (a ULID) that persists across state changes; two entities with the same field values but different IDs are different entities. An entity's fields and lifecycle match the DDD's own definition exactly — this document does not redefine what an entity *is*, only how it's expressed in code.
- **Aggregates** are the transactional consistency boundary within a module (Section 11's transaction rule) — an aggregate root is the only entity within its aggregate that outside code (including other services within the same module) addresses directly; a child entity is reached only through its aggregate root.
- **Value Objects** (Money as integer halala + currency, a phone number, an address) are immutable and compared by value, not identity — a Money value object MUST enforce the integer-halala invariant (Section 11) at construction, so an invalid monetary value cannot exist even transiently inside domain logic.
- **Domain Services** hold business logic that doesn't naturally belong to a single entity (a reservation-locking coordination that spans an Inventory Item and a new Reservation) — they are still module-internal, never a cross-module concept.
- **Factories.** An aggregate with non-trivial construction invariants (a new Order requiring a valid, non-empty item list and a resolvable store) is constructed through a factory method or a named static constructor that enforces those invariants at creation — never left to a caller assembling raw fields and hoping they're consistent.
- **Domain Events** (Section 14) are raised by the aggregate root as a side effect of a state change already committed, never as a way to communicate an intention that hasn't happened yet (Section 14's "fact, not instruction" rule, restated for where in the code an event actually originates).
- **Invariants.** A rule that must always hold for an entity or aggregate (an Inventory Item's on-hand quantity is never negative, an Order's total always equals the sum of its line items at Price Snapshot time) is enforced inside the domain model itself — at the point of mutation — not only checked afterward by a database constraint (Section 11) or a controller-level validation (Section 7). The database constraint is the backstop, not the primary mechanism, for a domain invariant.

---

## 14. Event Standards

Operationalizes `04-cross-cutting/integration-and-messaging.md` in full; every rule below is that document's, made checklist-concrete.

- **Publishing.** A publisher writes its owned entity's state change and the corresponding outbox row in the same module-local transaction (ADR-0029) — never as a separate, independently-failable step after commit. A publisher never blocks on, or expects a return value from, any consumer.
- **Consuming.** A consumer subscribes to specific named events; "subscribe to everything" is reserved for the two modules whose job requires it (Notification, Analytics). A consumer's failure to process an event is never silently swallowed — it succeeds, is retried, or surfaces as a recorded failure.
- **Naming.** `<Entity><PastTenseVerb>` exactly (Section 4's table); a failure gets its own named event (`InventoryReservationFailed`, `PaymentFailed`), never a flag on the success event.
- **Idempotency.** Every consumer MUST produce the same end state whether it receives a given event once or many times — at-least-once delivery is the assumed baseline for every event in the system, not an edge case. A consumer determines whether it has already handled a specific event occurrence before applying its effect.
- **Outbox.** Every publishing module owns its own outbox table, in its own transaction scope — never a shared, cross-module outbox (`data-architecture.md` Section 16). An outbox row is never updated to change the fact it records, only its delivery-status fields.
- **Retries.** Dispatch failures are retried by the dispatcher with exponential backoff against the durable outbox record; consumption failures are retried up to a bounded attempt count appropriate to that consumer's own workflow, and an exhausted retry surfaces as a recorded failure, never a silent drop.
- **Versioning.** An additive, backward-compatible event-schema change (a new optional field) needs no new version or name. A breaking change is introduced as a new, distinctly-named event (or an explicit version suffix), with the old event kept publishing until every consumer has migrated — mirroring ADR-0017's API deprecation discipline.
- **Ordering.** A consumer never assumes strict cross-event ordering across different modules' events; where ordering matters within one entity's own lifecycle, the consumer's own logic tolerates or correctly sequences events referencing the same entity — the transport does not guarantee it.

---

## 15. External Integration Standards

Operationalizes `integration-and-messaging.md` Section 12 (ADR-0032); every outbound call to an external system follows the provider-specific rule below, enforced through a mandatory provider wrapper per integration — a module never calls a provider SDK directly from scattered call sites.

| Integration | Owner | Timeout | Retry | Circuit breaker / fallback |
|---|---|---:|---|---|
| Moyasar payment API | Payment | 5 s | Idempotent create/status ops only, max 3 attempts, exponential backoff | Opens after 5 consecutive technical failures or 50% failure rate/2 min; online payment pauses, COD/card-on-delivery remain available |
| Moyasar webhooks | Payment | inbound | Signature + idempotency-key verification, processed at least once | Duplicate webhooks ignored after the first successful state transition |
| Unifonic OTP/SMS | Notification/Auth | 5 s | Max 2 send attempts per OTP request; no new OTP generated mid-retry | Fails the OTP flow loudly with a retry-later state, never a silent hang |
| Push notification service | Notification | 5 s | Max 3 attempts with backoff, via the outbox/job retry path | Business state never depends on push success; Worker App polls its assignment endpoint as fallback |
| Google Maps/geocoding | Store/User | 3 s autocomplete, 5 s server-side validation | Max 2 attempts | New-address validation fails gracefully; already-validated addresses continue to work |
| POS/card-on-delivery settlement import | Payment | 10 s | Scheduled retry with backoff | Manual reconciliation is the standing fallback; discrepancies stay open until reviewed, never silently written off |

**Every outbound call MUST:** carry a correlation ID (Section 9), be structured-logged, use an idempotency key wherever the provider supports one, and store only a provider response snapshot — never raw card data or secrets (Section 10). **Every inbound response or webhook MUST** be validated (signature, expected shape) before being treated as fact (`security-and-compliance.md` Section 8) — no exceptions for a provider considered "trusted."

No new external integration MAY be introduced without a corresponding row in this table (added via ADR, mirroring ADR-0032's own origin) and a named owning module — an integration with no owner or no documented resilience posture is not production-ready regardless of how well it works in the happy path.

---

## 16. Testing Standards

Extends `07-coding-standards/coding-standards.md` Section 8 and `08-ai-development/ai-development-rules.md` Section 9 into concrete test-type guidance; no specific test framework is named (Open Decision, Section 24's own honesty about what remains unpicked) — the rules below apply regardless of the framework eventually selected.

- **Unit tests** target a module's domain logic and service-layer decisions in isolation (repositories and external integrations mocked/faked) — proving a business rule holds (an order cannot be placed against insufficient stock), not that a specific internal method was called in a specific order (`coding-standards.md` Section 8's own framing).
- **Integration tests** exercise a module's service layer against a real (or realistic, e.g. a test-container) PostgreSQL instance, proving the persistence and transaction behavior Section 11 requires actually holds — not just that the code compiles against a mocked repository.
- **Contract tests** verify a module's public interface (REST endpoints and published event shapes) against its documented contract (the OpenAPI spec, Section 5; the event catalog, Section 14) — protecting a consumer module or client from an undocumented, silently breaking change.
- **API tests** exercise a full request/response cycle through the Backend's HTTP layer, including authentication and RBAC (Section 10) — proving the boundary behaves correctly, not only the service logic behind it.
- **Repository tests** verify query correctness and constraint behavior (Section 11) against a real database — a repository test that only exercises a mocked ORM has not verified the thing most likely to be wrong (an incorrect query, a missing index, a constraint violation).
- **Mocking.** External integrations (Section 15) are always mocked or faked in unit and most integration tests — a test suite MUST NOT depend on live network access to Moyasar, Unifonic, or Google Maps to pass. A dedicated, clearly-labeled contract or smoke-test suite MAY exercise a sandbox/staging provider endpoint, run separately from the ordinary CI gate (Section 20).
- **Coverage expectations.** Proportional to correctness criticality, not a uniform percentage (`coding-standards.md` Section 8, Section 13 — no specific threshold is asserted here either, consistent with that document's own honesty about what's not yet decided). Checkout, inventory reservation, payment confirmation, and every named race condition/idempotency-required workflow (`data-architecture.md` Section 13) REQUIRE explicit tests proving the concurrency and idempotency guarantee holds, not just the happy path — this is a floor, not a target to negotiate down.
- **Test naming.** Descriptive of behavior, not implementation — `should reject reservation when available quantity is zero`, not `test4` or `testReserveStock_case2`.
- **Fixtures.** Test data is built through the same factory/constructor discipline as production code (Section 13) — a fixture that bypasses an aggregate's own invariant-enforcing constructor to hand-assemble an invalid entity is testing a state the system could never actually reach, and hides bugs rather than catching them.

---

## 17. Performance Standards

Carries `nfr-matrix.md` Section 4's numbers forward as implementation-facing targets (ADR-0031) — this document does not re-derive them, only states what a change must not regress.

| Path | p95 | p99 |
|---|---|---|
| General authenticated API | ≤ 300 ms | ≤ 800 ms (excluding external provider wait time) |
| Checkout orchestration | ≤ 1.5 s | ≤ 3 s (excluding customer-side payment authorization time) |
| Catalog/search customer reads | ≤ 300 ms | ≤ 700 ms |
| Rider location ingest | ≤ 250 ms accepted by backend | — |

**Database query limits.** No endpoint on a latency-sensitive path (the table above) issues an unbounded query (no `LIMIT`, no pagination) or a query lacking an index on its filter/sort columns (Section 11). A query plan MUST be checked for any new query added to a path with a target above, not assumed correct.

**Caching.** Redis caching (Section 2) is applied only where `data-architecture.md` Section 3's "would losing this on a flush lose a business fact" test is satisfied as "no" — a cache is an accelerator with an explicit invalidation strategy, never introduced as an implicit second source of truth.

**Pagination.** Every collection endpoint paginates (Section 5) — this is also a performance rule, not only an API-shape one: an unpaginated collection endpoint is a latency and memory regression waiting to happen as data volume grows.

**N+1 prevention.** A list endpoint that triggers one query per returned item (an N+1 query pattern) is a defect, not an acceptable inefficiency — batched/joined loading is used instead, checked explicitly in code review for any new list endpoint.

**Connection pooling.** The Backend's PostgreSQL connection pool is sized and monitored as part of the infrastructure topology (`infrastructure-and-release.md` Section 3) — application code MUST NOT open ad hoc, unpooled database connections outside the framework's managed pool.

**Memory considerations.** A background job or a report-generation path that loads an unbounded result set into memory (Section 6's Report Generation, Reconciliation jobs) MUST stream or batch-process instead — a job that works today and falls over at 10x the current data volume is a scalability defect, not a future problem.

---

## 18. Code Quality Standards

- **Formatting and linting.** Enforced by tooling (a formatter and linter, package TBD — Section 24), run in CI (Section 20) as a required check — not a style guide re-litigated in code review. This document does not assert the specific tool.
- **Comments.** Default to none; a comment is added only where the *why* is non-obvious — a hidden constraint, a workaround for a specific external-provider quirk, an invariant that isn't self-evident from the code — never to restate *what* well-named code already shows.
- **Documentation.** A module's public interface is documented at the same level of detail `module-catalog.md` already uses (`coding-standards.md` Section 9); an OpenAPI spec (Section 5) documents every endpoint; a JSDoc/TSDoc block is reserved for a public interface method whose contract isn't obvious from its name and types alone.
- **TODO usage.** A `TODO` comment MUST reference either an issue/ticket or an Open Decision in the relevant architecture document (Section 1's cross-reference model) — a bare `TODO` with no tracked follow-up is equivalent to the silent-shortcut anti-pattern Constitution Principle 4.12 already forbids.
- **Deprecated code.** A deprecated function/endpoint/event is marked explicitly (a `@deprecated` annotation, an API version note per Section 5, an event-versioning note per Section 14) with its removal condition stated — never silently left in place indefinitely with no signal it shouldn't be used for new code.
- **Code duplication.** Three or more near-identical blocks of business logic across a module is a signal to extract a shared function *within that module* — but duplication across module boundaries is not automatically a problem to "fix" by sharing code (Section 3's shared-library caution): two modules independently validating "is this actor authorized" in the same shape is a violation (Section 10, must be centralized); two modules independently formatting a similar-looking response DTO is very often fine.
- **Cyclomatic complexity.** A function whose branching complexity makes its business rule hard to verify by reading it is a signal to decompose it into named, testable steps — particularly for correctness-critical logic (checkout, reservation, refund calculation) where a reviewer's ability to actually verify correctness by reading the code matters more than brevity.
- **Static analysis.** The module-boundary import-restriction lint (`coding-standards.md` Section 12) is one instance of a broader principle: static analysis catches what code review alone degrades under time pressure to catch reliably — every mechanical rule in this document that *can* be enforced by a linter or a CI check (money-type usage, ULID usage, forbidden `any`, the import-boundary rule) SHOULD be, over time, rather than relying on review discipline indefinitely.

---

## 19. Git Workflow

**Branch strategy.** Trunk-based, short-lived feature branches off `main` — `<type>/<short-kebab-description>` (Section 4's table), where `<type>` is one of `feat`, `fix`, `chore`, `refactor`, `docs`, `test`, `perf`, `security`. A branch lives only as long as its pull request is open; long-running, divergent branches are avoided because they make the module-boundary and architecture-compliance checks (Section 23) stale by the time they're finally reviewed.

**Commit message conventions.** Conventional-commits-style: `<type>(<scope>): <summary>`, where `<scope>` is the module name (matching Section 4's naming rule) where applicable — `fix(inventory): prevent negative on-hand quantity on concurrent reservation release`. The summary states *why*, not only *what*, wherever the *why* isn't obvious from the diff alone (this document's own general commenting philosophy, Section 18, applied to commit messages).

**Pull requests.** Every PR description states: what changed, why, which module(s) it touches, and which ADR(s)/architecture document(s) it's grounded in for any non-obvious choice (`08-ai-development/ai-development-rules.md` Section 5's citation discipline, restated as a PR-authoring rule). A PR that crosses a module boundary flags that explicitly, since it's exactly the kind of change the Architecture Compliance Checklist (`coding-standards.md` Section 11) exists to scrutinize hardest.

**Reviews.** At least one review is REQUIRED before merge (even in a solo-developer-plus-AI-assistant context, a second pass — human or a distinct review agent — is the review; Section 23's checklist is what it's checked against). A review that only checks "does it work" without running the Code Review and Architecture Compliance checklists (`coding-standards.md` Sections 10–11) has not actually reviewed the change against this project's own standard.

**Merge strategy.** Squash-merge to `main` by default, so `main`'s history reads as one logical change per merged PR, matching the one-commit-message-tells-the-whole-story discipline above; a PR with several genuinely distinct, independently-revertible changes SHOULD be split into separate PRs rather than squashed into one commit that hides that they were separable.

**Release tagging.** A Production release is tagged against the exact artifact promoted from Staging (`infrastructure-and-release.md` Section 11 — the same built artifact, never a separate Production-specific build), using semantic-version-shaped tags (`vMAJOR.MINOR.PATCH`) tracking the deployable's own release history, distinct from the API's own version scheme (Section 5), which tracks contract compatibility, not deployable release cadence.

**Hotfix process.** A Production defect requiring an out-of-band fix branches from the tagged release currently in Production (not from `main`'s latest, which may already contain unrelated, unreleased changes), goes through the same CI gate (Section 20) with no skipped checks, and is merged back into `main` immediately after release so `main` never diverges from what's actually running.

---

## 20. CI/CD Standards

Operationalizes `infrastructure-and-release.md` Section 11 (ADR-0030: GitHub Actions) into a concrete, required pipeline shape.

**Required pipeline stages, on every change, with no stage skippable via an override flag:**
1. **Build validation** — the deployable artifact builds cleanly; a build failure blocks merge outright.
2. **Linting** — the formatter/linter (Section 18) and the module-boundary import-restriction check (`coding-standards.md` Section 12) both run as required checks.
3. **Tests** — the full automated test suite (Section 16) runs; a failing test blocks merge.
4. **Security scanning** — dependency vulnerability scanning and secrets scanning (Section 10 — no committed credential reaches `main`) run on every change.
5. **Migration checks** — a schema migration (Section 11) is validated (applies cleanly against a fresh schema, and, where feasible, against a representative existing schema) before it's allowed to merge.
6. **Artifact generation** — a single build artifact is produced once, then promoted unmodified through Staging to Production (`infrastructure-and-release.md` Section 11) — never rebuilt per environment.
7. **Deployment approvals** — promotion from Staging to Production requires an explicit approval gate; Development and Staging deploy automatically on merge to their respective tracked branches, Production does not.

A change that fails any required stage does not merge — there is no "merge now, fix CI later" path for a required check, consistent with this project's own "never skip hooks" discipline carried from engineering practice into the pipeline itself.

---

## 21. Documentation Standards

- **Markdown** is the format for every document in this set — this document included — consistent with `architecture/README.md` Section 6's naming and formatting conventions.
- **Architecture updates.** A change that alters a module's public interface, dependency, or published/consumed event MUST update `module-catalog.md` (and any other affected `03-decomposition/` document) in the same change — not as a follow-up (`coding-standards.md` Section 9).
- **ADRs.** A significant, hard-to-reverse decision made during implementation (a new library category, a new external dependency, a schema change altering an already-Accepted ADR's assumption) gets an ADR before or alongside the code, following `decisions/README.md`'s process exactly — never retroactively, never skipped for feeling small at the time.
- **SDDs.** Every module's SDD MUST cite this document (Section 1) and MUST NOT restate a rule this document already makes at MUST-level; an SDD's own content is the module-internal detail (exact schema, exact endpoint list, exact event payloads) layered on top.
- **Diagrams.** Diagrams-as-code (Mermaid/PlantUML), stored as text under `architecture/diagrams/source/`, per `diagrams/README.md`'s convention — never an opaque, binary-only diagram file that can't be diffed or reviewed like any other document.
- **CHANGELOG.** A structural change to the documentation set itself (a new document, a renumbered section, a changed convention) is recorded in `architecture/CHANGELOG.md`, newest entry first, per its existing format — content-level history for a specific document remains in git blame and the ADRs that justified the change.
- **README.** `architecture/README.md`'s document index and "Current status" section are updated whenever a document's authorship status changes — an out-of-date index is itself a documentation defect.
- **Cross references.** A document references another by its exact path and, where citing a specific rule, its section — never a vague "see the architecture docs" pointer that forces the reader to search. This document's own citations throughout are the model to follow.

---

## 22. Definition of Done

A change is not "done" until every item below is true — not merely attempted — building directly on `08-ai-development/ai-development-rules.md` Section 11's Definition of Done, extended with this document's own concrete standards:

- [ ] **Architecture compliance** — passes the Architecture Compliance Checklist (`coding-standards.md` Section 11) in full; no module boundary, dependency rule, or ADR is contradicted.
- [ ] **SDD compliance** — satisfies the owning module's SDD, and the SDD itself is updated if the change reveals a gap in it.
- [ ] **Tests passing** — the full CI pipeline (Section 20) is green, including tests proportional to correctness criticality (Section 16).
- [ ] **Documentation updated** — `module-catalog.md` and any other affected architecture document (Section 21) reflect the change; no Open Decision was silently resolved without either an explicit note or a new ADR.
- [ ] **API documented** — the OpenAPI spec (Section 5) reflects every new or changed endpoint; every new event is reflected in the module's Published/Consumed Events list.
- [ ] **Security reviewed** — no hardcoded secret; RBAC and store-scope checks are present and correctly scoped (Section 10); step-up authentication is wired for any of the five operations Section 10 names, where applicable.
- [ ] **Performance reviewed** — no new query lacking an index on a latency-sensitive path (Section 17); no N+1 pattern introduced; pagination present on any new collection endpoint.
- [ ] **Logging implemented** — structured, correlation-ID-carrying, PII-masked, per Section 9.
- [ ] **Monitoring implemented** — any new correctness-critical or externally-facing path emits the metrics its category requires (`observability-and-operations.md` Section 5).
- [ ] **Code review completed** — by at least one reviewer, against the Code Review Checklist (`coding-standards.md` Section 10) and this document's own standards, not merely "does it run."

---

## 23. Developer Checklist

A concise, pre-PR checklist every engineer (or AI assistant) runs through before opening a Pull Request — the fast, personal pass that precedes the fuller Definition of Done (Section 22) and Code Review Checklist (`coding-standards.md` Section 10) a reviewer applies afterward:

- [ ] Did I read the owning module's `module-catalog.md` entry (and, if crossing a boundary, `module-communication.md`) before writing code, not from memory?
- [ ] Does every new identifier use a ULID, every timestamp UTC, every monetary value integer halala — with no exception? (Section 11)
- [ ] Is business logic in the service/domain layer, never the controller? (Section 13)
- [ ] Did I validate at the boundary (DTO/controller) and re-validate business state immediately before any commit that depends on it? (Sections 6–7)
- [ ] Does every new endpoint follow the URI/status-code/pagination/versioning conventions in Section 5, and is it in the OpenAPI spec?
- [ ] Does every workflow this document or `data-architecture.md` Section 13 names as idempotency-required actually implement idempotency, with a test proving it? (Sections 14, 16)
- [ ] Did I check the RBAC permission catalogue and, for a store-scoped operation, the store-scope check — and step-up authentication if the operation is one of Section 10's five high-risk operations?
- [ ] Are secrets entirely absent from my diff — no credential, key, token, or raw PII in a log line? (Sections 9–10)
- [ ] Did I add or update tests proportional to what I touched, including the named race condition or idempotency test if applicable? (Section 16)
- [ ] Did I update `module-catalog.md` (or the relevant document) if I changed a public interface, dependency, or event? (Section 21)
- [ ] Does my commit/branch naming follow Section 19, and does my PR description cite the ADR(s)/document(s) behind any non-obvious choice?
- [ ] Would this change pass every item in Section 22's Definition of Done if merged right now?

---

## 24. AI Development Guidelines

This section does not replace `08-ai-development/ai-development-rules.md` — that document remains the full, authoritative operational procedure for an AI coding assistant working on this project (required context per task type, forbidden patterns, prompting guidelines, known hallucination catalog). This section states the subset of that document's rules that specifically govern how AI-generated code relates to *this* document's engineering standards, as the task instructing this document's creation requires:

- **AI MUST never violate the architecture.** No AI-generated change contradicts an Accepted ADR, a module boundary (`module-catalog.md`), or a cross-cutting rule (`04-cross-cutting/`) — a gap or an apparent conflict is flagged and a resolution proposed, never silently resolved by picking an interpretation and proceeding (`ai-development-rules.md` Section 2, Section 4).
- **AI MUST NOT introduce new dependencies** (a library, a technology category, an external integration) not already named in `04-cross-cutting/technology-decisions.md` or Section 2 of this document, without first flagging that it requires a new ADR — never adding a dependency quietly because it solves the immediate problem conveniently.
- **AI MUST NOT bypass module boundaries** — every cross-module interaction goes through a public interface, a domain event, or a documented external integration (Sections 12–15); a "just this once" direct query into another module's table is never acceptable, including from AI-generated test or migration code.
- **AI MUST follow ADRs** — an Accepted ADR is binding; where this document and an ADR appear to disagree, the ADR wins and this document is treated as needing correction (this document's own Authority statement, Section header).
- **AI MUST NOT invent APIs** — a REST endpoint, an event name, or a permission name not already documented (`module-catalog.md`, this document's Sections 5/14) is a proposal to be flagged and added to the relevant document as part of the same change, never fabricated as if it already existed or silently assumed to follow an unstated convention.
- **AI-generated code MUST pass all quality gates** — the full CI pipeline (Section 20), the Developer Checklist (Section 23), and the Definition of Done (Section 22) apply identically to AI-authored and human-authored code; there is no separate, lighter bar for AI-generated output.
- **Human review remains mandatory.** No AI-generated change reaches `main` without the review Section 19 already requires — an AI assistant may draft, propose, and self-check against every checklist in this document, but merge authority and final architectural judgment remain with a human reviewer, consistent with Constitution Section 9's "documentation is the substitute for memory, not for judgment" framing.

For everything beyond this subset — required reading before starting a task, the full forbidden-pattern list, prompting discipline, and the named catalogs of common AI hallucinations and architectural violations specific to this codebase — `08-ai-development/ai-development-rules.md` is authoritative and is not duplicated here.

---

## Open Decisions

Carried forward, not newly introduced, from the documents this one builds on — this document does not silently resolve any of them:

- **Monorepo vs. polyrepo** for the Backend and four clients (`coding-standards.md` Section 14) — Section 3 above.
- **Specific test framework, linter, formatter, and import-restriction package** (`coding-standards.md` Sections 8, 12, 14) — Sections 16, 18, 20 above state what these tools must guarantee, not which tools are chosen.
- **Specific code-coverage thresholds** (`coding-standards.md` Section 8) — Section 16 states the principle (proportional to correctness criticality), not a percentage.
- **Accessibility standards** (`nfr-matrix.md` Section 14) — outside this document's backend-implementation scope; remains recommended pre-launch product/design work.
- **A formal legal/compliance review** (`nfr-matrix.md` Section 3.17) — this document's Security Standards (Section 10) are architecture's own assessment, not a substitute for one.
- **Monitoring/alerting tool choice** (`observability-and-operations.md` Section 17) — Section 9 above states the required log/metric shape, not the vendor.

None of the above is a gap in this document's own authorship — each is a genuinely below-architecture or outside-this-layer choice that a future SDD, a tooling decision, or a dedicated pre-launch workstream resolves, exactly as `07-coding-standards/coding-standards.md` and `06-quality-attributes/nfr-matrix.md` already state for themselves.
