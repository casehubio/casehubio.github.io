---
title: "Whose rules are these anyway?"
date: 2026-09-03
author: mdp
entry_type: note
subtype: diary
projects: [casehub-neocortex]
tags: [mindmap, identity, principal, rules, cognitive-architecture]
---

# Whose rules are these anyway?

The MindMap's derived edge rules had a blind spot. When agent A added an edge to the graph, every agent's rules fired — not just A's. The `DerivedEdgeDecorator` loaded all declarative rules at construction time from `DeclarativeRuleRegistry.allDerivedEdgeRules()`, flattened them into a single list, and ran them on every `addEdge` call. No way to know who was calling.

This mattered because different cognitive profiles carry different rules. An agent with a CONNECTIVE inference style should derive different transitive edges than one with CATEGORICAL. The `DeclarativeRuleRegistry` already supported per-agent lookup via `derivedEdgeRules(agentId)` — the wiring just wasn't there.

## The identity question

The fix seemed straightforward — add `agentId` to the input types and pass it through. But this surfaced a harder question: what do we call the thing that owns memory, rules, and perspectives when that thing might be a human or an agent?

`agentId` is too narrow. `userId` is too narrow the other way. We needed a term that covers both — the entity whose cognitive rules fire, whose memory is private, whose perspective overlays the shared graph.

I asked around. The answer landed on **Principal** — the standard identity term from security and authentication. "An entity whose identity can be verified." It covers humans and agents naturally, aligns with Quarkus's `SecurityIdentity`, and reads well: "the principal's rules," "the principal's memory."

From that, a three-layer hierarchy:
- **Principal** — who they are (stable identity, owns things)
- **Actor** — who is acting right now (principal in context, handles delegation)
- **Participant** — who is involved in this interaction (actor in a session)

The cognitive subsystem only needs the top layer. Actor and Participant are runtime concerns that platform owns.

## The implementation

`principalId` went onto `EdgeInput` and `NodeInput` as a nullable String. Backward-compatible constructors mean no existing code breaks — the 100+ raw constructor calls across the codebase compile unchanged. New code chains `.withPrincipalId("agent-1")`.

The decorators changed from loading all rules at construction time to resolving rules per-operation:

```java
private List<DerivedEdgeRule> resolveRules(String principalId) {
    if (registry == null) return programmaticRules;
    List<DerivedEdgeRule> resolved = new ArrayList<>(programmaticRules);
    if (principalId != null) {
        resolved.addAll(registry.derivedEdgeRules(principalId));
    } else {
        resolved.addAll(registry.allDerivedEdgeRules());
    }
    return resolved;
}
```

Programmatic rules (CDI beans) always fire — they're global infrastructure. Declarative rules (from YAML cognitive profiles) scope per-principal when one is provided, fall back to all rules when null. The null fallback preserves backward compatibility: every existing caller that doesn't set principalId gets exactly the same behaviour as before.

Three tests pin the contract: matching principal fires agent-specific rules, different principal doesn't, null fires all.

## The bonus fix

Along the way, we found a pre-existing test failure in `discoverTenants`. The Qdrant implementation only scanned registered schemas when enumerating tenants, but the test stored cases under an unregistered caseType. The tenant data existed in Qdrant but was invisible to the discovery scan. Fixed by using the registered schema in the test — the test was about tenant boundary tracking, not schema handling.

Tenants as "physical separation between worlds" is the right mental model. A tenant is an organisation-level isolation boundary. Nothing crosses it except admin operations.

## What this opens up

The formal `Principal` type landed in platform (#271) during this session. Neocortex uses `String principalId` for now — the integration to `Principal.id()` is mechanical and tracked as #276. Once wired, every cognitive subsystem operation can answer "whose perspective is this?" without thread-locals, CDI request scopes, or invisible coupling.

The deeper implication: perspectival overlays, memory ownership, and rule scoping all converge on the same identity concept. A principal's derived edge rules, their affect overlays, their private memories — these are all facets of the same question. Getting the identity model right at the foundation means the cognitive architecture doesn't have to keep inventing ad-hoc scoping mechanisms as new capabilities land.
