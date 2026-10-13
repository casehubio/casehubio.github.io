---
title: "Teaching Workers Who They Are"
date: 2026-08-04
author: mdp
entry_type: note
subtype: diary
phase: "Slice 1, Layer 4a"
projects: [casehub-soc]
tags: [casehub, eidos, agent-descriptors, health-probing, mitre-attck]
---

# Teaching Workers Who They Are

Until now, the SOC's six workers were anonymous. The engine knew their names and which capability each served, but nothing about their operational character — no epistemic domains, no disposition profile, no way to distinguish a deterministic rule-based extractor from an LLM reasoning over the same threat data. `AgentCandidateFactory` skipped health probing entirely and stamped every worker READY.

Agent descriptors change that. Each worker now carries an `AgentDescriptor` from eidos — a structured identity that declares what it can do, how confident it is across different threat contexts, and how it operates. The rule-based IOC enrichment worker declares high confidence for Initial Access, Execution, Command and Control, and Exfiltration — the four MITRE ATT&CK tactics where IOC types (IP addresses, file hashes, domains, URLs) are primary indicators. The LLM equivalent declares 0.9 confidence across all fourteen tactics. It's a general-purpose reasoner; trust scoring will learn the real distribution from outcomes.

## Two Paths, One Source

The design had an architectural wrinkle that took some tracing to find. Eidos and the engine consume descriptors through completely independent paths with no auto-bridging.

Path 1: `AgentDescriptorRegistrar` is an eidos SPI. Implement it, and `AgentDescriptorBootstrap` discovers your bean at startup and registers descriptors in the `AgentRegistry`. This is the eidos identity layer — persistent agent records, registry queries, discovery.

Path 2: `CaseDefinition.setAgentDescriptors()` is an engine API. Wire descriptors into the case definition, and `AgentCandidateFactory` probes `CapabilityHealth` before building candidates. This is the health probing layer — real-time availability, epistemic weakness, degradation.

Neither knows about the other. The `YamlCaseHub.augment()` javadoc hints at it — "programmatic workers, agent descriptors, or other modifications" — but nothing in the engine references `AgentRegistry`. The bridge is the application's responsibility. `SocAgentDescriptors` in api/ is the single source of truth; both `SocAgentRegistrar` (CDI bean for eidos) and `SocCaseHub.augment()` (engine wiring) draw from it.

## The Dependency That Didn't Exist

The spec originally said "no other eidos artifacts needed — eidos-runtime is already on the classpath via casehub-engine." Wrong. The engine depends on `casehub-eidos-api` only. Without `casehub-eidos` (runtime) on the classpath, `AgentDescriptorBootstrap` doesn't exist — the registrar is never collected — and `NoOpCapabilityHealth` handles all probes by returning READY unconditionally. Both registration paths would have been completely inert.

The design review caught this before any code was written. Three independent reviewers flagged it from different angles — coherence found the missing class reference, structure found the missing validation chain, robustness found the no-op fallback. The cross-cutting review connected all three into one finding: the entire feature's runtime path was missing.

Adding `casehub-eidos` (runtime) then triggered a CDI ambiguity — `DefaultCapabilityHealth` (eidos, `@DefaultBean`) competing with `NoOpCapabilityHealth` (engine, also `@DefaultBean`). The fix is a one-liner in `application.properties`, but the eidos runtime artifact ID itself was a trap: the module directory is `runtime/` but the Maven artifact is `casehub-eidos`, not `casehub-eidos-runtime`. The engine follows the same pattern — `runtime/` directory, `casehub-engine` artifact — but you don't discover this until dependency resolution fails and you read the POM.

## Disposition as Data

The descriptors include more than capability metadata. Each worker carries a disposition profile — rule-following, autonomy, risk appetite, delegation willingness. Rule-based workers are `strict/none/averse/false`. LLM workers are `moderate/guided/moderate/true`. This isn't cosmetic. When trust-weighted routing arrives, disposition gives the routing strategy structural data about operational character before any attestation history accumulates. It addresses the cold-start problem the Layer 2 design review identified: in Bootstrap phase, the LLM worker never fires because insertion order always selects the rule-based one. Disposition won't solve that alone — but it gives future routing policies something to work with beyond a coin flip.

The epistemic domain model uses MITRE ATT&CK tactics directly. The `AttackTactic` enum was already in the codebase; the descriptors just point at it. `DefaultCapabilityHealth` in eidos-runtime evaluates these domains during probing — though the engine currently passes a case UUID as the "task domain" in `ProbeContext`, which means domain-based discrimination won't activate until the engine enriches the context with actual threat metadata. The descriptors are correct to model now; the plumbing catches up later.
