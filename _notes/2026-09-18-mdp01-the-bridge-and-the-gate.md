---
layout: post
title: "The Bridge and the Gate"
date: 2026-09-18
entry_type: note
subtype: diary
projects: [casehubio/platform]
tags: [spring-boot-4, jackson, quarkus, version-alignment, testing]
series: issue-501-spring-deployment-readiness
---

# The Bridge and the Gate

Last session built nine Spring Data JPA stores. This session asked the question those stores exist to answer: does the whole thing actually boot?

The platform was still on Spring Boot 3.4.5 and Quarkus 3.32.2. The parent BOM had already bumped to Spring Boot 4.1 and Quarkus 3.39 for cross-repo alignment on transitive dependencies like JUnit and Jackson, but the platform was overriding both versions locally. Removing the overrides was the first move — and immediately surfaced the real design question.

## The Jackson Problem

Spring Boot 4 defaults to Jackson 3. Jackson 3 uses a completely different Maven group ID (`tools.jackson.core`) and Java package (`tools.jackson.databind`). Quarkus 3 is still on Jackson 2 and won't move to Jackson 3 until Quarkus 4 ships in November.

That would be fine — the two versions coexist on the classpath because they have different coordinates. The annotations even share the same `com.fasterxml.jackson.annotation` package deliberately. But three of our core modules take `ObjectMapper` as a constructor parameter: `DeliveryTracker`, `DeliveryRetryProcessor`, and `CallbackDispatcher`. Spring Boot 4's auto-configured mapper is Jackson 3's `JsonMapper` — a completely different type. The Quarkus CDI layer injects Jackson 2's `ObjectMapper`. If the core modules can't accept the same type from both frameworks, the dual-framework architecture breaks.

Spring Boot 4 ships `spring-boot-jackson2` for exactly this situation: a deprecated bridge module that auto-configures a Jackson 2 `ObjectMapper` instead of Jackson 3's `JsonMapper`. The deprecation is the point — it's a migration aid, not a permanent state. When Quarkus 4 aligns on Jackson 3, we drop the bridge and do one clean migration pass across everything. No throwaway abstractions, no split-brain period.

## The Groovy Surprise

After the version bump compiled clean across all eighty modules, the test suite revealed something unexpected: 73 tests failing with a `NullPointerException` deep in Groovy's `ClosureMetaClass.invokeOnDelegationObject`. The stack trace pointed at `Class.isAssignableFrom` with a null argument — which looks like a JVM bug or a Groovy internal error, not a version incompatibility.

Claude traced it to a mismatch between REST Assured 5.5.6 (pulled by the Quarkus BOM) and Groovy 5.0.8 (also pulled by the same BOM). REST Assured 5.x only supports Groovy 4. The Quarkus BOM bumped Groovy to 5 but didn't bump REST Assured in the same release — a transitive compatibility gap that fails at runtime with no compile-time signal.

The fix was REST Assured 6.0.1. But getting the version override to stick required pinning it in the consuming POM *before* the Quarkus BOM import. I'd initially pinned it in the parent BOM's `dependencyManagement`, expecting the explicit declaration to override the imported Quarkus BOM version. It didn't — Maven BOM-within-BOM imports flatten the dependency resolution, and the override precedence doesn't propagate through a second level of import. A genuinely surprising Maven behaviour that I hadn't encountered before.

## The Gate

The E2E test itself turned out to be the interesting part. A `@SpringBootTest` that boots the full Spring Boot application with all platform modules on the classpath — context loads, health endpoint responds, Jackson 2 bridge confirmed active. The test is four assertions and took ten minutes to write.

What took longer was discovering which auto-configurations could actually boot. Three of them — REST controllers, callback infrastructure, and MCP — depend on SPI implementations that `PlatformDefaultsManualConfig` doesn't provide. In Quarkus, `DefaultBeans.java` gives you thirty `@DefaultBean` fallbacks. The Spring equivalent only has two. The spring-generator that produces it skipped the simple no-arg factory methods and only generated the configurable ones.

So the test excludes those three auto-configurations and documents the gap explicitly. The next issue in the queue — agent spring auto-configurations — fills it. The test becomes the verification gate: as each auto-config gets its defaults, remove the exclusion, prove it boots.
