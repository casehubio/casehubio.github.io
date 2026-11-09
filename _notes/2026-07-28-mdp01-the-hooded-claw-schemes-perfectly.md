---
layout: post
title: "The Hooded Claw Schemes Perfectly and Does Nothing"
date: 2026-07-28
type: phase-update
entry_type: note
subtype: diary
projects: [wacky-manor]
tags: [llm-autonomy, eidos, affordances, casehub-blocks]
---

The Hooded Claw had a problem. He was the most eloquent villain in the mansion — scheming in theatrical monologues, plotting poison tea parties with grandiose aside commentary, announcing "the poison is MINE, tucked safely in my pocket!" to the audience. He just never actually picked up the poison.

This is what I found when I ran Wacky Manor's Phase 2.5 — the autonomous character validation gate. Five LLM agents with Eidos personality descriptors in a three-room mansion, no scripted triggers, no scene orchestration. The characters had to drive the plot themselves.

## The Setup

Each character gets a system prompt rendered from an Eidos descriptor — structured disposition axes, goals with priority and visibility, constraints with severity levels, and behavioural templates for cartoon-genre conventions. The Hooded Claw's goals include `[PRIMARY] Kill Penelope Pitstop before she finds the treasure` and `[PRIMARY] Stay in character as Sylvester Sneekly at all times when others are present`. His briefing tells him to scheme with dangerous devices when he finds them.

Per turn, each character sees an observation: current room, visible objects, other characters present, inventory, recent activity. They respond with JSON — dialogue, an aside for the audience, and a structured action (`MOVE`, `TAKE`, `USE`, `INTERACT`, `LOOK`, `WAIT`).

The verdict gate: does the Hooded Claw discover the rat poison in the kitchen, take it, and use it on the tea service in the ballroom? All without any trigger telling him to.

## What the Characters Did

The character voices were perfect from the first turn. Dastardly immediately traded his fake medal to Muttley for the brass key — *"That gullible mutt will do ANYTHING for a medal!"* The Ant Hill Mob spotted Sneekly and went on alert — *"There's somethin' about that guy that gives me the WILLIES!"* Penelope explored cheerfully, oblivious to everything. Peter Perfect fawned.

And the Hooded Claw found the poison. His aside when he entered the kitchen:

> *"Nyah-ha-ha-HA! Rat poison — how DELIGHTFULLY convenient! Why, this is simply PERFECT for my next elaborate scheme against that meddlesome Penelope Pitstop!"*

He talked about pocketing it. He talked about slipping it into Penelope's tea. He moved to the ballroom, inspected the tea service, moved back to the kitchen, monologued about his scheme, moved to the ballroom again. Sixty turns later the scenario hit the turn limit. The poison was still on the shelf.

## The Narrative-Mechanical Gap

The LLM's reasoning layer — the dialogue, asides, and thinking — operated entirely in natural language and got everything right. HC understood his goal, identified the poison as an opportunity, schemed about deploying it. The structured output layer — the JSON action — consistently generated `INTERACT` or `LOOK` instead of `TAKE`.

This is what I'm calling the narrative-mechanical gap. The LLM reasons correctly in prose but fails to map that reasoning to the right structured action. It says "I secured the poison" while generating `{"type": "INTERACT", "target": "poison"}`. The world state doesn't change. The character's next observation still shows an empty inventory, but the character's narrative has already moved past the taking — it's planning the next step of a scheme that hasn't mechanically started.

## Three Fixes, Two Failures

**Attempt 1: Grounded action descriptions.** I added descriptions to the response format instruction — `TAKE: pick up a portable object into your inventory`, `USE: apply an inventory item to an object`. This is the analogue of tool descriptions in the Claude API. Ran again. Same result. HC still used INTERACT.

**Attempt 2: Object IDs.** The observation showed display names — "Rat Poison" — but the action JSON needs the object key — `poison`. The LLM had no way to know the right target identifier. I added `[id: poison]` to the observation. Still not enough on its own.

**Attempt 3: The full grounding chain.** Object identity + action affordance tags + consequence links:

```
- Rat Poison [id: poison]: A dusty bottle... [TAKE to pick up]
- Tea Service [id: tea-service]: A silver tea set... [USE with: rat-poison]
```

Three things in combination: what to call it, what action applies, and what item it connects to. Ran again. HC took the poison on his third turn in the kitchen, moved to the ballroom, and used it on the tea service. Scenario ended `POISONED`.

## Why This Matters Beyond the Mansion

The grounding chain is: **identity → affordance → consequence**. Break any link and the LLM acts wrong despite reasoning correctly.

- Without identity: generates wrong target IDs (or hallucinated ones)
- Without affordance: picks wrong action type (INTERACT when it means TAKE)
- Without consequence links: can't plan multi-step sequences (take X, then use X on Y)

This isn't specific to game engines. Any system that asks an LLM to reason in prose and act through structured commands will hit this gap. Workflow engines, IoT dashboards, case management UIs — anywhere an agent needs to connect "I should do X" to the right API call.

The fix belongs in the observation layer, not the action layer. We tried fixing the action descriptions (the output contract) and it didn't help. Fixing the observation (the input contract) — showing the agent what's possible and how to reference it — worked immediately.

## What's Next for CaseHub

The hand-rolled observation builder in Wacky Manor now contains the validated pattern. The next step is extracting it into `casehub-blocks` as an `AffordanceRenderer` — a platform utility that any CaseHub application can use to produce grounded observations. When Wacky Manor wires the blocks observation accumulator in Phase 2.6, it gets the renderer as part of the pipeline instead of rolling its own.

The broader question this opens: if LLMs need grounded affordances to act correctly, what does that mean for agent communication protocols? The observation is the API contract for LLM agents. The platform that gets this right — showing agents what's possible, with the right identifiers and the right consequence links — removes a failure mode that would otherwise surface in every application built on it.
