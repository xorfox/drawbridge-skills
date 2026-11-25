---
name: react-integration
description: Add Drawbridge login (OIDC) to a React single-page app built with Vite. Covers wiring react-oidc-context with a public PKCE client, env configuration, sign-in/sign-out, and verifying the flow end to end. Use when a React app needs to authenticate users against a Drawbridge realm. Requires an issuer URL and client ID from a registered OIDC app (see the drawbridge-register-client skill).
---

# Drawbridge authentication for a React SPA (Vite)

Wire a React app built with Vite to authenticate users against a Drawbridge realm using OpenID Connect. The app is a **public client**: there is no client secret, PKCE (S256, on by default with the code flow) replaces it — nothing sensitive ships in the bundle.

Tested stack: Vite + React 19 + `react-oidc-context` 3.x + `oidc-client-ts` 3.x. If the project uses Next.js or another SSR framework, stop and tell the user this skill covers SPAs only.

## Before you start

You need three strings from an already-registered Drawbridge OIDC app:

1. **Issuer URL** — looks like `https://sso.<host>/realms/<realm-name>`
2. **Client ID**
3. **The registered redirect URIs** for that client

If the realm has no registered app yet, use the **drawbridge-register-client** skill first (framework-agnostic), or the user can register one in the Drawbridge dashboard (realm → Registered apps). The redirect URIs are exact-match: `http://localhost:5173` works for local development (loopback http is allowed), everything else must be `https`.

## Collect these inputs

