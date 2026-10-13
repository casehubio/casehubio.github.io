---
title: Thirteen Agents and a Split
date: 2026-08-21
author: Mark Proctor
series: casehub-fsitrading
entry: mdp10
tags: [implementation, agents, casehub, overnight-ops, yaml-casehub]
publish: false
---

# Thirteen Agents and a Split

After the spec review rewrote half the design, I expected the implementation to be the easy part. Six tasks, clear SPI contracts, known platform patterns. And mostly it was — but two things made me stop and think.

## The descriptor as a wiring registry

The C4a incident response system needs thirteen agents. Seven are rule-based — halt trading, close positions, alert the on-call trader, adjust limits, monitor, verify. Six are LLM stubs that will eventually reason about which positions to reduce, how to hedge, what a news event means for exposure.

The interesting design question isn't the agents themselves — each is a one-method class that takes incident context and returns an action map. The interesting part is how they get wired into the case lifecycle.

`YamlCaseHub` loads a YAML case definition and calls `augment()` to let the subclass modify it. Workers loaded from YAML arrive with `WorkerFunction.NONE` — they're structurally defined but functionally empty. The subclass replaces them with workers that have actual logic. Because `Worker` is a Java record, you can't mutate the function field in place. Instead you `removeIf` the empty workers, then `add` new ones with functions attached.

The `OvernightIncidentCaseDescriptor` sits between the CaseHub and the agents. It holds a static registry mapping agent names to implementations — the same names that appear in `MarketEventTypeDescriptor` for event-type routing and `IncidentSeverityDescriptor` for decomposition steps. When `augmentWorkers()` runs, it iterates the registry and wires each agent as a Worker with a capability binding.

This means the YAML defines *when* things fire (bindings triggered by context changes), the descriptors define *what* fires for each severity and event type, and the agents define *how* the work gets done. Three layers, cleanly separated. Adding a new agent for a new event type means one enum value, one descriptor entry, one agent class, one registry entry. No binding changes, no YAML changes.

## The split that wasn't empty

Code review caught a data corruption bug I wouldn't have found for weeks. `IncidentEntity` stores instruments as a comma-separated string — `String.join(",", instruments)`. The inverse — `instruments.split(",")` — reconstructs the list. Clean enough for non-empty lists.

But `String.join(",", List.of())` produces `""`. And `"".split(",")` does not produce an empty array. It produces `[""]` — a single-element array containing one empty string. The round-trip through JPA silently transforms an empty instrument list into a list containing one phantom instrument.

The fix is a one-liner: check `instruments.isEmpty()` before splitting. The bug would only manifest if an incident were created without instruments — unlikely in production, but the contract doesn't prohibit it. The kind of thing that surfaces six months later as a mysterious extra row in a report.

## Bridging detection and response

The C2 pipeline detects market events — trend reversals, regime changes — through `FsiMarketEventDetector`, which takes `Consumer<T>` callbacks in its constructor. C4 needs to observe those same events to trigger incident response. The bridge is clean: `MarketPulseConfiguration` produces the detector as a CDI bean, passing `Event<T>::fire` as the consumers. The detector fires its callbacks; CDI turns them into observable events; `FsiIncidentTrigger` observes them and classifies severity.

No coupling between C2 and C4. The detector doesn't know incidents exist. The trigger doesn't know how detection works. They share a vocabulary — `TrendReversalDetected`, `RegimeChanged` — and CDI handles the rest.

The severity classification itself uses off-hours amplification: a `MEDIUM` event (circuit breaker, news event) becomes `HIGH` before 07:00 or after 20:00. At 3am, nobody is watching. The system needs to be more aggressive about escalation, not less.

The overnight ops backend is feature-complete. Thirteen agents, two YAML case definitions, seven REST endpoints, JPA persistence, SLA enforcement, risk gating. The LLM agents are stubs with deterministic fallbacks — real intelligence comes when we wire in the model layer. The structure is there to receive it.
