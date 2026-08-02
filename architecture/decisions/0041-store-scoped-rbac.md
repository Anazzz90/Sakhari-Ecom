# 0041. Store-scoped RBAC (resource scope authorization)

**Decision ID:** ADR-0041
**Status:** Accepted
**Date:** 2026-07-30
**Deciders:** Principal Architect (solo-project decision authority), remediating ARR Critical Finding #7

## Context

ADR-0012 and `security-and-compliance.md` Section 7 establish fine-grained, named permissions (`order:manage`, `order:read_store`, etc.), but the ARR found this is fine-grained by *operation name* only — no document specifies how a permission check confirms a given Ops/Picker/Rider is authorized for *this specific store* versus any store. At "hundreds of stores" scale this is the multi-tenancy boundary: without an instance-scoping mechanism, an Ops user with `order:manage` could plausibly act on a store they are not assigned to.

## Problem

How is a permission check evaluated against *which store* the acting user is authorized for, not just *which operation* their role permits?

## Options considered

1. **Leave store scoping to each module's own business logic, ad hoc.** Rejected: this is exactly what produces the inconsistent-enforcement risk the ARR flagged — sixteen modules independently inventing (or forgetting to invent) the same check.
2. **Fold store scope into the permission name itself** (e.g., a permission per store, `order:manage:store_042`). Rejected: this produces an unbounded, per-store permission explosion at "hundreds of stores" scale, defeats the fixed, catalogued permission list ADR-0033 established, and conflates two orthogonal concerns (what operation, which resource instance) into one name.
3. **Authorization evaluates Permission AND Store Scope as two independent checks**, both required to pass, with store scope carried as its own claim alongside role/permission claims. Chosen.

## Decision

RBAC becomes resource-scoped. Authorization for any store-scoped operation requires **both**:

1. **Permission** — the actor's role grants the named permission for the operation (unchanged, ADR-0012/ADR-0033).
2. **Store Scope** — the target resource's store matches one of the actor's Assigned Stores.

`order:manage` for Store A does not authorize the same operation for Store B. Every operational user (Picker, Rider, Admin, Ops, Support — any actor other than Customer, whose operations are inherently self-scoped, e.g. `order:read_self`) has three attributes evaluated together: **Assigned Stores**, **Assigned Roles**, and **Assigned Permissions** (the permissions their roles grant, per the existing catalogue).

**Permission Evaluation Order:** a request is authorized only if, in order: (1) the caller is authenticated (Auth's existing `ValidateToken`), (2) the caller's role grants the named permission for the requested operation, and (3) where the operation targets a store-scoped resource, the resource's store is present in the caller's Assigned Stores. Failing any step denies the request; step 3 is skipped only for operations that are not store-scoped by nature (e.g., `settings:manage` against a global setting, `analytics:read` against a cross-store rollup an Admin role is entitled to).

**Resource Scope.** Store is the scoping resource at launch — the same store-isolation model already established (DDD Section 13). This decision generalizes the concept (naming it "Resource Scope," of which Store Scope is the current, only instance) specifically so a future scope dimension does not require re-deriving the evaluation model from nothing.

**Assigned Stores as an authorization primitive.** For workers (Picker, Rider), Assigned Stores is already effectively established as a single "home store" (`module-catalog.md` §4.2, DDD Section 5.6's "home store reference") — this decision does not change worker store assignment, only formalizes it as an instance of the same Store Scope mechanism Admin/Ops/Support staff now also carry, generalized to a set (Assigned Stores, plural) since an Ops or Support staff member may reasonably be assigned to more than one store, unlike a single-store Picker/Rider. Assigned Stores for Admin/Ops/Support staff is new authorization state, owned by **Auth**, following the same "Auth-internal, not separately named in the DDD" treatment already established for Session and Refresh Token (`module-catalog.md` §4.1's Ownership field) — because it is authorization-claim state, not business-profile data, it belongs alongside Auth's other session/claims state rather than becoming a new User-owned entity.

**Future regional scope extension.** As the platform expands beyond a single region (DDD Section 17.1, multi-country expansion), Resource Scope is expected to gain a second dimension (Region, above Store) — anticipated here explicitly so a future ADR extends this same evaluation model (Permission AND Store Scope AND Region Scope) rather than introducing a second, parallel authorization mechanism.

## Rationale

Splitting Permission and Store Scope into two independently-evaluated checks keeps the permission catalogue exactly as fixed and finite as ADR-0033 established (no per-store permission explosion) while still closing the ARR's exact concern: an operation's *legality* (permission) and its *applicability to this resource instance* (scope) are different questions, and conflating them either explodes the permission list or silently drops the scope check — this decision keeps both questions asked, every time, in a fixed order every module's controller applies identically (`security-and-compliance.md` Section 7's existing "every module's controller checks permissions before delegating" pattern, extended to include scope).

## Consequences

- `security-and-compliance.md` Section 7 gains a new subsection: Resource Scope, Store Scope, Permission Evaluation Order, and the future Region Scope note.
- `module-catalog.md`: Auth's Ownership field gains "Store Scope Assignment (Auth-internal, not separately named in the DDD, mirroring Session/Refresh Token)"; Auth's `ValidateToken` interface description is updated to note it returns Assigned Stores alongside role/permission claims.
- `data-ownership-map.md`: Auth's §12.1 Owned Tables gains `user_store_assignments`; Auth's Aggregate Roots note is updated to include this as Auth-internal state scoped to a User, exactly like Session/Refresh Token.
- Every module whose public interface accepts a store-scoped request (Order, Inventory, Delivery, Store, Payment where store-scoped, Catalog where store-specific) must apply the Store Scope check in addition to its existing permission check — an SDD-level, per-module obligation, not a new module dependency (the check is evaluated centrally by Auth's claims, exactly as permission checks already are, per `security-and-compliance.md` Section 2's centralization principle).
- `nfr-matrix.md` Section 3.7 (Security) gains store-scope enforcement as part of its acceptance criteria.
- Assigning or changing a user's Assigned Stores is itself a privileged, audited action (`security-and-compliance.md` Section 9) — consistent with permission/role changes already being within audit scope, and, per ADR-0040, itself a high-risk operation requiring step-up authentication ("permission changes").

## Future reconsideration conditions

Revisit with a superseding ADR when Region Scope is actually introduced (not speculatively), or if evidence shows a resource type other than Store needs independent scoping (e.g., a future marketplace seller-scope, DDD Section 17.3) — extending the same Permission-AND-Scope model, not replacing it.

## Related

- SRD section(s): FR-AUTH-004, Section 10.11 (Transition Requirements — "must validate ownership, store scope, role/capability")
- Related DDD entity/data area(s): 5.1 (User), 5.6 (Worker Profile — home store reference), Section 13 (Store Isolation)
- Related architecture principle(s): Principle 4.5, Principle 4.6
- Related ADRs: 0012, 0020, 0033, 0040
