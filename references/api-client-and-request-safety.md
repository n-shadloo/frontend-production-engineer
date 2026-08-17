# API client and request safety

openapi-fetch 0.17, openapi-typescript 7.13, TypeScript 5.9, against a Django
and DRF backend. This file owns the one client that every request passes
through. It also owns the rules that make a request safe to send, and the
normalizer that gives every failure one shape.

The schema behind the client, and the generator that reads it, are
`references/openapi-schema-and-codegen.md`. The cookie, the preflight, and the
proxy are `references/cross-origin-and-bff-proxy.md`. Where the call belongs in
the application is `references/data-access-and-mutations.md`.

## Principle

One client holds the defaults. A second client is a second set of defaults, and
only one of the two was reviewed.

A request with no deadline is a resource with no owner. It holds a socket until
the runtime gives up, and nothing reports the wait.

A retry repeats a write, unless the method or a key promises otherwise. Decide
which requests may repeat before you add the retry.

An error that reaches a component in the shape the server chose couples the
component to the server. Normalize once, at the edge, and the UI reads one
shape.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### One client, and the path comes from the schema

```ts
// Wrong: the component holds the URL.
// Failure: a missing trailing slash gives a 301 that drops the POST body, a
// typo gives a 404, and no type checks the parameters. The base URL, the
// credentials, and the timeout are set in one file and absent in the next.
const res = await fetch("/api/users/");
const users = await res.json();
```

```ts
// Correct: src/lib/api/client.ts holds the one client, typed by the schema.
import createClient from "openapi-fetch";
import type { paths } from "@/api/generated/schema";

export const api = createClient<paths>({ baseUrl: apiBase });
```

```ts
// Correct: the call names a path that the schema declares.
const { data, error, response } = await api.GET("/api/users/{id}/", {
  params: { path: { id } },
});
if (error) throw normalizeApiError(response.status, error);
```

The path is checked against the schema, so a typo and a missing slash are
compile errors. Every request goes through this module.
`references/data-access-and-mutations.md` owns which module calls it, and it
states that a data access module imports `server-only`.

### The trailing slash

Django `APPEND_SLASH` redirects a GET that has no trailing slash. It cannot
redirect a POST, because the body does not survive the redirect. Django raises
a `RuntimeError` that names the URL and the method.

Establish the behavior of the real backend once. Encode the exact form in the
schema paths, and the generated client then carries it everywhere. NEVER decide
the slash for each call.

### Two base URLs

| Variable | Where it is read | Value |
| --- | --- | --- |
| `DJANGO_URL` | The server only | The internal address, such as the Docker service name |
| `NEXT_PUBLIC_API_BASE_URL` | The browser | The public address of the API |

```ts
// Wrong: one variable serves both sides.
// Failure: inside Docker the server-side fetch resolves the public host, or
// it resolves localhost, and the request fails with ECONNREFUSED. The browser
// works, so the failure looks like a server bug rather than a config bug.
export const apiBase = process.env.NEXT_PUBLIC_API_BASE_URL;
```

```ts
// Correct: the side of the boundary decides the value.
export const apiBase =
  typeof window === "undefined"
    ? process.env.DJANGO_URL
    : process.env.NEXT_PUBLIC_API_BASE_URL;
```

`references/app-router-structure.md` owns the `NEXT_PUBLIC_` prefix, and it
states that a variable with the prefix enters the browser bundle. NEVER put an
API token or a secret header value in a `NEXT_PUBLIC_` variable. A request that
needs a secret starts on the server.

### Every request carries a deadline and a signal

```ts
// Wrong: the request has no timeout and no cancel.
// Failure: a slow backend holds the socket open. The user navigates away and
// the request continues, so a route that mounts and unmounts in a loop leaves
// a growing set of live requests.
const { data } = await api.GET("/api/users/{id}/", { params: { path: { id } } });
```

