---
title: The Review That Rewrote the Design
date: 2026-08-21
author: Mark Proctor
series: casehub-fsitrading
entry: mdp09
tags: [design-review, protocols, platform-coherence, spi, overnight-ops]
publish: false
---

# The Review That Rewrote the Design

The C4a overnight ops backend started as a straightforward SPI implementation exercise. The replan spec had it mapped out: case definition, HTN decomposition, risk classifier, SLA policy, seven REST endpoints. I thought the brainstorm would mostly confirm the spec's architecture and the design would flow from there.

It didn't.

## Eight decisions, three rounds of review

The brainstorm surfaced seven genuine design decisions before we even touched the spec. Notification mechanism — do we use the platform's `NotificationDispatcher` or roll our own? (Turns out the class exists in docs but isn't published as a Maven artifact yet — CDI events for now.) Where does the incident trigger originate? (Separate bean observing C2's events, not extending the detector.) Should the risk classifier share logic with C1's `FsiRiskAssessor`? That last one was worth thinking about deeply — the two classifiers operate at different architectural layers, receive different input types, and classify on different dimensions. The 10%/25% thresholds happen to match today, but they represent independent policies that should be free to diverge. DRY on two constants is trivial duplication; coupling them is real cost.

The decision review caught the first correction: `NotificationDispatcher` does exist in the platform docs, even though the class isn't shipped. That shifted D1 from "build custom" to "fire CDI events that the platform will observe when it ships." A better design — but one I wouldn't have reached without the adversarial pass.

## The spec review that actually worked

The automated spec review failed four times in a row. Agents kept dying at 300 seconds with exit code 1 and $0.00 reported cost. No stderr, no output file, no explanation. We eventually traced it to `--max-budget-usd` on the Claude CLI — a hard enforcement flag that kills the subprocess when budget is exhausted. The agents were spending their entire budget reading decompiled platform SPIs via IntelliJ, then dying during output generation.

The fix was simple: remove the flag. Budgets should be advisory, not execution gates. Once we did that, the review completed in 420 seconds and produced the most useful feedback of the session — `RiskDecision.GateRequired` is a record constructor with seven parameters (no builder), `BreachDecision.Exhausted` is the right variant for auto-execute (not `Fail`), and `CaseHub` is an abstract class you extend (not a DSL builder). Three HIGH findings that would have been compilation errors during implementation.

## Four protocols I didn't know applied

The real surprise was the protocol findings. The reviewer cited three platform protocols by name and ID — and all three turned out to be real, documented, and directly applicable:

**case-definition-layers** says application case definitions must use YAML + `YamlCaseHub` + a `*CaseDescriptor` companion. I had the case definition extending raw `CaseHub` with a programmatic `getDefinition()`. Wrong pattern — YAML separates workflow structure from worker logic, making the incident response reviewable by compliance auditors who don't read Java.

**descriptor-handler-pattern** says enum-dispatched behaviour must live in descriptor POJOs, not switch statements across service classes. My severity decomposition and event-type routing were static methods and maps in an orchestration bean. The fix was `IncidentSeverityDescriptor` and `MarketEventTypeDescriptor` — pure Java records that carry decomposition steps, SLA windows, agent routing, and fallback actions per enum value.

**module-tier-structure** (the Store SPI pattern) says JPA entities must live behind an SPI interface in the api module. I had `IncidentEntity` and `IncidentRepository` directly in app. The fix was `IncidentStore` as a pure-Java interface in api, with `JpaIncidentStore` in app.

The fourth — **oversight-action-gate-dedicated-hub** — means the risk classifier's action gates need their own dedicated `YamlCaseHub`, not programmatic bindings on the incident case. A subtlety about how the engine fires context-change events that I wouldn't have caught until runtime.

## What landed

Two implementation batches survived the design gauntlet. The API model layer is clean: `IncidentSeverity` enum, two descriptor POJOs, four CDI event records, the `IncidentStore` SPI, and the `MarketEventType` extension with COUNTERPARTY_FAILURE and MARGIN_CALL. The platform SPI layer has `FsiActionRiskClassifier` (three classification dimensions, `@RiskClassifier` auto-composed) and `FsiSlaBreachPolicy` (two-tier stateless escalation via `candidateGroups`, using `Exhausted` not `Fail`).

Four more tasks remain: YAML case definitions with descriptors, thirteen response agents, the incident trigger with callback bridging, and REST endpoints. The foundation is solid — the protocols shaped it into something that fits the platform rather than fighting it.
