---
layout: post
title: "Three Lies Your UI Tells"
date: 2026-11-24
entry_type: note
subtype: diary
projects: [casehubio/casehub-mosaic]
tags: [css-tokens, design-system, accessibility, aria, web-components, diagram]
---

# Three Lies Your UI Tells

Three fixes landed today. Each one corrected a component that
reported incorrect state — to the browser's rendering engine,
to its accessibility tree, or to its own data model.

## The colour that didn't exist

The diagram toolbar mapped nine diagram types to hardcoded hex values.
Blue for choreography, purple for HTN, amber for GOAP, and so on. The
colours worked in light mode. In dark mode they sat on a dark background
with contrast ratios that ranged from poor to invisible.

The fix was to migrate to `--pages-*` CSS tokens — the same light-background,
dark-text pattern that `pages-badge` already uses. The interesting part is
the mapping. The pages token system has six semantic categories: accent,
info, success, warning, danger, neutral. The toolbar needs seven visually
distinct colours. Something has to share.

`llm` and `playbook` both land on `success`. Green is close enough to teal
when the badge carries a text label, and the alternative is inventing a
seventh semantic category for one toolbar widget. When a design system
doesn't have enough resolution for a specific use case, the right answer
is usually to accept the collision, not to extend the system.

## The type nobody reads

The diagram workbench had `diagramType: 'choreography'` hardcoded for
its root level. The detection functions already existed —
`detectDiagramTypeFromSrc` reads the type from the filename convention,
`detectDiagramType` parses YAML content. The main diagram component
used both. The workbench didn't.

The twist: the root level currently renders through `<casehub-diagram>`,
which does its own detection internally. The hardcoded value sits in a
`DrillDownLevel` object that no rendering path reads for the root case.
The fix corrects data, not behaviour. Today. Tomorrow, when the
collapsed-strip sidebar shows type badges or the rendering path changes,
the wrong value would surface as a bug with no obvious cause — wrong
default, right tests, three months of commit history between introduction
and discovery.

Defensive correctness matters most when nothing fails.

## The label that lied to assistive technology

The confirmation card had a static ARIA label: "Confirmation required"
regardless of whether the user had already approved, declined, or let it
expire. Visual users see a "Confirmed" or "Expired" status badge.
Screen reader users heard "required" until the card left the DOM.

The fix is four ternary branches. The design question is subtler: the
visual status text says "Confirmed" (terse, state-like), but the ARIA
label says "Confirmation approved" (natural speech). Different modalities
want different phrasing for the same state. A screen reader user hears
the label read aloud — "approved" flows as spoken language in a way that
"confirmed" as a bare adjective doesn't.

## The pattern

None of these were bugs in the traditional sense. The components rendered.
The tests passed. The features worked. What they had in common: each one
said something that wasn't true. A colour that couldn't adapt. A type
that was always choreography. A label that was always "required". Small
lies, invisible to sighted users in light mode with a mouse — exactly
the conditions under which most components get tested.
