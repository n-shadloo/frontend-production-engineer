# Data access and mutations

Next.js 16.3 baseline, React 19.2.1 or later, against a Django and DRF backend.
This file owns the place where data enters the frontend and the place where a
mutation leaves it. The subjects are the data access layer and the shape of a
Server Action. They also include the choice between a Server Component fetch, a
Route Handler, and a browser fetch.

The route tree is `references/app-router-structure.md`. The server and client
split is `references/server-and-client-components.md`. Whether a response is
cacheable is `references/caching-and-revalidation.md`.

## Principle

One resource has one source per view. Either the server fetches it and passes
it down, or the client queries it. Two sources for one resource produce two
answers, and the user sees the difference.

A data access layer is the one module that talks to the backend. It holds the
base URL, the credentials, the types, and the error mapping. Code above it
calls a function, not an endpoint.

A secret that the browser can read is not a secret. A request that carries a
secret therefore starts on the server.

A mutation belongs to the code that owns the form, not to a separate endpoint
that the form must find.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### Where the call goes

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| Read data for the first render | Fetch in a Server Component through the data access layer, and pass the result down | The value changes after the mount, and the client must hold the newer value. | The value is fixed at the render, so a refresh of it needs a new request for the route. |
| A mutation from a form or a button in your own application | A Server Action, invoked by `<form action>`, so the form works without JavaScript | The caller is outside the application, which the third row covers. | Each action is a public endpoint that must verify the session again. |
| A public API for an external consumer, a webhook, a file upload, or a stream | A Route Handler in `route.ts` | The caller is a form inside the application, which the second row covers. | The route needs its own authorization, its own validation, and its own error shape. |
| A Client Component needs to refetch after the mount | Prefetch on the server, hydrate the client cache, then query on the client | Nothing on the screen needs the value again after the first render. | The payload carries the dehydrated cache, and the client bundle carries the query library. |
| Never | A Route Handler that calls your own Server Action | Never. The two are one code path with a network hop between them. | A second hop, a second serialization, and a form that now needs JavaScript. |

### The data access layer

```ts
// src/lib/dal/orders.ts
import "server-only";
import { cookies } from "next/headers";
import { z } from "zod";

// This response carries money, so the value is proven at the boundary
// rather than taken on trust.
const OrderPage = z.object({
  count: z.number(),
  next: z.string().nullable(),
  previous: z.string().nullable(),
  results: z.array(z.object({ id: z.string(), total: z.string() })),
});

export async function getOrders(page: number): Promise<z.infer<typeof OrderPage>> {
  const session = (await cookies()).get("sessionid")?.value;
  const response = await fetch(
    `${process.env.DJANGO_URL}/api/orders/?page=${page}`,
    {
      headers: session ? { Cookie: `sessionid=${session}` } : {},
      cache: "no-store", // per-user data: state the intent, never inherit it
    },
  );
  if (!response.ok) {
    throw new Error(`orders: ${response.status}`);
  }
  const parsed = OrderPage.safeParse(await response.json());
  if (!parsed.success) {
    throw new Error(`orders: ${z.prettifyError(parsed.error)}`);
  }
  return parsed.data;
}
```

Four rules hold for every module in the layer. Import `server-only`, so the
build fails when a client module imports it. Take the shape from the generated
schema, never from a hand-written interface, and never cast `response.json()`
to it. State the cache intent on every `fetch`, because the default depends on
the caching model. Throw on a non-`ok` response, so the segment `error.tsx`
renders instead of a `.map` on `undefined`.
`references/boundary-validation-and-api-types.md` rules on which boundary the
generated type covers and which one needs a schema as well.

This file owns where the module sits. It does not own the client that the
module calls. The typed client, the two base URLs, the timeout, the retry rule,
and the error normalizer are
`references/api-client-and-request-safety.md`.

### The DRF seam

The path is fixed. DRF views produce an OpenAPI 3.0.3 document through
drf-spectacular. `openapi-typescript` turns that document into TypeScript, and
the data access layer imports the result.
`references/openapi-schema-and-codegen.md` owns the generation config. The
sibling skill `django-api-contract` owns the server side of the contract.

Four kinds of drift break the frontend. Each one is silent until run time.

| Drift on the server | What the frontend sees |
| --- | --- |
| A serializer field is renamed and the types are not regenerated | `undefined` where TypeScript promised a value |
| The pagination envelope is not modeled | `.map` runs on an object, because the response is `{count, next, previous, results}` |
| The error envelope changes shape | `error.tsx` shows a generic message, and the action returns an unparsed error |
| A cookie `SameSite` or domain attribute changes | The browser drops the cookie on a direct call, and a 401 looks like a frontend bug |

