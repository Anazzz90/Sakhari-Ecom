# 0019. Restore ULID primary keys (supersedes ADR-0018; corrects DDD)

**Decision ID:** ADR-0019
**Status:** Accepted
**Date:** 2026-07-26
**Deciders:** Principal Architect (solo-project decision authority), confirmed by project owner

## Context
ADR-0013 originally adopted ULIDs. ADR-0018 superseded it with UUIDs, reasoned entirely from `DDD_Sakhari_Ecom_v1.0.md` Sections 1.2, 2.3, and 3.1, which at the time specified UUID primary keys with a rationale explicitly against sortable identifiers. The project owner has since confirmed that ULID was always the intended decision, and that the DDD's UUID text was itself the error, not ADR-0013's original reasoning. The DDD has been corrected accordingly (Sections 1.2, 2.3, 3.1 now specify ULID, with rationale text updated to treat lexicographic sortability as a benefit rather than a leak).

## Problem
With the DDD corrected back to ULIDs, does ADR-0018 (UUIDs) still stand, or must it be superseded to restore consistency between the architecture layer and the now-corrected, authoritative database design?

## Options considered
1. **Leave ADR-0018 in effect and treat the DDD correction as a documentation-only fix with no architectural consequence.** Rejected — ADR-0018's entire Decision and Rationale were derived from the UUID text that is no longer accurate; leaving it "Accepted" would mean an ADR whose stated reasoning no longer matches the document it was reasoning from.
2. **Silently edit ADR-0018 to reflect ULIDs.** Rejected outright — this is exactly what ADR immutability (`decisions/README.md`) exists to prevent. An ADR's Decision and Rationale are a historical record of what was decided and why, at the time; editing them after the fact is how documentation stops being trustworthy.
3. **Author a new ADR superseding ADR-0018**, restoring ULIDs, and explaining why — preserving the full, honest sequence (0013 ULID → 0018 UUID, based on a draft DDD error → 0019 ULID, based on the corrected DDD) rather than erasing any part of it.

## Decision
Restore ULIDs as the primary identifier for business entities across the system, per the corrected DDD. ADR-0018 is superseded by this ADR. ADR-0013's original reasoning is affirmed as correct in substance; it remains formally `Superseded by 0018` per ADR immutability, with this ADR (0019) recorded as the current, effective decision.

## Rationale
ADR-0013's original analysis was sound: ULIDs give non-sequential, non-guessable identifiers (avoiding the business-volume leakage risk of auto-incrementing keys) while remaining lexicographically sortable by creation time, which aids debugging, log correlation, and audit-trail readability (Principle 4.10) — a genuine benefit ADR-0018 traded away for a DDD rationale that has since been identified as incorrect and corrected. With the DDD now aligned to ULIDs, there is no remaining conflict between the architecture layer and the database design layer to reconcile in UUID's favor; restoring ULIDs removes the one artificial tradeoff ADR-0018 introduced (loss of sort order) without reintroducing any of the risks ADR-0013 and ADR-0018 both agreed to avoid (sequential leakage, central coordination dependency).

## Consequences
- All new entity tables use a ULID primary key convention, per both the corrected DDD and this ADR.
- Debugging, log correlation, and audit-trail readability regain ULID's natural creation-time sort order at the identifier level, which ADR-0018 had given up.
- Any module can still generate its own identifiers without a central coordinating sequence, preserved across all three ADRs in this chain (0013, 0018, 0019) and consistent with ADR-0002/0009's future-extractability goal.
- The decision chain (0013 → 0018 → 0019) is left fully visible in the index rather than collapsed — a reader tracing "why ULID" sees the DDD-error detour and its correction, not just the current answer, which is itself a demonstration of the ADR process working as designed under `decisions/README.md`.

## Future reconsideration conditions
None anticipated. Should either the architecture layer or the DDD want to reconsider identifier strategy again, that reconsideration should update both documents together in the same change, specifically to prevent a repeat of the divergence this ADR and ADR-0018 exist to correct.

## Related
- SRD section(s): n/a directly — an implementation-adjacent data convention, not a business requirement
- Related DDD entity/data area(s): DDD Section 1.2 (ULID Primary Keys), Section 2.3 (Primary Keys), Section 3.1 (ULID vs Sequential IDs) — all entity areas
- Related architecture principle(s): 4.1, 4.10
- Related ADRs: 0013 (original decision, formally superseded by 0018 but affirmed correct in substance by this ADR), 0018 (superseded by this ADR), 0003, 0009
