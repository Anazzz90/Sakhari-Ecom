# 0031. Production readiness targets

**Status:** Accepted

## Context

The NFR matrix, reliability, infrastructure, and operations documents all identified the absence of numeric targets as the largest production-readiness gap.

## Problem

Without measurable targets, implementation cannot know whether performance, availability, recovery, and operational behavior are acceptable.

## Decision

Adopt the following initial production-readiness targets for MVP launch:

| Area | Target |
|---|---|
| API latency, general authenticated API | p95 <= 300 ms, p99 <= 800 ms, excluding external provider wait time |
| Checkout orchestration API | p95 <= 1.5 s, p99 <= 3 s, excluding customer-side payment authorization time |
| Catalog/search customer reads | p95 <= 300 ms, p99 <= 700 ms |
| Rider location ingest | p95 <= 250 ms accepted by backend |
| Availability, transactional core | 99.9% monthly |
| Availability, customer browse/search | 99.5% monthly |
| Availability, notification/analytics consumer tier | 99.0% monthly |
| RPO, PostgreSQL transactional core/audit/ledger | <= 5 minutes |
| RTO, backend service restore | <= 60 minutes |
| RTO, database restore | <= 4 hours |
| Checkout success-rate alert | Critical if below 95% for 10 minutes excluding customer payment declines |
| Payment provider technical failure alert | High above 5% for 10 minutes; Critical above 15% |
| Event outbox backlog alert | High if oldest pending event > 5 minutes; Critical if > 15 minutes for checkout/fulfillment/payment events |
| Restore drill cadence | Monthly until stable production launch, quarterly afterward |
| Correctness | Zero tolerance for financial, inventory, audit, or ledger corruption |

These are launch targets, not permanent enterprise SLAs. They may be tightened with production evidence through a future ADR.

## Rationale

The targets are strict enough to gate implementation but modest enough for a single-region, modular-monolith MVP. Correctness remains non-negotiable; latency and availability targets are measured within that constraint.

## Consequences

- Performance tests and monitoring must be built around these numbers.
- Alert thresholds are no longer left to implementation guesswork.
- The production deployment must support RPO/RTO validation.

## Future Reconsideration Conditions

Revisit after real traffic, store count, and incident history provide better evidence.

## Related Documents

- Related architecture principle(s): Principle 4.1, Principle 4.10, Principle 4.12
- Related ADRs: 0002, 0003, 0021, 0029, 0030

