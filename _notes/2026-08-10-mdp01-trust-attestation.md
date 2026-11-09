---
layout: post
title: "Trust Attestation: Teaching the SOC to Learn from Its Analysts"
date: 2026-08-10
entry_type: note
subtype: diary
projects: [casehub-soc]
tags: [trust-scoring, attestation, bayesian-beta, implementation-routing, case-based-reasoning]
---

The SOC investigation pipeline has three worker capabilities — IOC enrichment, ATT&CK mapping, containment recommendation — each with a rule-based and an LLM implementation competing for the same slot. Until now, the engine had no signal for which to prefer. Both ran during bootstrap; whichever finished first wrote to context; the other's output was quietly discarded.

That changes with trust attestation. When an analyst resolves an incident, their decision — confirm severity, downgrade, escalate, or false positive — is now captured as a `LedgerAttestation` anchored to each worker's `WorkerDecisionEntry`. The platform's Bayesian Beta scoring reads these attestations and adjusts trust scores incrementally. Over many cases, workers that contribute to correctly-triaged incidents accumulate higher trust; workers whose pipelines produce false positives get penalised. The engine's `TrustWeightedImplementationRoutingStrategy` consumes these scores at routing time — no SOC-specific routing code needed.

The interesting design question was which dimensions to attest on. The original spec proposed four: triage accuracy, investigation thoroughness, containment effectiveness, and false positive rate. I cut it to two. Investigation thoroughness isn't evaluable from an analyst's binary decision — they see the combined pipeline output, not whether the IOC extraction was thorough. False positive rate is a derivable metric from triage accuracy's FLAGGED ratio, not a separate axis. The two survivors: `triage-accuracy` (was this a real incident?) and `containment-appropriateness` (was the containment recommendation proportionate?).

The verdict mapping forced a sharper question: what does DOWNGRADE mean for trust? The analyst is saying "real incident, wrong severity." That's SOUND for triage accuracy — the pipeline correctly identified something worth investigating. But it's FLAGGED for containment appropriateness — the recommendation was calibrated to a severity level that turned out to be overestimated. Two dimensions, differentiated verdicts for the same outcome. Without the second dimension, DOWNGRADE would be noise — too correct to penalise, too wrong to reward.

Claude's design review caught a material mistake in the spec. I'd proposed observing `CaseLifecycleEvent` directly — the raw CDI lifecycle event. The review pointed out that `CaseOutcomeObserver` exists as a purpose-built SPI for exactly this trigger. The existing `SocFaultedCaseReviewCreator` already implements it. `CaseOutcomeEvent` gives you `caseType` (clean namespace filtering), `caseFileSnapshot` as a `Map<String,Object>` (no JsonNode traversal), and `tenancyId` directly. Using the raw lifecycle event would have meant manual namespace string comparison and JsonNode navigation for no benefit.

The implementation also surfaced a naming collision worth noting. `SocCapabilities` defines `soc:`-prefixed agent-registration tags (`soc:host-isolation`, `soc:credential-revocation`). The case YAML uses unprefixed capability names (`containment-recommendation`). `WorkerDecisionEntry.capabilityTag` carries the YAML names. Filtering worker decisions against `SocCapabilities` constants would silently match nothing. The fix: a separate `SocCaseCapabilities` class for the YAML namespace — small but the kind of thing that burns an afternoon when you discover it empirically.

Trust scoring is the feedback loop that turns an investigation pipeline into an adaptive system. The attestations are the raw signal; the Bayesian Beta scoring is the mechanism; the routing strategy is the consumer. Each layer is owned by a different repo, each does one thing, and the SOC's only job is deciding what verdict to write. The platform handles the rest.