```ts
// Correct: the caller's signal and a timeout are combined.
export async function getShipment(id: string, caller: AbortSignal) {
  const signal = AbortSignal.any([caller, AbortSignal.timeout(10_000)]);
  const { data, error, response } = await api.GET("/api/shipments/{id}/", {
    params: { path: { id } },
    signal,
  });
  if (error) throw normalizeApiError(response.status, error);
  return data;
}
```

Two failures arrive as an exception rather than as a status. An abort rejects
with a `DOMException`, and a network or CORS failure rejects with a
`TypeError`. Neither carries a status, so neither is an `ApiError`. Let both
reach the error boundary.

### Retry only what is safe to repeat

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| GET, HEAD, PUT, or DELETE fails with 429, 503, or a network error | Retry with an exponential backoff and jitter. Obey `Retry-After`. Cap the attempts at 3. | The endpoint is not idempotent, and the method name does not show it. A DELETE that decrements a counter is one such endpoint. | The user waits for the sum of the backoff periods. Each retry adds load to a backend that is already at its limit. |
| POST or PATCH, with no idempotency key | NEVER retry. | The backend accepts an idempotency key, which is the next row. | The user reads a failure on a write that may have succeeded, and the user repeats it by hand. |
| POST with an `Idempotency-Key` header that the backend accepts | Retry is allowed. | The backend no longer accepts the key, or one key carries two different bodies. | The client must make a key for each write and hold it across the attempts. |
| Any 4xx other than 429 | NEVER retry. The result is deterministic. | Never. A deterministic answer does not change on a second request. | The user reads the failure at once, which is the correct result. |

A generic retry wrapper over every method is the common failure. It creates a
second order, a second charge, or a second row, and the user sees one of them.

DRF sends `Retry-After` with a 429 when a throttle rejects the request. Read
that header, and wait the period it states. A retry that ignores it produces a
loop of 429 responses, and the throttle rejects every one of them.

### One error normalizer

```ts
// Wrong: the component reads the body that the server sent.
// Failure: a 400 field dictionary renders as "[object Object]" in a toast.
// The field errors never reach the form, so the user sees no message beside
// the input that is wrong.
catch (e) { toast((e as Error).message); }
```

```ts
// Correct: src/lib/api/errors.ts gives every failure one shape.
export interface ApiError {
  status: number;
  code: string;                          // the DRF code, never the message
  message: string;                       // a fallback for a person to read
  fieldErrors?: Record<string, string[]>;
  retryable: boolean;
  raw: unknown;
}

const RETRYABLE = new Set([429, 502, 503, 504]);

export function normalizeApiError(status: number, body: unknown): ApiError {
  const retryable = RETRYABLE.has(status);
  const isRecord =
    body !== null && typeof body === "object" && !Array.isArray(body);

  // DRF 400: { field: ["msg"], non_field_errors: ["msg"] }
  if (status === 400 && isRecord) {
    const fieldErrors: Record<string, string[]> = {};
    collectFieldErrors(body, "", fieldErrors);
    return {
      status,
      code: "validation_error",
      message: fieldErrors["non_field_errors"]?.[0] ?? "Correct the fields that are marked.",
      fieldErrors,
      retryable: false,
      raw: body,
    };
  }

  // DRF { "detail": "..." } for 401, 403, 404, 405, and 429
  if (isRecord && "detail" in body) {
    const detail = body as { detail: unknown; code?: unknown };
    return {
      status,
      code: String(detail.code ?? status),
      message: String(detail.detail),
      retryable,
      raw: body,
    };
  }

  return { status, code: String(status), message: "The request failed.", retryable, raw: body };
}

// A nested serializer produces { address: { city: ["msg"] } }, and a list
// serializer produces one entry for each row. Both flatten to the dotted path
// that names the form control, such as address.city and items.1.sku.
function collectFieldErrors(
  node: unknown,
  path: string,
  out: Record<string, string[]>,
): void {
  const join = (part: string): string => (path === "" ? part : `${path}.${part}`);

  if (typeof node === "string") {
    if (path !== "") out[path] = [node];
    return;
  }
  if (Array.isArray(node)) {
    const messages = node.filter((item): item is string => typeof item === "string");
    if (messages.length === node.length) {
      if (path !== "" && messages.length > 0) out[path] = messages;
      return;
    }
    node.forEach((item, index) => collectFieldErrors(item, join(String(index)), out));
    return;
  }
  if (node !== null && typeof node === "object") {
    for (const [key, value] of Object.entries(node)) {
      collectFieldErrors(value, join(key), out);
    }
  }
}
```

