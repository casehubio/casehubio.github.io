---
title: The login button is not a component
date: 2026-11-19
author: Mark Proctor
tags: [mosaic, web-components, oauth, webauthn, identity, architecture]
---

# The login button is not a component

Every web framework ships a "Sign in with Google" button example. It's the hello world of OAuth integration. And it's wrong — not because the code doesn't work, but because it treats authentication UI as a monolith. The button, the redirect flow, the consent screen, the credential list — they're all tangled into one component that only works in one context.

We hit this wall building CaseHub's connection management. Two distinct consumers needed the same authentication primitives: platform apps (settings pages with the full Lit/pages stack) and a standalone MCP server that opens `http://127.0.0.1:8080/connections` in the browser when a tool needs credentials. Same UX patterns, completely different hosting contexts. A "LoginComponent" that owns its own fetch lifecycle is useless to the second consumer.

## The decomposition

The answer was three layers, not one:

**Primitives** — the smallest visual pieces. A `provider-status-row` that shows Google's icon, "Connected", and a Disconnect button. A `scope-list` that expands to show granted and missing OAuth scopes. A `passkey-row` for a registered YubiKey. A `connect-button` that renders the provider's brand colours and icon.

**Controllers** — non-visual logic that any component can use. `OAuthFlowController` handles the popup window, falls back to full-page redirect if the popup is blocked, verifies the `postMessage` origin (the security bit everyone forgets), and matches the state parameter. `ProviderBrandRegistry` maps provider IDs to their SVG marks and brand colours — ships Google, GitHub, Slack, Discord out of the box.

**Pre-configured pages** — compositions of primitives for common contexts. `connection-management-page` renders the full settings view: service connections with status badges and scope details, plus a passkey section with device names and revoke buttons. `login-page` renders the social login buttons with an "or sign in with passkey" divider. Both are ready-made — drop a tag, pass data, done.

The primitives are pure views. Data in via properties, user actions out via `pages-event`. No fetch logic, no REST coupling, no assumptions about where they're hosted.

```html
<mosaic-connection-management
  .connections=${connectionData}
  .credentials=${passkeyData}
  .providers=${['google', 'github', 'slack']}>
</mosaic-connection-management>
```

The MCP server uses the same `connection-management-page` component. It just wires the events to its own local credential store instead of a REST API.

## The OAuth flow controller

The popup-vs-redirect decision is messier than it looks. Popup is better UX (the settings page stays open) but popup blockers are aggressive. The controller tries popup first, then falls back:

1. Open a popup with the authorization URL
2. If `window.open` returns null — popup blocked. Save state to sessionStorage, redirect the whole page
3. Listen for `postMessage` from the popup
4. **Verify `event.origin`** — this is the step most OAuth tutorials skip, and it's the one that matters. Without it, any page can send you a fake authorization code
5. Verify the state parameter matches
6. Resolve the promise

The controller is a plain class, not a Lit component. The `connect-button` uses it internally, but any component that needs an OAuth flow can instantiate it directly.

## Provider branding and trademark reality

Social login buttons have brand guidelines. Google specifies exact colours and the "G" mark. GitHub has the Invertocat. The question is whether an open-source library can ship these.

The answer is yes, with a trademark notice. This is nominative use — the logos identify the service, not claim endorsement. Auth.js (next-auth) ships provider SVGs directly. The `social-login-buttons` package excludes them from the MIT license with a note that logos are trademarks of their respective owners. Apple is the exception — their guidelines prohibit redistribution, so the registry ships a slot instead of a logo.

The `ProviderBrandRegistry` is instance-based. The default singleton has the four built-in providers. Consumers override entries or add new ones:

```typescript
import { defaultBrandRegistry } from '@casehubio/mosaic-connection-manager';

defaultBrandRegistry.register('apple', {
  providerId: 'apple',
  displayName: 'Apple',
  icon: appleSignInSvg,
  brandColor: '#000',
});
```

## WebAuthn as a first-class peer

Passkeys aren't an afterthought bolted onto the OAuth dashboard. They sit alongside service connections as a different kind of credential — same management UX, different interaction pattern. The connection management page has two sections: "Service Connections" (OAuth) and "Passkeys" (WebAuthn). The login page shows social buttons above an "or sign in with passkey" divider.

Three passkey primitives handle the two ceremonies and the credential list:

- `passkey-register-button` — calls `navigator.credentials.create()` for enrollment
- `passkey-authenticate-button` — calls `navigator.credentials.get()` for sign-in
- `passkey-row` — displays a registered credential with its device name, transports, and a revoke button

Both ceremony buttons follow the same two-step pattern as the OAuth connect button: click without options emits a request event (host fetches challenge from server), host sets the options property, button runs the browser ceremony on the next click.

## What this enables

The connection-manager package ships 6 primitives, 2 controllers, and 1 pre-configured page. The login-page package adds the authentication context. Together they cover 7 of the epic's child issues (#225–#230, #232) plus the MCP server auth page (#328).

Platform apps drop `<mosaic-connection-management>` into a settings panel. The MCP server drops it into its standalone page. A mobile-first app that only needs passkey sign-in imports `<mosaic-passkey-authenticate>` and nothing else.

The platform REST endpoints (casehubio/platform#898) are a separate concern — the components don't know or care about the fetch layer. That's the point. The login button is not a component. It's a composition.

---

*Source: [casehubio/casehub-mosaic#224](https://github.com/casehubio/casehub-mosaic/issues/224) — Epic: Connection & Identity UI components*
