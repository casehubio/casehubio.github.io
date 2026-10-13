---
layout: post
title: "Three platform bugs from one policy class"
date: 2026-08-10
entry_type: note
subtype: diary
projects: [casehub-soc]
tags: [casehub-work, sla-breach, breach-decision, platform-gaps]
---

I set out to implement `SocSlaBreachPolicy` — the SOC escalation chain that routes
unclaimed or overdue WorkItems from tier-1 analysts up through tier-2, tier-3, and
eventually to the SOC manager. The policy itself is straightforward: read
`candidateGroups` to know which tier the WorkItem is currently at, read the scope
path to determine priority, return a flat `BreachDecision`. No state machine, no
persistence. The WorkItem's own groups are the state.

What made this interesting was what broke along the way.

The `BreachDecision` sealed interface has five variants, and the one that looks most
useful — `Chained` — is the one that will hurt you. The fluent API reads beautifully:
`Extend(12hr).thenOnBreach(EscalateTo(SOC_MANAGER))`. Extend first, then escalate.
Except that's not what it does. `ExpiryLifecycleService.executeBreachDecision()`
processes `Chained` as an atomic try/catch: execute the primary, and only if it throws
`BreachExecutionFailed`, fall back to the secondary. `executeExtend()` never throws —
it just bumps `expiresAt` and returns. So the fallback is unreachable. P3 incidents
would infinite-loop on Extend, getting a fresh 12-hour window every breach cycle,
never escalating.

The same trap hits `EscalateTo.thenOnBreach(new Fail(...))`. EscalateTo with non-empty
groups always succeeds. The Fail never fires. The `thenOnBreach` name is doing real
harm here — it reads as "then, on the next breach" when the actual semantics are "catch
this specific exception type." We dropped `Chained` entirely and went with pure stateless
flat returns.

That discovery cascaded into two more. The `scopeExpression` field on YAML humanTask
bindings — the mechanism for encoding priority tier in the scope path — was evaluated
by the engine and passed through `HumanTaskScheduleEvent.resolvedScope()`, but the
handler ignored it. `createInline()` used `target.scope()` (the static field, null when
only an expression is set). The dynamic scope was computed and then discarded. Same for
`resolvedTitle`. The fix was two ternary operators.

And once the WorkItem could reach `ESCALATED` status via `BreachDecision.Exhausted`,
the engine's `PlanItemCompletionApplier` turned out to have no `case ESCALATED:` in
its status switch. The PlanItem stayed `DELEGATED` permanently. The output mapping
never ran. The case stalled. Another one-line fix — `case ESCALATED:
item.markFaulted(); break;` — but one that no downstream app could work around.

Three platform issues, three small fixes, each requiring a full dependency rebuild.
The policy class itself is clean — priority from the scope path, tier from the groups,
flat decision out. But the path from "design looks right" to "actually works in the
runtime" crossed three module boundaries where assumptions didn't hold.

The outputMapping carries one last subtlety: when a WorkItem reaches ESCALATED, no
analyst selected an outcome. The JQ expression sees `.outcome == null`. We added a
null-outcome fallback that maps to `"escalated"`, satisfying the case goal without
needing a separate CDI observer. The platform applies the mapping; the application
defines what null means.
