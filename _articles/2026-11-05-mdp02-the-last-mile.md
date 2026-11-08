---
layout: post
title: "The Last Mile — Why SPIs Without Wiring Are Just Wishful Thinking"
date: 2026-11-05
entry_type: article
subtype: diary
projects: [casehubio/neocortex]
tags: [architecture, cdi, spring, corpus-management, dependency-injection, platform]
series: issue-763-cognitive-knowledge-management
---

# The Last Mile — Why SPIs Without Wiring Are Just Wishful Thinking

Two days ago we proved corpus-management is domain-neutral. Four schemas, one API, zero domain-specific code. The abstraction was clean. The test results were clean. And none of it was usable.

The gap was one line of code that couldn't compile:

```java
@Inject CorpusCatalogue catalogue;
```

Every other cognitive subsystem in neocortex — memory, mindmap, observability, CAPS — has CDI wiring. You add the module to your dependencies, set a config property, and inject the SPI. corpus-management had the SPIs but not the last mile. A Quarkus application couldn't consume it without manually constructing a `SqliteDataSourceFactory`, creating a `HikariDataSource`, passing it to `SqliteCorpusCatalogue`, then passing that to `ChainWalkerFederatedSearch`. Twenty lines of boilerplate that every consumer would duplicate.

For Spring applications, the situation was the same with different annotations.

This is the last mile problem, and it's more common than it should be.

## The pattern: SPIs need safe defaults

The core design principle is that every SPI should be injectable everywhere, always. If a module is on the classpath, its SPIs should resolve — even if the backend isn't configured. The question is what they resolve *to*.

![Bean resolution showing three-tier fallback from SPI to NoOp to real implementation](svg/corpus-bean-resolution.svg)

The three-tier resolution:

1. **SPI** — the interface (`CorpusCatalogue`, `CorpusEntryStore`, `FederatedCorpusSearch`). Lives in `corpus-management-api`. Pure Java. No framework dependency.

2. **NoOp fallback** — a bean that implements the SPI and does nothing. Returns `List.of()`, `Optional.empty()`, `HealthReport.HEALTHY`. This bean exists by default — it's always there unless something displaces it.

3. **Real implementation** — `SqliteCorpusCatalogue`, `SqliteCorpusEntryStore`, `ChainWalkerFederatedSearch`. Created from configuration when the backend is enabled. When present, displaces the NoOp.

The practical consequence: any Quarkus or Spring application that has corpus-management on the classpath can `@Inject CorpusCatalogue` without configuring anything. They get the NoOp. When they're ready to use it for real, they set `casehub.corpus-management.sqlite.enabled=true` and provide a path. No code change — just configuration.

This matters because platform modules compose. The cognition pipeline depends on memory, which depends on mindmap. A test that exercises cognition shouldn't need to configure SQLite paths for every downstream dependency. The NoOps let it focus on what it's testing.

## Why the frameworks differ — and why the implementation doesn't care

The same `SqliteCorpusCatalogue` class serves both Quarkus and Spring. It's plain Java with a constructor that takes a `DataSource`. It has no `@Inject`, no `@ApplicationScoped`, no `@Component`. It doesn't know which framework created it.

![Module architecture showing one implementation serving two frameworks through thin wiring modules](svg/corpus-module-architecture.svg)

The wiring modules are thin — each translates the same pattern into its framework's idiom:

**Quarkus CDI** uses build-time decisions. `@IfBuildProperty` removes beans from the container at build time — they don't exist in the compiled application. `@DefaultBean` marks the NoOp as lowest priority, displaced by any non-default producer. The `CorpusManagementBeans` producer creates all three implementations from a shared `HikariDataSource`, gated by a single build property.

```java
@ApplicationScoped
@IfBuildProperty(name = "casehub.corpus-management.sqlite.enabled",
                 stringValue = "true", enableIfMissing = false)
public class CorpusManagementBeans {

    @Produces @Singleton
    public CorpusCatalogue corpusCatalogue(CorpusManagementConfig config) {
        return new SqliteCorpusCatalogue(dataSource(config));
    }
    // ...
}
```

**Spring Boot** uses runtime conditions. `@ConditionalOnProperty` checks the configuration at startup. `@ConditionalOnMissingBean` registers the NoOp only if no real bean exists. The inner `@Configuration` class groups the conditional beans so the property check applies to all of them at once.

```java
@Configuration
@ConditionalOnProperty(name = "casehub.corpus-management.sqlite.enabled",
                       havingValue = "true")
static class SqliteConfiguration {

    @Bean
    CorpusCatalogue corpusCatalogue(HikariDataSource corpusManagementDataSource) {
        return new SqliteCorpusCatalogue(corpusManagementDataSource);
    }
    // ...
}
```

The difference is when the decision happens — Quarkus at build time, Spring at runtime — but the decision itself is the same: is the backend configured? If yes, create real beans. If no, fall back to NoOps.

Configuration binding follows each framework's convention. Quarkus uses `@ConfigMapping` with an interface hierarchy and SmallRye Config validation. Spring uses `@ConfigurationProperties` with a mutable POJO. Same property names (`casehub.corpus-management.sqlite.path`), different binding mechanisms. An application.properties file works with both.

## The subtle bug: required properties in optional modules

The Quarkus config originally declared `sqlite.path` as a required `String`:

```java
interface SqliteConfig {
    String path();  // no @WithDefault — required
}
```

This works when the beans are active. But `@ConfigMapping` can be validated at startup even when nobody injects it. If corpus-management is on the classpath for its NoOps but the SQLite path isn't set, the application fails to start.

The fix: `Optional<String> path()`. The config mapping accepts a missing value. The producer checks presence and throws a clear error if someone enables SQLite without providing a path:

```java
String path = sqlite.path()
    .orElseThrow(() -> new IllegalStateException(
        "casehub.corpus-management.sqlite.path must be set when sqlite is enabled"));
```

This is the kind of thing that only surfaces in integration — a module that works in isolation but breaks when composed with others that don't need its backend. The NoOp pattern only works if the NoOp path is truly zero-configuration. A required property, even for an inactive backend, violates that.

## Why this matters for platform libraries

Most platform work isn't about algorithms or clever abstractions. It's about this: making the right thing happen when someone adds a dependency and types `@Inject`. The SPI design, the domain-neutral records, the federated search walker — those were the interesting problems. But they were inaccessible until a Quarkus application could write one annotation and get a working bean.

The wiring modules are 100 lines each. They're not clever. They translate known patterns into known frameworks. But without them, the proven-general, tested, domain-neutral corpus-management module is an academic exercise. With them, it's infrastructure.

The lesson applies beyond this project: if you're building a platform library, the last mile isn't the SPI design or the implementation. It's the `@Bean` method, the `@DefaultBean`, the `@ConditionalOnMissingBean` that make your abstraction reachable from a single annotation. Skip the last mile and you've built a library that works but can't be used.
