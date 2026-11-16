---
layout: post
title: "The Throwing No-Op"
date: 2026-10-22
entry_type: note
subtype: diary
projects: [casehubio/neocortex, casehubio/examples]
tags: [cognitive-architecture, anti-pattern, AIMD, mood-appraisal, observability]
---

# The Throwing No-Op

I went looking at two open issues — #662 and #663 — both filed during a parameter tuning run that showed a 34% LLM cognitive tick rate. The hypothesis was that `session.query()` bypassed AIMD admission control, starving cognitive pipeline calls of session permits. Sounds plausible. Was wrong.

We traced the actual code paths. `CognitionCore` calls `ParallelLlmExecutor.execute()` with `"tick"` as the admission context. That calls `pipeline.companionAdmission("tick", ceiling)` — a separate AIMD controller with its own concurrency budget. Cognitive ticks never competed with scenario calls for permits. The two pipelines are already isolated. The 34% rate had a different root cause entirely: consolidation trigger decay, fixed by #661's monotonic accumulation.

Both issues closed. But the investigation turned up something more interesting than the original bug.

`RuleBasedSituationClassifier.classify()` threw `UnsupportedOperationException`. The comment explained why — keyword matching misclassifies narrative text. "Bullied" maps to dominance (the problem described), not acceptance (the gallant response). Fair enough. But the fix was to throw, not to return empty.

Three production call sites injected this classifier via CDI and called `classify()`: `BehavioralSynthesisPhase`, `DriveProfileDerivationPhase`, and `CharacterCognition`. Every one of them caught the exception and swallowed it. Zero activations produced. Zero errors logged at the call site. The degradation was completely invisible — consolidation phases ran, produced results, but with no situation classification informing the drive profiles. The symptom was subtly wrong output, not failure.

One call site in `BehavioralSynthesisPhase.mergeDriveIntensities()` had no catch at all. If a drive type wasn't in the static mapping, the code fell through to `classifier.classify()` and would have crashed the consolidation phase at runtime.

The fix was one line: `return List.of()` instead of `throw`. A no-op is the correct behaviour when a component is disabled — it means "I have nothing to contribute," not "I refuse to participate." Callers already handle empty results gracefully. They don't handle explosions gracefully, even when they catch them.

This is a pattern worth naming: the *throwing no-op*. When you need to disable a component but it implements an interface that other code depends on, the temptation is to throw `UnsupportedOperationException` to make it obvious the component isn't ready. But in a system with CDI injection and optional dependencies, "obvious" means "caught and swallowed at every call site." The throw becomes indistinguishable from silence — except it's worse than silence, because silence would at least let a `null` check short-circuit the entire code path. The throw runs the path, fails, catches, and produces nothing. All the cost, none of the signal.

While we were in the neighbourhood, I added two things. `CognitiveTickMetrics` — a record with atomic counters tracking mood appraisal and BDI extraction outcomes (success, fallback, skipped, error). The 34% claim from the parameter tuning run was based on manual log parsing of a single run. Without instrumentation, we can't confirm whether companion AIMD separation actually improved things. Now there's a `tickMetrics()` method that gives the answer in one call.

The other addition: a mood appraisal fallback. When the `SessionPipeline` is unavailable, `appraiseMood()` was recording zero PAD delta — mood decayed toward neutral with no new interaction signal. The heuristic replays the last successful PAD delta at 50% strength per tick, fading exponentially until it crosses a 0.01 threshold. Not a psychological model — just signal preservation. Better than silence; clearly marked as `"fallback"` in the cause field so metrics can separate it from real appraisals.

The question the metrics will answer: is the 34% still the real number, or has companion AIMD separation already fixed it? If the success rate is above 80%, the fallback rarely fires and the issue is closed. If it's still low, there's a real concurrency problem that the AIMD investigation correctly ruled out of the cognitive pipeline — meaning it's somewhere else in the stack.
