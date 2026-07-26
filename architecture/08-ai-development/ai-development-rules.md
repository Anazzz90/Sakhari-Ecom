# AI Development Rules — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. New document — no prior skeleton existed for this scope. |
| **Stability** | Stable — this document is a consolidation and operational restatement of rules already established elsewhere; it should change only when one of those underlying documents changes. |
| **Authority** | Subordinate to every other document in `/architecture` — this document invents no new rule; it exists to make rules already established elsewhere impossible to miss or misapply when the one generating code is an AI assistant with no memory of prior sessions. Where anything here appears to conflict with another document, that other document wins, and this document is wrong until corrected. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** Every other document in `/architecture` was written assuming its reader would eventually connect it to the others. This document is that connection, made explicit and operational, specifically for an AI coding assistant that starts every session cold. It exists because Constitution Section 9 (AI Development Philosophy) states the *principle* — "documentation is the substitute for memory" — and this document is the *procedure* that principle demands: what to read, what never to do, and how to know a change is actually finished.

**Scope.** Required context before generating code, architecture constraints, forbidden patterns, prompting guidelines, module ownership rules, transaction rules, security rules, testing expectations, a review checklist, a definition of done, and two catalogs of known failure modes — common AI hallucinations and common architectural violations — specific to this codebase. This document contains no new architectural content; every rule here is a pointer to, or restatement of, a rule established in full elsewhere.

**Intended audience.** An AI coding assistant, exclusively. Where this document says "you," it means whichever assistant is reading it in order to generate, review, or reason about code for this project. A human reviewer may still find it useful as a checklist, but it is not written for a human learning the architecture for the first time — `01-Architecture-Design-Specification.md` is that document.

