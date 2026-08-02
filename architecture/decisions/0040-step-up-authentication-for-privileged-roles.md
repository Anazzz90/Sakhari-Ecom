# 0040. Step-up authentication for privileged roles

**Decision ID:** ADR-0040
**Status:** Accepted
**Date:** 2026-07-30
**Deciders:** Principal Architect (solo-project decision authority), remediating ARR Critical Finding #6

## Context

OTP (ADR-0027) is the sole authentication factor for every actor, including Admin/Ops/Support staff holding `refund:approve`, `settings:manage`, and similar high-blast-radius permissions (`security-and-compliance.md` Section 7's permission catalogue). ADR-0027 names SIM-swap as an accepted risk with no compensating control. The ARR found this disproportionate: a SIM-swapped or otherwise compromised privileged-staff phone number grants an attacker refund-approval and settings-management capability behind the same single factor as an ordinary customer login.

## Problem

What additional authentication is required before a privileged actor may perform a high-risk operation, and which roles does it apply to?

## Options considered

1. **Require a second factor for every Admin/Ops/Support login, unconditionally.** Rejected: this changes the baseline login experience for every staff member on every session, including low-risk, read-only operations (`support:read`, `analytics:read`) that don't warrant the added friction — a blanket requirement is heavier than the risk it addresses.
2. **Require step-up authentication only at the moment a high-risk operation is attempted, scoped to the specific privileged roles that can perform one.** Chosen.
3. **Rely on shorter access-token TTLs alone for privileged roles.** Rejected: TTL reduction (already 15 minutes per ADR-0033) limits a *stolen token's* window but does nothing against a *compromised OTP channel itself* (SIM-swap) being used to establish a fresh, fully legitimate session in the first place — a different threat this decision specifically targets.

## Decision

Customers continue to authenticate via OTP only (ADR-0027 unchanged for the Customer role). Four privileged roles require **step-up authentication** — a second, independent verification, performed in addition to the actor's existing OTP-authenticated session — before performing a high-risk operation:

- **Super Admin**
- **Operations Manager**
- **Finance**
- **Support Lead**

These are privileged classifications within the existing Admin/Ops/Support role categories (`security-and-compliance.md` Section 7) — this decision does not introduce new RBAC role categories or restructure the permission catalogue; it identifies which existing role-holders are subject to step-up before specific operations, layered on top of the permission check those operations already require.

Step-up authentication may use either a **TOTP** (time-based one-time password, app-generated) or an **approved enterprise MFA** mechanism — the specific provider/mechanism is an SDD-level, not architecture-level, choice, consistent with how this document treats other credential mechanics (Section 1's scope exclusions).

**High-risk operations requiring fresh step-up authentication** (an operation performed within a bounded, short window after step-up verification — not a standing grant):
- Refund approval (`refund:approve`)
- Permission changes (role/permission grants or revocations)
- System settings changes (`settings:manage`)
- Financial adjustments (Payment Ledger `Adjustment`/`Chargeback` entries, per ADR-0037)
- Sensitive audit access (`audit:read` against another actor's detailed activity, distinct from an actor's own routine operational views)

A step-up challenge is independent of, and does not replace, the actor's underlying OTP-authenticated session (Section 4) or the RBAC permission check (Section 7) already required for the operation — it is an additional gate, evaluated fresh at the moment of the high-risk operation, not cached across a session the way a normal permission claim is.

## Rationale

Scoping step-up to specific roles and specific high-risk operations, rather than every privileged login, keeps the added friction proportional to the actual blast radius (Principle 4.1's correctness-over-convenience ordering, applied here to security rather than performance) — a Support Lead reading tickets is not asked to re-verify; a Support Lead approving a refund is. Requiring step-up to be *fresh* (evaluated at the moment of the operation, not a standing session flag) closes the specific SIM-swap gap ADR-0027 named: even a fully compromised phone number and a valid OTP-established session cannot alone perform a high-risk operation without also passing the independent step-up challenge.

## Consequences

- `security-and-compliance.md` gains a new section (Step-Up Authentication) under Authentication, defining the four roles, the high-risk operation list, and the freshness requirement.
- `module-catalog.md`: Auth's Public Interfaces gain `InitiateStepUpChallenge` and `VerifyStepUpChallenge`; Auth's Published Events gain `StepUpAuthenticationCompleted` and `StepUpAuthenticationFailed`, both within audit scope (`security-and-compliance.md` Section 9).
- Payment's `IssueRefund` (for `refund:approve`), Settings' `UpdateSetting`, and any permission-management operation must each verify a fresh step-up completion before executing, in addition to their existing RBAC permission check — an SDD-level integration point for each owning module, not a new cross-module dependency (step-up verification is a call to Auth, which every module already depends on for `ValidateToken`).
- `nfr-matrix.md` Section 3.7 (Security) gains a reference to step-up as part of the acceptance criteria for privileged-role protection.
- `observability-and-operations.md`: step-up completion/failure is added to the Security Logs category (Section 3) alongside existing authentication-event logging.

## Future reconsideration conditions

Revisit if evidence shows the four named roles are too coarse (a role needs step-up for only a subset of its own permissions) or too narrow (a fifth role gains a high-risk permission and needs inclusion) — extended by amending the role/operation lists directly, since this is an operational-detail list rather than a structural boundary; a genuinely new authentication mechanism (passkeys, hardware keys) replacing TOTP/enterprise MFA would warrant a superseding ADR.

## Related

- SRD section(s): FR-AUTH-004 (RBAC), Section 12 (Security requirements — "admin 2FA available")
- Related DDD entity/data area(s): 5.1 (User)
- Related architecture principle(s): Principle 4.1, Principle 4.6, Principle 4.10
- Related ADRs: 0012, 0026, 0027, 0033, 0037, 0039
