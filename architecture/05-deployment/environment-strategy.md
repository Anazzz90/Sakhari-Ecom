# Environment Strategy

**Stability:** Stable once written — environment topology (dev/staging/prod) rarely changes.
**Status:** Skeleton — pending authoring.

## Purpose
Defines the set of environments the system runs in, what differs between them, and how configuration/secrets vary per environment.

## Relationship to other documents
- **SRD:** Region/residency constraints (Saudi Arabia data residency) apply to every environment, not just production — stated here explicitly.
- **security-and-compliance.md:** Secrets-per-environment rules must follow that document's credential management section.
- **SDD:** Per-service configuration documentation should reference this document's environment list rather than redefining it.
- **Implementation:** CI/CD environment variables and deployment targets should map 1:1 to what's declared here.

## Sections to be authored
1. Environment list and purpose of each
2. What varies per environment (data, scale, external integration mode e.g. sandbox vs. live payment gateway)
3. Promotion path between environments
