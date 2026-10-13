---
layout: post
title: "Onboarding the household"
date: 2026-09-23
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [onboarding, keycloak, oidc, household, cbr-migration]
---

The question behind #116 was simpler than it looked: how does a family go from "installed the app" to "using it"? Until now the answer was Flyway seeds and REST calls — fine for demos, useless for anyone else.

The design landed on Keycloak Dev Services as the OIDC provider in dev mode. Real tokens, real roles, real tenant isolation — same code path as production. The key decision: household IS the tenancy. `Household.id` equals `tenancyId`, one-to-one, no indirection. A bootstrap admin in the realm config solves the chicken-and-egg problem (no users → can't authenticate → can't reach onboarding).

The `VoiceEnrollmentService` went in as a `@DefaultBean` no-op. The interface is there; #120 provides the real implementation when avatar voice detection is ready. The onboarding wizard has 5 steps and the settings view has 4 tabs — both Lit components following the existing patterns from the dock workbench work.

Most of the implementation time went to a different problem: the platform SNAPSHOTs had moved under us. The CBR types went through three names — `PlanCbrCase`, `ResolvedCase`, and finally `CbrPlanRecord`. `TrustGateService` moved packages. `WorkItemLifecycleEvent` moved to the API module. `Binding.humanTask()` became `.target()`. Each fix was mechanical but finding the right jar to check against took longer than the actual rename.

The discovery that surprised me: main doesn't test-compile. The JPA port that landed via slot 194 updated the entities but left test files referencing the old Panache static finders. The Vite build is also broken on main — a missing `@casehubio/yaml-core` dependency in pages-ui. Neither is caused by this branch, but both block `work-end` verification.
