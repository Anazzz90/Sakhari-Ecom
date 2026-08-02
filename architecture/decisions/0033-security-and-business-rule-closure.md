# 0033. Security and business-rule closure

**Status:** Accepted

## Context

The ARR identified several implementation-blocking ambiguities: RBAC permission inventory, token lifetimes, session storage, refund eligibility authority, Settings readers, Worker App push, geocoding mediation, and POS/card-on-delivery reconciliation.

## Problem

These decisions affect security, money movement, store routing, operational assignment, and configuration ownership. Leaving them to implementation would create inconsistent behavior.

## Decision

Close the gaps as follows:

1. **RBAC permission catalog:** each module owns named permissions for its public operations. Baseline permissions are:
   - Auth: `auth:request_otp`, `auth:verify_otp`, `auth:refresh`, `auth:revoke_session`
   - User: `user:read_self`, `user:update_self`, `worker:read`, `worker:manage`, `shift:manage`
   - Store: `store:read`, `store:manage`, `service_zone:manage`
   - Catalog: `catalog:read`, `catalog:manage`
   - Inventory: `inventory:read`, `inventory:adjust`, `inventory:reconcile`
   - Cart: `cart:read_self`, `cart:mutate_self`
   - Order: `order:create`, `order:read_self`, `order:read_store`, `order:manage`, `order:cancel`
   - Payment: `payment:read`, `payment:collect_cod`, `payment:reconcile`, `refund:request`, `refund:approve`, `refund:issue`
   - Promotion: `promotion:read`, `promotion:manage`
   - Delivery: `picking:work`, `delivery:work`, `delivery:assign`, `rider_location:write`
   - Notification: `notification:read_self`, `notification:manage_templates`
   - Support: `support:read`, `support:manage`, `support:comment`, `support:escalate`
   - Analytics: `analytics:read`
   - Audit: `audit:read`
   - Settings: `settings:read`, `settings:manage`
2. **Token lifetimes and storage:** access tokens last 15 minutes; refresh tokens last 7 days idle and 30 days absolute. Refresh token families, sessions, revocation, and reuse-detection state are stored in PostgreSQL. Redis may cache validation metadata but is not authoritative.
3. **Refund eligibility:** Order owns automatic eligibility rules from order state, cancellation reason, delivery failure, and substitution outcome. Support may initiate a refund request with a reason. Payment validates payment/refund constraints and executes only after eligibility is approved by Order or an authorized Support approval flow.
4. **Settings readers:** Auth, Notification, Order, Payment, Delivery, Inventory, Promotion, Store, and Analytics may read Settings through `SettingsService`; only Settings writes settings.
5. **Worker App push:** Worker App push notifications are confirmed for task assignment and operational alerts. Polling remains the fallback.
6. **Geocoding mediation:** Customer clients may use Google Maps client SDK for autocomplete only. Backend performs authoritative geocoding validation, coordinate persistence, service-zone resolution, and order-time address snapshot.
7. **POS/card-on-delivery:** Payment owns POS settlement reconciliation. MVP supports manual import/entry plus discrepancy workflow; provider API integration remains optional enhancement after terminal provider confirmation, not a blocker to record card-on-delivery correctly.

## Rationale

These closures choose conservative defaults: PostgreSQL for authoritative security state, backend authority for routing, explicit permission names, and Payment ownership for financial reconciliation.

## Consequences

- SDDs must map every endpoint to one or more permissions.
- Auth schema must include sessions/refresh-token families.
- Support and Order both participate in refund workflows, but Payment remains refund record owner.
- Settings becomes a declared dependency for several modules.
- Worker App implementation must include push plus polling.
- Client-side geocoding cannot be trusted as authoritative.

## Future Reconsideration Conditions

Revisit with a superseding ADR if passkeys replace OTP, a dedicated Workforce module returns, POS provider API becomes mandatory, or Settings grows into a policy/rules engine.

## Related Documents

- Related architecture principle(s): Principle 4.1, Principle 4.5, Principle 4.6, Principle 4.10
- Related ADRs: 0012, 0020, 0022, 0026, 0027, 0032

