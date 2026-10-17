---
layout: post
title: "Manifest-Driven Agent Routing — One Line to Rule Them All"
date: 2026-10-17
entry_type: article
subtype: diary
projects: [casehubio/platform]
tags: [agent, manifest, routing, serviceloader, standalone]
series: issue-628-standalone-agent-provider-factory
---

# Manifest-Driven Agent Routing — One Line to Rule Them All

Until today, getting an `AgentProvider` in casehub-platform required CDI. The container discovered backends via `Instance<AgentBackend>`, wired the registry, loaded the manifest, and produced the routing provider. If you were outside CDI — topology integration tests, standalone tools, anything not running inside Quarkus — you were on your own. The `casehub-ops` topology tests had reimplemented routing with raw HTTP calls to Ollama because there was no other way in.

The manifest already declares which providers to use. The classpath already declares which backend implementations are available. Consumer code was duplicating both.

## The Consumer Experience

Here's what it looks like now:

```java
var provider = RoutingAgentProvider.discover(projectDir);
```

One line. The manifest in `projectDir` declares your providers. ServiceLoader discovers the backend implementations on the classpath. The routing provider wires itself. Consumer code never touches a backend class, a registry, or a manifest type.

Invoke with default model:

```java
provider.invoke(AgentSessionConfig.of(systemPrompt, userPrompt));
```

Invoke with requirements — tier, capability, locality:

```java
provider.invoke(AgentSessionConfig.builder()
    .modelQuery(ModelQuery.builder()
        .tier(ModelTier.FLAGSHIP)
        .requiredCapabilities(Set.of(ModelCapabilities.TOOL_USE))
        .locality(ModelLocality.LOCAL)
        .build())
    .build());
```

Invoke with a manifest alias:

```java
provider.invoke(AgentSessionConfig.builder()
    .model("fast")
    .build());
```

The consumer expresses requirements. The platform resolves them. The four-step resolution chain — alias lookup, tier reference, model registry query, backend key match — handles the rest.

## The Manifest

An `agent-config.yaml` in your project directory declares providers:

```yaml
providers:
  - vendor: ollama
    host: http://localhost:11434
  - vendor: claude
    credential: ${ANTHROPIC_API_KEY}
  - vendor: openai
    credential: ${OPENAI_API_KEY}

aliases:
  fast:
    model: llama3.2
    backend: ollama
  flagship:
    tier: FLAGSHIP
    prefer-vendor: anthropic

defaults:
  backend: claude
```

`discover()` walks the directory hierarchy upward, merges manifests from parent directories, handles remote sources with cycle detection — all the machinery that `ManifestLoader` already provided. The standalone path inherits the full discovery pipeline.

## How discover() Works

Three things happen inside that one line:

**1. Manifest loading.** `AgentConfigLoader.withDefaults(projectDir)` creates a loader with sensible no-op stubs for the nine constructor parameters that previously required CDI beans — credential stores, model registries, DID resolvers, local model reconcilers. The manifest loads and processes exactly as it would under Quarkus, minus the CDI lifecycle.

**2. Backend discovery via ServiceLoader.** Each backend core module registers a `BackendProvider` implementation:

```
META-INF/services/io.casehub.platform.agent.BackendProvider
```

```
io.casehub.platform.agent.ollama.OllamaBackendProvider
```

The provider knows how to create its backend from manifest properties — host, API key, model defaults, timeouts. Put `agent-ollama-core` on the classpath, declare `vendor: ollama` in the manifest, and Ollama is available. No configuration beyond what the manifest provides.

**3. Registry + routing assembly.** `discover()` matches manifest provider entries to ServiceLoader-discovered providers by vendor key. Matched providers create backends, which register in an `InMemoryBackendInstanceRegistry`. The `RoutingAgentProvider` gets the registry, the manifest aliases, the populated model registry, and the default backend key — same construction that `RouterBeans` does under CDI.

Unresolvable providers — manifest declares a vendor with no matching jar on the classpath — log a warning and get skipped. The provider starts with whatever IS available. This matches CDI behaviour: missing optional modules don't prevent startup.

## ModelQuery — Requirements, Not Implementations

The key design principle: consumer code specifies what it needs, not what serves it. `ModelQuery` is the requirements contract:

