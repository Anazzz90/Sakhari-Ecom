# Non-Functional Requirements Matrix — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. |
| **Stability** | Stable in structure (which quality attributes matter and why); evolving in specific numeric targets as real production data becomes available — see the notice in Section 2. |
| **Authority** | Subordinate to every document it cites. This matrix asserts no new architecture — every "Architectural Decisions Supporting It" entry points to a decision made elsewhere; this document's own contribution is the synthesis, not new content. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** The single, comprehensive scorecard for every non-functional requirement (quality attribute) the architecture is judged against — what each attribute means for Sakhari Ecom specifically, why it matters to the business, which architectural decisions already support it, how it would be measured, and what's still open. Where `04-cross-cutting/reliability-and-performance.md` explains the performance/scalability *approach* and `04-cross-cutting/observability-and-operations.md` explains the operational *practice*, this document is the *master reference* — one place that answers "what does 'good' mean for this system, across every quality dimension, and where did that answer come from."

**Scope.** Seventeen quality attributes (Performance, Availability, Reliability, Scalability, Maintainability, Modularity, Security, Auditability, Recoverability, Usability, Accessibility, Observability, Portability, Extensibility, Interoperability, Testability, Compliance), each with Description, Architectural Goal, Business Importance, Architectural Decisions Supporting It, Measurement Criteria, Acceptance Criteria, Risks, Trade-offs, and Future Considerations. Plus: consolidated Performance, Availability, Recovery, Security, Maintainability, Scalability, Deployment, and Operational target/objective sections; a Risk Assessment; and an Architecture Traceability matrix.

**Intended audience.** The project owner, deciding what "production-ready" means before launch; an AI coding assistant checking whether a proposed change would regress a quality attribute the architecture already commits to; a future QA engineer or auditor needing one document that maps every quality concern to its architectural backing.

