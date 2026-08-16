# Caching and revalidation

Next.js 16.3 baseline. This file owns how long a response lives and what makes
it fresh again: the caching model that the project runs, the `"use cache"`
directive, the four revalidation APIs, and the render mode that the build
reports for each route. The route tree is
`references/app-router-structure.md`. The server and client split is
`references/server-and-client-components.md`. The place where the data enters
is `references/data-access-and-mutations.md`.

## Principle

A cache without a declared invalidation trigger is a stale-data generator.
Decide the trigger before you write the cache.

A cache is keyed by what the code passes to it, not by who asked. A response
that depends on the user therefore never enters a shared cache.

A page is rarely all static or all dynamic. The correct unit is a static shell
with a dynamic hole inside it.

An invalidation on the server does not reach a cache in the browser. Name the
cache you mean.

## Pinned-stack depth

### First, read the model

The default behavior of `fetch` depends on one flag. Read `next.config.ts`
before you write any cache code.

| The project has | The model | What it means |
| --- | --- | --- |
| `cacheComponents: true` | Cache Components | Everything is dynamic. Nothing caches until you add `"use cache"`, a `cacheLife`, and a `cacheTag`. Partial Prerendering is automatic. The segment `revalidate` and `dynamic` exports and `experimental.ppr` are superseded. |
| No `cacheComponents` key | The previous model | Use the `fetch` options `cache`, `next.revalidate`, and `next.tags`, plus `unstable_cache` and the route segment config. |

NEVER depend on a default. State the intent on every `fetch`, so the code
means the same thing under either model.

### `"use cache"` declares a lifetime and a tag

```ts
// Correct: a cache scope with both declarations.
import { cacheLife, cacheTag } from "next/cache";
import type { components } from "@/lib/api/schema";

export async function getPublishedProducts() {
  "use cache";
  cacheLife("max");
  cacheTag("products");
  const response = await fetch(`${process.env.DJANGO_URL}/api/products/`);
  if (!response.ok) throw new Error(`products: ${response.status}`);
  return (await response.json()) as components["schemas"]["PaginatedProductList"];
}
```

Every `"use cache"` scope declares a `cacheLife` and a `cacheTag`. Without the
tag, nothing can invalidate the entry on purpose. Take the profile name from
the `cacheLife` profiles of the installed version.

### A cache that holds per-user data is a leak

```tsx
// Wrong: the cache scope reads the session.
// Failure: one user's invoices are served to the next user. Request data is
// also forbidden inside a cache scope, so this fails the build under
// cacheComponents. Where it does build, the leak is silent until a customer
// reports it.
async function getInvoices() {
  "use cache";
  cacheTag("invoices");
  const token = (await cookies()).get("session")?.value;
  return fetch(`${process.env.DJANGO_URL}/api/invoices/`, {
    headers: { Authorization: `Bearer ${token}` },
  });
}
```

```tsx
// Correct: a static shell, and a dynamic hole for the per-user part.
export default function Page() {
  return (
    <>
      <StaticHeader />
      <Suspense fallback={<InvoicesSkeleton />}>
        <Invoices />
      </Suspense>
    </>
  );
}

async function Invoices() {
  const token = (await cookies()).get("session")?.value; // dynamic, uncached
  const data = await fetchInvoices(token);
  return <InvoiceList items={data} />;
}
```

NEVER read `cookies()`, `headers()`, or `searchParams` inside a `"use cache"`
scope. Read them outside it, and put the result behind `<Suspense>`.

### Choose the revalidation API

| Condition | Action |
| --- | --- |
| Static content, and eventual consistency is acceptable | `revalidateTag(tag, 'max')`. It serves the stale value and revalidates in the background. |
| The user just changed their own data and must see it now | `updateTag(tag)`. Server Actions only. It gives read-your-writes. |
| Uncached data elsewhere on the page must refresh after an action | `refresh()`. Server Actions only. It does not touch the cache. |
| Invalidation by path | `revalidatePath('/path')` |
| A Client Component must reflect a server mutation | `router.refresh()`. The last resort. |

```ts
// Wrong: the single-argument call.
// Failure: the form is deprecated in Next 16. The intended cache profile is
// never applied, so the entry lives for a period nobody declared.
revalidateTag("products");
```

```ts
// Correct: the tag and the cacheLife profile.
revalidateTag("products", "max");
```

`router.refresh()` is not a substitute for an invalidation. It refetches
everything on the route, it hides the stale-cache bug that caused it, and it
adds load on every call. Reach for it only when no tag covers the change.

### The browser keeps its own cache

