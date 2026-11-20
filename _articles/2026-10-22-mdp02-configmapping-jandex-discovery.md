---
layout: post
title: "Why Your @ConfigMapping Won't Inject (Even When Other Beans Do)"
date: 2026-10-22
entry_type: article
subtype: diary
projects: [casehubio/examples, casehubio/neocortex]
tags: [quarkus, cdi, jandex, configmapping, cognitive-architecture]
series: issue-139-externalize-need-satisfaction
---

# Why Your @ConfigMapping Won't Inject (Even When Other Beans Do)

I had a clean three-line change: take two `@ConfigMapping` interfaces that already existed in neocortex-cognition, inject them via CDI instead of constructing hardcoded records. The neocortex side already had `@ConfigMapping(prefix = "casehub.needs-pyramid")` with `@WithDefault` annotations. The examples side was bypassing it entirely — building manual records with the same values. Remove the records, add two parameters to the producer method, done. Minus 29 lines, plus 3.

It compiled. The 18 default values matched field-by-field. No competing producers, no test overrides, no scope mismatch. Everything looked right.

Then Quarkus refused to start.

```
[1] Unsatisfied dependency for type DriveAdaptationConfig and qualifiers [@Default]
    - injection target: parameter 'config' of ManorConsolidationBeans#driveAdaptationPhase()
```

The neocortex-cognition module doesn't ship a Jandex index. Without one, Quarkus can't discover its `@ConfigMapping` interfaces during augmentation. SmallRye Config needs the annotation in the combined Jandex index to auto-generate the CDI bean — if it can't see the annotation, no bean gets registered.

## The Misleading Existence Proof

Here's what made this tricky: other CDI beans from the same unindexed module were already working in the application. `NeedTierMappingProvider` is defined in neocortex-cognition and used in the producer method I was modifying — and it resolves fine. `MoodPromptSection`, `AppraisalPromptSection`, `CognitionCoreFactory` — all neocortex-cognition beans, all active.

Claude investigated and initially reported an existence proof: if `PromptRenderingConfig` (a `@ConfigMapping` in the same module) already works via CDI, then our interfaces should too. A contradicting investigation said the module lacks Jandex and the beans can't resolve. Both had evidence.

The resolution: the beans that work all have local implementations in the wacky-manor application. `ManorNeedTierMappingProvider` implements `NeedTierMappingProvider` and carries `@ApplicationScoped`. CDI finds the local bean through wacky-manor's own Jandex index — it never needs to discover the interface in the unindexed cognition module. The interface is just a type on the classpath.

`@ConfigMapping` is different. There's no local implementation. SmallRye Config must auto-generate one — but only if it sees the `@ConfigMapping` annotation during augmentation. Without Jandex on the cognition JAR, it doesn't see it.

The existence proof was real but irrelevant. Locally-implemented interfaces and auto-generated config mappings use different discovery paths.

## The Fix Had Its Own Trap

The obvious fix — add `quarkus.index-dependency` to `application.properties` — worked for `@ConfigMapping` discovery but exposed a new problem. Indexing the entire cognition module made Quarkus discover `LlmReflectionSynthesizer`, an `@Alternative @Priority(1)` bean that conflicts with the application's own `ManorReflectionSynthesizer`. The cognition module was unindexed for a reason — though probably not this reason. It was an accidental firewall.

The actual fix has two parts. First, add the Jandex plugin to the neocortex-cognition POM so the module ships its own index — consistent with 30 other neocortex modules that already do. Second, exclude `LlmReflectionSynthesizer` from CDI in the wacky-manor application. The exclusion is the right pattern: the application chooses its implementation, the library provides an alternative that's only active when nothing else is.

An audit across all 70 neocortex modules found three with significant CDI exposure and no Jandex: cognition (10 beans, 3 config mappings), cognitive-index (6 beans), and rag-core (3 config mappings). All three got the plugin in the same commit.

## The Stale Issue Discovery

While verifying the fix, I discovered the test suite hadn't been running — two pre-existing failures were blocking all `@QuarkusTest` execution. A missing `inference-bayes` dependency version in the POM and a `CognitionCore` constructor mismatch where tests passed 18 arguments to a 17-parameter constructor. Both were cross-repo sync issues that had accumulated silently.

Fixing those unblocked the tests: 622 passed, including CDI bootstrap validation. The `@ConfigMapping` beans resolved. Drive adaptation config and need satisfaction config are now tunable via `application.properties` — the original three-line intent, delivered with about 50 lines of infrastructure discovery along the way.

A sweep of open issues across the project turned up 24 that had already been fixed on main — commits referencing the issue numbers, code verified present, issues still marked open. They'd been fixed during larger work streams and never closed. The sweep was mechanical: check each open issue number against `git log`, verify the fix covers the full scope, close with a reference.

The config externalization itself was the smallest part of the session. The value was in what the audit uncovered: a Jandex gap affecting three modules, two test-blocking sync failures, and two dozen issues that should have been closed weeks ago. A three-line change that needed fifty lines of infrastructure to land safely — and cleaned up debt along the way.
