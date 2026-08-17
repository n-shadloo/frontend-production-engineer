# Session and token lifecycle

Next.js 16.3, React 19.2.6 or later, `djangorestframework-simplejwt` 5.5.1,
against a Django and DRF backend. This file owns the life of a credential in
the browser. The subjects are the auth model, the place that holds each value,
and the cookie that carries the session. They also include the refresh, the
logout, and the failure that must never end a session.

Which code decides that a request may proceed is
`references/route-protection-and-permissions.md`. The CSRF header exchange and
the CORS preflight are `references/cross-origin-and-bff-proxy.md`. The one
typed client and the `ApiError` that every failure becomes are
`references/api-client-and-request-safety.md`.

## Principle

A credential that a script can read is a credential that an injection can
send elsewhere. The browser holds one store that no script reads, and that
store is the `httpOnly` cookie.

An identity has a lifetime. Decide the lifetime, the renewal, and the
revocation together, because a gap between the three is a session that nobody
can end.

Concurrent failures describe one condition. Answer them once, and queue the
rest behind that one answer.

A failure of the backend is not a statement about the identity of the user.
Read the status before you discard the session.

The value that the server sets is the value that the browser keeps. A layer
that rewrites it in transit removes the session and reports nothing.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### Choose the auth model first

Fill this table before you write any auth code. The first row that matches
decides the model.

| Condition | Model | It reverses when | The cost |
| --- | --- | --- | --- |
| The frontend and Django answer on one origin, through a shared domain or a reverse proxy | Django `SessionAuthentication`, an `httpOnly` session cookie, and a CSRF token | A native mobile client starts to consume the same API, which the third row covers. | Every unsafe request carries `X-CSRFToken`, and the CSRF cookie must arrive before the first write. |
| The two origins differ, and you control both | SimpleJWT, with the access token in memory and the refresh token in an `httpOnly; Secure; SameSite=None` cookie. A session cookie with the same attributes also serves. | The two sides can share one origin, so the first row applies and four classes of failure disappear. | A cross-site cookie is fragile, and the browser policy on it continues to tighten. |
| A native mobile client consumes the same API | A token model. SimpleJWT with a bearer header, or the `django-allauth` headless `X-Session-Token` | The mobile client is withdrawn, so the browser is the only consumer. | The web client must still guard the token, so it takes an in-memory access token and an `httpOnly` refresh cookie. |
| A Server Component must render the identity | A session or a token that `cookies()` reads on the server | The identity appears after the first paint only, so the client can hold it. | An access token in memory alone is invisible to the server, so the value must sit in a cookie. |
| The product needs social login | `django-allauth` in headless mode, or Auth.js in front of Django for the provider protocol only | Auth.js starts to hold a second copy of the session that Django already owns. | A second system holds identity state, and the two must agree on every change. |
| The product is multi-tenant | The active organization in a URL segment, or in its own cookie. Every cache key carries it. | The product serves one tenant for each deployment. | Every query key, every prefetch, and every store must carry the tenant. |
| Someone proposes DRF `TokenAuthentication` or `knox` | Reject it for a browser application, unless a written exception exists | Never, while the client is a browser. | A static token never expires and never rotates, so one leak is permanent. |

The static token row is the one rejection in this table. NEVER take a static
DRF token for a browser application on the strength of its simplicity.

### Where each value lives

| The value | It lives in | NEVER in |
| --- | --- | --- |
| The refresh token | An `httpOnly; Secure; SameSite` cookie, and nothing else | `localStorage`, `sessionStorage`, a store, the query cache, a `NEXT_PUBLIC_` variable |
| The access token | A variable in memory, or an `httpOnly` cookie | `localStorage`, `sessionStorage` |
| The permission list and the roles | The user payload from the server, held in memory for the render | `localStorage` |
| The CSRF token | The readable `csrftoken` cookie, which is its design | — |

```ts
// Wrong: the token goes into web storage.
// Failure: every script on the page reads it, so one injection sink sends
// the refresh token to another host. The token stays valid after the tab
// closes, and no server-side revocation is possible until it expires.
localStorage.setItem("access", data.access);
localStorage.setItem("refresh", data.refresh);
```

```ts
// Correct: the server sets the refresh cookie, and the access token stays in
// one module variable that no other module writes.
// src/lib/auth/access-token.ts
let accessToken: string | null = null;

export function setAccessToken(next: string | null): void {
  accessToken = next;
}

export function getAccessToken(): string | null {
  return accessToken;
}
```

A page reload empties the memory variable. The first request then answers 401,
the single-flight refresh below runs once, and the refresh cookie restores the
session. Design for that first 401 rather than for a token that survives the
reload.

