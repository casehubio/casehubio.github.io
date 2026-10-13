---
title: "Closing the YAML parity gaps in the simulation framework"
date: 2026-09-20
author: mdp
entry_type: note
subtype: diary
tags: [simulation, yaml, corpus-loader, key-extractor, apt]
projects: [casehub-platform]
issue: 370
---

# Closing the YAML parity gaps in the simulation framework

Issue #370 was filed back when the simulation framework still required Java for
things that should have been YAML-declarable: key extractors, exhaustion policies,
thresholds, corpus loading. By the time I got to it, the #352 simulation DX work
had already landed most of that — `YamlSimulationConfig` parses `key-extractor`,
`exhaustion-policy`, and `threshold` natively, and corpus loading (inline and
external files) works out of the box. Three of the four gaps were already closed.

Two real gaps remained, and they're the interesting ones.

## Parameter names as key extractor specs

The `DeclarativeExtractorFactory` already supported `identity`, `field:accountId`,
and `composite:x,y`. But if you had a method like `balance(String accountId)`, the
natural YAML declaration is `key-extractor: accountId` — just the parameter name.
That didn't work. You had to write `key-extractor: identity` or `key-extractor: field:accountId`
depending on whether the input was a primitive or a record.

The fix has two pieces. The APT (`SimulationDecoratorProcessor`) already knew the
parameter names at build time via `method.parameterName(i)` — it just wasn't doing
anything with them. We added a `generateParameterEntries()` method that emits a
`META-INF/simulation-parameters.properties` file alongside the decorator and QN
constants. Format is simple: `spi.method=paramName:0,paramName:1`.

At runtime, a new `ParameterRegistry` loads these properties files from the classpath
(multiple JARs aggregate via `ClassLoader.getResources()`). The factory's new
`create(spec, qualifiedName)` overload tries the existing prefix-based specs first
(`identity`, `field:*`, `composite:*`, `rest-client`), then falls through to the
registry. If the spec matches a parameter name, it creates the right extractor
automatically — identity for single-arg methods, positional array extraction for
multi-arg.

The error message when a bare name doesn't match is deliberate: it lists the valid
parameter names from the registry, so the YAML author gets immediate feedback
instead of a generic "unknown spec" message.

## Format-independent corpus loading

The original `YamlSimulationConfig` had all its corpus file loading buried in
private methods — `loadExternalCorpusFiles()` and `openStream()`. That logic
assumed YAML format, and it wasn't accessible to external consumers like the pages
repo's `ScenarioOrchestrator`, which was importing a `YamlCorpusLoader` class that
didn't exist (it had been absorbed into `YamlSimulationConfig` during the #361
unified YAML work).

The feedback that shaped this design: corpus data shouldn't be tied to YAML. Finance
and clinical domains have their own standards. CSV is the lingua franca for tabular
reference data — transaction lists, account balances, diagnostic results. Coupling
corpus loading to YAML format is a framework assumption that doesn't match the
domain reality.

We extracted a `CorpusLoader` SPI with three implementations: `YamlCorpusLoader`
(`.yaml`/`.yml`), `JsonCorpusLoader` (`.json`), and `CsvCorpusLoader` (`.csv`).
A `CompositeCorpusLoader` dispatches by file extension. `YamlSimulationConfig`
delegates to the composite, and `SimulationConfigBeans` produces it as a CDI bean
for injection by pages.

The CSV convention uses reserved column names — `_qualified_name` for the method
QN, `_key` for the lookup key, `_tenancy_id` for tenant isolation. Everything else
becomes the output map. A finance team can drop their reference CSV into the
simulation corpus pipeline without rewriting it as YAML.

## What this opens up

The immediate consumer is connectors#94 — `BankFeedPlatform` and `EmailPlatform`
SPIs that need full YAML-driven simulation without Java boilerplate. With parameter-name
extractors and CSV corpus support, a connector author writes a `simulation.yaml` that
reads like the SPI's method signatures and references their domain data files directly.

The `CorpusLoader` SPI is also an extension point. Industry-specific formats — HL7
for clinical, FIX for trading — could plug in without touching the framework core.
That's speculative, but the boundary is clean enough that it wouldn't be forced.
