---
title: Platform audit and vertical slice redesign
date: 2026-07-29
type: diary
project: casehub-soc
tags: [architecture, planning, platform-audit]
---

casehub-soc has been dormant since Epic 1 landed the domain vocabulary in late June. A month later, the platform underneath it has changed enough to warrant a full audit before resuming.

The audit covered pages, blocks-ui, blocks (Java), engine, and parent — GitHub history and issues alongside local code, because not all docs stay current. Three agents ran in parallel, each deep-diving one area.

What came back was significant. blocks-ui now has 29 web components across 7 domains — work-item-workbench, sla-indicator, approval-gate, trust-score-panel, audit-trail-viewer, channel-activity, and more. The engine landed the unified execution model (DagPlan, TaskNode, DecompositionStrategy) and completed the virtual threads migration. CBR matured with CbrCaseTypeRegistration as a CDI extension point. Pages shipped 13 TypeScript packages including push runtime, OKLCH design tokens, and Maven SNAPSHOT publishing.

The original 11-epic sequential plan was built before any of this existed. It had a separate dashboard epic, a separate CBR epic, a separate LLM fusion epic — all deferred to the distant future. With blocks-ui providing the UI layer, CBR infrastructure production-ready, and ImplementationRoutingStrategy landed for dual-agent routing, those separations no longer make sense.

The rewrite adopts vertical slices built layer by layer. First slice: SIEM critical alert, end-to-end across 6 layers — alert ingestion, dual triage workers (rule + LLM), analyst review with SLA, trust and routing, CBR retain/retrieve, compliance and audit. UI components wired alongside each backend feature as it lands. No separate dashboard phase.

The design review was sharp. Two adversarial agents found 40 findings between them. After cross-referencing against actual source code (several "phantom SPI" claims turned out to be search-scope errors — the agents searched casehub-blocks instead of casehub-blocks-ui), 16 concrete fixes survived verification. The most significant: worker CDI wiring was fundamentally wrong. @DefaultBean/@Alternative picks one winner — it does not give you two candidates for the routing strategy to choose between. The fix: two named Worker beans with two case YAML bindings per capability. ImplementationRoutingStrategy selects at the binding level.

Other verified corrections: LedgerEntry must extend JpaLedgerEntry with JOINED inheritance and domainContentBytes() override. CDI events are CaseLifecycleEvent (not fabricated CaseInstanceCreatedEvent). Trust model has five phases not four. AgentProvider.invoke() returns Multi<AgentEvent> — LLM workers need a blocking adapter. Case YAML fields inputSchema/outputSchema are deprecated in favour of inputProjection/outputProjection.

Best discovery of the session: SituationStore was listed as the primary platform blocker for Layer 1. Turns out InMemorySituationStore AND JpaSituationStore both exist in casehub-ras. The gap issue we filed earlier in the session was closed hours later. Layer 1 is not blocked.

Four genuine platform gaps were filed: Drools CEP for event correlation (engine#809), multi-approver OversightGateService (engine#810), durable EventStore for pages-push (pages#256), and the now-closed SituationStore (parent#398). The multi-approver gate is 90% built — all three layers exist independently (engine risk classifier, work M-of-N counting, blocks-ui approval-gate with QuorumConfig), just needs a connector service.

Layer 1 is ready to build. Branch created, plan written, no blockers.
