# Performance, Scalability, and Reliability — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. |
| **Stability** | Stable in philosophy (correctness before performance, vertical scaling before horizontal, complexity earned not speculated — Principle 4.1, Constitution Section 6/13); evolving in specific targets and thresholds as real traffic data becomes available. |
| **Authority** | Subordinate to `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `02-Architecture-Decisions.md`, `03-System-Context.md`, `04-Technology-Stack.md`, `03-decomposition/module-catalog.md` / `module-communication.md`, and `04-cross-cutting/data-architecture.md` / `integration-and-messaging.md`. Does not repeat their content — see Section 2. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** Sets the system's performance and scalability philosophy and the concrete mechanisms — caching, search, read/write optimization, queueing, horizontal and database scaling, and the eventual path to service extraction — used to pursue it without compromising the correctness guarantees the rest of this documentation set establishes. It also carries this file's original scope: availability expectations and failure-handling strategy, the bridge between `06-quality-attributes/nfr-matrix.md`'s measurable targets (once authored) and how the system is actually built to hit them.

**Scope.** Performance philosophy, caching strategy, search strategy, read/write optimization, queue usage, horizontal scaling, database scaling, future microservice extraction, and capacity planning principles — plus availability targets, latency budgets, and failure-handling strategy (this file's pre-existing scope). Does not cover: infrastructure topology or the specific compute/scaling mechanics of *how* an additional instance gets provisioned (`05-deployment/infrastructure-and-release.md` Section 12, which this document's Section 8 extends rather than repeats); the Redis/PostgreSQL philosophy itself (`04-cross-cutting/data-architecture.md` Sections 2–3, assumed here); or event mechanics (`04-cross-cutting/integration-and-messaging.md`, assumed here). No code, no diagrams, no specific numeric SLAs where none has been established elsewhere — flagged in Open Decisions rather than invented.

**Intended audience.** The project owner, an AI coding assistant making a scaling-relevant decision (adding a cache, considering a read replica, evaluating whether a module is a service-extraction candidate), and the author of any future SDD, for whom this document's philosophy is the test any such decision must pass before being made.

**Cross-references.** Grounded in Principle 4.1 (correctness over raw performance), Constitution Section 6 (vertical scaling and simple caching preferred) and Section 13 (complexity is earned); ADR-0002/0009 (modular monolith, designed extractability); `module-catalog.md`'s Search entry (Section 4.5) and Future Expansion fields naming Order/Inventory as extraction candidates; `01-Architecture-Design-Specification.md` Sections 13 and 15 (Scalability Philosophy, Future Evolution Strategy).

## 2. Performance Philosophy

Performance is pursued *within* the bounds correctness already sets (Principle 4.1) — never the other way around. Concretely, this means: the checkout path (`module-communication.md` Section 10's canonical example) is fast because everything that isn't strictly required for order/inventory/payment correctness is pushed out to asynchronous events (ADR-0011), not because any correctness guarantee was loosened to make it faster. Every mechanism in this document — caching, search, read replicas, horizontal scaling — is an addition to a system whose correctness is already established, never a substitute for it, and never adopted speculatively (Constitution Section 13's "complexity is earned"): each is justified below by a specific problem it solves, not by anticipated future need.

## 3. Availability Targets and Failure-Handling Strategy

**Availability allocation.** The system's overall availability is only as good as its weakest required-synchronous dependency. On the transactional core (order placement, inventory reservation, payment initiation — `module-communication.md` Section 10), availability depends on the Backend, PostgreSQL, and (briefly, for coordination) Redis, plus whichever external system the specific operation requires (the Payment Gateway for payment steps). On everything else — notifications, analytics, search indexing — availability is decoupled by design (ADR-0011): a Notification or Analytics outage never becomes a checkout outage, because nothing in the transactional core synchronously depends on them (`module-catalog.md`'s Forbidden Dependencies for Order, Notification, Analytics, and Search all enforce this from different directions).

**Failure-handling strategy — what fails loudly vs. gracefully:**
- The transactional core fails **loudly and visibly**: a failed inventory reservation or payment initiation returns a clear failure to the caller, with Order's compensating action (`module-communication.md` Section 8) ensuring no inconsistent state is left behind. This is a deliberate choice — silently retrying or masking a failure here would risk exactly the correctness violations Principle 4.1 exists to prevent.
- The asynchronous, consumer-tier side (Notification, Analytics, Audit's non-blocking path, Search's index) degrades **gracefully**: a delayed notification, a temporarily stale search index, or a backlog of unprocessed analytics events are recoverable inconveniences, not business-integrity failures, and are treated accordingly (`integration-and-messaging.md` Section 8's retry strategy is what recovers them, not a page-worthy incident by itself).

**Exact numeric availability/latency targets** (an uptime percentage, a p95 latency budget for order placement) are not established in any prior document and are named in Open Decisions (Section 11) rather than invented here — this section establishes the allocation and failure-handling *philosophy*, which the eventual `nfr-matrix.md` should attach specific numbers to.

## 4. Caching Strategy

Two caching layers exist, each already fully specified elsewhere and only summarized here in their performance capacity:

- **Redis**, for read acceleration, session state, and rate limiting (`data-architecture.md` Sections 3, 6, 13) — the general-purpose cache for anything read frequently relative to how often it changes, always with PostgreSQL as the fallback source of truth (ADR-0004).
- **CDN**, for public, cacheable content — product imagery from S3 and Customer Web App static assets (`05-deployment/infrastructure-and-release.md` Section 3; `04-Technology-Stack.md` Section 11) — reducing latency for the public-facing surfaces specifically, independent of Redis's role for authenticated, dynamic requests.

**Why two layers and not one:** they solve different problems at different points in the request path. The CDN sits in front of the system entirely, serving requests that never reach the Backend at all; Redis sits inside the Backend, accelerating requests that do reach it. Collapsing them into one layer would either push public asset traffic through the Backend unnecessarily (defeating the CDN's purpose) or push authenticated, per-user data out to an edge cache inappropriate for it (a correctness and privacy risk `data-architecture.md`'s Redis boundary already rules out).

## 5. Search Strategy

Search's entire reason for existing (`module-catalog.md` Section 4.5) is a performance strategy in itself: relevance-ranked, filtered, sorted product discovery is a query shape PostgreSQL's own indexing is not the best fit for at scale, so Search maintains a derived, purpose-built index instead of every product query running against the primary transactional database. This is not a departure from Principle 4.2 — Search owns no primary data (Catalog and Inventory remain authoritative) — it is Principle 4.5's reporting/read-model exception applied to discovery rather than reporting.

**Why this matters for performance specifically:** without Search, every product listing or filter operation would compete with checkout, inventory reservation, and order-processing queries for the same PostgreSQL connections and query planner attention — exactly the read/write contention Section 6 below is designed to avoid. Search's event-driven index population (`integration-and-messaging.md` Section 7's Example 2 pattern) means query-time search performance is fully decoupled from Catalog/Inventory's own write load.

## 6. Read/Write Optimization

The general principle: **writes stay concentrated on PostgreSQL's primary instance; reads are pushed outward wherever they don't need to see the absolute latest write.** Concretely:
- Transactional reads that must see current state (checking availability before reserving stock, DDD Section 10.3's optimistic validation) read from the primary, in the same transaction as the write they're validating.
- Reporting and analytics reads (`module-catalog.md` Section 4.13) never query the primary's transactional tables directly — they read from Analytics' own rebuildable rollups, built from consumed events, specifically so a heavy reporting query can never compete with or slow down a checkout write.
- Search reads (Section 5) never touch PostgreSQL at query time at all — they read the derived index exclusively.
- As real read load materializes on paths that *do* need to query PostgreSQL directly (Catalog browsing being the likely first case), read replicas (Section 7) are the next lever — not before that evidence exists.

**Why this ordering:** Analytics and Search already remove the two largest classes of read load (reporting, discovery) from PostgreSQL's primary by architecture, not by infrastructure scaling — which is why read replicas (a scaling response) are listed after them, not before. The cheapest read optimization is one that was designed away entirely.

## 7. Queue Usage

Two distinct queue-shaped needs exist in the system, both already using infrastructure already in the stack rather than introducing a new technology (Constitution Section 3's "boring technology"):

- **Domain event dispatch** (`integration-and-messaging.md` Sections 2–11) — today, in-process within the modular monolith; the future transport question (a message broker on module extraction) is that document's Section 11 to answer, not repeated here.
- **Scheduled background jobs** (`integration-and-messaging.md` Section 6 — reservation expiry, notification retry, reconciliation, cleanup, rollup generation) — periodic work, not request-triggered, naturally implemented on top of Redis given Redis is already present in the stack for coordination (ADR-0004), rather than introducing a dedicated job-queue technology with no evidenced need for one beyond what Redis-backed scheduling already provides.

**Why not a dedicated message-queue technology today:** nothing about the system's current scale or requirements demands durability or throughput guarantees beyond what an in-process dispatcher plus Redis-backed scheduling already provides, and every consumer is already required to be idempotent (`integration-and-messaging.md` Section 9) specifically so this choice can change later without touching consumer logic — introducing a dedicated queue technology now would be exactly the kind of unearned complexity Constitution Section 13 warns against.

## 8. Horizontal Scaling

The Backend scales vertically first (`05-deployment/infrastructure-and-release.md` Section 12) and horizontally — multiple instances of the same deployable behind a load balancer — once vertical scaling is evidenced as insufficient, not preemptively. Two architectural properties make horizontal scaling straightforward when it's actually needed:

- **Stateless request handling.** JWT-based authentication (`security-and-compliance.md` Section 5) means no request depends on server-side session affinity — any Backend instance can handle any authenticated request, because the token itself carries what a sticky session would otherwise need to.
- **Shared, not per-instance, coordination.** Redis's locks, rate limits, and caches (`data-architecture.md` Sections 3, 6) are already external to any single Backend instance — adding a second instance doesn't fragment coordination state the way it would if locking or rate-limiting were implemented in-process per instance.

**What horizontal scaling does not change:** the in-process event dispatch (Section 7) would need to become cross-instance-aware once multiple Backend instances exist simultaneously — this is a real, named consequence of horizontal scaling, not a gap; it's the same transport question `integration-and-messaging.md` Section 11 already flags as a future decision, arriving slightly earlier (at horizontal scaling) than at full module extraction, but resolved the same way.

## 9. Database Scaling

PostgreSQL scales vertically first (a larger RDS instance class), consistent with Constitution Section 6's explicit ordering. Beyond that, two distinct mechanisms apply to two distinct problems:
- **Read replicas** address read load specifically (Catalog browsing being the most likely first case, per Section 6) — never write load, since a replica cannot accelerate the primary's write throughput.
- **Partitioning** addresses table *size and growth*, not load — the append-only, ever-growing tables (Inventory Ledger, Audit Log, Order Events — `data-architecture.md` Section 8, DDD Section 16.3's "Future Partitioning") are the named candidates, since their write pattern (insert-only, never updated) is exactly what partitioning by time period suits best.

Neither is introduced speculatively — both are named here as the *next* lever, to be pulled once monitoring (`05-deployment/infrastructure-and-release.md` Section 8) shows the current, simpler configuration is actually insufficient, not before.

## 10. Future Microservice Extraction

The scalability endgame this entire document builds toward: because every module is already decomposed with real boundaries and strict data ownership (Principles 4.4/4.5, enforced by ADR-0009), a module under evidenced, disproportionate load can be extracted into its own independently deployed service without redesigning its boundary (ADR-0002's own designed reversibility; `01-Architecture-Design-Specification.md` Section 15). `module-catalog.md` names Order and Inventory as the most plausible first candidates — Order because it's the orchestration center of the busiest workflow (checkout), Inventory because its reservation-locking behavior (`data-architecture.md` Section 7) is the most concurrency-sensitive part of the system. This document does not repeat that reasoning — it only confirms that everything else in this document (caching, search, read/write separation, horizontal scaling, database scaling) is designed to extend the *current* monolith's capacity specifically so extraction remains a deliberate, evidence-driven choice rather than a forced, urgent one.

## 11. Capacity Planning Principles

- **Size for the business that exists, not the business that might exist.** Launch scale is low thousands of orders per day across a handful of stores (SRD; `01-Architecture-Design-Specification.md` Section 2) — every mechanism in this document is sized to that reality first, consistent with the cost-discipline design goal (Constitution Section 5), and extended only as real usage data justifies it.
- **Monitor before scaling, not instead of it.** `05-deployment/infrastructure-and-release.md` Section 8's monitoring is what supplies the evidence Section 13's "complexity is earned" principle requires before any lever in this document (read replicas, horizontal scaling, partitioning, extraction) is pulled — a lever pulled without that evidence is a violation of Constitution Section 13, not a proactive optimization.
- **Prefer the cheapest lever that solves the actual problem.** The ordering in this document (architectural read/write separation before read replicas; vertical before horizontal; replicas before partitioning; partitioning before extraction) is not arbitrary — each step is more operationally costly than the last, and skipping ahead to a more complex lever without evidence that a simpler one is insufficient is exactly the unearned complexity this whole document's philosophy exists to prevent.

## 12. Trade-offs and Future Evolution

**Trade-offs accepted:** every mechanism in this document trades some theoretical maximum throughput for simplicity and correctness confidence at current scale — the same trade-off Constitution Section 6 names explicitly for the modular monolith as a whole, applied consistently down to caching, search, and scaling specifics here. Deferring read replicas, partitioning, and extraction until evidenced means the system will, at some point, feel real growing pains before any of these levers is pulled — an accepted cost of not over-building for a scale that may never fully materialize.

**Future evolution:** this document's Sections 3–11 are exactly the ones expected to gain concrete numbers, thresholds, and trigger conditions as real traffic data accumulates (Section 11's own principle) — the ordering and philosophy in Sections 1–2 are expected to hold regardless of what those numbers turn out to be.

## 13. Open Decisions

- **Exact availability and latency targets** (uptime percentage, p95/p99 latency budgets for order placement and other key journeys) are not established anywhere in the approved documents — Section 3 sets the allocation and failure-handling philosophy; specific numbers belong to `06-quality-attributes/nfr-matrix.md` once authored.
- **Specific horizontal-scaling and read-replica trigger thresholds** (what utilization or latency level actually justifies pulling that lever) are not decided — Section 11's "monitor before scaling" principle is established; the specific thresholds are not.
- **Whether background-job scheduling (Section 7) is Redis-backed via a lightweight library or a more structured job-processing approach** is SDD-level detail, not decided here.
- **Partitioning strategy specifics** (by what period, at what table-size threshold) for the append-only tables named in Section 9 are explicitly deferred in `data-architecture.md` and remain deferred here.
