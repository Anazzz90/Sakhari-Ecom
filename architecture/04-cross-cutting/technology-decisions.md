# Technology Decisions — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Stability** | Evolving in date/version detail; stable in the underlying reasoning. A technology being replaced later doesn't invalidate this document — it gets a new entry via the Future Replacement Strategy each technology already names. |
| **Status** | Authored — expands `01-Architecture-Design-Specification.md` Section 9 into full detail, and complements (never duplicates) `decisions/` where a dedicated ADR already exists. |
| **Authority** | Subordinate to `00-Architecture-Principles.md` and `01-Architecture-Design-Specification.md`. Where a technology already has a dedicated ADR, that ADR's Decision is authoritative; this document adds the deeper "why," alternatives, and lifecycle reasoning an ADR's concise format doesn't fully carry. |

## Purpose

Documents every technology choice actually in use across Sakhari Ecom's platform — not as a list of names and versions, but as a record of the architectural reasoning behind each one: what it's for, why it was chosen over real alternatives, what it costs, and how it could be replaced if it ever needs to be. Where `01-Architecture-Design-Specification.md` Section 9 gives a category-level summary and `decisions/` holds the individual decision records, this document is the connective layer that explains each technology in the context of the whole stack, so a reader — human or AI assistant — can understand not just *that* React Native was chosen, but *why it, and not the alternative, is the correct choice given everything else this system already is*.

## Relationship to other documents

- **`01-Architecture-Design-Specification.md`:** This document is the full expansion of the ADS's Section 9 (Technology Decisions), which deliberately stays at a generic, category level ("a cross-platform mobile framework") and names this document as the place each choice becomes concrete and is reasoned through in full.
- **`decisions/`:** Several technologies here already have a dedicated ADR (PostgreSQL — 0003; Redis — 0004; React Native — 0006; Next.js — 0007; React/Vite Admin Dashboard — 0008; Object Storage/S3 — 0016; primary-key identifiers — 0019; Moyasar — 0022; Unifonic — 0023; AWS `me-central-2` — 0024; NestJS — 0025; JWT — 0026; OTP — 0027; CDN — 0028). This document does not repeat those ADRs' Decision/Rationale; it cross-references them and adds material the ADR format doesn't carry well — a fuller alternatives comparison, and an explicit future-replacement strategy.
- **SRD:** Every technology named here traces to a finalized SRD decision (application structure, backend framework, database/cloud correction history) — this document does not introduce a technology the SRD gives no basis for.
- **DDD:** Where a technology choice is also a database-design convention (identifiers, timestamps, monetary storage), the DDD is the authoritative source for the convention itself; this document explains the architectural reasoning around adopting the datastore/mechanism that convention runs on.
- **`03-decomposition/` and `04-cross-cutting/data-architecture.md`, `security-and-compliance.md`, `integration-and-messaging.md`:** Each technology's role inside the system's structure, data rules, and security model is detailed in those documents; this document focuses on the technology itself, not the rules governing its use.
- **`05-deployment/`:** Explicitly out of scope here — how a technology is provisioned, scaled, or deployed belongs there. This document covers *what* was chosen and *why*, not *how it runs*.

---

## 1. React Native

**Role:** cross-platform mobile framework for the Customer Mobile App and the Worker App (Picker/Rider modes).

**Purpose:** enables both mobile client applications to be built once and run on iOS and Android, rather than maintaining four separate native codebases for one developer to own.

**Why chosen:** the SRD finalizes both mobile apps as React Native applications, superseding an earlier Flutter prototype (see ADR-0006). It keeps the mobile surfaces inside the same TypeScript/JavaScript ecosystem as the backend and web clients (see Section 4, "TypeScript-centered stack" reasoning threaded throughout this document), which matters more here than it would in a team setting: a solo developer benefits directly from not having to hold a second language's idioms, tooling, and failure modes in mind, and an AI coding assistant is more likely to generate correct code when the whole stack shares one type system and one set of conventions.

