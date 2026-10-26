---
layout: post
title: "The Branch That Built Itself Twice"
date: 2026-08-30
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [judgment, reconciliation, interface-divergence, multi-repo]
series: issue-994-governed-yield-v2
---

# The Branch That Built Itself Twice

I went to close the governed yield branch — 56 commits across three repos, judgment SPIs landed, verification pipeline wired, the whole epic done. The close ceremony hit a rebase conflict wall. Thirty-three commits behind main. Forty-three conflicting files.

That's when the audit got interesting.

Main already had judgment types. Different ones. Someone — a parallel session working from the same coordination spec — had landed `JudgmentVerifier`, `VerificationContext`, and `VerificationResult` on main while this branch was building its own versions of the same things. Two independent implementations of the same feature, both complete, both working, and incompatible at three interface points.

The `verify()` signature was the sharpest divergence. Main embedded the response data into `VerificationContext` — ten fields, one parameter, everything the verifier needs in a single object. The branch kept `JudgmentResponse` as a separate type and passed it alongside the context — two parameters, richer types, cleaner separation. Both designs had defensible rationale. But a latent NPE in blocks' `LlmJudgmentPhase` — which called `verify(response, null)` because the second parameter was easy to forget — settled it. The 1-arg design is structurally immune to that class of bug.

The package and type-level decisions followed the same pattern. Main used `api.spi.judgment` (sub-package) where the branch used flat `api.spi`. Main used `String` for `TrustTooLow` levels where the branch used `double`. In every case, main matched the coordination spec from the blocks#170 design session. The branch had been built either before that spec was finalised or without incorporating it.

What the branch had that main didn't: `CallerConfig` — a sealed type splitting judgment callers into Human, LLM, A2A, and Any variants. Typed evidence with structured `EvidenceRequirement` records instead of plain strings. An escalation SPI for post-verification failure handling. Gate unification pulling `ActionGateScheduler` into `JudgmentScheduler`. A distributed `CloudEventJudgmentScheduler`. These are genuine capabilities, not duplications.

The reconciliation question was whether to rebase (resolving 30+ design-level conflicts where every resolution requires a judgment call) or rebuild (new branch from main, porting the branch's unique value onto main's established interfaces). Rebase would have been mechanically possible but intellectually dishonest — pretending to merge when you're actually redesigning at every conflict point. A fresh branch makes the design decisions explicit.

Three commits landed on the v2 branch in this session: the foundation types (CallerConfig, typed Evidence, CallerIdentity), the DagNode judgment field cherry-picked clean, and an enriched `VerificationContext` that grafts the branch's type richness onto main's 1-arg shape. The remaining phases — escalation, gate unification, the response handler reconciliation — each build on this foundation.

The work-end ceremony also surfaced two bugs in the close orchestrator's slot-mode handling. `_phase_skip()` was writing single-key progress entries that bypassed per-repo fan-out entirely — secondary repos in a three-repo slot were silently skipped. And the per-repo iteration blocked forever on the first failing repo with no mechanism to try the others. Both already fixed upstream, neither visible until a real three-repo close attempted it.
