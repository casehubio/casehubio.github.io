---
layout: post
title: "Three Ways Tests Lie"
date: 2026-07-31
type: phase-update
entry_type: note
subtype: diary
projects: [engine]
tags: [testing, jpaf, personality-routing, bugs]
---

The JPAF personality adaptation pipeline had full infrastructure — signal recording, effective weight computation, reflection triggers, decay mechanics. Every component had unit tests. All green. The pipeline did nothing.

`PersonalitySignalRecorder.record()` had zero callers. The class was `@ApplicationScoped`, CDI would inject it, and seven unit tests verified its internal methods: reinforcement selects the right cognitive function, compensation picks the right compensatory function, reflection triggers decay on dampening. Every test passed because every test called the internal methods directly. Nobody tested that `record()` was ever invoked from the execution pipeline.

This is the first pattern: **tested components, untested wiring.** Unit tests prove the parts work. Integration points between the parts — where one class calls another — are where the system actually lives. A class with complete test coverage and zero callers is dead code with a green badge.

The second pattern is subtler. `DispositionSignalStore.decay()` multiplies activation counts by a decay factor. The JPAF paper specifies `0.2×` multiplicative decay — retain 20%, aggressive dampening when reflection rejects a personality shift. The implementation used `count * (1 - decayFactor)`. With `decayFactor = 0.2`, that retains 80%. Nearly inert.

The test said `decay_removes_fraction_of_counts`: 10 activations, `decay(0.20)`, assert equals 8. Green. Correct for the formula as written. Wrong for what the formula should do. The test didn't verify the spec — it verified the code. When the code is wrong, verifying the code just locks in the bug.

**Tests that verify the implementation rather than the specification are tests that protect bugs.** The decay tests had the right shape — call decay, check the result — but the expected values came from reading the code, not from reading the JPAF paper. Once the wrong formula was written, the test was written to match it, and the bug became invisible.

The third finding: `DefaultDispositionEvolution.evaluate()` called `signalStore.decay()` internally before returning `Evolved`. A method named `evaluate` that mutates state. The caller — `PersonalitySignalRecorder.checkReflection()` — handled the `Evolved` case by logging. It didn't call `clear()` or `decay()` because it didn't know it needed to — `evaluate()` had already done it, silently.

This is the query-that-mutates pattern. When a method named for evaluation has a side effect, callers can't reason about state. The `Dampened` path was handled correctly — the caller explicitly called `signalStore.decay()`. The `Evolved` path looked like it did nothing, but the state had already changed inside `evaluate()`. Two paths, same end result, different ownership — one explicit, one hidden.

All three patterns share a root: **false confidence.** Tests pass, so the code works. The formula produces a number, so the math is right. The method returns a result, so it's a query. Each creates a surface that looks correct while hiding a flaw underneath. The fix for each is the same: verify against the *intention*, not the *implementation*. What should `record()` be called by? What should `decay(0.2)` produce according to the spec? What should `evaluate()` leave unchanged?

The engine's JPAF pipeline is live now — signals flow from worker completion through reinforcement or compensation, effective weights shift, reflection triggers when thresholds cross. The decay factor is a `PreferenceKey`, configurable per tenancy. `evaluate()` is side-effect-free. The interesting question is whether the activation-count-based approach (no materialized TemporaryWeight state, computed at probe time) holds up under real workloads, or whether the integer truncation in decay creates ratcheting artifacts at low counts.
