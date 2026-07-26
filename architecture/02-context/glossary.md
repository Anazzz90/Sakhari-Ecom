# Glossary — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. Scope expanded beyond this file's original "architecture-vocabulary-only" framing (see Section 1) into the project's single canonical terminology reference, spanning business, architecture, security, infrastructure, development, and principle-level terms in one place. |
| **Stability** | Evolving — grows as new terms enter the project; the definitions of already-listed terms are stable and change only when the document they're grounded in changes. |
| **Authority** | Subordinate to every document it defines terms from — this glossary never redefines a term, it restates and cross-references the definition each source document already gives, so that a reader who came here first still ends up with the same understanding as a reader who found the source document directly. Where a term is generic industry vocabulary with no single Sakhari Ecom source, this glossary is the source, and states so. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** Standardizes terminology across the entire project — one place where "what does this word mean *here*" has a single, authoritative answer, for a reader who may otherwise encounter the same word used slightly differently across the SRD, the DDD, the sixteen module documents, and everyday conversation about the project.

**Scope.** Business-domain terms (Order, Cart, Inventory, and the rest of the commerce and fulfillment vocabulary), architecture terms (Module, Aggregate, Bounded Context, and the structural vocabulary the `/architecture` folder relies on), security terms (JWT, RBAC, OTP, and the identity/authorization vocabulary), infrastructure terms (PostgreSQL, Redis, CDN, and the technology vocabulary), development terms (ULID, DTO, Idempotency, and the engineering-practice vocabulary), and the architecture-principle terms (Thin Controller, Data Ownership, Eventual Consistency) that appear throughout every cross-cutting document. A dedicated Section 4 collects project-specific terminology — words or names that mean something particular to Sakhari Ecom and would not be obvious to someone joining from outside it. Consistent with this task's instruction, entries describe *what a term means and where it's used*, not implementation detail (no schema, no code, no configuration syntax).

**Intended audience.** Deliberately broad — this is the one document in `/architecture` written for developers, architects, QA engineers, DevOps engineers, product managers, designers, and AI coding assistants alike. Where a term means something different to two of those audiences (a "Service" means something different to a backend developer than to a product manager describing "customer service"), this glossary states the Sakhari Ecom-specific meaning explicitly and flags the ambiguity rather than assuming everyone already knows which meaning applies.

