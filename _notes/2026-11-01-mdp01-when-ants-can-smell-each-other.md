---
layout: post
title: "When Ants Can Smell Each Other"
date: 2026-11-01
entry_type: note
subtype: diary
projects: [casehubio/engine, casehubio/eidos]
tags: [stigmergy, swarm, eidos, agent-graph, coordination, architecture]
series: issue-1274-hive-mind-integration-wiring
---

# When Ants Can Smell Each Other

Stigmergy works because individuals don't need to know each other. An ant following a pheromone trail doesn't care who deposited it. A wiki editor fixing a paragraph doesn't coordinate with whoever wrote it. The environment carries the coordination signal, and each participant responds to what they find, not to who left it.

CaseHub's swarm coordination pipeline works the same way. Agents deposit signals, observe patterns, and respond to environmental state. The `SignalRegistry` accumulates consensus. The `StigmergyCoordinator` detects patterns — signal consensus, coordination storms, interest convergence. The `TeamDetector` clusters agents by behavioral fingerprint — shared interests, shared signals, complementary effects. None of these components know which specific agents are involved. They operate on anonymous patterns in shared state.

This is a strength. Anonymous coordination scales. It doesn't break when you add agents. It doesn't require a central dispatcher. But it has a blind spot: it can't see relationships.

## The Blind Spot

Consider provisioning. A swarm needs more capacity. The `SwarmProvisioner` checks budgets, consults an advisor, and provisions a new agent. At no point does it ask: are there agents already active with overlapping capabilities? Has any agent in this tenancy proven effective at this kind of task before?

Consider team detection. The `TeamDetector` computes an affinity matrix between all active agents — three factors averaged equally: shared observation interests, shared signal names, and complementary effect keys. It's a reasonable heuristic. But it's entirely behavioral. If two agents have collaborated successfully across dozens of prior cases, that history doesn't appear anywhere in the affinity score.

Consider activation. When an agent joins a swarm and transitions to ACTIVE, the coordinator records the state change and moves on. The newly activated agent has no idea who else is working in the same case. Its view of the world starts blank.

Each of these gaps has the same shape: the stigmergic system processes *what agents do* but has no access to *who agents are* relative to each other.

## What Eidos Brings

Eidos maintains an agent graph — a relational model of agents, their capabilities, their task histories, and their collaboration patterns. Two interfaces carry the data the swarm needs:

**`AgentGraphQuery`** answers questions about agent proximity. `coActiveAgents(externalRef, tenancyId)` returns which agents are currently active for a given reference. `topAgentsByOutcome(capabilityTag, taskDomain, tenancyId, limit)` ranks agents by their track record — Wilson lower-bound scoring over outcome history, so agents with zero observations sort last rather than getting lucky with a small sample.

**`RuntimeCollaborationQuery`** answers questions about collaboration history. `collaborators(agentId, tenancyId)` returns a list of agents this agent has worked with, each scored with an affinity value and tagged with the nature of the relationship — coactive, shared interest, shared signal, complementary.

Both interfaces were fully implemented in eidos. Both had tests. Neither was called from anywhere in the engine.

![Stigmergic coordination pipeline with eidos enrichment points](images/eidos-stigmergy-enrichment.svg)

## Three Injection Points

The wiring is straightforward. Each swarm class gains a nullable constructor parameter for the relevant eidos interface. When eidos is deployed, the queries enrich behavior. When it isn't, every call site skips gracefully and the system behaves exactly as before.

**SwarmProvisioner — before provisioning.** After budget checks and advisor consultation, the provisioner now queries `coActiveAgents` to see what's already active. It also checks for capability-tagged consensus signals (`swarm:need-capacity:nlp:case-processing`) and queries `topAgentsByOutcome` to rank candidates. This doesn't change the provisioning decision today — the provisioner still provisions a generic swarm agent — but it creates the data path for capability-aware provisioning when that arrives.

**StigmergyCoordinator — on activation.** When an agent transitions from JOINING to ACTIVE, the coordinator queries its co-active neighbors. This is logged for observability now; the important thing is that the data is available at the moment when the agent's operational context is being established.

**TeamDetector — in affinity scoring.** This is where eidos makes an immediate difference. The existing affinity matrix averages three behavioral factors. Eidos collaboration data arrives as a fourth factor. The blending formula preserves the existing 3-factor average when no eidos data exists, and introduces collaboration affinity as an equally-weighted fourth signal when it does:

```java
if (eidosAffinity > 0.0) {
    affinity[i][j] = (affinity[i][j] * 3.0 + eidosAffinity) / 4.0;
    affinity[j][i] = affinity[i][j];
}
```

Agents who have collaborated before get a boost in team clustering. The boost is proportional — a strong collaboration history can tip two agents into the same team cluster even if their current behavioral fingerprints don't strongly overlap. This matters in practice because behavioral fingerprints are session-scoped. Collaboration history spans cases.

## Why This Isn't Just Wiring

The interesting thing about this integration isn't the code. It's the architectural boundary it establishes between stigmergic coordination and graph intelligence.

Stigmergy operates on *anonymous environmental patterns* — signals, observations, behavioral fingerprints. It doesn't need to know identity. Eidos operates on *identified relational knowledge* — who collaborated with whom, who has a strong track record for this capability, who is co-active right now. It doesn't process environmental signals.

These are fundamentally different coordination models. Stigmergy is bottom-up — structure emerges from local interactions with shared state. Graph intelligence is top-down — structure is queried from a pre-existing relational model. Neither is sufficient on its own. Bottom-up emergence misses obvious relational cues. Top-down querying can't detect emergent patterns that aren't in the graph yet.

The wiring we landed doesn't merge these models. It supplements one with the other. The stigmergic pipeline stays in control — it still detects teams, still tracks roles, still drives provisioning. Eidos graph data is one more input, weighted equally with the behavioral factors. The swarm's emergent behavior gets informed by relational context, but it isn't overridden by it.

Whether this balance is right is an open question. I suspect that for mature swarms — cases where agents have deep collaboration histories — the graph data should carry more weight than the behavioral fingerprint of the current session. For novel swarms with no history, the behavioral fingerprint is all you have. A per-case weighting that adapts to graph data availability would be more interesting than a fixed 1:3 ratio. That's future work.nue
