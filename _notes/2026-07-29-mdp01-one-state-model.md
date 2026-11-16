---
layout: post
title: "One state model, not two"
date: 2026-07-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
---

The commitment lifecycle strategy in blocks-timeline had its own state model — COMMANDED, ACKNOWLEDGED, DONE, DECLINED. Four stages, linear progression, index-based status resolution. The real qhorus model has seven states: OPEN, ACKNOWLEDGED, FULFILLED, FAILED, DECLINED, DELEGATED, EXPIRED. And it's not linear — OPEN can jump directly to DECLINED without passing through ACKNOWLEDGED.

The timeline strategy was using `linearResolveStatus`, which marks stages before the current index as completed and stages after as pending. That's correct for a pipeline. For a branching state machine, it's wrong — a commitment that goes OPEN → DECLINED should show ACKNOWLEDGED as *skipped*, not *completed*. The existing `stateProgressionStrategy` already had `defaultResolveStatus`, which checks the transition set for visited states instead of assuming index order. The fix was delegation: `commitmentLifecycleStrategy` now calls `stateProgressionStrategy` and prepends a `transformData` adapter that maps `CommitmentLifecycleData` to `StateData`. The parallel ontology is gone.

DELEGATED surfaced a gap during the refactor. Core types treat it as terminal (`isTerminalCommitmentState` returns true), but `QHORUS_STAGES` had no `terminal` flag on it. The resolver returned `'active'` for a state that should render as completed. The fix was expanding `StageConfig.terminal` from `'success' | 'failure'` to `'success' | 'failure' | 'transfer'` — DELEGATED isn't a failure, it's a handoff. Both resolvers needed the new branch.

The other half of this work was adopting `commitment-state-pill` across channel-activity. Four components — channel-message, channel-task-panel, channel-correlation-panel, channel-thread — each had their own copy of the commitment badge CSS. Seven colour classes, duplicated four times. The pill component already existed in commitment-viz from the previous branch.

Promoting the pill to channel-activity meant adding commitment-viz as a dependency — which would have been a component-to-component dependency, violating the layering rule that components depend on core and pages, never on each other. The adversarial design review caught this before implementation started. The fix was promoting `commitment-state-pill` and its styling utilities to blocks-ui-core, where `blocks-confirm-dialog` already set the precedent for custom elements in core.

While replacing the badges, the correlationId bug from issue #101 fell out naturally. `channel-task-panel` and `channel-correlation-panel` were looking up commitments by `msg.id`, but `toCommitmentMap` keys by `correlationId`. Messages where `id !== correlationId` — replies, delegations — silently missed their commitment state.
