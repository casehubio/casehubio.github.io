---
layout: post
title: "Killing the god object — migrating 96 test files from QhorusMcpTools"
date: 2026-09-21
entry_type: note
subtype: diary
projects: [casehubio/qhorus]
tags: [testing, refactoring, migration]
series: issue-452-delete-qhorus-mcp-tools
---

`QhorusMcpTools` was 2092 lines of accumulated test infrastructure. Every test in the project injected it. Every channel creation was a 19-argument method call where 17 of them were null. After #451 stripped the MCP annotations, the class was inert — a CDI bean with no MCP footprint, used only by tests. Time to kill it.

The approach was pragmatic, not pure. I'd planned to replace the 19-arg calls with clean builder overloads — `helper.createChannel("name")` instead of `tools.createChannel("name", "desc", null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null)`. But 488 createChannel call sites across 96 files made a clean break impractical in one pass. We added a compat 19-arg overload to `QhorusTestHelper` that matches the old signature exactly — parse the CSV strings, construct the builder internally, return the domain type instead of the MCP DTO. The bulk replacement became mechanical: swap the import, rename the field, move on.

The module cycle was the first real constraint. `testing/` depends on `runtime`. `runtime` can't depend back on `testing`. The project already knew this — CLAUDE.md documents it for `RecordingChannelBackend`. The helper lives in both places: `testing/` for other modules, `runtime/src/test/` for runtime's own tests. Code duplication, but the alternative — a new intermediate module — adds more complexity than it removes.

The bulk migration covered imports, field injection (`tools` → `helper`), method calls, and inner type references. The regex caught most patterns but missed one: imports of inner types via the outer class's package path — `import io.casehub.qhorus.runtime.mcp.QhorusMcpToolsBase.CheckResult` survived the class-name replacement and needed a second cleanup pass.

The interesting failures were the behavioural gaps, not the compile errors. `getReplies` and `searchMessages` accepted a `readerInstanceId` parameter but silently ignored it — the visibility filtering that checked whether a targeted message (`capability:code-review`, `instance:alice`) should be visible to a given reader was in the old MCP tools but never migrated to the helper. Five tests relied on this filtering. `channelDigest` returned an empty topic breakdown because nobody added the `TopicStore` query. `listLedgerEntries` didn't validate the `sort` parameter or wrap `DateTimeParseException`. Each gap was a method that compiled and ran but didn't do what the tests expected.

The dual-copy problem caught us once. IntelliJ's MCP edited `testing/src/main/java/.../QhorusTestHelper.java` — the first match in its index. The runtime copy at `runtime/src/test/java/.../QhorusTestHelper.java` stayed untouched. We ran the tests three times before noticing the runtime tests were still using the old code. The fix: sync the copies with a Python `shutil.copy2` call that bypasses the IDE's file routing entirely.

After the last test turned green — 2038 passing, zero failures — we deleted `QhorusMcpTools.java` and `QhorusMcpToolsBase.java`. 2692 lines removed. What was a monolithic MCP-entangled test facade became a 1530-line helper with fifty methods and twenty inner record types. It delegates to the real services and stores. The 19-arg compat overloads still exist — they'll get cleaned up when individual test files are touched for other reasons. No rush.
