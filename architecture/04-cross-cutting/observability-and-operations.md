# Observability & Operations

**Stability:** Evolving — tooling and thresholds mature with real production experience.
**Status:** Skeleton — pending authoring.

## Purpose
Defines how the system is observed (logging, metrics, tracing), alerted on, and operated day-to-day by a single developer — what "healthy" looks like and how issues are noticed before customers report them.

## Relationship to other documents
- **SRD:** Informed by the SRD's solo-developer constraint — operational tooling must be low-overhead by design, not enterprise-scale tooling.
- **reliability-and-performance.md:** Observability is how the targets defined there are verified in production.
- **SDD:** Each service's SDD should state what it logs/emits and how it satisfies the standards set here.
- **Implementation:** Logging/metrics instrumentation in code should follow the conventions defined here (structured log format, required fields, naming).

## Sections to be authored
1. Logging standard (format, required fields, PII redaction rules — tied to security-and-compliance.md)
2. Metrics & dashboards (what's measured, at what layer)
3. Alerting philosophy (what pages a solo developer vs. what waits for morning)
4. Incident response process (lightweight, solo-appropriate)
