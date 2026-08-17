# Cross-origin requests and the BFF proxy

Next.js 16.3, Django 4.0 or later, django-cors-headers, DRF 3.17. This file
owns what the browser does when the frontend origin and the Django origin
differ. It also owns the CSRF token that a session request carries, and the
proxy in Next that removes the difference.

The one client that sends the request is
`references/api-client-and-request-safety.md`. Where the call belongs, and the
decision between a Route Handler and a Server Action, are
`references/data-access-and-mutations.md`. The rules for `proxy.ts` are
`references/app-router-structure.md`.

## Principle

The browser attaches a cookie under rules that the server states and the
browser enforces. Read the rules before you debug the request.

A preflight is a separate request. A failure in it never reaches the view, so
the server log is empty and the console holds the only report.

Same origin removes a class of failure. It does not fix one failure, and that
is why it is the better default for a session cookie.

A proxy that takes its destination from the request is a door onto the internal
network. Fix the destination in code.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### Choose the topology first

| Condition | Topology | It reverses when | The cost |
| --- | --- | --- | --- |
| Session-cookie auth, and the two origins differ | Same origin, through a Next rewrite or a Route Handler proxy | The auth strategy moves to a token in a header, which the next row covers. | Every browser request passes through the Node process, so that process must carry the traffic of the API. |
| Token auth in an `Authorization` header, and CORS is configured | The browser calls Django directly | The token moves into a cookie, or a request starts to need a secret header. | The CORS settings become a permanent maintenance item, and the Django address is public. |
| The internal address must stay hidden, or a secret header must be added | A Route Handler proxy, which runs on the server only | Never, while the address or the header must stay on the server. | One route to write and maintain for each endpoint that the browser reaches. |
| A mutation from a server context | A Server Action that calls Django with the server base URL | The caller is outside the application, so a Route Handler serves it. | The action is a public endpoint, and it must verify the session inside itself. |

`references/data-access-and-mutations.md` holds the table that decides where a
call lives, and that table stays the authority. This section decides the
topology under it, and the topology is a property of the auth strategy.

The same-origin choice removes the preflight, the `Access-Control-*` headers,
the `SameSite` rule, and the third-party cookie policy of the browser. Four
classes of failure disappear together, and none of them can return.

### CORS, as the browser reports it

| The console or the network tab shows | Cause | What the frontend does |
| --- | --- | --- |
| "No 'Access-Control-Allow-Origin' header is present" | The origin is absent from `CORS_ALLOWED_ORIGINS` | Name the exact origin, with its scheme and its port, and ask for the entry |
| The response is opaque, and the request carries `credentials: "include"` | The server answered `Access-Control-Allow-Origin: *` | Ask for the explicit origin and for `CORS_ALLOW_CREDENTIALS`. The wildcard with credentials is forbidden by the specification, and the browser rejects it |
| A custom header makes a working request fail | The header is absent from `CORS_ALLOW_HEADERS` | Name the header, such as `X-CSRFToken` or `Idempotency-Key` |
| The response carries two `Access-Control-Allow-Origin` values | Django and the reverse proxy both add CORS | Ask for one layer only. The browser rejects two values |
| The request never appears in the Django log | The preflight `OPTIONS` failed | Read the `OPTIONS` response in the network tab, never the view |

The frontend states the origin, the header, and the method that it needs. The
sibling skill `secure-code-auditor` owns the server settings and their risk.
NEVER work around a CORS failure with a proxy that forwards an arbitrary host.

### CSRF for session auth

```ts
// Wrong: the POST carries the session cookie and no token.
// Failure: Django answers 403 with the detail "CSRF Failed: CSRF token
// missing". The user sees a generic error, and the request never reaches the
// view, so the server log names no application fault.
await fetch("/api/things/", {
  method: "POST",
  credentials: "include",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(payload),
});
```

