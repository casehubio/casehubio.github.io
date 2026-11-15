---
layout: post
title: "Scaffold Becomes a Platform"
date: 2026-10-18
author: mdp
entry_type: note
subtype: diary
projects: [casehubio/scaffold]
tags: [scaffold, platform, app-descriptor, CDI, service-mediation, topology, SSE]
series: issue-59-ops-console-platform
status: draft
---

# Scaffold Becomes a Platform

A month ago I wrote about perspectives — YAML descriptors that tell scaffold what to be. The ops manager was the first perspective: a scaffold deployment that manages other CaseHub applications. That entry ended with a working parser and a REST endpoint. The shape was right but the name wasn't, and everything beneath it was hardwired.

This entry is about what happened when we took that idea seriously. Scaffold is no longer a dashboard that happens to read a config file. It's a platform — a host for CaseHub apps, where the ops manager is the first tenant.

## The rename that wasn't cosmetic

"Perspective" was the wrong word. A perspective implies a view onto something that already exists. What scaffold actually does is host an application — it provides the runtime, the service mediation, the UI shell, and the event infrastructure. The YAML file isn't a perspective on a system. It's the definition of an app.

So `PerspectiveConfig` became `AppConfig`. `PerspectiveLoader` became `AppLoader`. `perspective.yaml` became `app.yaml`. The root key changed from `perspective:` to `app:`. Every Java class, every TypeScript type, every test string, every REST path. IntelliJ's refactoring handled the cross-cutting rename; the YAML key and REST path changes were manual.

This sounds mechanical — and it mostly was. But naming matters. "App" carries the right connotation: scaffold hosts apps. Each app declares its modules, views, datasets, and connections. The ops manager is an app. A compliance dashboard would be a different app. An IoT monitoring console, another. Same scaffold runtime, different `app.yaml`.

## The service mediation layer

The harder problem was decoupling scaffold from ops. Before this work, scaffold had a compile dependency on `casehub-ops-service`. Every ops bean was directly injectable. That works when scaffold *is* the ops dashboard, but it breaks the moment you want scaffold to host anything else.

The solution is a service mediation layer with a CDI dual-implementation pattern:

```java
// Interface in io.casehub.flow.service
public interface TopologyService {
    TopologySnapshot getSnapshot();
}

// Embedded impl — activated when ops-service is on the classpath
@ApplicationScoped @Alternative @Priority(100)
public class EmbeddedTopologyService implements TopologyService { ... }

// Client impl — fallback when running distributed
@ApplicationScoped @DefaultBean
public class ClientTopologyService implements TopologyService { ... }
```

`@Alternative @Priority(100)` wins when the embedded implementation is available as a CDI bean. `@DefaultBean` provides the fallback — a REST client that calls the service remotely. The consumer code injects `TopologyService` and doesn't know which one it got.

Three service interfaces follow this pattern: `TopologyService` for the live service graph, `DeploymentService` for provisioning and deploy actions, and `ReconciliationService` for drift detection and remediation loops. Each has its own value types — `TopologySnapshot` with nested `LoopEntry` and `NodeEntry`, `DeploymentStatus`, `DriftReport` with `DriftEntry`. The embedded implementations are stubs for now; they'll delegate to real ops beans once the cross-repo work lands.

Making `casehub-ops-service` an optional Maven dependency was the final step. Scaffold compiles and runs without it. The module probe in `ModuleRegistryResource` uses `Class.forName` reflectively, so no compile-time coupling remains.

## Live topology with SSE

The topology graph is the flagship feature — a live visualisation of the service mesh, coloured by reconciliation status. The backend is a JAX-RS endpoint that observes CDI `CloudEvent` events and streams them via Server-Sent Events:

```java
@Path("/api/ops/reconciliation/events")
public class ReconciliationEventResource {
    void onCloudEvent(@ObservesAsync CloudEvent event) {
        // filter for desiredstate events, broadcast to SSE subscribers
    }
}
```

The frontend connects with a plain `EventSource` and dispatches parsed events onto the pages event bus. The `<casehub-topology-graph>` Lit component listens for `reconciliation-event` and updates node status colours in real-time — green for converged, amber for drifted, red for faulted. Layout is handled by ELK's layered algorithm, which computes node positions from dependency edges.

The `app.yaml` descriptor now has a `connections:` section alongside `datasets:` — datasets are REST-polled, connections are SSE push. The ops app's first connection is `reconciliation-events`, pointing at the SSE endpoint.

## What this opens up

Scaffold can now host any CaseHub app that provides an `app.yaml`. The contract is clear: declare your modules, views, datasets, and connections. Scaffold handles activation, service mediation, and the UI shell. The ops manager is the proof that the contract works — it declares seven tabs, seven datasets, one SSE connection, and three optional modules, all resolved at boot.

The service interface pattern is the part I'm most interested in watching evolve. Right now the embedded and client implementations are stubs. When the cross-repo work in `casehub-ops` delivers the topology compiler and deployment target annotations, the embedded impls will delegate to real beans — and scaffold won't change at all. That's the test of whether the abstraction is right.
