# Coding Standards — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. New document — no prior skeleton existed for this scope. |
| **Stability** | Stable in principle (module boundaries, naming-matches-domain-language, tests-protect-business-rules — all direct extensions of the Constitution); evolving in specific tooling as the codebase matures. |
| **Authority** | Subordinate to `00-Architecture-Principles.md` (especially Section 8, Coding Philosophy, which this document expands into concrete practice), `02-Architecture-Decisions.md`, `03-decomposition/module-catalog.md` / `module-communication.md`, and `04-cross-cutting/data-architecture.md` / `security-and-compliance.md`. |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** Translates `00-Architecture-Principles.md` Section 8's Coding Philosophy — and every other document's architectural rules — into the concrete practices a change to this codebase is expected to follow: how the repository and modules are structured, how things are named, how logging/validation/error-handling are done consistently, what "well-tested" means here, what documentation a change owes, and what a reviewer (human or AI) actually checks before approving a change.

**Scope.** Repository and module structure, naming conventions, logging, validation, error handling, testing philosophy, documentation standards, and two checklists — one for ordinary code review, one for architecture-compliance review specifically. Consistent with this whole task's instruction, this document does **not** contain code, syntax examples, or a specific style guide's exact rules (semicolons, quote style, line length) — those are tooling-enforced (Section 5's own principle) and belong to a linter/formatter configuration, not architecture documentation. Where a specific tool or framework choice isn't already established elsewhere, this document says so explicitly (Section 12, Open Decisions) rather than inventing one.

**Intended audience.** Primarily an AI coding assistant generating or reviewing code, for whom this document is the concrete, checkable form of everything the rest of `/architecture` establishes more abstractly; also the project owner, reviewing a change or a PR against the checklists in Sections 10–11.

**Cross-references.** Directly expands `00-Architecture-Principles.md` Section 8 (Coding Philosophy) and Section 10 (Anti-Patterns). Grounded in ADR-0002/0009 (module boundaries), ADR-0005 (TypeScript-centered stack), ADR-0014/0015/0019 (UTC, money, ULIDs), `module-catalog.md` (module/interface structure), `module-communication.md` (dependency and communication rules), `data-architecture.md` (data conventions), `security-and-compliance.md` (validation/secrets), and `integration-and-messaging.md` (event naming). Does not repeat any of their content — this document is where they converge into what a single change must satisfy.

## 2. Repository Structure

Every module's internal folder structure mirrors its boundary exactly (ADR-0009): a module's directory contains its own service layer, its own data-access code, and its own public-interface definitions, and nothing outside that directory ever imports its internals — only its declared public interface (`module-communication.md` Section 5). This is the structural, on-disk expression of "no cross-module repository access," not merely a convention layered on top of it: if a module's data-access code is not importable from outside its own directory, the anti-pattern becomes structurally awkward to commit by accident, not just against the rules.

**Whether the Backend and the four client applications live in one repository (a monorepo) or five separate repositories (one per deployable, matching ADR-0017's independent-release-cadence reasoning) is not decided in any prior document.** This document does not assert one — see Open Decisions (Section 12). What it does establish, regardless of the answer: wherever the Backend's code lives, its internal structure follows the module boundaries in `module-catalog.md` exactly — fifteen module directories, one per module, named identically to the module names used throughout this documentation set (Auth, User, Store, Catalog, Search, Inventory, Cart, Order, Payment, Promotion, Delivery, Notification, Analytics, Audit, Settings).

## 3. Module Structure

Within each of the fifteen module directories, the same internal shape applies, consistent with Principle 4.6/4.7 and ADR-0009:

- **A public interface** — the operations listed in that module's `module-catalog.md` entry, and nothing else, importable from outside the module.
- **A service layer** — where business logic actually lives (Principle 4.6). This is the only layer permitted to make decisions; everything else in the module supports it.
- **A thin controller/entry-point layer** — translates an inbound REST request (`module-communication.md` Section 4) into a call on the service layer and the result back into a response. Contains no conditionals or business decisions (Principle 4.7).
- **Data-access code**, reachable only from within the module's own service layer — never imported or called from outside the module, and never bypassed by another module reaching the database directly (ADR-0009, this document's Section 2).
- **Event publishing/consuming code**, where applicable — publishers raise events about the module's own owned entities only (`integration-and-messaging.md` Section 2); consumers handle events per that document's idempotency requirement (Section 9).

**Why this shape is uniform across all fifteen modules, even though they differ enormously in complexity** (compare Settings to Order): a consistent internal shape is what lets a developer or an AI assistant orient inside any module — including one it has never worked in before, in a session with no memory of prior ones — using the same mental map every time, rather than re-learning module-specific conventions module by module.

## 4. Naming Conventions

- **Business/domain names in code match this documentation set's vocabulary exactly** (Constitution Section 8: "naming reflects the product's business language"). A module named Order in `module-catalog.md` is named Order in code — not "Orders," not "OrderService" as the module's own name, not a synonym. An entity named Inventory Reservation in the DDD is named consistently (not "Hold," not "Lock") wherever it appears in code, logs, or events.
- **Event names follow `integration-and-messaging.md` Section 3's convention exactly** (`<Entity><PastTenseVerb>`) — this document does not restate it, only confirms no module-specific variation is permitted.
- **Database naming follows the DDD's own conventions** (Section 2: snake_case tables/columns, `id` primary keys, `<entity>_id` foreign keys, `created_at`/`updated_at`/domain-specific timestamp columns) — this document does not repeat them, only confirms application code respects them at the data-access layer rather than introducing a second naming scheme that then needs translation.
- **No abbreviation that obscures a term already defined elsewhere in this documentation set.** If the DDD or `module-catalog.md` spells out "Inventory Reservation," code does not abbreviate it to "InvRes" — an abbreviation is a second vocabulary an AI assistant or a returning developer has to learn to map back to the documented one, which is exactly the kind of implicit rule Constitution Section 3 warns against.

## 5. Logging

Every module logs in the structured format `05-deployment/infrastructure-and-release.md` Section 7 establishes system-wide — this document adds only the module-level discipline: every log entry identifies the module and operation that produced it, so a log line is traceable back to a specific entry in `module-catalog.md` without guesswork. As that document already establishes, application logs are not the audit trail (`security-and-compliance.md` Section 9) — logging code and audit-recording code are never the same call, even when they're triggered by the same event.

## 6. Validation

Input is validated once, at the boundary — a controller validates that an inbound REST request is well-formed (Principle 4.7's own boundary between transport-shape validation and business-rule validation) before the service layer ever sees it; the service layer then applies business-rule validation (is this order allowed, is this stock available) as part of its own logic, not as a second pass over the same transport-shape concerns. Internal, module-to-module calls (`module-communication.md` Section 5) trust that the calling module already validated what it's passing, consistent with Constitution Section 8's "error handling exists at boundaries, not everywhere" — a module does not re-validate the shape of data it receives from another module's public interface on every call, only the business conditions specific to its own operation.

## 7. Error Handling

Two categories of error are handled differently, and code must make the distinction explicit rather than treating every failure identically:

- **Expected business-rule rejections** (insufficient stock, an ineligible promotion, a permission check that fails) are part of normal control flow — returned as a clear, specific outcome the caller can act on, never thrown as if they were unexpected system failures.
- **Unexpected system failures** (a database connection lost mid-transaction, an external integration timing out) are surfaced clearly and never silently swallowed (Constitution Section 10's "silent failure of consequential actions" anti-pattern) — a failed workflow moves to an explicit failure or pending state rather than being retried blindly or ignored (DDD Section 10.5, already established in `data-architecture.md` Section 13 and applied here to error-handling code specifically).
- **Every error a module raises is attributable to the module and operation that raised it** — consistent with Section 5's logging discipline, so an error is diagnosable from where it surfaced, not just that it occurred.

## 8. Testing Philosophy

Tests protect business rules, not implementation shape (Constitution Section 8's own principle, restated here as the operative testing standard): a test proves that an order cannot be placed against insufficient stock, not that a specific internal method was called in a specific order. This has concrete consequences for what "well-tested" means in this codebase:

- **Tests are written against a module's public interface** (Section 3), the same boundary everything else in this documentation set treats as the unit of correctness — not against internal implementation details that are free to change as long as the interface's behavior doesn't.
- **Correctness-critical paths get the most rigorous coverage**, directly proportional to Principle 4.1's ordering: checkout (order placement, inventory reservation, payment initiation — `module-communication.md` Section 10) and the named race conditions (`data-architecture.md` Section 13 — concurrent reservation attempts, duplicate payment callbacks, duplicate delivery assignment) require explicit tests proving the concurrency and idempotency guarantees actually hold, not just the happy path.
- **Idempotency is tested directly** for every workflow `data-architecture.md` Section 13 and `integration-and-messaging.md` Section 9 name as idempotency-required — a test that only exercises a workflow once has not verified the guarantee that actually matters for those workflows.
- **The consumer tier's graceful-degradation behavior is tested for the failure case, not only the success case** (`reliability-and-performance.md` Section 3) — a test verifying that a Notification failure does not block order placement is as important as a test verifying a notification is sent successfully.
- Tests are written in the same language as the code they test, consistent with the TypeScript-centered stack (ADR-0005) — this document does not name a specific test framework, since none is decided in any prior document (Section 12).

## 9. Documentation Standards

- **Documentation is part of "done"** (Constitution Section 11's own principle) — a module whose behavior changed without a corresponding update to its `module-catalog.md` entry (new public interface operation, new published/consumed event, changed dependency) is unfinished work, not a documentation backlog item.
- **A significant, hard-to-reverse decision gets an ADR** before or alongside the code that implements it (`decisions/README.md`'s own "what qualifies as ADR-worthy" test) — not retroactively, and not skipped because the change felt small at the time; Constitution Principle 4.12 treats an unrecorded shortcut as a violation regardless of size.
- **A module's public interface is documented at the same level of detail `module-catalog.md` already uses** — what each operation does, what it depends on, what it forbids — so a future SDD or an AI assistant reading the module's own code finds documentation consistent with, not divergent from, the architecture layer.

## 10. Code Review Checklist

A reviewer — human or AI — checks a change against all of the following before approving it:

- [ ] Does this change respect the module's ownership boundary — no direct access to another module's data, only its public interface (ADR-0009)?
- [ ] Is business logic in the service layer, not the controller (Principle 4.6/4.7)?
- [ ] Are money values integer halala, never floating point (ADR-0015)?
- [ ] Are timestamps stored and computed in UTC, converted to local time only at presentation (ADR-0014)?
- [ ] Are new entity identifiers ULIDs (ADR-0019)?
- [ ] Are transactions correctly scoped to a single module's data, with any cross-module consistency achieved through orchestration and compensation, not an implied cross-module transaction (`module-communication.md` Section 8)?
- [ ] Does a workflow that requires idempotency (Section 8 above) actually implement it, not just the happy path?
- [ ] Are secrets absent from the change entirely — no credential, key, or token committed, consistent with `security-and-compliance.md` Section 10?
- [ ] Does every consequential action this change introduces produce an audit trail where DDD Section 12.6 requires one?
- [ ] Is the change covered by tests proportional to its correctness criticality (Section 8)?
- [ ] Is any new or changed public interface, event, or dependency reflected in `module-catalog.md` (Section 9 above)?

## 11. Architecture Compliance Checklist

A separate, higher-level pass — for changes that might affect the architecture itself, not just a module's internals:

- [ ] Does this change introduce a new module, redraw an existing module's boundary, or reassign data ownership? If so, does it have a backing ADR, and is `module-catalog.md` updated?
- [ ] Does this change introduce a new technology, library category, or external dependency not already named in `04-Technology-Stack.md`? If so, does it have a backing ADR (per that document's own backfill-candidate pattern)?
- [ ] Does this change violate any module's Forbidden Dependencies list (`module-catalog.md` Section 4, per module)?
- [ ] Does this change bypass authentication or RBAC authorization for any request path (`security-and-compliance.md` Sections 4, 7)?
- [ ] Does this change duplicate business logic across two clients, or between a client and the Backend (Principle 4.6)?
- [ ] Does this change introduce a breaking API change without a new version (ADR-0017)?
- [ ] Does this change add a synchronous dependency on Notification, Analytics, Audit, or Search to a business operation's completion, outside the one named exception (`module-communication.md` Section 3)?
- [ ] Does this change contradict any `Accepted` ADR? If the contradiction is intentional, is a new, superseding ADR being proposed alongside it (`decisions/README.md`'s lifecycle rules)?

## 12. Trade-offs and Future Evolution

**Trade-off accepted:** this document deliberately does not mandate a specific test framework, linter, formatter, or repository topology — those are left as tooling choices to be made (and recorded, if significant enough to warrant an ADR) rather than asserted here without a documented basis, consistent with this whole task's instruction not to invent decisions. The cost is that this document is slightly less immediately actionable than a full style guide would be; the benefit is that it does not silently misrepresent an undecided question as settled.

**Future evolution:** as those tooling choices are made, this document's checklists (Sections 10–11) are the right place to add tool-specific items (a specific linter rule, a specific coverage threshold) — the checklists themselves are expected to grow; the principles behind them (Sections 2–9) are expected to hold.

## 13. Open Decisions

- **Monorepo vs. polyrepo** for the Backend and four client applications (Section 2) — not decided in any prior document.
- **Specific test framework, linter, and formatter** (Section 8, Section 12) — not decided; this document establishes what must be true of tests and code quality, not which tools enforce it.
- **Specific code-coverage thresholds** for correctness-critical paths (Section 8) — not decided; "proportional to correctness criticality" is the principle, not a percentage.
- **Whether module directories live in a shared "modules" root or are otherwise organized within the Backend repository** — a below-architecture-level detail, left to whichever SDD or initial project scaffold is authored first.
