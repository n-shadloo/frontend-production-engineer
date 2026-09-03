# Security headers and the Content Security Policy

Next.js 16.3 with `proxy.ts` on the Node runtime, React 19.2.6 or later,
TypeScript 5.9, against a Django and DRF backend. This file owns the response
headers that the application emits, and the Content Security Policy above them.
The subjects are the header set, the per-request nonce, and the policy that a
static route can hold. They also include the directive that admits the Django
origin, the directive that stops a frame, and the order in which a policy reaches
enforcement.

The sink that a policy is the second defence for is
`references/untrusted-markup-and-injection.md`. The endpoint behind the policy is
`references/exposed-endpoints-and-destinations.md`.

## Principle

A header is a rule that the browser enforces. It works on a page that the review
missed, and it costs the reader nothing.

A policy is a second defence. It never replaces the escape and the sanitiser, and
a team that treats it as the first defence ships the sink anyway.

Two layers that both set one header produce a value that nobody predicted. One
layer owns the header set.

A policy that allows every inline script allows the injected one. The allowance
is the whole attack surface.

A policy that no report tested breaks the application on the day it enforces.
Measure first, then enforce.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but in
decline, and alive only in legacy code.

### The header set

| Header | What it stops | The value to ship |
| --- | --- | --- |
| `Content-Security-Policy` | A script that the page did not intend to run | A nonce-based policy. The next sections build it. |
| `Strict-Transport-Security` | A downgrade of the connection to plain HTTP | `max-age=63072000; includeSubDomains; preload` |
| `X-Content-Type-Options` | The browser guessing a type that the server did not state | `nosniff` |
| `Referrer-Policy` | A path and a query string leaking to another origin | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | A frame or a script reaching the camera, the microphone, or the position | An explicit deny list for every feature that the product does not use |
| `frame-ancestors`, inside the policy | Another site framing the application, so a press lands on a hidden control | `'none'`, or an explicit origin list |
| No `x-powered-by` | The response that names the framework | `poweredByHeader: false` in `next.config.ts` |

`frame-ancestors` supersedes `X-Frame-Options`. Keep `X-Frame-Options: DENY`
beside it only where the product must serve a browser that reads no policy.

One layer emits the set. A header that both the application and the reverse
proxy set arrives twice. The browser then resolves the pair by its own rule, and
not by the intention of the team.
`references/runtime-process-and-reverse-proxy.md` owns which layer that is. The
default for this stack is the application, because `proxy.ts` is the only
source of a per-request nonce. Confirm that the proxy adds none of these
headers.

Read the headers of the deployed application after every deploy. A header that a
configuration states and a response omits is the common failure, and no test in
the repository catches it.

### The nonce comes from `proxy.ts`

```ts
// Wrong: the policy admits every inline script.
// Failure: 'unsafe-inline' allows the injected <script> as readily as the one
// the page wrote. The policy is present in the response and stops nothing.
"Content-Security-Policy": "script-src 'self' 'unsafe-inline'",
```

```ts
// Correct: proxy.ts. One nonce for each request, on the request and on the
// response.
import { NextResponse, type NextRequest } from "next/server";

export function proxy(request: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString("base64");

  const policy = `
    default-src 'self';
    script-src 'self' 'nonce-${nonce}' 'strict-dynamic';
    style-src 'self' 'nonce-${nonce}';
    img-src 'self' blob: data:;
    font-src 'self';
    connect-src 'self' ${process.env.DJANGO_ORIGIN} ${process.env.WS_ORIGIN};
    object-src 'none';
    base-uri 'self';
    form-action 'self';
    frame-ancestors 'none';
    upgrade-insecure-requests;
  `
    .replace(/\s{2,}/g, " ")
    .trim();

  const headers = new Headers(request.headers);
  headers.set("x-nonce", nonce);
  headers.set("Content-Security-Policy", policy);

  const response = NextResponse.next({ request: { headers } });
  response.headers.set("Content-Security-Policy", policy);
  return response;
}
```

