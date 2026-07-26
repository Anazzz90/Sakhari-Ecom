# 0014. UTC timestamp storage

**Decision ID:** ADR-0014
**Status:** Accepted
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
Saudi Arabia operates on Arabia Standard Time (UTC+3, no daylight saving), and the platform's delivery promise (10–20 minutes) and worker shift scheduling depend on precise, unambiguous duration and sequencing calculations.

## Problem
In what timezone should timestamps be stored and computed, given that display must be in local Saudi time while computation — delivery-time SLAs, ordering sequences, audit trails — must never be ambiguous or subject to drift from any future timezone or policy change?

## Options considered
1. **Store timestamps in local Saudi time (Arabia Standard Time) directly.** Reads naturally in the source system, but becomes ambiguous the moment any computation or future market crosses a timezone boundary.
2. **Store timestamps with no explicit timezone, assumed local by convention.** Cheapest to implement, but relies on every reader and writer sharing an unstated assumption — exactly the kind of implicit rule Constitution Section 3 warns against.
3. **Store all timestamps in PostgreSQL as UTC (timezone-aware) values**, converting to local time only at presentation in the client applications.

## Decision
All timestamps are stored in PostgreSQL as UTC (timezone-aware) values. Conversion to Arabia Standard Time, or any other local time, happens only at the presentation layer in the client applications.

## Rationale
Options 1 and 2 both make correctness dependent on every future reader and writer agreeing on an unstated timezone assumption — a violation of Constitution Section 3's "explicit beats implicit" conviction, and a direct risk to Principle 4.1: an SLA computed against an ambiguous timestamp is not correct, it is approximately correct. UTC storage is the industry-standard practice that removes the ambiguity entirely, keeps duration and ordering calculations exact, and keeps Principle 4.10's audit trail unambiguous regardless of who reads it or when.

## Consequences
- Duration and ordering calculations — delivery SLAs, audit-trail sequencing, event ordering (ADR-0011) — are unambiguous and require no timezone-aware arithmetic in the service layer.
- Every client is responsible for correct conversion to Arabia Standard Time for display, which must be tested explicitly rather than assumed correct.
- No business logic in the service layer reasons about "local time" internally — local time is a presentation concern only, consistent with Principle 4.6/4.7's boundary between business logic and presentation.
- Scheduled/background processing (for example, event consumers under ADR-0011) can run correctly regardless of the infrastructure's own local timezone configuration.

## Future reconsideration conditions
None anticipated. This is a foundational, low-cost-to-maintain convention consistent with the platform's plausible future geographic expansion, and reversing it after real data exists would be materially more expensive than maintaining it from the outset.

## Related
- SRD section(s): Section 1.2 (Saudi Arabia-only launch), delivery-time promise (10–20 minutes)
- Related DDD entity/data area(s): all entity areas carrying created/updated/lifecycle timestamps
- Related architecture principle(s): 4.1, 4.10
- Related ADRs: 0013
