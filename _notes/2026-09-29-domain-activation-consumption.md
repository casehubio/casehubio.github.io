---
layout: post

title: "Domain Activation — Teaching Agents Their Emotional Geography"
date: 2026-09-29
entry_type: note
subtype: diary
series: cognitive-intelligence
projects: [casehubio/blocks, casehubio/neocortex]
author: mdp
tags: [cognitive-systems, domain-activation, dtw, affect-correlation, blocks]
---

# Domain Activation — Teaching Agents Their Emotional Geography

Neocortex already knows which knowledge domains correlate emotionally — DomainActivation computes pairwise DTW on time-bucketed PAD series, runs surrogate significance tests, and can tell you that an agent's emotional state in "work projects" tracks closely with "family relationships." What it couldn't do was tell the *agent* that. The data lived in neocortex with no path to the prompt.

The shape of the problem was clear from the start: consume `DomainActivation.correlate()`, cache the results, render them as natural language the agent can reason about. The implementation pattern existed — SocialComparison landed the previous session using `CognitiveProfileParticipant` at the TERMINAL phase with a section customizer. Follow that template, done.

Except the template was wrong for this case.

## The Mediator Trap

The first design proposed a `DomainActivationMediator` — an `@ApplicationScoped` CDI bean observing `ConsolidationCompleted` events, caching results, and making them available on prompt assembly. Clean analogy to `ConsolidationMediator`, which does exactly this for consolidation artifacts.

The decision review caught the flaw: ConsolidationMediator is a zero-computation event cache. It receives pre-computed artifacts bundled with the CDI event and stores them. DomainActivation requires heavy computation — DTW with 200-iteration surrogate significance tests across subgraph pairs. Running that in a CDI `@Observes` method blocks the event dispatch thread.

Worse: `ConsolidationCompleted` carries only `tenantId`. `DomainActivation.correlate()` requires a `PrincipalId` — it's agent-scoped, because mood and experience memories are per-agent. The event has no agent identity.

The fix was a tick participant with a consolidation-dirty-flag. On `ConsolidationCompleted`, the existing `ConsolidationMediator` records a timestamp. The new `DomainActivationParticipant` checks that timestamp against its own last-computation time on each tick. If consolidation happened since the last computation, recompute. If not, return cached. The tick provides agent identity through `CognitionTickContext.agentId()`. Computation runs on the tick thread, not the event thread.

One new method on `ConsolidationMediator` — `lastConsolidationTimestamp(String tenantId)` — read-only, doesn't affect the existing drain mechanism. The participant is a plain class (not a CDI bean), matching `CognitiveProfileParticipant`.

## The Cascade

DomainActivation returns `Optional.empty()` if any queried subgraph has no entities or memories. With all-pairwise queries, one empty subgraph poisons the entire result. We query pairwise — each call uses exactly two subgraph IDs, so an empty subgraph only kills pairs involving it.

The expensive part is context correlation — mood-affect DTW with significance testing (200 surrogate iterations per pair) and experience-event impact analysis with bootstrap confidence intervals. We don't need those for pairs that won't render. So: DTW scan first with no context domains (cheap), filter to MODERATE or stronger (similarity >= 0.4), then context correlation only for surviving pairs. Typically 1-3 pairs survive out of the full pairwise set, so the cascade avoids most of the expensive computation.

## What the Agent Sees

The prompt section renders narrative, not numbers. SocialComparisonPromptSection uses numeric distances ("distance 0.57") because it's comparing specific agent-pair interactions in real time. Domain activation correlations are time-windowed aggregates — the exact DTW similarity score isn't actionable. What matters is the strength category and the direction.

```
Your emotional patterns across knowledge domains:
- Your emotional states in "work projects" and "family relationships"
  are strongly correlated — when one domain activates emotionally,
  the other tends to follow. Your mood is significantly associated
  with activity in work projects.
```

Mood correlation only appears when statistically significant (p < 0.05). Event impacts only render when the confidence interval doesn't span zero. An agent won't be told "collaborative events increase your pleasure" unless the evidence supports it.

## What This Opens Up

Domain activation gives agents a kind of emotional self-awareness that's hard to script. An agent that knows its work-domain and family-domain emotions track together might approach a stressful work conversation differently — or at least reference the pattern when reflecting. Combined with the SocialComparison and CognitiveProfile work from the previous session, agents now have three layers of cognitive self-knowledge feeding their prompts: what they know about entities, how they differ from other agents, and how their emotions move across domains.

The deferred piece is worth noting: `DomainActivation.correlate()` fails entirely if any queried subgraph is empty. The pairwise workaround handles this, but a proper fix — partial results from N-way queries — would eliminate the cascade's query multiplication. That's a neocortex change, tracked separately.
