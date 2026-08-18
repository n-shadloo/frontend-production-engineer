# Exposed endpoints and outbound destinations

Next.js 16.3 with `proxy.ts` on the Node runtime, React 19.2.6 or later, Zod 4.4,
TypeScript 5.9, against a Django and DRF backend. This file owns the endpoints
that the framework publishes on behalf of the application, and the destinations
that the server code of the application reaches. The subjects are the Server
Action as a public HTTP endpoint, its identifier, the origin check over it, and
the Route Handler beside it. They also include the value that an endpoint
returns, and the outbound request whose destination someone else chose. The last
subjects are the redirect target, and the response that must never enter a
cache.

The gate inside an action is `references/route-protection-and-permissions.md`.
The order inside an action is `references/data-access-and-mutations.md`. The
policy over the page that calls the action is
`references/security-headers-and-csp.md`.

## Principle

A function that the network can reach is a public endpoint. The user interface in
front of it is not a boundary.

A framework protection is narrow by design. Read what each one covers, and never
read a protection as authentication.

An endpoint returns a payload, not a screen. Everything in that payload is
readable by whoever called it.

A destination that a request supplies is a destination that an attacker chose.
The server reaches addresses that no browser can reach.

A cached response outlives the identity that produced it. A response that one
person may read must never wait in a cache for the next person.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but in
decline, and alive only in legacy code.

### A Server Action is a public POST endpoint

| The framework does this | The framework does not do this |
| --- | --- |
| It accepts the POST method only | It authenticates the caller |
| It compares `Origin` against `Host` and `X-Forwarded-Host` | It authorizes the caller against the object |
| It encrypts the variables that the action closes over, for each build | It validates the input |
| It gives each action a non-deterministic identifier | It limits the rate of the calls |
| — | It shapes the return value |

Read the two columns as one rule. The framework stops a browser on another site
from posting the action with the cookies of a reader. It stops nothing else. A
program with a session cookie, or with no session at all, posts the action
directly.

```ts
// Wrong: the action assumes that the page in front of it decided who may call.
// Failure: the identifier of the action sits in a client chunk. A request with
// no session posts it, and the record disappears. The hidden button in the
// interface never mattered.
"use server";

export async function archiveInvoice(invoiceId: string) {
  await db.invoice.archive(invoiceId);
}
```

```ts
// Correct: the gate, the parse, and the ownership check all sit inside the
// action.
"use server";

import "server-only";
import { z } from "zod";
import { verifySession } from "@/lib/dal/session";
import { archiveInvoiceFor } from "@/lib/dal/invoices";

const Schema = z.object({ invoiceId: z.uuid() });

export async function archiveInvoice(raw: unknown) {
  const session = await verifySession();
  if (session === null) return { error: "unauthorized" };

  const parsed = Schema.safeParse(raw);
  if (!parsed.success) return { error: "invalid" };

  await archiveInvoiceFor(session, parsed.data.invoiceId); // Django owns the check
  return {};
}
```

`references/route-protection-and-permissions.md` owns the gate and the rule that
no action takes an identity as a parameter.
`references/data-access-and-mutations.md` owns the fixed order, which is
authorize, validate, mutate, invalidate, and redirect.
`references/boundary-validation-and-api-types.md` owns the parse. This file owns
the reason that all three are mandatory: the endpoint is reachable whatever the
interface shows.

Test the endpoint the way an attacker reaches it. Read the identifier from a
client chunk, post to it with no cookie, and expect a 401 or a 403. A 200 with a
mutation is a failed task.

### The identifier changes, and the deploy must survive it

Next.js creates the identifier of an action at compile time and caches it for at
most 14 days. A new build produces a new identifier, and a cache invalidation
does the same.

A reader who holds an old page therefore meets "Failed to find Server Action"
during a mutation. Handle that error and reload the page. Prefer a rolling deploy,
so both builds answer while the change lands.

