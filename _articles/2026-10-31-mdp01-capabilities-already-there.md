---
layout: post
title: "The Capabilities Were Already There"
date: 2026-10-31
entry_type: article
subtype: diary
projects: [casehubio/neocortex]
tags: [cognitive-architecture, cdi, wiring, salience, dead-code]
series: issue-736-wire-unwired-cognitive-caps
---

# The Capabilities Were Already There

The cognitive pipeline had everything it needed. ConcernBasedSalienceStrategy — a full Klinger Current Concerns implementation that filters perception based on active drives and goals. EntityAwareSalienceStrategy — a decorator that boosts salience for observations mentioning known entities. GoalEmotionMoodBridge — a bridge from goal outcomes back into emotional state. CognitiveGoalOrchestrator — progressive goal graph management via MindMap.

All of them existed. None of them ran.

A systematic audit found six categories of dead or disconnected code across the neocortex cognitive pipeline. Types that had been carefully designed, implemented, and tested — then lost during the extraction from agency to neocortex. The composition root wiring didn't make the journey.

This is the quiet failure mode of modular cognitive architectures. The components work. The tests pass. The system runs. But the thing you designed — the perception filter that foregrounds relevant concerns, the affect loop that lets goal success influence mood — is silently absent. Nothing breaks. The system just operates at a fraction of its design capacity.

## What "wired" means

The neocortex cognitive pipeline uses a three-layer wiring pattern. Each capability needs all three to activate:

![Three-layer wiring pattern](images/three-layer-wiring.svg)

**Layer 1: CognitivePipelineBuilder** — a fluent builder that accepts nullable fields. Each capability gets a builder method. If the capability's dependencies aren't on the classpath, the field stays null.

**Layer 2: CognitionCore.configure*()** — the composition root creates the tick participant, registers it at the correct phase (FOUNDATION, SOURCE, DERIVED, or TERMINAL), and optionally chains prompt section customisers. The phase determines when the participant runs relative to other cognitive processes.

**Layer 3: CognitionCoreFactory** — CDI wiring. `@Inject Instance<T>` with an `isResolvable()` guard. If the bean exists on the classpath, it's injected. If not, the builder method is never called and the capability doesn't activate.

This pattern gives clean separation of concerns. A capability that requires MindMapStore only activates when MindMapStore is on the classpath. A capability that requires an LLM backend only activates when SessionPipeline is available. The pipeline degrades gracefully rather than failing on missing dependencies.

The pattern also means a capability that's missing any of the three layers is dead. The class exists. The tests pass. Nobody notices.

## Salience: from NoOp to concern-driven perception

The most interesting fix was the salience pipeline. The `SalienceStrategy` SPI had a `NoOpSalienceStrategy` as its `@DefaultBean` — meaning every observation got equal salience regardless of active drives or goals. ConcernBasedSalienceStrategy existed but was never produced by CDI.

The fix was a conditional producer in `CognitionDefaultBeans`:

```java
SalienceStrategy salienceStrategy(
        Instance<MindMapStore> mindMapStore,
        Instance<EntityAnnotationIndex> annotationIndex) {
    SalienceStrategy base;
    if (mindMapStore.isResolvable()) {
        base = new ConcernBasedSalienceStrategy();
    } else {
        base = new NoOpSalienceStrategy();
    }
    if (annotationIndex.isResolvable()) {
        return new EntityAwareSalienceStrategy(base, annotationIndex.get());
    }
    return base;
}
```

Two independent activation conditions compose naturally. ConcernBased displaces NoOp when MindMapStore is available. EntityAware layers on top when EntityAnnotationIndex is available. Either can be present without the other. When both are absent, NoOp still functions as a safe default.

This required one API change: `EntityAwareSalienceStrategy` originally took `tenantId` as a constructor parameter — which doesn't work for a CDI-managed singleton. The fix was adding `tenantId` to `SalienceContext` so it's available at call time rather than construction time. A backward-compatible five-argument constructor preserves existing callers.

## Why dead code isn't just technical debt

The usual conversation about dead code is about cleanup — remove it, reduce the surface area, cut the maintenance burden. This is different. These capabilities were designed to work together. ConcernBasedSalienceStrategy makes the appraisal pipeline pay attention to what matters. GoalEmotionMoodBridge closes the affect loop between goal outcomes and emotional state. Without them, the cognitive system operates in a mode where every observation gets equal weight and goal success has no emotional consequence.

The system doesn't crash. It just doesn't do what it was designed to do.

Systematic auditing — walking the composition root, checking which capabilities have all three wiring layers, verifying that config flags actually gate something — is the only way to catch this. Unit tests don't help because the individual classes work fine. Integration tests don't help unless they specifically assert that the wired capability changes pipeline behaviour. The gap is between "this class works" and "this class is reachable from the thing that runs."

For cognitive architectures specifically, this matters more than in most systems. A web service with a dead endpoint is a cleanup task. A cognitive pipeline with a dead perception filter is a design failure that's invisible to every metric except the quality of the system's reasoning.
