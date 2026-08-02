# Security Architecture — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. |
| **Stability** | Stable in principle (identity is centralized, authorization is centrally evaluated, nothing trusts a client's self-reported permissions — Principle 4.6, ADS Security Philosophy); evolving in specific control detail as threats, features, and regulatory guidance evolve. |
| **Authority** | Subordinate to `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `decisions/README.md`, `02-context/system-context.md`, `04-cross-cutting/technology-decisions.md`, and `03-decomposition/module-catalog.md` / `module-communication.md`. Does not repeat their content — see Section 2. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** Defines how Sakhari Ecom knows who is calling, decides what they're allowed to do, protects the credentials and secrets that make both possible, and records security-relevant activity — as one coherent model applied identically across all sixteen backend modules, not a per-module choice. This document is where the SRD's compliance obligations (SAMA, PDPL, mada) and the Constitution's "security is a default state, not an added feature" conviction (Section 11) become concrete, binding rules.

**Scope.** Covers identity, authentication (OTP, JWT, refresh tokens), authorization (RBAC, permissions), API-level security, the security dimension of audit logging, and secrets/credential management, plus a compliance mapping from SAMA/PDPL obligations to the controls in this document that satisfy them. It does **not** cover: *why* JWT and OTP were chosen over alternatives (`04-cross-cutting/technology-decisions.md` Sections 8—9 — this document assumes those choices and describes how they're used architecturally); per-module dependency rules (`03-decomposition/module-communication.md`); the Audit module's data model and push-mechanism detail (`04-cross-cutting/data-architecture.md` Section 15 — this document adds the security *rationale* for audit, not its data mechanics); event delivery mechanics (`04-cross-cutting/integration-and-messaging.md`); or infrastructure-level network security, TLS termination, or secret-storage topology (`05-deployment/infrastructure-and-release.md`). No code, no token formats, no cryptographic parameters — those are SDD-level.

**Intended audience.** The project owner, an AI coding assistant implementing or reviewing anything touching identity, authorization, or secrets, and the author of any future module SDD — for whom this document's rules are the floor every module's security-relevant code must meet, not a suggestion to weigh against convenience (Constitution Principle 4.12).

**Cross-references.** Grounded in ADR-0012 (RBAC with Fine-Grained Permissions); `module-catalog.md`'s Auth and User entries (Section 4.1—4.2) and ADR-0020 (the Auth/User split); `module-communication.md`'s Section 3 (Auth's one named synchronous exception, calling Notification for OTP dispatch) and Section 7 (Auth as the Foundation-layer module everything else depends on); `04-cross-cutting/technology-decisions.md` Sections 8 (JWT) and 9 (OTP); `02-context/system-context.md`'s Trust Boundaries per zone and its treatment of the SMS/OTP Provider and Push Notification Service as external systems; and DDD Section 12 (Audit Strategy) and Section 7.5 (Business Integrity Rules — permission/role checks, rider OTP/proof validation).

## 2. Security Principles

Four convictions govern every rule in this document:

- **Security is centralized, never re-implemented per client or per module.** Every authentication and authorization decision is made once, by the Auth module (identity) and the RBAC evaluation it anchors (permissions) — never independently by a client's UI, and never by a second, parallel check invented inside another module (Principle 4.6). A client's own restrictions (graying out a button) are a usability courtesy, not a security control (`02-context/system-context.md` Sections 3—4).
- **Nothing is trusted by default across a boundary.** Every request crossing from a client into the Backend is authenticated and authorized fresh, regardless of what the client believes about its own permissions (`02-context/system-context.md` Section 5's Trust Boundaries). Every response from an external system (the Payment Gateway, the SMS/OTP Provider) is validated before being treated as fact, for the same reason.
- **Compromise of one surface must not cascade.** The Customer Platform, the Operations Platform's two applications, and every external integration are all independently authorized against the same central model, so that compromising one grants nothing toward another (`02-context/system-context.md` Section 5, Section 12).
- **Every consequential action is attributable and recorded.** Authentication events, authorization decisions on sensitive actions, and administrative overrides are all within Principle 4.10's audit scope — security and auditability are the same discipline applied to different questions ("who is this" and "what did they do"), not two separate concerns.

## 3. Identity

Identity is split across two modules, per ADR-0020: **Auth** owns the core identity/credential record (the DDD's User entity, Section 5.1) — the bare fact of who someone is and how they prove it — while **User** owns profile data (Customer Profile, Address, and the folded-in Workforce entities). This split keeps authentication concerns (Section 4) and profile concerns cleanly separated: a module needing to know "is this caller who they claim to be" talks to Auth; a module needing to know "what is this person's name/address/shift" talks to User. Neither module trusts the other's data directly — User resolves the identity a profile belongs to by calling Auth's interface (`module-catalog.md` Section 4.2's Dependencies), never by assuming a shared table.

Every actor in the system — Customer, Picker, Rider, Admin, Ops, Support — is represented by exactly one identity record in Auth, regardless of which client application they use to reach the system (`02-context/system-context.md` Section 2). There is no separate identity system per client.

## 4. Authentication

Authentication is phone-based and OTP-driven for every actor — there are no passwords anywhere in this system (`04-cross-cutting/technology-decisions.md` Section 9's own rationale: no password storage or reset liability, and a better fit for the platform's phone-first Saudi market). The flow, at the architecture level:

1. A client requests a login code for a phone number. The request reaches Auth's `RequestOtp` interface (`module-catalog.md` Section 4.1).
2. Auth calls Notification's `SendNotification` synchronously — the one named exception to the "no synchronous dependency on Notification" rule (`module-communication.md` Section 3) — specifically to get a dispatch confirmation before telling the client to expect a code.
3. Notification calls the external SMS/OTP Provider (`02-context/system-context.md`) and reports dispatch status back to Auth.
4. The client submits the received code to Auth's `VerifyOtp` interface. Auth validates it, and on success issues a token pair (Section 5) and establishes a session.
5. Auth publishes `UserAuthenticated` and `OtpRequested` (`module-catalog.md` Section 4.1's Published Events) — the latter specifically to give Section 9's audit trail a record that a login attempt occurred, independent of whether it succeeded.

**The same mechanism, reused, not duplicated, for delivery confirmation.** Rider proof-of-delivery uses OTP as well (DDD Section 7.5, "Rider OTP/proof validation") — a code shown to the customer and entered by the rider at handoff. This is architecturally the same OTP verification capability Auth exposes for login, invoked in a delivery context by the Delivery module rather than introducing a second, parallel verification mechanism (`04-cross-cutting/technology-decisions.md` Section 9's own reasoning for why one mechanism serves both purposes).

**OTP abuse protection (ADR-0039).** Because OTP is the sole authentication factor for every actor, the request/verification flow is protected by concrete, numeric defaults — all configurable through Settings, none hardcoded:

| Control | Default |
|---|---|
| Maximum OTP requests per phone number | 5 per hour |
| Maximum verification attempts per issued code | 5 per code |
| Resend cooldown | 60 seconds between requests |
| Temporary lockout | 30 minutes, triggered after the request or attempt ceiling is reached |
| Device- and IP-based throttling | Enabled, independent of and in addition to the phone-number-scoped limits above |
| Progressive backoff | Applied after repeated failures within a lockout-eligible window |
| OTP length | 6 digits |
| OTP expiry | 5 minutes from issuance |

Phone-scoped and device/IP-scoped throttling operate independently, so an attacker cannot bypass one limit by rotating what the other is keyed on. Redis continues to enforce these limits (Section 8) as non-authoritative, coordination-only counters — losing them on a flush degrades to a temporarily looser limit, never an incorrect authorization decision.

## 4A. Step-Up Authentication (ADR-0040)

Customers authenticate via OTP only (Section 4, unchanged). Four privileged roles — **Super Admin, Operations Manager, Finance, and Support Lead** — are classifications within the existing Admin/Ops/Support role categories (Section 7) that require a second, independent verification (**step-up authentication**) before performing a high-risk operation, layered on top of the OTP-authenticated session and the RBAC permission check those operations already require.

Step-up authentication uses either a **TOTP** (app-generated time-based one-time password) or an **approved enterprise MFA** mechanism — the specific provider is an SDD-level choice, not decided here.

**High-risk operations requiring fresh step-up authentication** (evaluated at the moment of the operation, never cached across a session the way a normal permission claim is):
- Refund approval (`refund:approve`)
- Permission changes (role/permission grants or revocations)
- System settings changes (`settings:manage`)
- Financial adjustments (Payment Ledger `Adjustment`/`Chargeback` entries, per ADR-0037)
- Sensitive audit access (an actor's detailed activity, beyond an actor's own routine operational views)

A step-up challenge is independent of the actor's underlying OTP session and the RBAC permission check — even a fully compromised phone number (a SIM-swap, the specific risk ADR-0027 named without a compensating control) and a validly-issued session cannot alone perform a high-risk operation without also passing step-up. Auth's `InitiateStepUpChallenge`/`VerifyStepUpChallenge` interfaces (`module-catalog.md` Section 4.1) carry this; `StepUpAuthenticationCompleted`/`StepUpAuthenticationFailed` are both within Section 9's audit scope.

## 5. JWT

Once authenticated, a caller holds a JWT access token, carrying identity and role/permission claims, presented on every subsequent request (`04-cross-cutting/technology-decisions.md` Section 8). Architecturally, this token is what lets every module's request-handling path answer "who is calling and what are they allowed to do" without a database round-trip on the common path — Auth's `ValidateToken` interface (`module-catalog.md` Section 4.1) verifies the token's signature and claims in-process, consistent with the modular monolith's single-deployable shape (ADR-0002).

**What the token is not:** it is not itself an authorization decision. A valid, unexpired token proves identity and carries role claims; whether a specific action is *permitted* is still evaluated by RBAC (Section 7) against those claims on every request — a token is never treated as a standing grant to do anything the claims don't specifically cover.

## 6. Refresh Tokens and Session Management

Because JWTs are hard to revoke instantly by design (`04-cross-cutting/technology-decisions.md` Section 8's own named drawback), access tokens are short-lived, and a longer-lived refresh token is used to obtain new ones without forcing re-authentication via OTP on every expiry. Per ADR-0033, the concrete parameters are:

- **Access token TTL: 15 minutes.** Short enough that a leaked access token has a small, bounded window of usefulness even with no server-side revocation check on the common request path (Section 5).
- **Refresh token TTL: 7 days idle, 30 days absolute.** A refresh token not used for 7 days expires even if within its 30-day absolute lifetime; the 30-day absolute cap bounds how long a session can persist through ordinary, continued use before OTP re-authentication is required again. There is no session lifetime beyond these two bounds — a session is not indefinite by design.
- **Rotation on use.** Each use of a refresh token to obtain a new access token also issues a new refresh token and invalidates the one just used — a refresh token is single-use, not a standing credential reused indefinitely.
- **Reuse detection.** Presenting an already-rotated (previously used) refresh token is treated as a signal of possible compromise — the entire token family descending from that refresh token is invalidated, forcing re-authentication, rather than silently issuing another token pair.
- **Revocation strategy.** Auth's `RevokeSession` interface (`module-catalog.md` Section 4.1) invalidates a refresh token (and its full family) on demand — immediate and explicit, not deferred to natural expiry. Used for: user-initiated logout, administrative action against a compromised or terminated account, and the reuse-detection path above.
- **Logout behavior.** A client-initiated logout calls `RevokeSession` for the current session's refresh token family and discards its local access token; the access token already issued remains technically valid for up to its remaining 15-minute TTL (no server-side check blocks it mid-life on the common path), but cannot be renewed once its refresh token is revoked — logout's real effect is "this session cannot refresh again," not an instant kill of an in-flight access token.
- **Refresh behavior.** A client presents its current refresh token before its access token expires (or promptly after); Auth validates it against the stored session/family record, and on success issues a new access/refresh pair per the rotation rule above. A refresh presented after the refresh token's own TTL has lapsed forces full OTP re-authentication — there is no silent extension beyond the stated bounds.
- **Device handling.** Each login (Section 4) establishes its own session and refresh-token family, scoped to the device/client instance that authenticated — one identity (Auth's User record) may hold multiple concurrent sessions (e.g., Customer Mobile App and Customer Web App at once), each independently revocable via `RevokeSession` without affecting the others. There is no cap on concurrent sessions per identity at the architecture level; a per-device session limit, if ever needed, is a future ADR-worthy policy change, not an implicit default.
- **Redis/PostgreSQL responsibilities.** Per ADR-0033, PostgreSQL is authoritative for sessions, refresh-token families, and revocation/reuse-detection state (Auth's Session and Refresh Token entities — `module-catalog.md` Section 5) — a session's existence and validity can always be reconstructed from PostgreSQL alone. Redis may cache validation metadata (e.g., a fast-path lookup to avoid a PostgreSQL round-trip on every request) purely as an accelerator, consistent with Redis's non-authoritative role (`data-architecture.md` Section 3) — losing a Redis-cached session entry on a flush degrades to a PostgreSQL lookup, never to an incorrect authorization decision.

**Why rotation-with-reuse-detection, not a simple long-lived refresh token:** it gives the system a way to detect and respond to token theft (a stolen, already-used refresh token being replayed) without requiring every request to hit a session store — the common case (a legitimate client rotating its own token) stays fast, and the attack case (replay) is the one path that pays the cost of triggering full revocation.

## 7. Authorization, RBAC, and the Permission Model

Authorization is Role-Based Access Control with fine-grained, named permissions (ADR-0012), evaluated centrally by Auth for every request — never re-implemented or approximated by an individual module.

- **Roles:** Customer, Picker, Rider, Admin, Ops, Support — the SRD's actor categories, with Admin/Ops/Support as RBAC-differentiated roles within the single "Admin/Ops" actor category the SRD names (`02-context/system-context.md` Section 2).
- **Permission naming convention:** every permission is named `<resource>:<operation>`, where `<resource>` matches a module or entity name from `module-catalog.md` exactly (no synonyms, per `coding-standards.md` Section 4) and `<operation>` is a lowercase verb describing the allowed action. This is the one naming scheme for permissions system-wide — no module invents its own shape (`manage-catalog` in the earlier illustrative form is superseded by `catalog:manage` below, for consistency).
- **The permission catalogue (ADR-0033) is the authoritative, current list**, one module at a time:
  | Module | Permissions |
  |---|---|
  | Auth | `auth:request_otp`, `auth:verify_otp`, `auth:refresh`, `auth:revoke_session` |
  | User | `user:read_self`, `user:update_self`, `worker:read`, `worker:manage`, `shift:manage` |
  | Store | `store:read`, `store:manage`, `service_zone:manage` |
  | Catalog | `catalog:read`, `catalog:manage` |
  | Inventory | `inventory:read`, `inventory:adjust`, `inventory:reconcile` |
  | Cart | `cart:read_self`, `cart:mutate_self` |
  | Order | `order:create`, `order:read_self`, `order:read_store`, `order:manage`, `order:cancel` |
  | Payment | `payment:read`, `payment:collect_cod`, `payment:reconcile`, `refund:request`, `refund:approve`, `refund:issue` |
  | Promotion | `promotion:read`, `promotion:manage` |
  | Delivery | `picking:work`, `delivery:work`, `delivery:assign`, `rider_location:write` |
  | Notification | `notification:read_self`, `notification:manage_templates` |
  | Support | `support:read`, `support:manage`, `support:comment`, `support:escalate` |
  | Analytics | `analytics:read` |
  | Audit | `audit:read` |
  | Settings | `settings:read`, `settings:manage` |

  Every permission belongs to exactly one owning module — the same module that owns the resource named in the permission — consistent with Principle 4.5. A permission is grouped into one or more roles (Customer holds `cart:*`, `order:create`, `order:read_self`, and similar self-scoped permissions; Admin holds the broadest set; Ops and Support hold operational subsets appropriate to their role, not the full Admin set), rather than a single coarse flag per role — this is what lets an Ops user and a Support user, both inside the Admin/Ops Dashboard, have genuinely different capability without collapsing into one over-privileged role (ADR-0012's own rationale).
- **Read, write, and administrative/operational operations are distinguished by name, not inferred from context:** a `:read` (or `:read_self`/`:read_store`) permission never implies a corresponding `:manage`/`:adjust`/`:issue`/`:approve` permission, and vice versa — `inventory:read` does not grant `inventory:adjust`; `order:read_self` does not grant `order:manage`. Every module's catalogue entry above separates its read-only operations from its create/update/delete and administrative operations by distinct permission names, so a role can be granted visibility without capability, or capability scoped narrower than full visibility (e.g., `order:read_store` for store staff without `order:manage`).
- **Future extensibility.** A new module follows the same `<resource>:<operation>` convention when it's added (`capability-boundary-map.md` Section 7's governance process applies to its permissions exactly as it applies to its boundary). A new operation on an existing module's resource gets a new named permission, never an implicit broadening of an existing one — consistent with ADR-0017's additive-change discipline applied to the permission model instead of the API.
- **Every module's controller checks permissions before delegating**, not after — a request that fails an RBAC check never reaches the owning module's business logic at all. This keeps authorization a single, consistent gate at the API boundary (`module-communication.md` Section 4) rather than a check every module must remember to perform itself.
- **No client is trusted to self-report its role or hide unauthorized UI as a substitute for enforcement** — the Admin/Ops Dashboard may hide an Admin-only action from an Ops user's screen for usability, but the Backend re-checks the permission on the actual request regardless (`02-context/system-context.md` Section 4's Trust Boundaries).

**Resource Scope, Store Scope, and Permission Evaluation Order (ADR-0041).** A permission alone is insufficient for any store-scoped operation. Authorization for a store-scoped resource requires **Permission AND Store Scope**: the actor's role must grant the named permission, *and* the resource's store must be one of the actor's **Assigned Stores** — `order:manage` for Store A does not authorize the same operation against Store B. Every operational user (Picker, Rider, Admin, Ops, Support) carries three evaluated attributes: Assigned Stores, Assigned Roles, and Assigned Permissions (the permissions those roles grant).

**Permission Evaluation Order:** (1) the caller is authenticated (`ValidateToken`); (2) the caller's role grants the named permission for the operation; (3) where the operation targets a store-scoped resource, the resource's store is present in the caller's Assigned Stores — skipped only for operations that are not store-scoped by nature (e.g., `settings:manage` against a global setting). Failing any step denies the request.

Store is the current, only instance of a more general **Resource Scope** concept (named generally so a future scope dimension does not require re-deriving the model). Assigned Stores is owned by Auth as Auth-internal authorization state (`module-catalog.md` §4.1's Ownership field), the same "not separately named in the DDD" treatment already established for Session and Refresh Token — carried in `ValidateToken`'s returned claims alongside role/permission claims, not a second, separately-fetched lookup. Assigning or changing a user's Assigned Stores is itself a permission change, subject to Section 4A's step-up authentication requirement and Section 9's audit scope.

**Future regional scope extension.** As the platform expands beyond a single region (DDD Section 17.1), Resource Scope is expected to gain a second dimension (Region, above Store) — a future ADR extending this same Permission-AND-Scope evaluation model, not a new, parallel authorization mechanism.

## 8. API Security

- **One versioned, authenticated surface.** Every client reaches the Backend through the single versioned REST API (ADR-0017; `module-communication.md` Section 4) — there is no unauthenticated endpoint carrying business consequence, and no module-specific API surface that bypasses the shared authentication/authorization gate.
- **Validation at the boundary.** Input is validated where the system meets the outside world — the request-handling path, before a thin controller ever calls into a module's service layer (Constitution Section 8's "error handling exists at boundaries," Principle 4.7). Internal, module-to-module calls trust the guarantees already established at that boundary rather than re-validating defensively at every hop.
- **Rate limiting** is enforced using Redis (`04-cross-cutting/data-architecture.md` Section 3) — a coordination use case entirely consistent with Redis's non-authoritative role: a rate-limit counter can be lost on a Redis flush with no business-data consequence, only a temporary loosening of the limit.
- **External responses are never trusted uncontested.** A Payment Gateway callback, an SMS/OTP Provider delivery-status webhook, or a Push Notification Service response is validated (signature/webhook verification, expected-shape checks) before the Backend treats it as fact — the same boundary discipline applied to inbound client requests, applied symmetrically to outbound integrations (`02-context/system-context.md` Section 5's Trust Boundaries).

## 9. Audit

Audit is a security control, not only a bookkeeping one: it is what makes "who did this, and were they allowed to" answerable after the fact, for disputes, incident investigation, and regulatory review alike (Principle 4.10). The data model, push mechanism, and full field-level detail belong to `04-cross-cutting/data-architecture.md` Section 15 and the Audit module's entry in `module-catalog.md` — this section states only the security-specific framing:

- **Authentication events** (`OtpRequested`, `UserAuthenticated`, session revocation) are within audit scope, independent of whether they succeeded — a pattern of failed authentication attempts is itself security-relevant.
- **Sensitive administrative actions** — permission/role changes, refunds, price/settings overrides, and any action DDD Section 12.6 names as audit-required — are recorded with before/after values, actor, and (where applicable) a reason code, specifically so a permission escalation or an unusual override is reviewable, not just logged.
- **Audit is distinct from authorization.** A recorded audit entry does not retroactively make an action permitted — RBAC (Section 7) is the gate; audit is the record of what the gate allowed or an administrator overrode.

## 10. Secrets

Secrets in this system fall into two categories, both held outside application code and outside version control without exception:

- **JWT signing keys** — the single highest-blast-radius secret in the system, since a compromised signing key would invalidate the trust of every issued token at once (`04-cross-cutting/technology-decisions.md` Section 8's own named drawback). Key rotation must be possible without a full-system outage, and access to the signing key is the most tightly scoped credential in the platform.
- **External integration credentials** — the Payment Gateway, SMS/OTP Provider, Push Notification Service, and Geocoding/Maps Provider each require credentials, held only by the Backend module that owns that integration (`02-context/system-context.md` Section 5, Section 7) — Payment's gateway credentials are never reachable from, or duplicated into, any other module.

**Where secrets live** is a deployment-layer concern (managed secret storage provided by the Cloud Infrastructure named in `04-cross-cutting/technology-decisions.md` Section 10) and is detailed in `05-deployment/infrastructure-and-release.md`, not here. What this document establishes is the architectural rule: secrets are never embedded in code, never committed to the repository, and never shared more broadly than the single module that needs them to call its owned external integration.

## 11. Data Protection and Compliance Mapping

| Obligation | Source | Control |
|---|---|---|
| Saudi data residency | SRD; ADR-0003, ADR-0010 (Cloud Infrastructure) | PostgreSQL, Redis, and Object Storage all provisioned within a Saudi-compliant region (`04-cross-cutting/technology-decisions.md` Section 10); no durable business data or backup leaves that region. |
| PDPL — data minimization and retention | SRD; DDD Sections 11, 12 | Retention and purge rules defined per entity category in `04-cross-cutting/data-architecture.md` Sections 9, 14, 16; Rider Location specifically flagged for short-term retention and PDPL review (DDD Section 5.23). |
| PDPL — accountability for sensitive-data access | DDD Section 12 | Audit Log records administrative access to sensitive customer/worker data (Section 9 above), not just changes to it. |
| SAMA — payment handling integrity | SRD; ADR-0010, ADR-0015 | Payment state changes only on verified gateway confirmation, rider cash collection, or card-on-delivery record — never on client-supplied payment status (DDD Section 5.24's own constraint); monetary values stored as exact integer halala (ADR-0015), never floating point. |
| mada / card / BNPL processing | SRD; `02-context/system-context.md` | Payment Gateway integration owned exclusively by the Payment module; no other module or client ever holds gateway credentials or calls the gateway directly. |

This table maps obligations to the controls elsewhere in this document and in `data-architecture.md` that satisfy them — it does not introduce new controls beyond what Sections 3—10 already establish.

## 12. Open Decisions

- **Rider OTP/proof-of-delivery's exact interaction with the Delivery module's interface** (Section 4) is described at the architecture level only; whether it reuses Auth's `VerifyOtp` interface directly or a Delivery-owned wrapper around it is SDD-level detail.
- **Support's finer-grained ticket permissions** (distinguishing, for example, assigning a ticket from merely commenting on one, beyond the `support:read`/`support:manage`/`support:comment`/`support:escalate` catalogue in Section 7) may grow more granular as the Support module's SDD is written; the catalogue in Section 7 is the current authoritative baseline, not a ceiling.
- Token lifetimes, session storage, and the RBAC permission catalogue — previously open here — are resolved by ADR-0033 (Sections 6–7 above) and must not be treated as open going forward.
- Concrete OTP abuse-protection limits, step-up authentication for privileged roles, and store-scoped authorization — all previously unaddressed gaps identified by the 2026-07-30 Architecture Readiness Review — are resolved by ADR-0039, ADR-0040, and ADR-0041 (Sections 4, 4A, and 7 above) and must not be treated as open going forward.

