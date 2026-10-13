---
layout: post
title: "The Spec That Skipped the Audit"
date: 2026-10-01
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [yaml-apps, design, gap-analysis, documentation-audit]
---

# The Spec That Skipped the Audit

The session started with a clear enough issue: design spec for 100% YAML CaseHub applications. Cross-repo gap analysis, critical path, three progressive demos. I jumped straight in — brainstormed the architecture, captured ten design decisions, wrote a 900-line spec covering app manifests, Maven packaging, project structure, topology, the lot.

The spec itself is solid. A YAML app is a standard Maven JAR project with a parent POM that pre-configures everything. `casehub-app-parent` pulls in scaffold-runtime (extracted from scaffold-backend), which handles YAML case definition loading, agent config, endpoint registration — the whole bootstrap. The user drops YAML files in conventional directories, runs `mvn quarkus:dev`, and gets a running app. Java is always under the hood but never leaks through the authoring surface unless you want it to.

Ten decisions lock down the architecture: hybrid parent POM with Quarkus uber-jar packaging, manifest plus convention-based discovery, dual-mode project layout, topology in the manifest, two-tier Java escape hatch, first-class scenario and playbook directories, Quarkus profile-based deployment separation. Three progressive demos build from a single-file hello-case to a multi-case coordinated-ops deployment.

The problem is what I didn't do. The slot has 32 repos. I examined four — engine, platform, scaffold, examples. The gap analysis declares "platform requires no changes" based on a 12% sample of the ecosystem. Eidos, blocks, desired state, ras, neocortex, qhorus, connectors, work, workers — all unexamined. Any of them could surface capabilities that a YAML app needs, APIs that should be documented, or architectural decisions that reshape the design.

The branch name was `platform-brief-prep`. The actual intent was: audit all repos first, sync their contributor and consumer guides, reconstruct ARC42STORIES and RAG-able API docs from git history, and only then produce a platform brief with talking points. The audit is the foundation; the brief is the output. I built the roof.

The design spec isn't wasted — it answers real questions about how YAML apps assemble and run. But it was premature. The right sequence was: understand everything the platform offers (audit), make that understanding current and accessible (documentation sync), then design how to expose it (the YAML app surface), then communicate it (the brief). I did step three without steps one and two.

Issue #107 captures the actual work: platform-wide guide audit, API doc generation, and executive brief. The #101 design spec and its Phase 1 implementation issues (#103-#106) are parked for after the audit delivers a complete picture of what the platform already has.

The lesson is straightforward. A 32-repo platform can't be designed from a 4-repo sample. The audit wasn't a bureaucratic prerequisite — it was the analytical foundation the design needed to stand on.
