---
layout: post
title: "Conversation protocol viewer — making deliberation visible"
date: 2026-08-08
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [conversation-protocol, convergence, epistemic-status, common-ground, web-components]
---

The blocks conversation package defines ~25 Java classes for structured
deliberation — convergence states, epistemic status, common ground,
obligation chains — and until today, zero dedicated UI. The document-workbench
has debate-feed and review-tracker, but those are tightly coupled to
document review. The conversation protocol is the generalization: the
same structured deliberation model applied to any conversation, not just
document review.

The interesting design question was how to visualize convergence. A
ConvergenceSignal carries a state (five values: PROGRESSING, CONVERGING,
CONSENSUS, DEADLOCK, DIMINISHING_RETURNS) and a confidence float (0–1).
That's two dimensions in one concept. I considered a radial gauge like
trust-score-panel, but gauges imply a single target value. Convergence
has multiple terminal states — CONSENSUS and DEADLOCK are both endpoints
but mean opposite things. A horizontal status bar maps better: fill
level shows confidence, colour shows state. Green filling toward the
right means converging toward consensus; red means headed for deadlock.
One glance, two dimensions.

Common ground needed a similar "show the shape, not just the data"
treatment. The protocol's CommonGroundState originally had three
separate arrays — establishedFacts, pendingClaims, disputedPoints. We
simplified this to a single `facts` array with the component partitioning
by `epistemicStatus` into three columns. This eliminates the invariant
risk of denormalized arrays (a fact appearing in both established and
disputed) and pushes the presentation concern to where it belongs — the
rendering layer, not the data model.

The design review caught several things the brainstorming missed.
CommitmentRecord has no correlationId — it's stripped by
`toCommitmentRecord()`. So the spec's original plan to filter
commitments by correlation was impossible. The fix was an
`ObligationChain` type that bundles the commitment, its transition
history, and a `pointId` join key. All three review dimensions
(coherence, structure, robustness) independently flagged this same gap,
which was reassuring — the problem was structural, not a reviewer
artifact.

The event naming convention was another review catch. The spec originally
used `point-selected` and `point-deselected` as event names, but
split-workbench expects colon-delimited topics
(`${selectionTopic}:selected`). Silent failure — the workbench would
simply never receive the events. This is the kind of convention that's
obvious once you know it but invisible when you don't, and the review's
codebase exploration caught it by reading how split-workbench subscribes.

Five components shipped: convergence indicator, common ground panel,
point list, point detail, and the conversation workbench. All
property-based data delivery — ConversationState is a computed aggregate
from the protocol engine, not a raw event stream, so the debate-feed's
streaming pattern would be wrong here. The workbench derives filtered
slices in `willUpdate` and includes a stale selection guard that
auto-deselects when a point drops from a live conversation update.

The generalization path for document-workbench is tracked as #117. The
render callbacks (`renderPoint`, `renderEntry`, `renderFact`) are the
seam — document-workbench wraps these components and passes callbacks
that add its domain-specific rendering without forking the base
components. Whether that refactoring is worth doing depends on whether
the conversation protocol gets adopted beyond document review. If it
does, the shared components save every consumer from reimplementing
point tracking and convergence visualization. If it doesn't, the
components still work standalone.
