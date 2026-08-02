# 0039. OTP abuse protection defaults

**Decision ID:** ADR-0039
**Status:** Accepted
**Date:** 2026-07-30
**Deciders:** Principal Architect (solo-project decision authority), remediating ARR Critical Finding #5

## Context

`security-and-compliance.md` Section 8 stated only that "rate limiting is enforced using Redis," with no concrete parameters. Since OTP (ADR-0027) is the sole authentication factor for every actor in the system, including privileged Admin/Ops/Support staff, undefined limits leave a live SMS-pumping-fraud and brute-force surface open — an attacker can cycle phone numbers to drain SMS budget or brute-force a short numeric code with no documented ceiling on attempts.

## Problem

What are the specific, numeric limits governing OTP request rate, verification attempts, and lockout behavior?

## Options considered

1. **Leave limits as an implementation-time (SDD-level) decision.** Rejected: this is exactly the pattern the ARR flagged as producing undefined, inconsistent behavior across implementers — the same reasoning ADR-0031 already applied to close latency/RPO/RTO ambiguity applies here.
2. **Adopt industry-standard OTP defaults (comparable to typical SMS-OTP fintech/e-commerce practice) as production defaults, explicitly configurable rather than hardcoded.** Chosen.
3. **Set stricter limits than industry-typical, prioritizing abuse-resistance over customer convenience.** Rejected as the default: stricter limits (e.g., 2 requests/hour) would materially degrade legitimate customer experience (a customer who mistypes an address and needs to retry checkout) for a threat model better addressed by IP/device throttling and progressive backoff than by punishing normal retry behavior.

## Decision

Auth enforces the following production defaults, all configurable through Settings (per ADR-0033's existing Settings-reader pattern — Auth is already a Settings reader) rather than hardcoded:

| Control | Default |
|---|---|
| Maximum OTP requests per phone number | 5 per hour |
| Maximum verification attempts per issued code | 5 per code |
| Resend cooldown | 60 seconds between requests |
| Temporary lockout | 30 minutes, triggered after the request or attempt ceiling is reached |
| Device- and IP-based throttling | Enabled, independent of and in addition to the phone-number-scoped limits above |
| Progressive backoff | Applied after repeated failures within a lockout-eligible window, increasing the effective cooldown before the next allowed attempt |
| OTP length | 6 digits |
| OTP expiry | 5 minutes from issuance |

Redis continues to enforce these limits (`security-and-compliance.md` Section 8, `data-architecture.md` Section 3) — rate-limit counters remain non-authoritative, coordination-only state; losing them on a Redis flush degrades to a temporarily looser limit, never an incorrect authorization decision, consistent with Redis's existing architectural role.

## Rationale

Numeric defaults close the exact gap the ARR identified while remaining configurable — the same pattern ADR-0034 used for batching-eligibility thresholds (Settings-owned, not hardcoded) applied here to a security control instead of an operational one. Phone-scoped and device/IP-scoped throttling operating independently means an attacker cannot bypass phone-number limits by simply rotating numbers from a single device, nor bypass device limits by rotating devices against a single phone number. Progressive backoff on top of a flat lockout window specifically addresses distributed, slow-drip abuse attempts that a flat window alone would not catch.

## Consequences

- `security-and-compliance.md` Section 4 (Authentication) gains this table as the concrete parameters underneath its existing rate-limiting statement.
- Auth's SDD (when written) must implement all eight controls together, not a subset — device/IP throttling is additive to, not a substitute for, phone-scoped limits.
- Settings gains new configuration keys for each numeric value in the table above, consistent with its existing role as the platform's operational-knob owner (`module-catalog.md` §4.16).
- Security logging (`observability-and-operations.md` Section 3) already names "repeated failed authentication" as a logged pattern — this decision gives that pattern concrete thresholds to alert against.

## Future reconsideration conditions

Revisit via a superseding ADR if production evidence shows the defaults are either too permissive (measurable abuse incidents) or too strict (measurable legitimate-customer lockout complaints) — tightened or loosened with evidence, never adjusted speculatively.

## Related

- SRD section(s): FR-AUTH (OTP-based authentication)
- Related DDD entity/data area(s): 5.1 (User — identity/credential record)
- Related architecture principle(s): Principle 4.6 (centralized security), Principle 4.1
- Related ADRs: 0004, 0012, 0023, 0027, 0033
