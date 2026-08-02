# Event Architecture — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored — covers the asynchronous/event half of this file's original scope in full, including durable delivery (ADR-0029) and external-integration resilience (ADR-0032). **Not covered here:** synchronous API conventions (already substantially defined by ADR-0017 and `module-communication.md` Section 4, not repeated here). |
| **Stability** | Stable in mechanism (events are asynchronous, one-way, ownership-scoped, and durably queued — Principle 4.8, ADR-0011, ADR-0029); evolving in the specific event catalog as modules and features grow. |
| **Authority** | Subordinate to `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `decisions/README.md`, `02-context/system-context.md`, `04-cross-cutting/technology-decisions.md`, `03-decomposition/module-catalog.md`, `03-decomposition/module-communication.md`, and `04-cross-cutting/data-architecture.md`. Does not repeat their content — see Section 2. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** `module-communication.md` established that domain events are one of the system's four allowed communication modes and set the ownership rule (a module may only publish events about entities it owns) and naming convention. This document is where that gets fully specified: exactly how events are published and consumed, how background processing relates to events, how a business event becomes a customer- or worker-facing notification, how delivery failures are retried, how duplicate delivery is handled safely, how event schemas evolve, and how the current in-process event mechanism is expected to change if a module is ever extracted into its own service.

**Scope.** Domain events and everything downstream of them: naming, publish/consume rules, the transactional outbox that makes publication durable, background jobs, the notification flow, retry, idempotency, versioning, external-integration resilience, and future transport strategy. Does **not** cover: the ownership rule and naming convention themselves (already established in `module-communication.md` Sections 6 and restated only where necessary for continuity); per-module published/consumed event lists (the authoritative, growing list is `module-catalog.md` Section 4, per module — this document does not re-list them); synchronous REST/API conventions (ADR-0017, `module-communication.md` Section 4); the outbox table's storage/indexing mechanics (`data-architecture.md` Section 16); or any payload schema, queue technology, or code — no code, no message formats, no diagrams.

**Intended audience.** The project owner, an AI coding assistant implementing an event publisher or consumer, and the author of any future module SDD, for whom this document's rules on outbox durability, retry, idempotency, and versioning are non-negotiable defaults for every event in the system, not a per-event choice.

**Cross-references.** Builds on ADR-0011 (Event-Driven Asynchronous Side Effects), ADR-0029 (PostgreSQL Transactional Outbox for Domain Events), ADR-0032 (External Integration Resilience), ADR-0010 (Transactional Checkout, clarified by ADR-0021 — events are raised only after a module-local transaction commits), Principle 4.8, `module-catalog.md`'s per-module Published/Consumed Events fields, `module-communication.md` Sections 6—8 (events, dependency rules, transaction boundaries), and `data-architecture.md` Section 13 (idempotency and concurrency, including deadlock-avoidance ordering) and Section 16 (outbox table ownership) — this document applies that same discipline specifically to event consumption and external calls.

## 2. Domain Events

A domain event is an immutable statement that something already happened to an entity, raised by the module that owns that entity, after the transaction that made it true has committed (ADR-0011, ADR-0010). Events are the system's only asynchronous communication mode (`module-communication.md` Section 2) — everything else is either a synchronous interface call or a REST request from a client.

Three properties hold for every event in the system, without exception:

- **Ownership-scoped:** a module publishes only events about entities it owns (`module-communication.md` Section 6). Order publishes `OrderPlaced`; Inventory publishes `InventoryReserved`. No module ever publishes an event on another module's behalf.
- **Fact, not instruction:** an event states what happened (`OrderPlaced`), never what a consumer should do about it. A publisher has no expectation about who reacts, or how — that decision belongs entirely to each consumer.
- **Raised after commit, never before or instead of one:** an event is a byproduct of a completed, committed transaction (Section 5), never a substitute for one. Nothing about an entity's state depends on an event being published — if publishing failed after a commit, the entity's state is still correct; only the downstream reaction is delayed until the failure is resolved (Section 8).

## 3. Event Naming

The convention, established in `module-catalog.md` and `module-communication.md` and formalized here: **`<Entity><PastTenseVerb>`** — `OrderPlaced`, `InventoryReserved`, `PaymentAuthorized`, `DeliveryCompleted`. Three rules follow from it:

- The entity name in the event matches the owning module's entity name exactly (as used in `module-catalog.md`'s Data Ownership fields) — no synonyms or abbreviations that would make an event's owner ambiguous at a glance.
- The verb is always past tense, because an event describes something that already happened, never a command or a request (Section 2's "fact, not instruction" rule) — `ReserveInventory` is a synchronous interface operation (`module-communication.md` Section 5); `InventoryReserved` is the resulting event.
- A failure has its own named event, not a flag on the success event — `InventoryReservationFailed` exists alongside `InventoryReserved`, `PaymentFailed` alongside `PaymentAuthorized`, so a consumer that only cares about failures doesn't have to inspect a success event's payload to find out it wasn't one.

Delivery batching's events (ADR-0034) follow this same convention with `Batch` as the entity — `DeliveryBatchCreated`, `BatchRouteUpdated`, `BatchCompleted`, `BatchCancelled` — with one deliberate variant: `DriverAssignedToBatch` names the driver as the entity acted upon, mirroring the existing precedent of `AssignRider` (an interface operation) and `DeliveryAssigned` (its resulting event) already being about the *assignment relationship*, not just the assignee. This is the same pattern applied one level up, from an order-to-rider assignment to a rider-to-batch one — not a new naming scheme.

`DeliveryFailed` and `DeliveryRejected` (ADR-0036) are ordinary, regular applications of the convention — no variant needed. `DeliveryCompletedWithPayment` (ADR-0035) is a second deliberate variant, alongside `DriverAssignedToBatch`: it qualifies `DeliveryCompleted` with a compound suffix rather than inventing a wholly separate event name, because it is genuinely the same underlying fact (a delivery completed) with one additional, conditionally-present detail (a payment was collected) — a consumer already handling `DeliveryCompleted` is not required to also understand `DeliveryCompletedWithPayment` to stay correct, but Order specifically consumes both, since only the latter carries the collection detail it must forward to Payment.

## 4. Publishers

Per ADR-0029, a publisher does not raise an event as a separate, independently-failable step after its transaction commits — it writes the event as an **outbox record**, in the same module-local PostgreSQL transaction as the entity change the event describes (Section 5's rule that a transaction never spans two modules' tables is unaffected: the outbox table for a given event is owned by the same module that owns the entity, per `data-architecture.md` Section 16).

- A publisher writes its owned entity change and the corresponding outbox record together, inside one transaction. Either both are committed or neither is — there is no window where a fact is true in the entity's table but the event announcing it was never durably recorded, and no window where an event is durably recorded for a fact that was then rolled back.
- A publisher never waits for any consumer's reaction, never receives a return value from publishing beyond the outbox write succeeding as part of its own transaction, and never learns which or how many consumers exist.
- Once the transaction commits, the fact of "this event needs to be delivered" already survives a process crash, a deploy, or a restart — durability comes from the outbox row in PostgreSQL, not from the publishing process staying alive. A background dispatcher (Section 8) is responsible for reading pending outbox rows and getting them to consumers; it, not the original publisher, owns retrying that delivery.

## 5. Consumers

- A consumer subscribes to specific named events from specific modules — there is no "subscribe to everything" pattern except where a module's job genuinely requires broad visibility (Notification and Analytics, per their entries in `module-catalog.md`).
- A consumer must be **idempotent** with respect to the events it handles (Section 9) — receiving the same event twice must never produce a different or duplicated effect than receiving it once.
- A consumer never assumes strict cross-event ordering unless this document or a future event-specific note says otherwise. Where ordering matters within one entity's lifecycle (for example, `OrderPlaced` before `OrderReadyForFulfillment` for the same order), the consuming module's own logic must tolerate or correctly sequence events referencing the same entity, rather than assuming the transport guarantees delivery order.
- A consumer's failure to process an event must never be silently swallowed — it either succeeds, or it is retried (Section 8), or it surfaces as a recorded failure. A "silent failure of consequential actions" (Constitution Section 10's named anti-pattern) applies to event consumption exactly as it applies to any other operation.

**Resolving the Audit invocation question, carried forward from `module-communication.md` and `data-architecture.md`:** Audit's push model (DDD Section 12.1) works through **both** mechanisms, applied to different cases —
- For actions that already produce a domain event for other reasons (an order placed, a payment authorized, a permission changed as part of a settings update), **Audit consumes the same event other modules consume** — no separate "audit event" is published in parallel; the existing event is Audit's input.
- For actions with no other reason to publish an event (most notably, an administrator *viewing* sensitive data, which changes nothing and so has no natural domain event), **the acting module calls Audit's `RecordAuditEntry` interface directly and synchronously**, since there's no event to piggyback on.

Both paths honor the same guarantee: audit recording never blocks or fails the underlying business operation (`module-catalog.md` Section 4.14's Boundaries) — the direct-call path is fire-and-forget with respect to the caller's own success (a failed audit write is itself surfaced as an operational failure to investigate, never as a reason to fail the action it was recording).

## 6. Background Processing

Not every asynchronous process in the system is triggered by a domain event — some are scheduled, sweeping over data on a cadence rather than reacting to a specific fact. The DDD names eight (Section 15), each mapped here to how it relates to the event system:

| Background job | Trigger | Relationship to events |
|---|---|---|
| Reservation Expiry | Scheduled sweep | Finds reservations past their checkout/substitution window and releases them (`data-architecture.md` Section 7); publishes `InventoryReservationReleased` as a normal event once acted on — the *trigger* is a schedule, but the *outcome* is announced the same way any other Inventory change is. |
| Batch Formation Evaluation (ADR-0034) | Scheduled sweep, short interval, per store | Evaluates delivery-batching eligibility among packed/awaiting-rider orders (DDD Section 15.8); publishes `DeliveryBatchCreated` once eligible orders are grouped — the *trigger* is a schedule, but never one that delays an order past its own individual rider-assignment timeout (SRD BR-020); an order that doesn't find a batch partner in time is assigned individually, exactly as it is today. |
| Report Generation / Rollup Generation | Scheduled | Consumes historical events and records to build Analytics' rebuildable rollups (`module-catalog.md` Section 4.13); publishes nothing (Analytics is a pure consumer). |
| Notification Retry | Scheduled, informed by delivery-status events | Retries failed sends where safe (Section 8); not itself event-triggered, but reads the same delivery-status data Notification's own consumed events populate. |
| Inventory Reconciliation | Scheduled | A data-recovery mechanism (`data-architecture.md` Section 14), not itself event-driven; approved adjustments it produces generate ordinary Inventory Ledger events. |
| Shift Reconciliation | Scheduled, tied to shift-end state | Similarly a scheduled sweep; its outcomes (closure, discrepancy flags) are recorded through Payment/Audit's normal mechanisms, not a special event type. |
| Cleanup | Scheduled | Purges genuinely disposable data (`data-architecture.md` Section 9) — expired drafts, old notification metadata, retention-window-exceeded location history; not event-triggered and does not itself publish domain events, since purging carries no business fact worth announcing. |

**Why the distinction matters:** a background job is a *when* mechanism (a schedule), while an event is a *what happened* mechanism (a fact). Conflating them — for example, expecting a reservation to expire the instant its window closes, event-style — would be a mistake; expiry is only ever as prompt as the sweep's own cadence, which is an operational parameter (`05-deployment/infrastructure-and-release.md`'s eventual concern), not an architectural guarantee this document makes.

## 7. Notification Flow

The complete path from a business event to a customer or worker actually receiving a message, consolidating what `module-catalog.md` and `module-communication.md`'s Example 3 each partially describe:

1. A module publishes a domain event as a normal consequence of its own committed transaction — `OrderPlaced`, `OrderStatusChanged`, `PaymentAuthorized`, `PaymentFailed`, `DeliveryAssigned`, `DeliveryCompleted`, and others, per each module's Published Events list.
2. Notification consumes the event. It does not judge whether the event is "notification-worthy" beyond a fixed mapping of event type to message template and channel — that business judgment is expressed by which events exist and what they mean, not re-decided inside Notification (`module-catalog.md` Section 4.12's Boundaries).
3. Notification calls the appropriate external channel — the SMS/OTP Provider or the Push Notification Service (`02-context/system-context.md` Section 5) — and records the attempt, per DDD Section 14.5's "Notification History."
4. Delivery status (sent, delivered, failed) is recorded against that attempt. A failure enters the retry path (Section 8); it never causes the original business event to be re-raised or the originating module to be informed that notification failed — business state must never depend on notification delivery succeeding (DDD Section 5.33's own constraint, restated here because it is the load-bearing rule of this whole flow).
5. Where delivery is confirmed via the external channel's own callback, that confirmation is itself validated before being trusted (`security-and-compliance.md` Section 8's symmetric-trust rule applied to Notification specifically).

**One exception to the "consume, don't get called" pattern:** OTP dispatch (`security-and-compliance.md` Section 4) is the one case where Auth calls Notification synchronously rather than Notification reacting to an event — because that specific flow needs a dispatch confirmation before the client is told to expect a code, which an asynchronous, unconfirmed event cannot provide in time.

## 8. Retry Strategy

Two distinct things can fail and are retried differently:

- **Event dispatch failing** (the outbox dispatcher cannot reach a consumer, or a consumer raises an error) is retried by the dispatcher with exponential backoff, against the durable outbox record — never against the publisher's own transaction, which already committed and is not re-run. Per ADR-0029, the dispatcher records each delivery attempt (attempt count, last error, next-retry time) against the outbox row and marks the event processed only once every subscribed consumer has handled it successfully. The publisher itself has nothing left to retry once its transaction committed — Section 4's outbox write already guarantees the fact will eventually be announced; this is not optional or best-effort.
- **Event consumption/handling failing** (a consumer's reaction to an event throws, times out, or otherwise doesn't complete) is retried with backoff by the dispatcher on that consumer's behalf, up to a bounded number of attempts appropriate to that consumer's own workflow (DDD Section 10.5's "failed workflows should move to explicit failure/pending states rather than silently repeating irreversible actions" applies directly here). A consumer that exhausts its retries surfaces the event as a recorded failure — visible to Notification's own delivery-status history (Section 7) where relevant, or to Audit where the failure itself is consequential — never a silent drop. Per ADR-0031, an outbox backlog where the oldest pending checkout/fulfillment/payment event exceeds 5 minutes is a High operational alert, and 15 minutes is Critical.
- **Cross-instance behavior:** because pending work lives in PostgreSQL, not in any one backend process's memory, multiple horizontally-scaled backend instances (ADR-0030) can run the dispatcher concurrently without duplicating delivery — a dispatcher instance claims a pending outbox row (e.g., via a `SELECT ... FOR UPDATE SKIP LOCKED`-style claim, an SDD-level mechanism) before processing it, so two instances never both dispatch the same event at the same time. If an instance crashes mid-dispatch, an unclaimed or timed-out claim is picked up by another instance or the same instance on restart — the event is never lost, only possibly delivered again, which Section 9's idempotency requirement already makes safe.
- **Ordering guarantees:** the outbox preserves each publisher's own commit order for events it writes, and the dispatcher delivers a given module's outbox rows in that order. Across different modules' events, or across retries interleaving with newly-published events, no global ordering is guaranteed — Section 5's existing rule stands: a consumer must tolerate or correctly sequence events referencing the same entity rather than assuming transport-level ordering.
- **Exactly-once vs. at-least-once, and why at-least-once is the deliberate choice:** the outbox guarantees each committed fact is durably queued for delivery **at least once** — retries after a crash, a timeout, or a consumer failure can redeliver the same event. The system does not attempt exactly-once delivery, because guaranteeing it end-to-end (across a dispatcher, a network, and a consumer) would require either a distributed transaction or a consumer-side deduplication ledger indistinguishable in cost from just making consumers idempotent. Since Section 9 already requires every consumer to be idempotent, at-least-once delivery plus idempotent consumption is exactly-once *in effect*, at a fraction of the mechanism.

**Named retry-sensitive workflows**, restated from `data-architecture.md` Section 13 specifically in their capacity as event consumers where applicable: notification sends (Section 7), inventory reservation/release (where triggered by the Reservation Expiry background job, Section 6), and refund processing. Payment confirmation/webhook handling and order creation are retry-sensitive at the *synchronous interface* level (`module-communication.md` Section 7's checkout example), not primarily as event consumers, and are governed by `data-architecture.md` Section 13 directly. External-provider-facing retries (Moyasar, Unifonic, push, Google Maps, POS/card-on-delivery settlement) follow ADR-0032's per-integration timeout/retry/circuit-breaker table — see Section 12 below.

## 9. Idempotency

Every event consumer must produce the same end state whether it receives a given event once or multiple times — at-least-once delivery is the assumed baseline for every event in this system, not an edge case to guard against occasionally. This is the same idempotency discipline `data-architecture.md` Section 13 establishes for retry-sensitive workflows generally, applied here specifically to the event-consumption path:

- A consumer determines whether it has already handled a specific occurrence of an event before applying its effect — for example, Notification does not send a second SMS for the same `OrderPlaced` occurrence if it receives that event twice due to a retried publish.
- Idempotency is the consumer's responsibility, not the publisher's or the transport's — a publisher publishing the same fact twice by mistake (rather than as a legitimate retry) is a bug to fix at the source, but a well-built consumer tolerates it without a duplicated business effect either way.
- This is what makes Section 8's retry strategy safe: retrying a publish or a consumption step is only an acceptable default *because* every consumer is required to handle the resulting duplicate delivery correctly.

## 10. Event Versioning

Event schemas evolve the same way the API does (ADR-0017's principle, applied to events): additive, backward-compatible changes — a new optional field on an existing event — do not require a new version or a new event name; every existing consumer continues to work unmodified. A breaking change to an event's meaning or required fields is introduced as a **new, distinctly-named event** (or an explicit version suffix where the same fact genuinely needs a new shape), with the old event kept publishing alongside it until every consumer has migrated — mirroring ADR-0017's deprecation discipline rather than inventing a separate policy for events.

**Why events don't get their own versioning scheme distinct from the API's:** consistency. An AI coding assistant or a developer reasoning about "how do we evolve a contract without breaking a consumer" should be able to apply one mental model to both REST endpoints and domain events, not learn two different policies for what is fundamentally the same problem — a contract with more than one party depending on it.

## 11. Event Transport: Outbox Today, Broker on Extraction

Events are dispatched **in-process**, within the single modular-monolith deployable (ADR-0002) — a consumer handling an event runs inside the same application that published it, with no network hop and no external message broker. What makes this durable, per ADR-0029, is that the *source of pending work* is a PostgreSQL outbox table, not process memory: an in-process dispatcher reads pending outbox rows and invokes consumers, but if the process crashes, the pending rows are still there for the same instance (on restart) or another instance (Section 8's cross-instance claim rule) to pick up. This is the correct choice at current scale for the same reason the modular monolith itself is (Constitution Section 6, Section 13): a message broker is real operational complexity a solo developer should not take on before evidence demands it, and PostgreSQL already supplies the durability a broker would otherwise exist to provide.

**What changes on module extraction:** when a module is eventually pulled into its own independently deployed service (`module-communication.md` Section 11), its events can no longer be dispatched in-process to consumers living in a different process. At that point, the in-process dispatcher is replaced by a message broker (the specific technology is not decided here — a future ADR's job, once extraction is actually planned) that reads from the same kind of durable, per-module outbox and delivers across the process boundary instead of within one process. Two things are designed to make this transition low-friction rather than a rewrite:

- **The ownership and naming rules (Sections 2—3) do not change** — a broker-delivered `OrderPlaced` means exactly what an in-process `OrderPlaced` means today, with the same publisher, the same shape, the same consumers.
- **Idempotency (Section 9) is already a requirement for every consumer today**, not something extraction newly demands — a message broker's own at-least-once delivery semantics (the near-universal default for this kind of infrastructure, and already this system's own model per Section 8) are already what every consumer in this system is built to tolerate.

The trade-off accepted today is a small one: an in-process dispatcher adds a background-worker responsibility to the backend runtime (Section 8), and horizontal scaling requires the claim-based coordination Section 8 describes rather than each instance dispatching independently. Both are bounded, known costs — not open questions.

## 12. External Integration Resilience

Per ADR-0032, every external integration is wrapped by its owning module and follows a provider-specific timeout, retry, and circuit-breaker/fallback rule — the same retry discipline this document applies to event consumption (Section 8) applied instead to outbound calls to systems this architecture does not control:

| Integration | Owner | Timeout | Retry | Circuit breaker / fallback |
|---|---|---:|---|---|
| Moyasar payment API | Payment | 5 s connect/read | Idempotent create/status operations only, max 3 attempts, exponential backoff; non-idempotent calls never retried without a provider idempotency key | Circuit opens after 5 consecutive technical failures or a 50% technical failure rate over 2 minutes; online payment becomes temporarily unavailable while cash-on-delivery/card-on-delivery remain available |
| Moyasar webhooks | Payment | inbound | Signature and idempotency-key/payment-reference verification; processed at least once | Duplicate webhooks are ignored after the first successful state transition |
| Unifonic OTP/SMS | Notification/Auth | 5 s | Max 2 send attempts for the same OTP request; no new OTP generated mid-retry | Unavailability fails the login/delivery OTP flow loudly with a retry-later state, never a silent hang |
| Push notification service | Notification | 5 s | Max 3 attempts with backoff, through the outbox/job retry path (Section 8) | Business state never depends on push success; the Worker App must poll its assignment endpoint as fallback |
| Google Maps/geocoding | Store/User | 3 s autocomplete, 5 s server-side validation | Max 2 attempts | Unavailability fails new address validation gracefully; already-validated addresses continue to work |
| POS/card-on-delivery settlement import | Payment | 10 s | Scheduled retry with backoff | Manual reconciliation is the standing fallback; unresolved settlement discrepancies stay open until reviewed, never silently written off |

Every outbound call carries a correlation ID, is structured-logged, uses an idempotency key wherever the provider supports one, and stores only a provider response snapshot — never raw card data or secrets (`security-and-compliance.md` Section 10). Provider wrappers are mandatory: a module never calls a provider SDK directly from scattered call sites, so the resilience rule above is enforced in one place per integration, not re-implemented ad hoc.

**Why different providers get different failure postures:** Payment and OTP failures sit on the transactional core and must fail visibly (`reliability-and-performance.md` Section 3); push and geocoding can degrade gracefully because business state never depends on them; POS settlement can fall back to a manual process because it was never the only path to recording a card-on-delivery collection correctly.

## 13. Open Decisions

- **Synchronous API conventions** (versioning mechanics beyond ADR-0017, error-response format, request/response validation detail) remain the original skeleton's Section 1 scope, substantially already covered by ADR-0017 and `module-communication.md` Section 4, but not consolidated into one dedicated reference document.
- **An integration inventory** (a table linking every external integration to its owning module and its SDD, beyond Section 12's resilience table above) is not produced here — `02-context/system-context.md` Section 7 already provides the consolidated external-systems table this would otherwise duplicate; a dedicated inventory is only worth producing once per-integration SDDs exist to link to.
- **The dispatcher's exact claim mechanism** (e.g., `SELECT ... FOR UPDATE SKIP LOCKED` versus another claim strategy, per Section 8's cross-instance rule) is SDD-level implementation detail, not decided here.
- Historical module-ownership questions were resolved by ADR-0020 and must not be treated as open by event or integration design. Durability, retry, and external-resilience questions previously open here were resolved by ADR-0029 and ADR-0032 and must not be treated as open going forward.
- The delivery-collected-payment and delivery-failure/rejection events (Section 3 above) — previously missing from the event catalog, identified by the 2026-07-30 Architecture Readiness Review — are resolved by ADR-0035 and ADR-0036 and must not be treated as open going forward.

