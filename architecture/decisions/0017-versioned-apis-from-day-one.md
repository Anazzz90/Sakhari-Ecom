# 0017. Versioned APIs from day one

**Decision ID:** ADR-0017
**Status:** Accepted
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
`00-Architecture-Principles.md` Principle 4.11 already states, at the principle level, that a breaking API change is introduced as a new version, never as a silent change to an existing one. Four independently-releasable clients (Customer Mobile and Worker App on app-store release cadences, Customer Web, Admin/Ops Dashboard) all depend on one shared backend API.

## Problem
Should API versioning be deferred until the first breaking change is actually needed, or established as a convention from the first endpoint, given that retrofitting versioning onto an already-live, unversioned API is itself a breaking change?

## Options considered
1. **No versioning until a breaking change is unavoidable**, introduced reactively at that point. Avoids upfront structure, but forces the very first breaking change to invent and retrofit a versioning scheme under time pressure while an already-shipped, unversioned client depends on the old behavior.
2. **Versioning only for the mobile-facing endpoints**, leaving web/admin endpoints unversioned. Reduces upfront work, but the Customer Web App and Admin Dashboard are equally independent release surfaces (ADR-0007, ADR-0008) with their own deployment cadence — this would just move the stranding risk onto whichever surface was left out.
3. **Every backend API endpoint carries an explicit version from the first release**, applied uniformly across all four client applications.

## Decision
Every API endpoint is versioned from the first release, applied consistently across all four client applications — not introduced reactively, and not scoped to only some clients.

## Rationale
Direct application of Principle 4.11. Option 1 was rejected because it recreates, at the worst possible moment, the exact client-stranding scenario the principle exists to prevent. Option 2 was rejected because the SRD's own application structure treats all four clients as independently-released surfaces (Principle 4.11's own justification); leaving any of them out of the versioning scheme just relocates the risk rather than removing it.

## Consequences
- A breaking change to any endpoint has a clean mechanism — a new version — from day one, with no special-case "first version bump" migration to design later under pressure.
- Every client's dependency on a specific API version is explicit and traceable.
- Carries the ordinary discipline cost of versioning on every endpoint even before it is strictly needed — a small amount of upfront structure, accepted deliberately.
- Any new endpoint follows the established versioning convention without exception, and a concrete deprecation policy for retiring old versions must be defined once a second version actually exists.

## Future reconsideration conditions
None anticipated for the decision to version from day one. The specific deprecation and support-window policy for retiring old versions is expected to be defined as a concrete policy once real version churn occurs, without reopening this ADR.

## Related
- SRD section(s): Section 2 (Application Structure — four independently-releasable clients)
- Related DDD entity/data area(s): n/a — an API contract convention, not a data-ownership decision
- Related architecture principle(s): 4.11
- Related ADRs: 0006, 0007, 0008
