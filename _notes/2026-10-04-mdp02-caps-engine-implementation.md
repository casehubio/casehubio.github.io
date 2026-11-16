---
layout: post
title: "From Topology to Attractors: Implementing the CAPS Engine"
date: 2026-10-04
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [caps, psychology, behavioral-inference, implementation, cognitive-systems]
series: issue-406-emergent-behavioral-synthesis
---

# From Topology to Attractors: Implementing the CAPS Engine

[Continues from: Mechanical Psychology: Building a CAPS Engine for Emergent AI Behaviour](2026-10-04-mdp01-mechanical-psychology-caps-engine.md)

The design session ended with "implementation starts next session." This is that session. Six batches, from SQLite persistence through LLM-backed situation classification, and a bug that revealed a conceptual gap I hadn't anticipated.

## Storage First

The InMemoryCapsEngine from the foundation batch proved the settling algorithm worked. But an agent's connection weights represent their entire psychological history — losing those between restarts isn't an option.

SqliteCapsEngine follows the pattern established by every other SQLite store in neocortex: HikariCP with WAL mode, Flyway migrations, three tables. `caps_agent_state` holds the generation counter. `caps_connection_weights` stores the per-agent weight overlays — excitatory, inhibitory, decay resistance, and precision for each connection. `caps_node_states` tracks activation thresholds and resting activations.

The interesting design choice was delete-and-reinsert for the weight maps rather than individual upserts. An agent might have 50+ connection weight overrides. Batch inserting after a single delete is simpler than tracking which weights changed and upserting individually. The generation counter guards against lost updates if two processes somehow try to write the same agent state.

## The Keyword Classifier Problem

The CAPS engine needs to know which input nodes a situation activates. The topology YAML already carries keywords on every input node — `rejection: [rejected, dismissed, unwanted, turned away, refused, ignored]`. A rule-based classifier that matches description text against these keywords is the mechanical baseline.

The first confidence formula was wrong. I had `matchCount / keywordCount` — so matching "rejected" against rejection's 6 keywords gave 0.167 confidence, below the 0.3 threshold. But natural descriptions hit one or two keywords at most. "She felt rejected by her peers" is clearly about rejection despite only matching one keyword out of six.

The fix was simple: `min(1.0, matchCount * 0.3)`. Each keyword match contributes 0.3 confidence. One match clears the threshold. Two matches give 0.6. Three give 0.9. This matches the intuition that even a single keyword is a meaningful signal — the threshold exists to filter noise, not to demand exhaustive keyword coverage.

Multi-word keywords needed a different matching path. "left out" won't match single-token comparisons. The classifier uses `fullText.contains()` for phrases with spaces, exact token matching for single words.

## BehavioralSynthesisPhase — Where CAPS Meets Consolidation

This is the bridge between the CAPS engine and the existing consolidation pipeline. It runs at `@Priority(19)` — after ExperienceConsolidationPhase (15) graduates experience memories into the COGNITIVE subgraph, but before MergeDetectionPhase (20) scans for duplicate nodes.

The phase does four things per consolidation tick:

1. Finds graduated experience nodes that haven't been processed yet
2. Classifies each into CAPS situation types
3. Settles the CAPS graph for that agent to produce behavioural attractors
4. Creates or strengthens attractor nodes in the BEHAVIORAL subgraph

The attractor nodes carry a `CapsGenerated` trait. MergeDetectionPhase needed updating to exclude these — CAPS attractor nodes have names like "approach" and "distrust" that would otherwise trigger false merge candidates with similarly-named nodes in other subgraphs.

### The Bug Claude Caught

During code review, Claude flagged that the phase wasn't marking processed nodes. The `findUnprocessedGraduated` method filters on `!node.property("caps-processed").isPresent()` — but I'd never written the property after processing. Every consolidation tick would re-classify and re-settle all graduated nodes from the beginning. No error, no crash, just silently redundant work growing linearly with the number of graduated experiences.

This is the category of bug where the symptoms only appear at scale. A test with five experience nodes runs fine — the overhead is invisible. An agent with five hundred graduated experiences re-running the CAPS settling algorithm five hundred times per tick would be a different story. The fix was one `store.updateNode()` call after successful settling.

## CognitiveEmergenceTest — Testing Emergence

The emergence test framework is an abstract base class that lets you seed a scenario and assert on what emerges. Set up an agent with a disposition, feed them experiences, settle the CAPS graph, and check which behavioural attractors appear.

```java
var scenario = scenario("agent1", new DispositionAxes(
    "cooperative", "moderate", "bold", "moderate", "cooperative"));

scenario.withExperience(
    "Was rewarded and praised for helping others", 0.7, 0.8, 5);
scenario.applyExperiences();

SettlingResult result = scenario.settle(Map.of("reward", 0.8));
assertAttractorPresent(result, "approach");
```

The most revealing test was the disposition comparison. Same situation (social threat at 0.6 intensity), two agents with different dispositions. The bold agent produces a stronger approach attractor than the cautious agent. Both produce some withdrawal — which is psychologically correct. A bold person facing a social threat doesn't ignore it; they approach more readily than a cautious person would.

I initially wrote the assertion as "bold agent approach > bold agent withdraw" — which failed. The CAPS topology correctly models that even bold agents show withdrawal responses to threats. The assertion should compare between agents, not within one agent's attractor strengths. The topology knows more psychology than my test assumed.

## The LLM Classifier

The keyword classifier is a deterministic baseline. It handles obvious cases — "she was punished" matches punishment, "he felt abandoned" matches abandonment. But it can't handle novel descriptions, metaphorical language, or complex multi-situation narratives.

The LLM classifier sends the description to an LLM with a topology-aware system prompt listing all 30 input nodes, their categories, and their keywords. The LLM returns JSON: `{"activations": [{"node": "rejection", "confidence": 0.8}]}`. The classifier validates node IDs against the topology, filters below-threshold confidences, and clamps values to [0, 1].

It lives in its own module — `caps-llm-classifier` — keeping the LLM dependency (AgentProvider from platform) isolated from the pure-Java caps-engine. When the module is on the classpath, it activates via CDI. When it isn't, BehavioralSynthesisPhase falls back through `Instance<SituationClassifier>`. The keyword classifier is a plain Java class used directly by test infrastructure; the LLM classifier is a CDI bean used at runtime. They serve different contexts, not competing alternatives.

## What's Next

The CAPS engine is the mechanical inference layer. It computes behavioural tendencies from experiences filtered through personality. What it doesn't do — yet — is connect to the rest of the cognitive pipeline.

The next issues in the epic will wire the outputs: BehavioralPromptSection renderer (#410) surfaces the crystallised attractor strengths to the LLM as a behavioural profile. Runtime gut feeling (#409) handles novel situations where no pre-crystallised attractor exists. And the calibration problem — validating that the estimated weights produce psychologically coherent behaviour — remains the deepest open question. The topology is grounded in published research. Whether the composition of six models through shared mediating nodes produces emergent behaviour that trained psychologists would recognise as realistic is something only empirical testing will answer.
