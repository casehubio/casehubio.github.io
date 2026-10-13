---
title: "Wiring the Ops Perspective"
date: 2026-10-06
author: mdp
entry_type: note
subtype: diary
projects: [scaffold]
tags: [perspectives, ops, mcpdomain, graphql, fleet, naming]
series: ops-perspective
status: draft
---

# Wiring the Ops Perspective

The perspective infrastructure landed two weeks ago — scaffold reads a YAML file at boot, activates modules, renders tabs. But the tabs were stubs. Fleet showed a placeholder table. Deploy was an empty heading. The data layer pointed at endpoints that didn't exist.

Today we connected it to real ops data. The short version: add `casehub-ops-service` as a Maven dependency, update the perspective YAML datasets, point the frontend views at actual entity fields. The long version involved three repos, two module extractions, and a naming debate I didn't expect.

## The module boundary problem

The ops repo's `@McpDomain` API classes — `OpsApplicationApi`, `OpsDeploymentApi`, six more — lived in `ops/app`, a Quarkus application with its own main class and config. Scaffold can't depend on another Quarkus app. The service layer needed to be in a library module.

That meant extracting `ops/service` — all the API classes, JPA entities, CDI services, K8s implementations, case descriptors, and Flyway migrations — into a standalone jar that both ops/app and scaffold could consume. The extraction landed as casehub-ops#118. Once that was in, enabling GraphQL generation (which ops had explicitly disabled with `-AgenerateGraphQL=false`) was casehub-ops#119.

Both prerequisites had to land before scaffold could add a single Maven dependency.

## The engine broke everyone

When we finally added the dependency and built, Quarkus CDI resolution failed. Not because of ops — the engine itself was broken. New subsystems (stigmergy coordination, convergence detection, improvement/evolution) had been added with unconditional producer methods that required beans only available in the full engine runtime. Thirteen unsatisfied dependencies across four producer classes.

The same build broke on main without our changes. Every consumer of `casehub-engine` was affected. We filed casehubio/engine#1218 with the full dependency map and it was fixed within the session.

A second issue surfaced: the `casehub-work-engine-adapter` artifact in scaffold's POM was 30 days stale. The engine had renamed the module to `casehub-engine-work-adapter` and moved `EngineStrategyResolver` from `engine.internal.routing` to `engine.runtime.routing`. The old artifact was still in the local Maven cache, compiled against the pre-rename package. Fixing it was a one-line POM change.

## What "fleet" means

I expected the naming to be obvious. It wasn't. The first instinct was "Applications" — that's what ArgoCD calls its top-level entity, what Humanitec uses, what Backstage organises around. Industry consensus.

But the ops perspective doesn't manage a collection of applications. A database isn't an application. A message broker isn't an application. An agentic mesh isn't an application. The ops perspective manages heterogeneous deployed systems — each one internally composed of containers, databases, agents, networking.

Meanwhile, Claudony already uses "fleet" extensively — 22 classes across two packages for agent session pool management. And Rancher's Fleet product handles multi-cluster GitOps distribution. Three different uses of the same word.

I landed on keeping "Fleet" for the ops perspective tab. The insight: a fleet is a managed collection of instances of the same kind of thing. Claudony manages a fleet of agent sessions. Rancher manages a fleet of K8s clusters. The ops perspective manages a fleet of deployed CaseHub systems. Each fleet manager owns the infrastructure for its kind of thing. The word isn't contested — the scope is different.

## The wiring

With the module boundary sorted and the engine fixed, the actual wiring was straightforward. The `@McpDomain` annotation processor generates REST resources, GraphQL resolvers, and MCP tool definitions — all pre-packaged in the ops-service jar. Adding it to scaffold's classpath exposes eight API domains automatically:

- **Fleet** tab → `OpsApplicationApi` (register, list, manage)
- **Deploy** tab → `OpsDeploymentApi` + `OpsApprovalApi` (plan, approve, execute, rollback)
- **Health** tab → `OpsClusterApi` (cluster connectivity, node counts)
- **Drift** tab → `OpsReconciliationApi` (drift detection, reconciliation triggers)
- **Situations** tab → `OpsCaseApi` (incidents, changes, problems)
- **Updates** tab → `OpsDeploymentApi` (version tracking, rollback history)

The frontend views bind to REST datasets from the perspective YAML. GraphQL resolvers are also available at `/graphql` for future queries that need field selection or relationship traversal. The pages-ui data layer doesn't have a native GraphQL source yet, so the dataset bindings use REST — but the GraphQL schema is there when we need it.

## What's next

The perspective tabs render and the data layer is wired, but every tab shows an empty table. The ops service has the API classes and JPA entities, but no data exists yet — no applications registered, no clusters configured, no deployments recorded. The next step is standing up a real deployment through the ops perspective: register an application, configure a cluster, deploy it via DesiredState, watch the reconciliation loop converge.

That's when the ops perspective stops being infrastructure and starts being a product.