**Alternatives considered:**
- *Native development per platform (Swift/Kotlin)* — best access to platform capability, but quadruples the codebases (Customer iOS, Customer Android, Worker iOS, Worker Android) a single developer must maintain; rejected as unsustainable at this team size.
- *Flutter* — the project's own documented history: an earlier prototype was built in Flutter and explicitly superseded. Flutter's Dart language would have kept the mobile surfaces outside the shared TypeScript ecosystem the rest of the stack now uses.
- *A wrapped Progressive Web App* — avoids app-store distribution friction entirely, but was not seriously pursued: reliable background location for riders, camera/barcode-adjacent picking workflows, and push notification reliability are all weaker in a wrapped-web context than in a true mobile framework, and the Worker App's operational reliability requirements (Principle 4.1) don't tolerate that gap.

**Benefits:** one codebase per app across both platforms; shared TypeScript ecosystem and tooling with the rest of the stack; genuine code-sharing opportunity between the Customer Mobile App and Worker App for cross-cutting concerns (auth flow, networking, design-system primitives) without merging their application boundaries (Principle 4.6/4.7); a mature, widely-used framework with a large enough ecosystem that uncommon problems usually have a known solution.

**Drawbacks:** a JavaScript-bridge (or new-architecture) performance layer that, while generally sufficient for a CRUD-and-catalog-heavy commerce app, is a real gap against fully native for the most demanding UI work; dependency on native modules for cutting-edge platform capability (e.g., background-location precision for riders may eventually need a custom native module rather than a pure-JS solution); larger app bundles than a lean native build; the ecosystem's own churn (library and React Native version upgrades) is a recurring, if modest, maintenance cost.

**Future replacement strategy:** because business logic lives entirely in the Backend (Principle 4.6) and both React Native apps are thin presentation layers over a versioned API (ADR-0017), replacing React Native — with native-per-platform development or a different cross-platform framework — is a client-side rewrite bounded by that API, not a system-wide redesign. ADR-0006 names the trigger condition explicitly: a proven, evidenced React Native limitation blocking a committed requirement that no native-module bridge can solve. A replacement could proceed incrementally (React Native supports native module interop, allowing screen-by-screen migration) or as a parallel rewrite released as a new client version with no backend change required.

---

## 2. Next.js

**Role:** the framework for the Customer Web App, the web half of the Customer Platform.

**Purpose:** serves the public, customer-facing website — browse, cart, checkout, order history, profile, support, promotions — with rendering suited to a public, discoverability- and performance-sensitive storefront.

**Why chosen:** the SRD finalizes the Customer Web App as Next.js (ADR-0007). Public product and category pages benefit from server rendering for load performance and search-engine discoverability in a way an internal, always-authenticated tool (the Admin/Ops Dashboard, Section 3) does not — which is why the two web clients deliberately use different frameworks rather than one default across the whole system. Staying on React/TypeScript keeps the Customer Web App in the same ecosystem as the Customer Mobile App (Section 1), supporting shared component patterns and shared API-contract types, and directly serving Principle 4.6's requirement that the two Customer Platform clients never diverge in business behavior.

**Alternatives considered:**
- *A client-side-only single-page app*, the same category of tooling used for the Admin/Ops Dashboard (Section 3) — rejected for the public storefront specifically because it forfeits server-rendering's SEO and first-load performance benefits, which matter for a public, conversion-sensitive surface in a way they don't for an internal tool.
- *A non-JavaScript/TypeScript web stack* — rejected outright; it would break the shared-language, shared-type advantage every other technology decision in this document relies on, introducing a second ecosystem for a solo developer to maintain for no evidenced benefit.
- *A different React meta-framework* — not seriously pursued once Next.js's rendering flexibility (per-route SSR/SSG/ISR) was confirmed sufficient for the catalog-plus-checkout hybrid the storefront needs; no documented gap justified evaluating further.

**Benefits:** SEO discoverability for product and category pages; fast first paint for a conversion-sensitive public surface; shared component and type ecosystem with the Customer Mobile App and the Backend; flexible per-route rendering strategy, allowing static category pages to coexist with dynamic cart/checkout flows.

