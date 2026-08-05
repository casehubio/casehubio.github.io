---
layout: note
title: "When Your Progress Model Can't Keep Up"
date: 2026-08-04
entry_type: note
subtype: diary
type: phase-update
projects: [casehub-work]
tags: [progress, extensibility, rollback, casehub, platform-design]
published: true
---

# When Your Progress Model Can't Keep Up

Three built-in shapes — percentage, count, step — cover most progress tracking. But "most" is a ceiling, and a platform that only handles the predicted cases isn't a platform. It's a library with opinions.

CaseHub's progress module just gained three capabilities that make it genuinely extensible: custom JSON Schema shapes, visualisation hints, and single-instance rollback controls. Each solves a real problem for consumers building on the platform.

## Custom shapes: bring your own validation

The `custom` shape type lets consumers define a JSON Schema in the `definition` field and validate state against it on every update. The schema is stored alongside the instance — no external registry, no classpath dependency, no waiting for a platform release.

```java
ProgressCreateRequest request = new ProgressCreateRequest(
    tenancyId, "deployment", nodeId, "custom",
    mapper.createObjectNode().put("score", 85).put("healthy", true),
    null, null,
    mapper.createObjectNode()
        .putPOJO("schema", deploymentHealthSchema)
        .put("rollbackField", "score"),
    null, "gauge");
```

The interesting design question was rollback detection. For percentage, we know what "backward" means — the number went down. For a custom shape, we don't.

Three tiers handle this. Most consumers won't configure rollback detection at all — state changes are always `STATE_UPDATED`. Consumers who want simple regression detection declare a `rollbackField` pointing at a numeric property in their schema. And consumers with domain-specific rollback semantics — a deployment health score that considers multiple dimensions, not just a single number — implement a `CustomRollbackDetector` and reference it by ID. Same SPI pattern as rollup strategies, same resolution via `StrategyResolver`.

The tiers compose without leaking complexity. A flat score uses tier two. A multi-dimensional health check uses tier three. Neither needs to know the other exists.

## Visualisation modes: a hint, not a prescription

A `visualisationMode` field on every `ProgressInstance`. Set at creation, returned on queries, ignored by the platform entirely.

```java
public final class VisualisationModes {
    public static final String GAUGE = "gauge";
    public static final String PROGRESS_BAR = "progress-bar";
    public static final String STEP_LIST = "step-list";
    public static final String TIMELINE = "timeline";
    public static final String TREE_MAP = "tree-map";
    public static final String COUNT_BADGE = "count-badge";
}
```

Platform constants are conventions. Consumers can use `"gantt"` or `"deployment-topology"` without waiting for us. The platform provides the vocabulary; consumers extend it.

The default mapping is documented but unenforced: percentage → gauge, count → progress-bar, step → step-list. A dashboard that renders a percentage as a sparkline instead of a gauge doesn't violate anything — it made a rendering choice.

## Rollback controls: harder than "undo last change"

This started as the simplest feature and turned into the most interesting.

The rollback policy is straightforward: set `rollbackPolicy: "denied"` on a progress instance, and any backward state movement via `PUT /state` is rejected with a 409. Forward movement is unaffected. This protects against accidental regression — a consumer reporting stale data, a race condition between two updaters — without blocking the explicit `POST /rollback` endpoint, which is a deliberate undo action.

The explicit rollback endpoint is where things got interesting. The naive algorithm — take the `previousState` of the most recent event — oscillates. Call rollback twice and you're back where you started, because the first rollback emitted an event that the second one undoes.

The obvious fix — skip `ROLLED_BACK` events when scanning — breaks for a different reason. When a consumer legitimately updates state backward via the normal `PUT /state` endpoint, the progress model auto-detects the regression and emits `ROLLED_BACK`. Skip those events, and you've just jumped over a change the consumer made deliberately.

The root problem: organic backward movement and explicit rollback both emit the same `ROLLED_BACK` change type. The event trail can't distinguish them.

We solved this with a forward-state-sequence algorithm. Build the list of states that were set by `CREATED` and `STATE_UPDATED` events — the forward progression, ignoring all `ROLLED_BACK` events. Find the current state's position in that list. Return the one before it.

Consecutive rollbacks walk back through the forward history: S0 → S1 → S2, first rollback → S1, second rollback → S0. No oscillation. If the current state isn't in the forward sequence (set by an organic backward movement), a fallback path finds the event that produced it and undoes that — restoring the state before the regression.

The algorithm sidesteps the change type ambiguity entirely by reconstructing the progression history and navigating it positionally, instead of trying to classify events by source.

## What this means for consumers

Custom shapes mean your domain-specific progress — deployment health, ML training metrics, compliance checklists — gets the same validation, event semantics, and rollup cascade as the built-in shapes. No special treatment, no second-class API.

Visualisation hints give dashboards a standard convention without prescribing rendering. Rollback controls let orchestrators protect progress state from accidental regression while keeping deliberate undo available.

Multi-instance coordinated rollback — rolling back an entire subtree atomically — is next, under the federation epic. The single-instance foundation built here is the prerequisite.
