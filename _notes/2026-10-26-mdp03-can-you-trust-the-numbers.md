---
layout: post
title: "Can You Trust the Numbers? — Model Validation for Bayesian Networks"
date: 2026-10-26
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
series: bjt-inference-engine
tags: [bayesian-networks, inference, validation, cross-validation, calibration, model-selection]
---

# Can You Trust the Numbers? — Model Validation for Bayesian Networks

The BJT engine can learn CPTs from data (`ParameterLearner`), track competing hypotheses, compose networks, and run influence diagrams. But all of that rests on a question nobody was answering: *is the learned model any good?*

You fit a network to data. It produces posteriors. The posteriors look reasonable. Ship it? No. Without validation, you don't know if the CPTs generalise beyond the training data. Without model selection, you don't know if a simpler network explains the data just as well. Without calibration, you don't know if "the model says 70%" means anything close to "this happens 70% of the time."

Three utilities now answer those questions. All pure Java, all static, all in `inference-bayes` with zero external dependencies.

## The validation stack

![Validation stack — log-likelihood at the base, cross-validation and model selection above, calibration at the top](images/validation-stack.svg)

Log-likelihood is the foundation. For each observation in the dataset, for each variable, look up the probability the model assigns to the observed outcome given the observed parent values. Sum the logs. Higher is better — a model that assigns high probability to observed events has higher log-likelihood.

The computation reads directly from CPTs. No inference needed. For a variable X with parents A, B:

```
ll += log P(X=observed | A=observed, B=observed)
```

Missing values are skipped — consistent with how `ParameterLearner` handles incomplete data. Zero probabilities produce negative infinity, which correctly penalises models that assign zero probability to events that actually happen.

This single method is reused by everything above it. Cross-validation scores each fold with it. BIC and AIC wrap it with a complexity penalty. There's one implementation, not three.

## Cross-validation: does it generalise?

The question log-likelihood alone can't answer: are you measuring fit or memorisation? A model with enough parameters can fit the training data perfectly and fail on new observations. Cross-validation tests generalisation by splitting the data into train and test partitions.

![5-fold cross-validation — each fold trains on 80% and tests on 20%](images/cross-validation-flow.svg)

The API takes a generic fitting function:

```java
var result = ModelValidator.crossValidate(
    structure, data,
    (s, d) -> ParameterLearner.fit(s, d, 1.0),
    FoldSplitter.kFold(10),
    42L);
```

The `BiFunction<BayesNetwork, List<Map<String, String>>, BayesNetwork>` parameter means this works with any model-fitting approach — `ParameterLearner` today, `StructureLearner` when it lands. The validation layer doesn't need to change.

`FoldSplitter` is a `@FunctionalInterface` with three strategies via static factories: `kFold(k)` for standard k-fold, `stratified(k, variable)` to preserve class distribution in each fold, `holdOut(fraction)` for a single train/test split. The seed parameter makes everything deterministic — same data, same seed, same folds.

The result carries per-fold scores plus mean and standard deviation. High variance across folds is a red flag: the model's performance depends heavily on which observations are in the training set, which usually means insufficient data or an overparameterised model.

## Model selection: is simpler better?

Two networks. Same data. Different structures — one has an edge A→B, the other doesn't. Which fits the data better *after accounting for complexity*?

Raw log-likelihood always favours the more complex model. More parameters means more freedom to fit the data. BIC penalises complexity:

```
BIC = −2 × logLikelihood + k × ln(n)
```

where `k` is the number of free CPT parameters and `n` is the number of observations. Lower is better. The `k × ln(n)` term grows with both parameter count and data size — BIC becomes increasingly hostile to unnecessary parameters as you collect more data.

AIC uses a lighter penalty (`2k` instead of `k × ln(n)`) and is less conservative. BIC is the default for `compare()` because overfitting sparse parent configurations is a real risk in Bayesian networks — a variable with 3 parents has 8+ parent configurations, and rare configurations get dominated by the Laplace prior rather than actual data.

Free parameter counting is automatic. For each variable: `parentConfigs × (outcomes − 1)`. The `-1` is because each CPT row sums to 1, so one entry per row is determined. A binary root has 1 free parameter. A binary child with a binary parent has 2. A ternary child with two binary parents has 8. The numbers add up fast, which is exactly why BIC matters.

## Calibration: when the model says 70%, does it mean it?

This is the question that matters most for the cognitive engine. When the appraisal pipeline says "there's a 0.7 probability this situation is threatening," and the agent's response changes based on that number, the number needs to be trustworthy. Not just correlated with outcomes — *calibrated*.

![Calibration plot — well-calibrated model hugs the diagonal, overconfident model deviates](images/calibration-bins.svg)

Calibration works by binning predicted probabilities and comparing to observed frequencies. For each variable, for each outcome, for each observation: what probability does the model assign to this outcome? Did it actually happen?

Group all those (predicted, actual) pairs into equal-width bins. In the 0.7 bin, the average predicted probability should be around 0.7, and about 70% of the predictions in that bin should have had actual=1. If the model says 0.7 and the outcome happens 50% of the time, the model is overconfident.

The Brier score summarises calibration in a single number — mean squared error between predicted probability and the binary indicator. It decomposes into three interpretable components:

- **Reliability** (calibration error) — how far the bins are from the diagonal. Lower is better. A perfectly calibrated model has reliability = 0.
- **Resolution** — how much the model's predictions discriminate between different outcomes. Higher is better. A model that always predicts 0.5 has zero resolution.
- **Uncertainty** — the base rate variance. Not under the model's control.

The decomposition gives you actionable diagnostics. High reliability with high resolution means the model discriminates well but its probabilities are miscalibrated — you might be able to fix it with temperature scaling. Low resolution means the model isn't learning useful distinctions from the data regardless of calibration.

## What this enables

With these three utilities, the workflow for building a Bayesian network from data becomes disciplined:

1. **Fit** the model with `ParameterLearner`
2. **Cross-validate** to check generalisation
3. **Compare** alternative structures with BIC to find the right level of complexity
4. **Calibrate** to verify the probabilities are trustworthy

When structure learning lands (#675), the same workflow applies — swap `ParameterLearner.fit()` for `StructureLearner.learn()` in the fitting function, and the validation pipeline doesn't change. That's why the generic `BiFunction` parameter exists: the validation utilities are agnostic to how the model was built. They care about what it predicts.
