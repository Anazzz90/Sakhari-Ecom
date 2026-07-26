# 0007. Next.js for Customer Web App

**Decision ID:** ADR-0007
**Status:** Accepted
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
The SRD finalizes the Customer Web App as a Next.js web application, which together with the Customer Mobile App (ADR-0006) forms the Customer Platform: two presentations of one product sharing the same backend APIs, authentication flow, and business rules end to end (Principle 4.6).

## Problem
What web framework should serve the customer-facing website (browse, cart, checkout, order history, profile, addresses, support, promotions), such that it stays in lockstep with the Customer Mobile App's business behavior and fits the solo-developer, TypeScript-centered constraints already established?

## Options considered
1. **A client-side-only single-page-application framework** — the same category of tooling used for the Admin/Ops Dashboard (ADR-0008). Simple, but weaker for a public, discoverability-sensitive storefront that benefits from server rendering.
2. **A non-JavaScript web stack.** Rejected outright — it would break ADR-0005's shared-language rationale and introduce a language a solo developer would need to context-switch into for one surface only.
3. **Next.js**, a server-rendered/hybrid React framework in the same TypeScript ecosystem as the rest of the stack.

## Decision
Build the Customer Web App in Next.js.

## Rationale
Customer-facing pages — product listings, category pages, checkout — benefit from server rendering for load performance and search discoverability in a way the staff-only Admin/Ops Dashboard does not need, which is why the two web clients deliberately use different frameworks (see ADR-0008) rather than one default. A non-JavaScript stack was rejected on ADR-0005's grounds. Next.js keeps the Customer Platform's web half in the same TypeScript/React ecosystem as its mobile half (ADR-0006), enabling shared component patterns and shared API-contract types, and supporting Principle 4.6's requirement that the two customer clients never diverge in business behavior.

## Consequences
- Shared component/design patterns and shared API-contract types with the Customer Mobile App and backend.
- A rendering strategy suited to a public, performance- and discoverability-sensitive storefront.
- Server rendering introduces a running Node.js server as part of the Customer Web App's deployment shape, not just static assets — this must be accounted for in `05-deployment/infrastructure-and-release.md`.
- Checkout- and pricing-relevant logic stays in the backend's service layer (Principle 4.6) even though Next.js can execute server-side code — the framework choice does not create a second home for business logic.

## Future reconsideration conditions
Evidence that the storefront's actual traffic and SEO needs are better served by a different rendering strategy within the same framework family (for example, static generation for specific pages) — a refinement, not necessarily a framework change. A full framework change is not anticipated absent a fundamental shift in the Customer Web App's requirements.

## Related
- SRD section(s): Section 2 (Application Structure — Customer Web App), Customer Platform definition
- Related DDD entity/data area(s): n/a — a client technology choice, not a data-ownership decision
- Related architecture principle(s): 4.6, 4.11
- Related ADRs: 0005, 0006, 0008