- Issuer URL and client ID (above)
- Dev origin — default `http://localhost:5173` (Vite's default port); adjust if the project runs elsewhere
- Whether a production callback is known yet (can be added later without changing the client ID)

## Step 1 — Install dependencies

```sh
npm install oidc-client-ts react-oidc-context
# or: yarn add oidc-client-ts react-oidc-context
```

## Step 2 — Environment variables

`.env.local` (git-ignored by Vite convention):

```
VITE_OIDC_ISSUER=https://sso.example.com/realms/my-realm
VITE_OIDC_CLIENT_ID=app-my-app
```

`src/vite-env.d.ts` (keeps TypeScript happy):

```ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
	readonly VITE_OIDC_ISSUER: string;
	readonly VITE_OIDC_CLIENT_ID: string;
}

interface ImportMeta {
	readonly env: ImportMetaEnv;
}
```

## Step 3 — Create the UserManager

`src/auth.ts`:

```ts
import { UserManager } from 'oidc-client-ts';

// The whole Drawbridge integration is three strings from your registration:
// issuer URL, client ID, redirect URI. There is no client secret: the app is
// a public client and PKCE (S256, the library default for the code flow)
// replaces it.
export const userManager = new UserManager({
	authority: import.meta.env.VITE_OIDC_ISSUER,
	client_id: import.meta.env.VITE_OIDC_CLIENT_ID,
	redirect_uri: 'http://localhost:5173',
	post_logout_redirect_uri: 'http://localhost:5173',
	scope: 'openid profile email'
});
```

The `redirect_uri` must match one of the client's registered redirect URIs **exactly** — scheme, host, port, and path. `http://localhost:5173` is Vite's default dev origin; if the project serves elsewhere (different port, or the CRA port 3000 case below), substitute the actual origin in both URI fields **and** make sure it is one of the registered redirect URIs.

If the app should keep users logged in across refreshes for the token lifetime, add `automaticSilentRenew: true`. Leave it off for a login-only integration: a refresh then means logging in again, with no iframes or refresh tokens to reason about.

## Step 4 — Wrap the app in the AuthProvider

`src/main.tsx`:

```tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { AuthProvider } from 'react-oidc-context';
import { userManager } from './auth';
import App from './App';

createRoot(document.getElementById('root')!).render(
	<StrictMode>
		<AuthProvider userManager={userManager}>
			<App />
		</AuthProvider>
	</StrictMode>
);
```

`react-oidc-context` handles the OIDC callback automatically: when the provider is redirected back with a code, it completes the token exchange before rendering.

## Step 5 — Use the auth state

`src/App.tsx` (minimal pattern — adapt the UI to the app):

```tsx
import { useAuth } from 'react-oidc-context';

function App() {
	const auth = useAuth();

	if (auth.isLoading) return <p>Checking your session…</p>;

	if (auth.error) {
		return (
			<div>
				<p>{String(auth.error)}</p>
				<button onClick={() => void auth.signinRedirect()}>Try again</button>
			</div>
		);
	}

	if (!auth.isAuthenticated) {
		return <button onClick={() => void auth.signinRedirect()}>Log in with Drawbridge</button>;
	}

	const profile = auth.user?.profile;
	return (
		<div>
			<p>Signed in as {profile?.email}</p>
			<button onClick={() => void auth.signoutRedirect()}>Log out</button>
		</div>
	);
}

export default App;
```

Useful state on `auth.user`: `profile` (ID-token claims — email, name, groups), `access_token` (for calling your backend APIs), `expires_at` (unix seconds).

## Step 6 — Verify

**Verify yourself before reporting done:**

1. Start the dev server and confirm it boots with no compile or type errors.
2. Fetch `<issuer URL>/.well-known/openid-configuration` and confirm it returns JSON — this proves the issuer resolves and is reachable from this machine. A failure here means a wrong or unreachable issuer; fix that before anything else.
3. Grep the running build's output (or the browser console on load) for the env vars actually being injected — the two most common silent failures are an `.env.local` that isn't loaded and a variable name typo.

**Then hand this checklist to the user** — the login round-trip needs their browser:

1. Open the app and click log in — the browser should land on the realm's Drawbridge login page (with a Register link if the realm owner has enabled self-registration).
2. Sign in (or register + verify email if that flow is on) — the browser returns to the app signed in, with email visible.
3. Log out — the browser returns to the app signed out.

If the user reports a failure at the login redirect, walk the troubleshooting table below before changing any code — the cause is almost always configuration, not the wiring.

## Going to production

Add the production callback (e.g. `https://app.example.com`) to the client's redirect URIs — via the **drawbridge-register-client** skill, the Drawbridge dashboard (realm → Registered apps), or `updateOidcClient` if you have the Drawbridge MCP server. The client ID does not change; make the URIs environment-driven in `auth.ts` (e.g. `${window.location.origin}`) so one build serves both environments.

## Adapting to another bundler (CRA, webpack, Parcel)

The OIDC code is identical — only environment-variable access and the dev origin change:

- **Env vars:** `import.meta.env.VITE_OIDC_ISSUER` becomes `process.env.REACT_APP_OIDC_ISSUER` (CRA) or whatever your bundler's define mechanism provides (webpack `DefinePlugin`, Parcel `--env` prefixes). Keep the same two variables, prefixed appropriately, and never commit `.env.local`.
- **Typing:** drop `src/vite-env.d.ts`; declare the two `process.env` fields yourself (or via `env.d.ts` with the CRA types).
- **Dev origin:** CRA serves on `http://localhost:3000` by default — use that as `redirect_uri` and register it as one of the client's redirect URIs.

React Native is out of scope: it has no browser redirect flow and uses OS-level auth tabs instead.

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| Redirected to login, then an "invalid redirect uri" error | `redirect_uri` in `auth.ts` does not exactly match a registered URI. Compare scheme/host/port/path character by character. |
| Loop after login — back at the app still signed out | The callback route isn't the same page that mounted the `AuthProvider` (custom routing setup). Ensure the app renders the provider at the redirect URI. |
| `authority` metadata fetch fails | Issuer URL wrong. It ends with the realm name and has no trailing slash: `https://sso.example.com/realms/my-realm`. Test it: append `/.well-known/openid-configuration` and expect JSON. |
| Claims like groups missing from the profile | Depends on the realm's client configuration/mappers. The `openid profile email` scope always yields sub, email, name basics. |
| Works in one browser, silent-renew fails in another | Third-party cookie blocking — inherent to iframe-based silent renewal; prefer full-page redirect login or refresh tokens if the realm allows them. |

## Out of scope

- Registering the OIDC client itself — see the **drawbridge-register-client** skill.
- Next.js / SSR frameworks, SAML.
- Backend token validation — the access token your app receives is a JWT your API can verify against the issuer's JWKS endpoint.