```tsx
// Correct: the page reads the nonce and hands it to the script.
import { headers } from "next/headers";
import Script from "next/script";

export default async function Page() {
  const nonce = (await headers()).get("x-nonce") ?? undefined;
  return <Script src={analyticsSrc} strategy="afterInteractive" nonce={nonce} />;
}
```

`'strict-dynamic'` lets a script that the nonce admits load the scripts that it
needs. Without it, every chunk of the application needs its own entry in the
policy, and the list goes stale on the next build.

`references/app-router-structure.md` owns `proxy.ts` and states the work that the
file may do. A response header is permitted work. NEVER add a session read or a
data call beside the nonce.

`next.config.ts` emits a static value for each of the other headers. It cannot
emit the policy, because the policy carries a value that changes for each
request.

### A static route holds no nonce

| The route | The policy | The cost |
| --- | --- | --- |
| Dynamic, so it renders for each request | A nonce, `'strict-dynamic'`, and no `'unsafe-inline'` | The route stays dynamic, which it already is |
| Static, incrementally revalidated, or partially prerendered | A hash for each inline script, or a strict policy with no inline script at all | The team maintains the hash list |

```tsx
// Wrong: a prerendered route reads the nonce of the build.
// Failure: the value is baked into the static file. Every reader receives the
// same nonce, an attacker downloads the page and reads it, and the injected
// script carries the value that the policy admits.
export const dynamic = "force-static";

export default async function Page() {
  const nonce = (await headers()).get("x-nonce") ?? undefined;
  return <Script src={widgetSrc} nonce={nonce} />;
}
```

```ts
// Correct: next.config.ts emits a fixed policy for the prerendered routes, and
// a hash covers each inline script.
import type { NextConfig } from "next";

const STATIC_POLICY = [
  "default-src 'self'",
  "script-src 'self' 'sha256-THE_HASH_THAT_THE_BUILD_REPORTS'",
  "object-src 'none'",
  "base-uri 'self'",
  "frame-ancestors 'none'",
].join("; ");

const nextConfig: NextConfig = {
  poweredByHeader: false,
  async headers() {
    return [
      {
        source: "/(marketing|about|pricing)/:path*",
        headers: [{ key: "Content-Security-Policy", value: STATIC_POLICY }],
      },
    ];
  },
};

export default nextConfig;
```

A nonce needs a per-request render. A route that Next.js prerenders holds the
nonce of the render that produced the file. Every later reader then receives the
same value, which is no protection at all.

NEVER make a route dynamic to obtain a nonce. The static render is a performance
property that `references/performance-budgets-and-measurement.md` measures, and a
hash-based policy protects the same route at no such cost.

### `connect-src` names the Django origin

A strict policy blocks every request that a directive does not admit. `fetch`,
`XMLHttpRequest`, and a WebSocket handshake all read `connect-src`.

An application that talks to Django on another origin therefore needs that origin
in `connect-src`, and a `wss://` endpoint needs its own entry. This is the most
common way that a correct policy breaks a working application.

Take the origin from an environment variable, so the development origin, the
staging origin, and the production origin each reach the policy without an edit.

`references/cross-origin-and-bff-proxy.md` owns the topology decision. An
application that proxies Django through a Route Handler needs `'self'` alone,
because every request is same-origin. An application that calls Django directly
needs the Django origin in the directive, and it needs the CORS headers that the
same file describes. The two rules are separate, and both must hold.

`references/push-transport-and-connection.md` owns the WebSocket. The policy must
admit its origin, or the handshake fails before the connection opens.

### Report first, then enforce

Ship `Content-Security-Policy-Report-Only` for a week. Read the reports, and
correct every violation that first-party code produces. Then rename the header
and enforce.

Expect zero violations from first-party code at the end of that week. A violation
from a third-party script is a decision about that script, and
`references/secret-boundary-and-supply-chain.md` states it.

Grade the deployed application with an external checker. Read the header set
with a header grader, such as securityheaders.com or Mozilla Observatory. Read
the policy itself with the Google CSP Evaluator. Target a grade of A.