**Drawbacks:** introduces a running Node.js server process into the Customer Web App's runtime shape, not just static assets — a real addition to what must be operated, even if the specifics belong to `05-deployment/`; server-side data-fetching code must be held to the same discipline as any other client code — it must never become a second home for business logic that belongs in the Backend's service layer (Principle 4.6, ADR-0007's own caveat); framework version-upgrade churn, shared with the rest of the React ecosystem.

**Future replacement strategy:** because checkout and pricing logic stay backend-owned regardless of rendering framework, a future change is a web-client concern bounded by the same versioned API. ADR-0007 names its own reconsideration path: evidence that the storefront's actual traffic and SEO needs are better served by a different rendering strategy (for example, full static generation for specific pages) is evaluated as a refinement within the Next.js/React family first, before a full framework change is considered — minimizing churn unless the evidence clearly demands more.

---

## 3. React (Admin/Ops Dashboard, with Vite)

**Role:** the framework for the Admin/Ops Dashboard, the internal, staff-only half of the Operations Platform.

**Purpose:** serves catalog, inventory, order, worker, store, and dispatch management, and reporting/analytics, for Admin, Ops, and Support users — an authenticated-only, browser-based internal tool.

**Why chosen:** the SRD finalizes the Admin/Ops Dashboard as React with Vite (ADR-0008), deliberately distinct from the Next.js Customer Web App (Section 2). An internal, always-authenticated tool has no server-rendering or SEO requirement, so Next.js's server-rendering machinery would add operational weight the dashboard's use case doesn't need; a lighter single-page-app build optimizes instead for internal development speed, which matters more for a tool that changes frequently as operations needs evolve. Staying on React keeps the dashboard in the same component and TypeScript ecosystem as the rest of the stack even though it remains a fully separate client application (Principle 4.6/4.7).

**Alternatives considered:**
- *Next.js, for stack uniformity with the Customer Web App* — rejected specifically because the server-rendering overhead buys nothing for an internal, authenticated-only tool; uniformity for its own sake was not judged worth the operational cost.
- *A no-code/low-code admin panel generator* — rejected because such tools tend to place business logic and permission checks inside the generated tool itself rather than the Backend's service layer, directly violating Principle 4.6; it would also make the RBAC model (Section 8 below) harder to enforce centrally.
- *A different SPA framework (e.g., Vue, Angular)* — not pursued; it would break the shared-component/shared-type advantage with the rest of the React-based stack for no evidenced benefit.

**Benefits:** fast local development and build iteration (Vite), appropriate for a frequently-changing internal tool; component-pattern reuse potential with other React-based surfaces where genuinely useful; no server-rendering operational overhead; simplicity matched to an internal, always-authenticated audience.

**Drawbacks:** no code sharing with the Next.js-based Customer Web App beyond shared types and API-client code; no built-in SEO or first-load optimization — irrelevant for this audience, but a real, named absence rather than an oversight; as the dashboard's own feature surface grows, bundle-size and code-splitting discipline becomes a genuine, ongoing maintenance concern for a single-page app.

**Future replacement strategy:** ADR-0008 names its own trigger: the dashboard's complexity growing to a point where server-rendering or a more structured framework materially improves internal velocity, evaluated only against evidenced friction (slow builds, real developer pain), never preference. Any future swap — including a later move to Next.js if server-rendering benefits emerge for the dashboard — is bounded by the same versioned backend API and the same RBAC model (Section 8), so no backend change is required to change this client's framework.

---

## 4. NestJS

**Role:** the single backend framework hosting all business logic across every capability module, structured as a modular monolith (ADR-0002).

**Purpose:** provides the application framework and module system the Backend zone (`02-context/system-context.md` Section 5) is built on — the runtime that every client ultimately talks to, and the only place business logic is permitted to live (Principle 4.6).

**Why chosen:** NestJS's own module system gives structural vocabulary for exactly the boundary discipline the architecture already requires independent of any framework — capability-aligned decomposition (Principle 4.4) and strict per-module data ownership (Principle 4.5, enforced by ADR-0009). Rather than a solo developer inventing and manually enforcing that structure on top of a minimal framework, NestJS provides it as a first-class concept, backed by dependency injection that makes isolating service-layer business logic from thin controllers (Principle 4.7) a natural pattern rather than a discipline problem. It is TypeScript-first, keeping the backend inside the same language and type ecosystem as every client (Sections 1–3), and its ecosystem has mature, well-trodden support for the auth mechanisms this document covers next (JWT, Section 8; OTP, Section 9) and for database integration with PostgreSQL (Section 5).

**Alternatives considered:**
- *A minimal, unopinionated Node framework (Express or Fastify directly)* — more flexible, but would require a solo developer to invent and consistently enforce the same modular, dependency-injected structure NestJS provides out of the box; higher discipline burden, and more surface area for an AI coding assistant to generate structurally inconsistent code across different sessions with no memory of prior conventions.
- *A non-Node backend language or runtime* — rejected under the same reasoning that shaped every client-side choice above: it would break the shared-language, shared-type advantage across the whole stack for no evidenced benefit, and would cost a solo developer a second runtime's idioms and failure modes to hold in mind.
- *A serverless-functions-first backend* — rejected: it fragments the modular monolith's deliberate single-deployable boundary discipline (ADR-0002) into many independently deployed functions, reintroducing exactly the distributed-systems complexity (coordination, cold starts, cross-function transaction handling) ADR-0002 was written to avoid at this scale.

**Benefits:** a built-in module system that mirrors and helps enforce Principle 4.4/4.5's capability and ownership boundaries; dependency injection that makes Principle 4.6/4.7's service-layer/thin-controller split a natural default rather than a constant discipline exercise; strong end-to-end TypeScript support; a mature ecosystem covering auth, validation, and database integration, reducing how much custom infrastructure code a solo developer must write and maintain; a large enough community that an AI assistant is more likely to generate idiomatic, correct code against it.

**Drawbacks:** real learning-curve and ceremony cost (decorators, dependency-injection containers) relative to a minimal framework; NestJS's module system does not automatically enforce the "no cross-module repository access" rule (ADR-0009) — a shared NestJS module can still be misused to reach across a capability boundary if reviewers aren't deliberate about it, so the framework provides the vocabulary for the boundary, not a guarantee of it; ordinary framework upgrade-cycle maintenance, shared with the rest of the Node ecosystem.

**Future replacement strategy:** the module structure NestJS expresses today is an architectural requirement (Principles 4.4–4.7) independent of the framework — a future replacement, whether a different Node framework or the extraction of an individual module into its own service (ADR-0002/0009's designed reversibility), would preserve the capability-module boundaries themselves and change only the underlying framework or runtime executing them. NestJS is backed by ADR-0025; a future backend framework change would require a superseding ADR.

---

## 5. PostgreSQL

**Role:** the single system of record for all durable business state (ADR-0003), provisioned via Amazon RDS.

**Purpose:** holds every durable business fact — orders, inventory, payments, accounts, entity lifecycle state — as the one place the system's truth lives (Principle 4.2).

**Why chosen:** the domain is genuinely relational — customers, stores, inventory, orders, payments, and workers all relate to each other in ways a relational model expresses naturally, and the DDD's own Sections 3.8–3.9 arrive at the same conclusion independently. ACID transactions are what make ADR-0010's atomic-checkout guarantee possible at all; a datastore without strong transactional guarantees would have made that principle unenforceable rather than merely harder. Amazon RDS removes patching, backup, and failover from a solo developer's manual responsibility, and PostgreSQL's JSONB support gives exactly the bounded flexibility (bilingual product text, provider payload snapshots) the DDD scopes for it (Section 1.10) without becoming a substitute for relational modeling of core entities.

**Alternatives considered:**
- *A NoSQL/document store* — the DDD explicitly rejects this (Section 3.9): no SRD requirement justifies it, and it would weaken the natural relational structure across customer, store, inventory, order, and payment data for no compensating benefit.
- *A self-managed PostgreSQL server, instead of a managed RDS instance* — rejected on operational-simplicity grounds; patching, backup, and failover are exactly the kind of operational burden a solo developer shouldn't carry manually when a managed equivalent exists.
- *A different relational database (e.g., MySQL)* — not seriously evaluated; PostgreSQL's JSONB support, extension ecosystem, and RDS maturity were sufficient on their own merits, with no documented reason found to consider an alternative RDBMS.

**Benefits:** ACID transactions directly enable ADR-0010's correctness guarantee for checkout and inventory reservation; strong referential integrity prevents orphaned business records (DDD Section 1.5); JSONB provides flexibility precisely where the DDD scopes it appropriate, without letting schema-less data creep into core entities; RDS's managed operations meet the operational-simplicity design goal directly; a mature, thoroughly documented database that an AI coding assistant is more likely to generate correct schema and query code against (Constitution Section 3's "boring technology" conviction).

**Drawbacks:** every durable write must pass through PostgreSQL, bounding throughput below what a more distributed design could theoretically achieve — an accepted tradeoff, not an oversight (ADR-0003); vertical scaling has a real ceiling that will eventually require read replicas, partitioning (DDD Section 16.3), or other scaling mechanisms as real load materializes; RDS instance-class and storage costs require ongoing attention as usage grows, a discipline cost for a solo developer without a dedicated infrastructure function.

**Future replacement strategy:** ADR-0003 names its own reconsideration condition precisely: a specific, evidenced workload that PostgreSQL's own extensions genuinely cannot serve would introduce a supplementary, explicitly non-authoritative specialized store alongside PostgreSQL — not replace it. A full replacement of PostgreSQL as the system of record is treated as a one-way door requiring overwhelming evidence; the far more likely evolution path is read replicas, partitioning, or a future module extraction (ADR-0002/0009) giving that specific module room to introduce its own supplementary store, never a wholesale swap of the system of record itself.

---

## 6. Redis

**Role:** acceleration and coordination only — caching, sessions, rate limiting, short-lived locks and idempotency keys (ADR-0004) — never authoritative for a business fact.

**Purpose:** speeds up reads and provides cheap coordination primitives (most concretely, the locking mechanism ADR-0010's inventory reservation depends on) without adding load to PostgreSQL for every request.

**Why chosen:** Principle 4.3 and the DDD's Section 3.10 arrive at the same non-authoritative scoping independently, which is a strong signal this boundary is correctly drawn rather than an arbitrary architectural preference. Redis is the de facto standard for this role, with well-understood failure modes and deep ecosystem support inside the NestJS/Node world; it is specifically needed for the reservation-locking mechanism the DDD names directly (Section 10.1, "Redis Locking") and for geospatial assistance around rider location that a pure key-value cache wouldn't support as naturally.

**Alternatives considered:**
- *No in-memory store at all* — rejected under ADR-0004: it removes a low-risk, genuinely useful tool for no correctness benefit, and would push coordination needs (locks, rate limits) onto PostgreSQL in ways that would work against the delivery-appropriate-latency design goal.
- *Redis as a secondary, authoritative datastore for hot-path entities* — rejected outright, the "cache as database" failure mode both Principle 4.3 and the DDD explicitly warn against; a restart, eviction, or flush would silently lose data nobody thought to protect.
- *A different in-memory store (e.g., Memcached)* — not pursued; Redis's richer data structures (needed for locks, rate-limit counters, and geospatial rider-location assistance per DDD Section 3.10) aren't matched by a pure key-value cache, and no evidenced reason favored the simpler tool over Redis's broader capability.

**Benefits:** fast reads and cheap coordination without touching PostgreSQL for every request; safe to flush, resize, or restart at any time with zero business-data risk (ADR-0004); directly supports the locking mechanism ADR-0010's inventory-reservation guarantee depends on; geospatial data structures usable for rider-location assistance without standing up a separate specialized store.

**Drawbacks:** some read paths pay a PostgreSQL round-trip or a cache-then-verify cost that treating Redis as authoritative would avoid — an accepted tradeoff, not a gap; every new proposed use of Redis must be reviewed against "would losing this on a flush lose a business fact?" — an ongoing discipline cost rather than a one-time decision; a separate managed component to monitor and size correctly as load grows, even though it is far lower-maintenance than PostgreSQL itself.

**Future replacement strategy:** none anticipated for the non-authoritative boundary itself — Principle 4.3 and the DDD both make this a near-permanent rule (ADR-0004). The specific scope of what's cached or coordinated through Redis is expected to expand as real load patterns emerge; if a specific coordination need ever outgrows Redis's model, that need would be addressed with a narrowly-scoped new tool via a new ADR, without touching Redis's existing role for everything else it already does well.

---

## 7. S3 (Object Storage)

**Role:** storage for product imagery and other file/media assets, referenced — never embedded — from PostgreSQL rows (ADR-0016).

**Purpose:** holds binary media outside the transactional database, keeping PostgreSQL focused on business facts (Principle 4.2's scope) rather than binary payloads.

**Why chosen:** BLOB storage inside PostgreSQL would bloat the system-of-record database and slow the backups ADR-0003's operational-simplicity reasoning depends on; a managed object store is purpose-built for this instead. Using the same cloud provider already hosting PostgreSQL (Section 10) keeps the data-residency compliance surface to a single vendor relationship rather than several, which matters directly for the SRD's Saudi-residency requirements, and pairs naturally with a CDN (Section 11) for public asset delivery.

**Alternatives considered:**
- *BLOBs directly in PostgreSQL* — rejected under ADR-0016 for the reasons above: it works against Principle 4.2's intent and against the operational-simplicity goal by bloating backups.
- *Local or instance disk storage* — rejected outright; it does not survive instance replacement and is fundamentally incompatible with a managed-infrastructure, disaster-recoverable approach.
- *A different object storage provider, decoupled from the primary cloud provider* — not pursued; it would introduce a second vendor relationship and a second data-residency compliance surface to verify, with no evidenced benefit over using the same already-compliant provider hosting PostgreSQL.

**Benefits:** database backups stay fast and focused on actual business state; media can be served and cached independently (via a CDN, Section 11) without adding load to the database tier; durability and availability are handled by a managed service rather than custom infrastructure; storage scales independently of compute or database sizing.

**Drawbacks:** introduces a second place data effectively "lives" — facts in PostgreSQL, media in object storage — requiring explicit lifecycle discipline; an orphaned object with no referencing row, or a row referencing a deleted object, is a real class of bug this split creates (ADR-0016's own named consequence); every module that stores a reference to an object owns that reference's lifecycle as an ongoing responsibility, not a one-time setup task.

**Future replacement strategy: ** a standard, portable pattern — objects addressed by key or URL, with no business logic depending on the specific storage provider's proprietary API beyond upload/retrieve and provisioning. A future move to a different object storage provider is a data-migration exercise (copy objects, update stored references) rather than an architectural change, consistent with ADR-0016's own framing of this as a low-risk, well-understood pattern.

---

## 8. JWT (JSON Web Tokens)

**Role:** the token format representing an authenticated session across API calls from all four client applications, carrying the identity and role claims RBAC (ADR-0012) evaluates.

**Purpose:** lets the Backend verify who is calling and what they're allowed to do on every request, without requiring a server-side session-store lookup on the common path.

**Why chosen:** a single backend serves four independently-releasable clients (`02-context/system-context.md`); a stateless-ish, self-contained token avoids a Redis round-trip on every authenticated request, which matters given Section 6's deliberately narrow scoping of what Redis is used for. JWT is an industry-standard format with mature library support throughout the NestJS ecosystem (Section 4), and it naturally carries the role/permission claims the Identity & Access capability's centralized RBAC evaluation (ADR-0012) needs to check on every request, consistent with Principle 4.6's requirement that authorization decisions live in exactly one place.

**Alternatives considered:**
- *Opaque server-side session tokens, backed by a Redis session store* — simpler to revoke instantly (delete the session server-side), but requires a Redis lookup on every authenticated request, reintroducing exactly the "coordination store in the hot path" cost Section 6 is deliberately cautious about; rejected in favor of the lower-friction common case, accepting a harder revocation story as a named tradeoff below.
- *A third-party identity/auth platform issuing its own token format* — effectively rejected already at the requirements level: the SRD's own history records an earlier managed-auth direction superseded specifically over Saudi data-residency compliance, which is the same reasoning that shaped ADR-0003's in-house-backend decision.

**Benefits:** no session-store round-trip needed to verify identity claims on the common request path; a well-understood, widely-audited token format with mature tooling; efficiently carries the role/permission context RBAC evaluates (ADR-0012); works identically across all four clients with no client-specific authentication mechanism to design or maintain separately.

**Drawbacks:** JWTs are hard to revoke instantly by design — a token remains valid until it expires unless an additional mechanism (short expiry plus refresh-token rotation, or an explicit revocation list) is layered on top, which is a real design responsibility this choice creates rather than solves for free; token size grows with claim count, a minor overhead on every request; because a compromised signing key would invalidate every issued token at once, key management itself becomes a high-stakes, carefully-scoped concern under the Security Philosophy (`01-Architecture-Design-Specification.md` Section 12).

**Future replacement strategy:** JWT is an implementation detail sitting entirely behind the Identity & Access capability's interface (Principle 4.6); every client already reaches it only through the Backend's versioned auth flow (ADR-0017), so no client-side business logic depends on the token's internal structure — only on "send this token with each request." A future move to a different token format or session model is therefore a backend-only change plus a coordinated client update to the new token-handling contract, not a system-wide redesign. JWT session strategy is backed by ADR-0026; a future token/session change would require a superseding ADR.

---

## 9. OTP (One-Time Password)

**Role:** the authentication mechanism for customers and workers (phone-based login, no password), and the delivery-confirmation mechanism riders use with customers at handoff (DDD Section 7.5, "Rider OTP/proof validation").

**Purpose:** verifies identity at login and verifies successful delivery handoff, using the same phone-based mechanism for both rather than introducing a second one.

**Why chosen:** the SRD establishes a phone-first, Saudi market reality with a COD-heavy payment mix, where phone/OTP is a more natural fit than email/password for the customer and worker populations this platform actually serves. It also avoids the platform taking on responsibility for password storage, hashing, and reset flows — a category of security surface that OTP-based authentication sidesteps entirely. Reusing the same Unifonic SMS/OTP Provider integration (`02-context/system-context.md` Sections 5 and 7) for both authentication and delivery-confirmation, rather than standing up a second mechanism for the latter, keeps the number of distinct external-facing auth flows to secure and test to a minimum.

**Alternatives considered:**
- *Traditional email/password authentication* — rejected as a weaker fit for a phone-first customer base, and as taking on password-reset and credential-breach liability the platform has no need to carry when a phone-based alternative serves the same population better.
- *Social login only* — rejected: it does not cover the Worker App's staff-only population, who need centrally-provisioned identity, and it does not address delivery-confirmation at all.
- *A dedicated proof-of-delivery mechanism separate from OTP (e.g., photo capture only)* — not chosen as the primary mechanism; the DDD does not foreclose this as a future complement to OTP, but this document does not assert it as a current, committed integration.

**Benefits:** no password storage or reset liability; a natural fit for the target market's phone-first identity habits; one provider integration and one mechanism serving both authentication and delivery-confirmation, rather than two distinct flows to secure; every OTP request and verification is a natural, low-effort audit point (Principle 4.10).

**Drawbacks:** fully dependent on SMS delivery reliability from the external Unifonic SMS/OTP Provider — a provider outage or delay becomes an authentication or delivery-confirmation outage, not merely a degraded experience; SMS-based OTP carries known interception risks (SIM-swap and SS7-level attacks) that an app-based authenticator or hardware second factor would avoid — an accepted, named risk at this stage rather than an unconsidered one; a recurring per-message cost that scales with user and order volume, unlike a one-time implementation cost for a password-based system.

**Future replacement strategy:** OTP delivery is already abstracted behind the Identity & Access and Notifications capabilities' interfaces, and behind the Unifonic SMS/OTP Provider external integration named in `02-context/system-context.md`. A future move to an app-based authenticator, passkeys, or a different delivery channel is a backend-capability change plus a client update to the new verification flow — not a system-wide redesign. The provider itself can also be swapped independently of the OTP mechanism, since the Backend already treats every response from it as a validated, not-trusted-by-default external boundary (`02-context/system-context.md` Section 5's Trust Boundaries). OTP-first authentication is backed by ADR-0027; a future authentication mechanism change would require a superseding ADR.

---

## 10. Cloud Infrastructure (AWS `me-central-2`)

**Role:** the managed hosting substrate for PostgreSQL (RDS), object storage (S3), and supporting platform services, provisioned within a Saudi-compliant region (`02-context/system-context.md`'s `ext.cloud-provider`).

**Purpose:** provides the managed infrastructure the Backend and Infrastructure zones run on, satisfying Saudi data-residency requirements without a solo developer having to operate physical or self-managed infrastructure.

**Why chosen:** the SRD's own version history records that an earlier managed backend-as-a-service direction was superseded specifically because it could not satisfy Saudi data-residency requirements, correcting toward an AWS-hosted, in-house backend built on RDS. AWS `me-central-2`'s Saudi regional presence and compliance posture, combined with RDS and S3's operational maturity, satisfy the residency requirement and the operational-simplicity design goal at the same time. Using one provider for both the database (Section 5) and object storage (Section 7) keeps the compliance-verification surface to a single vendor relationship.

**Alternatives considered:**
- *A different major cloud provider (Azure, GCP)* — not documented as seriously evaluated; the SRD's own correction (v2.3) already anchored the decision on AWS once the initial BaaS direction was ruled out on residency grounds, and no evidenced reason to revisit that choice has since emerged.
- *Self-hosted or bare-metal infrastructure* — rejected outright as incompatible with the operational-simplicity design goal for a single developer with no dedicated infrastructure function.
- *A multi-cloud approach* — rejected as adding operational and compliance-verification surface with no evidenced benefit at launch scale, directly contrary to Constitution Section 13's "complexity is earned, not speculative" conviction.

**Benefits:** managed RDS and S3 remove patching, backup, and failover from manual responsibility; regional presence directly supports Saudi data-residency compliance; mature, thoroughly documented services that an AI coding assistant is more likely to help configure correctly (Constitution Section 3); a single vendor relationship simplifies compliance verification and operational overhead for a solo developer.

**Drawbacks:** real vendor lock-in risk — RDS and S3 are conceptually portable but carry provider-specific operational tooling and pricing models that make a full migration a genuine project, not a trivial swap; regional or provider-level outages are a dependency outside the platform's own control; cost visibility and optimization require ongoing, deliberate attention as usage scales, a discipline cost with no dedicated cloud-cost function to absorb it.

**Future replacement strategy:** explicitly deployment-level in its specifics, and out of scope for this document to design in detail (see `05-deployment/infrastructure-and-release.md`) — but architecturally, because PostgreSQL and S3 are both used through their standard interfaces (SQL; object key/URL) rather than deep provider-proprietary APIs, a future provider change is a data-migration and re-provisioning exercise, not a rewrite of business logic. This is nonetheless a one-way-door decision in practice, given real switching cost, and would only be revisited under significant evidenced pressure — a sustained compliance, cost, or reliability failure specific to the current provider — not a routine reconsideration.

---

## 11. CDN (Content Delivery Network)

**Role:** edge caching and delivery for public, cacheable assets — product imagery from Object Storage (Section 7), and static assets from the Customer Web App (Section 2).

**Purpose:** reduces latency and origin load for the platform's public-facing surfaces by serving cacheable content from edge locations rather than the origin on every request.

**Why chosen:** the Customer Web App and product imagery are both public, cacheable, latency-sensitive surfaces, and the platform's broader promise of a fast, responsive customer experience (alongside the 10–20 minute delivery promise itself) benefits directly from edge delivery rather than every request reaching the Backend or S3 origin directly. It also keeps those origins focused on their own duties rather than absorbing repeat, cacheable traffic.

**Alternatives considered:**
- *No CDN, serving all assets directly from S3 and the Next.js origin* — rejected as leaving avoidable latency on the table for a delivery-speed-sensitive product, and as unnecessarily loading the origins with repeat, cacheable requests a CDN would otherwise absorb.
- *A self-managed edge-caching layer* — rejected outright; this is a well-solved, commodity problem, and building a custom solution would add operational complexity a solo developer shouldn't take on where a managed offering already exists (Constitution Section 3's "boring technology" conviction).
- *A third-party CDN vendor, distinct from the cloud provider hosting the rest of the infrastructure* — not yet decided at the architecture level; this document defaults to assuming the cloud provider's native CDN offering for single-vendor simplicity (pairing naturally with Section 10) unless a future ADR finds a specific, evidenced reason to introduce a second vendor.

**Benefits:** lower latency for public, cacheable content without any change to business logic; reduces repeat load on S3 and the Next.js origin; a standard, well-understood technology with commodity-level operational risk rather than a novel one.

**Drawbacks:** requires cache-invalidation discipline whenever a product image or static asset changes — a stale CDN cache serving outdated content is a real, if low-severity, class of bug; adds one more piece of infrastructure to configure correctly (cache headers, invalidation triggers), a small but real addition to the deployment surface covered in `05-deployment/`; geographic edge coverage relevant to a Saudi-only launch must be verified rather than assumed, since not every CDN's edge network has equally strong regional presence.

**Future replacement strategy:** CDN choice is a delivery-layer optimization sitting in front of already-portable origins (S3 objects addressed by key; Next.js-rendered pages) — swapping CDN providers is a DNS and configuration change plus a cache-warm period, not a change to any business logic or data model. It can be revisited freely as pricing, performance, or regional-coverage evidence warrants, without an architectural redesign. CDN usage is backed by ADR-0028; a future CDN strategy change should be recorded if it changes provider class, cache responsibility, or deployment shape.

---

## Summary: ADR coverage

All major technology choices documented here now have accepted ADR coverage. Minor implementation choices such as exact library packages, SDK versions, token lifetimes, cache headers, and deployment tooling remain SDD or implementation-level decisions unless they change the architecture itself.

