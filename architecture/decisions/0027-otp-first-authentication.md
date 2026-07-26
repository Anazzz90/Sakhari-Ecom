# 0027. OTP-first authentication

**Status:** Accepted

## Context

The SRD establishes phone-based OTP login for customers and workers, with admin access handled through the backend's centralized auth model. OTP is also reused for rider delivery confirmation.

## Problem

OTP was documented as a technology choice but not captured as a dedicated architecture decision.

## Options Considered

1. Email/password for all actors.
2. Social login.
3. Phone OTP as the primary customer/worker authentication mechanism.

## Decision

Use OTP-first authentication for customers and workers, delivered through Unifonic via the Notification/Auth boundary. Rider delivery confirmation also uses OTP/proof validation through the same conceptual mechanism.

## Rationale

Phone OTP fits the Saudi market, avoids password-storage/reset burden for customer and worker populations, and aligns with delivery-confirmation workflows.

## Consequences

- Auth owns OTP verification state and session creation.
- Notification owns SMS delivery through Unifonic.
- OTP rate limiting and abuse protection are required.
- Password-based customer/worker auth must not be introduced silently.

## Future Reconsideration Conditions

Reconsider if OTP delivery reliability, SIM-swap risk, cost, or regulatory requirements justify passkeys, app authenticators, or additional factors.

## Related Documents

- Related SRD section(s): Auth, Notifications, Delivery proof
- Related DDD entity/data area(s): User/Auth, Notification, Delivery Assignment
- Related architecture principle(s): Principle 4.6, Principle 4.10

