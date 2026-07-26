# Event Architecture — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored — covers the asynchronous/event half of this file's original scope in full. **Not yet covered:** synchronous API conventions (already substantially defined by ADR-0017 and `module-communication.md` Section 4, not repeated here) and external-integration retry/timeout/circuit-breaking patterns specific to the Payment Gateway, SMS/OTP Provider, Push Notification Service, and Geocoding Provider (`02-context/system-context.md` Section 7) — flagged in Open Decisions (Section 12) as remaining work, not silently dropped. |
| **Stability** | Stable in mechanism (events are asynchronous, one-way, and ownership-scoped — Principle 4.8, ADR-0011); evolving in the specific event catalog as modules and features grow. |
| **Authority** | Subordinate to `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `decisions/README.md`, `02-context/system-context.md`, `04-cross-cutting/technology-decisions.md`, `03-decomposition/module-catalog.md`, `03-decomposition/module-communication.md`, and `04-cross-cutting/data-architecture.md`. Does not repeat their content — see Section 2. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** `module-communication.md` established that domain events are one of the system's four allowed communication modes and set the ownership rule (a module may only publish events about entities it owns) and naming convention. This document is where that gets fully specified: exactly how events are published and consumed, how background processing relates to events, how a business event becomes a customer- or worker-facing notification, how delivery failures are retried, how duplicate delivery is handled safely, how event schemas evolve, and how the current in-process event mechanism is expected to change if a module is ever extracted into its own service.

**Scope.** Domain events and everything downstream of them: naming, publish/consume rules, background jobs, the notification flow, retry, idempotency, versioning, and future transport strategy. Does **not** cover: the ownership rule and naming convention themselves (already established in `module-communication.md` Sections 6 and restated only where necessary for continuity); per-module published/consumed event lists (the authoritative, growing list is `module-catalog.md` Section 4, per module — this document does not re-list them); synchronous REST/API conventions (ADR-0017, `module-communication.md` Section 4); or any payload schema, queue technology, or code — no code, no message formats, no diagrams.

**Intended audience.** The project owner, an AI coding assistant implementing an event publisher or consumer, and the author of any future module SDD, for whom this document's rules on retry, idempotency, and versioning are non-negotiable defaults for every event in the system, not a per-event choice.

**Cross-references.** Builds on ADR-0011 (Event-Driven Asynchronous Side Effects), ADR-0010 (Transactional Checkout — events are raised only after a transaction commits), Principle 4.8, `module-catalog.md`'s per-module Published/Consumed Events fields, `module-communication.md` Sections 6—8 (events, dependency rules, transaction boundaries), and `data-architecture.md` Section 13 (idempotency as a concurrency mechanism — this document applies that same discipline specifically to event consumption).

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

## 4. Publishers

- A publisher raises an event as the final step of the transaction that made it true, or immediately after that transaction commits — never before, and never as part of deciding whether to commit.
- A publisher never waits for any consumer's reaction, never receives a return value from publishing beyond confirmation that the event was accepted for delivery, and never learns which or how many consumers exist.
- A publisher is responsible for the event actually being raised at least once per fact — if the publishing step itself fails (as opposed to the underlying transaction), that is treated as an operational failure to be retried at the publish step, not silently dropped (Section 8).

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

Not every asynchronous process in the system is triggered by a domain event — some are scheduled, sweeping over data on a cadence rather than reacting to a specific fact. The DDD names seven (Section 15), each mapped here to how it relates to the event system:

| Background job | Trigger | Relationship to events |
|---|---|---|
| Reservation Expiry | Scheduled sweep | Finds reservations past their checkout/substitution window and releases them (`data-architecture.md` Section 7); publishes `InventoryReservationReleased` as a normal event once acted on — the *trigger* is a schedule, but the *outcome* is announced the same way any other Inventory change is. |
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

- **Event publication itself failing** (the publish step, after the underlying transaction already committed) is retried at the publishing module's own boundary until it succeeds — the fact already happened and must eventually be announced; this is not optional or best-effort.
- **Event consumption/handling failing** (a consumer's reaction to an event throws, times out, or otherwise doesn't complete) is retried with backoff at the consumer, up to a bounded number of attempts appropriate to that consumer's own workflow (DDD Section 10.5's "failed workflows should move to explicit failure/pending states rather than silently repeating irreversible actions" applies directly here). A consumer that exhausts its retries surfaces the event as a recorded failure — visible to Notification's own delivery-status history (Section 7) where relevant, or to Audit where the failure itself is consequential — never a silent drop.

**Named retry-sensitive workflows**, restated from `data-architecture.md` Section 13 specifically in their capacity as event consumers where applicable: notification sends (Section 7), inventory reservation/release (where triggered by the Reservation Expiry background job, Section 6), and refund processing. Payment confirmation/webhook handling and order creation are retry-sensitive at the *synchronous interface* level (`module-communication.md` Section 7's checkout example), not primarily as event consumers, and are governed by `data-architecture.md` Section 13 directly.

## 9. Idempotency

Every event consumer must produce the same end state whether it receives a given event once or multiple times — at-least-once delivery is the assumed baseline for every event in this system, not an edge case to guard against occasionally. This is the same idempotency discipline `data-architecture.md` Section 13 establishes for retry-sensitive workflows generally, applied here specifically to the event-consumption path:

- A consumer determines whether it has already handled a specific occurrence of an event before applying its effect — for example, Notification does not send a second SMS for the same `OrderPlaced` occurrence if it receives that event twice due to a retried publish.
- Idempotency is the consumer's responsibility, not the publisher's or the transport's — a publisher publishing the same fact twice by mistake (rather than as a legitimate retry) is a bug to fix at the source, but a well-built consumer tolerates it without a duplicated business effect either way.
- This is what makes Section 8's retry strategy safe: retrying a publish or a consumption step is only an acceptable default *because* every consumer is required to handle the resulting duplicate delivery correctly.

## 10. Event Versioning

Event schemas evolve the same way the API does (ADR-0017's principle, applied to events): additive, backward-compatible changes — a new optional field on an existing event — do not require a new version or a new event name; every existing consumer continues to work unmodified. A breaking change to an event's meaning or required fields is introduced as a **new, distinctly-named event** (or an explicit version suffix where the same fact genuinely needs a new shape), with the old event kept publishing alongside it until every consumer has migrated — mirroring ADR-0017's deprecation discipline rather than inventing a separate policy for events.

**Why events don't get their own versioning scheme distinct from the API's:** consistency. An AI coding assistant or a developer reasoning about "how do we evolve a contract without breaking a consumer" should be able to apply one mental model to both REST endpoints and domain events, not learn two different policies for what is fundamentally the same problem — a contract with more than one party depending on it.

## 11. Future Event Bus Strategy

Today, events are dispatched **in-process**, within the single modular-monolith deployable (ADR-0002) — publishing an event and a consumer handling it both happen inside the same running application, with no network hop and no external message broker. This is the correct choice at current scale for the same reason the modular monolith itself is (Constitution Section 6, Section 13): a message broker is real operational complexity a solo developer should not take on before evidence demands it.

**What changes on module extraction:** when a module is eventually pulled into its own independently deployed service (`module-communication.md` Section 11), its events can no longer be dispatched in-process to consumers living in a different process. At that point, the in-process dispatcher is replaced by a message broker (the specific technology is not decided here — a future ADR's job, once extraction is actually planned) for events crossing the new process boundary. Two things are designed to make this transition low-friction rather than a rewrite:

- **The ownership and naming rules (Sections 2—3) do not change** — a broker-delivered `OrderPlaced` means exactly what an in-process `OrderPlaced` means today, with the same publisher, the same shape, the same consumers.
- **Idempotency (Section 9) is already a requirement for every consumer today**, not something extraction newly demands — a message broker's own at-least-once delivery semantics (the near-universal default for this kind of infrastructure) are already what every consumer in this system is built to tolerate.

The trade-off accepted today is a small one: in-process events give no delivery durability beyond the process's own lifetime (if the application crashes between publish and a consumer handling it, an in-memory-only dispatcher could lose that delivery). Whether the current implementation already backs this with a durable queue (versus a purely in-memory dispatcher) is an SDD-level/implementation decision this document does not make — but Section 9's idempotency requirement holds regardless of the answer, precisely so that decision can be made or changed later without touching every consumer's own logic.

## 12. Open Decisions

- **Whether the in-process event dispatch is backed by a durable queue or a purely in-memory mechanism** (Section 11) is not decided here — an SDD-level choice, made safe either way by Section 9's idempotency requirement.
- **Synchronous API conventions** (versioning mechanics beyond ADR-0017, error-response format, request/response validation detail) remain the original skeleton's Section 1 scope, substantially already covered by ADR-0017 and `module-communication.md` Section 4, but not consolidated into one dedicated reference document.
- **External-integration retry/timeout/circuit-breaking patterns** specific to the Payment Gateway, SMS/OTP Provider, Push Notification Service, and Geocoding Provider (the original skeleton's Section 3) are not detailed in this document — Section 8 covers *event* retry; a provider-specific pattern (timeouts, circuit-breaking thresholds per external system) remains outstanding work, most naturally paired with `05-deployment/infrastructure-and-release.md` or a per-integration SDD.
- **An integration inventory** (the original skeleton's Section 4, a table linking every external integration to its owning module and its SDD) is not produced here — `02-context/system-context.md` Section 7 already provides the consolidated external-systems table this would otherwise duplicate; a dedicated inventory is only worth producing once per-integration SDDs exist to link to.
- Historical module-ownership questions were resolved by ADR-0020 and must not be treated as open by event or integration design.