Prefer the DRF `code` over the message. A DRF `ErrorDetail` carries both, and
the message is translated, so a branch on the message breaks under a second
locale.

Three DRF shapes reach this function. A `ValidationError` detail is a
dictionary, a list, or a nested structure. `collectFieldErrors` walks all three
into one flat map of dotted paths, and the last return catches the non-JSON
form.

The dotted path is what a form control carries. A nested field therefore
arrives as `address.city`, and a row of a list arrives as `items.1.sku`.
Confirm the list shape against the deployed DRF version, because 3.17 changed
the error output of a list serializer. Where the backend runs
`drf-standardized-errors` the envelope is different, and
`references/boundary-validation-and-api-types.md` holds
`StandardizedErrorBody`. Map that envelope in the same function, and keep one
`ApiError` at the output.

A 400 becomes `fieldErrors`, and those errors belong beside the input.
`references/suspense-and-actions.md` states that a validation error returns as
state and never throws. This file owns the dotted path, and
`references/form-submission-and-server-errors.md` owns the map from that path
to a form control.

### The empty body

```ts
// Wrong: every response is parsed as JSON.
// Failure: a 204 from a DELETE has no body, so .json() throws "Unexpected end
// of JSON input". The delete succeeded, and the UI reports an error.
const data = await response.json();
```

```ts
// Correct: the status and the content type decide.
if (response.status === 204) return null;
const type = response.headers.get("content-type") ?? "";
const data = type.includes("application/json")
  ? await response.json()
  : await response.text();
```

A 500 from Django in production returns HTML, not JSON. The content-type test
catches it, and the normalizer produces a readable `ApiError` rather than a
parse error.

### The paginated response

`references/boundary-validation-and-api-types.md` owns the `Paginated<T>` type.
This file owns how the next page is requested.

```ts
// Wrong: the client computes the next page.
// Failure: it breaks when the backend moves to CursorPagination, where the
// cursor is opaque and no page number exists. It also breaks on
// LimitOffsetPagination, where the parameter is an offset and not a page.
const next = await api.GET("/api/orders/", { params: { query: { page: page + 1 } } });
```

```ts
// Correct: the server states the next URL, and the client follows it.
export async function fetchNextPage(
  nextUrl: string | null,
  signal: AbortSignal,
): Promise<unknown> {
  if (nextUrl === null) return null;
  const response = await fetch(nextUrl, { signal }); // an absolute URL from DRF
  const body: unknown = await response.json();
  if (!response.ok) throw normalizeApiError(response.status, body);
  return body; // the caller parses it against the schema of that endpoint
}
```

`PageNumberPagination` and `LimitOffsetPagination` send `count`, `next`,
`previous`, and `results`. `CursorPagination` sends no `count`, and it permits
no jump to a page. Read the style from the schema once for each endpoint. Give
the UI no page control that the style cannot serve.

### A file upload

```ts
// Wrong: the code sets the multipart content type.
// Failure: the header then carries no boundary parameter, so Django parses no
// part. request.FILES is empty and the endpoint returns a 400 that names a
// required file the browser did in fact send.
await api.POST("/api/media/", {
  body: formData,
  headers: { "Content-Type": "multipart/form-data" },
});
```

```ts
// Correct: the runtime sets the header and its boundary.
const form = new FormData();
form.append("file", file);
form.append("caption", caption);
await api.POST("/api/media/", { body: form }); // no Content-Type here
```

`references/file-upload-and-transport.md` owns the file picker, the progress
bar, and the retry of a part. This file owns the request that carries the file.

## Verification

