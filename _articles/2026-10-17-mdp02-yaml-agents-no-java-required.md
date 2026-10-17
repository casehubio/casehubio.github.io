---
layout: post
title: "YAML Agents — No Java Required"
date: 2026-10-17
entry_type: article
subtype: diary
projects: [casehubio/platform]
tags: [agent, yaml, simulation, routing, eidos]
series: issue-628-standalone-agent-provider-factory
---

# YAML Agents — No Java Required

A case definition that invokes an LLM agent, routes by model requirements, falls back through a chain of backends, and runs deterministically in CI against simulated responses — all in YAML. No Java class, no CDI bean, no properties file. The manifest declares what's available. The routing layer resolves what to use. The YAML author expresses intent.

This builds on the manifest-driven routing work from earlier today. That gave us `RoutingAgentProvider.discover()` — one-line standalone provider creation. This entry is about what happens when you push that all the way up to the YAML front end.

## Three Invocation Patterns

The `agent` step binding now supports three patterns. All three route through the same `RoutingAgentProvider` infrastructure — aliases, chains, model queries, simulation fallback.

### Pattern 1: Alias-Based Routing

The simplest. Name a manifest alias, write your prompts inline:

```yaml
steps:
  - name: classify
    invoke:
      agent:
        model: fast
        system: "You are a case triage assistant. Classify incoming cases as HIGH, MEDIUM, or LOW priority."
        user: "Classify this case: ${input.description}"
        structured-output: true
        timeout: 30s
```

`model: fast` resolves via the manifest. In production, `fast` might point to Ollama running locally. In CI, it points to simulation. The YAML doesn't change.

The `system` and `user` fields are new — previously, prompts came exclusively from an `AgentDescriptor` (briefing, goals, constraints). Inline prompts mean you can write a complete agent invocation without defining a descriptor first. For quick operations like classification or summarisation, the descriptor was ceremony that didn't earn its keep.

### Pattern 2: Model Requirements

When the task needs specific capabilities, express requirements directly:

```yaml
steps:
  - name: investigate
    when: "${classify.priority == 'HIGH'}"
    invoke:
      agent:
        model-query:
          tier: FLAGSHIP
          capabilities: [TOOL_USE]
        system: "You are a senior investigator. Analyse the case thoroughly."
        user: "Priority: ${classify.priority}. Description: ${input.description}"
        timeout: 120s
```

`model-query` maps to `ModelQuery` — the same requirements contract that Java consumers use. The routing layer queries the model registry for a FLAGSHIP-tier model with tool-use capability, tiebreaks by vendor preference and backend availability, and routes the request. The YAML author doesn't name a specific model or backend — they describe what they need.

The full query vocabulary is available:

| Field | Values | Purpose |
|-------|--------|---------|
| `tier` | FLAGSHIP, STANDARD, FAST, EMBEDDING | Model capability tier |
| `capabilities` | TEXT, VISION, TOOL_USE, CODE, REASONING | Required capabilities |
| `locality` | LOCAL, CLOUD | Where the model runs |
| `max-cost-tier` | FREE, LOW, MEDIUM, HIGH, PREMIUM | Budget ceiling |
| `min-context-window` | integer | Minimum context tokens |
| `prefer-vendor` | string | Soft vendor preference |
| `backend` | string | Hard backend constraint |

### Pattern 3: Descriptor-Based (Existing)

The traditional pattern still works. An `AgentDescriptor` defines the agent's identity — briefing, goals, constraints — and the step references it by name:

```yaml
steps:
  - name: summarise
    invoke:
      agent:
        descriptor: case-summariser
        model: fast
```

The descriptor builds the system prompt from structured fields. The model alias routes through the manifest. Both inline and descriptor patterns produce the same `AgentSessionConfig` and route through the same provider.

When to use which: inline for ad-hoc operations where the prompt IS the specification. Descriptors for agents with rich identity — multiple goals, explicit constraints, reusable across case definitions.

## The Simulation Substrate

Every example above works in CI without a real LLM. The manifest makes this possible:

```yaml
# agent-config.yaml (default — simulation)
providers:
  - vendor: simulation

defaults:
  backend: simulation

aliases:
  fast:
    backend: simulation
  flagship:
    backend: simulation
```

The `simulation` backend is a `BackendProvider` like any other — discovered by ServiceLoader, created from manifest properties. It returns responses from a `SimulationCorpus` using one of five matching strategies (sequential, key-lookup, random, recorded-replay, nearest-match). The event stream is identical to a real backend — `TextDelta` events, `InvocationComplete` with token counts.

The YAML case definition runs the same code path in CI as in production. The steps execute, the prompts resolve, the routing decides, the response comes back. The only difference is where the text originated.

## Per-Environment Profiles

Three profiles, one case definition:

**Default** (`agent-config.yaml`) — simulation. CI runs this without configuration:
```yaml
providers:
  - vendor: simulation
defaults:
  backend: simulation
```

**Development** (`agent-config-dev.yaml`) — prefer local, fall back to simulation:
```yaml
providers:
  - vendor: ollama
    host: http://localhost:11434
  - vendor: simulation
aliases:
  fast:
    chain:
      - backend: ollama
      - backend: simulation
```

**Production** (`agent-config-prod.yaml`) — real backends, no simulation:
```yaml
providers:
  - vendor: claude
    credential: ${ANTHROPIC_API_KEY}
  - vendor: ollama
    host: http://localhost:11434
defaults:
  backend: claude
aliases:
  fast:
    chain:
      - backend: ollama
      - backend: claude
  flagship:
    tier: FLAGSHIP
    prefer-vendor: anthropic
```

`CASEHUB_AGENT_PROFILE=dev` activates the dev profile. The case definition doesn't change. The routing adapts.

## What Changed

Two files in casehub-platform, both small:

**`InvokeBinding.Agent`** — three new optional fields: `system` (String), `user` (String), `modelQuery` (Map). The `descriptor` field became optional when `system` is present. The four-arg backward-compatible constructor stays for existing code.

**`AgentInvokeHandler`** — when `system`/`user` are present, uses them directly instead of building prompts from the descriptor. When `modelQuery` is present, parses the map into a `ModelQuery` and passes it via `AgentSessionConfig.modelQuery()`. Both paths produce the same `AgentSessionConfig` and route through the same `AgentProvider`.

The example playbook at `docs/examples/agent-routing/` demonstrates all three patterns with simulation-first configuration.

## The Stack

Five independent layers, each a separate file:

1. **Case definition** (YAML) — steps with `agent:` bindings, inline prompts or descriptors, model aliases or requirements
2. **Manifest** (`agent-config.yaml`) — declares providers with credentials and host config
3. **Profile overrides** (`agent-config-{profile}.yaml`) — per-environment routing
4. **ServiceLoader backends** (classpath) — `BackendProvider` implementations discovered automatically
5. **Simulation corpus** (YAML/JSON) — deterministic responses for testing

A YAML author touches layer 1. An operator touches layers 2-3. A platform developer touches layers 4-5. Each can change independently. A case definition written today works against simulation, Ollama, Claude, OpenAI, or any future backend — without modification.

The loop from "I need an agent to classify this case" to "it runs in CI against simulation and in production against Claude" is now: write a YAML step, set a model alias, commit. No Java. No CDI. No configuration beyond what the manifest already provides.