```ts
// Correct: the token comes from the cookie, and it goes in the header.
function readCookie(name: string): string | null {
  const match = document.cookie.match(new RegExp("(^|; )" + name + "=([^;]*)"));
  const value = match?.[2];
  return value === undefined ? null : decodeURIComponent(value);
}

await fetch("/api/things/", {
  method: "POST",
  credentials: "include",
  headers: {
    "Content-Type": "application/json",
    "X-CSRFToken": readCookie("csrftoken") ?? "",
  },
  body: JSON.stringify(payload),
});
```

The cookie must exist before the first write. Call an endpoint that carries the
`ensure_csrf_cookie` decorator, once, at the start of the session. The
`Set-Cookie` header then arrives with the response.

Four facts govern the exchange. Django reads the token from the
`csrfmiddlewaretoken` form field, and it falls back to the `X-CSRFToken`
header. `CSRF_HEADER_NAME` sets that header name, and the default is
`HTTP_X_CSRFTOKEN`. Django also checks the `Origin` header of the request, and
it checks the `Referer` header on HTTPS. Since Django 4.0 every value in
`CSRF_TRUSTED_ORIGINS` must carry its scheme, and a bare host name is rejected.

A same-origin topology still needs the token. It removes the CORS layer and the
`SameSite` problem, and Django still enforces CSRF on every unsafe method.
`references/api-client-and-request-safety.md` states that a multipart request
sets no `Content-Type`, and the `X-CSRFToken` header still applies to it.

### The proxy Route Handler

```ts
// Wrong: the destination comes from the request.
// Failure: this is server-side request forgery. An attacker points ?target=
// at the link-local address 169.254.169.254 and reads the cloud metadata
// service, which holds the credentials of the instance. The proxy runs on the
// server, so it reaches every address the server reaches.
export async function POST(request: NextRequest) {
  const target = request.nextUrl.searchParams.get("target")!;
  return fetch(target, { method: "POST", body: await request.text() });
}
```

```ts
// Correct: app/api/things/route.ts. The upstream is fixed in code.
import { NextResponse, type NextRequest } from "next/server";

const UPSTREAM = process.env.DJANGO_URL; // server only, never NEXT_PUBLIC_
const MAX_BODY = 1_000_000;

export async function POST(request: NextRequest) {
  const body = await request.text();
  if (body.length > MAX_BODY) return new NextResponse(null, { status: 413 });

  const upstream = await fetch(`${UPSTREAM}/api/things/`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      cookie: request.headers.get("cookie") ?? "",
    },
    body,
    signal: AbortSignal.timeout(15_000),
  });

  return new NextResponse(upstream.body, {
    status: upstream.status,
    headers: {
      "content-type": upstream.headers.get("content-type") ?? "application/json",
    },
  });
}
```

Four rules hold for every proxy route. The upstream host is a constant in the
code, and it never comes from a parameter, a header, or a body. Each exported
function serves one method, so no allowlist is needed. The body has a size cap,
and the request has a timeout. Only the headers that the upstream needs are
forwarded.

A rewrite in `next.config.ts` serves the same purpose with no code, and it
suits a plain pass-through. Take the Route Handler where the request needs a
secret header, a size cap, or a rewritten response.

WARNING: `proxy.ts` is a different file with a different job. It rewrites,
redirects, and sets headers, and it never authorizes.
`references/app-router-structure.md` owns it, and it records CVE-2025-29927,
where a forged `x-middleware-subrequest` header skipped the file completely.

### What the frontend states about the cookie

The frontend does not set the cookie attributes. It states which attributes the
application depends on, and the backend team sets them.

| Attribute | Why the frontend depends on it |
| --- | --- |
| `httpOnly` on the session cookie | No script reads it, so the token cannot leave through an injection |
| `Secure` | The cookie travels on HTTPS only |
| `SameSite` | It decides whether a cross-site request carries the cookie at all |
| The domain | It decides which subdomain receives the cookie |

A change to any of these breaks a direct browser call, and the frontend sees a
401 that looks like an application fault. Read the `Set-Cookie` header in the
network tab before you change any frontend code. The sibling skill
`secure-code-auditor` owns the server-side choice of each attribute.

## Verification

