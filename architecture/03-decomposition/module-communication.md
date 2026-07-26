# Module Communication — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. |
| **Stability** | Stable in rule, evolving in the specific dependency graph as modules and events are added. The *shape* of allowed communication (REST at the boundary, in-process interface calls and domain events internally, no cross-module data access) is as stable as ADR-0002/0009/0011, which it implements directly. |
| **Authority** | Subordinate to `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `02-Architecture-Decisions.md`, `03-System-Context.md`, `04-Technology-Stack.md`, and `03-decomposition/module-catalog.md` (05). This document does not redefine any module's ownership or responsibilities — it defines the rules for how ownership boundaries are crossed. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** `module-catalog.md` establishes *what* each of the fifteen backend modules owns and is responsible for. This document establishes *how* they are allowed to reach each other, and — just as importantly — how they are not. It is the enforcement layer for ADR-0009's "no cross-module repository access" rule: a boundary that can't be crossed incorrectly only if the correct ways to cross it are written down as precisely as the forbidden ones.

**Scope.** This document covers:
- Which forms of communication are allowed and forbidden, in general and per-module.
- How REST fits into the picture (client-facing only) versus how modules actually talk to each other internally (in-process calls, not REST).
- The full inter-module dependency graph, built from `module-catalog.md`'s per-module Dependencies and Forbidden Dependencies fields.
- How transactions behave when a business operation spans more than one module's owned data.
- How circular dependencies are prevented by design, not just by discipline.

This document does **not** cover: event payload structure, delivery guarantees, retry, or idempotency mechanics (`09-Event-Architecture.md`); database schema or persistence mechanics (`07-Data-Architecture.md`); authentication/authorization mechanics (`08-Security-Architecture.md`); or any implementation detail — no code, no method signatures, no class names. "Communication Examples" (Section 9) are narrative walkthroughs of who calls whom and in what order, not code.

**Intended audience.** Same as `module-catalog.md`: the project owner, an AI coding assistant about to add or change an inter-module call, and the author of any future module SDD, for whom this document's dependency graph is a boundary to design within, not a starting point to renegotiate.

**Cross-references.** Builds directly on `module-catalog.md`'s per-module Dependencies, Forbidden Dependencies, Public Interfaces, Published Events, and Consumed Events fields (Section 6 below aggregates them into one graph). Implements ADR-0002 (Modular Monolith), ADR-0009 (Module Ownership and No Cross-Module Repository Access), ADR-0010 (Transactional Checkout and Inventory Reservation), ADR-0011 (Event-Driven Asynchronous Side Effects), ADR-0017 (Versioned APIs), and Principles 4.4–4.9 and 4.11. Resolves one point `module-catalog.md` explicitly deferred here — see Section 7's note on Payment initiation.

---

## 2. Allowed Communication

Exactly four forms of communication exist in this system. Nothing outside these four is permitted:

1. **Client → Backend, via REST.** The only way any client (Customer Mobile App, Customer Web App, Worker App, Admin/Ops Dashboard) reaches the system, through the versioned API (ADR-0017). See Section 3.
2. **Module → Module, via public interface (synchronous, in-process call).** The only way one module invokes another's behavior directly. See Section 4.
3. **Module → Module, via domain event (asynchronous).** The only way a module reacts to something that happened elsewhere without being invoked directly. See Section 5.
4. **Module → External system.** Only by the specific module `03-System-Context.md` and `module-catalog.md` name as that integration's owner (Payment ↔ Payment Gateway; Auth and Notification ↔ SMS/OTP Provider; Notification ↔ Push Notification Service). No other module ever calls an external system directly.

## 3. Forbidden Communication

- **Direct cross-module data access of any kind** — no module queries, joins against, or writes to a table owned by another module, whether through a shared repository, a raw query, or a shared database connection used to reach outside its own tables. This is ADR-0009's rule and the Constitution's named anti-pattern (Section 10); it has no exception anywhere in this system.
- **Business logic in a controller.** A REST controller translates a request into a call on the owning module's public interface and translates the result into a response — nothing else (Principle 4.7).
- **A module calling a module listed in its own Forbidden Dependencies** (`module-catalog.md`, Section 4, per module) — these lists exist specifically to keep the dependency graph (Section 6) a one-directional graph, not a tangled one.
- **A synchronous dependency on Notification, Analytics, Audit, or Search for a business operation to complete**, with one narrow, explicitly-named exception: Auth calls Notification synchronously to confirm OTP dispatch (`module-catalog.md` Section 4.1), because that specific flow needs to know delivery was attempted before telling the customer to check their phone. No other synchronous dependency on these four modules exists anywhere in the system — see Section 6's "consumer tier."
- **A module exposing its own separate REST surface.** There is one versioned API surface for the whole Backend (ADR-0017); a module's public interface is reachable through that shared surface, never through an endpoint it stands up independently.
- **Modules calling each other over REST/HTTP.** REST is the boundary protocol between clients and the Backend, not the protocol modules use to talk to each other inside the one deployable — see Section 4.

## 4. REST Communication

All four clients consume one versioned REST API (ADR-0017), owned collectively by the Backend and routed internally to the correct module. A few rules keep this precise:

- **Endpoint-to-module mapping is 1:1 with public interfaces.** Every REST endpoint corresponds to exactly one operation named in a module's Public Interfaces list (`module-catalog.md`, Section 4). There is no REST endpoint that spans two modules' worth of logic — an operation that needs two modules' involvement is orchestrated by one module (see Section 7's checkout example), which exposes the combined operation as its own single interface.
- **Controllers are thin and stateless with respect to business logic** (Principle 4.7): they parse and validate the request shape, invoke the owning module's interface, and translate the result to a response. They hold no business rules — a controller never decides, for example, whether stock is available; it calls Inventory's interface and reports what Inventory decided.
- **REST is external-facing only.** Once a request is inside the Backend and routed to the correct module, every further step — that module calling another module, that module publishing an event — happens through Section 4/5's mechanisms, never by one module issuing an HTTP request to another. This is a direct consequence of the modular monolith (ADR-0002): modules share one process and one deployment, so there is no network boundary between them to put REST across, and pretending there is one would add latency and failure modes the architecture is specifically designed to avoid at this scale.

## 5. Internal Service Interfaces

Each module exposes exactly one public interface — the operations listed in `module-catalog.md`'s Public Interfaces field for that module. This is the *only* way another module (or a controller) may invoke its behavior.

- **In-process calls, not network calls.** Because every module lives inside the same deployable (ADR-0002), a call from Order to Inventory's `ReserveStock` is an ordinary in-process method invocation, not a network request — no serialization, retry, or timeout handling beyond what any function call already has.
- **One contract, two entry paths.** The same interface operation is invoked whether the caller is a REST controller translating an external request, or another module calling directly. This is what keeps controllers thin: there is no separate "internal API" with different rules than the "external API" — there is one interface per module, period.
- **The interface is the enforcement boundary for ADR-0009.** A module's actual data-access code (however it reads/writes its own tables) is never exposed as part of this public interface — only business operations are (`ReserveStock`, not `GetInventoryItemRow`). This is what makes "no cross-module repository access" a structural property of the interface design, not just a rule someone has to remember to follow.

## 6. Domain Events

Events are the asynchronous half of module communication (ADR-0011, Principle 4.8): a module publishes a fact about something that already happened to its own owned data, and any number of other modules may react, without the publisher knowing or caring who's listening.

- **A module may only publish events about entities it owns.** Order publishes `OrderPlaced` because Order owns Order; Inventory publishes `InventoryReserved` because Inventory owns Inventory Reservation. This mirrors Section 4's data-ownership rule exactly: just as a module is the only one that can *write* its own data, it is the only one that can *announce* something happened to it.
- **Publishers never wait on consumers**, and never know how many consumers exist. Adding a new consumer of an existing event (for example, a future Analytics subscription to an event only Notification currently consumes) never requires a change to the publishing module.
- **Naming convention:** `<Entity><PastTenseVerb>` — `OrderPlaced`, `InventoryReserved`, `PaymentAuthorized` — established in `module-catalog.md` and used consistently across every module's Published/Consumed Events fields.
- Full event mechanics — delivery guarantees, retry behavior, idempotency requirements, versioning, and the eventual authoritative list of every event in the system — are `09-Event-Architecture.md`'s subject. This document only establishes events as an allowed communication mode and the ownership rule governing who may publish what.

## 7. Dependency Rules

The dependency graph is built directly from every module's Dependencies and Forbidden Dependencies fields in `module-catalog.md`. It resolves into five layers, each of which may only depend downward, never upward or sideways into an unrelated layer:

| Layer | Modules | May depend on |
|---|---|---|
| **Foundation** | Auth, User, Store, Settings | Nothing (Auth depends on Notification only for OTP dispatch — see Section 3's named exception) |
| **Reference** | Catalog, Search | Foundation, plus Search → Catalog (event-driven, not synchronous) |
| **Commerce** | Cart, Inventory, Promotion | Foundation, Reference |
| **Transaction orchestration** | Order, Payment | Foundation, Reference, Commerce (Order is the orchestrator; Payment depends only on Order) |
| **Fulfillment** | Delivery | Foundation, Order |
| **Consumer tier** (no one depends on these; they depend on almost nothing) | Notification, Analytics, Audit | Nothing synchronously required of them; they consume events from every layer above |

**Why a layered, one-directional graph:** this is what makes the modular monolith safely extractable later (Constitution Section 13; `01-Architecture-Design-Specification.md` Section 15) — a module that only ever depends downward can be pulled into its own service without discovering a hidden upward dependency that turns into a distributed cycle. It is also what makes the system reasoning-friendly for a solo developer and an AI assistant with no memory of prior sessions: "can module X call module Y" is answered by table lookup, not by tracing the whole codebase.

**Resolving an open point from `module-catalog.md`:** Payment's entry in the catalog left open whether Order calls Payment directly to initiate a payment, or whether Payment reacts to a consumed `OrderPlaced` event instead. This document settles it, per the DDD's own Transaction Boundaries (Section 9.1, "Order Creation"), which lists payment *initialization* as a participating step in order creation's own atomicity expectations: **Order calls Payment's `InitiatePayment` synchronously**, as the next orchestration step after inventory reservation succeeds (Section 8's example walks through this in full). Final payment *confirmation* (the gateway's asynchronous callback/webhook result, DDD Section 9.3, a separate transaction boundary) arrives as a consumed event (`PaymentAuthorized`/`PaymentFailed`/`PaymentCaptured`) that Order reacts to afterward. `module-catalog.md`'s Payment and Order entries should be read with this resolution in mind.

## 8. Transaction Boundaries

Each module manages its own database transaction over its own owned data — **no transaction ever spans two modules' tables.** This is a direct consequence of ADR-0009 (a module's data is only ever written by that module) and is consistent with the DDD's own Transaction Boundaries (Section 9), which models "Order Creation" (9.1) and "Inventory Reservation" (9.2) as two separate transactional units, each with its own atomicity and rollback expectations, not one merged transaction spanning both modules' tables.

For a business operation that touches multiple modules' data — checkout being the clearest example — cross-module consistency is achieved by **orchestration with compensation**, not a distributed transaction:

1. The orchestrating module (Order, per ADR-0010's "checkout belongs to Order Module") performs its own step in its own transaction.
2. It calls the next module's interface synchronously and waits for a definitive success or failure.
3. If a later step fails, the orchestrating module performs an explicit compensating action on what it already committed (for example: if inventory reservation fails after the order record was created, Order cancels/rolls back the order record it just created, satisfying the DDD's own stated expectation that "no confirmed order should remain" if reservation fails) — rather than relying on a single ACID transaction to undo both steps at once.

This is a deliberate architectural choice, not a gap: at this system's scale, a small number of well-understood orchestration flows (checkout being the primary one) is far simpler to reason about and debug than distributed transaction machinery would be, and it keeps every module's persistence concerns fully local to itself — which is exactly what makes future module extraction (Section 7) viable. The trade-off is that a compensating rollback is not instantaneous the way a database `ROLLBACK` is — there is a brief window where an order record exists in a not-yet-confirmed state before a failed reservation triggers its cancellation. This window, and how it's surfaced (or hidden) from the customer, is left to the relevant module's SDD to detail; the architectural guarantee this document makes is only that the window is bounded, explicit, and always resolved to a consistent end state — never left ambiguous.

**Note:** this reconciles the tension `module-catalog.md`'s Open Decisions flagged between ADR-0010's exact wording ("a single PostgreSQL transaction encompassing order creation and inventory reservation") and the DDD's two-transaction model. This document treats the DDD's model — and the orchestration-with-compensation pattern above — as the operative one. ADR-0010's wording should be tightened by a future clarifying ADR to match; until then, this section is the authoritative description of how checkout's transactional behavior actually works.

## 9. Circular Dependency Prevention

The dependency graph in Section 7 is a directed acyclic graph (DAG) by design: every layer may only depend on the layers listed as available to it, never on a layer above it or beside it. Two mechanisms keep it that way:

- **Structural:** the consumer tier (Notification, Analytics, Audit) and the reference tier's Search module are deliberately built to have nothing depend on them synchronously — a module that only ever *receives* calls or *consumes* events cannot be part of a synchronous cycle, because nothing waits on it to do anything.
- **The event mechanism is the designed escape valve** for any relationship that would otherwise require an upward or sideways dependency. The clearest example: Search needs Catalog's product data to build its index, but Catalog must never depend on Search (Catalog is Reference-tier, Search must never become something Catalog waits on). The resolution is that Search *consumes* Catalog's published events (`ProductCreated`, `ProductUpdated`, `ProductDeactivated`) rather than Catalog calling into Search — the dependency exists, but it flows through an asynchronous, one-directional event rather than a synchronous call that could ever become a cycle.

**Enforcement today** is code review against this document's dependency table, applied the same way to human-written and AI-generated code (Constitution Principle 4.12) — there is no compiler-level boundary inside a single NestJS deployable that prevents an import across module boundaries by itself (`04-Technology-Stack.md`'s NestJS entry makes this same point: the framework gives structural vocabulary for the boundary, not a guarantee of it). **Future enforcement**, once the codebase is large enough to justify the tooling investment, is expected to be automated (a lint rule or module-boundary-checking tool) rather than relying on review discipline indefinitely — consistent with ADR-0009's own future-enforcement note.

## 10. Communication Examples

Narrative walkthroughs only — no code, no message payloads (those belong to `09-Event-Architecture.md`).

**Example 1 — Checkout (Place Order), the canonical cross-module orchestration:**
1. A client sends a REST request to place an order. The controller validates the request shape and calls Order's `PlaceOrder` interface.
2. Order calls Cart's `GetCart` to read the customer's current selection.
3. Order creates its own Order/Order Item records, in its own transaction, in a not-yet-confirmed state.
4. Order calls Inventory's `ReserveStock`, synchronously, for the items just recorded — a separate transaction, owned entirely by Inventory (Section 8).
5. If reservation fails, Order performs its compensating action (cancels the order it just created) and returns a failure to the client. The flow ends here.
6. If reservation succeeds, Order calls Payment's `InitiatePayment`, synchronously (Section 7's resolved open point).
7. Order publishes `OrderPlaced`. Notification and Analytics consume it asynchronously; nothing waits on their reaction.
8. Later, Payment's gateway integration resolves the payment and Payment publishes `PaymentAuthorized` (or `PaymentFailed`). Order consumes this event and updates order status accordingly — this step is fully decoupled in time from the original request/response cycle.
9. Once payment is confirmed, Order publishes `OrderReadyForFulfillment`. Delivery consumes it and begins picking-session orchestration, entirely outside Order's own transaction or request/response cycle.

**Example 2 — Product search, an event-driven read model staying in sync:**
1. An Admin updates a product's price through the Admin/Ops Dashboard. The request reaches Catalog's `UpdateProduct` interface.
2. Catalog commits the change in its own transaction and publishes `ProductUpdated`.
3. Search consumes the event asynchronously and updates its derived index. Catalog has no awareness this happened.
4. A customer later searches for that product. The request reaches Search's `SearchProducts` interface directly — Search answers from its own index, with no synchronous call to Catalog at query time.

**Example 3 — OTP login, the one named synchronous exception:**
1. A client requests a login code. The request reaches Auth's `RequestOtp` interface.
2. Auth calls Notification's `SendNotification` synchronously (Section 3's named exception) specifically to get a dispatch confirmation before telling the client to expect a code.
3. Notification calls the external SMS/OTP Provider (`03-System-Context.md`) and returns a dispatch result to Auth.
4. Auth publishes `OtpRequested` for audit purposes and returns success/failure to the client.

**Counter-example — what this document forbids:** a hypothetical "products frequently bought together" feature implemented by Catalog directly querying Order's tables for co-purchase patterns. This is forbidden outright (Section 3) regardless of how convenient the join would be — the correct design is a read model (owned by Analytics or Search, per Principle 4.5's reporting exception) built from consumed `OrderPlaced` events, exactly as Search's product index is built from consumed Catalog events in Example 2.

---

## 11. Trade-offs and Future Evolution

**Trade-off accepted now:** in-process synchronous calls between modules are fast, simple, and require no network-failure handling — a direct benefit of the modular monolith (ADR-0002) that a distributed system would not have. The cost is discipline: nothing prevents a developer, or an AI assistant generating code quickly, from importing across a module boundary the way a network boundary would prevent by construction. This document, `module-catalog.md`, and (once automated tooling exists) a boundary-checking lint rule are what stand in for the network boundary microservices would otherwise provide for free.

**Future evolution:** when a module is eventually extracted into its own independently deployed service (Constitution Section 13; `01-Architecture-Design-Specification.md` Section 15), the change is concentrated exactly where this document says communication happens:
- **Section 5's in-process interface calls** become network calls (the specific mechanism — REST, gRPC, or otherwise — is a future decision, not one this document makes). The *contract* — what operations exist and what they mean — does not need to change, if the boundary was respected as designed.
- **Section 6's domain events** already cross a conceptual boundary asynchronously; extracting a module changes their transport (an in-process event dispatcher today, a message broker later) but not their meaning or their publish/consume ownership rules. This transport question is explicitly `09-Event-Architecture.md`'s "Future Event Bus Strategy" to answer, not this document's.
- **Section 8's orchestration-with-compensation pattern** for cross-module consistency does not change at all on extraction — it was never a single-process transaction to begin with, so moving a participant to a different process changes nothing about how the pattern works.

This is the concrete payoff of every rule in this document: the discipline costs something today, and buys the option to change the deployment shape later without rewriting how modules relate to each other.

---

## 12. Open Decisions

- **Payment initiation vs. event-triggering** — resolved in this document (Section 7); `module-catalog.md`'s Payment and Order entries should be read in light of this resolution rather than their original, more tentative wording.
- **Audit's exact invocation mechanism** (direct synchronous call from every writing module, vs. broad event subscription) remains open, carried forward from `module-catalog.md` — this document establishes only that Audit must never block a business operation from completing regardless of which mechanism is chosen; the mechanism itself is `09-Event-Architecture.md`'s to settle.
- **Automated circular-dependency and boundary enforcement tooling** (Section 9) is named as a future direction, not a current commitment — no specific tool or timeline is decided here.
- The seven reconciliation points `module-catalog.md`'s own Open Decisions section raised (Support module's ownership, Refund/Picking/Workforce/Pricing folding, Search's lack of a backing ADR, the Auth/User split) are unaffected by this document and remain open at the same status.
