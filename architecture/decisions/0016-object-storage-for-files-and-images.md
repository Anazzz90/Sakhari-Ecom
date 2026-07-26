# 0016. Object storage for files and images

**Decision ID:** ADR-0016
**Status:** Accepted
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
The platform needs to store product imagery and other file/media assets. `01-Architecture-Design-Specification.md` already names object storage as the platform's media/asset layer, and Principle 4.2 scopes PostgreSQL to durable business state, not binary payloads.

## Problem
Where should product images and other file/media assets live, given that PostgreSQL is reserved for durable business state and storing large binary payloads directly in the relational database would work against that boundary and against operational simplicity?

## Options considered
1. **Store binary files as BLOBs directly in PostgreSQL.** Keeps everything in one datastore, but bloats the system-of-record database, slows backups, and works against Principle 4.2's intent that PostgreSQL hold business facts, not media payloads.
2. **Store files on local/instance disk on the backend server.** Cheapest to wire up initially, but does not survive instance replacement and is incompatible with a managed-infrastructure, disaster-recoverable approach.
3. **Store files in managed object storage**, with PostgreSQL rows holding only a reference (key/URL) to each object.

## Decision
Product images and other file/media assets are stored in managed object storage; PostgreSQL rows reference the object's key or URL, never the binary content itself.

## Rationale
Option 1 was rejected as working directly against Principle 4.2 and against the operational-simplicity design goal — a bloated system-of-record database is slower to back up and restore, exactly the kind of avoidable operational burden a solo developer shouldn't carry. Option 2 was rejected outright: it does not survive instance replacement and is incompatible with the managed-infrastructure approach already adopted for the database (ADR-0003). Managed object storage is purpose-built for this, keeps the database lean, and cleanly separates media durability/availability concerns from business-data concerns.

## Consequences
- Database backups and restores stay fast and focused on actual business state.
- Media can be served or cached independently (for example, via a CDN in front of object storage) without touching the database tier.
- The system now has two places data lives — PostgreSQL for facts, object storage for media — which must be kept consistent; an entity referencing a deleted object, or an orphaned object with no referencing entity, is a class of bug to guard against.
- Any module that stores a reference to an object is responsible for that reference's lifecycle (created, and cleaned up on deletion) as part of its own data ownership under Principle 4.5.

## Future reconsideration conditions
None anticipated. This is a standard, low-risk pattern consistent with the managed-infrastructure approach already adopted for the database (ADR-0003), unlikely to need revisiting independent of a broader infrastructure change.

## Related
- SRD section(s): Catalog/product-media requirements (inventory and catalog scope)
- Related DDD entity/data area(s): Product/Catalog media references, and any Worker/Admin-uploaded document areas
- Related architecture principle(s): 4.2, 4.5
- Related ADRs: 0003