**Cross-references.** Grounded in the SRD, the DDD, `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `02-context/system-context.md`, the full `03-decomposition/` set (`module-catalog.md`, `module-communication.md`, `capability-boundary-map.md`, `data-ownership-map.md`, `service-decomposition.md`), the `04-cross-cutting/` set, `07-coding-standards/coding-standards.md`, `08-ai-development/ai-development-rules.md`, and the ADR set in `decisions/`. Every entry below names the document its definition is grounded in rather than asserting authority of its own.

## 2. How to Read an Entry

Every term has a **Definition** (what it means, in plain language), **Context within Sakhari Ecom** (how it specifically applies to this project, with a source-document pointer), **Related Terms** (what to also read to avoid confusing this term with a neighbor), and, where genuinely useful, **Notes** (a disambiguation, a common point of confusion, or a caveat). Terms are organized alphabetically by first letter, in one continuous list — the categories in this task's own request (Business Domain, Architecture, Security, Infrastructure, Development, Architecture Principles) were used to make sure nothing was missed, not as the final organizing structure.

---

## 3. Terms A–W

### ADR (Architecture Decision Record)

**Definition:** A single, immutable document recording one significant, hard-to-reverse architectural decision — the context that forced it, the options considered, the choice made, and the consequences accepted.

**Context within Sakhari Ecom:** Every significant decision in this project (from the modular monolith itself to the choice of Moyasar as payment gateway) is recorded as a numbered ADR under `decisions/`, currently numbered 0001 through 0028. ADRs are never edited after being marked `Accepted` — a changed decision is a new ADR that supersedes the old one (`decisions/README.md`).

**Related Terms:** Source of Truth, Modular Monolith.

**Notes:** Do not confuse "ADR-worthy" with "worth documenting somewhere" — `decisions/README.md` is explicit that implementation details and easily-reversible choices do not get an ADR; they belong in code or an SDD instead.

### Aggregate

**Definition:** A cluster of related data that changes together and is only ever accessed or modified through one entry point, its Aggregate Root.

**Context within Sakhari Ecom:** `data-ownership-map.md` identifies each module's aggregates explicitly — for example, Order (with Order Item, Order Event, and Price Snapshot as dependent children) is the DDD's own named "central transactional aggregate" (DDD §6.4).

**Related Terms:** Aggregate Root, Entity, Module.

**Notes:** An aggregate is a data-modeling concept, smaller than a Module — a single module (Payment, for example) can own more than one aggregate (Payment itself, and separately Cash Remittance).

### Aggregate Root

**Definition:** The one entity within an aggregate that everything else in the aggregate is reached through — never accessed via its dependent children directly.

**Context within Sakhari Ecom:** Named per module in `data-ownership-map.md` §12 (for example, Inventory Item is the Aggregate Root for Inventory's stock-state aggregate; Support Ticket is the root for Support's ticket-and-comments aggregate).

**Related Terms:** Aggregate, Repository.

**Notes:** A Repository is scoped to an Aggregate Root — code never fetches a dependent child (an Order Item, a Support Ticket Comment) except through its root.

### API Version

**Definition:** An explicit marker on the Backend's API contract, incremented only for a breaking change; additive, backward-compatible changes never require one.

**Context within Sakhari Ecom:** Established by ADR-0017 ("Versioned APIs from day one") — every endpoint is versioned from the first release, applied uniformly across all four client applications, so no client is ever stranded by a change it didn't expect.

**Related Terms:** Public Interface, Module Communication.

### Aggregate Root

*(see above)*

### Authentication

**Definition:** Verifying that a caller is who they claim to be.

**Context within Sakhari Ecom:** Owned entirely by the Auth module, via OTP (never a password) plus a JWT/refresh-token session (`04-cross-cutting/security-and-compliance.md` Section 4; ADR-0026, ADR-0027).

**Related Terms:** Authorization, JWT, OTP, RBAC.

**Notes:** Do not conflate with Authorization — Authentication answers "who is this," Authorization answers "what are they allowed to do." Sakhari Ecom deliberately keeps these two questions owned by different parts of the system (Auth issues identity claims; RBAC evaluation, not Auth itself, decides permission).

### Authorization

**Definition:** Deciding what an already-authenticated caller is allowed to do.

**Context within Sakhari Ecom:** Evaluated centrally via RBAC and fine-grained Permissions on every request, never re-implemented per module and never trusted from a client's own claim about its role (`security-and-compliance.md` Section 7; ADR-0012).

**Related Terms:** Authentication, RBAC, Permission, Role.

### Bounded Context

**Definition:** A Domain-Driven Design pattern term for a boundary within which a particular domain model and its vocabulary apply consistently, and outside of which the same word may mean something different.

**Context within Sakhari Ecom:** Not used as a formal term anywhere in the project's own documents — the DDD (Database Design Document) does not use DDD-pattern terminology internally, and this project's actual realization of the concept is the Capability Boundary / Module pairing (`capability-boundary-map.md`, `module-catalog.md`).

**Related Terms:** Capability Boundary, Module.

**Notes:** Included here specifically because a reader arriving with prior Domain-Driven Design experience may look for this term. Treat "Capability Boundary" as this project's closest equivalent, not an exact synonym — Sakhari Ecom's module boundaries are drawn around business capabilities (Principle 4.4), which is a related but not identical framing to classic bounded-context modeling.

### Branch

**Definition:** A generic retail/commerce industry term for a physical retail location.

**Context within Sakhari Ecom:** **Not used.** Sakhari Ecom's own term for a fulfillment location is Store (specifically, a dark store — see Store).

**Related Terms:** Store, Warehouse.

**Notes:** Included specifically to prevent confusion — do not use "Branch" in code, documentation, or conversation about this project; use "Store."

### Business Capability

**Definition:** A named, business-meaningful thing the business does (ordering, inventory management, payment settlement) that maps to exactly one accountable module.

**Context within Sakhari Ecom:** Every one of the sixteen backend modules owns exactly one named Business Capability (`capability-boundary-map.md` §4, e.g., Order's is "Order Management & Checkout Orchestration").

**Related Terms:** Capability Boundary, Module, Bounded Context.

### Cache

**Definition:** A layer that holds a temporary, faster-to-read copy of data whose authoritative source lives elsewhere.

**Context within Sakhari Ecom:** Two cache layers exist: Redis (server-side, for authenticated/dynamic data) and the CDN (edge, for public static/media content) — `04-cross-cutting/reliability-and-performance.md` Section 4 explains why both exist rather than one.

**Related Terms:** Redis, CDN, Source of Truth.

**Notes:** A cache is never authoritative in this system — Principle 4.3 and `data-architecture.md` Section 3 are explicit that Redis (and by the same logic, the CDN) can be flushed or invalidated at any time with zero business-data consequence.

### Capability Boundary

**Definition:** The line drawn around a Business Capability, defining what's inside a module's accountability and what's explicitly not.

**Context within Sakhari Ecom:** Formally documented per module in `capability-boundary-map.md`, including each module's Explicit Non-Responsibilities and Forbidden Responsibilities — the boundary is a decision boundary, not just a data boundary (`capability-boundary-map.md` Section 2).

**Related Terms:** Business Capability, Module, Bounded Context, Data Ownership.

### Cart

**Definition:** A customer's in-progress, pre-checkout selection of items.

**Context within Sakhari Ecom:** Owned by the Cart module (DDD §5.16); explicitly reserves no inventory and commits to nothing — a "Transactional Draft" in the DDD's own words, isolated from every capability that only matters once Checkout begins (`module-catalog.md` §4.7).

**Related Terms:** Checkout, Order, Inventory, Reservation.

### CDN (Content Delivery Network)

**Definition:** An edge-caching network that serves public, cacheable content from locations close to the requester, rather than from the origin server on every request.

**Context within Sakhari Ecom:** Serves product imagery (from Object Storage) and Customer Web App static assets (`04-Technology-Stack.md` Section 11; ADR-0028). Never serves authenticated or dynamic per-user data.

**Related Terms:** Cache, Object Storage.

### Checkout

**Definition:** The end-to-end business process that turns a Cart into a committed Order — validating the cart, reserving inventory, initiating payment, applying promotions, and recording the outcome.

**Context within Sakhari Ecom:** Owned and orchestrated entirely by the Order module (ADR-0010, ADR-0021; `module-communication.md` Section 10's worked example) — Order calls Inventory and Payment as explicit, synchronous steps, each in its own module-local transaction, with compensation if a step fails.

**Related Terms:** Order, Cart, Reservation, Transaction Boundary.

### Customer

**Definition:** The actor who browses, orders, pays for, and receives delivery of products.

**Context within Sakhari Ecom:** One of four named actor types (alongside Picker, Rider, and Admin/Ops); reaches the system through the Customer Platform (Customer Mobile App + Customer Web App), which always shares one backend and one set of business rules (Principle 4.6).

**Related Terms:** Worker, Customer Platform (Section 4).

### Data Ownership

**Definition:** The rule that every persistent business entity has exactly one module that may write it, and that every other module reaches it only through that module's public interface, a consumed event, or a purpose-built read model.

**Context within Sakhari Ecom:** The foundational data rule of the whole system (Principle 4.5, ADR-0009), documented exhaustively per entity in `data-ownership-map.md`, including a complete ownership matrix (who may read, who may modify, by what access method).

**Related Terms:** Capability Boundary, Repository, Source of Truth.

### Delivery

**Definition:** The last-mile process of a Rider carrying a packed order from a Store to a Customer.

**Context within Sakhari Ecom:** Owned by the Delivery module together with in-store Picking (DDD Section 6.4/ADR-0020 folded Picking into Delivery's boundary) — Delivery Assignment and Rider Location are the entities involved (DDD §5.22–5.23).

**Related Terms:** Picking, Order, Worker.

### Distributed Lock

**Definition:** A coordination mechanism that lets only one concurrent process act on a given resource at a time, across multiple requests or processes.

**Context within Sakhari Ecom:** Implemented via Redis, used specifically around inventory reservation to prevent two customers from reserving the same last unit of stock (DDD §10.1, §10.6; `data-architecture.md` Section 7).

**Related Terms:** Redis, Reservation, Optimistic Locking.

**Notes:** A lock only coordinates who gets to *attempt* an operation first — the operation's actual result is still only real once committed in PostgreSQL (`data-architecture.md` Section 3).

### Domain Event

**Definition:** An immutable statement that something has already happened to an entity, published by the module that owns it, for any number of other modules to react to asynchronously.

**Context within Sakhari Ecom:** The system's only asynchronous communication mode (ADR-0011; `04-cross-cutting/integration-and-messaging.md`), following the `<Entity><PastTenseVerb>` naming convention (e.g., `OrderPlaced`) established in `module-catalog.md` and used consistently across all sixteen modules.

**Related Terms:** Eventual Consistency, Module, Public Interface.

### DTO (Data Transfer Object)

**Definition:** A plain data structure used to carry information across a boundary (a request, a response, an event payload), containing no business logic of its own.

**Context within Sakhari Ecom:** Implied throughout the REST API contracts (Public Interfaces) and event payloads described across `module-catalog.md` and `integration-and-messaging.md`, though the term itself is not formally used in any Sakhari Ecom document — it belongs to implementation, which this glossary and `07-coding-standards/coding-standards.md` both deliberately keep out of architecture documentation.

**Related Terms:** Public Interface, Domain Event, Entity, Value Object.

### Entity

**Definition:** A domain object with a persistent identity that continues to matter even as its other attributes change over time.

**Context within Sakhari Ecom:** Every row-level business object the DDD catalogs (Order, Product, Payment, and so on — DDD Section 4's Entity Catalog) is an Entity in this sense; `module-catalog.md` and `data-ownership-map.md` both use "entity" consistently in this meaning.

**Related Terms:** Aggregate, Aggregate Root, Value Object.

### Eventual Consistency

**Definition:** A consistency model where data is guaranteed to become correct across a system over time, but not necessarily at the exact instant a change happens.

**Context within Sakhari Ecom:** The consistency model for everything crossing a module boundary asynchronously (Domain Events) — `data-architecture.md` Section 6 is explicit that strong consistency applies only *within* one module's own data, never across modules.

**Related Terms:** Domain Event, Transaction Boundary, Source of Truth.

### Idempotency

**Definition:** The property that performing an operation more than once produces the same result as performing it once.

**Context within Sakhari Ecom:** A required property of every retry-sensitive workflow (order creation, payment confirmation, inventory reservation/release, notification sends, refund processing, rider assignment — DDD §10.4; `data-architecture.md` Section 13) and every domain event consumer (`integration-and-messaging.md` Section 9), because at-least-once delivery is the assumed baseline everywhere in this system.

**Related Terms:** Domain Event, Transaction.

### Inventory

**Definition:** The business capability of tracking how much stock exists and is available to sell.

**Context within Sakhari Ecom:** Owned by the Inventory module across three distinct entities — Inventory Item (current stock), Inventory Reservation (in-flight claims), and Inventory Ledger (historical movement) — deliberately kept separate rather than one mutable quantity field (`data-architecture.md` Section 6).

**Related Terms:** Reservation, Ledger, Store.

### JWT (JSON Web Token)

**Definition:** A signed token format that carries identity and claims, verifiable without a database lookup.

**Context within Sakhari Ecom:** The Backend's access-token format, issued by Auth after successful OTP verification, paired with a rotating Refresh Token for session renewal (ADR-0026; `security-and-compliance.md` Sections 5–6).

**Related Terms:** Refresh Token, Authentication, RBAC.

### Ledger

**Definition:** An append-only record of every movement of a quantity (stock, cash) and the reason for it, explaining how the current state came to be true.

**Context within Sakhari Ecom:** The Inventory Ledger (DDD §5.15) is the canonical example — immutable, append-only, and the mechanism that makes a stock discrepancy *explainable*, not just *visible* (`data-architecture.md` Section 8).

**Related Terms:** Inventory, Audit, Source of Truth.

**Notes:** Do not confuse a Ledger with an Audit Log — a Ledger explains quantity/financial movement; an Audit Log records who did what and why, for compliance and accountability (DDD §12.7 draws this distinction explicitly).

### Modular Monolith

**Definition:** A single deployable application, internally organized into strictly-bounded modules, as opposed to either an unstructured monolith (no internal boundaries) or microservices (many independently deployed services).

**Context within Sakhari Ecom:** The Backend's architecture (ADR-0002) — sixteen modules, one deployable, with module boundaries enforced as strictly as a network boundary would enforce them in a microservices architecture, specifically so extraction remains possible later without it being required now.

**Related Terms:** Module, Service, Aggregate.

**Notes:** This is the single most important term for an AI assistant to hold correctly (`08-ai-development/ai-development-rules.md` Section 12 names "assuming a microservices architecture" as a specific, named hallucination risk for this project) — module-to-module calls are in-process, never network calls.

### Module

**Definition:** A bounded unit of the Backend, owning a Business Capability and its data exclusively, exposing behavior only through its Public Interface.

**Context within Sakhari Ecom:** There are exactly sixteen: Auth, User, Store, Catalog, Search, Inventory, Cart, Order, Payment, Promotion, Delivery, Notification, Support, Analytics, Audit, Settings (ADR-0020; `module-catalog.md`).

**Related Terms:** Modular Monolith, Capability Boundary, Service, Aggregate.

### Object Storage

**Definition:** A storage service for binary files and media, addressed by key rather than by file-system path, distinct from a relational database.

**Context within Sakhari Ecom:** Holds product imagery and other media assets; PostgreSQL rows hold only a reference (key/URL) to the object, never the binary content itself (ADR-0016).

**Related Terms:** PostgreSQL, CDN.

### Optimistic Locking

**Definition:** A concurrency strategy that assumes conflicts are rare, allows an operation to proceed, and revalidates that the data hasn't changed before finally committing.

**Context within Sakhari Ecom:** Applied before committing checkout-adjacent workflows — order state, inventory availability, payment state, worker eligibility, promotion eligibility, and store status are all revalidated immediately before commit (DDD §10.3; `data-architecture.md` Section 13).

**Related Terms:** Distributed Lock, Transaction, Idempotency.

### Order

**Definition:** A customer's committed, confirmed request to purchase and receive specific items.

**Context within Sakhari Ecom:** Owned by the Order module — the DDD's own "central transactional aggregate" (DDD §6.4) — comprising Order, Order Item, Order Event, and Price Snapshot; Order orchestrates Checkout but does not own Inventory, Payment, Promotion, or Delivery data even while coordinating across all of them.

**Related Terms:** Checkout, Cart, Aggregate Root, Ledger.

### OTP (One-Time Password)

**Definition:** A short-lived, single-use code sent to a claimed identity (typically by SMS) to verify possession of that identity.

**Context within Sakhari Ecom:** The sole authentication mechanism for every actor — no passwords exist anywhere in this system (ADR-0027) — delivered via Unifonic (ADR-0023), and reused for rider delivery-confirmation as well as login (`security-and-compliance.md` Section 4).

**Related Terms:** Authentication, JWT.

### Permission

**Definition:** A single, named, fine-grained grant of capability (for example, "manage-catalog" or "issue-refund") that a Role may include.

**Context within Sakhari Ecom:** The unit RBAC evaluates on every request — deliberately fine-grained rather than a coarse role flag, so an Ops user and a Support user (both inside the Admin/Ops Dashboard) can have genuinely different capability (ADR-0012; `security-and-compliance.md` Section 7).

**Related Terms:** Role, RBAC, Authorization.

### Picking

**Definition:** The in-store process of gathering an order's items from shelves and packing them for handoff to a rider.

**Context within Sakhari Ecom:** Owned by the Delivery module (folded in per ADR-0020) as Picking Session and Picking Session Item (DDD §5.20–5.21); triggered by Order reaching a fulfillment-ready state, reported back to Order via events, never by writing to Order's own tables.

**Related Terms:** Delivery, Order, Worker.

### PostgreSQL

**Definition:** An open-source relational database management system.

**Context within Sakhari Ecom:** The sole system of record for every durable business fact in the system, provisioned via a managed service in AWS `me-central-2` (ADR-0003, ADR-0024) — no entity in this system is ever authoritative anywhere else.

**Related Terms:** Source of Truth, Redis, Entity.

### Promotion

**Definition:** A discount or offer rule applied to eligible orders, plus the codes and usage history associated with it.

**Context within Sakhari Ecom:** Owned by the Promotion module — evaluated and applied by Order during Checkout, never applied by Promotion itself, keeping discount rules reusable across every current and future path that needs one (`capability-boundary-map.md` §4.10).

**Related Terms:** Order, Checkout.

### Public Interface

**Definition:** The set of operations a module exposes for other modules and the API layer to call — the only door into a module from outside it.

**Context within Sakhari Ecom:** Named explicitly per module in `module-catalog.md`'s "Public Interfaces" field, and expanded into named Public Services per module in `service-decomposition.md` — nothing beneath this layer (Internal Services, Application Services, Domain Services, Repositories) is ever reachable from outside the module.

**Related Terms:** Module, Repository, Service.

### Queue

**Definition:** A mechanism for holding work to be processed asynchronously, typically in order and with some durability guarantee.

**Context within Sakhari Ecom:** Used conceptually for two things — in-process Domain Event dispatch and Redis-backed scheduled background jobs (reservation expiry, notification retry, reconciliation) — both built on infrastructure already in the stack rather than a dedicated queue technology (`04-cross-cutting/reliability-and-performance.md` Section 7).

**Related Terms:** Domain Event, Redis, Idempotency.

### RBAC (Role-Based Access Control)

**Definition:** An authorization model where permissions are granted to roles, and actors are granted roles, rather than permissions being assigned to individuals directly.

**Context within Sakhari Ecom:** The system's authorization model (ADR-0012), evaluated centrally by Auth on every request, using fine-grained Permissions grouped into Roles (Customer, Picker, Rider, Admin, Ops, Support).

**Related Terms:** Role, Permission, Authorization.

### Redis

**Definition:** An in-memory data store commonly used for caching, coordination, and short-lived state.

**Context within Sakhari Ecom:** Used exclusively for caching, sessions, rate limiting, and short-lived coordination (locks) — never as an authoritative store for any business fact (Principle 4.3, ADR-0004).

**Related Terms:** Cache, Distributed Lock, PostgreSQL.

### Refresh Token

**Definition:** A longer-lived credential used to obtain a new access token without requiring the user to fully re-authenticate.

**Context within Sakhari Ecom:** Issued alongside every JWT access token, rotated on each use, with reuse of an already-rotated token treated as a compromise signal that invalidates the entire token family (ADR-0026; `security-and-compliance.md` Section 6).

**Related Terms:** JWT, Authentication.

### Refund

**Definition:** Money or compensation owed back to a customer for a completed or partially completed order.

**Context within Sakhari Ecom:** Owned by the Payment module (folded in per ADR-0020) as a payment-reversal operation — Payment executes and records a refund; whether an order is *eligible* for one on business grounds is a currently open question between Order's own logic and Support's human-driven workflow (`capability-boundary-map.md` Section 8).

**Related Terms:** Payment, Order, Support.

### Repository

**Definition:** The data-access layer for a module's own owned tables — the only code permitted to read or write them directly.

**Context within Sakhari Ecom:** Named per module and per Aggregate Root in `service-decomposition.md` §8 (e.g., Order's OrderRepository) — never shared across modules, never called from outside its own module's Application and Domain Services (ADR-0009; `coding-standards.md` Sections 2–3).

**Related Terms:** Data Ownership, Aggregate Root, Service.

### Reservation

**Definition:** A temporary hold against available stock, created for an in-flight order and either consumed (stock actually deducted) or released (stock restored).

**Context within Sakhari Ecom:** Owned by the Inventory module as Inventory Reservation (DDD §5.14); created by Order calling Inventory synchronously as the step immediately following order-record creation (`module-communication.md` Section 7).

**Related Terms:** Inventory, Distributed Lock, Order.

### Role

**Definition:** A named collection of Permissions, assigned to an actor.

**Context within Sakhari Ecom:** Customer, Picker, Rider, Admin, Ops, and Support are the system's Roles (ADR-0012) — Admin, Ops, and Support are RBAC-differentiated roles within the single "Admin/Ops" actor category the SRD names.

**Related Terms:** Permission, RBAC, Authorization.

### Service

**Definition:** An overloaded term with three distinct, non-interchangeable meanings in software architecture generally.

**Context within Sakhari Ecom:** Specifically means one of: (1) a **Public/Internal/Application/Domain Service** — an internal code-organization layer within one Module (`service-decomposition.md`), never independently deployed; (2) a **microservice** — a pattern this project explicitly does **not** use (see Modular Monolith); or (3) informal business usage ("customer service," "the service a Worker provides") unrelated to either technical meaning.

**Related Terms:** Modular Monolith, Module, Public Interface, Repository.

**Notes:** This is the glossary's single most important disambiguation. When this project's own documents say "Service" without qualification, they almost always mean meaning (1) — an internal layer inside a Module, reachable only in-process, never over a network. An AI assistant should never infer meaning (2) from the word "service" appearing in this project's documentation.

### Service Ownership

**Definition:** The principle that a module's internal service layers (Application, Domain) and its Repository are used only by that module, never shared or called externally.

**Context within Sakhari Ecom:** The internal-structure counterpart to Data Ownership — `service-decomposition.md` Section 5 explains how this is enforced structurally, not just by convention: a Repository is simply never part of a module's Public Services.

**Related Terms:** Data Ownership, Repository, Public Interface, Module.

### Settings

**Definition:** Used generically for platform/application configuration; also the proper name of Sakhari Ecom's Settings module.

**Context within Sakhari Ecom:** Owns global and store-scoped configuration values that don't belong to any one business capability (`module-catalog.md` §4.16) — see Section 4 if disambiguating from a generic "settings" conversation.

**Related Terms:** Module, Store.

### Soft Delete

**Definition:** Marking a record inactive/deactivated rather than physically removing it, so historical references to it remain resolvable.

**Context within Sakhari Ecom:** Used for master/profile data (Customer Profile, Product, Store, and similar — `data-architecture.md` Section 9) — never used as a substitute for lifecycle state (a cancelled Order is a state, not a soft-deleted row) and never used for append-only/immutable records (a Ledger or Audit Log entry is never soft-deleted, because it's never deleted at all).

**Related Terms:** Entity, Ledger.

### Source of Truth

**Definition:** The one place a fact is authoritative — where, if any other copy of that fact disagrees, the source of truth is correct and the other copy is wrong.

**Context within Sakhari Ecom:** PostgreSQL, system-wide, for every durable business fact (Principle 4.2, ADR-0003) — Redis, the CDN, Search's index, and Analytics' rollups are all explicitly non-authoritative derived copies, never a source of truth for anything.

**Related Terms:** PostgreSQL, Data Ownership, Cache.

### Store

**Definition:** A physical fulfillment location that stocks and serves orders for the customers around it.

**Context within Sakhari Ecom:** Specifically a **dark store** at launch — a fulfillment-only location with no public storefront, serving a defined Service Zone (DDD §5.4–5.5). Owned by the Store module.

**Related Terms:** Branch (not used), Warehouse (not used), Inventory.

**Notes:** Sakhari Ecom uses "Store" for both the entity name and the informal term for the physical location — there is no separate "Branch" or "Warehouse" concept in this project (see those entries).

### Thin Controller

**Definition:** The principle that a request-handling entry point translates a request into a service call and the result back into a response, containing no business decision of its own.

**Context within Sakhari Ecom:** Principle 4.7, enforced structurally per module (`service-decomposition.md` Section 7) — a controller calls only a module's Public Service, never an Application Service, Domain Service, or Repository directly.

**Related Terms:** Public Interface, Business Capability (business logic belongs there, not in the controller).

### Transaction

**Definition:** A set of database operations executed as one atomic unit — either all of them succeed, or none of them take effect.

**Context within Sakhari Ecom:** Every multi-record write within a single module is protected by a PostgreSQL transaction (Principle 4.9, ADR-0010; `data-architecture.md` Section 5) — see Transaction Boundary for the rule about what a transaction is never allowed to span.

**Related Terms:** Transaction Boundary, PostgreSQL, Idempotency.

### Transaction Boundary

**Definition:** The rule defining what a single transaction is allowed to cover — in this project, exactly one module's own owned tables, never more.

**Context within Sakhari Ecom:** No transaction ever spans two modules' tables (ADR-0009, ADR-0021) — cross-module consistency (Checkout being the clearest example) is achieved through orchestration with explicit compensation instead, detailed fully in `module-communication.md` Section 8.

**Related Terms:** Transaction, Eventual Consistency, Data Ownership.

### ULID (Universally Unique Lexicographically Sortable Identifier)

**Definition:** An identifier format that is both non-sequential/non-guessable and sortable by creation time.

**Context within Sakhari Ecom:** The mandatory primary-key identifier format for every entity in the system (ADR-0019, restoring ADR-0013's original decision after a since-corrected DDD draft briefly specified UUIDs instead — see ADR-0018).

**Related Terms:** Entity, Aggregate Root.

**Notes:** `08-ai-development/ai-development-rules.md` Section 12 names assuming UUIDs instead of ULIDs as a specific, real hallucination risk for this project — always check ADR-0019, not general industry habit.

### Value Object

**Definition:** A domain object defined entirely by its attributes, with no persistent identity of its own — two value objects with the same attributes are considered equal, unlike two Entities.

**Context within Sakhari Ecom:** Not formally distinguished from Entity anywhere in the project's own documents — the DDD catalogs everything as an "Entity" without a separate value-object category. A Price Snapshot (DDD §5.32) is the closest conceptual example (an immutable, frozen record of values at a point in time), though the DDD does not label it this way.

**Related Terms:** Entity, Aggregate.

**Notes:** Included for readers with prior Domain-Driven Design background; do not assume Sakhari Ecom's own documents will use this term, since they don't.

### Warehouse

**Definition:** A generic supply-chain term for a large-scale storage facility, typically not customer-facing.

**Context within Sakhari Ecom:** **Not used.** Sakhari Ecom's dark stores serve as both storage and fulfillment point directly — there is no separate warehouse tier in the current architecture.

**Related Terms:** Store, Branch (not used).

**Notes:** Included specifically to prevent confusion — if a future capability introduces a genuine warehouse tier distinct from a store, that would be a new entity and a new ADR (`capability-boundary-map.md` Section 7's process), not an existing but undocumented concept.

### Worker

**Definition:** An umbrella term for a Picker or a Rider — staff who fulfill orders, as opposed to Admin/Ops staff who manage the platform.

**Context within Sakhari Ecom:** Worker profile, capability, shift, and availability data are all owned by the User module (ADR-0020); Workers reach the system through the Worker App, part of the Operations Platform (Section 4).

**Related Terms:** Customer, Picking, Delivery, Operations Platform (Section 4).

---

## 4. Project-Specific Terminology

Terms specific to Sakhari Ecom that a new developer, designer, or AI assistant would not know without reading this project's own documents first.

**Sakhari Ecom** — The project's name. An earlier prototype existed under the working name "RapidDash" (referenced in `SRD_RapidDash_v2.6_Saudi.md`, kept as historical reference); the production platform is Sakhari Ecom.

**The Constitution** — Informal name for `00-Architecture-Principles.md`, used throughout the architecture documentation set to refer to the twelve Core Architectural Principles and the engineering philosophy binding every other document.

**The ADS** — Informal name for `01-Architecture-Design-Specification.md`, the master, single-source-of-truth description of the architecture's actual shape.

**SRD** — Software Requirements Document (`SRD_Sakhari_Ecom_v2.6_Saudi.md`). Says what the product must do and under what business/regulatory constraints. The authoritative business-requirements source; architecture documents derive from it, never override it.

**DDD** — In this project, **Database Design Document**, not the general industry term "Domain-Driven Design." Describes entity ownership, relationships, retention, and integrity rules for the database layer (`DDD_Sakhari_Ecom_v1.0.md`). This distinction matters enough that it is stated explicitly in multiple architecture documents to prevent an AI assistant from assuming the general DDD meaning.

**SDD** — Software (or Software/System) Design Document, module-level, not yet created for any module. Sits between the architecture layer and implementation, detailing one module's internal design, schema, and API contracts.

**Customer Platform** — Collective name (from `01-Architecture-Design-Specification.md` Section 5) for the Customer Mobile App and Customer Web App together — two presentations of one product, never diverging in business behavior.

**Operations Platform** — Collective name (introduced in `02-context/system-context.md`) for the Worker App and Admin/Ops Dashboard together, mirroring "Customer Platform."

**Dark Store** — A fulfillment-only retail location with no public storefront, stocked and staffed to fulfill quick-commerce orders within a Service Zone. Sakhari Ecom's Store entity refers to this model at launch.

**Halala** — The Saudi Riyal's minor currency unit (1 SAR = 100 halala). Every monetary value in the system is stored as an integer count of halala, never a floating-point or decimal SAR value (ADR-0015).

**Moyasar** — The launch payment gateway for online card, mada, and Apple Pay payments (ADR-0022), owned exclusively by the Payment module. Not to be confused with Cash-on-Delivery or Card-on-Delivery, which are platform-managed payment paths that don't involve Moyasar.

**Unifonic** — The launch SMS/OTP provider (ADR-0023), owned exclusively by the Notification module (for delivery) and requested via the Auth module (for OTP dispatch).

**me-central-2** — The AWS region hosting Sakhari Ecom's infrastructure (ADR-0024), chosen specifically for Saudi data-residency compliance.

**mada** — Saudi Arabia's national domestic debit card network; one of the payment methods Moyasar processes.

**COD (Cash on Delivery)** — A payment path where the customer pays a rider in cash at handoff, reconciled by the Payment module as a Cash Remittance, distinct from Card-on-Delivery.

**Card-on-Delivery** — A payment path where the customer pays a rider via a physical POS/terminal at handoff, reconciled by the Payment module as a Card-on-Delivery Record — distinct from COD and from online card payment via Moyasar.

**Open Decisions** — A documentation convention used throughout `/architecture`: rather than silently resolving a gap or inconsistency discovered while authoring a document, it is recorded in that document's own "Open Decisions" section for deliberate, later resolution.

**Forbidden Dependencies / Forbidden Responsibilities / Forbidden Data Access** — Three related but distinct per-module "must never" lists, each from a different lens: Forbidden Dependencies (`module-catalog.md`) are modules a module must never call; Forbidden Responsibilities (`capability-boundary-map.md`) are business decisions a module must never make; Forbidden Data Access (`data-ownership-map.md`) is data a module must never read or write directly.

**Backfill (ADR backfill)** — The practice, used repeatedly in this project's history, of writing an ADR to formally record a decision that was already implicitly made (in the SRD, in prior architecture prose) but never given its own record — see `decisions/README.md`.