**Cross-references.** This document draws from and does not duplicate: `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, the full ADR set (`decisions/`, 0001–0028), `03-decomposition/` (module-catalog, module-communication, capability-boundary-map, data-ownership-map, service-decomposition), `04-cross-cutting/` (data-architecture, security-and-compliance, integration-and-messaging, technology-decisions, reliability-and-performance, observability-and-operations), `05-deployment/` (environment-strategy, infrastructure-and-release), and `07-coding-standards/` / `08-ai-development/`.

## 2. A Note on Numeric Targets

Per this task's own instruction, this document does not invent numeric SLAs that have not already been approved. Across the entire architecture documentation set, **no document has yet established a specific numeric target** for latency, uptime percentage, RPO/RTO, or throughput — `reliability-and-performance.md` Section 13, `infrastructure-and-release.md` Section 14, and `observability-and-operations.md` Section 17 each independently and explicitly leave these open, for the same reason: asserting a number with no documented basis would itself be an architectural invention this project has consistently avoided. Every "Measurement Criteria" and "Acceptance Criteria" field below that would otherwise need a number instead states plainly: **not yet numerically defined; to be finalized before production.** This is a genuine, load-bearing finding of this matrix, not a gap in its authoring — see Section 12 (Risk Assessment) for what that gap itself means as a production-readiness risk.

## 3. Quality Attribute Matrix

### 3.1 Performance

| Field | Detail |
|---|---|
| **Description** | How quickly the system responds, particularly on the customer-facing checkout path. |
| **Architectural Goal** | Correctness-bounded speed (Principle 4.1): the synchronous transactional core stays fast because everything non-critical is pushed to asynchronous events (ADR-0011). |
| **Business Importance** | Directly supports the SRD's 10–20 minute delivery promise; checkout latency has a direct, immediate relationship to conversion. |
| **Architectural Decisions Supporting It** | ADR-0011 (event-driven side effects), ADR-0004 (Redis caching), ADR-0028 (CDN), ADR-0002 (in-process module calls, no internal network hops); `reliability-and-performance.md` Sections 2, 4, 6. |
| **Measurement Criteria** | API latency (p95/p99) on the checkout path and other key journeys — see Section 4. |
| **Acceptance Criteria** | Not yet numerically defined; to be finalized before production (Section 2). |
| **Risks** | Undefined targets mean a performance regression may go undetected until customers notice; premature optimization ahead of proven correctness is a named anti-pattern (Constitution Section 10). |
| **Trade-offs** | Correctness is prioritized over raw performance by design (Principle 4.1) — a deliberate, accepted cost. |
| **Future Considerations** | Read replicas, horizontal scaling, partitioning as evidenced need arises (`reliability-and-performance.md` Sections 8–9). |

### 3.2 Availability

| Field | Detail |
|---|---|
| **Description** | The proportion of time the system correctly serves requests. |
| **Architectural Goal** | Allocated asymmetrically: the transactional core is protected; the consumer tier (Notification, Analytics, Search) is allowed to degrade gracefully (`reliability-and-performance.md` Section 3). |
| **Business Importance** | Direct revenue impact (no orders can be taken while down) and customer trust. |
| **Architectural Decisions Supporting It** | ADR-0002 (single, simple deployable), ADR-0003/ADR-0004 (managed services remove a class of operational failure); `infrastructure-and-release.md` Sections 9–10; `observability-and-operations.md` Section 7 (alerting). |
| **Measurement Criteria** | Uptime percentage, dependency-health signal (`observability-and-operations.md` Section 5). |
| **Acceptance Criteria** | Not yet numerically defined; to be finalized before production. |
| **Risks** | Single-region deployment carries regional-outage exposure, accepted at current scale (Constitution Section 13). |
| **Trade-offs** | Single-region operational simplicity over multi-region redundancy. |
| **Future Considerations** | Multi-region deployment if evidence (a sustained, costly outage) demands it — not planned speculatively. |

### 3.3 Reliability

| Field | Detail |
|---|---|
| **Description** | The system consistently produces correct results and never silently corrupts or loses business data. |
| **Architectural Goal** | Correctness-first (Principle 4.1); transactional integrity (ADR-0009, ADR-0010, ADR-0021); mandatory idempotency for retry-sensitive workflows (`data-architecture.md` Section 13). |
| **Business Importance** | Financial and inventory correctness is existential — a double charge or an oversold item destroys trust immediately and may carry legal/financial consequence. |
| **Architectural Decisions Supporting It** | ADR-0009, ADR-0010, ADR-0021, ADR-0015 (exact-integer money), ADR-0019 (ULIDs); `data-architecture.md` throughout. |
| **Measurement Criteria** | Named race-condition and idempotency test coverage (`coding-standards.md` Section 8); checkout success rate, inventory reservation failure rate (`observability-and-operations.md` Section 5). |
| **Acceptance Criteria** | Zero tolerance for financial or inventory data corruption; specific failure-rate thresholds not yet numerically defined. |
| **Risks** | The compensating-action window in orchestrated checkout (`module-communication.md` Section 8) is a brief, bounded period of not-yet-confirmed state — correctly bounding and surfacing it is a real implementation risk, not just a design one. |
| **Trade-offs** | Reliability is prioritized over raw throughput on every correctness-critical path. |
| **Future Considerations** | Structured failure-injection ("chaos") testing once the system is mature enough to justify the investment. |

### 3.4 Scalability

| Field | Detail |
|---|---|
| **Description** | Ability to absorb growth in order volume, store count, or catalog size without an architectural redesign. |
| **Architectural Goal** | Vertical scaling first, then read replicas/horizontal scaling/partitioning, then module extraction — in that order, pulled only by evidence (`reliability-and-performance.md` Sections 8–10). |
| **Business Importance** | Supports store-count and order-volume growth without a costly rewrite blocking the business. |
| **Architectural Decisions Supporting It** | ADR-0002/ADR-0009 (extractability by design); `reliability-and-performance.md` entirely; `service-decomposition.md` Section 6 (extraction mechanics). |
| **Measurement Criteria** | Sustained capacity-trend monitoring (`observability-and-operations.md` Section 14) — database performance, queue sizes, cache hit ratio. |
| **Acceptance Criteria** | Not yet numerically defined (no specific order-volume ceiling established anywhere); the qualitative bar is "sized for the business that exists" (Constitution Section 5). |
| **Risks** | Scaling too early wastes a solo developer's limited time; scaling too late risks customer-facing degradation — both are named, opposite failure modes. |
| **Trade-offs** | Cost discipline and simplicity today over scale headroom that may never be needed (Constitution Section 6). |
| **Future Considerations** | Order and Inventory extraction, read replicas, and table partitioning are all pre-identified, evidence-gated candidates (`reliability-and-performance.md` Section 10; `service-decomposition.md` Section 13). |

### 3.5 Maintainability

| Field | Detail |
|---|---|
| **Description** | How easily the system can be understood, changed, and fixed over time — specifically by one developer with AI assistance and no institutional memory across sessions. |
| **Architectural Goal** | A modular monolith with a uniform internal structure per module, and documentation that substitutes for memory (Constitution Section 9). |
| **Business Importance** | Existential for a solo-developer team — an unmaintainable system means the business cannot evolve at all. |
| **Architectural Decisions Supporting It** | ADR-0002, ADR-0009, ADR-0020; `coding-standards.md`, `08-ai-development/ai-development-rules.md`, and the entire `/architecture` set itself. |
| **Measurement Criteria** | Code Review and Architecture Compliance checklist pass rate (`coding-standards.md` Sections 10–11); documentation currency (is `module-catalog.md` kept in sync with reality). |
| **Acceptance Criteria** | Every module follows the uniform internal shape (`service-decomposition.md` Section 3); every Open Decision is tracked, not silently ignored. |
| **Risks** | Documentation drift — a module changing without its catalog entry being updated is explicitly named as unfinished work (`coding-standards.md` Section 9). |
| **Trade-offs** | Upfront documentation and review discipline cost, in exchange for long-term change safety (Constitution Section 6). |
| **Future Considerations** | Per-module SDDs as implementation begins, each held to this same documentation-currency standard. |

### 3.6 Modularity

| Field | Detail |
|---|---|
| **Description** | The degree to which the system is decomposed into independent, loosely-coupled units with enforced boundaries. |
| **Architectural Goal** | Sixteen modules, strict ownership, no cross-module repository access (ADR-0009, ADR-0020). |
| **Business Importance** | Enables independent evolution of business capabilities and, eventually, a larger team working without constant coordination. |
| **Architectural Decisions Supporting It** | ADR-0002, ADR-0009, ADR-0020; `module-catalog.md`, `module-communication.md`, `capability-boundary-map.md`, `service-decomposition.md`. |
| **Measurement Criteria** | Compliance with the service dependency matrix (`service-decomposition.md` Section 12); Architecture Compliance checklist results. |
| **Acceptance Criteria** | Zero cross-module repository access violations; the dependency graph remains acyclic. |
| **Risks** | Boundary erosion over time — today enforced by code review, not automated tooling (`module-communication.md` Section 9). |
| **Trade-offs** | More upfront structure than an undifferentiated monolith would require. |
| **Future Considerations** | Automated boundary-enforcement tooling (a lint rule or similar) once the codebase is large enough to justify building it. |

### 3.7 Security

| Field | Detail |
|---|---|
| **Description** | Protection of identity, data, and transactions from unauthorized access or manipulation. |
| **Architectural Goal** | Centralized authentication and authorization, OTP-only identity, fine-grained RBAC, isolated secrets (`security-and-compliance.md`). |
| **Business Importance** | Protects customer PII and payment data and the platform's regulatory standing — a breach is potentially existential. |
| **Architectural Decisions Supporting It** | ADR-0012, ADR-0026, ADR-0027; `security-and-compliance.md` entirely. |
| **Measurement Criteria** | Security-log and audit coverage; RBAC permission-check coverage; secrets-scan compliance (part of `coding-standards.md`'s review checklist). |
| **Acceptance Criteria** | No hardcoded secrets, no bypassed RBAC checks, every consequential action audited (DDD §12.6's defined scope). |
| **Risks** | JWT signing-key compromise has a system-wide blast radius (`security-and-compliance.md` Section 10's own named risk); SMS-based OTP carries known interception risk, accepted at this stage (`04-Technology-Stack.md` Section 9). |
| **Trade-offs** | Centralizing authorization creates one high-value target requiring strong protection, in exchange for avoiding the worse alternative of inconsistent, scattered security logic. |
| **Future Considerations** | App-based authenticators or passkeys if evidence justifies moving beyond SMS OTP. |

### 3.8 Auditability

| Field | Detail |
|---|---|
| **Description** | The ability to reconstruct who did what, when, and why, across the whole system. |
| **Architectural Goal** | An append-only Audit Log, Inventory Ledger, and Order Event history, owned by a dedicated Audit module (Principle 4.10). |
| **Business Importance** | Regulatory compliance (SAMA, PDPL), dispute resolution, and incident diagnosis all depend on this existing. |
| **Architectural Decisions Supporting It** | ADR-0009 (Audit's ownership boundary); `data-architecture.md` Section 15; `security-and-compliance.md` Section 9. |
| **Measurement Criteria** | Percentage of DDD §12.6-designated consequential actions that actually produce an audit entry. |
| **Acceptance Criteria** | Every action DDD §12.6 names as audit-required is recorded with actor, action, affected entity, and timestamp. |
| **Risks** | An audit-recording failure could go unnoticed unless it is itself monitored (`observability-and-operations.md` Section 5). |
| **Trade-offs** | Real, ongoing write-volume and storage cost, accepted for the explainability it buys. |
| **Future Considerations** | Partitioning/archival of audit data as volume grows (`data-architecture.md` Section 8) — a scaling response, not a boundary change. |

### 3.9 Recoverability

| Field | Detail |
|---|---|
| **Description** | Ability to restore correct operation after a failure, at both the data and infrastructure level. |
| **Architectural Goal** | Compensating actions and reconciliation jobs at the data layer (`data-architecture.md` Section 14); managed, automated backups with point-in-time recovery at the infrastructure layer (`infrastructure-and-release.md` Section 9). |
| **Business Importance** | Minimizes data loss and downtime impact when something does go wrong. |
| **Architectural Decisions Supporting It** | ADR-0021 (compensation model); `infrastructure-and-release.md` Sections 9–10; `observability-and-operations.md` Sections 9–11. |
| **Measurement Criteria** | RPO/RTO per data class (`observability-and-operations.md` Section 11) — see Section 6 below. |
| **Acceptance Criteria** | A restore-verification practice exists and is periodically exercised; exact cadence not yet defined. |
| **Risks** | An unverified backup is not a real backup (`observability-and-operations.md` Section 9's own framing). |
| **Trade-offs** | Single-region simplicity over multi-region resilience, at current scale. |
| **Future Considerations** | Multi-region disaster recovery if evidence demands it. |

### 3.10 Usability

| Field | Detail |
|---|---|
| **Description** | How easily the intended actors (customers, workers, admins) accomplish their goals through the system's clients. |
| **Architectural Goal** | The architecture stays out of usability's way: presentation belongs entirely to the clients (Principle 4.6), and centralizing business logic guarantees the same behavior everywhere a customer might encounter it. |
| **Business Importance** | Directly drives conversion (customers) and productivity (workers/admins). |
| **Architectural Decisions Supporting It** | ADR-0006/ADR-0007/ADR-0008 (client framework choices, chosen partly for UX fit to each audience); Principle 4.6/4.7 (consistency guarantee). |
| **Measurement Criteria** | Not an architecture-layer metric — belongs to product/design measurement; the architecture's own contribution is measured indirectly through Performance (Section 3.1) and cross-client behavioral consistency. |
| **Acceptance Criteria** | No client-specific business-logic divergence (Principle 4.6) — this is the architecture's entire usability contract. |
| **Risks** | A slow or inconsistent backend degrades usability regardless of how well-designed a client is. |
| **Trade-offs** | Not applicable at this layer. |
| **Future Considerations** | Usability targets, research, and testing belong to product/design documentation outside `/architecture`'s scope. |

### 3.11 Accessibility

| Field | Detail |
|---|---|
| **Description** | Usability by people with disabilities — screen-reader support, sufficient contrast, motor-impairment accommodation, and similar. |
| **Architectural Goal** | Not an architecture-layer concern — accessibility is realized entirely in client-side implementation (Customer Mobile App, Customer Web App, Worker App, Admin/Ops Dashboard), which this documentation set does not design. |
| **Business Importance** | Legal and ethical inclusivity; broader market reach. |
| **Architectural Decisions Supporting It** | None — no ADR addresses accessibility, and none should, since it belongs to client design standards, not backend architecture. |
| **Measurement Criteria** | Not defined anywhere in the approved architecture documents. |
| **Acceptance Criteria** | Not defined anywhere in the approved architecture documents. |
| **Risks** | The absence of any documented accessibility standard is itself a real gap — see Section 12. |
| **Trade-offs** | Not applicable at this layer. |
| **Future Considerations** | A dedicated accessibility standard should be established at the product/client design level before or alongside production launch — flagged here, not resolved here. |

### 3.12 Observability

| Field | Detail |
|---|---|
| **Description** | The ability to understand the system's internal state from its external outputs — logs, metrics, and traces. |
| **Architectural Goal** | Structured logging with correlation IDs, three-tier monitoring, and calibrated alerting (`observability-and-operations.md` entirely). |
| **Business Importance** | Lets a solo developer detect and diagnose issues before, or as, customers notice them. |
| **Architectural Decisions Supporting It** | Constitution Section 11; `observability-and-operations.md`; `reliability-and-performance.md` Section 3. |
| **Measurement Criteria** | Metrics catalog coverage (`observability-and-operations.md` Section 5); alert coverage of every Critical-severity path (Section 7 of that document). |
| **Acceptance Criteria** | Every module satisfies the Production Readiness Checklist's logging/metrics/health-check items (`observability-and-operations.md` Section 15). |
| **Risks** | Alert fatigue or, conversely, gaps, if severity thresholds are miscalibrated. |
| **Trade-offs** | Instrumentation adds real, ongoing development overhead per module. |
| **Future Considerations** | Specific monitoring/logging tooling selection remains an open decision (`observability-and-operations.md` Section 17). |

### 3.13 Portability

| Field | Detail |
|---|---|
| **Description** | Ease of moving the system, or a piece of it, to a different underlying technology or cloud provider. |
| **Architectural Goal** | Standard interfaces (SQL, object key/URL) over deep provider-proprietary APIs; module boundaries make internal portability (e.g., swapping Search's indexing technology) low-risk by design. |
| **Business Importance** | Reduces the cost and risk of vendor lock-in, provider-specific outages, or unfavorable pricing changes. |
| **Architectural Decisions Supporting It** | ADR-0003, ADR-0016, ADR-0024; `04-Technology-Stack.md` Sections 10–11. |
| **Measurement Criteria** | Not formally measured — a qualitative assessment of how much provider-proprietary API surface any module actually depends on. |
| **Acceptance Criteria** | No module depends on a cloud-provider-proprietary API beyond ordinary provisioning (`04-Technology-Stack.md` Section 10). |
| **Risks** | Real switching cost still exists despite standard interfaces — `04-Technology-Stack.md` Section 10 itself calls this "a one-way-door decision in practice." |
| **Trade-offs** | Single-provider operational simplicity now, against portability effort later. |
| **Future Considerations** | Provider change is only reconsidered under sustained, evidenced compliance, cost, or reliability failure — not routinely. |

### 3.14 Extensibility

| Field | Detail |
|---|---|
| **Description** | Ease of adding new business capability without redesigning existing architecture. |
| **Architectural Goal** | A formal, ADR-gated process for adding a new capability (`capability-boundary-map.md` Section 7), rather than an existing module's boundary quietly stretching. |
| **Business Importance** | Supports business growth — loyalty programs, marketplace, multi-country expansion, dynamic pricing (all named in DDD Section 17) — without an architectural rewrite. |
| **Architectural Decisions Supporting It** | `capability-boundary-map.md` Section 7; ADR-0009/ADR-0020's own precedent for how a module boundary is formally added or changed. |
| **Measurement Criteria** | Not formally measured — judged qualitatively by whether a proposed new capability satisfies Section 7's six-point test without exception. |
| **Acceptance Criteria** | A new capability follows the documented process; no capability is ever added by silently expanding an existing module's responsibility. |
| **Risks** | Boundary "stretching" without going through governance — explicitly named as an architecture violation (`capability-boundary-map.md` Section 6's loyalty-points example). |
| **Trade-offs** | Governance overhead against the speed of shipping a new feature. |
| **Future Considerations** | Loyalty, marketplace, multi-country, batch delivery, and dynamic pricing are all pre-named future candidates (DDD Section 17), none committed to yet. |

### 3.15 Interoperability

| Field | Detail |
|---|---|
| **Description** | The system's ability to work correctly with external systems and standards. |
| **Architectural Goal** | Every external integration is owned by exactly one module and validated at the boundary before being trusted (`02-context/system-context.md` Section 7; `security-and-compliance.md` Section 8). |
| **Business Importance** | Payment (Moyasar) and SMS (Unifonic) integrations must work reliably with real external partners for the business to function at all. |
| **Architectural Decisions Supporting It** | ADR-0022, ADR-0023, ADR-0017 (API versioning also protects interoperability with the platform's own four clients). |
| **Measurement Criteria** | External integration success/failure rates (`observability-and-operations.md` Section 5 — payment failures, notification failures). |
| **Acceptance Criteria** | Every external response is validated before being trusted; no client ever calls an external system directly. |
| **Risks** | Terminal/POS integration for card-on-delivery reconciliation remains an explicitly unresolved "Phase 1 spike" (`module-catalog.md` §4.9). |
| **Trade-offs** | One provider per integration, for simplicity, over redundancy across multiple providers. |
| **Future Considerations** | Terminal/POS resolution; the still-open question of whether geocoding is client-embedded or backend-mediated (`02-context/system-context.md`). |

### 3.16 Testability

| Field | Detail |
|---|---|
| **Description** | Ease of verifying, through automated and manual testing, that the system behaves correctly. |
| **Architectural Goal** | Tests target public interfaces, not internals (`coding-standards.md` Section 8); correctness-critical and idempotency-sensitive paths receive the most rigorous coverage, deliberately unevenly. |
| **Business Importance** | Confidence in every change without a dedicated QA team — critical given the solo-developer, AI-assisted delivery model. |
| **Architectural Decisions Supporting It** | `coding-standards.md` Section 8; `08-ai-development/ai-development-rules.md` Section 9. |
| **Measurement Criteria** | Test coverage weighted by correctness criticality — deliberately not a single, uniform coverage percentage (`coding-standards.md` Section 13's own refusal to assert a threshold with no basis). |
| **Acceptance Criteria** | Every named race condition and idempotency-required workflow (`data-architecture.md` Section 13) has an explicit test proving the guarantee, not just the happy path. |
| **Risks** | No specific test framework has been selected yet (`coding-standards.md` Section 13's Open Decision), which could delay establishing real coverage discipline. |
| **Trade-offs** | Rigor concentrated on critical paths, deliberately less on low-risk modules — an intentional imbalance, not an oversight. |
| **Future Considerations** | Test framework selection and a coverage-threshold policy, once real experience justifies specific numbers. |

### 3.17 Compliance

| Field | Detail |
|---|---|
| **Description** | Adherence to relevant legal, regulatory, and payment-industry obligations. |
| **Architectural Goal** | Every named regulatory obligation mapped to a specific architectural control (`security-and-compliance.md` Section 11); data residency enforced structurally (ADR-0024). |
| **Business Importance** | A legal operating requirement for doing business in Saudi Arabia — non-negotiable, never traded off against another quality attribute. |
| **Architectural Decisions Supporting It** | ADR-0024 (`me-central-2` residency), ADR-0022 (Moyasar/mada); `security-and-compliance.md` Section 11's compliance-mapping table. |
| **Measurement Criteria** | Coverage of the obligation-to-control mapping table (`security-and-compliance.md` Section 11) — is every named obligation still mapped to a real, implemented control. |
| **Acceptance Criteria** | Every SAMA, PDPL, and mada obligation named in the SRD has a corresponding architectural control in that mapping table. |
| **Risks** | New or clarified regulatory requirements could emerge that the current mapping doesn't yet cover; this document's mapping is architecture's own assessment, not a substitute for formal legal review. |
| **Trade-offs** | None — compliance obligations are treated as constraints the architecture must satisfy, never as something weighed against convenience. |
| **Future Considerations** | A formal legal/compliance review before production launch is explicitly recommended — this documentation set is an engineering artifact, not legal counsel. |

## 4. Performance Targets

| Target | Status |
|---|---|
| API latency (p95/p99), checkout and other key journeys | Not yet numerically defined; to be finalized before production (`reliability-and-performance.md` Section 13). |
| Checkout success rate | Not yet numerically defined; monitored per `observability-and-operations.md` Section 5. |
| Cache hit ratio | Not yet numerically defined; monitored, not target-gated at this stage. |

## 5. Availability Targets

| Target | Status |
|---|---|
| System uptime percentage | Not yet numerically defined; to be finalized before production (`infrastructure-and-release.md` Section 14). |
| Planned-maintenance downtime | Effectively none for ordinary releases, by design (ADR-0017's versioned APIs; `observability-and-operations.md` Section 13) — no target needed for the common case. |

## 6. Recovery Targets

| Data Class | RPO Approach | RTO Approach | Numeric Target |
|---|---|---|---|
| Transactional core (Orders, Payments, Inventory) | Near-zero, via PostgreSQL point-in-time recovery | Restore-driven | Not yet numerically defined |
| Derived/rebuildable (Search index, Analytics rollups) | Effectively unbounded — regenerated from source events, not restored | Regeneration time, not restore time | Not applicable as a restore target |
| Cache/session (Redis) | Not meaningful by design (Principle 4.3) | Not meaningful by design | Not applicable |
| Audit/ledger | Same as transactional core — treated as a cross-check on recovery itself | Restore-driven | Not yet numerically defined |

Full framework: `observability-and-operations.md` Section 11.

## 7. Security Objectives

- Centralize authentication and authorization; never re-implement either per module or per client (`security-and-compliance.md` Section 2).
- No password-based credentials anywhere in the system (ADR-0027).
- Every consequential action is attributable and auditable (Principle 4.10).
- Every external response is validated before being trusted, symmetrically with inbound client validation (`security-and-compliance.md` Section 8).
- Secrets are never in code or version control, and are scoped to the single module that needs them (`security-and-compliance.md` Section 10).

## 8. Maintainability Objectives

- One uniform internal structure across all sixteen modules (`service-decomposition.md` Section 3).
- Documentation is part of "done" — a module change without a corresponding documentation update is unfinished work (`coding-standards.md` Section 9).
- Every significant, hard-to-reverse decision is recorded as an ADR before or alongside the code that implements it.
- An AI coding assistant, cold in a new session, should be able to act correctly using only this documentation set (Constitution Section 9).

## 9. Scalability Objectives

- Vertical scaling before horizontal; read replicas before partitioning; partitioning before module extraction (`reliability-and-performance.md` Sections 8–10).
- No scaling lever pulled without evidenced need — monitored, never speculative (`reliability-and-performance.md` Section 11).
- Every module boundary is drawn so the module behind it could be extracted into its own service without a redesign (ADR-0002/ADR-0009).

## 10. Deployment Objectives

- Three environments — Development, Staging, Production — with no environment skipped in the promotion path (`environment-strategy.md` Sections 2, 4).
- Every environment holds distinct, non-reused credentials for every external integration (`environment-strategy.md` Section 3).
- A single Backend artifact is what's verified in Staging and what's promoted to Production, unmodified (`infrastructure-and-release.md` Section 11).

## 11. Operational Objectives

- Observability is a correctness concern, not an add-on (Constitution Section 11).
- Alerting and incident response are calibrated to a single operator, not an enterprise team (`observability-and-operations.md` Sections 2, 7–8).
- A postmortem's findings are binding — they change a threshold, a runbook, or a missing alert, not just a written record (`observability-and-operations.md` Section 16).

## 12. Risk Assessment

| Risk | Affected Quality Attributes | Severity (Qualitative) | Mitigation Status |
|---|---|---|---|
| No numeric SLA targets exist yet for latency, availability, or recovery | Performance, Availability, Recoverability | High — blocks a genuine production-readiness signoff | Explicitly named as required pre-production work in three independent documents (Section 2) |
| Module boundaries are enforced by code review only, not tooling | Modularity, Maintainability | Medium — real but bounded by a solo developer's own discipline and this documentation set | Automated tooling named as future work (`module-communication.md` Section 9) |
| JWT signing-key compromise has a system-wide blast radius | Security | High — a single point of catastrophic failure if mismanaged | Key management scoped as the most tightly-controlled credential in the system (`security-and-compliance.md` Section 10) |
| No accessibility standard is documented anywhere | Accessibility, Usability | Medium — a real gap, not yet even flagged elsewhere in the project | Named here for the first time; recommended as pre-launch product/design work |
| Terminal/POS integration for card-on-delivery is unresolved | Interoperability, Reliability | Low at current scale — COD/CoD already function without it | Explicitly deferred as a "Phase 1 spike" (`module-catalog.md` §4.9) |
| Single-region deployment | Availability, Recoverability | Medium — accepted trade-off, not an oversight | Documented and deliberate (Constitution Section 13); revisit only under evidenced pressure |
| No formal legal/compliance review has been performed | Compliance | High — architecture's own compliance mapping is not a substitute for legal signoff | Explicitly recommended before production (Section 3.17) |

## 13. Architecture Traceability

| Quality Attribute | Primary Supporting ADRs | Primary Supporting Documents |
|---|---|---|
| Performance | ADR-0002, ADR-0004, ADR-0011, ADR-0028 | `reliability-and-performance.md` |
| Availability | ADR-0002, ADR-0003, ADR-0004 | `infrastructure-and-release.md`, `observability-and-operations.md` |
| Reliability | ADR-0009, ADR-0010, ADR-0015, ADR-0019, ADR-0021 | `data-architecture.md` |
| Scalability | ADR-0002, ADR-0009 | `reliability-and-performance.md`, `service-decomposition.md` |
| Maintainability | ADR-0002, ADR-0009, ADR-0020 | `coding-standards.md`, `ai-development-rules.md` |
| Modularity | ADR-0002, ADR-0009, ADR-0020 | `module-catalog.md`, `module-communication.md`, `capability-boundary-map.md`, `service-decomposition.md` |
| Security | ADR-0012, ADR-0026, ADR-0027 | `security-and-compliance.md` |
| Auditability | ADR-0009 | `data-architecture.md` §15, `security-and-compliance.md` §9 |
| Recoverability | ADR-0021 | `infrastructure-and-release.md`, `observability-and-operations.md` |
| Usability | ADR-0006, ADR-0007, ADR-0008 | (product/design documentation, outside `/architecture`) |
| Accessibility | None | (not yet addressed anywhere) |
| Observability | — | `observability-and-operations.md` |
| Portability | ADR-0003, ADR-0016, ADR-0024 | `04-Technology-Stack.md` |
| Extensibility | ADR-0009, ADR-0020 | `capability-boundary-map.md` §7 |
| Interoperability | ADR-0017, ADR-0022, ADR-0023 | `02-context/system-context.md` |
| Testability | — | `coding-standards.md`, `ai-development-rules.md` |
| Compliance | ADR-0024, ADR-0022 | `security-and-compliance.md` §11 |

## 14. Open Decisions

- **Every numeric SLA target** (latency, uptime, RPO/RTO, throughput) named across this document as "not yet numerically defined" — the single largest, consolidated open item in the entire architecture documentation set, already independently flagged in `reliability-and-performance.md`, `infrastructure-and-release.md`, and `observability-and-operations.md`, and restated here as one master list rather than three scattered ones.
- **Accessibility standards** — genuinely absent from every prior document, not merely deferred; recommended as pre-launch product/design work, outside this architecture layer's own authority to define.
- **A formal legal/compliance review** — this document's compliance mapping (Section 3.17, Section 13) is architecture's own assessment and should not be treated as a substitute for one before production.
- **Test framework, coverage thresholds, and monitoring/logging tooling** — carried forward from `coding-standards.md` and `observability-and-operations.md`'s own Open Decisions, restated here for completeness rather than re-litigated.
- All module-ownership and business-decision Open Decisions carried through `module-catalog.md`, `capability-boundary-map.md`, `data-ownership-map.md`, and `service-decomposition.md` (Support's undeclared relationships, refund eligibility's exact decision owner, Auth's session-storage mechanism) remain open at the same status and are not restated in full here.
