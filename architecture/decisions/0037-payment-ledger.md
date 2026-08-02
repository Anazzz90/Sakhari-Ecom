# 0037. Payment ledger

**Decision ID:** ADR-0037
**Status:** Accepted
**Date:** 2026-07-30
**Deciders:** Principal Architect (solo-project decision authority), remediating ARR Critical Finding #3

## Context

`data-architecture.md` Section 8 gives Inventory an append-only ledger specifically because "current state... can only ever answer 'what is true now,' never 'why'" (DDD Section 3.7). The ARR found the identical argument was never applied to money: Payment owns `payments`, `payment_history`, `cash_remittances`, `card_on_delivery_records`, and `refunds` (`data-ownership-map.md` §12.9) as independent tables with no unifying ledger entity proving a payment's authorized/captured/refunded amounts reconcile, and no mechanism proving system-wide money-in equals money-out. ADR-0015 addresses only money's *representation* (integer halala), never its *bookkeeping structure*.

## Problem

How does the system durably, immutably record every financial movement against an order, so that Payment's state can be *explained*, not just *observed* — the same guarantee Inventory's ledger already provides for stock?

## Options considered

1. **Rely on Payment History alone.** Rejected: Payment History (DDD Section 5.25) is scoped to "payment state changes and provider results" — a state-transition log for a single Payment record, not a movement-level financial ledger spanning payments, refunds, COD/card-on-delivery collections, and settlement across an order's full financial lifecycle. It answers "what state did this payment move through," not "what is the complete, itemized financial history of this order."
2. **Add reconciliation logic that recomputes totals on demand from the existing five tables.** Rejected: this recreates, at query time and without durability, exactly the "explainable, not just visible" guarantee a ledger provides for free — and every future reconciliation job would need to independently re-derive the same join logic, with no single durable source of truth for "what happened, in order."
3. **Introduce a Payment Ledger entity, structurally mirroring the Inventory Ledger, append-only and immutable, recording every financial movement as its own entry.** Chosen.

## Decision

Payment gains a new owned entity: the **Payment Ledger**. Every financial movement against an order generates an immutable ledger entry, including at minimum: Payment Authorized, Payment Captured, COD Collected, Card-on-Delivery Collected, Refund Issued, Refund Reversed, Settlement Recorded, Adjustment, and Chargeback.

Each ledger entry contains: Ledger ID, Payment ID, Order ID, Entry Type, Amount (integer halala, per ADR-0015), Currency, Reference (gateway/provider reference where applicable), Source (which actor or system process produced the entry), and Timestamp (UTC, per ADR-0014).

Ledger entries are immutable — never updated or deleted, exactly as the Inventory Ledger and Audit Log already require (`data-architecture.md` Section 9). Payment aggregates (a payment's current authorized/captured/refunded totals) are derived from ledger history where appropriate — the ledger is the durable record of *why* a payment's current state is what it is; the Payment entity's own current-state fields remain the fast-path answer to "what is true now," exactly as Inventory Item and Inventory Ledger divide that same responsibility today (`data-architecture.md` Section 6).

Reconciliation against payment-provider settlement reports (Moyasar, per ADR-0022, and any future POS/terminal settlement source, per ADR-0032) is performed against the Payment Ledger, the same way Inventory Reconciliation (DDD Section 15.4) is performed against the Inventory Ledger — a scheduled job comparing the ledger's recorded movements against the provider's own settlement record, flagging discrepancies for review rather than silently accepting them.

## Rationale

This applies `data-architecture.md` Section 8's own stated reasoning ("without it, a current quantity is a number with no history behind it — correct until it's wrong, with no way to determine when or why") identically to money, closing the exact asymmetry the ARR identified. It costs nothing architecturally that Inventory's ledger didn't already cost: one new append-only table, owned exclusively by Payment (no new module, no ownership boundary change), following the same soft-delete/immutability rule (`data-architecture.md` Section 9) and the same outbox-backed durability discipline (Section 16) every other consequential Payment write already follows.

## Consequences

- `data-architecture.md`: gains a Payment Ledger section, structurally parallel to Section 8's Inventory Ledger, and an explicit reconciliation-against-settlement-reports description.
- `data-ownership-map.md`: Payment's Owned Tables gains `payment_ledger`; Payment's Owned Entities and the Ownership Matrix (Section 13) gain a Payment Ledger row, marked "— (internal history)" exactly as Inventory Ledger already is.
- `module-catalog.md`, `capability-boundary-map.md`, `service-decomposition.md`: Payment's Ownership/Data Ownership fields gain Payment Ledger; Payment's Published Events gain `PaymentLedgerRecorded`, mirroring `InventoryLedgerRecorded`; Payment's Repositories (service-decomposition) gain a `PaymentLedgerRepository`, and Payment's Internal Services gain a `LedgerRecordingService`, mirroring Inventory's own internal-service shape exactly.
- `DDD_Sakhari_Ecom_v1.0.md`: gains a new entity (Payment Ledger), following the same numbering-by-addition convention ADR-0034 established (new entities appended, not inserted mid-sequence, to avoid renumbering every cross-reference).
- Payment's SDD (when written) must model ledger-entry recording as part of the same transaction as the financial-state change it records (Payment locks its own Payment row before writing to the ledger, consistent with `data-architecture.md` Section 13's existing per-module lock ordering for Payment).
- Every existing Payment public interface that changes financial state (`ConfirmPayment`, `RecordCashCollection`, `RecordCardOnDeliveryCollection`, `IssueRefund`) now also writes a ledger entry — an internal implementation obligation on Payment's own service layer, not a new public interface.

## Future reconsideration conditions

Revisit only if evidence shows ledger write volume requires partitioning/archival sooner than the Inventory Ledger's own eventual archival need (DDD Section 16) — handled by the same scaling mechanism, not a redesign.

## Related

- SRD section(s): 10.2 (Payment State Machine), 3.2 (COD/Card-on-Delivery reconciliation)
- Related DDD entity/data area(s): 5.15 (Inventory Ledger, structural precedent), 5.24–5.27 (Payment, Payment History, Cash Remittance, Card-on-Delivery Record); new entity Payment Ledger
- Related architecture principle(s): Principle 4.5 (module data ownership), Principle 4.10 (auditability)
- Related ADRs: 0009, 0014, 0015, 0020, 0022, 0029, 0032