```bash
# 1. Read the preflight and the CORS headers of the real backend.
curl -i -X OPTIONS "$NEXT_PUBLIC_API_BASE_URL/api/things/" \
  -H "Origin: $APP_ORIGIN" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: content-type,x-csrftoken"

# 2. Confirm that no response carries the wildcard with credentials.
curl -si "$NEXT_PUBLIC_API_BASE_URL/api/things/" | rg -i 'access-control-allow-(origin|credentials)'

# 3. Confirm that the CSRF cookie arrives.
curl -si "$NEXT_PUBLIC_API_BASE_URL/api/csrf/" | rg -i 'set-cookie'

# 4. Find a write that carries credentials and no token. Read every hit.
rg -n -A6 "credentials: ['\"]include" src/ | rg -v 'X-CSRFToken'

# 5. Find a proxy route that reads its destination from the request. This must
#    print nothing.
rg -n 'searchParams.get\(|headers.get\(' -g '**/route.ts' src/ | rg -i 'url|target|host|upstream|redirect'

# 6. Find a proxy route with no timeout. Read every hit.
rg -l 'fetch\(' -g '**/route.ts' src/ | xargs rg --files-without-match 'AbortSignal'

# 7. Confirm that the upstream address is server-only.
rg -n 'NEXT_PUBLIC_.*(DJANGO|UPSTREAM|INTERNAL)' src/

# 8. Read the session cookie attributes that the backend sets.
curl -si -X POST "$NEXT_PUBLIC_API_BASE_URL/api/auth/login/" | rg -i 'set-cookie'
```

## Review checklist

- [ ] Is the topology decided from the auth strategy, and is it recorded?
- [ ] Does a session-cookie application run same origin, through a rewrite or a
      Route Handler?
- [ ] Is `Access-Control-Allow-Origin` an explicit origin wherever the request
      carries credentials?
- [ ] Is CORS added in one layer only?
- [ ] Does every unsafe request under session auth carry `X-CSRFToken`?
- [ ] Does the application obtain the `csrftoken` cookie before the first
      write?
- [ ] Does every value in `CSRF_TRUSTED_ORIGINS` carry its scheme?
- [ ] Is the upstream host of every proxy route a constant in the code?
- [ ] Does every proxy route cap the body size and set a timeout?
- [ ] Does a proxy route forward only the headers that the upstream needs?
- [ ] Is the upstream address absent from every `NEXT_PUBLIC_` variable?
- [ ] Are the session cookie attributes that the frontend depends on recorded?
- [ ] Is `proxy.ts` free of every authorization decision?

## Handoffs

- The one client, the base URLs, the timeout, and `normalizeApiError` →
  `references/api-client-and-request-safety.md`.
- The schema, the generator, and the drift gate →
  `references/openapi-schema-and-codegen.md`.
- The decision between a Server Component fetch, a Route Handler, and a browser
  call → `references/data-access-and-mutations.md`.
- `proxy.ts`, its permitted work, CVE-2025-29927, and the `NEXT_PUBLIC_` prefix
  → `references/app-router-structure.md`.
- The session strategy, the token store, the cookie prefixes, the lifetime,
  and the refresh after a 401 →
  `references/session-and-token-lifecycle.md`. That file owns the whole
  attribute set of the session cookie; the table above states the four
  attributes that a cross-origin request depends on. The redirect after a 401,
  and the gate on the route, are
  `references/route-protection-and-permissions.md`.
- The CSP that must name the Django origin and the `wss://` endpoint, and the
  response headers → `references/security-headers-and-csp.md`. The judgment of
  an injection sink is `references/untrusted-markup-and-injection.md`. That
  domain holds a veto.
- The Nginx configuration in front of Node, and the TLS termination → domain 22
  `build-deploy-and-runtime-ops`. Not integrated yet.
- The DRF permission class, the server-side CSRF enforcement, the CORS
  settings, and the cookie attributes → the sibling skill
  `secure-code-auditor`. This file owns what the browser sends and stores, and
  the UI behavior for a 401, a 403, and a CSRF failure.
