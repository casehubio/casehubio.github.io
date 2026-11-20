---
layout: post
title: "The graph that forgot its edges"
date: 2026-11-20
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [knowledge-graph, extraction, edge-inference, co-occurrence, consolidation, mindmap-intelligence]
series: issue-547-edge-extraction-scaling
---

# The graph that forgot its edges

Better entity extraction made the knowledge graph worse. That's the counterintuitive finding that #547 quantified: the edge/node ratio dropped from 1.20 to 0.76 as entity extraction improved. Entities up 64%, edges up 3%. The graph was filling with isolated nodes — entities mentioned in conversation or biography but connected to nothing.

## Why improving extraction makes graphs sparser

The extraction pipeline — both the Python biographical pipeline and the Java runtime `MindMapExtractor` — processes text in passages. Each passage gets its own LLM call. Entities and edges are extracted together, within the context of that passage.

The old pipeline found fewer entities per passage, so the entities it did find were densely connected — they were the main characters of that passage, and their relationships were obvious. As extraction improved and started pulling out secondary entities (places mentioned in passing, organisations named once, goals implied by action), the new entities arrived without the relationship context that would connect them to the rest of the graph. The LLM never saw them in the same prompt as the entities they relate to.

This is a structural isolation problem: entity A appears in passage 3, entity B appears in passage 7, and no extraction call ever sees both. Cross-passage edges don't get created because no prompt contains both endpoints.

## Co-occurrence as a signal

The fix is a post-extraction edge inference pass. The question was how to identify which entity pairs are worth asking the LLM about — sending all N² pairs is too slow for a graph with 276+ entities.

Memory co-occurrence turned out to be a strong signal. When the extraction pipeline creates memories (experience records, relationship records), it tags them with `subject` and `related_entities` attributes. Two entities that share a memory — appear as subject and related entity in the same record — probably have a relationship, even if they were never in the same extraction prompt.

The algorithm builds a co-occurrence matrix from these memory attributes, filters out pairs that already have edges, ranks by co-occurrence count, and sends the top candidates to the LLM with their shared context. The LLM's job is narrow: given this pair of entities and the contexts where they co-occur, what relationship type fits? It doesn't need to discover the entities or wonder whether they're related — the co-occurrence already established that.

## Two pipelines, same fix

The biographical extraction pipeline in Python and the runtime consolidation pipeline in Java both suffer from the same structural isolation. I implemented the same pattern in both.

**Python (`extract_structure.py`):** A new `infer_cross_passage_edges()` function runs after all per-passage extraction completes. It builds co-occurrence from the `related_entities` fields in extracted memories, sends candidates directly to the Anthropic API (bypassing the per-passage `call_llm` wrapper, which added noise like "Passage:" framing), and merges inferred edges into the dedup set. Configurable via `--edge-inference-top-n`.

**Java (`EdgeInferencePhase`):** A new consolidation phase at `@Priority(22)` — after `ExperienceConsolidation` creates the experience nodes that feed co-occurrence, before `MergeDetection` merges duplicate nodes. It scans for under-connected nodes (degree below a configurable threshold), builds co-occurrence from `CaseMemoryStore`, and uses `ParallelLlmExecutor` for the LLM call. Opt-in via `@IfBuildProperty`, disabled by default.

## Exact matching matters

The first implementation used `String.contains()` to check whether a node name appeared in a memory's subject or related_entities field. A node named "Art" matched "Arthur". A node named "San" matched "San Francisco" and "Sandra". The co-occurrence matrix was full of false positives.

The fix: split `related_entities` by comma, lowercase-normalize each part, and check exact set membership against the under-connected node names. No substring matching. The biographical pipeline already did this correctly — it parsed `related_entities` as a list from the start.

## What this doesn't catch

Co-occurrence only finds edges between entities that appear together in memories. Two entities that never appear in the same memory but are conceptually related — two art movements from different passages, two cities that share a cultural link — won't show up as candidates. The existing convergence extraction step handles some of these through synthesis-level inference over generated memories. For the rest, the graph will need embedding-based similarity or a periodic full-graph review pass. Those are future work.

The immediate target is pragmatic: get the edge/node ratio back above 1.0. The co-occurrence signal is the low-hanging fruit — most missing edges are between entities that the memories already connect, just not explicitly.
