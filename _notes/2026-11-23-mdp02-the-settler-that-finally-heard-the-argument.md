---
layout: post
title: "The Settler That Finally Heard the Argument"
date: 2026-11-23
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [cognitive-dissonance, caps, constraint-satisfaction, shultz-lepper, behavioral-synthesis, direct-path, settling, design]
---

# The Settler That Finally Heard the Argument

In the previous entry I described how DissonanceDetectionPhase detects contradictions between a character's beliefs and behaviour, then renders the felt tension as a prompt section. The character *feels* the contradiction — arousal rises, pleasure drops, and the prompt describes "an uncomfortable tension between parts of yourself that don't fit together."

But there was a gap. The CAPS settler — the actual constraint-satisfaction engine that shapes the character's behavioural landscape — never saw the contradiction directly. It only saw the downstream consequences: the character's conversation behaviour, influenced by the tension prompt, eventually produced experiences that graduated into the COGNITIVE subgraph, got classified by SituationClassifier, and finally settled through CAPS. Two ticks minimum. Often more, because graduation requires corroboration.

This is the indirect path. It works. It's even psychologically faithful — Festinger describes dissonance reduction as gradual, not instantaneous. But it has a theoretical fidelity problem.

## The constraint satisfaction gap

Shultz and Lepper (1996) formalised dissonance as constraint satisfaction: beliefs are nodes in a connectionist network, consonance relations are excitatory connections, dissonance relations are inhibitory, and the network settles to minimise constraint violations. This is literally what the CAPS settler does. It takes input activations, propagates through excitatory and inhibitory connections, and converges on an attractor pattern.

But in the indirect path, the settler never sees the competing belief activations. It sees *derivative* experience-based classifications — a flattened, single-label summary of what the character did during conversation. The constraint satisfaction happens, but over the wrong inputs. The competing beliefs that caused the tension are lost in translation.

## Giving the settler the actual argument

The direct path fixes this. DissonanceDetectionPhase already asks the LLM for CAPS activation maps when it detects a contradiction. For example: "I am a loyal friend" maps to `{acceptance: 0.7, sharing: 0.5}`, while "I would betray anyone for power" maps to `{dominance: 0.8, competition: 0.7}`. These were stored on tension nodes for observability but never consumed.

Now BehavioralSynthesisPhase reads these activations through a DissonanceActivationSource SPI and merges them into the combined activation map before settling. Both sides of each contradiction go in, scaled by tension strength. At strength 0.85, the settler sees `acceptance: 0.60` and `dominance: 0.68` in the same settling pass. Because `acceptance` and `dominance` have inhibitory connections in the CAPS topology, the network cannot sustain both — it has to resolve the conflict. The resulting attractor pattern reflects the actual tension, not a derivative approximation.

The merge uses `Math::max` — same strategy as drive-intensity merging. Dissonance activations only override experience or drive activations when they're the stronger signal per input node. This means the direct path adds pressure without overriding what the character is currently doing.

## What emerged

The module dependency graph posed an interesting constraint. BehavioralSynthesisPhase lives in `mindmap-intelligence`, DissonanceDetectionPhase lives in `cognition`, and `cognition` already depends on `mindmap-intelligence`. Direct injection would create a circular dependency. The solution was a small SPI in `cognition-api` — a `DissonanceActivationSource` interface that BehavioralSynthesisPhase injects via `Instance<T>`, with graceful degradation when dissonance is disabled.

The priority reorder (DissonanceDetectionPhase from `@Priority(25)` to `@Priority(17)`) means detection runs before settling, so activations are ready when BehavioralSynthesisPhase needs them. The tradeoff: detection no longer sees same-tick attractor changes. Since attractors are stable across ticks, this is a non-issue in practice.

Both paths now coexist. The direct path provides same-tick attractor shifts — the character's behavioural landscape changes as soon as a contradiction is detected. The indirect path captures the character's conversational response to tension across subsequent ticks. They operate at different timescales and reinforce each other. The settler finally hears both sides of the argument.
