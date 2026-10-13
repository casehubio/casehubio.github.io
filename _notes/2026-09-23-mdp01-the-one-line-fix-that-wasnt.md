---
layout: post
title: "The One-Line Fix That Wasn't"
date: 2026-09-23
entry_type: note
subtype: diary
projects: [casehubio/platform]
tags: [spring-boot, maven, code-generation, auto-configuration]
series: issue-384-spring-deployment-audit
---

# The One-Line Fix That Wasn't

The audit said it was a one-line fix: add `PlatformAutoConfiguration` to the hand-written `AutoConfiguration.imports` file. Twenty-nine NoOp fallback beans would register. Spring Boot would stop crashing on missing SPIs. Simple.

It was not simple.

## The first surprise: thirteen orphans

Adding the line worked for `platform-spring`. But checking the other generated Spring modules revealed that *none of them* had their imports files in the JAR at all. Thirteen auto-configuration classes — governance, identity, expression, all nine agent backends — compiled fine but were invisible to Spring Boot.

The cause was in `AbstractGeneratorMojo.registerSourceRoot()`. It called `project.addCompileSourceRoot()`, which registers a directory for Java compilation. The `META-INF/spring/AutoConfiguration.imports` file sitting alongside the Java sources? Not Java. Not compiled. Not copied. Not in the JAR. Silently absent for months.

The fix was `project.addResource()` with a `META-INF/**` include filter — but the positioning matters. Maven processes resources in list order, and later entries overwrite earlier ones. `addResource()` appends. So the generated imports file *overwrote* the hand-written one in `platform-spring`, eliminating the manual entries I'd just added. Inserting at position zero — `getResources().add(0, resource)` — reverses the priority: generated resources load first, hand-written resources overwrite when they exist.

## The second surprise: mock beans with raw params

With all auto-configs now loading, the integration test crashed immediately. `MockCurrentPrincipal` in the generated `PlatformAutoConfiguration` took raw `String` and `List<String>` params — no `@Value` annotations. The hand-written `PlatformDefaultsManualConfig` had the same bean with proper `@Value` bindings, but Spring was creating the generated version first.

The fix had two parts. A `ManualBeanScanner` now scans the consumer project's `*ManualConfig.java` files, collects their `@Bean` return types, and excludes matching beans from generation entirely. And `@AutoConfigureBefore` on the manual config guarantees it loads before the generated one, so its properly-annotated beans are registered first.

## The third surprise: NoOps that don't yield

Even with the mock beans excluded, the NoOp fallbacks were colliding with JPA implementations. Both `NoOpAccessControlProvider` and `SpringAccessControlProvider` registered — `NoUniqueBeanDefinitionException`.

The root cause is a semantic gap between Quarkus and Spring. Quarkus `@DefaultBean` checks all assignable types: if *any* `AccessControlProvider` exists, the NoOp is suppressed. Spring's bare `@ConditionalOnMissingBean` defaults to the method's return type — `NoOpAccessControlProvider`, not `AccessControlProvider`. The JPA module provides the interface type; the NoOp checks the concrete type. Different types. Both conditions pass.

The generator now resolves the first non-`java.*` interface from the return type's class hierarchy via Jandex and uses it as the explicit `@ConditionalOnMissingBean` value. Combined with `@AutoConfigureOrder(LOWEST_PRECEDENCE)` on generated auto-configs, fallback beans consistently yield to real implementations regardless of loading order.

## One more: initMethod on the wrong class

A smaller bug surfaced along the way. The scanner detected `@PostConstruct` methods on the Quarkus *producer class* and applied `initMethod` to every bean that class produces. `ClaudeAgentBeans` has a `validateBinary()` method — reasonable for the CLI client bean, but `ClaudeAgentProvider` doesn't have that method at all. Spring Boot helpfully told us it couldn't find `validateBinary` on a class that never claimed to have it. Scanning the bean's return type instead of the producer class fixed this.

## What this actually was

The audit described a one-line imports collision. That part was accurate — the hand-written file did shadow the generated one. But registering those twenty-nine beans correctly required fixing the resource pipeline, the bean exclusion logic, the condition semantics, the load ordering, and the init method targeting. Five modules, eight files, four distinct bugs — all hidden because the auto-configs had never actually loaded.

The "one-line fix" turns out to be the most common lie in software: the fix is trivial, *if nothing else is broken*. Something else is always broken.