```java
ModelQuery.builder()
    .tier(ModelTier.FLAGSHIP)                           // FLAGSHIP, STANDARD, FAST, EMBEDDING
    .requiredCapabilities(Set.of("VISION", "TOOL_USE")) // must support these
    .locality(ModelLocality.LOCAL)                       // LOCAL or CLOUD
    .maxCostTier(CostTier.MEDIUM)                       // budget cap
    .minContextWindow(128_000)                           // minimum context
    .preferVendor("anthropic")                           // soft preference, not a hard filter
    .build()
```

The model registry — populated from the manifest's model catalog — resolves the query against available models. The routing provider tiebreaks: prefer the default backend, then the preferred vendor, then first match. The consumer gets the best available model meeting their requirements without knowing which backend handles it.

Manifest aliases work the same way. Define `fast` as an alias, and `AgentSessionConfig.builder().model("fast").build()` resolves it through the alias → model chain → query resolution path.

## Model Chains and Fallback

Aliases can define chains with automatic fallback:

```yaml
aliases:
  reliable:
    chain:
      - model: claude-opus-4-6-20260901
        backend: claude
      - tier: FLAGSHIP
        prefer-vendor: anthropic
      - tier: STANDARD
```

If the first entry in the chain fails with a rate limit or connection error, routing falls through to the next entry. The consumer sees a single invocation; the retry logic is internal to the routing provider.

## The Ops Pattern — Remote, Local, Simulated

This is where chains get interesting. The `casehub-ops` topology tests need to verify agent behaviour across the full stack — case routing, workitem completion, ledger updates. They need LLM responses, but they don't always need *real* LLM responses. And they certainly don't need tests that break when Ollama isn't running or the API key is expired.

The manifest solves this with a three-tier chain:

```yaml
providers:
  - vendor: claude
    credential: ${ANTHROPIC_API_KEY}
  - vendor: ollama
    host: http://localhost:11434
  - vendor: simulation

aliases:
  topology:
    chain:
      - backend: claude
      - backend: ollama
      - backend: simulation
```

The chain tries Claude first — the real thing, cloud inference, full capability. If the API key is missing or the rate limit hits, it falls through to Ollama running locally. If Ollama isn't running either, simulation handles it. The test code is identical in every environment:

```java
var provider = RoutingAgentProvider.discover(testConfigDir);
provider.invoke(AgentSessionConfig.builder().model("topology").build());
```

In CI with no API keys and no Ollama: simulation handles every request. On a developer's machine with Ollama: local inference runs. In staging with Claude credentials: real cloud inference. Same test, same manifest, same code path. The only thing that changes is which backends are available on the classpath and which credentials are set.

### Environment-Based Routing with Profiles

The manifest supports profiles — activated by `CASEHUB_AGENT_PROFILE` or passed to `discover()`:

```java
// Production — real backends only
var provider = RoutingAgentProvider.discover(projectDir, "production");

// CI — force simulation, no network calls
var provider = RoutingAgentProvider.discover(projectDir, "ci");

// Dev — prefer local
var provider = RoutingAgentProvider.discover(projectDir, "dev");
```

Profiles override the base manifest. A `ci` profile can strip the remote providers entirely and set simulation as the default:

```yaml
# agent-config.yaml (base)
providers:
  - vendor: claude
    credential: ${ANTHROPIC_API_KEY}
  - vendor: ollama
    host: http://localhost:11434
  - vendor: simulation

aliases:
  topology:
    chain:
      - backend: claude
      - backend: ollama
      - backend: simulation

# agent-config-ci.yaml (profile override)
defaults:
  backend: simulation
aliases:
  topology:
    backend: simulation
```

In CI, `CASEHUB_AGENT_PROFILE=ci` forces simulation for everything — no chain fallback needed, no waiting for connection timeouts. In dev, the chain handles availability naturally. In production, the profile can exclude simulation entirely so a misconfigured chain never accidentally serves canned responses.

The override works at every level. Force a specific backend for a single test by setting `backend` directly in the alias. Force an entire profile for an environment via the env var. Or let the chain discover what's available and route accordingly.

### Simulation as a Backend, Not a Mock

The simulation API is the seventh `BackendProvider` — `SimulatedBackendProvider` with vendor key `"simulation"`. It's not a mock framework bolted onto the side. It's an `AgentBackend` that returns responses from a `SimulationCorpus`.

