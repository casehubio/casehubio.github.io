---
title: "Perspectives and the Ops Manager"
date: 2026-09-21
author: mdp
entry_type: note
subtype: diary
projects: [scaffold]
tags: [perspectives, ops, desiredstate, design, yaml]
series: ops-perspective
status: draft
---

# Perspectives and the Ops Manager

Scaffold has been a generic aggregation app — whatever's on the classpath, it shows tabs for it. Ten views, conditional rendering for ops and IoT, zero custom components. That worked fine as a console. But the question has been nagging: what should scaffold actually *be* when someone deploys it for a specific purpose?

The answer is perspectives. A perspective is a YAML file that tells scaffold what to be — which modules to activate, what UI to render, what datasets to wire in. Not a Maven profile. Not a separate module. Just a declaration. Scaffold reads it at boot and configures itself accordingly.

The first perspective is the ops manager: a scaffold deployment that provisions, deploys, and manages other CaseHub applications. CaseHub managing CaseHub.

## Three kinds of project

The design hinges on a distinction I've been thinking about for a while. CaseHub projects come in three flavours:

**Pure YAML** — nothing but YAML files in a git repo. Case definitions, agent topology, channel wiring, LLM manifests, eidos org structures. No `pom.xml`, no Java, no build step. Scaffold itself is the runtime — it loads the definitions from a mounted volume and runs them.

**Java** — a traditional Quarkus application with its own `pom.xml`, custom SPIs, domain logic. fsitrading is the canonical example.

**Hybrid** — Java project with YAML definitions. Most real apps end up here.

The ops perspective manages all three, but the experience is fundamentally different. A pure-YAML project is fully declarative — the ops perspective can parse every definition and render semantic views: agent topology diagrams, eidos org charts, channel wiring, manifest tables. For Java projects, those definitions are opaque code. You still get runtime observability — health, drift, scaling — but you lose design-time visibility.

That asymmetry is the feature, not a limitation. The pure-YAML path becomes the showcase: deploy an app in minutes with no build step, see every design decision rendered as a live diagram overlaid with runtime state. The Java path is for production systems that need custom logic. Both managed uniformly through DesiredState.

## The investigation

I needed to understand the current state of every relevant repo before designing anything. We fanned out across the platform — desiredstate, ops, engine, eidos, qhorus, claudony, platform (yaml-core), parent docs. The findings reshaped the design in ways I hadn't anticipated.

The biggest one: qhorus is an embedded library, not a standalone service. Each deployed app has its own qhorus instance with its own database. I'd been thinking about shared mesh topology — turns out that's not how it works. Each deployment is self-contained.

Another: the desiredstate plugin model already supports pure-YAML node types — spec schema, actual-state checks, provisioner steps, fault policy, CBR, RAS situations, all in YAML. But nobody has built a real consumer. The plugin model exists only in test fixtures. The ops perspective becomes the forcing function to make pure-YAML projects real.

LLM manifests (`agent-config.yaml`) landed five days before this session. Zero adoption in any example. The ops perspective needs them to work — showing manifest configuration is one of the semantic views. So the examples need updating too.

## Two layers of DesiredState

The spec review caught something I'd glossed over. The ops perspective operates two distinct DesiredState layers:

**Infrastructure** — the ops perspective's own concern. Container topology: app containers, PostgreSQL containers, networks, volumes. A new `ops-container` module implements the DesiredState SPI quad for Podman (dev mode) and reuses the existing K8s infrastructure (production).

**Application** — the deployed app's concern. Agent topology, channels, case types, trust policies. The `ops-deployment` module handles this, running inside the deployed app's process.

The ops perspective owns infrastructure. It observes application state — in-process for shared scaffold instances hosting YAML apps, via REST proxy for dedicated containers and Java apps.

## Case-managed from day one

I decided the ops perspective should run on the engine from the start. Provisioning a deployment is a case. Drift detection triggers situations. Resolutions feed back into CBR across the entire managed fleet. A scaling fix for one deployment informs the next deployment with a similar pattern.

The ops/app module already has case descriptors for every operational concern — `DriftRemediationCaseDescriptor`, `ScalingEventCaseDescriptor`, `IncidentResponseCaseDescriptor`, `ServiceUpgradeCaseDescriptor`. The ops perspective uses these directly rather than reinventing them.

## What's built

The first batch of implementation is done: perspective loading. Scaffold now reads `perspective.yaml` at boot, activates the declared modules, and renders perspective-driven tabs instead of the hardcoded list. The frontend tries `GET /api/perspective` first — if present, builds tabs from a view registry; if absent, falls back to the existing module-detection behaviour. Backward compatible. Six stub ops views (Fleet, Deploy, Health, Drift, Situations, Updates) are wired in. The navigation system is dynamic — tab-to-slug mappings come from the perspective response, not hardcoded entries.

Four batches remain: the Podman provisioning layer (`ops-container` module with the DesiredState SPI quad), fleet management and health polling via `ReconciliationLoop`, case-managed provisioning, and drift detection with manual reconciliation. The plan's review caught that I should use `ReconciliationLoop` rather than custom polling, use Vert.x `WebClient` for Podman's Unix socket API (Java's `HttpClient` doesn't support Unix sockets), and keep all ops domain logic in the ops repo rather than embedding it in scaffold.