Set `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` to a base64 AES key of 32 bytes on every
deployment that runs more than one instance. Without a shared key, each instance
encrypts the closed-over variables with its own key, and a request that reaches
the wrong instance fails.

CAUTION: the key is a secret of the server. `references/secret-boundary-and-supply-chain.md`
states that no such value carries the `NEXT_PUBLIC_` prefix.

### `allowedOrigins` behind a proxy or a CDN

The built-in origin check compares `Origin` against `Host` and
`X-Forwarded-Host`. A reverse proxy or a CDN rewrites those values, so a
forwarded request fails the check and the action aborts.

```ts
// Correct: next.config.ts names the origins that a forwarded request carries.
const nextConfig: NextConfig = {
  experimental: {
    serverActions: {
      allowedOrigins: ["app.example.com", "*.app.example.com"],
    },
  },
};
```

NEVER read that setting as a security control that replaces the gate.
CVE-2026-27978 affected Next.js 16.0.1 through 16.1.6 and is fixed in 16.1.7: the
framework treated an `Origin` header of `null` as an absent origin, so a request
from a sandboxed frame passed the check. The gate inside the action is what held
during that window.

### A Route Handler answers one method, with a cap

```ts
// Wrong: one function serves every method, and no limit bounds the body.
// Failure: a caller posts 200 MB to a handler that expected a small JSON
// object. The Node process buffers it, and the route becomes a denial of
// service.
async function handler(request: NextRequest) {
  return handle(request);
}

export { handler as GET, handler as POST, handler as PUT, handler as DELETE };
```

```ts
// Correct: one export for each permitted method, a cap, and the gate.
import { NextResponse, type NextRequest } from "next/server";
import { verifySession } from "@/lib/dal/session";

const MAX_BODY = 100_000;

export async function POST(request: NextRequest) {
  const session = await verifySession();
  if (session === null) return new NextResponse(null, { status: 401 });

  const body = await request.text();
  if (body.length > MAX_BODY) return new NextResponse(null, { status: 413 });

  return handle(session, body);
}
```

An unexported method answers 405 by itself, so the export list is the allowlist.
`references/cross-origin-and-bff-proxy.md` owns the proxy handler that stands in
front of Django, with its fixed upstream and its timeout.
`references/file-upload-and-transport.md` owns `serverActions.bodySizeLimit` and
the threshold at which a file stops travelling through Node.

### The return value is public to the caller

An action and a handler return a payload. A React Server Component render also
produces a payload, and the browser receives it in full.

Return the fields that the interface renders, and no others. A record with an
internal note, an audit trail, or the email address of another person reaches
the caller whole. No component has to render those fields for that to happen.

NEVER return the text of an exception. `references/error-and-empty-state-copy.md`
owns the message that the reader sees, and it states that exception text stays on
the server. A Django traceback that reaches the browser names the framework, the
file paths, and often the query.

Read the payload of each protected route with the cookies cleared, and confirm
that it holds no private value.
`references/route-protection-and-permissions.md` states that check as a gate.

### No destination comes from the request

```ts
// Wrong: the handler fetches whatever the caller names.
// Failure: server-side request forgery. A caller points the parameter at the
// link-local address 169.254.169.254 and reads the credentials of the instance.
// The handler runs on the server, so it reaches every address the server
// reaches.
export async function GET(request: NextRequest) {
  const url = request.nextUrl.searchParams.get("url")!;
  return fetch(url);
}
```

```ts
// Correct: an allowlist of hosts, one scheme, and no redirect to follow.
const ALLOWED_HOSTS = new Set(["images.example.com", "cdn.example.com"]);

export async function GET(request: NextRequest) {
  const raw = request.nextUrl.searchParams.get("url") ?? "";

  let url: URL;
  try {
    url = new URL(raw);
  } catch {
    return new NextResponse("Bad URL", { status: 400 });
  }

  if (url.protocol !== "https:" || !ALLOWED_HOSTS.has(url.hostname)) {
    return new NextResponse("Forbidden", { status: 403 });
  }

  return fetch(url, { redirect: "error" });
}
```