```bash
# 1. Find a fetch outside the client module. Read every hit.
rg -n "fetch\(['\"\`]" src/ -g '!src/lib/api/**'

# 2. Find a URL literal in a component. This must print nothing.
rg -n "['\"]/api/" src/app src/features src/components

# 3. Find a request with no signal. Read every hit.
rg -n 'api\.(GET|POST|PATCH|PUT|DELETE)\(' -A4 src/ | rg -v 'signal'

# 4. Find a retry over a POST. This must print nothing.
rg -n -B4 'retry' src/lib/api | rg 'POST|PATCH'

# 5. Find a raw error body in a component. This must print nothing.
rg -n 'toast\(.*error|e\.message|err\.message' src/app src/features

# 6. Confirm that the normalizer covers each DRF shape.
pnpm vitest run src/lib/api/errors.test.ts

# 7. Find a .json() with no status guard. Read every hit.
rg -n -B3 '\.json\(\)' src/lib/api

# 8. Find a manual multipart header. This must print nothing.
rg -n 'multipart/form-data' src/

# 9. Confirm that the two base URLs are separate values.
rg -n 'DJANGO_URL|NEXT_PUBLIC_API_BASE_URL' src/lib/api
```

## Review checklist

- [ ] Does every request go through the one typed client?
- [ ] Is every path taken from the generated schema, with no URL literal in a
      component?
- [ ] Does the trailing slash of every path match the backend?
- [ ] Are the server base URL and the browser base URL separate values?
- [ ] Is every secret absent from a `NEXT_PUBLIC_` variable?
- [ ] Does every request carry a timeout and an abort signal?
- [ ] Is a retry limited to an idempotent method, or to a POST with an
      idempotency key?
- [ ] Does a retry obey `Retry-After` and cap its attempts?
- [ ] Does every failure pass through `normalizeApiError` before a component
      reads it?
- [ ] Does the normalizer prefer the DRF `code` over the message?
- [ ] Does a 400 produce `fieldErrors` rather than one string?
- [ ] Is a 204 and an empty body handled without a JSON parse?
- [ ] Does the code follow the `next` URL rather than compute a page?
- [ ] Does a multipart request leave the `Content-Type` to the runtime?

## Handoffs

- The schema, the generator, the drift gate, and the case convention →
  `references/openapi-schema-and-codegen.md`.
- The cookie on a cross-site request, the CSRF header, and the proxy →
  `references/cross-origin-and-bff-proxy.md`.
- The `Paginated<T>` type, the error envelope types, and the parse that proves
  a response → `references/boundary-validation-and-api-types.md`.
- Which module calls this client, and the order inside a Server Action →
  `references/data-access-and-mutations.md`.
- The `NEXT_PUBLIC_` prefix and the browser bundle →
  `references/app-router-structure.md`.
- The validation error that returns as state, and the 5xx that throws →
  `references/suspense-and-actions.md`.
- The `queryFn`, the query keys, the `staleTime`, and the mutation state built
  on this client → `references/server-state-and-query-cache.md`. That file
  applies `ApiError.retryable` to the retry option of a query.
- The single-flight refresh, the token store, and the status that ends a
  session → `references/session-and-token-lifecycle.md`. This file owns only
  the 401 as an `ApiError`, and no retry rule here may repeat a refresh. The
  redirect after a 401 is
  `references/route-protection-and-permissions.md`.
- The push transport, the connection that holds it open, and the streamed
  response in NDJSON → `references/push-transport-and-connection.md`. This
  file owns the deadline on an ordinary request, and a streamed response has
  no single deadline. The parse over each frame is
  `references/live-events-and-cache-merge.md`.
- The map from a dotted path to a form control, and the form-level region that
  takes `non_field_errors` →
  `references/form-submission-and-server-errors.md`.
- The file picker, the progress bar, and the upload interface →
  `references/file-upload-and-transport.md`.
- The words that a person reads in an error message → domain 15
  `ux-writing-and-content-design`. Not integrated yet.
- The throttle rate, the N+1 query, and the latency behind a slow endpoint →
  the sibling skill `django-performance-optimizer`.