`references/client-and-url-state.md` states that a preference which the server
renders belongs in a cookie rather than in `localStorage`. This file adds the
harder rule for a credential and for a permission list.

### The cookie that carries the session

```
Set-Cookie: __Host-refresh=<jwt>; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=604800
```

| Attribute | What the frontend depends on |
| --- | --- |
| `HttpOnly` | No script reads the value, so an injection cannot send it elsewhere |
| `Secure` | The cookie travels on HTTPS only |
| `SameSite` | It decides whether a cross-site request carries the cookie at all |
| `Path` and `Max-Age` | They decide the reach of the cookie and how long the session lives with no renewal |
| The `__Host-` prefix | The browser rejects the cookie unless it is `Secure`, carries `Path=/`, and carries no `Domain` |
| The `__Secure-` prefix | The browser rejects the cookie unless it is `Secure`. Take this prefix with an explicit `Domain` where one session spans two subdomains |

Four rules bind the attributes. MDN states that a `__Host-` cookie must come
from an HTTPS page, must carry no `Domain`, and must carry `Path=/`. A
`SameSite=None` cookie must also carry `Secure`. One cookie holds 4 KB, so a
JWT carries `sub`, `role`, `exp`, `iat`, and `jti`, and nothing more. A
`Set-Cookie` header from Django must pass through every Route Handler and
every reverse proxy unmodified, or the browser stores nothing.

A `SameSite=None` cookie with no `Secure` produces this console message, and
the browser stores nothing:

```
Cookie "myCookie" rejected because it has the "SameSite=None" attribute but is missing the "secure" attribute
```

CAUTION: confirm that the `__Host-` prefix survives your own reverse proxy
before you depend on it. A proxy that rewrites the path can invalidate the
prefix rules. Read the header with `curl -I` through the real proxy, and read
`Path=/` and the prefix in the output.

The frontend does not set these attributes. It states which ones the
application depends on, and the backend team sets them. The sibling skill
`secure-code-auditor` owns the server-side choice of each attribute.
`references/cross-origin-and-bff-proxy.md` owns what a cross-site request does
with the cookie once it exists.

### The refresh is single-flight

```ts
// Wrong: each 401 starts its own refresh.
// Failure: five concurrent requests answer 401 together and start five
// refresh calls. With ROTATE_REFRESH_TOKENS and BLACKLIST_AFTER_ROTATION the
// first call blacklists the token, the other four present a blacklisted
// token, the backend reports reuse, and the session ends mid-visit.
async function request(input: RequestInfo, init?: RequestInit) {
  let response = await fetch(input, { ...init, credentials: "include" });
  if (response.status === 401) {
    await fetch("/api/auth/refresh/", { method: "POST", credentials: "include" });
    response = await fetch(input, { ...init, credentials: "include" });
  }
  return response;
}
```

```ts
// Correct: src/lib/auth/refresh.ts holds one in-flight promise, and every
// caller waits on it.
let inFlight: Promise<boolean> | null = null;

function refreshOnce(): Promise<boolean> {
  inFlight ??= fetch("/api/auth/refresh/", {
    method: "POST",
    credentials: "include",
    signal: AbortSignal.timeout(10_000),
  })
    .then((response) => response.ok)
    .finally(() => {
      inFlight = null;
    });
  return inFlight;
}

export async function requestWithAuth(
  input: RequestInfo,
  init?: RequestInit,
): Promise<Response> {
  const response = await fetch(input, { ...init, credentials: "include" });
  if (response.status !== 401) return response;

  const refreshed = await refreshOnce();
  if (!refreshed) {
    await endSession();
    throw new Error("session expired");
  }
  return fetch(input, { ...init, credentials: "include" }); // retry once only
}
```

Three rules hold. The promise lives in a module, never in a component, so
every caller in the tab shares it. The original request repeats once, and a
second 401 after the refresh ends the session. A 401 from the refresh endpoint
itself is terminal. It reports an expired token or a reused token, and neither
improves on a second attempt.

NEVER decode `exp` on the client to decide when to refresh. The clock of the
device drifts against the clock of the server, and a drift produces an
intermittent 401 straight after a successful refresh. Let the 401 from the
server start the refresh. The SimpleJWT `LEEWAY` setting absorbs the drift on
the server side, and the sibling skill `secure-code-auditor` owns its value.

### What ends a session, and what does not

