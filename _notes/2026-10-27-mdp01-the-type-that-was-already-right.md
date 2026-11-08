---
layout: post
title: "The Type That Was Already Right"
date: 2026-10-27
entry_type: note
subtype: diary
projects: [casehubio/casehub-mosaic]
tags: [typescript, web-components, agency, type-systems, design-review]
series: issue-288-grounded-pattern-views
---

# The Type That Was Already Right

The previous entry left mosaic's pattern views grounded in real Java types — fourteen views, six categories, each reflecting the actual agency sealed interfaces. This entry covers what came next: an Execution/Reports tab architecture for all pattern views, and a type rework for CognitionConfig that almost went in the wrong direction.

The tab motivation is straightforward. Every pattern view renders configuration — what a system IS set up to do. None of them show runtime results — what the system DID. An Execution/Reports split in PatternViewMixin makes that boundary explicit. The Execution tab wraps the existing rendering. The Reports tab is a stub for now — the runtime report types don't exist yet in TypeScript — but the architecture is in place for when they do.

The implementation is minimal. PatternViewMixin gains `renderTabBar()` and `renderTabPanel()`, two override points (`renderExecution()` and `renderReports()`), and a reactive `_activeTab` state. Full ARIA compliance: roving tabindex, `aria-controls`, `aria-labelledby`, Home/End key support. The migration per view is mechanical — move the existing `render()` body into `renderExecution()`, add a five-line tab wrapper. Five views migrated with zero rendering changes.

The interesting part was CognitionConfig.

Mosaic-core had `CognitionConfig` with thirteen sections typed as `Record<string, unknown>` — opaque modules like `drive`, `mood`, `narrative`. The plan was simple: audit the Java source, replace the opaque types with real ones. I asked Claude to check neocortex's `CognitionConfig.java`. It came back with 23 boolean toggles — `moodEnabled`, `drivesEnabled`, `reflectionEnabled` — plus separate CDI config beans for each subsystem. The rework looked clean: replace the thirteen opaque sections with boolean toggles, add per-subsystem typed configs on the side.

The design review caught it.

Claude's reviewer read the spec and went straight to the Java source. The finding: I'd conflated two different Java types. `CognitionConfig.java` in neocortex is a CDI runtime toggle set — it enables and disables subsystems inside the running application. It never flows to the frontend. `CognitionDefinition` in agency's `agentic-yaml` module is the YAML schema type — thirteen optional `*ConfigSpec` sections that map directly to what a YAML file contains. The existing mosaic-core `CognitionConfig` with its thirteen sections was structurally correct all along. The opaque `Record<string, unknown>` types weren't the wrong shape — they were the right shape with the wrong depth.

The corrected rework kept the thirteen-section structure and typed each section's fields from the `*ConfigSpec` Java records. `MoodConfigSpec` got `baseline` (pleasure/arousal/dominance), `decayTimeConstant`, `maxDisplacement`. `DriveConfigSpec` got eleven fields including axis weights, modulation parameters, and affiliation thresholds. Thirteen interfaces in total, all matching `CognitionDefinition` field for field.

The cognition-view rework followed. The old view rendered generic cards with key tags — `baseLevel`, `decayRate` — pulled from opaque objects. The new view renders typed config fields with proper labels, Duration values highlighted, and axis weight bar charts for drive configuration. Sections show as configured or unconfigured. It looks different because it knows what the fields mean.

The lesson is one I keep relearning: when the existing type structure doesn't match your mental model of the Java source, check whether your mental model is wrong before replacing the structure. The thirteen-section `CognitionConfig` was modelled on the YAML definition, not the runtime toggles. The design review's adversarial pass caught the conflation before it shipped — a correction that would have been expensive to reverse once views were built against the wrong type.
