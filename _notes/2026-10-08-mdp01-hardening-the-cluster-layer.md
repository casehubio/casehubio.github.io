---
layout: post
title: "Hardening the cluster layer"
date: 2026-10-08
entry_type: note
subtype: diary
projects: [casehubio/qhorus]
tags: [cluster, split-brain, cache, cdi, e2e]
series: issue-488-cluster-hardening
---

The cluster bug fixes from last week (#485–#487) landed the routing and heartbeat corrections, but left gaps in test coverage and resilience that this session addressed.

The most interesting piece was the split-brain fallback. `WriteRoutingDecorator` had been silently falling back to local dispatch when a proxy to the owner node failed — a WARN log was the only signal. In a partition scenario, this creates unreconciled writes on a non-owner node with no programmatic way for consumers to react. We added two mechanisms: a `ProxyFallbackEvent` CDI event that fires on every proxy failure (channelId, owner, sender, message type — enough for a consumer to alert, reconcile, or retry), and a configurable fail-fast mode via `casehub.qhorus.relay.proxy-fallback`. In `local` mode (the default), behaviour is unchanged — the event fires and the fallback proceeds. In `fail` mode, the decorator throws `ProxyDispatchException` after firing the event, so the write doesn't land anywhere. CP vs AP, the consumer's choice.

A detail worth noting: the decorator is a POJO, not a CDI bean — `RelayProducer` constructs it. The CDI `Event<ProxyFallbackEvent>` is injected into the producer and passed through. The localNodeId for the event uses `clusterManager.nodeId()` rather than re-resolving the config, which avoids the subtle divergence where the config might be absent but the cluster manager already resolved the hostname at startup.

The cache module's CDI wiring tests surfaced a Quarkus gotcha. `CacheProducer` used `@IfBuildProperty(enableIfMissing=true)` — cache on by default. To test the disabled path, the `@TestProfile` set `cache.enabled=false`. This crashed with `ConfigValidationException`: the gate excludes all beans injecting `CacheConfig`, which unregisters the `@ConfigMapping`, and SmallRye rejects the now-orphaned property. The general rule about not setting properties under a disabled gate's prefix is well-known, but with `enableIfMissing=true` you face a contradiction — you must set the property to disable the gate, but setting it crashes. The fix was changing to `enableIfMissing=false`, making cache an explicit opt-in like relay. Correct for an optional library module anyway.

The cross-node proxy dispatch E2E test completes the container-level validation: send from node-b to a channel created on node-a, verify via shared PostgreSQL. It's gated behind `-Pwith-e2e-cluster` and Podman — can't run in a standard CI cycle, but confirms the routing fix end-to-end through the actual container stack.
