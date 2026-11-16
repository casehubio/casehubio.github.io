---
layout: post
title: "The Spring Generation Contract"
date: 2026-10-04
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [spring, code-generation, architecture, mcpdomain]
series: issue-1206-spring-completeness
---

# The Spring Generation Contract

I'd assumed the Spring story for engine still had massive gaps — the epic listed 7 child issues and 2 blockers. Turns out most of the mechanical work was already done. Five modules had auto-configuration generation live via `spring-generator`. The catch-all `EngineSupportAutoConfiguration` covered 10 more. The real picture was narrower than the issues suggested.

The interesting discovery was about `@McpDomain`. One of the "blocking" issues proposed migrating to JAX-RS annotations on SPI interfaces. But `@McpDomain` with `@PlatformQuery`/`@PlatformMutation` was built specifically to avoid JAX-RS — it abstracts over deployment targets so one annotation set produces REST controllers, GraphQL resolvers, and MCP tool registrations. JAX-RS locks you into one surface. The generator pipeline already reads `@McpDomain`; switching to JAX-RS would break it.

The actual blocker turned out to be structural, not annotation-model. The `@McpDomain` annotations sit on concrete CDI classes in the `rest` module, but `graphql-spring-generator` scans `engine-api` for interfaces. Moving the contract to SPI interfaces in `api` is the real work — and it's cleaner architecture regardless of generation.

That led to what became the central design decision: all Spring bridging code must be generated. No hand-written `@Bean` definitions, no hand-written event adapters, no hand-written controllers. The generators are the single source of truth; the Quarkus codebase is the authority. Hand-written Spring code drifts — we've seen it with the 37 `*SpringAdapter.java` files across four modules, where `@Async` placement was inconsistent with no correlation to the Quarkus source patterns.

There are honest exceptions. `persistence-spring-jpa` has 12 repositories using Spring Data JPA with JPQL and tenant context management — the Quarkus side uses Panache, a fundamentally different API. No shared source to generate from. `RuntimeManualConfig.java` is 771 lines of CDI `Instance<T>` stubs that have no Quarkus counterpart. These stay hand-written, each with documented rationale.

One gotcha surfaced during the first SPI extraction. I used `java.util.concurrent.Flow.Publisher` as the return type for SSE methods on the interface — framework-neutral, JDK built-in, implemented by both Mutiny `Multi<T>` and Reactor `Flux<T>`. The Quarkus annotation processor generated a JAX-RS resource from it, but emitted `import java.util.concurrent.Flow$Publisher` — the JVM internal name for nested classes. The compiler saw `Publisher` as an unresolved symbol. The fix was to use `Multi<T>` directly, which is a top-level class the generator handles correctly. The nested-class import bug is a Jandex code generation issue that'll hit anyone using `Flow.Publisher` with annotation processors that generate source from bytecode metadata.

The generation mandate is now a standing protocol. If a `-spring` module has a hand-written `@Bean`, `@EventListener`, `@RestController`, or `@ControllerAdvice` without a documented exception, it's a violation.
