# 0006. React Native for Customer and Worker mobile apps

**Decision ID:** ADR-0006
**Status:** Accepted
**Date:** 2026-07-24
**Deciders:** Principal Architect (solo-project decision authority)

## Context
The SRD finalizes the Customer Mobile App and the Worker App (combining Picker and Rider under one app with mode selection) as iOS and Android React Native applications, superseding an earlier Flutter prototype. ADR-0005 establishes TypeScript as the stack's shared language.

## Problem
Should the two mobile surfaces (Customer, Worker) be built natively per platform, or with a shared cross-platform framework, given one developer must build and maintain both across iOS and Android?

## Options considered
1. **Native development (Swift/Kotlin) per platform per app** — four codebases (Customer iOS, Customer Android, Worker iOS, Worker Android). Best access to platform capability, at four times the codebases for a solo developer.
2. **A different cross-platform framework** (for example, Flutter) — the project's own history includes an earlier Flutter prototype, superseded in favor of the current direction.
3. **React Native, shared between the Customer Mobile App and the Worker App**, within the TypeScript-centered stack established in ADR-0005.

## Decision
Build the Customer Mobile App and the Worker App in React Native.

## Rationale
Native per-platform development was rejected as unsustainable for one developer maintaining two mobile applications across two platforms (Constitution Section 3, Design Goal: maintainability by a single developer). React Native was chosen over remaining on the earlier Flutter direction specifically because it keeps the mobile surfaces in the same TypeScript/JavaScript ecosystem as the backend and web clients (ADR-0005), maximizing the odds that an AI coding assistant generating code across the stack stays correct and consistent, and matching the SRD's own finalized direction.

## Consequences
- Shared tooling, shared language, and real code-sharing opportunity for cross-cutting mobile concerns (auth flow, networking, design-system primitives) between the Customer Mobile App and the Worker App.
- The two apps remain fully separate applications per Principle 4.6/4.7's client-boundary discipline and the Worker App's own mode rules (no customer functionality present in the Worker App) — any shared mobile code lives in a clearly-scoped shared package, never by one app importing from the other's application code.
- A future capability requiring a native platform feature React Native cannot bridge would need an explicit bridging solution, evaluated case by case.

## Future reconsideration conditions
A proven, evidenced React Native limitation — a committed product requirement blocked by a native capability React Native genuinely cannot reach even via bridging — would prompt reconsideration for the specific affected surface. Not anticipated at launch scale.

## Related
- SRD section(s): Section 2 (Application Structure — Customer Mobile App, Worker App), Section 2.2 (Worker App Mode Rules, AMR-001/AMR-002), Section 0 (Document History — Flutter prototype superseded)
- Related DDD entity/data area(s): n/a — a client technology choice, not a data-ownership decision
- Related architecture principle(s): 4.6, 4.7; Constitution Section 3
- Related ADRs: 0005, 0007, 0008