| The response | What the application does | It reverses when | The cost |
| --- | --- | --- | --- |
| 401 on an ordinary request | Refresh once, then repeat the request | The refresh endpoint itself answered, which the next row covers. | One extra round trip on the first request after the access token expires. |
| 401 on the refresh endpoint | End the session. This is terminal. | Never. The token is expired or reused, and a second attempt returns the same answer. | The user signs in again, and the unsaved work of the view is lost. |
| 403 with a CSRF detail | Obtain the `csrftoken` cookie again, and repeat the write once | The session itself is gone, so the 401 path above serves. | One extra request to the endpoint that sets the CSRF cookie. |
| 403 with any other detail | Render the state for a permission that the user does not hold. Keep the session. | Never. A 403 states that this identity may not do this. | Each route states its own answer, so two routes can answer one status in two ways. |
| 500, 502, 503, or 504 | Render a retry state. NEVER end the session. | Never. A fault of the backend says nothing about the identity. | The user waits, and the view holds a state that a retry must clear. |
| A `TypeError` or a `DOMException` | Let it reach the error boundary. Neither carries a status. | The abort was deliberate, so the view discards it. | The boundary renders its fallback rather than one part of the view. |

`references/api-client-and-request-safety.md` owns `normalizeApiError` and the
`ApiError` shape. This file owns which of those statuses touches the session.
`references/server-state-and-query-cache.md` owns the retry rule that reads
`ApiError.retryable`, and no retry rule may repeat a refresh.

### The logout clears every copy

```ts
// Wrong: the client discards its own state and stops there.
// Failure: the refresh cookie is still valid, so a new tab restores the
// session. The query cache still holds the rows of the user, and a back
// navigation renders them. The other open tabs stay in the application.
export function endSession() {
  setAccessToken(null);
  router.push("/login");
}
```

```ts
// Correct: src/lib/auth/end-session.ts revokes on the server first, then
// clears every client copy, then tells the other tabs.
import { queryClient } from "@/lib/query/client";

const authChannel = new BroadcastChannel("auth");

export async function endSession(): Promise<void> {
  await fetch("/api/auth/logout/", {
    method: "POST",
    credentials: "include", // the server blacklists the token and clears the cookie
  });
  setAccessToken(null);
  queryClient.clear();
  authChannel.postMessage({ type: "SESSION_ENDED" });
  window.location.assign("/login"); // a full load, so no cached RSC payload survives
}

authChannel.onmessage = (event: MessageEvent<{ type: string }>) => {
  if (event.data.type === "SESSION_ENDED") window.location.assign("/login");
};
```

Five steps run in this order. The server revokes the token and clears the
cookie, because only the server can. The access token in memory becomes
`null`. `queryClient.clear()` empties the cache, and
`references/server-state-and-query-cache.md` owns that cache. The
`BroadcastChannel` message reaches the other tabs of the same origin.
`window.location.assign` performs a full load, so the Router Cache of the
browser holds no rendered payload of the previous identity.

`router.push` is not enough here. It leaves the client cache and the Router
Cache in place, and `references/caching-and-revalidation.md` states that a
server invalidation does not reach the Router Cache.

### The Django seam

| The endpoint | The request | The response |
| --- | --- | --- |
| `POST /api/token/`, from `TokenObtainPairView` | The credentials | `{access, refresh}` |
| `POST /api/token/refresh/`, from `TokenRefreshView` | The refresh token | `{access}`, and a new `refresh` where `ROTATE_REFRESH_TOKENS` is on |
| `/_allauth/{client}/v1/...`, from `django-allauth` headless | The flow of that endpoint | The session state, with `X-Session-Token` for an application client |
| The `dj-rest-auth` routes | Login, logout, password, and registration | The wrapper over `django-allauth` |

Take the request and the response types from the generated client. A change to
a serializer then becomes a compile error.
`references/openapi-schema-and-codegen.md` owns the generator, and
`references/boundary-validation-and-api-types.md` owns the parse over the
result.

Six SimpleJWT settings change what the frontend must do.
`ACCESS_TOKEN_LIFETIME` and `REFRESH_TOKEN_LIFETIME` decide how often the
refresh runs. `ROTATE_REFRESH_TOKENS` and `BLACKLIST_AFTER_ROTATION` together
make a reused refresh token terminal, and they make the single-flight rule
above mandatory. `AUTH_HEADER_TYPES` decides the prefix of the header, and the
default is `Bearer`. `LEEWAY` absorbs the clock drift. Read the installed
values from the backend team, and record them beside the client.