WARNING: development needs `'unsafe-eval'` in `script-src`, because React uses
`eval` for its development tooling. Gate that allowance on the development
environment. A production response that carries `'unsafe-eval'` is a failed
review.

An exception for `'unsafe-inline'` or `'unsafe-eval'` in production needs a
written record and a date on which it ends. An exception with no date is a
permanent hole.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those facts on 17 August 2026. A cell that holds no
date is a package with a current registry entry on that date. This file does not
state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `proxy.ts`, with the Web Crypto API | The only place that a per-request nonce comes from. It needs no package. | Next.js 16.3 | 2026-08-03 | Vercel, active | None on 16.3 |
| Recommend | `headers()` in `next.config.ts` | Every header whose value does not change for each request. | Next.js 16.3 | 2026-08-03 | Vercel, active | None on 16.3 |
| Recommend | `poweredByHeader: false` | Every project. It removes the header that names the framework. | Next.js 16.3 | 2026-08-03 | Vercel, active | None on 16.3 |
| Conditional | `next/script` with a `nonce` prop | Only where a nonce-based policy is already in place. | Next.js 16.3 | 2026-08-03 | Vercel, active | None on 16.3 |
| Conditional | Trusted Types, through `require-trusted-types-for 'script'` | Only as a second layer, and only after a report-only run. `references/untrusted-markup-and-injection.md` states the rule. | Web platform | — | Baseline since 2026-02 | — |
| Reject | `'unsafe-inline'` or `'unsafe-eval'` in a production `script-src` | The policy then admits the injected script. Take a nonce with `'strict-dynamic'`. | — | — | — | — |
| Reject | A header that the application and the reverse proxy both set | The response carries two values, and the browser picks one. | — | — | — | — |
| Reject | A forced dynamic render, taken only to obtain a nonce | It spends the static render for no gain. Take a hash-based policy. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| Every API call fails after the policy ships | `connect-src` omits the Django origin | The violation reports in the browser console | Add the Django origin and the `wss://` endpoint to the directive |
| The WebSocket never opens | `connect-src` omits the `wss://` origin | The same reports | Add the endpoint. `references/push-transport-and-connection.md` owns the connection |
| A vendor script does not run | The policy admits no script from that origin | The same reports | Decide the script first, then admit its origin or self-host it |
| The policy is present and stops nothing | `'unsafe-inline'` sits in `script-src` | Read the response header, or run the policy through an evaluator | Take a nonce with `'strict-dynamic'` |
| The nonce is the same on every response | The route is prerendered | Reload the route twice and compare the header | Take a hash-based policy for that route |
| The header set differs between two routes | Two layers emit headers | Read the headers of several routes on the deployed application | Emit the set from one layer |
| The application works in development and breaks in production | The development allowance for `'unsafe-eval'` never reached production, and inline code depended on it | Compare the two policies | Give the inline code a nonce |
| Another site frames the application | `frame-ancestors` is absent | Read the response header | Set `frame-ancestors 'none'`, or name the permitted origins |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next 16 renamed `middleware.ts` to `proxy.ts`, on the Node runtime | A `middleware.ts` file is present, so the nonce code never runs | Run `npx @next/codemod@latest rename-middleware-to-proxy .`. `references/app-router-structure.md` owns the rename |
| `frame-ancestors` supersedes `X-Frame-Options` | `X-Frame-Options` is the only clickjacking control in the configuration | Add `frame-ancestors` to the policy, and keep the old header only for a browser that reads no policy |
| Trusted Types reached a cross-browser baseline in February 2026 | An enforced policy with no report-only run before it | Move to report-only, read the reports, then enforce |

## Verification

