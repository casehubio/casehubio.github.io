---
layout: post
title: "One Dispatch Mechanism"
date: 2026-09-27
entry_type: note
subtype: diary
projects: [casehubio/platform]
tags: [yaml-core, pattern-matching, state-machines, hardening]
---

# One Dispatch Mechanism

The YAML orchestration language had two separate dispatch systems. `match/cases` used `MatchPattern` — a sealed interface with `ValuePattern`, `StructuralPattern`, and `DefaultPattern` — for step-level branching. State machine transitions used direct enum identity comparison inside `EventRouter`. Same concept, different implementations.

The regularity principle says: learn one mechanism, apply it everywhere. If pattern matching is how you dispatch in step-level branching, it should be how you dispatch in state machine transitions too. The shapes are identical — value equality, structural subset matching, catch-all defaults. The only thing missing was `AnyOfPattern` for the `from: [PENDING, ACTIVE]` case, where a transition applies from multiple source states.

Adding `AnyOfPattern` to the sealed interface was trivial — four lines of `matches()` logic wrapping a list iteration. The interesting part was threading it through `EventRouter`. The old design keyed mappings by event string in a `Map<String, List<EventMapping>>` for O(1) lookup. The new design uses a flat list with pattern matching on both the event (`on`) and the source state (`from`). For state machines with a handful of transitions this is fine; the EventRouter matches `current.name()` against `fromPattern` so the YAML layer can work with strings throughout.

The Builder stays backward-compatible — the existing `on(String event, S from, S to)` methods create `ValuePattern` instances internally. The new overloads take `MatchPattern` directly, and `registerTransitionsForPattern()` expands patterns into the state machine's internal transition map so the CAS validation still works. `AnyOfPattern` resolves each value to an enum constant; `DefaultPattern` registers from every non-terminal state.

The session also landed an ADR formalising the three-layer evaluation model — imperative (`if`, `match`), reactive (`when`), and a future rules engine with its own distinct keywords. The `when` → `if` rename from the previous session established this boundary; the ADR records it as a standing constraint so future language extensions can't accidentally reuse keywords across layers.

Four hardening issues closed alongside: `StepWalker` now has a depth limit (32 levels, with nesting path in error messages), `DefaultProcessExecutor` enforces `maxOutputBytes` with a bounded read loop instead of `readAllBytes()`, five duplicate `parseTimeout` methods collapsed to `DurationParser.parseOrNull()`, and `McpStepCatalogWiring` dropped its reflection-based dispatch in favour of a `ToolDispatcher` interface in `platform-api`.

The queue still has four items — runtime evaluators for structural step types and decorators (#465-#466), then `try/catch/finally` and CSP `select` (#463-#464). Those are M-to-L scale and need their own design sessions. The groundwork laid today — MatchPattern shapes flowing through EventRouter, the depth-limited StepWalker, the evaluation model vocabulary contract — gives them a solid foundation to build on.