Regenerate the types in CI, and fail the build on a diff. The types are an
artifact, never an edit.

### Proxy Django through Next, or call it from the browser

| Condition | Action | Reason | It reverses when | The cost |
| --- | --- | --- | --- | --- |
| The browser must send an httpOnly session cookie to Django | Proxy through a Route Handler or a rewrite | The request stays same-origin, so no CORS preflight and no dropped cookie | The auth strategy moves to a token in an `Authorization` header. | Every request pays a hop through the Node process, and that process carries the traffic of the API. |
| A secret or an API key must go on the request | Proxy on the server | The secret never reaches the browser bundle | Never, while the request needs the secret. | The same hop, plus a route to maintain for each endpoint that the browser reaches. |
| A public GET, no secret, latency-sensitive, CORS already configured | Call Django from the client | It removes a network hop | The endpoint starts to need a cookie or a secret, which the two rows above cover. | The internal address is public, and the CORS rules become a permanent maintenance item. |
| The response streams | Either. Measure. | Both support a stream; the hop adds latency | The measurement states one of the two, and the record of it decides later calls. | The measurement itself, once for each stream endpoint. |
| Data for the first render | Fetch in the data access layer, server to server | No browser round trip, no CORS, and the secret stays safe | The value must change after the mount, which the client cache then owns. | The first byte waits for the backend, so a slow endpoint holds the whole render. |

### A Server Action

```ts
// app/posts/actions.ts
"use server";

import { z } from "zod";
import { updateTag } from "next/cache";
import { redirect } from "next/navigation";
import { getSession } from "@/lib/dal/session";
import { createPost } from "@/lib/dal/posts";

const Schema = z.object({ title: z.string().min(1) });

export type State = { error?: string };

export async function createPostAction(
  _prev: State,
  formData: FormData,
): Promise<State> {
  const session = await getSession();          // 1. authorize inside the action
  if (!session) return { error: "unauthorized" };

  const parsed = Schema.safeParse({ title: formData.get("title") });
  if (!parsed.success) return { error: "The title is required." };

  const post = await createPost(session, parsed.data); // 2. mutate
  updateTag(`user-${session.userId}-posts`);           // 3. invalidate
  redirect(`/posts/${post.id}`);                       // 4. redirect, last
}
```

The order is fixed: authorize, validate, mutate, invalidate, redirect.

```ts
// Wrong: the redirect sits inside a try/catch.
// Failure: redirect() signals by throwing. The catch swallows the signal,
// the user stays on the form, and the action reports a false error.
try {
  const post = await createPost(session, parsed.data);
  redirect(`/posts/${post.id}`);
} catch (error) {
  return { error: "Could not create the post." };
}
```

```ts
// Correct: the redirect runs after the try block, outside every catch.
let post;
try {
  post = await createPost(session, parsed.data);
} catch {
  return { error: "Could not create the post." };
}
updateTag(`user-${session.userId}-posts`);
redirect(`/posts/${post.id}`);
```

A Server Action is a public HTTP endpoint. Verify the session inside it every
time. NEVER treat a `proxy.ts` redirect, or a hidden form field, as the
authorization.

### A Route Handler

```ts
// Wrong: the Route Handler calls your own Server Action.
// Failure: an extra network hop, a second serialization of the same data,
// and the form loses its progressive enhancement, because it now needs
// JavaScript to reach the endpoint.
// app/api/posts/route.ts
import { createPostAction } from "@/app/posts/actions";

export async function POST(request: Request) {
  const formData = await request.formData();
  return Response.json(await createPostAction({}, formData));
}
```

```tsx
// Correct: the form calls the Server Action directly, through the dispatcher
// that useActionState returns.
// app/posts/new/new-post-form.tsx
"use client"; // reason: useActionState holds the state that the Action returns
import { useActionState } from "react";
import { createPostAction } from "../actions";

export function NewPostForm() {
  const [state, formAction, isPending] = useActionState(createPostAction, {});
  return (
    <form action={formAction}>
      <label htmlFor="title">Title</label>
      <input id="title" name="title" required />
      {state.error && <p role="alert">{state.error}</p>}
      <button type="submit" disabled={isPending}>
        Create
      </button>
    </form>
  );
}
```

