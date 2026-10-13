---
layout: post
title: "Per-Type Event Levels and Some Repo Hygiene"
date: 2026-08-20
entry_type: note
subtype: diary
projects: [casehub-examples]
tags: [wacky-manor, observation, blocks, event-levels]
series: issue-7-small-issues-batch
---

All manor events used to share a single `EventLevel("manor", 0)`. Every dialogue, every movement, every aside — same priority. That's fine when TieredObservationRenderer only tiers by count, but it means level-based filtering is impossible. When budget pressure hits and the renderer needs to drop low-value events rather than summarise everything equally, there's no signal to work with.

The fix needed two repos. `PartitionedObservationService` in blocks bakes a single `EventLevel` into its constructor and stamps every event uniformly. To get per-event levels, I added a `Function<E, EventLevel>` constructor overload that resolves the level per event. The existing single-level constructor delegates with a constant function — nothing breaks.

On the manor side, three levels replace the flat `MANOR`:

- **DIALOGUE (30)** — dialogue and aside events
- **ACTION (20)** — non-move actions
- **MOVEMENT (10)** — move actions

The resolver maps event type and `ActionType` to the right level. A `switch` on type handles dialogue/aside; a guard clause catches `MOVE` before the switch. Ordinals are spaced by 10 for future insertion.

The levels are carried on each `LevelEvent` now, available for filtering when the renderer eventually needs it. The renderer itself doesn't change — it still tiers by count. But the data is there.

While looking at the batch of issues originally queued for this branch (#7, #35, #36, #37), three turned out to be filed in the wrong repo. #35 references trellis's `ActionService` — that's `Hortora/trellis`, not examples. #36 is about platform protocols and ARC42STORIES — `casehubio/docs`. #37 is `GatedAgentSession` leak detection — the class lives in `casehubio/platform`'s agent-gate module. Moved all three and closed the originals.
