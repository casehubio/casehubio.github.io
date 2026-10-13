---
layout: post
title: "Trust scores and the data that was already there"
date: 2026-08-22
entry_type: note
subtype: diary
projects: [casehub-soc]
tags: [soc, web-ui, trust, routing, cbr, blocks-ui, platform-coherence, design-review]
---

The Trust tab was supposed to be the hard one. Phase 1 (Incidents) was data display. Phase 2 (Workbench) was form interaction. Phase 3 — trust scores, routing rationale, CBR similarity — felt like it would need new platform surface area. Routing decision logs that don't exist yet. Trust dashboard components built from scratch. Reconstruction of routing choices from scattered data.

None of that turned out to be true.

## The review that changed the design

I'd designed the entire Phase 3 around a false premise: that the platform doesn't persist routing decisions. The spec proposed reconstructing routing rationale at query time — look up which agent was assigned, look up current trust scores for all candidates, and fake a comparison view. The trade-off was documented and accepted: "shows current trust scores, not scores-at-routing-time."

Claude caught it during the decision review. `WorkerDecisionEntry` already has three fields I hadn't checked: `trustScoreAtRouting` (the score at routing time), `thresholdApplied` (the policy threshold), and `routingRationale` (a full JSON snapshot of `SelectionContext` — the selected candidate, all alternatives, their scores, phases, and human-readable reasons). The platform stores exactly what I was proposing to build.

The same review surfaced that `blocks-routing-rationale`, `blocks-trust-score-panel`, `blocks-trust-workbench`, and `blocks-similarity-panel` all exist as production blocks-ui components. I'd proposed building three SOC-specific replacements with impoverished data contracts — 4 fields per candidate where the platform component expects 9.

Sixteen findings across two review rounds. Every one of them was right. The revised spec is half the size of the original and produces a richer UI.

## What blocks-ui composition looks like in practice

The Trust view composes entirely from existing blocks-ui primitives. The fleet overview is six `blocks-trust-score-panel` instances in a CSS grid — each one takes an `actor-id` and `endpoint`, fetches its own data, renders global trust, dimension breakdowns, observation counts, and maturity phases. No custom card component needed.

The routing drill-down creates `blocks-routing-rationale` elements dynamically — one per capability in the selected case. Each receives `RoutingRationaleData` deserialized from `WorkerDecisionEntry.routingRationale` and enriched with current observation counts from `ActorTrustScoreRepository`. The component renders threshold markers, phase badges, policy summaries, and candidate comparison bars. The only SOC-specific component in the entire view is `soc-cbr-summary` — a thin Lit wrapper that computes an outcome statistics banner ("80% of 5 similar incidents were confirmed threats") and passes the incident list to `blocks-similarity-panel`.

Three Java files, three TypeScript files, one modified `index.ts`. The REST layer is straightforward — deserialize persisted JSON, enrich with current metadata, return. The complexity is in the platform components and the data already stored there.

## The enrichment gap

`SelectionContext.SelectedCandidate` has 4 fields (workerId, score, phase, reason). `CandidateScore` — the blocks-ui contract — has 9. The endpoint fills the gap: `trustScoreAtRouting` from the entry for the selected agent, `decisionCount` and maturity phase from `ActorTrustScoreRepository` for current state, workload score defaulted to 0.0 (not persisted — acceptable for v1), and the SelectionContext's pipeline phase ("trust", "cbr") mapped to the component's maturity phase (BOOTSTRAP/QUALIFIED/BORDERLINE).

This is the right boundary. The core routing decision — which agent was selected and why — is persisted and accurate. The supplementary metadata is enriched from current state. If a future platform version persists the full `CandidateScore` at routing time, the endpoint drops the enrichment logic and passes through directly. No UI changes.

## What this opens up

Phase 4 (Compliance) is the last tab. The audit trail viewer, compliance summary, and GDPR erasure action are all blocks-ui components that follow the same composition pattern. The real question after Phase 4 is whether the SOC web app's routing drill-down model — case-centric instead of agent-centric — is general enough to push upstream as an alternative mode for `blocks-trust-workbench`. The existing composite is per-agent (show one agent's routing history). The SOC view is per-case (show all agents' routing for one case). Both are valid; the data supports both.
