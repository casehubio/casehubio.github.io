---
layout: post
title: "Two Registries and a Broken Contract"
date: 2026-09-04
entry_type: article
subtype: diary
projects: [casehubio/fsitrading]
tags: [pages, blocks-ui, compliance, gdpr, ui-panels]
series: issue-23-knowledge-compliance
author: mdp
---

# Two Registries and a Broken Contract

The C6 epic had three layers: teach the system to remember incidents (CBR pipeline), give it compliance and governance tooling (post-mortem, GDPR, regulatory grid), and surface all of it through the Ops Centre UI. The first two were architecture. The third should have been plumbing.

It was — but the plumbing exposed two problems that would have been invisible until runtime.

## The Panel Registry Gap

The casehub-pages framework has a `hostPanel("case-explorer")` DSL that slots blocks-ui web components into a dock workbench layout. Importing a blocks-ui component registers it as a custom element via Lit's `@customElement` decorator — `customElements.define('blocks-case-explorer', CaseExplorer)`. That's the browser's built-in registry.

But the pages runtime uses a separate `panelRegistry` — a plain `Map<string, string>`. When the activation code encounters a `host-panel` component, it calls `lookupPanel(typeName)`. If the type isn't registered, it prints "Unknown panel type" and renders nothing. No error thrown. No stack trace. Just a blank panel with a console warning.

Every `hostPanel()` call in both the Trading Desk and Ops Centre pages was broken. Thirteen panels, all silently failing. The code compiled, the build passed, and every panel would render empty at runtime.

The fix is one line per panel:

```java
registerPanel("case-explorer", "blocks-case-explorer");
```

Import registers the custom element with the browser. `registerPanel` registers it with the pages runtime. Two registries, both required, no documentation connecting them. The AML app had it right — I found the pattern by reading its `index.ts`.

## The DTO Boundary

The second problem was subtler. The blocks-ui components expect specific JSON shapes. The `compliance-summary` component expects `{ regulation, requirement, mechanism, status }` with status values `MET | PARTIAL | GAP | BREACHED`. The backend's `ComplianceResource` returns `TrustRoutingRequirement` — a platform type with different field names (`requirementId`, `citation`) and different status values (`CLOSED` instead of `MET`).

Neither side is wrong. The platform type serves the platform's domain model. The UI component serves the user's mental model. The gap lives at the boundary.

`ComplianceStatusRecord` bridges it: splits citation on " — " to extract regulation and requirement, maps CLOSED→MET, and returns the shape the component expects. A thin DTO, but the kind of thing that's invisible until a user clicks a tab and sees an empty grid.

The GDPR endpoint had a sharper version of the same problem. The `gdpr-erasure-action` component POSTs `{ subjectId, reason }`. The backend expected `{ traderId, reason }`. Jackson silently deserializes `subjectId` as null for a field named `traderId`, the validation rejects it, and the user gets a 400 with no explanation. Claude caught this during the coherence review — the kind of cross-layer mismatch that only surfaces when you read the TypeScript component source and the Java record side by side.

## What Closed

C6 is done. The Ops Centre now has four knowledge panels alongside the incident and approval panels: similar incidents from CBR retrieval, the five-requirement compliance grid, GDPR erasure with confirmation, and the audit trail viewer. The backend work from prior sessions — CBR pipeline, post-mortem bridge, compliance evaluators, erasure orchestration — feeds directly into these panels.

The follow-up issue tracks the gaps: the similarity endpoint doesn't yet use the selected case's features for the query (it does a generic CBR scan), the paginated history endpoint isn't built, and a few CBR integrations still need platform API support. Known gaps, not forgotten ones.
