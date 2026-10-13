---
layout: post
title: "Observability That Doesn't Pick Sides"
date: 2026-09-23
entry_type: note
subtype: diary
projects: [casehubio/platform]
tags: [spring-boot, quarkus, micrometer, observability, health-indicators, actuator]
series: issue-384-spring-deployment-audit
---

# Observability That Doesn't Pick Sides

The audit flagged zero Actuator integration as a dim 8 finding. No health indicators, no metrics, no info contributor. A Spring Boot app using the platform starters would report `UP` at `/actuator/health` — but that `UP` meant "Spring Boot is alive," not "the platform is healthy."

The obvious fix: create `platform-spring-actuator` with five `HealthIndicator` beans and some `MeterBinder` gauges. But the obvious fix has a design flaw.

## The vendor-neutral question

Metrics instrumentation needs to intercept SPI calls — counting agent invocations, timing ACL checks. The interception mechanism differs by framework: Spring uses `BeanPostProcessor`, Quarkus uses CDI `@Decorator`. But the *recording logic* — "start a timer, increment a counter, tag with the backend key" — is identical.

If we put everything in `platform-spring-actuator`, Quarkus gets nothing. Both frameworks already use Micrometer. The instrumented wrappers are plain POJOs — `new InstrumentedAgentBackend(delegate, meterRegistry)`. No framework coupling. Both sides just instantiate the same wrapper using their own decoration mechanism.

Three modules:

```
platform-observability-core     ← vendor-neutral: Micrometer wrappers + gauge binder
     ↙               ↘
platform-observability    platform-spring-actuator
(Quarkus @Decorator)      (Spring BeanPostProcessor)
```

## The reactive timer trap

`AgentProvider.invoke()` returns `Multi<AgentEvent>` — a cold reactive stream. Wrapping the method call with `Timer.start()` and `sample.stop()` measures how long it takes to *create* the `Multi` object: roughly zero. The actual work happens on subscription.

The fix uses Mutiny's `deferred` + `onTermination`:

```java
return Multi.createFrom().deferred(() -> {
    Timer.Sample sample = Timer.start();
    return delegate.invoke(config)
            .onTermination().invoke(() -> sample.stop(timer));
});
```

`deferred` delays the lambda until subscription. `onTermination` fires on completion, failure, *and* cancellation — all three paths stop the timer. The counter increment goes inside `deferred` too, so it counts actual invocations, not `Multi` object creations.

## The backend key problem

The initial design instrumented `AgentProvider` — but `AgentProvider` callers pass a model string like `"claude-sonnet-5"` or `"tier:FLAGSHIP"`. The actual backend key (`"claude"`, `"openai"`) is resolved internally by `RoutingAgentProvider`. An instrumented wrapper sitting outside the router can't extract it without duplicating the resolution logic.

Moving instrumentation to `AgentBackend` solved it cleanly. Each backend has `key()` — the tag is right there. In Spring, the `MetricsBeanPostProcessor` wraps each `AgentBackend` bean individually. In Quarkus, the `@Decorator` wraps each backend instance through CDI's per-delegate semantics.

## Keystore without DSS

The certificate expiry health indicator needed access to keystore metadata. The existing `KeyStoreManager` in `platform-signing` uses EU DSS's `Pkcs12SignatureToken` — which pulls in the entire DSS dependency tree. But checking certificate expiry is a standard JDK operation: `java.security.KeyStore` + `X509Certificate.getNotAfter()`. About thirty lines of code, zero external dependencies.

`KeyStoreExpiryChecker` in `observability-core` reads the same keystore file with JDK APIs. It's not duplicating `KeyStoreManager` — different purpose (read-only metadata inspection vs signing operations), different dependency profile (pure JDK vs DSS).

## What it opens up

The `casehub.platform.*` metric namespace is now live. An operator deploying a CaseHub Spring Boot app with actuator gets model registry health, agent backend discovery, delivery channel status, SCIM reachability, and certificate expiry warnings — without adding any configuration. The same indicators appear in Quarkus via MicroProfile Health.

The invocation metrics (`casehub.platform.agent.invocations`, `casehub.platform.acl.can_access`) are the first operational counters in the platform. They're the foundation for dashboards that answer "how often is each backend called?" and "what's the ACL check latency?" — questions that were previously invisible.