Three rules bind every outbound request whose destination varies. Allowlist the
host by exact name, which is what refuses a private address range and the
metadata address `169.254.169.254`. Test the protocol, so no other scheme
reaches the call. Set `redirect: "error"`, so a permitted host cannot forward
the request to one that the allowlist refuses.

Prefer no variable destination at all. `references/cross-origin-and-bff-proxy.md`
states the stronger rule for the Django proxy, where the upstream is a constant
in the code.

The same rule reaches three more surfaces.

| The surface | The failure | Who owns the configuration |
| --- | --- | --- |
| A `rewrites()` or a `redirects()` destination built from request input | CVE-2026-64645, which is a server-side request forgery and an open redirect through the configuration | `references/app-router-structure.md` owns `next.config.ts` |
| `/_next/image`, with a wildcard hostname in `remotePatterns` | The image optimizer fetches an address that the caller chose | `references/image-and-video-delivery.md` owns `remotePatterns` and `dangerouslyAllowSVG` |
| A Server Action that fetches a URL from its own input | CVE-2026-64649, which is a server-side request forgery through an action | This file |

Every destination in those three surfaces is a constant, or it passes an
allowlist. There is no third option.

### The redirect target is parsed, and then matched

A redirect that follows a request value sends the reader to another site while
the address bar still shows the application. That is an open redirect, and it is
the first step of a phishing flow.

Two checks are needed, and neither one is enough alone.

1. Refuse a value that does not start with a single `/`. This refuses an absolute
   URL, and it refuses a value such as `https:evil.example` that the URL parser
   resolves against the origin of the application.
2. Resolve the value against the origin of the application with `new URL()`, and
   compare the parsed origin. This refuses `/\evil.example`, which the parser
   normalizes into a protocol-relative URL that a string test passes.

NEVER match a raw string alone, and never match a pattern. Validate the parsed
URL object. `references/route-protection-and-permissions.md` holds the helper
that every redirect target passes through, and this file states the rule that the
helper implements.

### An authenticated response never enters a cache

A response that depends on the identity of the reader must not be cacheable at
any layer. The route, the CDN, and the reverse proxy each hold a cache, and each
one serves the stored copy to the next caller.

`Cache-Control: public` on a page of per-user content is the plain form of this
failure. The Next.js advisories CVE-2026-64647 and CVE-2026-64648 describe the
subtle form, where a cache confuses two responses.

