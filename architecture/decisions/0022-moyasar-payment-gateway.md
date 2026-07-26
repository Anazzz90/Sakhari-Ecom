# 0022. Moyasar payment gateway

**Status:** Accepted

## Context

The SRD explicitly selects Moyasar for online card, mada, and Apple Pay payments after comparing Saudi payment gateway options. Stripe was rejected because it is not usable for a Saudi-domiciled merchant account in this launch context.

## Problem

Architecture docs previously referred generically to "Payment Gateway," which allowed future assistants to treat the provider as undecided even though the SRD settled it.

## Options Considered

1. Keep the provider generic until implementation.
2. Use Stripe or a global gateway.
3. Record Moyasar as the accepted launch provider.

## Decision

Use Moyasar as the launch online payment gateway for mada, cards, and Apple Pay. Cash on Delivery and Card on Delivery remain platform-managed payment paths with reconciliation.

## Rationale

Moyasar is Saudi-focused, supports mada, has no monthly fee, and fits the Saudi-only launch better than broader multi-country providers whose extra coverage is not needed yet.

## Consequences

- Payment module owns the Moyasar integration.
- No client calls Moyasar directly.
- Gateway responses/webhooks must be verified before payment state changes.
- A future provider change is isolated behind Payment's interface.

## Future Reconsideration Conditions

Reconsider if Sakhari expands outside Saudi Arabia, Moyasar pricing/reliability becomes unacceptable, or a required payment method is unavailable through Moyasar.

## Related Documents

- Related SRD section(s): 3.2 Payment Gateway, Payments
- Related DDD entity/data area(s): Payment, Payment History, Refund
- Related architecture principle(s): Principle 4.1, Principle 4.6, Principle 4.10

