# 0023. Unifonic SMS/OTP provider

**Status:** Accepted

## Context

The SRD explicitly selects Unifonic for SMS/OTP because Twilio cannot reliably support Saudi domestic sender ID requirements for this use case.

## Problem

Architecture docs previously treated the SMS/OTP provider as undecided, which could cause implementation prompts to invent Twilio, Firebase, or another generic provider.

## Options Considered

1. Keep provider generic.
2. Use Twilio or Firebase phone auth.
3. Record Unifonic as the accepted launch provider.

## Decision

Use Unifonic as the launch SMS/OTP provider for authentication OTPs, rider delivery-confirmation OTPs, and SMS notification delivery where required.

## Rationale

Unifonic fits the Saudi regulatory and operational environment, supports local sender registration, and is significantly more practical for domestic Saudi SMS traffic.

## Consequences

- Notification owns Unifonic SMS delivery.
- Auth uses Notification for OTP dispatch rather than calling Unifonic directly.
- Provider callbacks/status responses must be validated before being trusted.
- Provider-specific implementation details belong in the Notification/Auth SDDs.

## Future Reconsideration Conditions

Reconsider if Unifonic delivery reliability, pricing, compliance support, or API quality becomes unacceptable.

## Related Documents

- Related SRD section(s): Auth, SMS/OTP, Notifications
- Related DDD entity/data area(s): User/Auth data, Notification
- Related architecture principle(s): Principle 4.6, Principle 4.8, Principle 4.10

