# 0038. Line-item aware partial refunds

**Decision ID:** ADR-0038
**Status:** Accepted
**Date:** 2026-07-30
**Deciders:** Principal Architect (solo-project decision authority), remediating ARR Critical Finding #4

## Context

The ARR found that partial refunds — routine in grocery/quick-commerce for substitutions, out-of-stock lines, and damaged items (SRD FR-PRICE-011, BR-029 already anticipate substitution-driven partial refunds) — had no defined granularity anywhere. DDD Section 5.28 (Refund) names "amount" and "order items" as a loose relationship without specifying line-level linkage, quantity, or how a refund interacts with a promotion already applied to the order.

## Problem

At what granularity does a Refund reference an order, and how is a partial-refund amount calculated so it stays consistent with the order's original, promotion-adjusted price?

## Options considered

1. **Refund at the order level only** (one amount, no line-item linkage). Rejected: this cannot represent "refund this one substituted item" without an out-of-band note explaining what the amount corresponds to — exactly the "visible but not explainable" gap ADR-0037 closes for the ledger, recreated at the refund-reason level instead.
2. **Recalculate a refund amount from current catalog price at refund time.** Rejected outright by an existing constraint DDD Section 5.28 already states and this decision does not change: "Refund amount uses order price snapshots, not current catalog price." Recomputing from current price would let a later price change silently alter a refund's fairness.
3. **Refund references one or more specific Order Items, at a quantity and using the order's own frozen Price Snapshot, with promotion adjustments already baked into that snapshot.** Chosen.

## Decision

A Refund becomes line-item aware. A Refund references: the Order, one or more Order Items (with quantity per item), the Original Unit Price (read from the order's Price Snapshot, per ADR-0020's Order-owned Price Snapshot — never recomputed from current catalog price), the Refunded Amount, and a Refund Reason.

Promotional discounts remain attached to the original order — a Refund never re-runs promotion evaluation. Refund calculations follow documented business rules (proportional reduction of a line's promotion-adjusted price where a promotion applied across multiple items, full line reversal where it applied to a single item) rather than recalculating a promotion's applicability at refund time.

Multiple partial refunds are supported per order — a sequence of Refund records, each independently line-item-scoped, whose cumulative Refunded Amount across all of an order's non-rejected, non-cancelled Refund records must never exceed the order's paid total. Refund history remains immutable (DDD Section 5.28's existing constraint: "Completed refunds are immutable except corrective refund records") — this decision does not change that; it only adds the line-item structure a refund amount is computed and recorded against.

Ownership remains with Payment (per ADR-0020, Refund is folded into Payment) — this decision changes Refund's internal structure, not its owning module.

## Rationale

Line-item linkage is what makes a partial refund *explainable* rather than merely a number with a free-text reason attached — the same principle ADR-0037 applies to the ledger, applied here to the refund's own internal shape. Grounding the amount in the order's frozen Price Snapshot (rather than current catalog price) preserves the constraint DDD Section 5.28 already established; this decision only makes that constraint enforceable at line granularity instead of only at the order-total level. Keeping promotion re-evaluation out of refund calculation avoids a second, parallel pricing engine inside Payment — Promotion's rules were already applied once, at order time, and the frozen snapshot is what a refund reads from, consistent with Promotion's own Forbidden Responsibility of never being called outside checkout (`capability-boundary-map.md` §4.10).

## Consequences

- `DDD_Sakhari_Ecom_v1.0.md`: Section 5.28 (Refund)'s Core Fields and Relationships are extended to name explicit Order Item references, quantity, and Original Unit Price (sourced from Price Snapshot); Constraints gains the cumulative-refund-never-exceeds-paid-total rule and the promotion-remains-attached-to-original-order rule.
- `data-ownership-map.md`: Payment's §12.9 Refund entry gains an explicit note that a Refund may reference multiple Order Items at line granularity; no ownership or aggregate-root change (Refund remains a dependent child of Payment, per Section 9.5/12.9).
- `module-catalog.md`, `capability-boundary-map.md`: Payment's Refund-related Ownership and Business Capability descriptions gain one clause noting line-item, multi-partial-refund support; `IssueRefund`'s interface now accepts a per-line breakdown, an internal contract detail, not a new interface name.
- Payment's SDD (when written) must implement the cumulative-refund-never-exceeds-paid-total check as a transactional validation at refund-creation time (Section 13's optimistic-validation discipline, `data-architecture.md`), and must derive Refund's Original Unit Price from Order's Price Snapshot via Payment's existing read of Order context (`module-catalog.md` §4.9's Dependencies), never by calling Promotion or Catalog directly.
- Every Refund entry also produces a Payment Ledger entry (ADR-0037) — Refund Issued or Refund Reversed — keeping the ledger's movement history and the refund's own line-item record in agreement by construction.

## Future reconsideration conditions

Revisit only if a future promotion type (dynamic pricing, DDD Section 17.9) requires a fundamentally different refund-proration rule than proportional/full-line reversal — a rule refinement within this same structure, not a boundary change, unless it requires new cross-module coordination.

## Related

- SRD section(s): FR-PRICE-011, BR-029, 10.8 (Refund State Machine)
- Related DDD entity/data area(s): 5.28 (Refund), 5.32 (Price Snapshot), 5.29–5.31 (Promotion, Promo Code, Promotion Usage)
- Related architecture principle(s): Principle 4.5, Principle 4.10
- Related ADRs: 0015, 0020, 0021, 0033, 0037
