---
layout: post
title: "From Strength to Distribution: Generalising Noisy-OR to Noisy-MAX"
date: 2026-10-26
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [bayesian, inference-bayes, noisy-max, cpt-estimation]
series: issue-678-noisy-max-generalisation
---

# From Strength to Distribution: Generalising Noisy-OR to Noisy-MAX

Noisy-OR has been doing the heavy lifting for LLM-elicited CPTs since the NetworkElicitor landed. Ask an LLM "how strongly does smoking cause lung cancer?" and you get a single number — a strength. That strength, combined with a leak probability, expands into a full conditional probability table without the LLM needing to specify every row of a potentially enormous truth table.

The constraint: binary children only. The child is either caused or absent, so a single strength per parent is enough to describe the causal influence. Variables with three or more outcomes — say, severity levels (severe, moderate, absent) — couldn't use this shortcut. The LLM had to produce a full direct CPT, which scales exponentially with parent count.

Noisy-MAX (Díez & Druzdzel 2007) removes this constraint.

## The shift: strength becomes distribution

![Noisy-OR vs Noisy-MAX — single strength per parent becomes a full distribution](images/noisy-max-vs-or.svg)

The key insight is simple. Instead of asking "how strongly does this parent cause the child?", ask "what distribution over outcomes does this parent produce when acting alone?" Each parent contributes a full distribution over the child's K outcomes rather than a single scalar.

For a three-outcome child (severe, moderate, absent), parent A might contribute `[0.6, 0.3, 0.1]` — a 60% chance of severe, 30% moderate, 10% absent when A is the only active cause. The leak distribution `[0.05, 0.10, 0.85]` says what happens with no active parents at all.

The combination rule works through cumulatives. Define Q_i(k) as the cumulative probability of outcome k or more severe, for parent i acting alone. For a set of active parents S (including leak), the joint cumulative is:

```
P(child ≤ k | S) = 1 - ∏_i (1 - Q_i(k))
```

Each parent independently tries to push the child toward more severe outcomes. The MAX takes the worst case — if *any* active parent would produce a severe outcome, the child is likely severe. Individual probabilities are then recovered by differencing consecutive cumulatives.

## Why this matters for LLM elicitation

The exponential wall in CPT estimation isn't the number of parents — it's the number of *outcome combinations*. A child with 3 outcomes and 4 binary parents needs a 48-entry CPT (16 rows × 3 columns). With Noisy-MAX, the LLM specifies 4 distributions of length 3 plus a leak — 15 numbers instead of 48. The gap widens fast: 5 parents and 4 outcomes goes from 128 entries to 24.

The NoisyOrExpander now auto-routes. If a NoisyOr spec is given for a variable with 3+ outcomes, it converts the single strength to a distribution (strength at index 0, remainder at the last index) and delegates to NoisyMaxExpander. Existing networks with binary variables are unaffected — NoisyMax with K=2 produces identical output to NoisyOr.

## The reduction property

This was the critical correctness test. A NoisyMax with two-outcome distributions `[s, 1-s]` and leak `[l, 1-l]` must produce exactly the same CPT as NoisyOr with strength `s` and leak `l`. The cumulative formula collapses:

```
Q_i(0) = s_i
P(child=caused | S) = 1 - ∏_i (1 - s_i)
```

Which is the standard Noisy-OR formula. The test verifies bit-for-bit agreement to 1e-10 precision.

The NetworkElicitor can now produce `"type": "noisy_max"` in its JSON responses, with per-parent distributions and a leak array. For binary variables it still emits `"noisy_or"` — the ResponseParser handles both, and the auto-routing means existing specs work transparently with multi-valued children.
