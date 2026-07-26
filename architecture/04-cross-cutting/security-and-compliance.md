# Security & Compliance Architecture

**Stability:** Stable baseline, evolving detail — the regulatory obligations (SAMA/PDPL) are stable; specific controls evolve as threats and features change.
**Status:** Skeleton — pending authoring.

## Purpose
Defines the system-wide approach to authentication, authorization, secrets management, data protection, and regulatory compliance (SAMA, PDPL) — the rules every module/runtime unit must follow, not a module-level implementation.

## Relationship to other documents
- **SRD:** Directly implements the compliance obligations named in the SRD (mada/SAMA/PDPL, Saudi data residency). This document is where those business-level obligations become engineering rules.
- **DDD:** References which entities carry personal/sensitive data, informed by entity definitions and retention requirements there.
- **SDD:** Each service's SDD must document how it satisfies the rules set here (e.g., how it authenticates callers, how it stores secrets).
- **Implementation:** Auth middleware, encryption-at-rest configuration, and access-control code should be traceable to a rule in this document.

## Sections to be authored
1. Identity & authentication model (customer, worker, admin — token strategy)
2. Authorization model (roles/permissions boundaries per the SRD actor table)
3. Data protection (encryption in transit/at rest, PII handling, residency)
4. Secrets & credential management
5. Compliance mapping (SAMA, PDPL) — obligation → control
