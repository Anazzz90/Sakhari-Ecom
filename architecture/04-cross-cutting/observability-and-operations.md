# Observability & Operations — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. |
| **Stability** | Stable in philosophy and structure (severity tiers, incident-response phases, the shape of what's monitored); evolving in specific thresholds, tooling, and runbook detail as real production experience accumulates. |
| **Authority** | Subordinate to `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `decisions/README.md`, `03-decomposition/module-catalog.md` / `module-communication.md` / `service-decomposition.md`, `04-cross-cutting/reliability-and-performance.md`, and `05-deployment/infrastructure-and-release.md`. This document does not restate infrastructure topology, availability targets, or backup mechanics those documents already own — it is the operational-practice layer built on top of them. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** Defines how Sakhari Ecom is watched, diagnosed, operated day to day, and recovered when something goes wrong — the practices that turn "the system is built correctly" into "the system stays correct and available while a single developer, assisted by AI, is the only one watching it." `05-deployment/infrastructure-and-release.md` names *what* infrastructure exists and its backup/DR mechanism at the topology level; `04-cross-cutting/reliability-and-performance.md` names *what* availability and failure-handling philosophy the architecture pursues; this document is *how those get operated in practice* — the deeper layer both of those documents explicitly deferred here.

**Scope.** Operational philosophy; logging strategy; monitoring (system, application, and business health); metrics; health checks; alerting philosophy and severity; incident response; backup and disaster recovery from an operational (not topology) angle; recovery objectives; operational runbooks; maintenance windows; capacity monitoring; a production readiness checklist; and operational best practices. This document describes **operational architecture — what must be true, and why — never implementation**: no specific logging/monitoring vendor, no dashboard configuration, no literal step-by-step runbook script. Per this task's instruction, vendor-specific tooling is named only where already approved elsewhere (AWS `me-central-2` — ADR-0024; PostgreSQL — ADR-0003; Redis — ADR-0004; Moyasar — ADR-0022; Unifonic — ADR-0023) and never invented here.

**Intended audience.** The project owner, operating this system as its sole on-call; an AI coding assistant instrumenting a module's logging/metrics or reasoning about an incident; the author of any future SDD, for whom this document's standards are what every module's observability instrumentation must satisfy.

**Cross-references.** Grounded in Constitution Section 11 ("observability is not optional... especially for a solo developer with no second pair of eyes to compensate") and Section 5 (Design Goals: operational simplicity). Builds on `reliability-and-performance.md` Section 3 (availability allocation, failure-handling strategy — assumed here, not repeated), `infrastructure-and-release.md` Sections 7–10 (logging/monitoring/backup/DR at the topology level), `security-and-compliance.md` Section 9 (audit as a security control) and Section 10 (secrets), `data-architecture.md` Section 14 (data-level recovery mechanisms), and `module-catalog.md`/`module-communication.md` (the sixteen modules and their dependency graph, which this document's health checks and metrics are organized around).

## 2. Operational Philosophy

- **Observability is a correctness concern, not an add-on** (Constitution Section 11) — a production issue this system cannot diagnose from its own logs, metrics, or audit trail is a design gap, exactly as much as a missing transaction boundary would be.
- **Calibrated to one operator, not a NOC.** Every practice in this document is sized for a single developer with AI assistance, not an enterprise operations team — this shapes the alerting tiers (Section 7), the incident-response process (Section 8), and the runbook philosophy (Section 11) throughout: fewer, higher-signal alerts rather than comprehensive, high-noise coverage.
- **The architecture's own asymmetry drives where operational effort concentrates.** `reliability-and-performance.md` Section 3 already establishes that the transactional core fails loudly while the consumer tier degrades gracefully — this document's monitoring and alerting priorities (Sections 4–7) follow that same asymmetry rather than treating every module as equally urgent.
- **Nothing is observed that isn't actionable.** A metric or log field earns its place only if it would change what someone does — either now (an alert) or later (a diagnosis) — consistent with the "boring technology" and operational-simplicity convictions running through this whole documentation set.
- **The system's own audit trail and event history are an operational asset, not just a compliance one** (`data-architecture.md` Section 15) — every incident-response and diagnosis practice in this document assumes that durable record exists and leans on it rather than reconstructing history from memory or scattered logs.

## 3. Logging Strategy

- **Structured logs.** Every log entry is structured (field-based, not free-text prose), with a consistent minimum field set across all sixteen modules — timestamp, module, operation, outcome, and a correlation ID (below) — established once in `infrastructure-and-release.md` Section 7 and restated here as the non-negotiable baseline every module's instrumentation must meet. Structured logs exist specifically so a solo developer's tooling can filter and aggregate without parsing prose.
- **Log levels.** A small, consistent set of severities — error, warning, info, debug — used identically across every module, so "how loud is this" means the same thing everywhere: *error* is an unexpected failure requiring attention; *warning* is a degraded-but-recovered condition (a retried operation, a fallback path taken); *info* is a normal, expected business event (an order placed, a payment confirmed); *debug* is detail useful only during active investigation, not enabled by default in Production (`05-deployment/environment-strategy.md` Section 3's environment-varying verbosity).
- **Correlation IDs.** Every request entering the system is assigned a correlation ID at the boundary (the controller layer — `service-decomposition.md` Section 7's Thin Controllers), propagated through every module and event that participates in fulfilling it. This exists so a single customer-facing action (place an order) that fans out across Order, Inventory, Payment, and eventually Delivery and Notification can be traced as one story across module boundaries, without which a solo developer would have to manually correlate timestamps across sixteen modules' worth of independent logs.
- **Request tracing.** The correlation ID is what request tracing is built on — the ability to reconstruct, after the fact, the full sequence of synchronous calls and asynchronous events a single request triggered (`module-communication.md` Section 10's worked checkout example is exactly the kind of flow tracing must be able to reconstruct). This is diagnostic infrastructure, not a business record — it complements but never substitutes for Order Event history or the Audit Log.
- **Audit logs.** Distinct from application logs by design (`security-and-compliance.md` Section 9; `data-architecture.md` Section 15) — the Audit Log is immutable, retained long-term, and answers "who did what and was it allowed," while application logs are disposable, retained short-term, and answer "what was the system doing at this moment." This document's logging strategy governs application logs only; the Audit Log's own strategy is `data-architecture.md` Section 15 and `security-and-compliance.md` Section 9's subject.
- **Security logs.** Authentication events (successful and failed OTP verification, session issuance/revocation), authorization denials (an RBAC check that failed), and access to sensitive data are logged with enough context to detect a pattern (repeated failed authentication, unusual administrative access) without duplicating the Audit Log's role — a security log is an operational early-warning signal; the Audit Log is the durable record of what was actually authorized to happen.

## 4. Monitoring

Three distinct kinds of "is everything okay," each answering a different question and drawing on different data:

- **System health.** Is the infrastructure itself functioning — the Backend compute layer, PostgreSQL, Redis, and the network path between them (`infrastructure-and-release.md` Sections 3–4)? This is the lowest-level, most infrastructure-adjacent kind of health, and the one closest to what Section 5's Health Checks directly answer.
- **Application health.** Is the Backend's own logic behaving correctly — are modules completing their operations, are transactions committing, are events being published and consumed, is the dependency graph (`module-communication.md` Section 7) functioning as designed? This is where Section 6's metrics (error rates, latency, queue sizes) live.
- **Business health.** Is the business itself functioning — are orders actually being placed and fulfilled, are payments actually succeeding, are deliveries actually completing? This is the highest-level kind of health, and the one most directly tied to whether customers are having a good experience right now, independent of whether every underlying metric looks nominal. A system can be "application healthy" (no errors, normal latency) while being "business unhealthy" (checkout success rate has quietly dropped) — this document treats the two as genuinely distinct monitoring concerns, not one collapsing into the other.

## 5. Metrics

Each metric exists to answer a specific operational question, organized by which of Section 4's three health categories it primarily serves:

| Metric | Answers | Health Category |
|---|---|---|
| API latency | Is the customer-facing path fast enough to support the delivery promise (`reliability-and-performance.md` Section 3)? | Application |
| Checkout success rate | Is the system's single most important business flow actually completing (`module-communication.md` Section 10)? | Business |
| Inventory reservation failures | Is stock genuinely unavailable, or is the reservation mechanism itself struggling under concurrency (`data-architecture.md` Section 7)? | Application / Business |
| Payment failures | Is Moyasar (or a payment path) rejecting legitimate attempts, distinct from a customer's own card being declined? | Business |
| Notification failures | Is the one intentionally-degradable capability (`module-catalog.md` §4.12) actually degrading gracefully, or silently failing beyond its designed tolerance? | Application |
| Queue sizes | Is asynchronous event/background-job processing keeping pace, or backing up (`reliability-and-performance.md` Section 7)? | Application |
| Cache hit ratio | Is Redis actually accelerating reads, or is the cache layer providing no benefit (`data-architecture.md` Section 3)? | System / Application |
| Database performance | Is PostgreSQL — the sole source of truth for everything — within its normal operating envelope (`data-architecture.md` Section 2)? | System |
| Background job success | Are the scheduled jobs (`integration-and-messaging.md` Section 6 — reservation expiry, reconciliation, rollup generation) actually completing, since their failure is often silent otherwise? | Application |

**Why this specific set:** each metric maps to a named architectural concern elsewhere in this documentation set — none is included for generic completeness. A metric with no corresponding "why would this number matter" is exactly the kind of unactionable observability Section 2 rules out.

## 6. Health Checks

- **Liveness.** Is the Backend process itself running and able to respond at all — the most basic possible check, answering only "should this instance be restarted," never "is the system working correctly."
- **Readiness.** Is the Backend not just alive but actually able to serve traffic correctly right now — has it finished startup, and can it reach the dependencies (below) it needs for a request to succeed. A live-but-not-ready instance should receive no traffic, consistent with the Backend's stateless, horizontally-scalable design (`reliability-and-performance.md` Section 8).
- **Dependency health.** Can the Backend actually reach PostgreSQL, Redis, and (where relevant to the specific check) the external systems it depends on (`02-context/system-context.md` Section 7 — Payment Gateway, SMS/OTP Provider, Push Notification Service, Geocoding Provider)? A dependency being unreachable is distinct from the Backend itself being unhealthy, and is diagnosed and alerted on differently (Section 7) — the system should be able to tell the difference between "I am broken" and "something I depend on is broken."

## 7. Alerting Philosophy

Calibrated to a single operator (Section 2) — the goal is the fewest alerts that reliably catch every problem that actually needs a human, not maximum coverage:

| Severity | Meaning | Example | Response Expectation |
|---|---|---|---|
| **Critical** | The transactional core is failing loudly (`reliability-and-performance.md` Section 3) — checkout, payment, or inventory reservation is broken for customers right now. | Checkout success rate collapses; PostgreSQL is unreachable. | Immediate, any time of day — this is the one tier that interrupts a solo developer outside working hours. |
| **High** | A capability is degraded but the transactional core still functions, or a Critical condition is imminent. | Elevated payment failure rate below outright breakage; database performance degrading toward a threshold. | Same-day, promptly. |
| **Medium** | A consumer-tier capability (Notification, Analytics, Search) is degraded — an accepted, designed-for kind of failure (`reliability-and-performance.md` Section 3) that still deserves attention before it compounds. | Notification delivery failures rising; Search index falling behind. | Within a normal working day, not overnight. |
| **Low** | An informational signal worth knowing about but with no urgency — a trend, not an incident. | Cache hit ratio slowly declining; background job duration trending up. | Reviewed periodically, not alerted individually. |

**Escalation principles:** severity is assigned by *business consequence*, not by which module or layer is technically involved — a Critical alert is Critical because checkout is broken, not because "Order" sounds important. An alert that doesn't clearly map to one of these four tiers is a signal the alert itself needs redesigning, not that a fifth tier is needed. Nothing escalates automatically to a second on-call, because there is only one — escalation in this system means "this alert's own severity was wrong and needs recalibrating," a distinct, evolving practice as real incidents provide evidence (Section 2's "calibrated to one operator").

## 8. Incident Response

Five phases, deliberately lightweight and solo-appropriate rather than a multi-role enterprise process:

1. **Detection.** An alert (Section 7) or a direct report (a customer or Support ticket) surfaces that something is wrong. Detection is judged by how close it comes to real-time for Critical/High severities — the whole point of Sections 5–7 is that detection should not depend on a customer noticing first.
2. **Diagnosis.** Correlation IDs and request tracing (Section 3) let the operator reconstruct what actually happened for an affected request; structured logs and metrics narrow down which module or dependency is the actual point of failure, distinct from where the symptom was first observed (a Critical checkout alert might diagnose back to Inventory's reservation locking, or to PostgreSQL itself).
3. **Mitigation.** The fastest safe action that stops the immediate business harm — which is not always the same as fixing the root cause. The architecture's own compensating-action and graceful-degradation design (`module-communication.md` Section 8; `reliability-and-performance.md` Section 3) already does a meaningful share of mitigation automatically; this phase covers what a human must additionally do.
4. **Recovery.** Restoring full, correct operation — which may mean a code fix, a data correction (using the Inventory Ledger or Audit Log to determine what actually happened and what needs correcting — `data-architecture.md` Sections 8, 15), or simply confirming an automatically-recovered system is genuinely healthy again.
5. **Postmortem.** A short, blameless review (appropriate to a one-person team) of what happened, why detection/diagnosis/mitigation took as long as they did, and what — if anything — in this document's own metrics, alerts, or runbooks should change as a result. This is the mechanism by which Section 7's alert thresholds and Section 11's runbooks are expected to improve over time, per Section 2's "evolving in specific thresholds... as real production experience accumulates."

## 9. Backup Strategy

`infrastructure-and-release.md` Section 9 already establishes the mechanism — automated, managed PostgreSQL backups with point-in-time recovery, and S3's own durability guarantees for object storage — not repeated here. This document's operational layer on top of that mechanism: a backup that has never been verified restorable is not actually a backup, only an assumption. Operationally, this means periodic restore verification is a required practice (not merely a passive assumption that automated backups are working), and that the Inventory Ledger and Audit Log's own append-only, immutable design (`data-architecture.md` Sections 8, 15) serve as an operational cross-check — even a successful restore should reconcile against them, not be trusted blindly.

## 10. Disaster Recovery

`infrastructure-and-release.md` Section 10 already establishes the approach (restore from managed backup to the most recent recoverable point; single-region deployment made safe by disciplined backup practice rather than multi-region redundancy — appropriate at current scale, Constitution Section 6/13). This document's operational addition: a disaster-recovery scenario is not just "restore the database" — it is restoring the *whole* operational picture, including confirming which in-flight orders, reservations, and payments were affected and require explicit reconciliation (leaning on Section 9's ledger/audit cross-check) rather than assuming a clean restore automatically means a clean business state. The specific scope of disaster scenarios this system is expected to tolerate (a regional outage, specifically) remains `infrastructure-and-release.md`'s own Open Decision, not resolved here.

## 11. Recovery Objectives

Recovery Point Objective (RPO — how much data loss is tolerable) and Recovery Time Objective (RTO — how much downtime is tolerable) are properties that differ by data class, not one number for the whole system:

- **Transactional core data** (Orders, Payments, Inventory) demands the tightest RPO — this is exactly the data Principle 4.1 and ADR-0009/0021's transactional guarantees exist to protect, and PostgreSQL's own point-in-time recovery (`infrastructure-and-release.md` Section 9) is the mechanism that keeps RPO close to zero for anything already committed.
- **Derived/rebuildable data** (Search's index, Analytics' rollups) has an effectively unbounded RPO/RTO in the sense that it can be rebuilt from source events (`data-architecture.md` Section 11) rather than restored from backup at all — its "recovery" is really regeneration, on its own timeline, with no business consequence to it taking longer.
- **Cache and session data** (Redis) has no meaningful RPO by design (Principle 4.3) — its loss is a performance event, not a recovery event.
- **Audit and ledger data** demands the same tight RPO as the transactional core, for the same reason Section 9 treats it as a cross-check rather than optional: an unrecoverable audit trail defeats the compliance and diagnostic purpose it exists for (`security-and-compliance.md` Section 9).

**Exact numeric RPO/RTO targets** (a specific number of minutes or hours per data class) are not established in any prior document — `infrastructure-and-release.md` Section 14 already names this as an open point, deliberately not asserted without a documented basis. This document establishes the *framework* (which data classes need which kind of guarantee, and why) rather than inventing numbers; see Section 17.

## 12. Operational Runbooks

A runbook, in this document's sense, is a named, repeatable response to a known category of operational situation — described here by what categories exist and what each must cover, never as a literal step-by-step script (per this task's instruction against implementation procedures):

- **Critical-alert response runbooks**, one per Critical scenario named in Section 7 (checkout failure, database unreachability) — each must state how to confirm the alert is real (not a monitoring false positive), the safe mitigation options available given the architecture's own compensation design (`module-communication.md` Section 8), and when to declare the incident resolved.
- **Dependency-outage runbooks** — what changes operationally when an external system (`02-context/system-context.md` Section 7) is degraded or unreachable, given each one's already-designed trust and validation treatment (`security-and-compliance.md` Section 8).
- **Data-correction runbooks** — how to safely use the Inventory Ledger, Order Events, and Audit Log (Section 9's cross-check role) to determine and apply a correction, consistent with the append-only, corrections-are-additive discipline those entities already require (`data-architecture.md` Sections 8–9).
- **Restore-verification runbooks** — the periodic practice named in Section 9, confirming a backup is genuinely restorable, not just assumed to be.

Each runbook's actual content (the literal steps) is implementation, not architecture, and belongs wherever operational procedures are maintained outside this documentation set — this document's job is only to establish that these categories must exist and what each is accountable for covering.

## 13. Maintenance Windows

Because the API is versioned from day one (ADR-0017) and the Backend is designed for rolling, artifact-consistent deployment (`infrastructure-and-release.md` Section 11's release strategy), planned maintenance does not require a customer-visible downtime window for an ordinary release — this is a direct, deliberate payoff of that architectural choice, not a separate operational capability bolted on. A maintenance window in the traditional sense (a scheduled period of reduced or no availability) is reserved for the rarer operations that genuinely require it — a database-level change that can't be made online, or an infrastructure change outside the Backend's own control — and is communicated in advance precisely because it is the exception, not the routine, consistent with the versioned-API discipline's whole purpose.

## 14. Capacity Monitoring

`reliability-and-performance.md` Section 11 already establishes the capacity-planning *principles* (size for the business that exists, monitor before scaling, prefer the cheapest sufficient lever) — this document's operational counterpart is what's actually watched to know when those principles' "evidence" threshold has been met: sustained trends (not momentary spikes) in database performance, queue sizes, and cache hit ratio (Section 5) are the leading indicators that a scaling lever (`reliability-and-performance.md` Sections 8–9 — horizontal scaling, read replicas, partitioning) is actually justified, distinct from the alerting thresholds in Section 7, which are about acute problems, not capacity trends.

## 15. Production Readiness Checklist

Before a module or feature is considered ready for Production traffic:

- [ ] Structured logging with the required field set (Section 3) is implemented and includes correlation IDs.
- [ ] The metrics relevant to this module's own health category (Section 5) are emitted.
- [ ] Liveness, readiness, and (where the module has one) dependency health checks are implemented (Section 5).
- [ ] Any new Critical or High-severity failure mode this module introduces has a corresponding alert (Section 7) and a runbook category (Section 12).
- [ ] The module's audit-worthy actions (per `security-and-compliance.md` Section 9 and DDD §12.6) are actually recorded, not just planned.
- [ ] The module satisfies `coding-standards.md`'s Code Review and Architecture Compliance checklists — this document does not duplicate those, only adds the operational items above.

## 16. Operational Best Practices

- **Prefer a metric that already exists over a new one** — Section 5's set was chosen because each maps to a real architectural concern; a proliferation of low-value metrics defeats Section 2's "nothing is observed that isn't actionable" as surely as having none at all.
- **Treat a postmortem's output as binding, not optional** — Section 8's fifth phase is what keeps this whole document evolving correctly; a postmortem that doesn't change anything (a threshold, a runbook, a missing alert) has not actually closed the incident.
- **Never let an alert's absence be mistaken for health** — a Critical path with no alert covering a specific failure mode is not "fine," it's an undiscovered gap in Section 7's coverage, and should be treated with the same seriousness as a bug in the business logic it's meant to watch.
- **Keep the operational surface as small as the architecture itself is** — exactly as the modular monolith (ADR-0002) and the technology stack (`04-Technology-Stack.md`) stay deliberately unglamorous and boring, the operational practices in this document favor the fewest, highest-signal tools and processes a solo developer can actually sustain, over comprehensive coverage that would go unmaintained.

## 17. Open Decisions

- **Exact numeric RPO/RTO targets per data class** (Section 11) — not established anywhere; this document provides the framework, `infrastructure-and-release.md` Section 14 already names the specific numbers as open, and both remain open together.
- **Exact alert thresholds** (what specific latency, error rate, or queue-size number triggers each severity in Section 7) are not decided — Section 7 establishes the tiers and their meaning; specific numbers are expected to be set and tuned against real production data, per Section 8's postmortem practice.
- **Specific monitoring/logging/alerting tooling** is explicitly out of scope per this task's own instruction — this document describes what must be observed and why, not which vendor or platform observes it; `infrastructure-and-release.md` Section 14 already names this as open.
- **Maintenance-window communication mechanics** (how far in advance, through which channel) for the rare case Section 13 describes are not decided — an operational-procedure detail below this document's architectural altitude.
- **Restore-verification cadence** (Section 9 — how often a backup restore is actually tested) is not decided — a frequency is an operational parameter to be set once real backup volume and risk tolerance are better understood, not asserted here without a documented basis.