```bash
# 1. Read the installed version before you write code.
node -p "require('next/package.json').version"

# 2. Find the policy. A project with none fails this domain.
rg -n 'Content-Security-Policy' proxy.ts src/proxy.ts next.config.ts

# 3. Find an unsafe allowance. Each hit must be gated on development.
rg -n "unsafe-inline|unsafe-eval" proxy.ts src/proxy.ts next.config.ts

# 4. Confirm that the framework header is off.
rg -n 'poweredByHeader' next.config.ts

# 5. Confirm that proxy.ts holds no session read beside the nonce. This prints
#    nothing.
rg -n 'verifySession|getSession|DJANGO_URL/api' proxy.ts src/proxy.ts

# 6. Read the header set of the deployed application.
curl -sI "$APP_ORIGIN" \
  | rg -i 'content-security-policy|strict-transport|x-content-type|referrer-policy|permissions-policy|x-powered-by'

# 7. Confirm that the nonce changes for each request. The two values differ.
curl -sI "$APP_ORIGIN" | rg -o "nonce-[A-Za-z0-9+/=]+"
curl -sI "$APP_ORIGIN" | rg -o "nonce-[A-Za-z0-9+/=]+"

# 8. Confirm that no header arrives twice.
curl -sI "$APP_ORIGIN" | rg -ic 'content-security-policy'

# 9. Load the primary route with the policy in report-only mode. The console
#    holds no violation from first-party code.

# 10. Grade the deployed application with a header grader and with a policy
#     evaluator. The target is a grade of A.
```

## Review checklist

- [ ] Does production carry a `Content-Security-Policy` header?
- [ ] Is `'unsafe-inline'` absent from the production `script-src`?
- [ ] Is `'unsafe-eval'` absent from the production `script-src`?
- [ ] Does a written exception with an end date cover any allowance that
      remains?
- [ ] Does the policy carry a nonce and `'strict-dynamic'` on a dynamic route?
- [ ] Does a hash-based policy cover each prerendered route?
- [ ] Is the nonce different on two successive responses?
- [ ] Does `connect-src` name the Django origin?
- [ ] Does `connect-src` name every `wss://` endpoint?
- [ ] Is `frame-ancestors` set to `'none'` or to an explicit origin list?
- [ ] Does the response carry `Strict-Transport-Security`,
      `X-Content-Type-Options`, `Referrer-Policy`, and `Permissions-Policy`?
- [ ] Is `poweredByHeader` set to `false`?
- [ ] Does exactly one layer emit the header set?
- [ ] Did the policy run in report-only mode before it enforced?
- [ ] Were the headers of the deployed application read after the last deploy?

## Handoffs

- The sink that the policy is the second defence for, and the sanitiser in front
  of it → `references/untrusted-markup-and-injection.md`.
- The Server Action surface, the outbound destination, and the redirect target →
  `references/exposed-endpoints-and-destinations.md`.
- The third-party script that the policy must admit, its `integrity` attribute,
  and the self-hosting decision →
  `references/secret-boundary-and-supply-chain.md`.
- `proxy.ts`, the work that it may do, and the rename from `middleware.ts` →
  `references/app-router-structure.md`. The rest of `next.config.ts` is the same
  file.
- The CORS headers, the preflight, and the proxy Route Handler in front of
  Django → `references/cross-origin-and-bff-proxy.md`.
- The `wss://` endpoint, the handshake, and the `Origin` check on it →
  `references/push-transport-and-connection.md`.
- The cookie attributes that the client depends on →
  `references/session-and-token-lifecycle.md`.
- The separate origin for user content, `Content-Disposition: attachment`, and
  `nosniff` → `references/served-content-and-downloads.md`.
- The cost of a dynamic render, and the budget that a policy must not break →
  `references/performance-budgets-and-measurement.md`. The strategy of a vendor
  script is `references/client-bundle-and-third-party-scripts.md`.
- The reverse proxy, the TLS termination, and the layer that emits the header set
  → `references/runtime-process-and-reverse-proxy.md`.
- The consent that must arrive before a tag manager renders, and the cookie
  inventory → `references/consent-gate-and-cookie-inventory.md`.
- The endpoint that receives a violation report, and the rule over its count →
  `references/correlation-and-telemetry.md`.
- The server-side headers of Django, and `ALLOWED_HOSTS` → the backend and its
  security review. This file owns the headers that the Next.js application
  emits.