`references/caching-and-revalidation.md` owns the cache decision of a route, and
`references/route-protection-and-permissions.md` states that a per-user route
stays dynamic and never enters a `"use cache"` scope. This file states the
consequence: a leaked cache entry is a data breach, and not a stale page.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those facts on 17 August 2026. A cell that holds no
date is a package with a current registry entry on that date. This file does not
state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `zod` | Every Server Action input and every Route Handler body. It is Standard Schema compliant. | 4.4 | Current | Active | None |
| Recommend | `server-only` | Every module that an action imports and that reads a secret. | Ships with Next.js | Current | Vercel, active | None |
| Recommend | The `URL` constructor | Every destination and every redirect target. It needs no package. | Web platform | — | — | — |
| Recommend | `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` | Every deployment that runs more than one instance. The value is a base64 AES key of 32 bytes. | Next.js 16.3 | 2026-08-03 | Vercel, active | None on 16.3 |
| Conditional | `serverActions.allowedOrigins` | Only behind a reverse proxy or a CDN, where the forwarded host differs. It is not a gate. | Next.js 16.3 | 2026-08-03 | Vercel, active | None on 16.3 |
| Reject | A `fetch`, a `rewrite`, a `redirect`, or an image URL built from request input with no allowlist | It is a server-side request forgery. CVE-2026-64645 and CVE-2026-64649 are two instances. | — | — | — | — |
| Reject | A wildcard hostname in `remotePatterns` | The optimizer then fetches an address that the caller chose. | — | — | — | — |
| Reject | A string test as the only check on a redirect target | It passes `/\evil.example`. Parse the value and compare the origin. | — | — | — | — |
| Reject | `Cache-Control: public` on a per-user response | The next caller receives the response of the previous one. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| A caller with no session mutates data | The action trusts the interface in front of it | Post to the action identifier with no cookie | Add the gate inside the action |
| One tenant reads or edits the record of another | The action trusts an identifier from the client | Change the identifier in a request, and expect a 403 | Check the ownership on the server for every object identifier |
| "Failed to find Server Action" after a deploy | The identifier rotated with the build | The error reaches the reader during a mutation | Handle the error and reload. Prefer a rolling deploy |
| An action fails only behind the CDN | The forwarded host fails the origin check | Compare the two topologies | Set `serverActions.allowedOrigins` |
| An action fails only on one instance | Each instance holds its own encryption key | Repeat the request until it reaches another instance | Set `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` |
| A handler reaches an internal service | The destination came from the request | Read every outbound `fetch` in `app/api` | Allowlist the host, and refuse a private range |
| The image endpoint fetches an internal address | `remotePatterns` holds a wildcard hostname | Read `next.config.ts` | Name each host. `references/image-and-video-delivery.md` owns the list |
| A reader lands on another site after a login | The redirect followed a request value | Press a crafted link, and watch the address bar | Parse the target and compare the origin |
| A reader sees the account of another person | A per-user response entered a cache | Read the route report, and read the cache headers | Keep the route dynamic, and set no public cache header |
| A stack trace renders in the interface | The endpoint returned the exception | Fail a request on purpose, and read the payload | Return a code, and keep the exception on the server |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| CVE-2026-27978 let an `Origin` header of `null` pass the built-in check, in Next.js 16.0.1 through 16.1.6 | `node -p "require('next/package.json').version"` reports a version below 16.1.7 | Upgrade. The gate inside the action is what holds meanwhile |
| The July 2026 security release fixed CVE-2026-64645 and CVE-2026-64649, which are two server-side request forgeries | A Next.js version from 16.0.0 to 16.2.10 | Upgrade to 16.2.11 or later on the 16 line, and to 15.5.21 on the 15 line |
| The same release fixed CVE-2026-64647 and CVE-2026-64648, which are cache confusion defects | The same version range | The same upgrade |
| CVE-2026-64643 discloses the endpoint identifier of a Server Function | The same version range | Upgrade. `references/session-and-token-lifecycle.md` records this defect |
| An action identifier is cached for at most 14 days | A deploy procedure that replaces every instance at once | Move to a rolling deploy, and handle the error in the client |

## Verification

```bash
# 1. Read the installed version before you write code.
node -p "require('next/package.json').version"

# 2. Find every Server Action. Read every hit.
rg -n '"use server"' -g '*.ts' -g '*.tsx' src/

# 3. Find an action whose file never parses its input. Read every hit.
rg --files-with-matches '"use server"' -g '*.ts' src/ \
  | xargs rg --files-without-match 'safeParse|\.parse\('

# 4. Find a Route Handler that exports a catch-all. This prints nothing.
rg -n 'export async function (handler|ALL)\b|handler as (GET|POST)' -g 'route.ts' src/app

# 5. Find an outbound fetch whose destination is a variable. Each hit needs an
#    allowlist beside it.
rg -n -B3 'fetch\((url|target|raw|dest)' -g '*.ts' src/app

# 6. Find a rewrite or a redirect built from request input. This prints nothing.
rg -n -A6 'rewrites\(|redirects\(' next.config.ts

# 7. Find a wildcard image host. This prints nothing.
rg -n "hostname:\s*['\"]\*\*" next.config.ts

# 8. Confirm that the encryption key is set, and that it is not public.
rg -n 'NEXT_SERVER_ACTIONS_ENCRYPTION_KEY' .env* | rg -v 'NEXT_PUBLIC_'

# 9. Post to a Server Action with no session. Read the identifier from a client
#    chunk, then send the request. Expect a 401 or a 403, and never a 200 with
#    a mutation.
curl -i -X POST "$APP_ORIGIN/" -H "Next-Action: $ACTION_ID"

# 10. Change an object identifier in a request to one that another account owns.
#     Expect a 403.

# 11. Press a crafted redirect link, such as one whose target starts with a
#     backslash. The reader stays on the application.

# 12. Read a protected route with the cookies cleared. The payload holds no
#     private value.
```

