# 0015. Integer halala money representation

**Decision ID:** ADR-0015
**Status:** Accepted
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
The Saudi Riyal (SAR) subdivides into 100 halala. The platform handles real payments across mada, card, BNPL, and cash-on-delivery (ADR-0010), and must never lose or misrepresent a fraction of a riyal across pricing, promotions, and payment capture.

## Problem
How should monetary amounts be represented and stored so that pricing, discounts, and payment capture are always exact, given that floating-point representations of decimal currency are a well-documented source of rounding error?

## Options considered
1. **Floating-point decimal values** (a `float`/`double` SAR amount). Familiar, but floating-point binary representation cannot exactly represent most decimal fractions, producing rounding drift over repeated calculation.
2. **A fixed-point decimal database type**, storing SAR directly with two decimal places. Workable at the database layer, but still invites accidental floating-point contamination the moment any client or reporting layer casts the value loosely.
3. **Integer halala** — every monetary amount stored and computed as a whole-number count of halala (1 SAR = 100 halala), converted to a SAR display value only at presentation.

## Decision
All monetary amounts are stored and computed internally as integers denominated in halala. Conversion to a SAR-denominated decimal display value happens only at the presentation layer.

## Rationale
Floating-point representation is rejected outright: rounding drift in monetary state is exactly the kind of failure Principle 4.1 exists to prevent, and any such drift would leave Principle 4.10's audit trail unable to give a clean answer for a discrepancy. A fixed-point decimal type is workable but does not remove the risk of accidental floating-point contamination at the application layer. Integer halala removes the ambiguity at the type level: an amount is either a whole number of halala or it is a bug, not a rounding question — the strongest guarantee available for the correctness-first standard Principle 4.1 sets.

## Consequences
- Every monetary calculation — pricing, discounts, payment capture, refunds — is exact integer arithmetic with no rounding ambiguity.
- Reconciliation against payment gateway records (mada, card, BNPL) is a direct integer comparison rather than a decimal-tolerance comparison.
- Every client and reporting surface must consistently convert halala to a SAR display value, and that conversion must be tested explicitly.
- No service, client, or report performs monetary arithmetic in a floating-point or SAR-decimal representation; conversion to decimal SAR happens once, at the final display step, never mid-calculation.

## Future reconsideration conditions
None anticipated for SAR. If the platform ever supports an additional currency with a different minor-unit structure, that would be handled as a new, explicitly-scoped currency-handling ADR rather than a change to this one.

## Related
- SRD section(s): Section 1.2 (SAR currency, mada payment requirement), pricing/promotions data requirements
- Related DDD entity/data area(s): Order, OrderItem, Payment/Transaction, Promotion/Pricing entity areas
- Related architecture principle(s): 4.1, 4.9, 4.10
- Related ADRs: 0010
