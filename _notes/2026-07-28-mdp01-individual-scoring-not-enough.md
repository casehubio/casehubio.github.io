---
layout: post
title: "Why Your Multi-Agent System Doesn't Remember Who Works Well Together"
date: 2026-07-28
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks]
tags: [cbr, routing, coordination, multi-agent, team-composition]
---

Most multi-agent systems route tasks to agents based on individual performance. Agent A handled this type of task well last time — send it to Agent A again. This works for single-step tasks. It falls apart the moment you have multiple agents working on the same case.

The reason is straightforward: individual competence doesn't predict team outcomes. An agent with a 0.9 success rate on analysis tasks and another with 0.85 on review tasks can fail together — their output formats clash, their approaches conflict, or they make incompatible assumptions about shared context. Swap one team member and the same case succeeds. The routing system that picked these agents has no idea, because it scored them independently.

This is the team composition problem. It shows up in any multi-agent orchestration system where the router picks agents one at a time without considering who else is already assigned. The information exists — every completed case records which agents worked together and whether the case succeeded — but no one queries it as a team signal.

## What the CBR literature offers

Case-Based Reasoning has been reasoning about teams for longer than LLM agents have existed. The most directly relevant work is [FlexiTeam](https://ceur-ws.org/Vol-3389/ICCBR_2022_Workshop_paper_90.pdf) (Mathew, Bergmann et al., ICCBR 2022), which extends Process-Oriented Case-Based Reasoning with resource nodes — workers attached to workflow tasks, carrying skill profiles and experience metadata. Their scoring function blends task similarity with team similarity. The case is the whole team+task, not fragments of it.

The [systematic review of CBR for LLM Agents](https://arxiv.org/abs/2504.06943) (Hatalis et al., 2025) maps the current landscape and identifies multi-agent CBR architectures as an open gap. Individual agent memory is well-studied — episodic memories, case banks, trajectory stores. Team memory is not. The review formalises the CBR cycle (retrieve, reuse, revise, retain) for LLM agent contexts, but the formalisations assume a single agent.

[Two-Agent CBR / EAGR](https://www.researchgate.net/publication/393158642) introduces a concept that turned out to be central to the design: adaptation-guided retrieval. Standard CBR retrieves the most *similar* case. AGR retrieves the most *adaptable* — the case whose solution can be most feasibly reused for the current problem. For team routing, this translates to: don't just find the team that succeeded on a similar task — find the team that can be re-assembled *now*, from the agents currently available.

Two more papers shaped the thinking. [CAST](https://arxiv.org/abs/2605.15041) (Pang et al., 2026) extracts complexity and failure profiles from past execution trajectories, calibrating how much reasoning effort an agent should invest. [Memento](https://arxiv.org/abs/2508.16153) (Zhou et al., 2025) uses non-parametric episodic memory with a neural case-selection policy, achieving 4.7-9.6% gains on out-of-distribution tasks. Both treat the complete execution as the unit of analysis.

## The design that survived adversarial review

I started with a new case type — `CoordinationCbrCase` — stored alongside existing plan cases, with a signal provider that queries the CBR store directly. An adversarial design review tore this apart on three points, and the design that emerged is better for it.

**The data already exists.** Every completed case already stores a `PlanCbrCase` with plan traces recording which agent handled which step. Team composition is extractable from what's already retained — worker names across the trace. What was missing was a team-centric *query*, not a team-centric *case*. Adding a `team_members` feature (as a `CategoricalList` — neocortex's native set-overlap similarity type) to existing cases at retention time gives team-centric retrieval through the same pipeline that handles problem-centric retrieval.

**One pipeline, two query angles.** Rather than injecting the CBR store directly (which would bypass the decorator chain — outcome weighting, trust weighting, scope decay, retrieval tracking), the design extends the retrieval service to run a second query with different feature weights: team composition weighted heavily, problem features weighted lightly. Same decorators, same caching, same lifecycle. No architectural split.

**Retention-first design.** The reviewer caught that I'd designed retrieval first and left retention as a follow-up. This is backward — the retention mechanism determines what data exists and when. The existing `CbrCaseRetainObserver` already fires on case completion with full access to plan traces. Extracting team features from data it already has is additive, not structural.

## How the scoring works

The `CoordinationSignalProvider` implements `RoutingSignalProvider` — the same SPI as the existing `PlanCompositionAnalyser`. It composes with the routing pipeline via `RoutingSignalAssembler`, discovered by CDI alongside other signal providers. No strategy replacement, no pipeline changes.

For each retrieved experience with two or more distinct agents in the plan trace, the provider scores every eligible candidate by team affinity — a similarity-weighted mean of case-level outcomes across experiences where that candidate was part of a multi-agent team.

The adaptation-guided retrieval layer multiplies each experience's weight by team re-assembly feasibility: the fraction of the historical team that is currently available in the candidate pool. An experience where 3 of 4 original team members are candidates scores higher than one where only 1 of 4 are — even if the latter had a better outcome. This is the [AGR principle](https://arxiv.org/abs/1905.12464) applied at scoring time rather than in the retrieval infrastructure, because it needs the candidate list, which is routing context the decorators don't have.

When no coordination experiences exist, the provider returns null and the assembler skips it. When they do exist, coordination is one signal alongside individual competence, trust, and load — weighted and combined, not overriding anything.

## Why this matters beyond CaseHub

Any system that orchestrates multiple AI agents on a shared task has this problem. The agent that's best individually may be worst in a particular team context. If your routing doesn't capture team composition effects, it will keep reassembling teams that fail together while never learning from teams that succeed together.

The specific mechanism — CBR with set-overlap similarity on team membership, weighted by adaptation feasibility — is one approach. The principle is more general: agent routing should treat team composition as a first-class retrieval dimension, not an afterthought derived from individual scores.

The research gap is real. The [CBR for LLM Agents review](https://arxiv.org/abs/2504.06943) identifies it explicitly. FlexiTeam addresses it for human teams in process management. What we've built here brings it to AI agent orchestration — and the architecture (enriching existing cases with team features, multi-angle retrieval on the same store, adaptation-guided scoring) is reusable by anyone with a CBR-backed agent routing system.

Communication patterns — message counts, convergence metrics, common ground state — are deferred to v2. The feature schema is designed for extension: adding new coordination dimensions is additive to both the feature schema and the retention path. No migration, no new case type, no separate lifecycle.

Sources:
- [FlexiTeam: POCBR for Team Organization (ICCBR 2022)](https://ceur-ws.org/Vol-3389/ICCBR_2022_Workshop_paper_90.pdf)
- [CBR for LLM Agents Review (arXiv 2504.06943)](https://arxiv.org/abs/2504.06943)
- [Two-Agent CBR / EAGR (ResearchGate)](https://www.researchgate.net/publication/393158642)
- [CAST: Case-Based Calibration (arXiv 2605.15041)](https://arxiv.org/abs/2605.15041)
- [Memento: Memory-Augmented Agents (arXiv 2508.16153)](https://arxiv.org/abs/2508.16153)
- [AGR via Graphical Models (arXiv 1905.12464)](https://arxiv.org/abs/1905.12464)
- [ADR 0001: CBR Coordination Memory Architecture](https://github.com/casehubio/blocks/blob/main/docs/adr/0001-cbr-coordination-memory-architecture.md)
