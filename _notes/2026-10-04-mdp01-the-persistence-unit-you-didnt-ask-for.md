---
layout: post
title: "The Persistence Unit You Didn't Ask For"
date: 2026-10-04
entry_type: note
subtype: diary
projects: [casehubio/soc]
tags: [quarkus, hibernate, cdi, multi-pu, upstream-restructuring]
---

# The Persistence Unit You Didn't Ask For

The engine, ledger, and qhorus repos all restructured their packages in the same wave. SOC wouldn't compile. The fix looked mechanical — rename imports, update `application.properties`, move on. It wasn't.

The first layer was straightforward: `io.casehub.persistence.memory` became `io.casehub.engine.persistence.memory`, `engine.internal` became `engine.runtime`, the `work-engine-adapter` artifact got a new name. Standard dependency churn. But each rename cascaded through three different config systems — CDI bean discovery, Hibernate entity scanning, and Flyway migration routing — and they interact in ways that aren't obvious until something breaks silently.

The real discovery was about Quarkus persistence units. SOC runs three datasources: default (work tables), qhorus (channel/messaging tables), and memory (neocortex). The Hibernate `packages` config assigns entities to PUs. Everything looks clean on paper. But every qhorus JPA store — `JpaChannelStore`, `JpaCrossTenantChannelStore`, all of them — injects `@Inject EntityManager em` without a `@PersistenceUnit` qualifier. That's the default PU. The entities are in the qhorus PU. The EntityManager can't see them.

This was invisible in the old layout because all qhorus entities lived directly in `io.casehub.qhorus.runtime`. When they moved to subpackages (`runtime.channel`, `runtime.data`, `runtime.message`, six more), we had to list each one explicitly in the Hibernate config — which made the PU assignment visible, which made the mismatch visible.

`UnknownEntityException: Could not resolve root entity 'Channel'`. The entity exists, it's indexed, it's annotated. But it belongs to a PU that the querying EntityManager has no access to. Quarkus doesn't route queries based on entity metadata. Each EntityManager is locked to one PU. If you don't qualify the injection, you get the default. Always.

The workaround was mock stores via `@Alternative @Priority` that displace the broken JPA implementations. SOC doesn't need cross-tenant channel queries — it just needs the beans to satisfy CDI injection for the qhorus services that do. The real fix belongs upstream: every qhorus store needs `@PersistenceUnit("qhorus")` on its EntityManager.

A second trap: the test `application.properties` had its own `quarkus.arc.exclude-types` to exclude one bean. SmallRye Config doesn't merge list properties across config sources — the test value replaced the production value entirely. Every CDI exclusion from the main config silently disappeared. Beans that should have been gone came back and caused ambiguity errors with no indication that a replacement had occurred.

The branch landed with 521 tests passing (22 skipped, 1 pre-existing failure disabled). The mechanical fixes were ten lines each. The diagnostic work to understand *why* they were needed was the actual session.
