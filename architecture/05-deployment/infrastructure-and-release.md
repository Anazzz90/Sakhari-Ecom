# Infrastructure & Release Strategy

**Stability:** Evolving — infrastructure topology and release process mature with team size and traffic.
**Status:** Skeleton — pending authoring.

## Purpose
Describes the production infrastructure topology (in prose — no diagrams at this phase) and how code moves from commit to production, including rollback strategy and disaster recovery approach.

## Relationship to other documents
- **SRD:** Directly constrained by the SRD's technology decisions (RDS not raw EC2, Saudi-region cloud, NestJS backend) and the solo-developer operating model.
- **environment-strategy.md:** This document describes the infrastructure *behind* each environment listed there.
- **SDD:** Per-service deployment specifics (container definitions, scaling rules) live in SDD documents and must be consistent with the topology described here.
- **Implementation:** CI/CD pipeline definitions and infrastructure-as-code should implement exactly what's described here — this document should never describe infrastructure that doesn't exist in code, or vice versa without an update.

## Sections to be authored
1. Infrastructure topology (compute, database, cache, object storage, networking — narrative form)
2. CI/CD pipeline stages
3. Release strategy (deployment method, rollback mechanism)
4. Disaster recovery (backup strategy, recovery targets)
