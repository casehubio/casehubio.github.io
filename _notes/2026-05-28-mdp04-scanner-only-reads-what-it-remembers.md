---
layout: post
title: "The Scanner Only Reads What It Remembers"
date: 2026-05-28
entry_type: note
subtype: diary
projects: [CaseHub Work]
tags: [flyway, quarkus, casehub-work, extension-authoring]
---

The task looked mechanical: rename `db/migration/` to `db/work/migration/` across
all casehub-work modules. Flyway path scoping, protocol PP-20260525-607b33 compliance,
close work#229. We'd done harder things in an afternoon.

It did not take an afternoon.

## The plan that looked obvious

I wanted the extension to self-register its migration path. When a consumer embeds
casehub-work, Flyway should just work — no `quarkus.flyway.locations` required, no
manual config, no reading the FLYWAY.md before your first deployment.

Quarkus provides `FlywayConfigurationCustomizer` exactly for this. An
`@ApplicationScoped` bean, a `customize(FluentConfiguration)` method, add the
location, done. I'd even confirmed from bytecode that unqualified beans apply to
the default datasource only — not to named ones like `qhorus`. The design held up
to scrutiny. The design review caught the wrong interface name
(`FlywayContainerCustomizer` doesn't exist; it's `FlywayConfigurationCustomizer`)
and the native image limitation, but the JVM story looked solid.

Then we moved the files and ran the tests. 747 passing. The extension worked.

## The thing that wasn't working

Twenty minutes later we installed the runtime JAR and ran the ledger module's
`@QuarkusTest` suite with `quarkus.flyway.locations=db/ledger/migration` only —
relying on the customizer to add the work path. Every test failed with
`Table "WORK_ITEM" not found`.

I added the customizer as an unremovable bean via `AdditionalBeanBuildItem`. Same
failure. The bean was present, the `customize()` method ran, the location string
was added to `FluentConfiguration`. And still: no tables.

The fix that worked was explicit consumer config:

```properties
quarkus.flyway.locations=classpath:db/work/migration
```

Which meant the customizer was not providing what I thought it was providing.

## What Quarkus actually does

`FlywayProcessor.build()` runs at augmentation time. It reads `quarkus.flyway.locations`
from build config, scans the classpath for SQL files at those locations, and records
the file list via `FlywayRecorder` into the bytecode output. That list is frozen at
build time.

At JVM runtime, `QuarkusPathLocationScanner.getResources(Location)` serves
migrations from that pre-registered list — it does not re-scan the classpath.
`FlywayConfigurationCustomizer.customize()` runs at startup and successfully adds
`classpath:db/work/migration` to `FluentConfiguration`'s location list. But when
Flyway asks the scanner for files at that location, the scanner has nothing
pre-registered there and returns empty.

The customizer works fine for standard Flyway. In Quarkus, the scanner is the wall.

## The 747 tests that lied

Then I realized: the 747 runtime tests had passed because of stale `target/classes/`
artifacts. We'd done `git mv src/main/resources/db/migration
src/main/resources/db/work/migration`, then run `mvn test`. Maven's
`process-resources` phase copies from `src/main/resources/` to `target/classes/`
but doesn't delete the old directory. So `target/classes/db/migration/` still had
all 30 SQL files. Quarkus's default `classpath:db/migration` scan found them there.
The tests passed against the wrong path.

`mvn clean install -DskipTests` cleared the stale directory. The runtime tests then
failed the same way. The move had never actually been verified.

## What we actually shipped

The extension self-registers via `FlywayConfigurationCustomizer` for non-Quarkus
Flyway environments, and we documented that explicitly. For Quarkus, the answer is:
every module that uses casehub-work schema must declare the path explicitly. We added
`quarkus.flyway.locations=classpath:db/work/migration` to thirteen test properties
files, updated FLYWAY.md and GOTCHAS.md, and wrote protocol PP-20260528 to
casehub-parent covering this as a platform-wide rule.

We also filed casehub-ledger#99 — the same pattern applies there. Once ledger
implements `NativeImageResourcePatternsBuildItem` for `db/ledger/migration`, two
more lines disappear from everyone's test properties.

The migrations are in the right place. The path collision problem is solved. And
I have a much cleaner mental model of what Quarkus Flyway does at build time versus
what it does at runtime — which it turns out are quite different things.
