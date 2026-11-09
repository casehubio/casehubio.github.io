---
title: "The Trust Data Was There All Along"
date: 2026-08-05
type: diary
tags: [trust, observability, bayesian, decay, casehub, devtown]
---

# The Trust Data Was There All Along

Every response from devtown's governance API includes trust scores. The reviewer fleet endpoint returns a `trustByCapability` map per agent. The reviewer detail endpoint adds dimension scores, decision counts, and recent outcomes. The system health endpoint carries fleet-wide trust averages.

The Reviewers tab renders four columns: `actorId`, `maturityPhase`, `openCommitments`, `totalDecisions`. The trust data flows through the network, arrives in the browser, and gets thrown away.

This is a pattern worth recognising. It's not a missing feature — the data pipeline is complete. Trust scores accumulate from attestations, per-capability routing policies apply thresholds, incident feedback degrades scores, and the routing layer responds to shifts. The machinery is structurally sound. What's missing is a window into it.

## What operators can't answer

An operator monitoring a fleet of reviewer agents can see who's busy and who's idle. They cannot see why agent-alpha was chosen over agent-beta for a security review, whether agent-gamma's trust is rising or falling, or what threshold an agent needs to clear for architecture-review. The routing layer makes these decisions on every assignment, records the rationale in a JSON field on `WorkerDecisionEntry`, and never surfaces it.

The fix turned out to be surprisingly thin. The governance API already serves the data — three new REST endpoints delegate to `TrustQueryService` for score proxying, routing history, and routing decision detail. The trust-workbench component already existed in blocks-ui, built with a split-pane layout showing trust-score-panel and routing-rationale views. The implementation was mostly wiring: a hundred-line Lit wrapper that bridges reviewer table row selection to the workbench's `actorId` property.

## The decay investigation that closed itself

The companion issue asked whether dormant contributors' scores should decay over time. The arguments were reasonable on both sides — decay prevents stale high scores from accumulating indefinitely, but it also punishes contributors for taking breaks.

The investigation revealed there's nothing to build. `ExponentialDecayFunction` already weights each attestation by `2^(-ageInDays / halfLifeDays)` on every `TrustScoreJob` batch run. The job recomputes all actors, not just those with new data. A dormant contributor's attestations age and lose weight on each recomputation; the Bayesian score regresses naturally toward the Beta(1,1) prior as evidence fades.

Adding score-level decay on top of attestation-level decay would double-penalise dormant contributors — incoherent with the Bayesian model where the prior already handles uncertainty from sparse data. The answer was in the existing code, not in new code.

## The wrapper pattern

One implementation detail worth noting: `hostPanel` in pages-ui passes static props via `configure()` at mount time. There's no reactive binding from table row selection to panel properties. The `blocks-trust-workbench` component accepts `actorId` as a Lit property but doesn't listen for external selection events — it only handles its own internal routing events.

The bridge is a thin devtown-owned wrapper (`devtown-reviewer-workbench`) that owns both the fleet table and the trust-workbench. It fetches the reviewer dataset, renders a `pages-table`, listens for `row-activate` events, extracts the actor ID, and conditionally renders the trust-workbench with the selected `actorId`. The Reviewers tab becomes a single `hostPanel("reviewer-workbench")` call — the wrapper handles all the interactive state.

This is a general pattern for integrating declarative layout frameworks with interactive components. When the framework's binding model doesn't support the interaction you need, a thin wrapper component that owns both sides of the interaction is cleaner than fighting the framework's abstractions.