| The change on the server | What the frontend sees |
| --- | --- |
| A cookie is renamed, such as `access`, `refresh`, or `csrftoken` | Every `cookies()` read and every presence test in `proxy.ts` returns nothing, and the session ends with no error |
| The error envelope changes shape | The application can no longer separate a 401, a 403, and a CSRF failure, so it answers all three the same way |
| A token lifetime changes | The refresh timing no longer matches the expiry, and the user meets an unexpected 401 |
| `ROTATE_REFRESH_TOKENS` is turned on | Concurrent refresh calls now blacklist each other, and the session ends mid-visit |
| A `Set-Cookie` header is rewritten by a proxy | The browser stores nothing, and the login appears to succeed and then fail |

The sibling skill `secure-code-auditor` owns the server-side settings and the
migration that the blacklist application needs. The sibling skill
`django-api-contract` owns the shape of each auth endpoint. This file owns
what the browser sends, what it stores, and what the UI does with the answer.

### The libraries

The table gives each library its latest version, its release date, and its
maintenance status. The package registries supplied those facts on
16 August 2026. This table carries no advisory column. Read the advisory
database yourself before you install any row of it.

| Tier | Library | The rule | Latest version | Maintenance |
| --- | --- | --- | --- | --- |
| Recommend | The Next.js built-ins — `cookies()`, `redirect()`, and a data access layer with React `cache()` | Take these before any package. They cover the whole mechanism of a session. | Next.js 16.3, released 3 Aug 2026 | Active |
| Recommend | `djangorestframework-simplejwt` | The default for a token model. It issues, refreshes, rotates, and blacklists. | 5.5.1, released 21 Jul 2025 | Active, and marked Production/Stable |
| Recommend | `django-allauth`, headless and MFA | Registration, email verification, password reset, social providers, TOTP, WebAuthn, and passkeys. | The current development documentation | Active |
| Recommend | `openapi-typescript` with `openapi-fetch` | The typed auth contract, generated from drf-spectacular. | 7.13 with 0.17, which are the pins of this stack | Active |
| Conditional | `iron-session` 8.x | An encrypted session cookie that Next writes, where Django stays the credential authority and Next holds a seal. | 8.x | Maintained. Some registry entries for it are archived |
| Conditional | Auth.js, also called NextAuth, version 5 | The provider protocol for social login, in front of Django. Reject it where Django already owns the session, because two systems then hold one identity. | Version 5, on a long beta line | Active |
| Conditional | `dj-rest-auth` | A wrapper over `django-allauth` for the DRF login, logout, and password routes. | — | Maintained |
| Audit-only | DRF static `TokenAuthentication`, and `knox` | An existing install only. Plan the move to SimpleJWT rotation, or to sessions. | — | In decline for new work |
| Reject | A token in `localStorage`, a hand-written OAuth flow, and the OAuth implicit flow | Each one is discredited. RFC 6749 and RFC 7636 describe the Authorization Code flow with PKCE, which replaces the implicit flow. | — | — |

`references/dependencies-and-git-workflow.md` owns the rule that a new
dependency states its replacement, its size, and its maintenance status.

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

Next.js 16.2.11 and 15.5.21 are the patched releases of 21 July 2026. Two of
the defects in that release bear on this domain. CVE-2026-64642 is a bypass of
`proxy.ts` under Turbopack with a single entry in `i18n.locales`.
CVE-2026-64643 discloses the endpoint of a Server Function, so an attacker
finds the endpoint without a session.

CVE-2025-29927 of March 2025 is the earlier bypass, with a CVSS score of 9.1,
and it affects every release from 11.1.4 to 15.2.2. All three prove one rule.
`references/route-protection-and-permissions.md` holds it, and it states that
the gate never lives in `proxy.ts` alone.

SimpleJWT 5.5.1 of 21 July 2025 is the pin. It adds the `0013_blacklist`
migration that earlier releases omitted. Ask the backend team to confirm that
the migration ran, because rotation revokes nothing without it.

## Verification

```bash
# 1. Read the installed versions before you write code.
node -p "require('next/package.json').version"
node -p "require('react/package.json').version"

# 2. Find a token, a refresh token, or a permission list in web storage. This
#    must print nothing.
rg -niE 'localStorage|sessionStorage' src/ | rg -iE 'token|access|refresh|permission|role'

# 3. Find a refresh call outside the one module. Read every hit.
rg -n 'auth/refresh' src/ -g '!src/lib/auth/refresh.ts'

# 4. Find refresh logic inside a component. This must print nothing.
rg -n 'refresh' -g '*.tsx' src/ | rg -i 'fetch\(|mutate'

# 5. Confirm that the logout clears the query cache and tells the other tabs.
rg -n 'queryClient.clear\(|BroadcastChannel' src/lib/auth

# 6. Find a 5xx that reaches the logout path. Read every hit.
rg -n -B4 'endSession|logout' src/ | rg '5[0-9][0-9]|!response.ok'

# 7. Read the cookie attributes that the backend sets, through the real proxy.
curl -sI -X POST "$NEXT_PUBLIC_API_BASE_URL/api/auth/login/" | rg -i 'set-cookie'

# 8. Confirm one refresh call. Expire the access token, start several requests
#    together, and count the calls to the refresh endpoint in the network tab.
#    There is one.

# 9. Confirm the cross-tab logout. Sign out in one tab, and read the other.

# 10. Confirm that a 5xx keeps the session. Stop the backend, load a protected
#     route, and confirm that the application shows a retry state.
```

