# 0001. Record architecture decisions as ADRs

**Status:** Accepted
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
The SRD's own version history (Section 0) shows this project has already reversed several significant technical choices — Supabase → in-house NestJS+RDS auth, Twilio → Unifonic for SMS/OTP, single-store → multi-store — each time by superseding the whole document. That works for *requirements* because the SRD is product-scoped and versioned as a whole. Architecture decisions are narrower and more numerous than requirements changes, and re-issuing a whole document per decision would bury the reasoning instead of preserving it. A solo developer working with AI coding assistants across many sessions also needs a durable, greppable record of *why* a structural choice was made, since that context does not live in code and will not reliably survive in conversation history.

## Options considered
1. **No formal decision record** — reasoning stays in commit messages and chat history only.
2. **Single running "decisions log" document** — one file, chronological, appended to.
3. **One ADR file per decision**, immutable once accepted, using the Michael Nygard-style template, stored under `architecture/decisions/`.

## Decision
Use one ADR file per decision (option 3), following the template in `template.md`, indexed in this folder's `README.md`.

## Rationale
A single running log (option 2) degrades into an unstructured wall of text and is hard for an AI assistant to selectively retrieve — "what does the log say about auth?" requires reading the whole file. Per-decision files are individually addressable, linkable from other architecture documents (e.g., `data-ownership-map.md` can cite the ADR that justified an ownership boundary), and immutable, which matches how the SRD's own superseding pattern already treats major decisions — just at finer granularity.

## Consequences
- Every future structural decision (new service, new datastore, new external dependency, reversal of a prior ADR) must be recorded as a new ADR, not folded into charter/principles/cross-cutting prose.
- The `decisions/README.md` index must be updated whenever a new ADR is added or a status changes.
- Because ADRs are immutable, correcting a past decision always means writing a new ADR and marking the old one `Superseded by`, never editing the old file's Decision section.

## Related
- SRD section(s): Section 0 (Document History & Why This Version Exists)
- Related DDD entity/data area(s): n/a — process decision, not a data-design decision
- Related ADRs: none (first record)