The action takes the previous state as its first parameter, so `<form action>`
receives the dispatcher and never the action itself. The pending state, the
errors that the Action returns, and the optimistic value are
`references/suspense-and-actions.md`.

Keep a Route Handler for the cases that a Server Action cannot serve. The cases
are an external consumer, a webhook from a payment provider, a file upload, and
a stream.

### The prefetch and hydration seam

```tsx
// Correct: the server prefetches, and the client cache starts warm.
// app/orders/page.tsx
import { dehydrate, HydrationBoundary } from "@tanstack/react-query";
import { getQueryClient } from "@/lib/query/client";
import { ordersListOptions } from "@/features/orders/api/orders";
import { OrdersTable } from "./orders-table";

export default async function Page() {
  const queryClient = getQueryClient(); // a new client for each server request
  await queryClient.prefetchQuery(ordersListOptions({ search: "", page: 1 }));
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <OrdersTable />
    </HydrationBoundary>
  );
}
```

Await the critical prefetch, and wrap the client subtree in the
`HydrationBoundary`. A client component that still refetches on mount has one
of the two missing. The server and the client call the same `queryOptions`
function, so both use one key. The `staleTime`, the key factory, and the
mutation state are `references/server-state-and-query-cache.md`.

## Verification

```bash
# 1. Every data access module is server-only. Each file printed here is not.
rg --files-without-match 'server-only' src/lib/dal

# 2. Find a Route Handler that imports a Server Action. This must print
#    nothing.
rg -n "from ['\"].*action" -g '**/route.ts' .

# 3. Find a redirect inside a catch block. Read each hit.
rg -n -B4 'redirect\(' -g '**/actions.ts' .

# 4. Confirm that every Server Action verifies the session.
rg -l '"use server"' -g '*.ts' . | xargs rg --files-without-match 'getSession|auth\('

# 5. Regenerate the typed client, then typecheck. An error is drift. The
#    generated folder is gitignored, so a diff on it proves nothing.
pnpm api:generate && pnpm typecheck

# 6. Confirm that no secret reaches the browser bundle.
rg -n 'NEXT_PUBLIC_[A-Z_]*(KEY|SECRET|TOKEN|PASSWORD)' .
```

## Review checklist

- [ ] Does each resource have exactly one source per view?
- [ ] Does every backend call go through the data access layer?
- [ ] Does every data access module import `server-only`?
- [ ] Do the response types come from the generated schema rather than a
      hand-written interface?
- [ ] Does the code model the DRF pagination envelope, not a bare array?
- [ ] Does every `fetch` state its cache intent?
- [ ] Does every non-`ok` response throw or return a mapped error?
- [ ] Does every Server Action verify the session inside the action?
- [ ] Does every Server Action validate the input with a schema?
- [ ] Does every `redirect()` run last, outside every try and catch?
- [ ] Is every Route Handler justified by an external consumer, a webhook, an
      upload, or a stream?
- [ ] Does no Route Handler call a Server Action?
- [ ] Does every prefetched query sit inside a `HydrationBoundary`?

## Handoffs

- The types of the generated client, the generics over a paginated response,
  and the parse at the boundary → domain 02
  `typescript-type-system-discipline`, in
  `references/boundary-validation-and-api-types.md`.
- The drf-spectacular config, the generator, and the schema artifact →
  `references/openapi-schema-and-codegen.md`. The server side belongs to the
  sibling skill `django-api-contract`.
- The typed client that this layer calls, the base URLs, the retry rule, and
  the error normalizer → `references/api-client-and-request-safety.md`.
- The CSRF token, the CORS symptoms, and the proxy Route Handler that hides the
  internal address → `references/cross-origin-and-bff-proxy.md`.
- The client cache config, the query keys, the mutations, and the optimistic
  state → `references/server-state-and-query-cache.md`. The filter that the URL
  holds is `references/client-and-url-state.md`.
- The session strategy, the token storage, and the role checks → domain 07
  `authentication-and-authorization`. Not integrated yet. The DRF permission
  classes belong to the sibling skill `secure-code-auditor`.
- The pending state of a form, the error that the Action returns, and the
  optimistic value → `references/suspense-and-actions.md`.
- The form binding, the field-level error mapping, and the multi-step flow →
  domain 11 `forms-and-validation`. Not integrated yet.
- The N+1 query and the endpoint latency behind a slow call → the sibling
  skill `django-performance-optimizer`. This file owns only the number of
  frontend requests and the place where each one starts.
- MSW handlers and the contract test against the schema → domain 20
  `testing-and-quality`. Not integrated yet.
