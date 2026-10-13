---
layout: post
title: "Closing the Loop on Authentication Events"
date: 2026-10-07
entry_type: note
subtype: diary
projects: [casehubio/platform]
tags: [authn, event-listener, spring, quarkus, spi]
series: epic-549-identity-auth-completion
---

# Closing the Loop on Authentication Events

The authn SPI had an `AuthenticationEventListener` interface sitting in platform-api with two methods — `onSocialLoginCompleted` and `onOAuthTokenRefreshFailed` — and a no-op `@DefaultBean` wired into nothing. The interface existed; the integration didn't.

Wiring it in meant touching three core classes. `AuthenticationRouterCore` now fires `onAuthenticationSuccess` after a successful `verify()` and `onAuthenticationFailure` when verify throws. `AbstractOAuthAuthenticationProvider` fires `onSocialLoginCompleted` after identity binding — the point where the actor, granted scopes, and first-login flag are all known. `OAuthTokenManagerCore` fires `onOAuthTokenRefreshFailed` in its catch block when token refresh fails silently.

The interesting design question was what happens when a listener itself throws. A buggy listener implementation in `AuthenticationRouterCore.verify()` would fire after the challenge is consumed — the auth succeeded, but the caller gets an exception, and the challenge can't be retried. In the failure path, a listener exception would suppress the original authentication error entirely. We wrapped every listener call in a `fireEvent()` guard that logs and swallows — the authentication flow is never disrupted by observer code.

The Spring side needed more than just mirroring the Quarkus producers. `AuthnSpringAutoConfiguration` gained `@ConditionalOnMissingBean` defaults for `JwtSigningKeyResolver`, `UserResolver`, and `ChallengeStore` — SPIs that have no JPA store and no Spring-side no-op. Without these, the auto-config couldn't compose in the integration test. The `ChallengeStore` default uses a `ConcurrentHashMap` rather than a true no-op, since an empty `store()`/`consume()` would break the challenge flow even in test scenarios.

With the authn starter on the integration test classpath, all eight composition assertions pass: `AuthenticationRouterCore`, `SessionManagerCore`, `JwtIssuerCore`, and the five JPA stores.

This closes the last four issues on the epic — the authn modules have full Spring parity, event wiring, and a composition gate proving they assemble correctly.
