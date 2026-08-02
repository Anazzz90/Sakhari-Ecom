# Architecture Design Specification (ADS) — Sakhari Ecom

### The Single Source of Truth for the System's Architecture

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Master — the authoritative high-level description of the architecture. Stable in shape; tightened (not contradicted) as the detail documents in `02-context/` through `06-quality-attributes/` are authored — see `README.md` Section 5. |
| **Authority** | Subordinate only to `00-Architecture-Principles.md` (the Constitution). Binding on every detail document, every SDD, and every implementation decision below it. Where a detail document conflicts with this ADS, the conflict is resolved explicitly — by correcting the detail document or by updating this ADS — never left standing. |
| **Amendment** | Structural changes follow the same discipline as the Constitution's Section 13: recorded in `CHANGELOG.md`, and any change to a decision already backed by an ADR is made through a new ADR, not a silent edit. |

This document does not restate the Constitution. Every principle it relies on is referenced by number (e.g., "Principle 4.2") rather than re-explained — read `00-Architecture-Principles.md` first if a reference is unfamiliar. What follows is how those principles become an actual system.

---

## 1. Executive Summary

Sakhari Ecom is a quick-commerce grocery and essentials delivery platform for the Saudi Arabian market, designed to be built and operated by a single developer working with AI coding assistants. The architecture that follows from this reality is deliberately unglamorous: one backend service organized internally around business capabilities, one relational database as the system of record, a small set of well-understood supporting technologies, and four client applications that all depend on the same backend logic rather than reimplementing it.

The architecture makes a small number of consequential bets, each traceable to a Constitution principle: correctness of money and inventory state is never traded for speed (Principle 4.1); there is exactly one place durable business facts live (Principle 4.2); the system is decomposed by what the business does, not by technical convention (Principle 4.4); and complexity — a second service, a second datastore, a distributed pattern — is earned by evidence, not adopted in anticipation of scale the business does not yet have (Constitution Section 13). The result is intended to be a system that is fast enough to meet a ten-to-twenty-minute delivery promise, trustworthy enough to hold real payments and real inventory, and simple enough for one person to fully understand.

This document is the master description of that system: what it is, why it takes the shape it does, and which of that shape is settled versus still to be detailed. It is written for two audiences equally — the developer who owns this project, and the AI coding assistants who implement it without memory of how it came to be this way.

---

## 2. Business Context

The architecture exists to serve a specific, bounded business reality, established in the SRD and not re-litigated here:

- **Market and promise:** grocery and daily-essentials delivery in 10–20 minutes, launching in Saudi Arabia only, across multiple dark stores serving distinct neighborhoods (initially Sakaka) rather than a single-store MVP.
- **Operating model:** one developer, AI-assisted, with no dedicated operations, QA, or support function — every architectural choice is filtered through what one person can build, run, and recover from.
- **Scale target:** low thousands of orders per day at launch — a scale where correctness, clarity, and operating cost matter far more than theoretical throughput ceilings.
- **Regulatory environment:** SAMA and PDPL compliance, Saudi data residency, and mada as a required domestic payment method are non-negotiable, not aspirational.
- **Payment reality:** cash-on-delivery is a significant share of orders, alongside card and BNPL — the architecture cannot assume payment is confirmed before fulfillment the way a fully pre-paid model could.
- **Actors:** four distinct user populations with different needs and different trust levels — Customer, Picker, Rider, and Admin/Ops — served by four purpose-built client applications rather than one undifferentiated app.

Every subsequent section of this ADS traces back to one or more of these facts. Where this document makes an architectural choice that isn't a direct consequence of something above, that choice is flagged as a judgment call in the Architectural Decisions Summary (Section 8), not presented as if the business context demanded it.

---

## 3. System Vision

The Constitution's Section 2 (Vision) states what kind of system this architecture aims to produce: correct under real financial and inventory stakes, legible to a returning developer or a memoryless AI assistant, and able to absorb growth without a rewrite. This ADS is that vision made concrete — the specific system that satisfies it.

Concretely: the system this specification describes is a single backend, internally organized around business capabilities (Section 9), fronted by four independently-releasable clients (Section 5) that share one API and one set of business rules (Principle 4.6). It holds one authoritative record of the truth (Principle 4.2), reacts to what happens inside it asynchronously wherever the customer isn't waiting on the outcome (Principle 4.8), and treats every financially or operationally consequential action as something that must be provably correct and provably recorded (Principles 4.9, 4.10). It is sized for the business described in Section 2 above, not for a business Sakhari Ecom might become.

---

## 4. Architectural Goals