The corpus is a collection of input/output pairs. Five strategies determine how inputs match to responses:

| Strategy | Behaviour |
|----------|-----------|
| **Sequential** | Returns the next response in order — predictable, good for scripted scenarios |
| **KeyLookup** | Matches input to output by an extracted key — deterministic mapping |
| **Random** | Picks a random response — stress testing, fuzz testing |
| **RecordedReplay** | Replays previously recorded real invocations — regression testing |
| **NearestMatch** | Scores inputs against the corpus and returns the closest — flexible |

The routing provider doesn't know it's talking to a simulation. The response comes back as the same `Multi<AgentEvent>` stream — `TextDelta` events, `InvocationComplete` with token counts, the full event model. Code downstream of the invocation has no way to distinguish a simulated response from a real one.

This is the point. The simulation backend isn't for unit testing individual methods — it's for integration testing full system behaviour where the LLM is one component in a larger pipeline. The pipeline exercises the same code paths, the same event processing, the same routing infrastructure. The only thing that's different is where the text came from.

### Recording and Replaying

`RecordedReplayStrategy` is particularly useful for regression suites. Record a real invocation once:

```java
var corpus = new InMemorySimulationCorpus();
var realProvider = RoutingAgentProvider.discover(prodConfigDir);

// Record
var events = realProvider.invoke(config).collect().asList().await().indefinitely();
corpus.record(config, events);

// Later — replay
var simProvider = RoutingAgentProvider.of(
    new SimulatedAgentBackend(corpus, new RecordedReplayStrategy()));
simProvider.invoke(config);  // returns the recorded response
```

The recorded corpus can be serialised to YAML or JSON and checked into the test fixtures. Tests run against recorded responses — fast, deterministic, no network calls. When the real API changes behaviour, re-record and the corpus updates.

### Route Observability Tells You What Happened

This is where `InvocationComplete.backendKey()` earns its keep. After a topology test runs, the assertion can check not just the result but *how* it was produced:

```java
var events = provider.invoke(config).collect().asList().await().indefinitely();
var completion = events.stream()
    .filter(e -> e instanceof AgentEvent.InvocationComplete)
    .map(AgentEvent.InvocationComplete.class::cast)
    .findFirst().orElseThrow();

// In CI: verify simulation handled it
assertEquals("simulation", completion.backendKey());

// Or: just log which backend was used — no assertion, just visibility
log.info("Topology test handled by: {}/{}", 
    completion.backendKey(), completion.instanceId());
```

The test doesn't need to know in advance which backend will handle it. The chain handles availability. The observability tells you after the fact.

## Route Observability

Every invocation reports which backend handled the request. `InvocationComplete` — the terminal event in the agent event stream — now carries `backendKey` and `instanceId`:

```java
provider.invoke(config)
    .subscribe().with(event -> {
        if (event instanceof AgentEvent.InvocationComplete ic) {
            System.out.println("Handled by: " + ic.backendKey()
                + "/" + ic.instanceId());
            // → "Handled by: ollama/default"
        }
    });
```

This is set by the routing layer after resolution, not by the backend itself. The consumer always knows the routing decision.

## BackendProvider — The Extension Point

Adding a new backend to the standalone path requires two things:

```java
public class MyBackendProvider implements BackendProvider {
    @Override
    public String vendorKey() { return "my-backend"; }

    @Override
    public AgentBackend create(Map<String, String> properties) {
        return new MyAgentBackend(
            properties.getOrDefault("host", "http://localhost:8080"),
            durationProperty(properties, "default-timeout", 120),
            intProperty(properties, "max-concurrent-sessions", 4)
        );
    }
}
```

Register in `META-INF/services/io.casehub.platform.agent.BackendProvider`:
```
com.example.MyBackendProvider
```

`BackendProvider` includes default methods for safe property parsing — `intProperty`, `longProperty`, `durationProperty` — that fall back to defaults on parse errors rather than crashing the entire provider setup.

The six platform backends ship with ServiceLoader registration: Ollama, OpenAI, Gemini (SDK-based), Claude, Codex, Gemini CLI (CLI-based). Put the corresponding `-core` jar on the classpath, and it's available to `discover()`.

## The of() Escape Hatches

