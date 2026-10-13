---
layout: post
title: "Closing the Safety Net on yaml-core"
date: 2026-09-28
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [yaml-core, testing, code-review]
series: issue-475-yaml-core-full-parity
---

# Closing the Safety Net on yaml-core

The yaml-core parity work has been open for a while now — 27 commits bringing the TypeScript runtime up to feature parity with the Java implementation. Step catalog, decorator chains, orchestration primitives, module inheritance, the works. But none of it had tests.

That's the kind of gap that bites you at 2am. Every commit on the branch was adding new surface area — blocking state machines, event routers, spawned tasks, validation pipelines — and none of it had a regression safety net. If a refactor broke the waiter cleanup in `awaitStateWithTimeout`, nothing would catch it until something timed out in production.

I wanted full coverage before closing the branch, so I filed it as a separate issue and worked through the entire source tree. The interesting part wasn't the test writing itself — it was what the review afterward turned up. Two of the findings were the kind of thing that would have been invisible without the tests forcing me to think about the code paths carefully.

The first was a waiter leak in `DefaultBlockingOrcStateMachine`. When `awaitStateWithTimeout` fires its timeout, it resolves the promise with `false` — but it never removes the waiter from the array. On the next state transition, the loop walks all waiters, finds the stale one, and calls `resolve()` on an already-settled promise. Harmless in isolation, but the waiter array grows without bound in any long-running scope. The fix was three lines: capture the waiter reference, splice it out when the timer fires.

The second was `DefaultScenarioScope.childScope()` using a cast to write a `private readonly` field: `(child as { _parent: ... })._parent = this`. It works, but it's fragile — a rename of `_parent` silently breaks the parent chain lookup. The fix was to accept the parent as a constructor parameter, which is what it should have been from the start.

Neither of these would have shown up in a feature test. They're the kind of structural correctness issues that only surface when you're writing focused unit tests and asking "what happens at the boundary?" The parity branch is now at 590 tests across 54 files, and the two bugs are fixed. The branch is ready to close.