`revalidateTag` invalidates the server cache. The Router Cache in the browser
holds the client-side result of the navigation, and the tag does not clear it.
A user who sees stale data after a mutation, while a fresh reload is correct,
is looking at the Router Cache. Call `updateTag()` in the Server Action for
read-your-writes, or call `router.refresh()` on the client.

### A dynamic route without a request API

```ts
// Wrong: the non-deterministic value runs at build time.
// Failure: the route prerenders once, so every visitor receives the same
// "random" pick and the same timestamp until the next deploy.
export default async function Page() {
  const pick = Math.floor(Math.random() * items.length);
  return <Item item={items[pick]} />;
}
```

```ts
// Correct: connection() makes the route wait for a real request.
import { connection } from "next/server";

export default async function Page() {
  await connection();
  const pick = Math.floor(Math.random() * items.length);
  return <Item item={items[pick]} />;
}
```

`connection()` replaces `unstable_noStore`. Under the previous model, a route
that must be fresh but tolerates a short cache uses `revalidate = 1`, not `0`.
A `revalidate` of `0` reclassifies the route as dynamic.

### `unstable_cache`

`unstable_cache` is audit-only. Migrate it to a `"use cache"` function: drop
the key-parts array, because the key is derived, and map the options to
`cacheLife` and `cacheTag`. One reason to keep it exists. `unstable_cache`
persists across deployments and across serverless instances, and `"use cache"`
does not. Keep `unstable_cache` where that persistence is a requirement, and
record the reason in the code.

### Read the build report

`next build` prints one symbol per route and a legend under the table. Read
the legend that your installed version prints, and compare every route against
its declared render mode. Two results are always findings. A route that reads
`cookies()`, `headers()`, or `searchParams` must never appear as static. A
route that the team declared static must not appear as dynamic.

A route that is unexpectedly dynamic has one of three causes: a stray
`runtime` export, a `searchParams` read at the page level, or an uncached data
call. Run `next build --debug` for the detail.

## Verification

```bash
# 1. Read the model before anything else.
rg -n 'cacheComponents' next.config.ts

# 2. List every cache scope, then confirm each has a tag and a lifetime.
rg -n '"use cache"' -g '*.ts' -g '*.tsx' .
rg -n 'cacheTag\(|cacheLife\(' -g '*.ts' -g '*.tsx' .

# 3. Find the deprecated single-argument call. This must print nothing.
rg -n 'revalidateTag\([^,)]*\)' -g '*.ts' -g '*.tsx' .

# 4. Find request data inside a cache scope. Read every hit.
rg -n -A12 '"use cache"' -g '*.ts' -g '*.tsx' . | rg 'cookies\(|headers\(|searchParams'

# 5. Build, and compare each route symbol against its declared render mode.
pnpm build
pnpm build --debug

# 6. Confirm the read-your-writes path after a mutation: submit the form,
#    then read the value back without a manual reload.
```

## Review checklist

- [ ] Is the caching model of the project established from `next.config.ts`
      before any cache code is written?
- [ ] Does every `fetch` state its cache intent rather than inherit a default?
- [ ] Does every `"use cache"` scope declare both a `cacheLife` and a
      `cacheTag`?
- [ ] Is `cookies()`, `headers()`, and `searchParams` absent from every
      `"use cache"` scope?
- [ ] Is every response that depends on the user outside every shared cache?
- [ ] Is every authenticated route reported as dynamic by the build, never as
      static?
- [ ] Does every cache entry have a named invalidation trigger?
- [ ] Is `revalidateTag` always called with the tag and the profile?
- [ ] Does a mutation that the user must see immediately use `updateTag()`?
- [ ] Is `router.refresh()` absent, unless no tag can cover the change?
- [ ] Does every remaining `unstable_cache` record why it stays?
- [ ] Does `connection()` precede every non-deterministic value on a route
      that must run per request?

## Handoffs

- The `staleTime`, the garbage collection time, and the client query cache →
  domain 06 `data-fetching-and-state`. Not integrated yet. This file stops at
  the server cache and the Router Cache.
- Whether a cached response may hold the data at all, and what a leaked
  response exposes → domain 17 `frontend-security`. Not integrated yet. That
  domain holds a veto.
- The database query cost behind a cache miss, and the server-side cache in
  Django → the sibling skill `django-performance-optimizer`.
- The `Cache-Control` header, the CDN, and the edge → domain 22
  `build-deploy-and-runtime-ops`. Not integrated yet.
- The LCP and the INP effect of a cache decision → domain 16
  `performance-and-web-vitals`. Not integrated yet.
