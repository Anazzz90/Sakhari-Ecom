# 0005. TypeScript-centered stack

**Decision ID:** ADR-0005
**Status:** Accepted
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
The finalized SRD application structure places the backend on a Node.js-based framework and all four client applications (Customer Mobile, Customer Web, Worker App, Admin/Ops Dashboard) on JavaScript-ecosystem frameworks. One developer, assisted by AI coding assistants, is responsible for all five codebases. `00-Architecture-Principles.md` Section 3 names "boring technology" as a competitive advantage at this scale and states that every additional language is maintenance debt paid by one person.

## Problem
What language strategy across the backend and four clients minimizes context-switching cost for a solo developer and maximizes the likelihood that AI-generated code is correct across the whole stack?

## Options considered
1. **A polyglot stack** — a statically-typed backend language distinct from the clients' JavaScript/TypeScript ecosystem, with native mobile development (Swift/Kotlin) for the two mobile apps. Offers per-platform idiomatic tooling at the cost of five separate language contexts for one developer to maintain.
2. **Plain JavaScript across the stack, without static typing.** Minimizes language count but forfeits compile-time contract checking between backend and clients.
3. **TypeScript across the backend and all four clients**, sharing one type system and tooling ecosystem.

## Decision
Adopt TypeScript as the primary language across the backend and all four client applications, rather than a polyglot stack or untyped JavaScript.

## Rationale
A polyglot stack was rejected as directly contrary to Constitution Section 3's "boring technology"/one-runtime conviction — five separate language contexts is exactly the kind of maintenance debt the principle warns against for a one-person team. Plain JavaScript was rejected because static types catch a class of contract mismatch (an API response shape drifting from what a client expects) before it reaches Principle 4.1's correctness-critical paths, and a shared type system reduces the surface where an AI assistant can generate a change on one side of a contract that silently doesn't match the other.

## Consequences
- One language and dependency ecosystem across backend and all clients; a single hiring/tooling/AI-prompting context.
- Potential to share request/response contract types directly between the backend and its clients, supporting Principle 4.11's versioned-API discipline (see ADR-0017).
- TypeScript's type system must be used rigorously — liberal use of untyped escape hatches forfeits the correctness benefit that justified the choice.
- All new backend or client code is written in TypeScript, not plain JavaScript.

## Future reconsideration conditions
A specific, evidenced performance ceiling in the Node.js/TypeScript backend that a different backend runtime would solve, and that no in-ecosystem fix (worker threads, native addons, horizontal scaling) resolves. Not anticipated at launch scale.

## Related
- SRD section(s): Section 2 (Application Structure — all four apps' finalized frameworks)
- Related DDD entity/data area(s): n/a — a language choice, not a data-ownership decision
- Related architecture principle(s): Constitution Section 3 (boring technology); 4.11
- Related ADRs: 0006, 0007, 0008
