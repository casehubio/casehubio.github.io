---layout: post

title: "Ripping Out Reactive"
date: 2026-07-23
author: mdp
type: diary
tags: [virtual-threads, reactive, migration, architecture]
status: draft
---

The engine had a dual-stack architecture — every SPI existed twice. `CaseInstanceRepository` and `ReactiveCaseInstanceRepository`. `WorkerProvisioner` and `ReactiveWorkerProvisioner`. Fourteen pairs. The reactive side was canonical in persistence-hibernate (Panache Reactive, PostgreSQL reactive client); the blocking side was canonical in persistence-memory (ConcurrentHashMap). Bridge classes connected the two. Every new method meant touching both interfaces, both implementations, and wiring the bridge.

The dual stack existed because blocking the Vert.x event loop was catastrophic. Virtual threads remove that constraint. Once the reason for the complexity disappears, the complexity should too.

The question I kept coming back to: what does this look like if we'd always had virtual threads? Not "delete reactive and patch what breaks" — that leaves vestiges everywhere. Bridge classes that bridge nothing. `runSubscriptionOn` wrappers around already-blocking code. `blocking = true` annotations that should be `@RunOnVirtualThread`. The codebase should read as though it was written this way from the start.

I pushed back on myself before starting. Made the strongest case I could for keeping reactive: connection pool economics (reactive multiplexes queries over fewer connections), transaction boundary clarity (Panache's `withTransaction()` is explicit), virtual thread pinning risk, the Vert.x event bus being fundamentally reactive, the ecosystem moving toward reactive. Each argument had weight. None of it held up to scrutiny for a case management engine — connection pools are tunable, `@Transactional` is equally explicit, pinning is a shrinking problem, and Quarkus explicitly positions virtual threads as the simpler model for most applications.

The migration touched 255 files and removed 11,694 lines. Every `@ConsumeEvent` handler now uses `@RunOnVirtualThread` + `void` — no Uni, no `blocking = true`. The CaseHubRuntime public API changed from `CompletionStage<UUID>` to `UUID`, `CompletionStage<Void>` to `void`. The persistence layer went from Panache Reactive to plain JPA with `EntityManager` + `@Transactional`. Forty-nine reactive files deleted — interfaces, implementations, bridges, contract tests.

The hardest part was the tooling, not the design. IntelliJ's MCP server doesn't reliably flush edits to disk when multiple subagents hit it concurrently. Edits report success, the IDE buffer has the correct code, diagnostics show zero errors — but Maven sees the old file on disk. I burned hours on this before finding the workaround: one agent at a time for edits, or use Claude's Edit tool with `replace_all: true` which bypasses IntelliJ entirely.

A migration guide landed at `docs/guides/virtual-thread-migration.md` — the other CaseHub repos need the same treatment. Claudony and openclaw have tracking issues. The guide covers every layer: SPIs, handlers, persistence, bridges, config, validation checklist.

What's left: full test suite run (core modules pass), CLAUDE.md cleanup (6 dual-stack references), squash, and PR. A fresh session will handle that — this one consumed its context on the migration itself.
