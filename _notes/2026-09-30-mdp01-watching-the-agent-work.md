---
title: "Watching the Agent Work"
date: 2026-09-30
author: mdp
tags: [evolution-conductor, agent-detail, observability, web-components]
status: draft
issue: "casehubio/engine#1181"
---

# Watching the Agent Work

The evolution conductor knows *what* to improve and *when*. It proposes improvements, gates them through approval stages, and tracks streams from introspection through to integration. What it didn't have was visibility into *how* the executing agent does the work — which skill it's running, what artifacts it's producing, whether the tests are passing.

That's the gap this branch fills. A new `agent-detail` component that surfaces the agent's methodology lifecycle inside the evolution workbench.

## The Composition Problem

The interesting design challenge wasn't the agent-detail component itself — it was how it fits into what already exists. The evolution workbench has five tabs (Streams, Inbox, Audit, Configuration, Health). Session management has its own workbench with terminal, git, health, and event tabs. The orchestration framework has execution monitors and timeline viewers.

The executing agent is a claudony session. The evolution conductor assigns it an improvement. The UI needs to show both the conductor's view (which improvement, what stage) and the session's view (terminal output, git status, events) in one place, without duplicating either.

The answer was composition, not construction. Agent-detail wraps `session-detail` for its terminal/git/health capabilities and adds methodology-specific tabs on top. One problem: session-detail listens for `session:selected` events at the document level. When embedded inside agent-detail, any other session list on the page would override the property-set session ID. The fix was a `composed` attribute that suppresses document-level listeners, letting the parent control identity directly.

## Methodology as Metadata

The evolution conductor already tracks eleven stages per improvement — introspect, research-scope, search, analyze, hypothesis-approval, implementation-plan, implement, submit-pr, pr-review, integrate, outcome-recording. Early designs proposed a separate soredium lifecycle running underneath these stages. But the evolution stages are already methodology-granular. Adding a parallel progression would create semantic overlap and a "which stage am I really in?" problem.

Instead, methodology is metadata on the current evolution stage. A `MethodologyMetadata` type carries the active skill name, the current artifact being produced, and timestamps. The agent reports structured methodology events — skill started, artifact created, test passed, forage captured — and the UI renders them in a chronological timeline using the same strategy pattern the orchestration workbench already uses.

## Safe Defaults

The control bar was the other design decision worth noting. The conductor has full terminal access to the executing agent. The UI should mirror that capability — but not by default. Pause and Resume are safe actions, shown without configuration. Force-transition, reassign, and terminate are destructive and require explicit opt-in via `controlConfig.allowedActions`. An operator who can see a button will eventually click it; the default should be the one that doesn't stop production.

Force-transition at a gated stage shows a warning and requires an override reason. The override creates an audit record that appears in the methodology timeline. The gate policy and the control bar operate on the same state — they had to be reconciled, not independent.

## What Shipped

Thirty files, 1400 lines across four batches: methodology types in evolution-config (avoiding a circular dependency with agent-detail), nine new EvolutionApi methods, a `methodologyEventsStrategy` for blocks-timeline, the agent-detail component with PushMixin and composed session-detail, evolution-streams row selection with workbench drill-down, schema registry, and three showcase pages in the example gallery.

The showcase pages are primed with sample data — ten methodology events tracing an agent through brainstorming, writing plans, TDD, tests passing, a commit, and a forage capture. You can click a stream row in the evolution workbench and watch the agent-detail panel appear below with methodology indicators and a control bar.

Also filed claudony#244 — pushing terminal UI from trellis back into claudony. The pages terminal component already handles buffering and WebSocket transport; session-detail should use it instead of polling into a `<pre>` tag. That's the next step for making this composition production-ready.
