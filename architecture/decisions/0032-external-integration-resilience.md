# 0032. External integration resilience

**Status:** Accepted

## Context

The system depends on Moyasar, Unifonic, push notification delivery, Google Maps/geocoding, and POS/card-on-delivery reconciliation. Prior documentation named these integrations but left timeout, retry, and circuit-breaker behavior open.

## Problem

Provider failures must not create duplicate payments, duplicate OTPs, lost delivery assignments, or ambiguous order/payment state.

## Decision

Every external integration is wrapped by its owning module and follows provider-specific resilience rules:

| Integration | Owner | Timeout | Retry | Circuit breaker / fallback |
|---|---|---:|---|---|
| Moyasar payment API | Payment | 5 s connect/read | Retry only idempotent create/status operations, max 3 attempts with exponential backoff; never retry non-idempotent calls without provider idempotency key | Open circuit after 5 consecutive technical failures or 50% technical failure rate over 2 minutes; online payment temporarily unavailable while COD/Card-on-Delivery remain available |
| Moyasar webhooks | Payment | inbound | Verify signature and idempotency key/payment reference; process at least once | Duplicate webhooks are ignored after first successful state transition |
| Unifonic OTP/SMS | Notification/Auth | 5 s | Max 2 send attempts for same OTP request; no new OTP generated during retry | If unavailable, login/delivery OTP flow fails loudly with retry-later state |
| Push notification service | Notification | 5 s | Max 3 attempts with backoff through outbox/job retry | Business state never depends on push success; Worker App must poll assignment endpoint as fallback |
| Google Maps/geocoding | Store/User | 3 s autocomplete, 5 s server validation | Max 2 attempts | If unavailable, new address validation fails gracefully; existing validated addresses continue |
| POS/card-on-delivery settlement import | Payment | 10 s | Scheduled retry with backoff | Manual reconciliation remains fallback; unresolved settlement discrepancies stay open until reviewed |

All outbound calls must use correlation IDs, structured logs, idempotency keys where supported, and provider response snapshots without storing raw card data or secrets.

## Rationale

Different providers have different business consequences. Payment and OTP failures affect core flows and must fail visibly; push and analytics can degrade; POS settlement can fall back to manual reconciliation but must not silently pass.

## Consequences

- Provider wrappers are mandatory; modules cannot call provider SDKs directly from scattered code.
- Payment and refund idempotency keys become required.
- Polling fallback is required for Worker App task assignment.
- Operational dashboards must distinguish provider technical failures from customer/business declines.

## Future Reconsideration Conditions

Revisit if provider SLAs, observed latency, or incident history justify different thresholds or redundant providers.

## Related Documents

- Related architecture principle(s): Principle 4.1, Principle 4.8, Principle 4.10
- Related ADRs: 0022, 0023, 0027, 0029

