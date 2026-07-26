# Architecture Decision Records (ADRs)

## Purpose
An ADR captures one significant architecture decision: the context that forced it, the options considered, the choice made, and the consequences accepted. ADRs are the project's memory — they exist so that six months from now (or so an AI coding assistant reading cold) the reasoning behind a structural choice doesn't have to be reverse-engineered from code.

## What qualifies as ADR-worthy
Record a decision here if it is:
- Expensive or awkward to reverse (choice of database, auth strategy, service boundary, client-app split)
- Likely to be questioned later ("why don't we just use X instead?")
- A deviation from, or an application of, a principle in `00-Architecture-Principles.md`

Do not record: implementation details, naming choices, or anything reversible in an afternoon. Those belong in code comments or SDD documents, if anywhere.

## Numbering and naming
`NNNN-short-kebab-title.md`, zero-padded four digits, strictly sequential by creation order, never reused or renumbered — even if a record is later superseded or rejected.

## Lifecycle
Every ADR has exactly one status at a time:

| Status | Meaning |
|---|---|
| `Proposed` | Under consideration, not yet acted on |
| `Accepted` | In effect — implementation should conform to it |
| `Superseded by NNNN` | No longer in effect; replaced by a newer ADR (link both directions) |
| `Deprecated` | No longer in effect; not replaced by anything (rare — usually a feature was removed) |

**ADRs are immutable once Accepted.** A changed decision is a *new* ADR that supersedes the old one, not an edit to the old one. This preserves the historical record of what was decided and when — editing history is how documentation stops being trustworthy.

## Index
Records are listed here as they're created, oldest first.

| # | Title | Status |
|---|---|---|
| [0001](0001-record-architecture-decisions.md) | Record architecture decisions | Accepted |
| [0002](0002-modular-monolith-over-microservices.md) | Modular monolith over microservices | Accepted |
| [0003](0003-postgresql-as-single-source-of-truth.md) | PostgreSQL as single source of truth | Accepted |
| [0004](0004-redis-as-cache-and-coordination-only.md) | Redis as cache/coordination only | Accepted |
| [0005](0005-typescript-centered-stack.md) | TypeScript-centered stack | Accepted |
| [0006](0006-react-native-for-customer-and-worker-mobile-apps.md) | React Native for Customer and Worker mobile apps | Accepted |
| [0007](0007-nextjs-for-customer-web-app.md) | Next.js for Customer Web App | Accepted |
| [0008](0008-react-web-admin-dashboard.md) | React web Admin Dashboard | Accepted |
| [0009](0009-module-ownership-no-cross-module-repository-access.md) | Module ownership and no cross-module repository access | Accepted |
| [0010](0010-transactional-checkout-and-inventory-reservation.md) | Transactional checkout and inventory reservation | Accepted |
| [0011](0011-event-driven-asynchronous-side-effects.md) | Event-driven asynchronous side effects | Accepted |
| [0012](0012-rbac-with-fine-grained-permissions.md) | RBAC with fine-grained permissions | Accepted |
| [0013](0013-ulids-for-entity-identifiers.md) | ULIDs for entity identifiers | Superseded by [0018](0018-uuid-primary-keys.md) |
| [0014](0014-utc-timestamp-storage.md) | UTC timestamp storage | Accepted |
| [0015](0015-integer-halala-money-representation.md) | Integer halala money representation | Accepted |
| [0016](0016-object-storage-for-files-and-images.md) | Object storage for files and images | Accepted |
| [0017](0017-versioned-apis-from-day-one.md) | Versioned APIs from day one | Accepted |
| [0018](0018-uuid-primary-keys.md) | UUID primary keys (supersedes ULIDs) | Superseded by [0019](0019-restore-ulid-primary-keys.md) |
| [0019](0019-restore-ulid-primary-keys.md) | Restore ULID primary keys (supersedes 0018; corrects DDD) | Accepted |
| [0020](0020-module-ownership-reconciliation.md) | Reconcile module ownership and add Support module | Accepted |
| [0021](0021-checkout-transaction-boundary-clarification.md) | Clarify checkout transaction boundaries | Accepted |
| [0022](0022-moyasar-payment-gateway.md) | Moyasar payment gateway | Accepted |
| [0023](0023-unifonic-sms-otp-provider.md) | Unifonic SMS/OTP provider | Accepted |
| [0024](0024-aws-me-central-2-deployment-region.md) | AWS me-central-2 deployment region | Accepted |
| [0025](0025-nestjs-backend-framework.md) | NestJS backend framework | Accepted |
| [0026](0026-jwt-session-model.md) | JWT access tokens with rotating refresh tokens | Accepted |
| [0027](0027-otp-first-authentication.md) | OTP-first authentication | Accepted |
| [0028](0028-cdn-for-public-cacheable-content.md) | CDN for public cacheable content | Accepted |

ADRs 0002–0017 backfill the decisions already implicit in the finalized SRD and the ADS's Section 8 (Architectural Decisions Summary) and Section 9 (Technology Decisions) — see `01-Architecture-Design-Specification.md`. ADR-0018 was authored once `DDD_Sakhari_Ecom_v1.0.md` became available and was found to specify UUID primary keys, conflicting with ADR-0013's ULID choice made before that document existed. The DDD's UUID text was subsequently identified as itself the error — ULID was the intended decision all along — and corrected; ADR-0019 supersedes ADR-0018 to restore ULIDs and record the full sequence rather than erase it. ADRs 0020–0028 close the major open architecture gaps found before SDD work: module ownership reconciliation, checkout transaction wording, finalized SRD providers (Moyasar, Unifonic, AWS `me-central-2`), and technology backfill decisions for NestJS, JWT, OTP, and CDN. The current, effective identifier decision is ADR-0019.

## How AI coding assistants should use this folder
Before generating code that makes or changes a structural choice (new dependency, new service, new datastore, new external integration), scan this index for `Accepted` records that constrain the choice. Treat `Accepted` ADRs as binding unless the user explicitly asks to revisit one — in which case, propose a new ADR rather than silently contradicting the old one.
