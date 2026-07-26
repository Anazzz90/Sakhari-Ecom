# 0025. NestJS backend framework

**Status:** Accepted

## Context

The SRD and ADS settle on an in-house TypeScript backend structured as a modular monolith. The backend needs strong module structure, dependency injection, thin controllers, and service-layer business logic.

## Problem

NestJS was documented in the technology decisions but had no dedicated ADR, leaving a major framework decision less visible than comparable frontend/database choices.

## Options Considered

1. Minimal Node framework such as Express/Fastify directly.
2. Non-TypeScript backend framework.
3. NestJS.

## Decision

Use NestJS for the backend modular monolith.

## Rationale

NestJS provides first-class module structure, dependency injection, TypeScript support, and conventions that match the architecture's module ownership and thin-controller rules. It reduces the amount of structure a solo developer must invent and enforce manually.

## Consequences

- Backend modules should map cleanly to NestJS modules.
- Controllers remain thin and services own business logic.
- NestJS does not by itself enforce architecture boundaries; linting/review still must prevent cross-module repository access.

## Future Reconsideration Conditions

Reconsider only if NestJS materially blocks a committed requirement or imposes unacceptable operational/development cost.

## Related Documents

- Related SRD section(s): Backend stack
- Related DDD entity/data area(s): All backend modules
- Related architecture principle(s): Principle 4.4, Principle 4.6, Principle 4.7

