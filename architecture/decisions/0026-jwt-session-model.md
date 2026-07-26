# 0026. JWT access tokens with rotating refresh tokens

**Status:** Accepted

## Context

The platform has four clients sharing one backend. Auth must support customer, worker, and admin flows while keeping business authorization centralized.

## Problem

The architecture documented JWT usage but did not have a dedicated ADR for the session model.

## Options Considered

1. Server-side sessions only.
2. Long-lived JWTs only.
3. Short-lived JWT access tokens plus rotating refresh tokens.

## Decision

Use short-lived JWT access tokens with rotating refresh tokens. Authorization remains server-enforced through RBAC and permission evaluation.

## Rationale

JWTs allow efficient authenticated requests across multiple clients without a database lookup on every request. Rotating refresh tokens reduce the risk of long-lived bearer credentials and support revocation/reuse detection.

## Consequences

- Token lifetimes are implementation parameters, not settled here.
- Refresh token storage and reuse detection belong to Auth.
- Clients must treat tokens as transport credentials only, never as authority to bypass backend authorization.

## Future Reconsideration Conditions

Reconsider if immediate revocation, device management, or compliance requirements make a server-session model preferable.

## Related Documents

- Related SRD section(s): Auth
- Related DDD entity/data area(s): User/Auth, Session, Refresh Token
- Related architecture principle(s): Principle 4.6, Principle 4.11