**Cross-references.** This document synthesizes, without repeating: `00-Architecture-Principles.md` (all twelve Core Principles, Section 9's AI Development Philosophy, Section 10's Anti-Patterns, Section 12's Decision-Making Guidelines); the full ADR set (`decisions/`); `03-decomposition/module-catalog.md` and `module-communication.md`; `04-cross-cutting/data-architecture.md`, `security-and-compliance.md`, and `integration-and-messaging.md`; and `07-coding-standards/coding-standards.md`.

## 2. Required Context Before Generating Code

Before writing or modifying a single line of code for this project, establish which of the following apply to the task, and read the corresponding document — not a summary of it, not a memory of it from an earlier session:

| If the task touches... | Read first |
|---|---|
| Any module at all | That module's full entry in `module-catalog.md` (Section 4) — Responsibilities, Boundaries, Ownership, Public Interfaces, Dependencies, Forbidden Dependencies, Published/Consumed Events |
| More than one module | `module-communication.md` in full — the dependency graph (Section 7), transaction boundaries (Section 8), and circular-dependency rules (Section 9) |
| Persistence, schema, or a query | `data-architecture.md` in full |
| Authentication, authorization, tokens, OTP, or secrets | `security-and-compliance.md` in full |
| A domain event (publishing or consuming) | `integration-and-messaging.md` in full |
| Any structural or naming choice | `coding-standards.md` in full |
| A choice that isn't clearly settled by any of the above | `decisions/README.md`'s index, checked for an `Accepted` ADR before assuming an answer |

**If, after checking, the task still touches a genuinely open question** (see each document's own Open Decisions section, and Section 11 below), the correct action is to say so and propose an approach — never to silently pick one and proceed as if it were settled (Constitution Section 9).

## 3. Architecture Constraints

The following are load-bearing and apply to every task without exception, restated here only as a checklist — each is fully explained in the document cited:

- One deployable Backend, sixteen fixed modules, no new module without an ADR (ADR-0002, ADR-0009, ADR-0020; `module-catalog.md`).
- PostgreSQL is the only source of truth; Redis is cache/queue/lock/rate-limit only, never authoritative (ADR-0003, ADR-0004; `data-architecture.md` Sections 2–3).
- No cross-module repository access, ever — a module reaches another module only through its public interface (ADR-0009; `module-communication.md` Section 2).
- REST is the client-facing protocol only; modules never call each other over REST — in-process interface calls and domain events are the only internal communication modes (`module-communication.md` Sections 3–5).
- Business logic lives only in a module's service layer; controllers are thin translators (Principle 4.6/4.7; `coding-standards.md` Section 3).
- Checkout belongs to the Order module, which orchestrates Cart, Inventory, Payment, and Promotion via synchronous interface calls, each within its own module-local transaction — never one transaction spanning modules (`module-communication.md` Sections 7–8).
- Inventory reservation happens after order-record creation, as its own transactional step (`data-architecture.md` Section 7; `module-communication.md` Section 8).
- The inventory ledger is append-only; nothing is ever hard-deleted from it (`data-architecture.md` Section 8).
- Money is integer halala; timestamps are UTC; identifiers are ULIDs — no exceptions, anywhere (ADR-0015, ADR-0014, ADR-0019).
- Master/profile data is soft-deleted; state-driven entities use lifecycle state, never deletion, to represent status (`data-architecture.md` Section 9).
- Authentication is OTP-based with no passwords; sessions use JWT access tokens plus rotating refresh tokens; authorization is RBAC with fine-grained permissions, evaluated centrally (ADR-0012; `security-and-compliance.md`).
- Every consequential action is auditable (Principle 4.10; `security-and-compliance.md` Section 9).
- APIs are versioned from day one; a breaking change is a new version, never a silent change (ADR-0017).
- A future service extraction must be possible without changing a module's business logic — which is exactly why every rule above exists (ADR-0002/0009; `01-Architecture-Design-Specification.md` Section 15).

## 4. Forbidden Patterns

Restating Constitution Section 10's anti-patterns as direct instructions, plus AI-specific additions:

- **Never** query, join against, or write to a table owned by a module other than the one you're working in — through any mechanism, including a "just this once" raw query.
- **Never** put a business decision (a validation, a calculation, an eligibility check) in a controller. If you find yourself writing `if` logic in a controller beyond request-shape checking, it belongs in the service layer instead.
- **Never** introduce shared mutable state between modules, or between requests within a module, outside of a transaction or an explicit, owned data structure.
- **Never** add an import or a call that would create a cycle in `module-communication.md` Section 7's dependency graph — check the table before adding a new inter-module call, not after.
- **Never** let a consequential action fail silently — an error is surfaced, logged, and where required, audited; it is never swallowed.
- **Never** invent a new module, a new external dependency, or a new technology choice without first checking whether an ADR already covers it, and flagging (not silently deciding) if one doesn't exist yet.
- **Never** answer an Open Decision from another document by picking an option and writing code as if it had been settled. Flag it, propose an approach, and let the project owner decide — or, if instructed to proceed, state the assumption explicitly in the change itself.
- **Never** optimize a correctness-critical path before its correctness is established and tested (Principle 4.1's ordering, Constitution Section 10).

## 5. Prompting Guidelines

Guidance for how this assistant should engage with tasks on this codebase, not for how a human should phrase requests:

- **State the plan before writing significant code**, referencing the specific documents and rules it's built on — this is what lets a human reviewer (who also doesn't remember everything in `/architecture`) verify the plan against the architecture before code exists, when correcting course is cheapest.
- **Cite the source of a non-obvious choice.** If a change relies on a specific ADR, a specific module's Forbidden Dependencies, or a specific convention, say which one — this is what makes the change auditable later, by a human or a future AI session, without re-deriving the reasoning.
- **Ask when genuinely blocked, proceed when not.** A gap in documented architecture (Section 11) warrants a question or a clearly-flagged assumption; a question already answered somewhere in `/architecture` does not warrant asking the project owner again — check first (Section 2).
- **Never fabricate a citation.** If unsure which document supports a claim, say so, or state the claim as a reasonable inference rather than attributing it to a document that doesn't actually say it — Section 10 names this failure mode directly.

## 6. Module Ownership Rules

- Every entity has exactly one owning module (`module-catalog.md` Section 4, Data Ownership field per module) — before writing a query or a migration, confirm the table belongs to the module you're working in.
- A module's Forbidden Dependencies list (`module-catalog.md`, per module) is not a suggestion — it is checked the same way a compiler error would be, before the code is considered correct.
- The five-layer dependency structure (`module-communication.md` Section 7 — Foundation, Reference, Commerce, Transaction Orchestration, Fulfillment, Consumer tier) determines direction: a lower layer never calls upward.
- Where a relationship seems to require an upward dependency (a Reference-tier module needing Commerce-tier data, for example), the answer is almost always a domain event, not an exception to the rule (`module-communication.md` Section 9's "escape valve").

## 7. Transaction Rules

- A transaction is scoped to one module's own tables — never write code that opens a single transaction spanning two modules' data (`module-communication.md` Section 8).
- A multi-module business operation (checkout being the canonical case) is orchestrated: each step is its own module-local transaction, called synchronously by the orchestrator, with an explicit compensating action if a later step fails (`module-communication.md` Section 8, worked example in Section 10).
- Every workflow DDD Section 10.4 and `data-architecture.md` Section 13 name as idempotency-required (order creation, payment confirmation, inventory reservation/release, notification sends, refund processing, rider assignment/reassignment) must be safe to retry without duplicating its effect — this is a requirement to implement, not an aspiration to note.
- Optimistic revalidation (`data-architecture.md` Section 13) — re-checking current state immediately before committing — is required wherever a workflow's correctness depends on state that could have changed since it was last read.

## 8. Security Rules

- Never trust a client-supplied value for identity, role, permission, or price — every request is authenticated and authorized fresh by the Backend, regardless of what the client claims (`security-and-compliance.md` Section 2, `02-context/system-context.md` Section 5's Trust Boundaries).
- Never hardcode, log, or commit a secret, credential, or signing key — secrets are always externalized (`security-and-compliance.md` Section 10).
- Never bypass RBAC's central evaluation with a module-local permission check — authorization is Auth's job, checked before a controller ever delegates to a service layer (`security-and-compliance.md` Section 7).
- Never call an external system (Payment Gateway, SMS/OTP Provider, Push Notification Service, Geocoding Provider) from any module other than the one `02-context/system-context.md` names as its owner.
- Never treat an external system's response as fact without validating it first — signature/webhook verification, expected-shape checks — the same discipline applied symmetrically to inbound client requests (`security-and-compliance.md` Section 8).
- Never implement password-based authentication anywhere in this system — OTP is the only credential mechanism, by design (`security-and-compliance.md` Section 4).

## 9. Testing Expectations

- Every change includes tests proportional to what it touches — a correctness-critical path (checkout, payment, inventory reservation) gets more rigorous coverage than a Settings CRUD operation, per `coding-standards.md` Section 8's ordering, not a uniform minimum applied everywhere equally.
- Any workflow this document's Section 7 names as idempotency-required gets an explicit test proving duplicate delivery/retry doesn't duplicate its effect — a test suite that only exercises the happy path once has not verified the guarantee that matters most for these workflows.
- A named race condition (`data-architecture.md` Section 13 — concurrent reservation of the same stock, duplicate payment callbacks, duplicate delivery assignment) gets a test that actually exercises the race, not just the sequential case.
- A change to the consumer tier (Notification, Analytics, Audit, Search) includes a test for its failure case specifically — proving the rest of the system tolerates it failing, not only that it succeeds when everything is healthy.

## 10. Review Checklist

Before presenting any change as complete, work through `coding-standards.md` Sections 10 and 11 (the Code Review Checklist and the Architecture Compliance Checklist) explicitly — every item, not a sample of them. This document does not duplicate those checklists; it requires running them.

## 11. Definition of Done

A change is not done until all of the following are true, not merely attempted:

- [ ] It satisfies every applicable item in `coding-standards.md`'s two checklists (Section 10 above).
- [ ] No Forbidden Pattern (Section 4) is present anywhere in the change.
- [ ] Every module boundary crossed by the change goes through a public interface, an event, or a documented external integration — never a direct data access.
- [ ] Tests exist and are proportional to the change's correctness criticality (Section 9).
- [ ] `module-catalog.md` (or another relevant document) is updated if the change adds, removes, or alters a public interface, a dependency, or a published/consumed event.
- [ ] No Open Decision (any document's own Open Decisions section) was silently resolved by the change — either it was left open with an explicit note, or a new ADR was proposed to resolve it.
- [ ] A significant, hard-to-reverse choice made along the way has a corresponding ADR, drafted or explicitly flagged as needed.
- [ ] The change was checked against Section 6 (Common AI Hallucinations) and Section 7 (Common Architectural Violations) below, specifically — not just against general good judgment.

## 12. Common AI Hallucinations

Specific, concrete failure modes an AI assistant is prone to on this project, named so they can be checked for directly rather than trusted to general carefulness:

- **Assuming UUIDs instead of ULIDs for identifiers.** This is not a hypothetical risk — it happened in this project's own history (`decisions/0013`, `0018`, `0019`): an earlier pass adopted UUIDs based on a draft error in the DDD, before the DDD was corrected and ULIDs restored. Always check `decisions/0019` (the current, effective decision) rather than a general assumption about what's "standard."
- **Assuming a microservices architecture** because the system is described in terms of "modules," "boundaries," and "services" throughout this documentation set. It is a single-deployable modular monolith (ADR-0002) with in-process communication (`module-communication.md` Section 4) — module-to-module calls are never network calls, and events are dispatched in-process, not via a message broker, today (`integration-and-messaging.md` Section 11).
- **Assuming Redis holds business data** because it's described as part of the "data architecture." It never does (Principle 4.3, ADR-0004) — anything Redis holds is disposable by design.
- **Ignoring the accepted provider decisions.** Moyasar is the accepted online payment gateway (ADR-0022), Unifonic is the accepted SMS/OTP provider (ADR-0023), and AWS `me-central-2` is the accepted production region (ADR-0024). Do not replace them with Stripe, Twilio, Firebase Auth, a generic cloud region, or another plausible default unless the user explicitly asks to revisit the ADR.
- **Inventing specific SLA numbers, backup retention windows, or scaling thresholds.** `reliability-and-performance.md` and `infrastructure-and-release.md` both explicitly leave these as Open Decisions. A plausible-sounding number ("99.9% uptime," "30-day retention") is still an invention if it doesn't trace to an approved document.
- **Assuming a specific test framework, linter, or monorepo/polyrepo structure** — `coding-standards.md` Section 12 explicitly leaves all of these open. Do not generate configuration or code that presumes one without flagging the assumption.
- **Re-opening ADR-0020's module ownership decisions.** Support is a module, Refund belongs to Payment, Picking belongs to Delivery, Workforce belongs to User, Price Snapshot belongs to Order, Auth/User are split, and Search is an accepted derived-read module. Do not treat the pre-ADR open questions as still unresolved.
- **Assuming Payment initiation is event-triggered.** It is a direct synchronous call from Order (`module-communication.md` Section 7) — an earlier version of `module-catalog.md` left this ambiguous before it was resolved; the resolved version is authoritative.

## 13. Common Architectural Violations

Distinct from Section 12: these are violations that can occur even when every fact used is correct — the code itself breaks a rule.

- Writing a query or an ORM relationship in one module's code that reaches into another module's table directly, even "just for a read" — the single most common way ADR-0009 gets violated in practice.
- Placing a validation or eligibility check (should this promotion apply, is this order cancellable) inside a controller instead of the owning module's service layer.
- Adding a synchronous call from Order, Cart, Inventory, Payment, Promotion, or Delivery into Notification, Analytics, Audit, or Search, outside the one named exception (Auth calling Notification for OTP dispatch) — these four exist specifically so nothing has to wait on them.
- Wrapping a checkout flow's multiple module calls in one shared database transaction instead of module-local transactions with orchestration and compensation (`module-communication.md` Section 8) — a natural-feeling shortcut that directly contradicts how this system's transaction boundaries are designed.
- Using a floating-point or decimal type for a monetary value anywhere, even temporarily "for a calculation" — integer halala, always (ADR-0015).
- Storing or comparing a timestamp in local time instead of UTC, even for a value that's "only used for display" — conversion happens at presentation only (ADR-0014).
- Adding a new REST endpoint without a version, or changing an existing endpoint's contract in a way that breaks an existing client, without introducing a new version (ADR-0017).
- Implementing a new module's internal structure without the public-interface/service-layer/controller separation `coding-standards.md` Section 3 requires — even for a module that feels "too simple to need it" (Settings, for example).
- Skipping RBAC evaluation because "the client's UI already hides this" — the Backend re-checks regardless of what a client chose not to show (`security-and-compliance.md` Section 7).

## 14. Open Decisions

This document introduces no Open Decisions of its own — it inherits and points to every other document's. Before treating any of the following as settled, check the cited document directly: exact token lifetimes and the full permission list (`security-and-compliance.md` Section 12); deadlock-avoidance entity-update ordering and archival timing (`data-architecture.md` Section 17); synchronous API conventions and external-integration retry patterns (`integration-and-messaging.md` Section 12); specific compute service, network topology, backup/RPO/RTO targets, and CI/CD tooling (`infrastructure-and-release.md` Section 14); exact SLA numbers and scaling thresholds (`reliability-and-performance.md` Section 13); and test framework, linter, and repository topology (`coding-standards.md` Section 13).

