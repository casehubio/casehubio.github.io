---
layout: post
title: "Four Annotations for Human-in-the-Loop"
date: 2026-08-19
entry_type: note
subtype: diary
projects: [casehub-work]
tags: [annotations, quarkus-extension, human-in-the-loop, casehub]
---

The programmatic path for creating WorkItems works — `HumanTaskFlowBridge.requestApproval()` plus `WorkItemCreateRequest.builder()` gets the job done. But it's fifteen to thirty lines per approval gate, and when you're wiring four or five of them across an agent model, the repetition obscures the intent.

The annotation model compresses that to one line:

```java
@HumanApproval(
    title = "Approve expense report",
    candidateGroups = {"finance-team"},
    priority = WorkItemPriority.HIGH,
    claimDeadline = "PT1H")
public String approve(String reportJson) {
    return null;
}
```

Four annotations, each an orthogonal concern: `@HumanApproval` for per-item configuration, `@RequiresQuorum` for M-of-N coordination, `@Escalate` for escalation policy, and `@SkillMatch` for routing strategy selection. They compose freely — stack what you need, the build extension validates the combinations at compile time. `@Escalate` without a base annotation? Build error. Void return type? Build error. Quorum parameters out of range? Build error.

The composition model matters because these annotations aren't just for standalone Quarkus beans. The blocks-engine-adapter will process the same annotations on `@Worker` and `@SupervisorAgent` methods, generating governance interceptors that integrate with the engine execution model. The `WorkAnnotationsProcessedBuildItem` coordinates between the two build extensions — the work processor emits it, the blocks processor checks for it and skips methods it handles itself. No double-processing.

The return type is the outcome type. The method body never executes — the interceptor creates the WorkItem, suspends the caller, and deserializes the human's resolution into the return type when it arrives. `String` passes through raw; any other type gets Jackson deserialization. `void` is a build error because there's nothing to return. This transparent suspension model means callers don't know or care that a human was involved — the method looks synchronous (or returns `Uni<T>` for async).

An interesting design constraint surfaced around the runtime module's dependencies. The annotations themselves only reference API types — `WorkItemPriority`, `OnThresholdReached` — so the runtime module depends on `casehub-work-api`, not on the full `casehub-work` runtime. If it pulled in the runtime transitively, every test that uses the annotations would need a datasource configured, because Hibernate would discover the WorkItem entities and demand one. Clean dependency boundaries prevent that contamination.

Testing the build extension processor was less straightforward than expected. The documented approach — `QuarkusUnitTest` with `@RegisterExtension` — silently failed to discover the extension. The build step simply never ran, and the app started with just CDI, no annotations extension. We spent time adding every metadata file the framework requires (`quarkus-extension.properties`, `quarkus-extension.yaml` in both runtime and deployment), installing both modules, comparing against a working reference extension line by line. Same structure, different result. We switched to testing the validation logic directly with the Jandex `Indexer` API — build an index from compiled test classes, call the processor's validation methods, assert the exceptions. Seven validation tests, all passing, and far more readable than the `QuarkusUnitTest` equivalent would have been.
