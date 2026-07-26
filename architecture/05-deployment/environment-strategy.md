# Environment Strategy — Sakhari Ecom

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Authored. |
| **Stability** | Stable — the environment list and promotion path are expected to hold for the life of the project; what evolves underneath them is captured in `infrastructure-and-release.md`, not here. |
| **Authority** | Subordinate to `00-Architecture-Principles.md`, `01-Architecture-Design-Specification.md`, `decisions/README.md`, `02-context/system-context.md`, `04-cross-cutting/technology-decisions.md`, and `04-cross-cutting/security-and-compliance.md` (for secrets handling per environment). |

## 1. Purpose, Scope, and Intended Audience

**Purpose.** Defines the "Production Environment" question at the level this document owns: what environments exist, what's true of each, what varies between them, and how a change moves from one to the next. `infrastructure-and-release.md` describes the infrastructure *behind* each environment named here — this document is the list and the rules, that one is the topology.

**Scope.** Environment list and purpose, what varies per environment (data, external-integration mode, logging verbosity), and the promotion path. Does not cover infrastructure topology, CI/CD pipeline mechanics, backup, disaster recovery, or scaling — all `infrastructure-and-release.md`'s subject (Section 2 below). Does not cover secrets storage mechanics — `security-and-compliance.md` Section 10 already establishes that secrets are never in code or version control; this document only states which environment holds which credentials conceptually.

**Intended audience.** The project owner and any AI coding assistant configuring environment-specific behavior (which payment gateway mode to call, which log verbosity to use) — this document is the source of truth for "what environment am I in and what does that mean," not something to redefine per feature.

**Cross-references.** Grounded in the Constitution's operational-simplicity design goal and one-developer constraint (Section 5, Section 7); `02-context/system-context.md`'s external systems (Payment Gateway, SMS/OTP Provider, Push Notification Service, Geocoding Provider), each of which behaves differently by environment (Section 3 below); `security-and-compliance.md` Section 10 (secrets).

## 2. Environment List

Three environments, matched to a one-developer operation rather than a larger team's staging topology:

| Environment | Purpose |
|---|---|
| **Development** | Local or per-developer, used for active feature work. Talks to sandbox/test modes of every external integration; never touches real customer data or real money. |
| **Staging** | A production-shaped environment for verifying a change before it reaches real customers — the last checkpoint in the promotion path (Section 4). Talks to sandbox modes of external integrations wherever the integration supports one (Payment Gateway, SMS/OTP Provider), so a staging run never sends a real SMS or moves real money. |
| **Production** | The live environment real customers, workers, and admins use. The only environment authorized to hold real customer data, process real payments, and send real notifications. |

**Why three, not more:** a larger team might add a dedicated QA environment, per-feature ephemeral environments, or a separate performance-testing environment. None of those are justified yet at this project's scale (Constitution Section 6's "vertical scaling and simple caching are preferred... added complexity is deferred until load actually demands it" applies to environment topology as much as to infrastructure) — three environments is the minimum that separates "code I'm actively changing," "code I'm about to trust," and "code real people depend on," and no further split is earned yet.

## 3. What Varies Per Environment

- **External integration mode.** Development and Staging call sandbox/test modes of the Payment Gateway and SMS/OTP Provider wherever the provider supports one; Production calls live. This is the single most important variation in the whole environment strategy — it's what makes it structurally impossible for a staging run to charge a real card or notify a real customer, rather than relying on developer discipline alone.
- **Data.** Development and Staging never contain real customer PII or real payment data — Staging is production-*shaped* (same schema, same scale characteristics where feasible) but populated with synthetic or anonymized data, consistent with `security-and-compliance.md` Section 11's PDPL data-minimization obligation extending to non-production environments, not just production.
- **Logging verbosity.** Development and Staging may log more verbosely (including request/response detail useful for debugging) than Production, which follows the structured, PII-conscious logging discipline `infrastructure-and-release.md` Section 6 describes.
- **Secrets.** Every environment holds its own distinct set of credentials for every external integration — a Staging credential is never a Production credential reused, and vice versa, consistent with `security-and-compliance.md` Section 10's per-module credential isolation extended across environments.

## 4. Promotion Path

Code moves Development → Staging → Production, in that order, with no environment skipped — the mechanics of *how* (CI/CD pipeline stages, automated testing gates) belong to `infrastructure-and-release.md` Section 9. What this document establishes is the rule: nothing reaches Production without having first run, unmodified, in Staging against sandboxed external integrations. This is the practical expression of Constitution Principle 4.1 (correctness over raw performance) applied to the release process itself — a faster path that skips Staging is explicitly rejected, not merely undocumented.

## 5. Trade-offs and Future Evolution

**Trade-off accepted:** three environments, with Staging as the only pre-production gate, is simpler to operate than a larger topology but means Staging alone carries the full weight of catching problems before Production — there is no additional QA environment to catch what Staging misses. This is an accepted cost of operational simplicity for a one-developer team (Constitution Section 5's Design Goal), not an oversight.

**Future evolution:** additional environments (a dedicated performance-testing environment, ephemeral per-feature environments) would only be added once a specific, evidenced gap in the three-environment model causes real problems — consistent with Constitution Section 13's "complexity is earned." Any such addition should be recorded as an update to this document's Section 2, not treated as a routine operational tweak.

## 6. Open Decisions

- **Exact Staging data refresh/anonymization process** (how synthetic or anonymized data gets into Staging, and how often) is not decided here — an operational procedure, not an architectural one, best defined once real production data volume exists to anonymize from.
- **Whether Development is strictly local-per-developer or includes a shared cloud-hosted development environment** is not decided — moot at a one-developer team size today, worth revisiting only if the team grows (Constitution Section 13).