For tests and platform-internal wiring where the manifest isn't the right entry point, `RoutingAgentProvider.of()` provides explicit construction:

```java
// Single backend, no manifest
var provider = RoutingAgentProvider.of(myBackend);

// Multiple backends with manifest
var provider = RoutingAgentProvider.of(
    List.of(ollamaBackend, claudeBackend),
    manifestResult,
    modelRegistry
);
```

These exist for test harnesses that need deterministic control — mock backends, fixed routing, no ServiceLoader discovery. They're not the consumer API.

## Pure YAML — Full Coverage Without Java

Here's where it all connects. The casehub YAML front end already has an `agent` step action. With manifest-driven routing, a YAML case definition gets the full agent capability stack — model selection, chain fallback, simulation — without a single line of Java.

A case definition that invokes an agent:

```yaml
steps:
  - agent:
      model: fast
      system: "You are a triage assistant for incoming cases."
      user: "Classify this case: ${case.description}"
      result: classification

  - agent:
      model-query:
        tier: FLAGSHIP
        capabilities: [TOOL_USE]
      system: "You are an analyst. Use the provided tools to investigate."
      user: "Investigate: ${classification.summary}"
      result: analysis
```

The first step uses a manifest alias (`fast`) — maybe Ollama locally, maybe GPT-4o in production. The second step uses a `ModelQuery` — give me a flagship model with tool-use capability, and the routing layer finds one. The YAML author doesn't name a backend, doesn't write a properties file, doesn't configure a client. The manifest declares what's available. The routing layer resolves what to use.

Variable interpolation — `${case.description}`, `${classification.summary}` — feeds case context into prompts. Results flow back into the case state. The agent step is a participant in the case lifecycle, not a side-channel call.

### Testing the YAML Path

This is where the manifest chain and simulation close the loop. The same YAML definition runs against three different backend configurations without modification:

**Production** (`CASEHUB_AGENT_PROFILE=production`):
```yaml
# agent-config-production.yaml
aliases:
  fast:
    backend: ollama
  # model-query resolves against the full model catalog
```

**Development** (`CASEHUB_AGENT_PROFILE=dev`):
```yaml
# agent-config-dev.yaml
aliases:
  fast:
    chain:
      - backend: ollama
      - backend: simulation
```

**CI** (`CASEHUB_AGENT_PROFILE=ci`):
```yaml
# agent-config-ci.yaml
defaults:
  backend: simulation
aliases:
  fast:
    backend: simulation
```

The YAML case definition doesn't change. The simulation corpus provides deterministic responses for each prompt pattern. CI runs are fast, repeatable, and don't need network access. When you need to verify against a real model — switch profiles.

The layers stack cleanly: YAML defines the case logic, the manifest defines the routing, profiles select the environment, simulation provides the testing substrate. Each layer is independent. Each can be changed without touching the others. And a YAML author who never reads `RoutingAgentProvider.java` gets the full power of the routing infrastructure just by referencing model aliases in their step definitions.

## What This Means

The `ManifestLlmClient` in `casehub-ops/topology-tests` — the one that reimplemented provider routing with raw HTTP calls — gets deleted. One `discover()` call replaces it. But the bigger win is what the topology tests can now do that they couldn't before: run the same test suite against remote, local, and simulated backends with a single manifest and zero conditional logic in the test code. The chain handles availability. Profiles handle environments. Simulation handles CI. Route observability tells you what happened.

Any standalone tool that needs to talk to LLMs through the platform's routing infrastructure can now do so without CDI. The manifest is the contract. The classpath is the discovery mechanism. Consumer code expresses requirements and gets routed to the right backend.

The CDI path is unchanged. `RouterBeans`, `BackendInstanceCoordinator`, `AgentConfigBeans` — all still work exactly as before. The standalone path shares the core classes but has its own construction: `discover()` + ServiceLoader instead of `@Inject` + `Instance<>`.

The full stack is now: YAML defines case logic → manifest declares providers → routing resolves requirements → simulation provides the testing substrate → profiles select the environment. Each layer is a separate file. Each can change independently. A YAML author gets the entire routing infrastructure without knowing it exists, and a platform operator gets environment-specific agent configuration without touching case definitions.

I'll follow up with the `ManifestLlmClient` deletion in a separate issue on casehub-ops.
