# Data Architecture — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. |
| **Stability** | Stable in philosophy (PostgreSQL as sole system of record, Redis as non-authoritative acceleration — Principles 4.2/4.3); evolving in mechanism as load patterns become real (caching scope, reconciliation cadence, archival timing). |
| **Authority** | Subordinate to `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `decisions/README.md`, `02-context/system-context.md`, `04-cross-cutting/technology-decisions.md`, and `03-decomposition/module-catalog.md` / `module-communication.md`. Does not repeat their content — see Section 2. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** Describes how data is actually treated across the whole system: what PostgreSQL and Redis are each for, who owns what, how transactions and concurrency are handled, and the specific data-level conventions (the inventory model, the ledger, soft deletes, ULIDs, UTC, money representation) every module's persistence must follow. Where `module-catalog.md` says *which module owns which entity* and `module-communication.md` says *how modules avoid sharing a transaction*, this document says *how data itself is modeled and protected* underneath all of that.

**Scope.** This document covers system-wide data philosophy and conventions that apply across every module. It does not cover: per-module dependency/communication rules (`03-decomposition/module-communication.md`); authentication/session data mechanics (`04-cross-cutting/security-and-compliance.md`); event payload structure (`04-cross-cutting/integration-and-messaging.md`); infrastructure-level backup, point-in-time recovery, or disaster-recovery topology (`05-deployment/infrastructure-and-release.md` — this document's own "Recovery Strategy," Section 15, is about data-level recovery mechanisms, not infrastructure DR); or any schema, SQL, or implementation detail, all of which belong to future module SDDs.

**Intended audience.** The project owner, an AI coding assistant designing or reviewing a module's schema or persistence code, and the author of any future module SDD, for whom this document's conventions are non-negotiable defaults, not per-module choices to reconsider.

**Cross-references.** Grounded in Principles 4.1—4.3, 4.9, and 4.10; ADR-0003 (PostgreSQL), ADR-0004 (Redis), ADR-0009 (Module Ownership), ADR-0010 (Transactional Checkout), ADR-0014 (UTC), ADR-0015 (Integer Halala Money), ADR-0019 (ULIDs), ADR-0035 (Delivery-Collected Payment Event), ADR-0037 (Payment Ledger), and ADR-0038 (Line-Item Aware Partial Refunds). `04-cross-cutting/technology-decisions.md`'s PostgreSQL and Redis entries cover *why* those technologies were chosen and their alternatives — this document does not repeat that, only how they're used architecturally. `module-communication.md` Section 8 covers *cross-module* transaction orchestration; Section 6 below covers the transaction mechanism itself, within a single module's boundary.

## 2. PostgreSQL Philosophy

PostgreSQL is the sole, exclusive system of record for all durable business state (Principle 4.2, ADR-0003) — every module's owned entities live here, and nowhere else is ever treated as authoritative for them. This is not a caching or performance decision; it is a correctness decision. A system with two authoritative stores eventually produces a moment where they disagree, with no principled way to know which one is right (ADR-0003's own rationale). PostgreSQL's ACID guarantees are what make Principle 4.9's transactional-integrity promise achievable at all, and its referential integrity is what prevents the orphaned-record class of bug the DDD names as a core data-integrity requirement (DDD Section 1.5).

**Why this matters architecturally, not just technologically:** every module's public interface (`module-communication.md` Section 5) is only trustworthy because the data behind it is guaranteed consistent by PostgreSQL's transaction model. If a module's own writes weren't transactionally safe, the whole "modules communicate only through interfaces" discipline (ADR-0009) would just be hiding an unreliable foundation behind a clean-looking API.

## 3. Redis Philosophy

Redis exists for exactly three purposes: caching, coordination (locks, rate limits), and short-lived operational state — and for nothing else (Principle 4.3, ADR-0004). It is never the durable source for inventory, orders, payments, refunds, shifts, or audit logs (DDD Section 3.10). This boundary is enforced by a simple test applied to every proposed use of Redis: **would losing this on a flush lose a business fact?** If yes, it belongs in PostgreSQL; if no, Redis is fair game.

The one place Redis does load-bearing work close to the correctness boundary is **short-lived locking around inventory reservation** (DDD Section 10.1, detailed in Section 8 below) — even there, Redis only coordinates *who gets to attempt* a reservation first; the reservation itself is not real until committed in PostgreSQL. Redis can be flushed, resized, or restarted at any time with zero business-data risk, which is the whole point of drawing the boundary this strictly.

## 4. Data Ownership

Every entity in the system has exactly one owning module, which is the only code permitted to read or write its tables directly (Principle 4.5, ADR-0009). This document does not restate the ownership table — `module-catalog.md` Section 5 (Cross-Module Summary Table) and the eventual `data-ownership-map.md` are the authoritative per-entity references. What this document adds is the *mechanical* consequence: because ownership is structural (enforced through public interfaces, not convention), a module's schema can change freely as long as its interface contract doesn't, and a migration touching a table outside the module that's supposed to own it is a correctness signal worth stopping for, not a routine change.

## 5. Transactions

Every workflow that updates more than one authoritative record within a single module's boundary must execute inside a database transaction (Principle 4.9; DDD Section 1.9, 10.2) — this applies to order creation, inventory reservation, packing completion, payment confirmation, refund processing, inventory adjustment, and shift reconciliation, each as its own atomic unit (DDD Section 9 names the atomicity and rollback expectation for each of these individually). A transaction never spans two modules' tables — that boundary, and how cross-module consistency is achieved instead (orchestration with explicit compensation), is `module-communication.md` Section 8's subject, not repeated here.

**Why per-workflow, not per-request:** a request that touches multiple modules (checkout being the clearest case) is *not* one transaction — it's a sequence of module-local transactions coordinated by an orchestrator. Within any single module, though, every multi-record change is atomic. This is what lets each module reason about its own data's correctness completely independently of what any other module is doing at the same moment.

## 6. Inventory Model

Inventory is tracked per store (DDD's store-isolation model, Section 13) as three related but distinct concepts, each with its own entity:

- **Inventory Item** — the current, authoritative stock level for a product at a store: on-hand quantity and reserved quantity, together determining what's actually available to sell.
- **Inventory Reservation** — a temporary hold against an Inventory Item's available quantity, created when an order needs stock and released or consumed depending on what happens to that order.
- **Inventory Ledger** — the append-only history of every stock movement and why it happened (Section 8).

**Why three entities and not one "quantity" field:** a single mutable quantity field can tell you what's true *now*, but not why, and cannot safely represent "this stock is spoken for but not yet actually shipped" — which is exactly the state an order needs between checkout and packing. Splitting current state (Inventory Item), in-flight claims (Inventory Reservation), and historical movement (Inventory Ledger) into three entities is what makes Section 7's reservation lifecycle and Section 8's ledger both possible without contradiction.

## 7. Reservation Lifecycle

An Inventory Reservation moves through a small number of states, each transition itself a transactional operation (Section 5):

1. **Requested** — Order calls Inventory's `ReserveStock` (`module-communication.md` Section 7's checkout example, step 4) as the step immediately following order-record creation. Inventory validates current availability, applies a short-lived Redis lock to prevent a race against a concurrent reservation attempt for the same stock (DDD Section 10.1, 10.6 — "two customers trying to reserve the same last item" is the named race condition this exists to prevent), and, if available, commits a reservation record in PostgreSQL that reduces available quantity without touching on-hand quantity yet.
2. **Reserved** — the confirmed, holding state. Stock is unavailable to any other order, but has not physically left the store.
3. **Consumed** — at packing completion (DDD Section 9.4), the reservation is converted into an actual stock deduction: on-hand quantity decreases, the reservation is closed, and an Inventory Ledger entry is recorded — all in one transaction, so a failed packing completion never partially deducts stock.
4. **Released or Expired** — if the order is cancelled, or if the reservation exceeds its allowed checkout/substitution window (DDD Section 15.1's "Reservation Expiry" background job), the reservation is released: available quantity is restored, with no ledger entry required (nothing was ever actually removed from stock).

**Why reservations expire automatically:** an abandoned checkout must not permanently lock stock away from other customers. The expiry window is a background-job concern (DDD Section 15.1), not a synchronous one — Inventory doesn't block waiting to see if a reservation will be used; it periodically sweeps for reservations that have overstayed their window.

## 8. Inventory Ledger

The Inventory Ledger is the append-only, immutable record of every stock movement and its reason (DDD Sections 1.7, 1.8, 3.7, 5.15, 14.3). It exists because current state — the Inventory Item's on-hand quantity — can only ever answer "what is true now," never "why is it true" (DDD Section 3.7's own framing, which this document adopts directly). Every deduction, adjustment, return, damage write-off, or reconciliation correction is a new ledger entry, never a rewrite of a past one (Principle 4.10's audit discipline, applied specifically to inventory).

**Why this matters beyond bookkeeping:** when a stock discrepancy is discovered during reconciliation (DDD Section 15.4, 9.9), the ledger is what makes the discrepancy *explainable* rather than just *visible*. Without it, an Inventory Item's current quantity is a number with no history behind it — correct until it's wrong, with no way to determine when or why.

## 8A. Payment Ledger

Per ADR-0037, Payment owns an append-only, immutable **Payment Ledger** — the same structural mechanism as the Inventory Ledger (Section 8), applied to money instead of stock, closing the exact asymmetry an Architecture Readiness Review identified between the two: Payment's own tables (`payments`, `payment_history`, `cash_remittances`, `card_on_delivery_records`, `refunds`) could previously answer "what is Payment's current state" but not "does the sum of every financial movement against this order actually reconcile."

Every financial movement against an order generates an immutable ledger entry — Payment Authorized, Payment Captured, COD Collected, Card-on-Delivery Collected, Refund Issued, Refund Reversed, Settlement Recorded, Adjustment, and Chargeback — each carrying a Ledger ID, Payment ID, Order ID, Entry Type, Amount (integer halala, Section 12), Currency, Reference, Source, and Timestamp (UTC, Section 11). Ledger entries are never updated or deleted (Section 9's append-only/immutable category applies to `payment_ledger` exactly as it already applies to `inventory_ledger`, `audit_logs`, and `order_events`).

**Why this matters beyond bookkeeping:** exactly Section 8's own reasoning, restated for money — a payment's current authorized/captured/refunded totals are a fast-path answer to "what is true now," derived from ledger history where appropriate, but the ledger itself is what makes that state *explainable* when a discrepancy surfaces, rather than merely *visible*.

**Reconciliation.** Ledger entries are reconciled against payment-provider settlement reports (Moyasar, per ADR-0022; POS/terminal settlement, per ADR-0032) the same way Inventory Reconciliation (Section 14) compares the Inventory Ledger against physical counts — a scheduled job flagging discrepancies for review, never silently accepted or auto-corrected.

**Delivery-collected payments (ADR-0035).** A ledger entry for a cash or card-on-delivery collection is written by Payment, inside Payment's own transaction, when Payment processes Order's forwarded `DeliveryCompletedWithPayment` handling — Delivery never writes a ledger entry, exactly as it never writes to any other Payment-owned table.

**Line-item refunds (ADR-0038).** A Refund Issued (or Refund Reversed) ledger entry is written alongside each line-item refund record, keeping the ledger's movement history and Refund's own line-item detail in agreement by construction — the ledger records that money moved and how much; the Refund record explains which order lines and at what original, promotion-adjusted price.

## 9. Soft Deletes

Soft deletes and hard deletes are used for different kinds of entities, and the DDD is explicit that the two must never be confused (Section 11):

- **Master/profile data** (Customer Profile, Address, Worker Profile, Store, Category, Brand, Product, Product Image, Inventory Item) is soft-deleted or deactivated — the record stays, marked inactive, so historical references (a past order's product, a past shift's store) remain resolvable.
- **Append-only/immutable records** (Inventory Ledger, Audit Log, Order Events, Payment History, Promotion Usage, Support Ticket Comments, finalized Price Snapshots) are never deleted at all, soft or otherwise — deleting history defeats the reason it exists.
- **State-driven entities** (Order, Payment, Refund, Picking Session, Delivery Assignment, Shift, Support Ticket) use lifecycle state to represent their status, never deletion — a cancelled order is an order in a cancelled state, not a deleted order (DDD Section 3.5's explicit rule).
- **Purging** (actual removal) is reserved for a narrow set of genuinely disposable data: expired draft/cart data, short-term notification metadata, worker location history past its retention window, and anything a privacy policy requires removed — always while preserving the business records legally required to be retained (DDD Section 11.5, 11.6).

**Why the distinction matters:** conflating "soft delete" with "this business object's lifecycle changed" is a direct path to losing the audit trail Principle 4.10 requires — a soft-deleted order looks the same in a query as a genuinely-removed one unless lifecycle state and deletion are kept conceptually and mechanically separate.

## 10. ULIDs

Every entity's primary identifier is a ULID (ADR-0019), applied uniformly across all sixteen modules' owned tables. This document does not repeat ADR-0019's rationale (non-sequential identifiers avoiding business-volume leakage, combined with lexicographic sortability by creation time) — what matters here is the practical consequence: every table's `id` column follows this one convention with no exceptions, which is what keeps cross-module event payloads (`04-cross-cutting/integration-and-messaging.md`), audit log references (Section 12), and log correlation all working against the same identifier scheme without translation.

## 11. UTC Time Storage

Every timestamp column is stored as UTC (ADR-0014, DDD Section 1.3), with conversion to Arabia Standard Time happening only at the presentation layer in the clients — never in a module's own business logic. This document does not repeat ADR-0014's rationale; the practical consequence is that every duration calculation (a reservation's expiry window, an SLA check, a shift's duration) is exact regardless of which module or background job performs it, because there is exactly one time zone in play anywhere inside the Backend.

## 12. Money Representation

Every monetary column across every module (Order, Payment, Promotion, Refund, Cash Remittance) stores integer halala — never a floating-point or ambiguous decimal representation (ADR-0015, DDD Section 1.11). This document does not repeat ADR-0015's rationale; the practical consequence is that no module, background job, or reconciliation process ever performs monetary arithmetic in anything other than exact integers, and conversion to a SAR display value happens exactly once, at final presentation — never mid-calculation, and never inside a database query that could silently introduce rounding.

## 13. Concurrency Strategy

Concurrency correctness rests on four mechanisms working together (DDD Section 10):

- **PostgreSQL transactions** (Section 5) protect every multi-record change within a module.
- **Redis locks** (Section 3, Section 7) coordinate short-lived, race-prone operations — inventory reservation being the primary case — without ever becoming the authority on the outcome.
- **Optimistic validation** — before committing a workflow, the acting module revalidates current state (order state, inventory availability, payment state, worker eligibility, promotion eligibility, store status) rather than trusting state read earlier in the request, since that state may have changed underneath it (DDD Section 10.3).
- **Idempotency** — order creation, payment confirmation/webhooks, inventory reservation/release, notification sends, refund processing, and rider assignment/reassignment (including batch formation and per-stop reassignment within a batch, ADR-0034 — reassigning one stop must never duplicate or disturb the other stop's own assignment) must all be safe to retry without duplicating their business effect (DDD Section 10.4), because external retries, mobile network instability, and provider callbacks are all expected, ordinary occurrences, not edge cases.

**Named race conditions this strategy exists to prevent** (DDD Section 10.6): two customers reserving the same last unit of stock; two pickers claiming the same order; multiple riders receiving the same packed order; duplicate payment confirmation callbacks; a refund retried after a partial provider failure; and, per ADR-0034, the same order being claimed by two concurrently-forming batches, or a batch's driver-assignment step racing against an individual stop's own cancellation. Each is prevented by some combination of database uniqueness constraints, transactional state checks, and application-level locks — not by any single mechanism alone. The last case is why Delivery Stop's constraint that a given Delivery Assignment appears in at most one stop at a time (DDD Section 5.41) is enforced transactionally at batch-formation time, not assumed.

**Retry discipline:** a retry is only safe where idempotency has been explicitly designed for that workflow (Section above); a failed workflow moves to an explicit failure or pending state rather than silently repeating an action that might not be reversible (DDD Section 10.5).

**Locking order and deadlock avoidance.** Because a transaction never spans two modules' tables (Section 5; ADR-0021), cross-module deadlocks are structurally impossible — a deadlock can only arise between two transactions contending for rows *within the same module's own tables*. Each module that touches more than one of its own tables in a single transaction acquires row locks in a fixed order, consistently applied across every workflow in that module, rather than an order that varies by call site:

- **Inventory** locks in the order: Inventory Item row, then any Inventory Reservation row(s) referencing it, then the Inventory Ledger insert (append-only, non-contending). Concurrent reservation attempts against the same Inventory Item therefore always queue behind the same lock, never behind each other's reservation rows in reverse order.
- **Order** locks its own Order row before its Order Item rows, and always in ascending order of Order Item primary key where more than one item row is touched in the same transaction, so two concurrent updates to the same order's items cannot lock them in opposite order.
- **Payment** locks its own Payment row before writing a Payment History entry, a Refund row (where applicable) after the Payment row it reverses — never the reverse — and the Payment Ledger insert last (append-only, non-contending, per ADR-0037 — mirroring Inventory's own ledger-last ordering), so a confirmation and a refund attempt racing against the same payment cannot deadlock against each other.
- **Delivery** locks a Delivery Assignment row before the Picking Session row it's derived from, consistently, so a concurrent assignment attempt and a concurrent picking-completion update cannot lock the pair in opposite order.

**Optimistic vs. pessimistic locking, applied per workflow, not uniformly:**
- **Pessimistic (row-level lock, held for the transaction's duration)** is used exactly where Section 7 already describes it — inventory reservation's Redis-coordinated, then PostgreSQL-committed claim on an Inventory Item — because the cost of losing a race (overselling the last unit) outweighs the cost of a short lock wait.
- **Optimistic (revalidate immediately before commit, no lock held between read and write)** is the default everywhere else a workflow's correctness depends on state that could change between read and commit (order state before confirming, promotion eligibility before applying, worker eligibility before assigning) — per Section 13's existing "Optimistic validation" bullet above. Optimistic validation is preferred by default because it does not hold a lock across a request's full round-trip; pessimistic locking is reserved for the one workflow (inventory reservation) where the race's business cost specifically justifies it.
- **Payment concurrency** is handled optimistically at the row level (a Payment row's state transition is guarded by a state check inside its own transaction, e.g. only a `Pending` payment can move to `Authorized`) combined with idempotency keys for provider callbacks (Section 8) — never a held lock spanning the external gateway round-trip, since a lock held across a network call to Moyasar would block the same payment's own webhook confirmation from ever landing.
- **Order concurrency** is handled optimistically: an order's state transitions are guarded by explicit state checks within its own transaction (Section 5), and the orchestration/compensation pattern (`module-communication.md` Section 8) — not a lock held across the whole checkout orchestration — is what resolves a downstream (Inventory/Payment) failure.

## 14. Recovery Strategy

This section covers data-level recovery — how the system recovers from a failed or interrupted business operation using its own data model — not infrastructure-level backup or disaster recovery, which is `05-deployment/infrastructure-and-release.md`'s subject. Three mechanisms carry this weight:

- **Compensating actions**, per `module-communication.md` Section 8: when an orchestrated, multi-module operation fails partway (the clearest case being inventory reservation failing after an order record was created), the orchestrating module performs an explicit compensating action rather than relying on a cross-module transaction rollback that doesn't exist by design.
- **Reconciliation jobs**, which recover from discrepancies that were never caught synchronously: Inventory Reconciliation (DDD Section 15.4) supports daily closing workflows and flags mismatches for review before they become silent data drift; Shift Reconciliation (DDD Section 9.9, 15.5) prevents a shift from closing until cash/card discrepancies are explicitly resolved or flagged, rather than assuming everything balanced.
- **The ledger and audit trail themselves** (Sections 8, and Section 12 below) are a recovery mechanism in the broadest sense: when something does go wrong, the durable record of what happened — not memory, not inference from current state — is what allows the discrepancy to be diagnosed and corrected, consistent with Principle 4.10's reasoning that auditability exists specifically so incidents are resolvable from evidence.

**Explicit non-goal:** this document does not define recovery point/recovery time objectives (RPO/RTO), backup frequency, or point-in-time restore procedures — those are properties of how PostgreSQL is deployed and operated, which `05-deployment/infrastructure-and-release.md` is responsible for.

## 15. Audit Data Flow

Briefly, because Section 12 (Audit module, `module-catalog.md`) and full event mechanics belong elsewhere: each domain module is responsible for emitting audit-worthy context when it performs a sensitive change (DDD Section 12.1) — a **push model**, not Audit broadly subscribing to every event in the system. Audit entries capture actor, action, affected entity, store context where applicable, before/after values for meaningful changes, and a reason code where required (DDD Section 12.2). `integration-and-messaging.md` Section 5 resolves the remaining invocation-mechanism question: Audit consumes the same event other modules consume where one already exists for the underlying action, and receives a direct, synchronous `RecordAuditEntry` call where no event exists (an administrator viewing sensitive data being the named case).

**Audit durability — no consequential action loses its audit intent.** Both invocation paths are backed by durable PostgreSQL writes, never a best-effort or in-memory recording step:

- **Event-consumed audit entries** are backed by the same outbox durability as any other event consumer (Section 16 below; `integration-and-messaging.md` Sections 4, 8): the triggering fact is durably queued the moment the owning module's transaction commits, and Audit's own processing of it is retried by the dispatcher until it succeeds, exactly like any other consumer. An audit entry sourced from an event cannot be silently lost by a process crash, because the event that would produce it survives in PostgreSQL independent of any process's uptime.
- **Direct-call audit entries** (the no-event case) are written by Audit inside its own PostgreSQL transaction at the moment `RecordAuditEntry` is invoked. This write does not block or fail the caller's own business operation (`module-catalog.md` Section 4.14's Boundaries) — it is fire-and-forget with respect to the caller's success — but "fire-and-forget for the caller" does not mean "unrecorded if it fails": a failed direct-call audit write is itself surfaced as an operational failure to be alerted on and retried (consistent with `observability-and-operations.md`'s treatment of audit-recording failure as a monitored condition), never silently dropped and never treated as acceptable data loss.
- **Failure recovery.** Because Audit Log rows are never deleted or overwritten (Section 9), and every audit-producing path above resolves to a durable PostgreSQL write (either directly, or via the outbox), an audit-recording failure is always a *detectable, retryable* gap — never an unrecoverable one. There is no code path in this architecture where a consequential action succeeds while its audit record is silently and permanently lost.
- **Compliance rationale.** SAMA and PDPL both require that consequential financial and data-access actions be reconstructable after the fact (`security-and-compliance.md` Section 11). A best-effort, in-memory-only audit write would make that reconstruction impossible exactly when it matters most — during an incident or a regulatory review, which is precisely when a process is most likely to have crashed or restarted. Routing audit durability through the same outbox mechanism already built for correctness-critical domain events (rather than inventing a second, audit-specific durability mechanism) keeps the guarantee consistent with everything else in this document and avoids a parallel, differently-behaved code path an implementer could get wrong.

## 16. Outbox Table Ownership

Per ADR-0029, every module that publishes domain events owns its own outbox table, in the same PostgreSQL database and the same transaction scope as the entities it owns — there is no shared, cross-module outbox table, consistent with Section 4's ownership rule applying to outbox rows exactly as it applies to any other data a module holds.

- **Ownership.** A module's outbox table is written to only by that module's own service layer, in the same transaction as the entity change the outbox row describes (`integration-and-messaging.md` Section 4). No other module ever writes to, or reads business meaning from, another module's outbox table directly — a consumer receives an event through the dispatcher, never by querying another module's outbox rows.
- **Persistence.** An outbox row records at minimum: the event name and payload, the owning module, a creation timestamp, a processing status, and delivery-attempt bookkeeping (attempt count, last error, next-retry time) per `integration-and-messaging.md` Section 8. It is never updated to change the fact it records — only its delivery-status fields change as the dispatcher processes it — preserving the same "facts are never rewritten" discipline Section 8 above applies to the Inventory Ledger.
- **Cross-instance behavior.** Multiple horizontally-scaled backend instances (ADR-0030) read from the same outbox tables; a dispatcher instance claims a pending row before processing it (`integration-and-messaging.md` Section 8), so ownership of *who processes which row* is arbitrated in PostgreSQL, not assumed per-instance.
- **Retention.** A processed outbox row is retained briefly for operational visibility and duplicate-detection, then purged on the same cadence as other disposable operational data (Section 9's Purging category) — the outbox is a delivery mechanism, not a permanent historical record; the Inventory Ledger, Audit Log, and Order Events remain the system's permanent history where a business fact needs one.

## 17. Trade-offs and Future Evolution

**Trade-offs accepted:** every write goes through PostgreSQL even where a faster path might exist (Section 2's own tradeoff, inherited from ADR-0003); reservation locking adds a small amount of Redis-coordination complexity to the checkout path in exchange for correctness under concurrent demand (Section 7); the ledger and audit trail add write volume and storage cost in exchange for explainability (Sections 8, 15). None of these are accidental — each is a deliberate application of Principle 4.1 (correctness over raw performance).

**Future evolution:** as real load materializes, the caching scope (what's cached in Redis, and for how long), archival timing (DDD Section 11.4 explicitly defers this until usage volume is known), and reconciliation cadence are all expected to be tuned — within the philosophy this document sets, not by revisiting the philosophy itself. A genuinely evidenced need for a specialized store (search-optimized indexing, time-series rider-location storage) would be introduced as a supplementary, explicitly non-authoritative addition alongside PostgreSQL, per ADR-0003's own future-reconsideration condition — never a replacement of PostgreSQL's system-of-record role.

## 18. Open Decisions

- **Archival timing and thresholds** (DDD Section 11.4) are explicitly deferred pending real usage data — not a gap in this document, but a genuinely not-yet-decidable point.
- Retention/purge specifics per DDD Section 11.5—11.6 (exact retention windows, anonymization triggers) are named in category but not given exact durations anywhere in the approved documents — a candidate for a future ADR once legal/accounting review (DDD Section 11.6) actually happens.
- Deadlock avoidance ordering (Section 13 above) and Audit's push mechanism (Section 15 above) were both previously open here and are now resolved; they must not be treated as open going forward.
- The Payment Ledger's existence and structure (Section 8A) — previously an unaddressed gap identified by the 2026-07-30 Architecture Readiness Review — is resolved by ADR-0037 and must not be treated as open going forward.

