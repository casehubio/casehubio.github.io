---
layout: note
title: "Per-Tenant SLA Preferences — the Infrastructure Was Already There"
date: 2026-09-01
entry_type: note
subtype: diary
projects: [casehub-work]
tags: [sla, multi-tenancy, preferences, cdi]
issue: 375
---

The issue said "per-tenant SLA defaults YAML." We built something different — and it took about 130 lines of production code, because the hard work was already done.

When #372 landed deployment-wide SLA defaults via `META-INF/work-sla-defaults.yaml`, the spec explicitly deferred per-tenant config to #375. The assumption was a tenant-aware YAML discovery mechanism — multiple YAML files, classpath scanning keyed by tenant ID. I started brainstorming there but stopped when I traced the existing code.

`ExpiryLifecycleService.buildBreachContext()` was already calling `preferenceProvider.resolve(new SettingsScope(tenancyId, scope, now))` and passing the result as `SlaBreachContext.preferences()`. The tenant+scope-aware preference infrastructure was wired in. `DeclarativeSlaBreachPolicy` was ignoring it. The platform had the answer; we just weren't using it.

The design review caught this and pushed it further — don't embed preference checks inside `DeclarativeSlaBreachPolicy`. Make it a CDI decorator. The reasoning: a decorator works with *any* `SlaBreachPolicy` implementation, not just the declarative one. A deployer running a custom policy still gets tenant preference overrides. The `CallbackSlaBreachPolicyDecorator` already demonstrates the pattern — same structure, different input source.

So the implementation is a `@Decorator` at `APPLICATION + 200` (inner to the callback decorator at `APPLICATION + 100`). It checks four preference keys — two breach actions, two extension hours. If the tenant has a preference set for the item's scope, the decorator returns immediately. Otherwise, pure passthrough. The underlying policy never knows it's being decorated.

Three guards that the spec review caught before they became bugs:

1. **Self-escalation.** If a tenant sets `escalateTo:team-leads` and the item is already assigned to `team-leads`, the decorator delegates instead of creating an infinite re-escalation loop. `DeclarativeSlaBreachPolicy` has the same guard via `unwrapSelfEscalation()`.

2. **Config fallback.** The extension hours fallback was hardcoded to 24. It should read `config.defaultExpiryHours()` — same source as the declarative policy uses. Otherwise a deployer who sets `casehub.work.default-expiry-hours=8` gets inconsistent behaviour depending on whether preferences or YAML fires.

3. **Parse resilience.** `MapPreferences.get()` calls the key's parser on stored String values. A malformed preference (`"invalid-action"`) would throw `IllegalArgumentException` and abort batch expiry processing for all items in the transaction. The decorator catches and delegates — same pattern as the callback decorator's remote-failure handling.

The broader observation: the platform's preference system already solves per-tenant, per-scope configuration. Building a parallel mechanism (tenant-keyed YAML, a new SLA config store) would have been architecturally wrong — it would duplicate what `PreferenceProvider` already provides. The issue's title pointed toward YAML, but the intent was tenant-level overrides. Preferences are how the platform does that.
