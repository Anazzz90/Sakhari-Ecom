# 0028. CDN for public cacheable content

**Status:** Accepted

## Context

The Customer Web App, product imagery, and other public cacheable assets need fast delivery without pushing repeat traffic through the backend or object-storage origin.

## Problem

CDN usage was documented as a technology decision but not captured in the ADR index.

## Options Considered

1. Serve all static assets directly from backend/origin storage.
2. Use a CDN for public cacheable content.
3. Introduce a third-party CDN distinct from the cloud provider immediately.

## Decision

Use a CDN for public, cacheable content such as product images and Customer Web App static assets. Prefer the cloud provider's native CDN initially unless a future ADR justifies another provider.

## Rationale

CDN delivery improves customer-perceived performance and reduces repeated origin load while keeping business logic and authenticated dynamic requests in the backend.

## Consequences

- Product/media assets must be addressable through object storage/CDN references.
- CDN must never become a source of business truth.
- Cache invalidation rules belong to deployment/frontend SDDs.

## Future Reconsideration Conditions

Reconsider CDN provider or strategy if regional performance, cost, or cache-control needs justify it.

## Related Documents

- Related SRD section(s): Customer Web App, Product media
- Related DDD entity/data area(s): Product Image, file metadata
- Related architecture principle(s): Principle 4.1, Principle 4.3

