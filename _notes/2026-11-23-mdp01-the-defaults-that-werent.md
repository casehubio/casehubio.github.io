---
layout: post
title: "The Defaults That Weren't"
date: 2026-11-23
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [caps, edge-inference, disposition, coping-bias, personality, json-parsing, cognitive-architecture]
---

# The Defaults That Weren't

Three bugs walked out of a code review session today. None of them were crashes. None of them were wrong outputs. They were the other kind — the kind where the system works exactly as written and produces exactly the wrong thing.

## The regex that assumed politeness

`EdgeInferencePhase.parseRelationships()` was using a regex to extract relationship triples from LLM responses:

```java
Pattern.compile("\"source\"\\s*:\\s*\"([^\"]+)\"\\s*,\\s*"
    + "\"target\"\\s*:\\s*\"([^\"]+)\"\\s*,\\s*"
    + "\"type\"\\s*:\\s*\"([^\"]+)\"");
```

The system prompt asks the LLM for `{"source": "...", "target": "...", "type": "..."}` in that order. The regex only matches that order. If the LLM returns `{"type": "knows", "source": "Alice", "target": "Bob"}` — perfectly valid JSON, same content — the regex silently drops the relationship. No error, no log, no indication that data was lost.

The JSON spec does not guarantee key ordering. The system prompt requests it, but requesting is not enforcing. We replaced the regex with Jackson's `ObjectMapper.readTree()` — four lines of null-safe field access instead of a pattern match against assumed politeness.

The existing tests from #1190 had already documented this limitation: `parseRelationships_differentKeyOrder_noMatch` was explicitly testing that reordered keys produced no results. That test name was a warning sign nobody acted on. After the fix, those same tests assert the correct behaviour — both orderings extract both relationships.

## The personality that was always cooperative

`BehavioralSynthesisPhase.run()` auto-initializes CAPS state for agents it hasn't seen before. The initialization needs a `DispositionAxes` to seed the network's connection weights:

```java
state = capsEngine.initializeAgent(tenantId, agentId,
    new DispositionAxes("cooperative", "moderate", "calculated",
                        "moderate", "cooperative"));
```

Every agent that was auto-initialized during consolidation got the same personality. The competitive agent got cooperative weights. The impulsive agent got calculated weights. The confrontational agent got cooperative conflict mode. The phase already had `effectiveDefaults` available — it was using it three lines later to look up expression styles — but the auto-init path hardcoded a default that overrode whatever the agent's registered personality actually was.

The fix is a lookup with a fallback: resolve the agent's disposition from `effectiveDefaults`, use the hardcoded default only when no profile is registered. The interesting question is how long this was silently flattening personality differences in the CAPS pipeline. The `CognitionTestHarness` had been working around it by pre-initializing CAPS state before calling consolidation — which meant the auto-init path was never exercised in the test harness. The production code path was untested for this exact case.

## The gradient that feedback swallowed

The coping bias test (`moderateThreat_copingBiasIsTiebreaker`) was asserting that coping defaults have more effect under moderate threat than strong threat. The idea: at moderate threat levels, the network's resting activations (set by coping defaults) act as a tiebreaker between competing strategies. At strong threat levels, connection-driven activation dominates and the resting bias washes out.

After #1192 added feedback loops to the CAPS topology, both values collapsed to ~2.5e-5 — effectively zero, differing by floating-point noise. The assertion was relaxed to `isCloseTo` with a 1e-4 tolerance to stop it from failing.

I ran a diagnostic. The numbers told the story: moderate divergence 0.0000251658, strong divergence 0.0000251658. Identical. Not approximately equal — identical to machine precision. The same pattern held with extreme coping contrast (planful=1.0, avoidant=0.0): moderate and strong divergence were both 0.0000419430.

The feedback loops equalized the effect. In a feedforward network, the resting activation has different leverage at different input intensities because the signal flows in one direction and the resting bias adds a fixed offset. With feedback loops, the recurrent energy redistributes the activation across the network until the resting bias contributes a constant delta regardless of input intensity. The property didn't weaken — it became threat-invariant.

The fix renames the test to `copingBiasDirectionallyDifferentiatesProfiles` and asserts what's actually true: the directional property (planful agent has higher planning-vs-avoidance gap than avoidant agent) holds at both threat levels, and the divergence is consistent across threat levels rather than gradient.

## The pattern

A regex that assumes ordering. A default that assumes identity. A test that assumes gradient. Three places where an assumption was encoded as a default and the system silently did the wrong thing — not because it was broken, but because it was working exactly as written.

The fix in each case is the same: replace the assumption with a lookup. Look up the actual JSON structure. Look up the actual personality. Look up the actual divergence values. The assumption was always a placeholder for information that was available but not consulted.
