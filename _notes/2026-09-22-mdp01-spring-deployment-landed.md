---
layout: post
title: "CaseHub Runs on Spring Boot"
date: 2026-09-22
entry_type: note
subtype: diary
projects: [casehubio/platform]
tags: [spring-boot, code-generation, core-extraction, deployment, epic-close]
series: issue-501-spring-deployment-readiness
---

Fourteen issues, five repos, and one question that started it all: can a platform built entirely on Quarkus CDI also deploy on Spring Boot without maintaining two codebases?

The answer is yes, and the branch that proves it just closed.

## The bet

The [prior entry](2026-09-14-mdp01-spring-generators-epic-shrank.md) covered the generator infrastructure — three Maven plugins that scan Quarkus source via Jandex and emit Spring equivalents. That work proved the generators could produce correct output. This epic proved they could produce a *complete* deployment.

The audit that kicked things off surveyed seven consumer repos. Platform had 9 Spring modules and 37 core modules already extracted. Engine, neocortex, qhorus, and work had gaps ranging from "small" to "significant" — work alone needed 28 REST controllers generated. The epic's job was to close every gap until a Spring Boot app could compose the full platform stack and start cleanly.

## What "core extraction" actually means

Every CDI-coupled module now has a `-core` counterpart. The core module contains a plain Java class with constructor injection — no CDI annotations, no Spring annotations, no framework imports at all. The Quarkus module retains its existing artifact name and delegates to the core POJO via `@Produces`. The Spring auto-configuration does the same via `@Bean`.

The translation table is small enough to memorise:

| Quarkus | Core POJO | Spring |
|---------|-----------|--------|
| `@Produces @DefaultBean` | Constructor-injected POJO | `@Bean @ConditionalOnMissingBean` |
| `Event<T>` | `Consumer<T>` callback | `ApplicationEventPublisher` |
| `Instance<T>` | `List<T>` constructor param | `ObjectProvider<T>` |

Three patterns cover the entire platform. The spring-generator handles the CDI bridging automatically — it sees `Event<T>` in a Quarkus beans class and emits a `Consumer<T>` lambda backed by `ApplicationEventPublisher`. `Instance<T>` becomes `ObjectProvider<T>` collected into a list. The translation is mechanical, which is exactly the point: no judgment calls means no drift.

## The generators matured under load

The prior epic built the generators. This epic stress-tested them against real modules. Three improvements came out of that pressure:

**Constructor scanning.** The original generator assumed `@Produces` methods were the unit of work. Real modules use constructor injection extensively — a core POJO with six parameters needs the generator to find the right constructor, resolve each parameter's type, and wire the Spring equivalents. Claude added `JandexProducerScanner` constructor resolution that walks the core class's constructors and maps each parameter type to its Spring counterpart.

**CDI bridging.** `Event<T>` and `Instance<T>` appear everywhere in CaseHub's event-driven architecture. The generator now recognises four bridging patterns: `EVENT_CONSUMER`, `SUPPLIER_DEP`, `LIST`, and `OPTIONAL`. Each maps a CDI idiom to its Spring equivalent without the generated code knowing which framework it targets.

**Exclusion control.** Not everything can be generated. SSE endpoints (`A2AResource`), multipart uploads (`ComplianceReportResource`), and a few others need hand-written Spring controllers. The `excludeClassNames` parameter lets each consumer repo list the classes the generator should skip.

## The composition gate

The `spring-integration-test` module is the proof. A `@SpringBootTest` that composes all nine JPA auto-configurations, verifies the health endpoint returns UP, and confirms Jackson serialisation is active. It starts on H2 with Hibernate DDL — no PostgreSQL, no external dependencies. If this test passes, the platform stack composes correctly under Spring Boot.

A few auto-configurations are excluded from the test — REST controllers, callback infrastructure, and OIDC — because they need request-scoped context or security infrastructure the test doesn't provide. Those are verified in their own module tests.

## Streams got extracted too

The four stream processor types (Kafka, AMQP, Camel, poll) each got the core extraction treatment. `StreamCloudEventFactory` became a shared utility — the CloudEvent construction logic that was duplicated across four Quarkus processors now lives in `streams-core` and is testable without any messaging infrastructure. Each protocol's core module is pure Java with protocol-specific libraries but no CDI.

The Spring adapters use Spring Cloud Stream for Kafka and AMQP, Camel's Spring Boot starter for Camel routes, and a `@Scheduled` poller for HTTP endpoints. Same four protocols, same event construction, different wiring.

## What this opens up

The immediate use case is deployment flexibility — organisations that standardise on Spring Boot can adopt CaseHub without a Quarkus runtime. But the deeper value is in the core extraction itself. Every module now has a framework-neutral layer that can be tested, composed, and reasoned about without any container. That's 30+ modules where the business logic is separated from the framework plumbing.

The next question is whether the starter POM (`spring-boot-starter`) gives Spring consumers a good enough developer experience, or whether they need a Spring Initializr integration that scaffolds a project with the right modules pre-selected. That's a different kind of work — DX, not infrastructure — and it depends on how the first real Spring deployments go.
