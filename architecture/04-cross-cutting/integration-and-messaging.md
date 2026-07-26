# Integration & Messaging Architecture

**Stability:** Evolving — grows as new external integrations and internal event flows are added.
**Status:** Skeleton — pending authoring.

## Purpose
Defines how modules and runtime units talk to each other (sync interface/API conventions, async/event patterns) and how the system talks to external providers (payment gateway, SMS/OTP, maps). Sets conventions once so every integration doesn't reinvent them.

## Relationship to other documents
- **SRD:** External providers named in the SRD (payment gateway, Unifonic for SMS/OTP) are the concrete integrations this document sets conventions for.
- **system-context.md:** That document says *what* the system talks to; this one says *how*.
- **SDD:** Per-integration implementation detail (specific endpoints, payloads) lives in SDD documents and follows the conventions set here.
- **Implementation:** API client code and event handlers should follow the retry/timeout/idempotency conventions defined here.

## Sections to be authored
1. Synchronous API conventions (versioning, error format, idempotency)
2. Asynchronous/event conventions (if/when introduced — naming, delivery guarantees)
3. External integration patterns (retry, timeout, circuit-breaking, webhook verification)
4. Integration inventory (links out to system-context.md entries)
