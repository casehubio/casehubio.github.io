---
layout: post
title: "Commitment state as a composable surface"
date: 2026-07-28
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [commitment, qhorus, composability, type-promotion]
---

Qhorus commitments have had a full state machine for months — OPEN through FULFILLED, FAILED, DECLINED, DELEGATED, EXPIRED — but the visual rendering was scattered across two components with duplicated CSS and divergent state models. The channel-message had one copy of the badge styles. The task panel had another. The blocks-timeline had a simplified 4-stage model (COMMANDED → ACKNOWLEDGED → DONE → DECLINED) that didn't even use the same vocabulary.

I wanted to fix this properly: standalone components that any surface can import without pulling in channel-activity's dependencies.

The design review was where the interesting decisions got made. The initial spec had `commitment-viz` importing types from `channel-activity` — which immediately violated the blocks-ui layering rule that components never depend on each other except through `blocks-ui-core`. Claude caught this in round 1 and pushed the types into core, where `TrustLevel` and the work-item types already live. That forced a second fix: the blocks-timeline had its own `CommitmentState` interface (a data-input shape, not the domain state enum), creating a naming collision with the canonical type now in core. Renamed it to `CommitmentLifecycleData` — the strategy's simplified 4-stage model is unchanged, just the interface name that feeds it.

The range-decorator was the other design question worth getting right. The original spec had it taking `QhorusMessage[]` as input — which reintroduced the same layering violation the type promotion had just fixed. We introduced `DecorableMessage`, a minimal `{id, correlationId}` interface that `QhorusMessage` satisfies structurally. The decorator function returns `RangeDecoration[]` — pure metadata. The feed decides what the decoration looks like. Different surfaces can use the same function with completely different visual treatments.

Four components, each under 100 lines: a state pill, a transition badge, a range bar (compact and detailed modes via a `mode` property), and the decorator function. The pill composes into the badge. The range bar reuses `pulseAnimation` from core for open commitments. Nothing knows about channel-activity.

The kpi-metric-row density fix landed on the same branch — the `density` property already existed with grid and padding breakpoints, but the sparkline height was hardcoded at 20px regardless of density. Now it scales: 40/32/24px for comfortable/compact/dense.

The commitment types sitting in core now means any future component that needs to render commitment state — case-explorer, work-item-detail, a hypothetical commitment dashboard — imports from the same place channel-activity does. The type promotion pattern is the same one trust-score types followed. It's becoming the standard path for domain types that cross component boundaries.
