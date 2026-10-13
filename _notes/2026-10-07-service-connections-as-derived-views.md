---
title: "Service connections as derived views"
date: 2026-10-07
entry_type: note
subtype: diary
series: issue-530-social-login-connection
projects: [casehubio/platform]
tags: [oauth, spi, authn, architecture]
author: mdp
---

OAuth tokens already live in `OAuthTokenStore`. Required scopes already live in `ScopeRegistry`. Whether a user is "connected to Google for Drive access" is just a question you can answer by combining those two — token exists AND scopes satisfied. So why add persistence for service connections?

We didn't. `ServiceConnectionProvider` is a facade — five methods, zero tables. `getConnection()` looks up the token, checks scope satisfaction, and returns CONNECTED, PARTIAL, or DISCONNECTED. `getAccessToken()` delegates to `OAuthTokenManagerCore.getValidToken()`, which already handles transparent refresh. `disconnect()` delegates to `revoke()`. The entire implementation is 86 lines of composition.

The interesting design question was where scope merging belongs. The issue requirement was "one consent screen" — when a user clicks Login with Google, the platform should request both identity scopes and service scopes (Drive, Calendar, whatever downstream modules registered). The infrastructure for this already existed: `AbstractOAuthAuthenticationProvider.initiate()` reads an `additionalScopes` hint from `AuthenticationContext` and merges it into the authorization URL. We needed something to populate that hint.

`ScopeMergingLoginCustomizer` does exactly that — queries `ScopeRegistry.requiredScopes(provider)` and injects the result as `additionalScopes`. It sits in `AuthenticationRouterCore.initiate()`, before the provider sees the context. One config property (`casehub.authn.merge-service-scopes`, default true) controls whether it fires.

A design review caught something we'd have missed in production: Google doesn't issue refresh tokens unless the authorization URL includes `access_type=offline`. Without it, the access token expires after an hour and `getValidToken()` returns empty — the service connection silently dies. That's filed as a separate issue (#550) since it touches `AbstractOAuthAuthenticationProvider.buildAuthorizationUrl()` directly.

The dependency direction question was the only real structural decision. `ScopeMergingLoginCustomizer` needed to be called from `AuthenticationRouterCore` (in `authn-core`), but the plan originally placed it in `authn-social-core`. That would have inverted the dependency — `authn-core` doesn't depend on `authn-social-core`. Since the customizer only depends on `ScopeRegistry` and `AuthenticationContext` (both in `platform-api`), it belongs in `authn-core`. Compile failure caught it immediately.

The `listConnections()` method is worth calling out. The obvious implementation — scan `OAuthTokenStore.findAllByActorId()` and map each token to a `ServiceConnection` — misses providers where scopes are registered but the user hasn't connected yet. We added `ScopeRegistry.registeredProviders()` (a default method returning the key set) so `listConnections()` can return DISCONNECTED entries for providers with registered scopes. A connection management UI needs to show "Connect Google" — not just "here are your existing connections."

Three follow-up issues filed in the connectors repo for the consumer side: the ConnectionPlatform SPI, Google scope registration, and the connection status UI. The platform provides the query surface; connectors provides the domain knowledge of which scopes each integration needs.
