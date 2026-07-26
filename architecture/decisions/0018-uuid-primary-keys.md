# 0018. UUID primary keys (supersedes ULIDs)

**Decision ID:** ADR-0018
**Status:** Superseded by [0019](0019-restore-ulid-primary-keys.md)
**Date:** 2026-07-26
**Deciders:** Principal Architect (solo-project decision authority)

## Context
ADR-0013 adopted ULIDs as the primary identifier scheme for business entities, reasoned from first principles in the absence of a Database Design Document. `DDD_Sakhari_Ecom_v1.0.md` has since been authored and is now the project's authoritative database design reference. Its Section 1.2 ("UUID Primary Keys"), Section 2.3 ("Primary Keys"), and Section 3.1 ("UUID vs Sequential IDs") state a firm, already-decided convention: "every primary business entity should use UUID identifiers unless a later implementation document justifies a narrower local identifier," explicitly to "avoid coupling external references to insertion order." This is now in direct conflict with ADR-0013's choice of ULIDs, whose defining property is a timestamp-prefixed, sortable identifier — the DDD's own stated rationale for UUIDs reads as a deliberate rejection of exactly that property, not an oversight.

## Problem
Given that the DDD — now the authoritative source for database design — has already specified UUID primary keys with an explicit rationale that argues against sortable identifiers, does ADR-0013's ULID decision stand, or must it be superseded to keep the architecture and the database design consistent?

## Options considered
1. **Keep ADR-0013 (ULIDs) and treat the DDD as needing correction.** Rejected as backwards: the DDD is the authoritative database design document and was authored with its own explicit rationale for this choice; an architecture-layer ADR does not get to overrule the database design layer's considered decision on a matter squarely inside that layer's ownership.
2. **Let both stand, unreconciled, and leave it to implementation to pick one.** Rejected outright — this is precisely the "documents silently disagree" failure mode the Constitution's decision-making guidelines and the ADR process exist to prevent (Constitution Section 12; `decisions/README.md`).
3. **Supersede ADR-0013 with a new ADR adopting UUIDs**, aligning the architecture layer with the now-authoritative DDD, and mark ADR-0013 `Superseded by 0018` per the ADR lifecycle rules.

## Decision
Adopt UUIDs (not ULIDs) as the primary identifier for business entities across the system, per the DDD's Section 1.2/2.3/3.1. ADR-0013 is superseded by this ADR.

## Rationale
The DDD is the authoritative source for database-level identifier design; ADR-0013 was a reasonable architecture-layer judgment call made in the absence of that document, not a decision that should now override it. The DDD's own stated reason for choosing UUIDs over a sortable scheme — avoiding any coupling between an identifier and insertion order, even the partial coupling a ULID's timestamp prefix creates — is a legitimate, deliberate tradeoff distinct from (and taking priority over) ADR-0013's emphasis on log/audit-trail sortability. The DDD also already provides the answer to ADR-0013's sortability concern directly: human-friendly, separately-generated order numbers for support and customer communication, with UUIDs remaining the internal identity (DDD Section 3.1) — meaning the audit-trail-readability rationale in ADR-0013 is addressed by a different, DDD-specified mechanism rather than by the primary key itself.

## Consequences
- All new entity tables use a UUID primary key convention, per the DDD, rather than ULIDs.
- The non-sequential, non-guessable property ADR-0013 also wanted (Principle 4.1, avoiding business-volume leakage) is preserved — UUIDs satisfy it as well as ULIDs did — so nothing about ADR-0013's original *problem* goes unsolved, only the *specific scheme* changes.
- Debugging and audit-trail readability lose ULID's natural creation-time sort order at the identifier level; this is accepted per the DDD's rationale, with `created_at` columns (DDD Section 2.8) and append-only history/ledger tables (DDD Sections 1.7, 1.8) serving as the ordering mechanism instead of the primary key itself.
- Any module can still generate its own identifiers without a central coordinating sequence (DDD Section 3.1's "support distributed generation if needed later"), so ADR-0002/0009's future-extractability rationale is preserved.
- This is the first ADR to supersede a prior one; `decisions/0013` is updated to `Superseded by 0018` per the ADR lifecycle (its Decision/Rationale sections are left untouched, only its Status line changes, consistent with `decisions/README.md`'s stated lifecycle rules).

## Future reconsideration conditions
None anticipated. Should the DDD itself be revised to reconsider this convention, that revision — not a future architecture-layer ADR reasoning independently about identifiers again — is what should trigger a further ADR here, since the database design layer owns this decision going forward.

## Related
- SRD section(s): n/a directly — an implementation-adjacent data convention, not a business requirement
- Related DDD entity/data area(s): DDD Section 1.2 (UUID Primary Keys), Section 2.3 (Primary Keys), Section 3.1 (UUID vs Sequential IDs) — all entity areas
- Related architecture principle(s): 4.1, 4.10
- Related ADRs: 0013 (superseded by this ADR), 0003, 0009
