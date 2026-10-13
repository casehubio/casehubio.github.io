---
title: "Annotations That Belong Where They Run"
date: 2026-08-20
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [annotations, cdi, interceptors, quarkus, classloader]
issues: [196, 197, 198, 199]
---

The ledger's annotation module shipped last session — `@Audited`, `@ComplianceSupplement`, `@Attested`, the full interceptor pipeline. Deployment tests passed. The example compiled. And then a real `@QuarkusTest` boot threw `LinkageError`.

## The classloader wall

Quarkus loads each extension's runtime JAR in its own classloader. When the `AuditedInterceptor` in the annotations extension constructed an `AuditRecord` referencing `ActorType` from the platform API, the JVM saw two different `ActorType` classes — one loaded by the extension classloader, one by the application classloader — and rejected the linkage.

The insidious part: `QuarkusUnitTest` in the deployment module uses a different classloading model. Everything loads in one classloader. So the tests passed. Only a real application boot surfaced the problem.

The fix was architectural, not a workaround. I moved `AuditedInterceptor`, `ComplianceSupplementContext`, and `ComplianceSupplementEnricher` from `annotations/runtime` into `casehub-ledger/runtime/service/intercept/` — the same module as `ProvenanceCaptureInterceptor`. The annotations module keeps only the annotation definitions and the deployment-time Jandex validation. The interceptor lives alongside its dependencies. Same classloader, no boundary crossing.

The dependency chain inverted cleanly: `runtime → annotations → api`. The annotations module became lighter — no CDI beans, no runtime behaviour, just marker annotations and build-time checks.

## Making annotations carry domain context

With the classloader fix in place, the next gap was obvious: `@Audited` methods produced entries with null `domainData`. The return value — which often IS the domain context — was being ignored.

The implementation is minimal. After `ic.proceed()`, Jackson's `convertValue` serializes the return value to `Map<String, Object>`. Scalars and nulls produce null domainData. One static `ObjectMapper`, one try-catch, no configuration. A method returning an `Order` record now produces `domainData` containing `{"status":"PLACED","total":49.99,"customerId":"alice"}` — closing most of the gap between annotation-driven `PlainLedgerEntry` and typed domain subclasses.

## Compliance as its own interceptor

`@ComplianceSupplement` started life requiring `@Audited` — the build-time validator rejected it standalone. But domain subclass consumers (the ones constructing `OrderLedgerEntry` manually and calling `repo.save()`) couldn't use it at all. They had the write path but no way to push compliance context through annotations.

Making `@ComplianceSupplement` an `@InterceptorBinding` with its own interceptor at `Priority(APPLICATION)` solved this. The standalone interceptor pushes the `ComplianceSupplementContext` before the method runs. If `@Audited` is also present (at `APPLICATION + 1`), it checks `isActive()` and skips the redundant push. The enricher fires during the save pipeline regardless — it just reads from the ThreadLocal context.

The pattern mirrors `@ProvenanceCapture` exactly. Same priority tier, same push/pop lifecycle, same composition model. A method can carry `@ComplianceSupplement` alone, `@Audited` alone, or both — and the interceptors compose correctly.

## The example trail

Bringing every example up to the current API was the unglamorous part. Multi-tenancy had landed since these examples were written, so every `repo.save(entry)` needed a tenancyId parameter, every `findBySubjectId(id)` gained a second argument, and every example needed a `CurrentPrincipal` implementation and `domainContentBytes()` override. Six examples, 47 tests, all green.

The eigentrust-mesh example had a subtler problem: `saveAttestation` uses `@Transactional(REQUIRES_NEW)`, which starts a new database connection that can't see uncommitted entries from the outer transaction — even after `em.flush()`. Splitting entry creation and attestation creation into separate `@Transactional` methods fixed it, but only when called through the CDI proxy, not as `this.` calls.

## What this opens up

The annotation-driven model is now feature-complete for its first iteration. A consumer can annotate a CDI method and get an immutable audit entry with domain context, compliance metadata, and attestation — without touching the ledger API directly. The programmatic path stays for domain subclasses and batch operations, but annotations are the recommended default.

The next gap is on the `@Attested` path: `domainData` from return values only flows through `LedgerAppender`, not `OutcomeRecorder`. An attested method's return value is currently dropped. That's a natural follow-up — but it needs `OutcomeRecord` to grow a `domainData` field first.
