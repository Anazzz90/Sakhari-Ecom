# Data Architecture — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. |
| **Stability** | Stable in philosophy (PostgreSQL as sole system of record, Redis as non-authoritative acceleration — Principles 4.2/4.3); evolving in mechanism as load patterns become real (caching scope, reconciliation cadence, archival timing). |
| **Authority** | Subordinate to `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `02-Architecture-Decisions.md`, `03-System-Context.md`, `04-Technology-Stack.md`, and `03-decomposition/module-catalog.md` / `module-communication.md`. Does not repeat their content — see Section 2. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** Describes how data is actually treated across the whole system: what PostgreSQL and Redis are each for, who owns what, how transactions and concurrency are handled, and the specific data-level conventions (the inventory model, the ledger, soft deletes, ULIDs, UTC, money representation) every module's persistence must follow. Where `module-catalog.md` says *which module owns which entity* and `module-communication.md` says *how modules avoid sharing a transaction*, this document says *how data itself is modeled and protected* underneath all of that.

**Scope.** This document covers system-wide data philosophy and conventions that apply across every module. It does not cover: per-module dependency/communication rules (`06-Module-Communication.md`); authentication/session data mechanics (`08-Security-Architecture.md`); event payload structure (`09-Event-Architecture.md`); infrastructure-level backup, point-in-time recovery, or disaster-recovery topology (`10-Deployment-Architecture.md` — this document's own "Recovery Strategy," Section 15, is about data-level recovery mechanisms, not infrastructure DR); or any schema, SQL, or implementation detail, all of which belong to future module SDDs.

**Intended audience.** The project owner, an AI coding assistant designing or reviewing a module's schema or persistence code, and the author of any future module SDD, for whom this document's conventions are non-negotiable defaults, not per-module choices to reconsider.

**Cross-references.** Grounded in Principles 4.1–4.3, 4.9, and 4.10; ADR-0003 (PostgreSQL), ADR-0004 (Redis), ADR-0009 (Module Ownership), ADR-0010 (Transactional Checkout), ADR-0014 (UTC), ADR-0015 (Integer Halala Money), and ADR-0019 (ULIDs). `04-Technology-Stack.md`'s PostgreSQL and Redis entries cover *why* those technologies were chosen and their alternatives — this document does not repeat that, only how they're used architecturally. `module-communication.md` Section 8 covers *cross-module* transaction orchestration; Section 6 below covers the transaction mechanism itself, within a single module's boundary.

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

## 9. Soft Deletes

Soft deletes and hard deletes are used for different kinds of entities, and the DDD is explicit that the two must never be confused (Section 11):

- **Master/profile data** (Customer Profile, Address, Worker Profile, Store, Category, Brand, Product, Product Image, Inventory Item) is soft-deleted or deactivated — the record stays, marked inactive, so historical references (a past order's product, a past shift's store) remain resolvable.
- **Append-only/immutable records** (Inventory Ledger, Audit Log, Order Events, Payment History, Promotion Usage, Support Ticket Comments, finalized Price Snapshots) are never deleted at all, soft or otherwise — deleting history defeats the reason it exists.
- **State-driven entities** (Order, Payment, Refund, Picking Session, Delivery Assignment, Shift, Support Ticket) use lifecycle state to represent their status, never deletion — a cancelled order is an order in a cancelled state, not a deleted order (DDD Section 3.5's explicit rule).
- **Purging** (actual removal) is reserved for a narrow set of genuinely disposable data: expired draft/cart data, short-term notification metadata, worker location history past its retention window, and anything a privacy policy requires removed — always while preserving the business records legally required to be retained (DDD Section 11.5, 11.6).

**Why the distinction matters:** conflating "soft delete" with "this business object's lifecycle changed" is a direct path to losing the audit trail Principle 4.10 requires — a soft-deleted order looks the same in a query as a genuinely-removed one unless lifecycle state and deletion are kept conceptually and mechanically separate.

## 10. ULIDs

Every entity's primary identifier is a ULID (ADR-0019), applied uniformly across all fifteen modules' owned tables. This document does not repeat ADR-0019's rationale (non-sequential identifiers avoiding business-volume leakage, combined with lexicographic sortability by creation time) — what matters here is the practical consequence: every table's `id` column follows this one convention with no exceptions, which is what keeps cross-module event payloads (`09-Event-Architecture.md`), audit log references (Section 12), and log correlation all working against the same identifier scheme without translation.

## 11. UTC Time Storage

Every timestamp column is stored as UTC (ADR-0014, DDD Section 1.3), with conversion to Arabia Standard Time happening only at the presentation layer in the clients — never in a module's own business logic. This document does not repeat ADR-0014's rationale; the practical consequence is that every duration calculation (a reservation's expiry window, an SLA check, a shift's duration) is exact regardless of which module or background job performs it, because there is exactly one time zone in play anywhere inside the Backend.

## 12. Money Representation

Every monetary column across every module (Order, Payment, Promotion, Refund, Cash Remittance) stores integer halala — never a floating-point or ambiguous decimal representation (ADR-0015, DDD Section 1.11). This document does not repeat ADR-0015's rationale; the practical consequence is that no module, background job, or reconciliation process ever performs monetary arithmetic in anything other than exact integers, and conversion to a SAR display value happens exactly once, at final presentation — never mid-calculation, and never inside a database query that could silently introduce rounding.

## 13. Concurrency Strategy

Concurrency correctness rests on four mechanisms working together (DDD Section 10):

- **PostgreSQL transactions** (Section 5) protect every multi-record change within a module.
- **Redis locks** (Section 3, Section 7) coordinate short-lived, race-prone operations — inventory reservation being the primary case — without ever becoming the authority on the outcome.
- **Optimistic validation** — before committing a workflow, the acting module revalidates current state (order state, inventory availability, payment state, worker eligibility, promotion eligibility, store status) rather than trusting state read earlier in the request, since that state may have changed underneath it (DDD Section 10.3).
- **Idempotency** — order creation, payment confirmation/webhooks, inventory reservation/release, notification sends, refund processing, and rider assignment/reassignment must all be safe to retry without duplicating their business effect (DDD Section 10.4), because external retries, mobile network instability, and provider callbacks are all expected, ordinary occurrences, not edge cases.

**Named race conditions this strategy exists to prevent** (DDD Section 10.6): two customers reserving the same last unit of stock; two pickers claiming the same order; multiple riders receiving the same packed order; duplicate payment confirmation callbacks; a refund retried after a partial provider failure. Each is prevented by some combination of database uniqueness constraints, transactional state checks, and application-level locks — not by any single mechanism alone.

**Retry discipline:** a retry is only safe where idempotency has been explicitly designed for that workflow (Section above); a failed workflow moves to an explicit failure or pending state rather than silently repeating an action that might not be reversible (DDD Section 10.5).

## 14. Recovery Strategy

This section covers data-level recovery — how the system recovers from a failed or interrupted business operation using its own data model — not infrastructure-level backup or disaster recovery, which is `10-Deployment-Architecture.md`'s subject. Three mechanisms carry this weight:

- **Compensating actions**, per `module-communication.md` Section 8: when an orchestrated, multi-module operation fails partway (the clearest case being inventory reservation failing after an order record was created), the orchestrating module performs an explicit compensating action rather than relying on a cross-module transaction rollback that doesn't exist by design.
- **Reconciliation jobs**, which recover from discrepancies that were never caught synchronously: Inventory Reconciliation (DDD Section 15.4) supports daily closing workflows and flags mismatches for review before they become silent data drift; Shift Reconciliation (DDD Section 9.9, 15.5) prevents a shift from closing until cash/card discrepancies are explicitly resolved or flagged, rather than assuming everything balanced.
- **The ledger and audit trail themselves** (Sections 8, and Section 12 below) are a recovery mechanism in the broadest sense: when something does go wrong, the durable record of what happened — not memory, not inference from current state — is what allows the discrepancy to be diagnosed and corrected, consistent with Principle 4.10's reasoning that auditability exists specifically so incidents are resolvable from evidence.

**Explicit non-goal:** this document does not define recovery point/recovery time objectives (RPO/RTO), backup frequency, or point-in-time restore procedures — those are properties of how PostgreSQL is deployed and operated, which `10-Deployment-Architecture.md` is responsible for.

## 15. Audit Data Flow

Briefly, because Section 12 (Audit module, `module-catalog.md`) and full event mechanics belong elsewhere: each domain module is responsible for emitting audit-worthy context when it performs a sensitive change (DDD Section 12.1) — a **push model**, not Audit broadly subscribing to every event in the system. Audit entries capture actor, action, affected entity, store context where applicable, before/after values for meaningful changes, and a reason code where required (DDD Section 12.2). This resolves part of `module-communication.md`'s Open Decisions note on Audit's invocation mechanism — the push model is confirmed by the DDD; the remaining open question is only whether that push is a direct synchronous call or a dedicated event, left to `09-Event-Architecture.md`.

## 16. Trade-offs and Future Evolution

**Trade-offs accepted:** every write goes through PostgreSQL even where a faster path might exist (Section 2's own tradeoff, inherited from ADR-0003); reservation locking adds a small amount of Redis-coordination complexity to the checkout path in exchange for correctness under concurrent demand (Section 7); the ledger and audit trail add write volume and storage cost in exchange for explainability (Sections 8, 15). None of these are accidental — each is a deliberate application of Principle 4.1 (correctness over raw performance).

**Future evolution:** as real load materializes, the caching scope (what's cached in Redis, and for how long), archival timing (DDD Section 11.4 explicitly defers this until usage volume is known), and reconciliation cadence are all expected to be tuned — within the philosophy this document sets, not by revisiting the philosophy itself. A genuinely evidenced need for a specialized store (search-optimized indexing, time-series rider-location storage) would be introduced as a supplementary, explicitly non-authoritative addition alongside PostgreSQL, per ADR-0003's own future-reconsideration condition — never a replacement of PostgreSQL's system-of-record role.

## 17. Open Decisions

- **Deadlock avoidance ordering:** the DDD (Section 10.7) explicitly does not define a consistent entity-update order for multi-entity transactions and states that "later implementation must document it for inventory/order/payment workflows." This document does not resolve it either — it is flagged here as outstanding work for whichever module's SDD first needs it (most likely Order or Inventory).
- **Audit's push mechanism** (direct synchronous call vs. dedicated event) — narrowed by Section 15 above to "a push model" but not fully resolved; left to `09-Event-Architecture.md`.
- **Archival timing and thresholds** (DDD Section 11.4) are explicitly deferred pending real usage data — not a gap in this document, but a genuinely not-yet-decidable point.
- Retention/purge specifics per DDD Section 11.5–11.6 (exact retention windows, anonymization triggers) are named in category but not given exact durations anywhere in the approved documents — a candidate for a future ADR once legal/accounting review (DDD Section 11.6) actually happens.
