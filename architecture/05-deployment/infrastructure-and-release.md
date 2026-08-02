# Infrastructure & Release Architecture — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. |
| **Stability** | Evolving — infrastructure topology and release process mature with real traffic and team size, within the philosophy this document sets (managed services, vertical scaling first, one deployable). |
| **Authority** | Subordinate to `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `decisions/README.md`, `02-context/system-context.md`, `04-cross-cutting/technology-decisions.md`, and `05-deployment/environment-strategy.md` (this document describes the infrastructure behind each environment that document lists). |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** Describes the production infrastructure topology and how code moves from commit to production — the deployment-layer detail every other architecture document has deliberately deferred here rather than covering itself. This is the document `02-context/system-context.md` (no deployment), `04-cross-cutting/data-architecture.md` (infrastructure-level backup/DR explicitly out of scope), and `04-cross-cutting/security-and-compliance.md` (secret-storage topology explicitly out of scope) all pointed to.

**Scope.** Infrastructure components, networking, storage, cache, logging, monitoring, backup, disaster recovery, CI/CD, and a scaling-strategy overview. Per this task's own instruction, cloud-provider detail stays abstract except where already established elsewhere (AWS, RDS, S3 — `04-cross-cutting/technology-decisions.md` Section 10, ADR-0003, ADR-0016): this document does not invent specific instance types, network topology, or a specific CI/CD tool where none has been decided — those gaps are named in Section 12 (Open Decisions) rather than assumed. Does not cover application-level scaling philosophy or capacity planning in depth — that's `04-cross-cutting/reliability-and-performance.md`'s subject; this document's Section 11 is a topology-level overview only, to avoid duplicating that document once it exists.

**Intended audience.** The project owner and any AI coding assistant working on deployment configuration, CI/CD pipeline definitions, or infrastructure-as-code — this document is what that code should match exactly; a divergence between what's described here and what's actually deployed is itself a signal something needs updating, in one document or the other.

**Cross-references.** Grounded in ADR-0002 (Modular Monolith — one deployable), ADR-0003 (PostgreSQL/RDS), ADR-0004 (Redis), ADR-0016 (S3), `04-cross-cutting/technology-decisions.md` Sections 5—7 and 10—11 (PostgreSQL, Redis, S3, Cloud Infrastructure, CDN), `environment-strategy.md` (the environments this topology serves), and `04-cross-cutting/data-architecture.md` Section 14 (data-level recovery, distinct from this document's infrastructure-level Section 9).

## 2. Production Environment — Topology Overview

Production consists of one Backend deployable (the modular monolith, ADR-0002) running against three managed data stores — PostgreSQL via RDS, Redis, and S3 — plus a CDN in front of public, cacheable content (product imagery, Customer Web App static assets), all provisioned within a Saudi-compliant AWS region (`04-cross-cutting/technology-decisions.md` Section 10). There is exactly one Backend to deploy, monitor, and reason about — a direct, deliberate consequence of the modular-monolith decision, not an oversight of a more elaborate topology.

## 3. Infrastructure Components

| Component | Role | Grounded in |
|---|---|---|
| Backend compute | Runs the single NestJS modular-monolith deployable, as an AWS ECS Fargate service behind an Application Load Balancer | ADR-0002, ADR-0030; `04-cross-cutting/technology-decisions.md` Section 4 |
| PostgreSQL (RDS) | System of record, Multi-AZ in Production | ADR-0003, ADR-0030 |
| Redis | Cache, sessions, rate limiting, short-lived coordination, via managed ElastiCache | ADR-0004, ADR-0030 |
| S3 (Object Storage) | Product imagery and file/media assets | ADR-0016 |
| CDN | Edge delivery for public, cacheable content, via CloudFront | `04-cross-cutting/technology-decisions.md` Section 11; ADR-0030 |
| Cloud Infrastructure Provider | Managed hosting substrate for all of the above, Saudi-compliant region (AWS `me-central-2`) | `04-cross-cutting/technology-decisions.md` Section 10; ADR-0024 |
| Secrets store | AWS Secrets Manager for production secrets | ADR-0030 |

Per ADR-0030, the Backend deployable runs as an ECS Fargate service — a managed container platform, not a self-managed EC2/Kubernetes cluster — chosen so a solo operator is not also managing servers or a Kubernetes control plane.

## 4. Networking

Per ADR-0030: a VPC separates public subnets (the ALB and CloudFront's edge) from private subnets (ECS tasks, RDS, and Redis) — the data stores and the Backend compute layer are never reachable directly from the public internet or a client, consistent with `02-context/system-context.md` Section 6's "Infrastructure is reachable only from the Backend" rule, now expressed as a networking-level requirement, not just an application-level one. Exact security-group rules and subnet CIDR allocation remain SDD/infrastructure-as-code-level detail, not architecture-level decisions.

## 5. Storage

PostgreSQL (RDS) and S3 are the two durable storage layers, each holding exactly what `04-cross-cutting/data-architecture.md` Sections 2 and 12 already establish they should — business facts in PostgreSQL, media/file assets in S3, with S3 objects referenced (never embedded) from PostgreSQL rows (ADR-0016). This document adds nothing to that division; it only confirms that the storage layer's physical form (managed RDS storage, managed S3 buckets) matches the architectural division, not a topic requiring separate rules.

## 6. Cache

Redis is deployed as a managed service, sized and scoped exactly as `04-cross-cutting/data-architecture.md` Section 3 and ADR-0004 already define: caching, sessions, rate limiting, and short-lived coordination — never a durable store. At the infrastructure level, this means Redis's own persistence settings (if any) are an operational convenience for warm-restart performance, never a reason to treat Redis as recoverable business data — if Redis's underlying storage were lost entirely, the system's correctness is unaffected, only its warm-cache performance (Section 3's own boundary, restated at the infrastructure layer).

## 7. Logging

Every component logs in a structured format, with a consistent minimum field set (timestamp, actor/context where applicable, operation, outcome) across all sixteen modules — one logging discipline, not sixteen ad hoc ones, consistent with Constitution Section 11's "observability is not optional." Two distinctions matter architecturally:

- **Application logs are not the audit trail.** `security-and-compliance.md` Section 9 and `data-architecture.md` Section 15 already establish the Audit module as the durable, structural record of consequential actions. Application logs are an operational troubleshooting aid — useful, but disposable and not subject to the same immutability/retention discipline as the Audit Log.
- **PII discipline applies to logs, not just the database.** Anything that would be sensitive in PostgreSQL (a customer's phone number, an address) is treated the same way in logs — redacted or omitted, not casually logged "for debugging," consistent with `security-and-compliance.md` Section 11's PDPL obligations extending everywhere data can end up, not just the primary datastore.

## 8. Monitoring

At the architecture level, Production is monitored for: the Backend's health and availability, PostgreSQL/Redis's health and resource saturation, and the external integrations' (`02-context/system-context.md` Section 7) own availability, since a Payment Gateway or SMS/OTP Provider outage is functionally an outage of whatever depends on it. Alerting philosophy — what pages a solo developer immediately versus what waits for a daily review — and incident-response process are explicitly `04-cross-cutting/observability-and-operations.md`'s deeper subject, not yet authored; this document establishes only *what* is monitored, not *how alerts are triaged or escalated*.

## 9. Backup

PostgreSQL (RDS) uses automated, managed backups with point-in-time recovery, consistent with the operational-simplicity design goal (Constitution Section 5) that motivated choosing a managed database in the first place (ADR-0003). Per ADR-0030/ADR-0031: Production backup retention is **35 days**, with point-in-time recovery enabled throughout that window. S3 relies on the durability guarantees of the managed object storage service itself (ADR-0016) rather than a separate, custom backup process. Restore verification is exercised **monthly** during launch stabilization and **quarterly** once operations are stable (ADR-0030, ADR-0031) — an unverified backup is not treated as a real backup.

## 10. Disaster Recovery

This is the infrastructure-level counterpart to `data-architecture.md` Section 14's data-level recovery mechanisms (compensating actions, reconciliation jobs, the ledger/audit trail). Per ADR-0031, the launch RPO/RTO targets are:

| Data class | RPO | RTO |
|---|---|---|
| PostgreSQL transactional core / audit / ledger | <= 5 minutes | Database restore <= 4 hours |
| Backend service (compute) | Not applicable (stateless) | Service restore <= 60 minutes |
| Derived/rebuildable (Search index, Analytics rollups) | Not applicable — regenerated from source events | Regeneration time, not a restore time |
| Redis (cache/session) | Not meaningful by design | Not meaningful by design |

- **Recovery approach:** restore PostgreSQL from its managed, automated backup (Section 9) to the most recent recoverable point within the 5-minute RPO; S3 relies on the object storage service's own durability rather than a restore procedure. A single-region deployment (appropriate at current scale, Constitution Section 6/13) is made safe by disciplined backup practice rather than multi-region redundancy the business does not yet need — `01-Architecture-Design-Specification.md` Section 14's own framing, restated here as the operative approach.
- **Restore drill cadence:** monthly until stable production launch, quarterly afterward (ADR-0030, ADR-0031), matching Section 9's backup-verification cadence.
- **Multi-region scope:** explicitly out of scope for MVP launch — a single-region deployment is the accepted trade-off (Section 13); multi-region is reconsidered only under sustained, evidenced regional-outage pressure, per ADR-0031's Future Reconsideration Conditions.

## 11. CI/CD

Code moves through the promotion path `environment-strategy.md` Section 4 establishes (Development ? Staging ? Production, no environment skipped) via an automated pipeline with, at minimum, these stages: automated tests run on every change; a build step produces the deployable artifact; the artifact deploys to Staging; and only after Staging verification does the same artifact promote to Production — the same built artifact, not a separate Production-specific build, so what was verified in Staging is exactly what reaches Production. Per ADR-0030, **GitHub Actions** is the CI/CD platform, matching the project's existing GitHub-based workflow.

**Release strategy:** deployments are released as a single Backend artifact per ADR-0002's one-deployable model. Rollback is a redeploy of the previous artifact version — made safer than it would otherwise be by ADR-0017's versioned API discipline, since a rolled-back Backend version continues serving whichever API version clients already depend on, rather than clients being stranded mid-release the way an unversioned API would risk.

## 12. Scaling Strategy — Overview

At the infrastructure level: the Backend, being a single deployable (ADR-0002), scales vertically first (a larger compute instance) and, once justified by evidence, horizontally (multiple instances of the same deployable behind a load balancer) — consistent with Constitution Section 6's explicit preference for vertical scaling and simple caching before horizontal or distributed complexity. PostgreSQL scales vertically first as well, with read replicas as the next step once read load specifically (not write load) justifies it. This is a topology-level summary only — the deeper reasoning (why vertical-first, what specifically would trigger horizontal scaling or read replicas, and the eventual module-extraction path) belongs to `04-cross-cutting/reliability-and-performance.md`, not repeated here.

## 13. Trade-offs and Future Evolution

**Trade-offs accepted:** a single-region deployment is simpler and cheaper to operate than a multi-region one, at the cost of regional-outage exposure that a solo developer accepts at launch scale rather than paying multi-region complexity for upfront (Constitution Section 13). Managed services (RDS, Redis, S3) cost more per unit of raw compute/storage than self-managed alternatives, in exchange for removing patching, failover, and backup operations from manual responsibility — the same trade-off `04-cross-cutting/technology-decisions.md` Section 10 already names for the cloud provider generally, restated here as it applies to infrastructure operations specifically.

**Future evolution:** as real traffic materializes, this document's Sections 8—12 (monitoring, backup, DR, CI/CD, scaling) are the ones expected to be tightened with real parameters, real tooling choices, and real drill procedures — the topology and philosophy in Sections 2—7 are expected to hold, with growth handled by extension (more compute, read replicas, a second region if evidence ever demands it) rather than a redesign, consistent with Constitution Section 13's "earn complexity."

## 14. Open Decisions

- **Exact VPC/subnet CIDR allocation and security-group rules** (Section 4) remain infrastructure-as-code-level detail, within the topology ADR-0030 establishes.
- **Monitoring/alerting tool choice and alert-triage philosophy** — detailed ownership belongs to `04-cross-cutting/observability-and-operations.md`.
- Compute service, network topology, backup retention, RPO/RTO targets, restore cadence, and CI/CD platform — previously open here — are resolved by ADR-0030 and ADR-0031 (Sections 3, 4, 9–11 above) and must not be treated as open going forward.

