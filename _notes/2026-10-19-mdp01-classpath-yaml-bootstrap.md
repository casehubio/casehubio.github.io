---
layout: post
title: "Teaching CaseHub to Find Its Own Configuration"
date: 2026-10-19
entry_type: note
subtype: diary
projects: [casehubio/engine, casehubio/parent, casehubio/casehub-ops]
tags: [bootstrap, spi, classpath-scanning, yaml, quarkus]
---

# Teaching CaseHub to Find Its Own Configuration

A CaseHub application has always needed Java code to wire itself up — CDI beans extending `CaseHub`, programmatic agent configuration, situation definitions built from lambda expressions. That works for developers who know the platform, but it's a steep on-ramp. The simpler path is YAML files dropped into a classpath directory: a case definition in `casehub/my-case.yaml`, an agent definition in `agents/planner.yaml`, and the platform figures out the rest.

The design has three layers. A parent module (`casehub-platform-bootstrap`) owns the SPI and scanner. Engine registrars consume the SPI for cases, agents, and situations. An ops registrar handles topology. Each layer knows nothing about the others — the scanner discovers registrars through CDI, sorts them by priority, and dispatches YAML files to whoever claimed that section key.

## The SPI

`ClasspathYamlRegistrar` is the contract. Four methods: `sectionKey()` (which YAML section you handle), `classpathPaths()` (where to look on the classpath), `priority()` (startup ordering), and `register(resourcePath, content)` (here's a file, do something with it). The scanner, `ClasspathBootstrap`, collects all registrar beans at startup, sorts by priority, and for each one scans the declared classpath paths for `.yaml`/`.yml` files.

The interesting constraint was startup ordering. `DefaultCaseDefinitionRegistry` discovers CDI `CaseHub` beans at priority 10. The YAML registrars run later — `AgentYamlRegistrar` at 100, `CaseYamlRegistrar` at 200, `SituationYamlRegistrar` at 300. A separate `CaseHubClassRegistrar` uses Jandex to find non-CDI `CaseHub` subclasses at priority 150 — classes that extend `CaseHub` but aren't managed by Arc. It instantiates them via reflection, extracts their `getDefinition()`, and registers the result. This covers the edge case of CaseHub subclasses packaged in library JARs outside the CDI bean archive.

## The Situation Problem

Cases and agents map cleanly to YAML — flat records with string fields. Situations are harder. A `SituationDefinition` has `ExpressionEvaluator` fields (correlation key expressions, event filters), sealed-interface chain modes (`And`, `Or`, `Threshold`, `Sequence`), and sealed trigger actions. Hand-rolling a YAML-to-record mapper would have been tedious and fragile.

The shortcut: all the ras-api types already carry Jackson `@JsonTypeInfo` and `@JsonSubTypes` annotations, and there's an existing `ExpressionEvaluatorDeserializer` that converts YAML strings to JQ expressions. So `SituationYamlRegistrar` uses a Jackson `ObjectMapper` with `YAMLFactory`, `JavaTimeModule`, and the expression deserializer — and the YAML-to-`SituationDefinition` mapping comes free. Expressions in YAML are parsed as JQ by default; the `{mvel: "..."}` syntax is supported if an expression engine registry is available.

One design decision worth naming: `SituationRegistrar` might not be on the classpath if the ras runtime module isn't deployed. The registrar injects it as `Instance<SituationRegistrar>` and checks `.isResolvable()` — if absent, it logs the discovery and moves on. The YAML file is still validated by deserialization; it just can't be registered until the ras runtime appears.

## The Production Bug That Wasn't

The audit caught something I'd missed entirely. `ClasspathBootstrap.discoverYamlFiles()` only handled `file:` protocol URLs — fine in Quarkus dev mode where classes are exploded directories, but in production (fast-jar or uber-jar), resources live inside JARs. The method silently returned an empty list for `jar:` URLs, meaning the entire bootstrap system would discover zero YAML files in any production deployment.

The fix was a `JarURLConnection` handler that enumerates JAR entries matching the base path. Straightforward once spotted, but the kind of thing that would have surfaced as a mysterious "my YAML files aren't loading" report weeks later.