The Constitution's Section 5 sets the priority order this architecture optimizes for: correctness, maintainability by one person, compliance and security by default, operational simplicity, cost discipline at launch scale, delivery-appropriate latency, and evolvability without rewrite. This ADS pursues each of them through specific, identifiable structural choices rather than as an abstract aspiration:

- **Correctness** is pursued through a single system of record (Section 6), transactional protection of atomic operations, and an audit trail for consequential actions (Section 10).
- **Maintainability by one person** is pursued through a single deployable backend organized by capability (Section 9) rather than a distributed system requiring coordinated deployment and debugging across services.
- **Compliance and security by default** is pursued through centralizing authentication, authorization, and data protection in the backend rather than distributing trust decisions across four clients (Section 11).
- **Operational simplicity** is pursued through boring, managed infrastructure and a deliberate avoidance of premature distributed-systems complexity (Sections 12, 13).
- **Cost discipline** is pursued by sizing every technology decision (Section 8) to launch scale, not to a hypothetical future one.
- **Delivery-appropriate latency** is pursued by keeping the customer-facing path — browse, order, pay — synchronous and fast, while pushing everything the customer isn't waiting on into asynchronous handling (Section 6).
- **Evolvability** is pursued by making capability boundaries (Section 9) real enough, and data ownership (Principle 4.5) strict enough, that a capability could later be extracted into its own service without redesigning the boundary itself — see Section 15.

---

## 5. Platform Overview

The platform consists of four client applications sharing one backend and one system of record.

| Application | Serves | Nature |
|---|---|---|
| Customer Mobile App | Customers | Public mobile application; the primary customer ordering surface. |
| Customer Web App | Customers | Public web application; browse, order, and account management for customers who prefer or require the web. |
| Worker App | Pickers and Riders | Internal mobile application distributed to onboarded staff, not publicly marketed; a single app with mode selection rather than two separate apps, because pickers and riders share nearly all non-task infrastructure. |
| Admin/Ops Dashboard | Admins, Ops, Support | Internal web application for catalog, inventory, order, worker, store, and dispatch management, and reporting/analytics. |

Together, the Customer Mobile App and Customer Web App form the **Customer Platform**: two presentations of one product, sharing the same backend APIs, the same authentication flow, and the same business rules end to end (Principle 4.6). There is no scenario in this architecture where the two customer clients disagree about what a customer is allowed to do or what something costs.

Behind all four clients sits one backend, one relational database of record, an in-memory store for acceleration and coordination, and object storage for media and static assets. There is exactly one backend to reason about, one place business logic lives, and one place the truth of the system resides — the client applications differ in audience and presentation, not in the system they depend on.

---

## 6. System Context

At the architecture level, the system's boundary is defined by who and what it exchanges information with. The full, authoritative account of this boundary lives in `02-context/system-context.md`; this section is the summary every reader of the ADS needs without leaving this document.

**Internal actors**, each interacting through the client application built for them:

- **Customer** — browses, orders, pays, tracks delivery.
- **Picker** — receives and fulfills picking tasks inside a dark store.
- **Rider** — receives and fulfills delivery tasks between a dark store and a customer.
- **Admin/Ops** — manages catalog, inventory, workers, stores, dispatch, promotions, and reporting.

**External systems** the platform depends on:

- A **payment gateway** supporting mada (Saudi Arabia's domestic debit network), card payment, and BNPL, in addition to cash-on-delivery handled entirely within the platform's own order and fulfillment logic.
- An **SMS/OTP provider** capable of registering Saudi domestic sender identities, used for authentication and delivery notifications.
- **Cloud infrastructure services** (managed database, object storage, and supporting managed services) operating within a Saudi-compliant region.

The system's boundary is drawn deliberately narrow: it trusts its own backend as the sole arbiter of business rules (Principle 4.6), and treats every external system as a boundary requiring validation, not an extension of its own trust (Constitution Section 8, "error handling exists at boundaries").

---

## 7. High-Level Architecture

Described structurally, without reference to specific technologies (see Section 8 for those):

The system is a **single backend service, internally organized into modules aligned to business capabilities** (Principle 4.4), each of which exclusively owns its own persisted data (Principle 4.5) and exposes its behavior through a service layer that is the sole holder of business logic (Principle 4.6). Every client-facing entry point into that service layer is deliberately thin — translation only, no business decisions (Principle 4.7).

Two distinct modes of communication exist inside and around this backend, corresponding to two distinct kinds of outcome:

- **Synchronous, transactional operations** for anything the caller must know the true outcome of before receiving a response — placing an order, authorizing a payment, reserving inventory. These are protected by database transactions so the system never represents a partially-completed business operation as if it were whole (Principle 4.9).
- **Asynchronous events** for everything downstream of a completed operation that the caller does not need to wait on — notifying a customer, triggering dispatch, updating analytics (Principle 4.8). This keeps the customer-facing path fast and insulates it from the reliability of slower downstream concerns.

Underneath the service layer sits a single system of record (Section 9's data layer), supported by an acceleration layer that may cache or coordinate but never independently holds a business fact (Principles 4.2, 4.3). Above the service layer, every client-facing surface consumes the same versioned API contract (Principle 4.11), so that the platform's four independently-released clients can evolve without forcing a synchronized backend release.

Every action with financial, inventory, or lifecycle consequence produces a durable, append-only record as a byproduct of the transaction that performed it (Principle 4.10) — auditability is a property of the write path, not a separate system bolted alongside it.

This shape is intentionally a **modular monolith**, not a distributed system of independently deployed services. A single deployable unit is the correct answer to this project's actual constraints (Section 2): one developer cannot safely operate, debug, and coordinate deployment across many services at this scale, and the business does not yet generate the load that would justify the operational cost of doing so. The internal module boundaries are drawn with the same rigor a true service boundary would require (Principles 4.4, 4.5) specifically so that this choice remains reversible — see Section 15.

---

## 8. Architectural Decisions Summary

The following decisions are already effectively settled — either stated explicitly in the finalized SRD or made necessary by it — and are summarized here so the ADS is a complete picture of the system's shape. Each is a candidate for a formal ADR under `decisions/`; as of this writing, only the meta-decision to use ADRs at all (`0001`) has been recorded, and backfilling the entries below into their own ADRs is listed as outstanding work in `decisions/README.md`.

| Decision | Summary | Primary driver |
|---|---|---|
| Backend built in-house on a single Node.js framework, not a backend-as-a-service platform | A managed auth/backend platform (evaluated and rejected) could not satisfy Saudi data-residency requirements; an in-house backend keeps the system of record fully within a compliant region. | Section 2 (regulatory environment); Principle 4.2 |
| PostgreSQL via a managed relational database service, not a self-managed database server | A managed service removes an entire category of operational burden (patching, backup, failover) a solo developer should not carry manually. | Design Goal: operational simplicity |
| Domestic SMS/OTP provider, not a global default provider | The default global provider evaluated could not register Saudi domestic sender identities, which is required for reliable local delivery of OTP and notifications. | Section 2 (regulatory/operational environment) |
| Four client applications rather than one universal app or a fully separate app per role | Bundle size, permission trust, attack-surface isolation, and independent release cadence favor separating customer-facing surfaces from worker/admin surfaces; picker and rider stay combined because they share nearly all non-task infrastructure. | Design Goal: security by default; operational simplicity |
| Two customer-facing clients (mobile and web) sharing one business logic layer, never diverging | Prevents the two customer surfaces from silently disagreeing about price, availability, or eligibility. | Principle 4.6 |
| Modular monolith over microservices at launch | Distributed-systems complexity is not justified by current scale or team size; capability boundaries are drawn to keep this reversible. | Constitution Section 6, Section 13; ADS Section 7, 15 |
| Cash-on-delivery modeled as a first-class payment path, not an edge case | A significant share of orders are expected to be COD; order and payment state must represent an unconfirmed-payment order honestly rather than assuming pre-authorization. | Section 2 (payment reality) |

Where this ADS makes a judgment call not directly forced by the business context in Section 2 — such as the specific asynchronous-eventing approach or the specific caching strategy — that judgment is recorded in the relevant cross-cutting document once authored, and backed by its own ADR at that time, per the Constitution's Section 12 decision-making guidelines.

---

## 9. Technology Decisions

The finalized technology choices, and the principle or constraint each realizes. This is a statement of *what* was chosen and *why it fits*, not a configuration or implementation guide — that belongs to SDD documents and code.

| Layer | Choice | Why this fits the architecture |
|---|---|---|
| Backend framework | A single Node.js-based backend framework, structured internally into capability-aligned modules | Realizes Principle 4.4 (capability-aligned structure) and 4.6 (one place for business logic) without requiring a second language or runtime for a solo developer to maintain. |
| System of record | A managed relational database service | Realizes Principle 4.2 directly — relational transactions are what make Principle 4.9's atomicity guarantee possible, and a managed service satisfies the operational-simplicity goal (Section 4). |
| Acceleration / coordination store | An in-memory key-value store, used for caching and coordination only | Realizes Principle 4.3 — accelerates reads and coordinates cross-request state without ever becoming a second source of truth. |
| Media and asset storage | Object storage | Holds product imagery and other static assets outside the transactional database, keeping the system of record focused on business state rather than binary payloads. |
| Customer-facing mobile client | A cross-platform mobile framework, shared with the Worker App | One mobile technology for both customer and worker mobile surfaces reduces the maintenance surface for a solo developer, while the applications themselves remain fully separate per Section 5. |
| Customer-facing web client | A server-rendered/hybrid web framework | Serves the web half of the Customer Platform under the same business logic as the mobile client (Principle 4.6), with independent release cadence from it (Principle 4.11). |
| Admin/Ops client | A single-page web application framework | Purpose-built, staff-only, browser-based tooling — optimized for internal operational speed rather than public distribution. |
| SMS/OTP delivery | A provider capable of Saudi domestic sender registration | Required for authentication and delivery notifications to work reliably for the actual target market (Section 2), which a global default provider could not satisfy. |
| Payment processing | A gateway supporting mada, card, and BNPL, alongside in-platform cash-on-delivery handling | Covers the full payment reality described in Section 2, not just the pre-paid card case. |

Each of these choices is a candidate for its own ADR (Section 8); this table exists so the ADS remains a complete, standalone picture of the stack even before that backfilling work is done.

---

## 10. Module Overview

The backend is organized around the business capabilities evident from the finalized SRD and the ownership expectations established in the DDD. This is a **capability-level view**, appropriate to the ADS — the authoritative module/capability boundaries belong in `03-decomposition/capability-boundary-map.md` and are expected to refine, not contradict, the groupings below once that document is authored against the SRD and DDD together.

| Capability area | Responsibility |
|---|---|
| Identity & Access | Authentication and authorization for all four actor types; the single point of trust every client relies on rather than re-implementing. |
| Catalog & Inventory | Product data, per-store stock levels, and availability — the source of truth Ordering and Admin/Ops both depend on. |
| Ordering | Order creation, order lifecycle state, and the transactional core that ties a customer's request to inventory reservation and payment (Principle 4.9). |
| Pricing & Promotions | Price calculation, discounts, and promotional rules applied consistently regardless of which customer client initiated the order (Principle 4.6). |
| Payments | Payment authorization and capture across mada, card, BNPL, and cash-on-delivery, and the record of what was actually charged (Principle 4.10). |
| Dispatch & Fulfillment | Assignment and tracking of picking and delivery tasks to Pickers and Riders, store-scoped per the finalized worker-assignment model, including grouping eligible deliveries into a single rider trip where operationally efficient (ADR-0034) — an optimization within this same capability, never a new one. |
| Store & Operations Management | Store definitions, worker-to-store assignment, and the operational configuration Admin/Ops manages. |
| Notifications | Outbound customer and worker communication (order status, OTP, delivery updates), consuming events raised elsewhere rather than being called synchronously by them (Principle 4.8). |
| Admin, Reporting & Analytics | Cross-cutting visibility for Admin/Ops — built as an explicit read path over the capabilities above, never a shortcut into their live tables (Principle 4.5's reporting exception). |

Each capability area owns its own data exclusively (Principle 4.5); where one needs another's data, it goes through that capability's own interface, never around it. This grouping is deliberately coarse at the ADS level — precise module boundaries, and the SRD/DDD ownership rules they implement, are the subject of `03-decomposition/service-decomposition.md` and `capability-boundary-map.md`.

---

## 11. Data Philosophy

The system's data philosophy is the direct composition of Principles 4.1, 4.2, 4.3, 4.9, and 4.10: correctness is prioritized over performance on every path that touches money or inventory; PostgreSQL is the only durable source of business truth; Redis accelerates and coordinates but never becomes a second source of truth; atomic business operations are protected by transactions; and consequential actions leave a durable record.

At the architecture level, this means data flows in one direction of authority: a client request reaches the owning capability's service layer, which is the only code permitted to write to that capability's tables (Principle 4.5); anything the client reads may be served from a cache for speed, but anything the client changes is committed to PostgreSQL before the system considers it true. Retention and lifecycle handling for each entity follow the SRD state machines and the DDD's data-retention and ownership rules — the architecture's role is to make sure the mechanism for enforcing a lifecycle transition (a status field guarded by a transaction, not a client-trusted flag) matches the rigor that entity's business importance requires.

The full mechanics of persistence, caching strategy, and consistency handling belong to `04-cross-cutting/data-architecture.md`, once authored; this section is the philosophy that document must remain faithful to.

---

## 12. Security Philosophy

Security is treated as a default property of the architecture, not a layer added before launch (Constitution Section 11). Concretely, that means authentication and authorization are centralized in the backend's Identity & Access capability (Section 10) and never re-implemented independently by a client — a client enforces UI-level restrictions for its own usability, but the backend is the only party that actually decides whether an action is permitted, consistent with Principle 4.6.

Each of the four client applications is trusted only for the actor it serves: the Worker App carries no customer functionality, the Admin/Ops Dashboard is staff-only and never publicly distributed, and the Customer Platform's two clients carry only customer-relevant permissions. This separation exists specifically to bound the damage a compromise of any one client can do — an attacker who fully compromises the Customer Mobile App gains nothing toward picker, rider, or admin capability, because that capability was never present to compromise.

Data protection follows directly from Section 2's regulatory environment: personal and financial data is protected in transit and at rest, retained and residency-bound consistent with PDPL and SAMA, and every access to it by staff is itself an auditable action under Principle 4.10 — an admin viewing or altering sensitive customer data is not invisible to the system that let them do it.

The full control set — specific authentication mechanisms, authorization model, and compliance-to-control mapping — belongs to `04-cross-cutting/security-and-compliance.md`, once authored; this section is the philosophy that document must remain faithful to.

---

## 13. Scalability Philosophy

Scalability is approached through the Constitution's explicit tradeoff (Section 6): vertical scaling and simple caching are preferred before horizontal or distributed scaling, and complexity is earned by evidence rather than provisioned speculatively (Constitution Section 13). At launch scale — low thousands of orders per day across a handful of stores — a single, well-resourced backend instance backed by a managed database comfortably serves the platform; there is no scalability problem at this stage that a distributed architecture would solve and a well-indexed relational database would not.

What the architecture protects, deliberately, is the *option* to scale further without a rewrite: because capabilities are decomposed with real boundaries and strict data ownership (Principles 4.4, 4.5), a capability under disproportionate load later — most plausibly Ordering or Catalog & Inventory as store count grows — can be scaled independently, and eventually extracted into its own deployable unit, without redrawing the boundary itself. Growth in store count is handled as a data-scale question (more rows, more stores, more workers) within the existing structure, not as a trigger for new services, until evidence says otherwise.

---

## 14. Reliability Philosophy

Reliability follows directly from where the architecture places its guarantees. On the transactional core — order placement, payment, inventory reservation — the system guarantees strict correctness through database transactions (Principle 4.9): these paths are allowed to fail loudly and visibly, but never to silently produce an inconsistent state. On the asynchronous side (Principle 4.8) — notifications, analytics, downstream side effects — the system accepts graceful degradation: a delayed or retried notification is a recoverable inconvenience, not a business-integrity failure, and is treated accordingly.

This asymmetry is deliberate: it concentrates reliability engineering effort exactly where the business stakes are highest (money and inventory) and spends less of it where the consequence of a transient failure is genuinely minor. A single-region deployment, appropriate at this scale (Section 13), is made safe by disciplined backup and recovery practice rather than by multi-region redundancy the business does not yet need — detailed in `05-deployment/infrastructure-and-release.md` once authored. Auditability (Principle 4.10) is itself a reliability mechanism: when something does go wrong, the system's own record of what happened is what allows a solo developer to reconstruct and resolve it without guesswork.

---

## 15. Future Evolution Strategy

This ADS describes the architecture as it stands, sized to the business context in Section 2 — not the architecture Sakhari Ecom might eventually need. The Constitution's Section 13 states the general philosophy (complexity is earned, principles are stable while their application evolves, changes are recorded explicitly); this section states what that means specifically for the system described above.

The modular monolith described in Section 7 is a starting point, not a ceiling. Because module boundaries in Section 10 are already drawn along real business capabilities with enforced data ownership (Principles 4.4, 4.5), the most likely evolution path is **extraction, not redesign**: a capability under real, evidenced load (most plausibly Ordering, Catalog & Inventory, or Notifications as order volume or store count grows) can be pulled out into its own deployable service using the boundary that already exists, rather than requiring the system to be re-decomposed from scratch. The same logic applies to the data layer: Redis's role can expand, and read models can be added for reporting (Section 10's exception under Principle 4.5), without disturbing PostgreSQL's status as the sole system of record.

As the detail documents in `02-context/` through `06-quality-attributes/` are authored, this ADS is expected to be revisited and tightened — its summaries replaced with precise references to the now-complete detail documents, per Section 4 of the top-level `README.md`. That tightening is routine maintenance, not architectural change, and does not by itself require an ADR. A change to the substance of a decision described here — a different technology, a different module boundary, a move away from the modular monolith — does require one, following the Constitution's Section 12 decision-making guidelines, and must be reflected back into this document rather than left to diverge from what the system has actually become.

