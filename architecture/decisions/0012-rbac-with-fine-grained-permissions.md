# 0012. RBAC with fine-grained permissions

**Decision ID:** ADR-0012
**Status:** Accepted
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
The SRD defines four actor types (Customer, Picker, Rider, Admin/Ops) with materially different capabilities, and the Admin/Ops Dashboard (ADR-0008) serves Admin, Ops, and Support users whose permitted actions differ from one another within that single actor category. `00-Architecture-Principles.md` Principle 4.6 requires business rules — including authorization decisions — to live in one place, and Principle 4.10 requires consequential actions to be auditable.

## Problem
How does the backend decide what an authenticated caller is allowed to do, in a way that is centralized, auditable, and precise enough to distinguish an Ops user from a Support user without hardcoding per-actor logic throughout the codebase?

## Options considered
1. **Hardcoded per-actor-type checks scattered through service code** (`if user.type === 'admin'`). Fast to write once, but duplicates authorization logic across call sites and cannot express Admin/Ops/Support distinctions cleanly.
2. **Coarse role flags** — a single boolean such as `isAdmin`. Simple, but cannot express fine-grained differences (an Ops user managing dispatch versus a Support user viewing order history) without collapsing them into one over-privileged role.
3. **Role-Based Access Control with fine-grained, named permissions** grouped into roles, evaluated centrally by the backend's Identity & Access capability.

## Decision
Adopt RBAC: named, fine-grained permissions (for example, manage-catalog, view-financial-reports, override-order) grouped into roles — Customer, Picker, Rider, Admin, Ops, Support, and any future refinement — evaluated centrally by the Identity & Access capability for every authorization decision.

## Rationale
Options 1 and 2 both push authorization decisions outward into scattered call sites or force actor granularity coarser than the SRD's actual actor table supports, violating Principle 4.6's single-place-for-business-rules requirement. Fine-grained named permissions let a role's capabilities change through configuration rather than code, and every permission check becomes an explicit, centrally-evaluated decision that can be logged as an auditable event under Principle 4.10, rather than an opaque boolean check with no record of what was actually decided.

## Consequences
- New staff roles or permission refinements (for example, splitting Support into tiers) are configuration changes, not code changes across scattered call sites.
- Every client — especially the Admin/Ops Dashboard (ADR-0008) — can render its UI based on the same permission set the backend actually enforces, rather than a client-maintained approximation of it.
- Permission definitions must be maintained deliberately as the system grows, or they risk sprawling into an unmanageable list; this is an accepted ongoing maintenance cost.
- No client is ever trusted to self-report its own permissions; the backend re-checks authorization on every request regardless of what a client's UI already hid, consistent with Principle 4.6.

## Future reconsideration conditions
Evidenced need for attribute- or context-based rules RBAC cannot express cleanly — for example, a permission that depends on an order's value or a store's proximity to the acting user. Such a case would prompt layering attribute-based rules on top of the RBAC foundation, not replacing it.

## Related
- SRD section(s): Section 1.3 (Target Users/Actor table), Admin Dashboard operations-workflow scope
- Related DDD entity/data area(s): User/Account, Role, Permission, and audit-log entity areas
- Related architecture principle(s): 4.6, 4.10
- Related ADRs: 0008
