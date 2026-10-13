---
layout: post
title: "Mood and engagement — the durability argument"
date: 2026-08-17
entry_type: note
subtype: diary
projects: [casehub-neocortex]
tags: [memory-api, mood, engagement, pad-model, agent-state]
series: issue-207-moodstate-mood-retrieval
---

The blocks research doc identified six composition patterns for autonomous agents. All but two compose existing platform capabilities. The two gaps: dynamic emotional state (mood) and conversational engagement scoring. Both are small — typed records, a retrieval utility, a CDI stream — but the placement question turned out to be more interesting than the implementation.

The obvious home for mood state is eidos, alongside `AgentDisposition` and `DispositionEvolution`. Mood is emotional identity, personality territory. And engagement scoring could reasonably be metadata on the existing `Outcome` events in the experience SPI. Both placements felt natural until I asked a harder question: what happens when the agent crashes?

## The durability argument

An agent needs to shut down and continue later. It needs to survive a restart after a crash. MoodState — the PAD values, the current emotional colour — is not transient in-memory state. It's durable state that must persist across process boundaries. And the persistence layer for agent state is memory, not personality.

This reframes the whole memory subsystem. It's not "things the agent remembers." It's "agent state that outlives a single execution." Experience events, relationship quality, reflections, mood, engagement outcomes — all of them are durable state that must survive a restart. Memory is the durability contract.

## PAD, not valence

The initial design had mood-modulated retrieval using a single valence dimension — positive or negative. The decision review caught this as architecturally incoherent: if MoodState uses three PAD axes (pleasure, arousal, dominance), retrieval modulation should use all three. An aroused agent should preferentially recall intense memories, not just positive ones. Mood-congruent recall in the affect literature operates on dimensional alignment, not a unidimensional axis.

The retrieval formula computes alignment as `1 - distance(memoryPAD, moodPAD) / sqrt(12)`, where `sqrt(12)` is the diagonal of the [-1,1]³ cube. Memories without PAD annotation are unaffected — graceful degradation for producers that haven't adopted the convention yet. CBR retrieval is excluded entirely — structured feature-vector search shouldn't be biased by how the agent feels.

## Engagement as a fourth domain

Engagement signals are temporally decoupled from the action they evaluate. The agent sends a message at T=0; whether the user responds, how quickly, whether the conversation continues — these can only be measured later. This rules out metadata on the existing `Outcome` event, which is recorded when the action completes.

The existing pattern already separates concerns into domains: experience, relationship, reflection. Each has a standalone event type, a converter, attribute keys, and its own `MemoryDomain`. Engagement scoring follows as a fourth domain — `EngagementEvent` per interaction, with typed signal fields (responded, responseTimeMs, responseLength, sentimentShift, reactionCount, continued) that are all nullable because not every platform supports every signal.

## What this opens up

The blocks Mood pattern can now compose `MoodState` with eidos dispositions — mood decays toward a personality-defined baseline using exponential decay, the same formula already used by `TemporalDecay` in the CBR chain. The StrategyLearning pattern can now record per-interaction engagement and detect trends via `TrendAnalyzer`. Neither pattern needed new infrastructure. They needed the right types in the right place with the right persistence story.
