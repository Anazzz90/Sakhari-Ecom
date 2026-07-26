# 0024. AWS me-central-2 deployment region

**Status:** Accepted

## Context

The SRD selects AWS `me-central-2` in Riyadh for production hosting to satisfy Saudi data residency expectations and reduce latency for the Saudi launch.

## Problem

Architecture docs referenced AWS generally and sometimes treated the cloud region as undecided, even though the SRD finalized AWS Riyadh as the production region.

## Options Considered

1. Keep cloud provider and region generic.
2. Use a non-Saudi region.
3. Use AWS `me-central-2` for production.

## Decision

Production infrastructure is hosted on AWS in `me-central-2` (Riyadh) unless a future ADR supersedes this decision.

## Rationale

This aligns with PDPL/data residency expectations, keeps customer and order data in-Kingdom by default, and supports the SRD's AWS/RDS/S3 direction while keeping operations manageable for a solo developer.

## Consequences

- PostgreSQL/RDS, Redis, object storage, backups, and durable business data must remain in the Saudi region by default.
- External SaaS or analytics tools that move personal data outside Saudi Arabia require explicit review before adoption.
- Multi-region deployment is not part of launch architecture.

## Future Reconsideration Conditions

Reconsider only for a major provider failure, compliance change, significant cost issue, or expansion outside Saudi Arabia.

## Related Documents

- Related SRD section(s): Cloud/region, Data Protection, Infrastructure
- Related DDD entity/data area(s): All durable business and personal data
- Related architecture principle(s): Principle 4.1, Principle 4.2, Principle 4.12