## Review checklist

- [ ] Is the auth model chosen from the decision table, and is the choice
      recorded?
- [ ] Is every token and every permission list absent from `localStorage` and
      `sessionStorage`?
- [ ] Does the refresh token live in an `httpOnly; Secure; SameSite` cookie?
- [ ] Does the access token live in memory, or in an `httpOnly` cookie?
- [ ] Does the session cookie carry a `__Host-` or a `__Secure-` prefix?
- [ ] Does every `SameSite=None` cookie also carry `Secure`?
- [ ] Is the session cookie under 4 KB, with no claim that the client does not
      read?
- [ ] Does the `Set-Cookie` header pass through every proxy unmodified?
- [ ] Is the refresh single-flight, in one module rather than in a component?
- [ ] Does the original request repeat once only after a refresh?
- [ ] Is a 401 from the refresh endpoint treated as terminal?
- [ ] Is the refresh started by a server 401, rather than by a client read of
      `exp`?
- [ ] Does a 5xx leave the session in place?
- [ ] Does the logout revoke on the server, clear the access token, clear the
      query cache, tell the other tabs, and perform a full load?
- [ ] Are the SimpleJWT lifetimes and the rotation settings of the backend
      recorded beside the client?
- [ ] Do the auth request and response types come from the generated client?

## Handoffs

- The enforcement layers, the data access layer, the Server Action gate, and
  the permission model in the UI →
  `references/route-protection-and-permissions.md`.
- The CSRF token exchange, the CORS preflight, the proxy Route Handler, and
  the topology that removes a cross-site cookie →
  `references/cross-origin-and-bff-proxy.md`.
- The one typed client, the timeout, the retry rule, and `normalizeApiError` →
  `references/api-client-and-request-safety.md`.
- The query cache that the logout clears, and the retry rule over an
  `ApiError` → `references/server-state-and-query-cache.md`.
- The store, and the preference that a cookie carries rather than
  `localStorage` → `references/client-and-url-state.md`.
- The generated auth types and the schema behind them →
  `references/openapi-schema-and-codegen.md`. The parse over the response is
  `references/boundary-validation-and-api-types.md`.
- `proxy.ts`, the `NEXT_PUBLIC_` prefix, and the route files →
  `references/app-router-structure.md`.
- The Router Cache that a full load clears →
  `references/caching-and-revalidation.md`.
- The carrier that a handshake takes for this same credential, and the close
  code that ends a connection → `references/push-transport-and-connection.md`.
  This file owns where the credential lives, and that file owns how it travels
  on a handshake.
- The login form and its fields →
  `references/form-schema-and-field-binding.md`. The field errors of a
  credential failure are `references/form-submission-and-server-errors.md`, and
  the multi-step flow of an MFA challenge is
  `references/multi-step-forms-and-unsaved-work.md`. This file owns the error
  taxonomy that the form maps.
- The CSP that makes an injection unable to read a token, and the response
  headers → `references/security-headers-and-csp.md`. The injection sink itself
  is `references/untrusted-markup-and-injection.md`. That domain holds a veto.
- The accessible name of a sign-in control, and the accessibility cost of a
  CAPTCHA → `references/semantics-and-accessible-names.md`. The focus order
  over those controls → `references/keyboard-focus-and-live-regions.md`. That
  domain holds a veto.
- The words in a session-expiry message →
  `references/error-and-empty-state-copy.md`.
- The MSW handler for each auth state, the Playwright storage state, and the
  expired-token fixture → domain 20 `testing-and-quality`. Not integrated yet.
- The reverse proxy in front of Node that must forward `Set-Cookie` → domain 22
  `build-deploy-and-runtime-ops`. Not integrated yet.
- The SimpleJWT settings, the blacklist migration, the password hash, the
  rate limit on the login endpoint, and the server-side CSRF enforcement → the
  sibling skill `secure-code-auditor`. This file owns what the browser sends
  and stores, and the UI behavior for a 401, a 403, and a CSRF failure.
- The shape of each auth endpoint as a published contract → the sibling skill
  `django-api-contract`.