## Review checklist

- [ ] Does every `"use server"` function verify the session inside its own body?
- [ ] Does every `"use server"` function parse its input before any read or
      mutation?
- [ ] Does the server check the ownership of every object identifier that a
      caller supplies?
- [ ] Does every Route Handler export one function for each permitted method?
- [ ] Does every Route Handler cap the size of the body?
- [ ] Does every endpoint return the fields that the interface renders, and no
      others?
- [ ] Is exception text absent from every payload that reaches the browser?
- [ ] Is every outbound destination a constant, or does it pass a host
      allowlist?
- [ ] Does an outbound request refuse a private address range and
      `169.254.169.254`?
- [ ] Does an allowlisted request set `redirect: "error"`?
- [ ] Is `remotePatterns` free of a wildcard hostname?
- [ ] Is every `rewrites()` and `redirects()` destination free of request input?
- [ ] Does every redirect target pass both the leading-slash test and the parsed
      origin test?
- [ ] Is `serverActions.allowedOrigins` set wherever a proxy or a CDN rewrites
      the host?
- [ ] Is `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` set on every multi-instance
      deployment?
- [ ] Is every per-user response outside every cache, at the route, the CDN, and
      the proxy?
- [ ] Did somebody post to an action with no session, and read the status?

## Handoffs

- The gate inside an action, the four enforcement layers, and the redirect
  helper → `references/route-protection-and-permissions.md`. That file also
  owns the rule that no action takes an identity as a parameter.
- The fixed order inside an action, and the data access module →
  `references/data-access-and-mutations.md`.
- The parse over every value from outside the program →
  `references/boundary-validation-and-api-types.md`.
- The proxy Route Handler in front of Django, its fixed upstream, its timeout,
  and the CORS headers → `references/cross-origin-and-bff-proxy.md`.
- `next.config.ts`, `rewrites()`, `redirects()`, and `proxy.ts` →
  `references/app-router-structure.md`.
- `remotePatterns`, `dangerouslyAllowSVG`, and the image optimizer advisories →
  `references/image-and-video-delivery.md`.
- `serverActions.bodySizeLimit`, and the threshold at which a file stops
  travelling through Node → `references/file-upload-and-transport.md`.
- The cache decision of a route, `"use cache"`, `cacheTag`, and `cacheLife` →
  `references/caching-and-revalidation.md`.
- The words that a refusal renders, and the rule that exception text stays on the
  server → `references/error-and-empty-state-copy.md`.
- The action identifier disclosure of CVE-2026-64643, and the credential itself →
  `references/session-and-token-lifecycle.md`.
- The policy over the page that calls the action →
  `references/security-headers-and-csp.md`. The sink inside that page is
  `references/untrusted-markup-and-injection.md`.
- The framework security floor, and the advisory triage →
  `references/secret-boundary-and-supply-chain.md`.
- The idempotency key on a retried write, and the `ApiError` shape →
  `references/api-client-and-request-safety.md`.
- The test that posts to an action with no session →
  `references/end-to-end-journeys-and-flake-control.md`.
- The DRF permission class, the object-level check, the rate limit, and every
  server-side enforcement → the sibling skill `secure-code-auditor`. This file
  owns the endpoints that the Next.js application publishes.
