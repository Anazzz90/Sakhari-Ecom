# 0008. React web Admin Dashboard

**Decision ID:** ADR-0008
**Status:** Accepted
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
The SRD finalizes the Admin/Ops Dashboard as a staff-only, browser-based single-page application built with React and Vite, never publicly distributed, serving Admins, Ops, and Support for catalog, inventory, order, worker, store, and dispatch management and reporting/analytics.

## Problem
What web technology should serve an internal, authenticated-only tool where the priorities (internal build/iteration speed, no public discoverability requirement) differ materially from the public Customer Web App's?

## Options considered
1. **The same server-rendered framework as the Customer Web App (Next.js)**, for stack consistency. Would carry server-rendering overhead an internal, always-authenticated tool doesn't need.
2. **A no-code/low-code admin panel generator.** Fast to stand up, but tends to place business logic and access rules in the tool itself rather than the backend service layer.
3. **A React single-page application built with Vite**, distinct from the Customer Web App's framework, staying in the same TypeScript/React ecosystem (ADR-0005).

## Decision
Build the Admin/Ops Dashboard as a React single-page application using Vite, distinct from the Next.js Customer Web App.

## Rationale
An internal, authenticated-only tool has no server-rendering or SEO requirement, so Next.js's server-rendering overhead buys nothing here — a lighter single-page-app build optimizes instead for internal development speed, which matters more for a tool that changes frequently as operations needs evolve. A no-code/low-code generator was rejected because it would place business logic and permission checks outside the backend service layer, directly violating Principle 4.6. Staying on React keeps the dashboard in the same component and TypeScript ecosystem as the rest of the stack (ADR-0005) even though it remains, per Principle 4.6/4.7, a fully separate client application from the Customer Web App.

## Consequences
- Fast local development and build times appropriate to a frequently-iterated internal tool.
- Component-pattern reuse potential with other React-based surfaces where genuinely useful, without forcing shared deployment infrastructure with the Customer Web App.
- No automatic code sharing with the Next.js-based Customer Web App beyond shared types and API-client code.
- The dashboard, like every client, contains no independent business logic (Principle 4.6) — it is a thin operational surface over the same backend API every other client uses, and every action it exposes is subject to the same RBAC checks (ADR-0012) enforced by the backend.

## Future reconsideration conditions
The dashboard's own complexity growing to a point where server-rendering or a more structured framework materially improves internal velocity — evaluated only against evidenced pain (slow builds, real developer friction), not preference.

## Related
- SRD section(s): Section 2 (Application Structure — Admin/Ops Dashboard)
- Related DDD entity/data area(s): n/a — a client technology choice, not a data-ownership decision
- Related architecture principle(s): 4.6, 4.7
- Related ADRs: 0005, 0007, 0012
