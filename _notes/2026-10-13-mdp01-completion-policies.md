---
layout: post
title: "Three patterns, one parallel block"
date: 2026-10-13
entry_type: note
subtype: diary
projects: [casehubio/platform]
tags: [yaml-core, orchestration, discriminator, parallel, concurrency]
series: issue-392-discriminator-pattern
---

The runtime orchestration spec deferred the discriminator pattern back in September — advance on first completion from a set of parallel steps, but let the remaining steps keep running. Distinct from race, which cancels the losers. The issue framed it as either a new `discriminator` keyword or a `race` option. Both were wrong.

Race, discriminator, and today's wait-all parallel differ in exactly one axis: what happens when N steps complete inside a concurrent group. Race cancels remaining. Discriminator continues collecting. Parallel waits for all. That's a completion policy on the same execution substrate, not three independent language constructs.

The design review caught this. The original brainstorming proposed both a new `OrcDiscriminator` primitive in yaml-core AND a `discriminator:` YAML keyword — but then chose to compose from OrcLatch + SpawnedTask, which contradicted the "new primitive" decision. The reviewer called it: barrier and quorum are step types composed from primitives, not primitives themselves. The discriminator follows the same pattern.

The unified model extends `parallel:` with two optional fields:

```yaml
parallel:
  advance-after: 1          # default: all
  cancel-remaining: false    # default: false
  steps:
    - step: primary-db
      action: query-primary
    - step: fallback-api
      action: query-fallback
```

`advance-after: 1` with `cancel-remaining: false` gives discriminator semantics. With `cancel-remaining: true`, race. With `advance-after: 2`, quorum-continue. And `advance-after: all` (or just omitting it) preserves today's wait-all behaviour. One block, two options, four patterns.

The implementation uses a two-latch pattern — an advancement latch (count = N, only successful completions count down) and a completion latch (count = M, any termination counts down). `ParallelCompletionTracker` mirrors `QuorumTracker`'s structure: atomic counters for success and failure, unreachable-threshold detection that force-drains the advancement latch when too many steps fail.

The interesting detail was scope ownership for race. The discriminator spawns branches on the parent scope so they survive the parallel step returning — but race needs to interrupt remaining tasks after the first succeeds. The fix: race creates a child scope for its branches. `branchScope.close()` after advancement interrupts everything in the child scope without affecting the parent. Clean separation — the same code path handles both cases, just with a different scope target.

The Walker needed polymorphic parsing — `parallel:` now accepts either a list (today's syntax, backward compatible) or a map with the policy fields. The type check dispatches at parse time; both produce the same `ParallelStep` record. The evaluator takes a dual-path approach: `advanceAfter == steps.size()` uses the existing executor pattern unchanged; anything else routes to the SpawnedTask + latch path.

What this opens up: the yaml-step-runtime now has a general-purpose early-advance mechanism. Scenarios can model "query three data sources, proceed on the first response, collect the rest in the background" without custom orchestrator code. And race — reserved but never implemented — now has an implementation path: `parallel: { advance-after: 1, cancel-remaining: true }`.
