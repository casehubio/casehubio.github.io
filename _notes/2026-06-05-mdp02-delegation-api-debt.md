---
layout: post
title: "Delegation API debt from #245"
date: 2026-06-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-work]
tags: [ci, delegation, lifecycle, ledger]
---

CI had been red on `casehubio/work` for a while. Not a flaky test — four separate failures, all from the same root cause: `WorkItemService.delegate()` grew a `DeclineTarget` parameter in issue #245, and four call sites outside the `runtime` module never got the memo.

The first one was easy enough: `LedgerIntegrationTest` tried to call `delegate(id, "alice", "bob")` and wouldn't compile. Fixed, pushed. CI failed again.

`CreditDecisionScenario` had the same missing argument. But it had a second problem — after delegating to bob, it called `claim()` to transfer control. That made sense before #245, when delegation returned the item to the pool with `PENDING` status. Now delegation sets `DELEGATED` and names a specific delegatee; the delegatee accepts by calling `acceptDelegation()`, not `claim()`. Wrong method, 409.

While fixing that, I noticed `LedgerEventCapture` was silently dropping both new event types. When `WorkItemService.acceptDelegation()` fires `WorkItemLifecycleEvent.of("DELEGATION_ACCEPTED", ...)`, the factory method lowercases the name and prepends the CloudEvents prefix: `io.casehub.work.workitem.delegation_accepted`. The observer's `EVENT_META` lookup table maps lowercase underscore suffixes — `delegated`, `assigned`, `started`, and so on. `delegation_accepted` wasn't there. Neither was `delegation_declined`. Both dropped silently. No ledger entry, no exception.

The factory method Javadoc does say "lowercased automatically" — the non-obvious part is that `LedgerEventCapture` has a static table that must be kept in sync with every new event name the service fires.

Fourth call site: `WorkItemNativeIT.delegatePath_claimThenDelegate` asserted `status == "PENDING"` after delegation. Same pre-#245 assumption. Updated to `"DELEGATED"`.

Four sites, four commits, all under #250. CI is green.

The build stops at the first failing module, so none of the later failures were visible until the `ledger` compilation error was fixed. The debt was worse than it looked from the first red.
