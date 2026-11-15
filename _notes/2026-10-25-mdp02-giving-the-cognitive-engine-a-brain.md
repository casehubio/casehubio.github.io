---
layout: post
title: "Giving the Cognitive Engine a Brain"
date: 2026-10-25
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [visualization, three-js, cognitive-engine, brain-layout, design]
series: issue-691-brain-graph-layout
---

# Giving the Cognitive Engine a Brain

The cognitive workbench has had a 3D graph view since #580 — an ego-centric sphere layout where you BFS from a focal node and everything radiates outward. It works for exploring a neighborhood, but it tells you nothing about *where* things sit in the cognitive architecture. A person node and a goal node and a CAPS behavioral pattern all float in the same undifferentiated sphere. The spatial arrangement is arbitrary.

I wanted the shape itself to communicate what the system is. A brain.

## The Region Mapping

The idea is straightforward: subgraph types already represent functionally distinct cognitive subsystems. Map each one to a brain region based on rough neuroanatomical parallels, seed the nodes there, and let force-directed layout settle within the constraint.

Nine regions. PERSON and ORGANISATION nodes go to the temporal-parietal junction (social cognition). GOALs go to the prefrontal cortex (planning and executive function). ACTIVITY and PLACE nodes go to the hippocampal region (episodic memory). CAPS behavioral patterns sit in the basal ganglia (habitual response). And so on.

This mapping is suggestive, not a neuroscience claim. But it immediately gives spatial semantics to what was previously random placement. When you see an edge running from the temporal-parietal region into the prefrontal cortex, you're watching a social entity connect to a planning goal — and the arc across the brain tells that story visually.

## The Part I Didn't Expect

The limbic region — emotional state — doesn't correspond to any subgraph type. Mood and affect live as memories and PAD values on nodes, not as their own graph structure. I initially proposed three options: ambient glow, synthetic nodes, or drop the row entirely.

The answer turned out to be both synthetic node types. Mood snapshots — recent mood memories materialised as nodes showing the emotional trajectory over time. And affect projections — shadow nodes for entities from other regions that carry strong emotional weight, connected back to their originals by edges. A person node with high arousal in the social region gets a faint shadow in the limbic region, and the connecting edge shows the emotional association.

## Three Interaction Layers

The ego-centric view worked by reloading the graph around a clicked node. The brain view can't do that — the spatial arrangement IS the information. Reloading around a single node would destroy it.

So I separated interaction into three composable layers. Pill filters in the sidebar toggle entire regions — brighten what you care about, darken the rest. Clicking a node highlights its ego-neighborhood in-place without reloading anything — the node and its connections brighten while everything else dims. Cross-region edges become the most visible elements, showing how a person connects to goals across the brain.

The third layer is the explicit exit. Double-click a node and the camera zooms into that region while the brain mesh fades out. The ego-centric BFS graph loads in that spatial area, centered on the focus node. "Back to brain" reverses the animation. The brain is home; ego-centric exploration is a temporary excursion into detail.

## The Wireframe Problem

The brain shape needs a visible outline — without it, nodes are just colored dots floating in space with no spatial context. My first assumption was to create a Three.js wireframe mesh from the brain's 80 vertices and 140 faces, add it to the ForceGraph3D scene, done.

The 3d-force-graph library bundles Three.js internally. Loading a separate Three.js before it causes version conflicts — `Timer is not a constructor`. Loading it after produces objects the renderer can't process — `intersectsFrustum is not a function`. Two versions of the same library creating structurally incompatible objects.

The fix was to stop fighting the rendering pipeline and work with it. Instead of a Three.js mesh, I placed invisible fixed-position nodes at every brain mesh vertex, connected by edges that trace the triangulated surface. The wireframe uses ForceGraph3D's own rendering — same pipeline as the data nodes, no foreign objects in the scene. The outline nodes are transparent (zero opacity, near-zero size), so only the connecting edges are visible, tracing the brain's silhouette as a faint indigo wireframe.

## Containment

Keeping 300 nodes inside an organic shape while force-directed layout does its thing turned out to be a three-layer problem.

Layer one: each region is defined by a set of bounding planes. Containment is one dot product per plane per node per tick — a velocity-based restoring force that nudges nodes toward the region center when they drift outside. Not a hard projection (that causes oscillation at boundaries) but a smooth push-back.

The problem is that bounding boxes are rectangular. Some region boxes extend past the brain surface. Nodes that are technically "inside their region" end up visually outside the brain.

Layer two: an ellipsoidal outer containment. The brain is roughly 130 units across, modeled as an ellipsoid with semi-axes derived from the mesh dimensions. Any node outside the ellipsoid gets a distance-proportional push toward center — gentle near the boundary, strong further out. This catches everything the rectangular regions miss.

Layer three: the charge repulsion and seeding. 86 goal nodes in the prefrontal region are far too dense when seeded at the center with small jitter. Seeding uniformly across each region's full bounding volume and cranking charge repulsion to -250 forces visible spacing between nodes. The containment layers keep them from escaping; the repulsion fills each region toward its edges rather than clumping at center.

The balance between these three forces — region containment, brain boundary, and charge repulsion — is still not where I want it. It looks like a brain, but not convincingly enough. The region separation is clear and the wireframe gives spatial context, but I think the whole layout approach needs rethinking. That's a separate issue.

## What the Brain Revealed About the Data

The most valuable thing the brain view did wasn't visual — it was diagnostic. Looking at the settled layout, a third of the nodes are floating disconnected. No edges at all.

The numbers: 185 entities, 172 edges, 60 orphans. 32% of the knowledge graph is unreachable by traversal.

These are entities the extraction pipeline pulled from Frida Kahlo's biography but never linked to other entities. "Frida and Diego's marriage" exists as a node but connects to nothing — no edge to Frida, no edge to Diego, no edge to any event. The information is there; the relationships aren't.

This matters beyond visualization. Consolidation phases that walk the graph (merge detection, community summary, curiosity signals) can't reach these nodes. Retrieval modulation that scores by graph proximity to active goals gives them a zero. They're knowledge the system has but can't use.

I filed #695 to fix the extraction pipeline's relationship linking. The target is under 10% orphan rate. The brain view made the gap immediately obvious — a list of entity names never would have.

## The Demo I Still Want

Watching a brain fire in real-time as the cognitive engine processes. The LLM receives input, appraisal lights up the limbic region, drives fire in the frontal lobes, CAPS settling pulses through the basal ganglia, memories form and fade into their regions. The WebSocket channel already exists for pushing events. The brain layout is the spatial foundation that makes that animation legible — each region lighting up in sequence tells the story of how the agent thinks.

The brain isn't good enough yet for that demo. But the diagnostic value — seeing the knowledge graph's shape, spotting the connectivity gaps, understanding which cognitive regions are dense and which are empty — that's already earning its keep.
