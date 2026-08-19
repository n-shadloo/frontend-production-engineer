# Server state and the query cache

TanStack Query 5.101 or later, React 19.2.6 or later, Next.js 16.3, against a
Django and DRF backend. This file owns the cache that holds server state, and
every read and every write that passes through it. The subjects are
`queryOptions`, the key factory, the cache times, and the lifetime of a
`QueryClient`. They also include the mutation and its reconciliation, the
optimistic update, the infinite query over DRF pagination, and the four states
of a data view.

Where a value lives when the backend does not own it is
`references/client-and-url-state.md`. The module that calls the backend is
`references/data-access-and-mutations.md`. The one typed client and the
`ApiError` that every failure becomes are
`references/api-client-and-request-safety.md`.

## Principle

Server state has an owner outside the program. It goes stale on its own
schedule, so it needs a cache with a declared lifetime and not a variable.

One query has one definition. A second definition is a second cache entry that
holds the same data under a different key, and only one of the two revalidates.

A key is the identity of a request. A key that omits an input gives one answer
to two questions, and the answers overwrite each other.

A write to the backend makes something on the screen wrong. The write states
what, or the screen stays behind the database.

An optimistic value is a claim about a result that has not arrived. Every claim
needs a way back.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### One `queryOptions` object for one query

```tsx
// Wrong: the component holds the definition, and the key omits the filters.
// Failure: every filter shares the one entry ['orders'], so the results of two
// filters overwrite each other and the list flickers. The definition is also
// unreachable from the mutation that must invalidate it, and from the server
// that must prefetch it.
const { data } = useQuery({
  queryKey: ["orders"],
  queryFn: () => api.GET("/api/orders/", { params: { query: filters } }),
});
```

```ts
// Correct: src/features/orders/api/orders.ts holds the key, the fetcher, and
// the options object. The server, the component, and the mutation all import
// this one module.
import { queryOptions } from "@tanstack/react-query";
import { api } from "@/lib/api/client";
import { normalizeApiError } from "@/lib/api/errors";

export type OrderFilters = {
  search: string;
  page: number;
};

export const orderKeys = {
  all: ["orders"] as const,
  lists: () => [...orderKeys.all, "list"] as const,
  list: (filters: OrderFilters) => [...orderKeys.lists(), filters] as const,
  details: () => [...orderKeys.all, "detail"] as const,
  detail: (id: OrderId) => [...orderKeys.details(), id] as const,
};

async function listOrders(filters: OrderFilters, signal: AbortSignal) {
  const { data, error, response } = await api.GET("/api/orders/", {
    params: { query: filters },
    signal: AbortSignal.any([signal, AbortSignal.timeout(10_000)]),
  });
  if (error) throw normalizeApiError(response.status, error);
  return data;
}

export function ordersListOptions(filters: OrderFilters) {
  return queryOptions({
    queryKey: orderKeys.list(filters),
    queryFn: ({ signal }) => listOrders(filters, signal),
    staleTime: 30_000,
  });
}
```

Four rules bind every module of this kind. Put it at
`src/features/<feature>/api/`, which
`references/directory-and-module-boundaries.md` owns. NEVER import
`server-only` here. The browser imports this module too. Take the `signal` that
Query supplies, and combine it with a timeout. Throw on a failure, so the error
reaches the `error` state of the hook.

That `server-only` guard belongs to a data access module, and
`references/data-access-and-mutations.md` states which modules carry it.

### The key factory

The key is hierarchical, and every level is serializable. A key that starts
with `["orders"]` matches a prefix invalidation of `orderKeys.all`, and a key
under `orderKeys.lists()` matches a prefix invalidation of the lists alone.

| The key must encode | Reason |
| --- | --- |
| Every filter, sort, page, and search term | Each set of inputs is a different answer. |
| The id of the record | A detail of one record is not a detail of another. |
| The locale, where the backend translates the response | One cache entry per language. |
| The user, only where one browser session reads two identities | An identity change otherwise serves the previous user's rows. |

NEVER put a value that changes on every render into a key. A new object
literal, a `Date`, and a computed value each give a new key on each render.
Each new key starts a new request.

### The cache times

| Option | What it decides | The v5 default |
| --- | --- | --- |
| `staleTime` | How long the data counts as fresh. Query sends no request for fresh data. | `0`, so the data is stale at once |
| `gcTime` | How long an inactive entry stays in memory before Query removes it. | 5 minutes |
| `retry` | How many times a failed `queryFn` repeats. | 3, with an exponential backoff |
| `refetchOnMount`, `refetchOnWindowFocus`, `refetchOnReconnect` | Whether a stale entry refetches on each event. | `true` |

Set `staleTime` on every `queryOptions` object from how fast the resource
changes on the server. A reference list that changes each week takes minutes. A
price that changes each second takes zero. A default of `0` on a list that a
navigation shows again produces a request on every visit.

### One `QueryClient` for each request, and one for the browser

```tsx
// Wrong: the client sits in module scope.
// Failure: the server module loads once, so every request shares one cache.
// One user receives the rows of another user. The defect appears in
// production only, and it is intermittent, because it needs two requests
// close together.
const queryClient = new QueryClient();

export default function Providers({ children }: { children: React.ReactNode }) {
  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>;
}
```

```ts
// Correct: a new client for each request on the server, and one client in the
// browser.
// src/lib/query/client.ts — no directive, so the page and the provider both
// import this one module.
import { QueryClient, isServer } from "@tanstack/react-query";
import { isApiError } from "@/lib/api/errors";

export function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        // Above zero, so the browser does not refetch what the server
        // prefetched, immediately after the hydration.
        staleTime: 60_000,
        // ApiError.retryable is false for every 4xx except 429, so a
        // deterministic failure never repeats.
        retry: (failureCount, error) =>
          isApiError(error) && error.retryable && failureCount < 3,
      },
    },
  });
}

let browserQueryClient: QueryClient | undefined;

export function getQueryClient() {
  if (isServer) return makeQueryClient();
  browserQueryClient ??= makeQueryClient();
  return browserQueryClient;
}
```

```tsx
// src/app/providers.tsx
"use client"; // reason: QueryClientProvider holds client state
import { QueryClientProvider } from "@tanstack/react-query";
import { getQueryClient } from "@/lib/query/client";

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={getQueryClient()}>{children}</QueryClientProvider>
  );
}
```

```ts
// src/lib/api/errors.ts — the guard sits beside normalizeApiError, because
// that function returns a plain object and not an Error instance.
export function isApiError(error: unknown): error is ApiError {
  return typeof error === "object" && error !== null && "retryable" in error;
}
```

`references/api-client-and-request-safety.md` owns the `ApiError` shape, and it
states which statuses are retryable. The default `retry` of Query is 3 for
every failure. Without the rule above, a 400 and a 403 each repeat three times,
and the user waits through two backoff periods for a deterministic answer.

A hook that runs outside this provider throws "No QueryClient set, use
QueryClientProvider to set one". The provider file needs `"use client"`, and
`references/server-and-client-components.md` states why a provider goes in its
own small wrapper rather than on the layout.

### The prefetch and the client share one key

`references/data-access-and-mutations.md` owns the page that prefetches and the
`HydrationBoundary` around the client subtree. Two rules make that seam work,
and both belong here.

The server and the client call the same `queryOptions` function, so the key is
the same value. A client that builds its key by hand gets a second entry, and
it fetches again on the mount.

The server `QueryClient` needs `staleTime` above zero. With the default of `0`
the hydrated data is stale on arrival, `refetchOnMount` fires, and the browser
repeats the request that the server already made.

NEVER put a prefetch inside a `"use cache"` scope. A cache scope reads no
request data, and a query for the signed-in user needs it.
`references/caching-and-revalidation.md` owns that rule.

### A data view renders four states

```tsx
// Correct: loading, error, empty, and ready. The empty state is designed.
"use client"; // reason: useQuery reads the client cache
import { keepPreviousData, useQuery } from "@tanstack/react-query";
import { ordersListOptions, type OrderFilters } from "@/features/orders/api/orders";

export function OrdersList({ filters }: { filters: OrderFilters }) {
  const { data, isPending, isError, isPlaceholderData } = useQuery({
    ...ordersListOptions(filters),
    placeholderData: keepPreviousData,
  });

  if (isPending) return <OrdersSkeleton rows={10} />;
  if (isError) return <OrdersError />;
  if (data.results.length === 0) return <OrdersEmpty search={filters.search} />;

  return (
    <ul aria-busy={isPlaceholderData}>
      {data.results.map((order) => (
        <li key={order.id}>{order.reference}</li>
      ))}
    </ul>
  );
}
```

| Flag | It is true when |
| --- | --- |
| `isPending` | The entry holds no data yet |
| `isFetching` | A request is in flight, whether or not data is present |
| `isLoading` | `isPending` and `isFetching` together, which is the first load |
| `isPlaceholderData` | The rendered value is a stand-in, and the real value is in flight |

Read `isPending` for the first paint, and `isFetching` for a quiet indicator
over data that is already on the screen. A view that reads `isFetching` for its
skeleton replaces the whole list on every background refetch.

`data.results` is the DRF pagination envelope.
`references/boundary-validation-and-api-types.md` owns the `Paginated<T>` type
and the parse that proves it.

A read through `useSuspenseQuery` or `useSuspenseInfiniteQuery` has no
`isPending` branch, so it needs a `<Suspense>` boundary above it and a reachable
error boundary. `references/suspense-and-actions.md` owns where that boundary
goes and what shape the fallback takes.

### `initialData` or `placeholderData`

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| Real data is available to seed the entry, such as a record from the list that the detail page opens | `initialData`. Query writes it to the cache, and `staleTime` applies to it. | The seed is partial, so it does not hold every field that the view reads. | The seed enters the cache, so `staleTime` can hold a partial record on the screen. |
| A stand-in is needed while the request runs, such as the previous page of a table | `placeholderData`. Query does not write it to the cache, and `isPlaceholderData` is true. | The stand-in is real data that the cache should keep, which the row above covers. | The view must read `isPlaceholderData` and mark the stand-in, or the user acts on it. |
| A page change must not empty the table | `placeholderData: keepPreviousData`, imported from the package. | The two pages hold unrelated rows, so the previous page misleads the reader. | The table shows the previous page while the next one loads. |

In v5 `keepPreviousData` is a function that `placeholderData` takes. The v4
boolean option of the same name no longer exists.

### Every mutation states what it makes stale

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| The response carries the whole new record | `setQueryData` on the detail key. No request. | The server computes a field that the response omits. | The cache holds a record that no request proved, until the entry goes stale. |
| The write changes a list whose order or totals the server computes | `invalidateQueries` on the list prefix. | The response already carries the computed values, which the row above covers. | Every active query under that prefix sends a request. |
| Only the queries that are on the screen must refetch now | `refetchQueries`. | An inactive entry must also be correct on the next mount. | The inactive entries keep the old value, and one of them can reach the screen first. |
| Nothing on the screen shows the changed data | State that in a comment. Add no reconciliation. | A view starts to read the changed data. | The comment is the only record, so a later reader must trust it. |

```ts
// Wrong: the mutation invalidates the whole cache.
// Failure: every entry in the application refetches at once. A screen with
// twelve queries sends twelve requests for one write, and the backend
// throttle rejects some of them.
onSuccess: () => queryClient.invalidateQueries();
```

```ts
// Correct: the prefix names the subtree that the write changed.
onSuccess: () => queryClient.invalidateQueries({ queryKey: orderKeys.lists() });
```

A mutation with neither an invalidation nor a cache write is incomplete. The
user saves a record, the list keeps the old value, and the defect looks like a
backend fault.

A Server Action that mutates on the server uses `updateTag` instead, and
`references/caching-and-revalidation.md` owns that call. Use the Query cache for
a write that a client component sends. Use a Server Action for a write that a
`<form action>` sends.

### The optimistic update needs a rollback

```ts
// Wrong: the cache changes, and nothing puts it back.
// Failure: the request fails, and the row keeps the value that the user
// asked for. The screen now disagrees with the database, and only a reload
// corrects it.
useMutation({
  mutationFn: setOrderPaid,
  onMutate: (order) => {
    queryClient.setQueryData(orderKeys.detail(order.id), { ...order, paid: true });
  },
});
```

```ts
// Correct: cancel, snapshot, write, restore on a failure, and reconcile at the
// end.
export function useSetOrderPaid() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: setOrderPaid,
    onMutate: async (order: Order) => {
      await queryClient.cancelQueries({ queryKey: orderKeys.detail(order.id) });
      const previous = queryClient.getQueryData<Order>(orderKeys.detail(order.id));
      queryClient.setQueryData<Order>(orderKeys.detail(order.id), {
        ...order,
        paid: true,
      });
      return { previous };
    },
    onError: (_error, order, context) => {
      if (context?.previous) {
        queryClient.setQueryData(orderKeys.detail(order.id), context.previous);
      }
    },
    onSettled: (_data, _error, order) =>
      queryClient.invalidateQueries({ queryKey: orderKeys.detail(order.id) }),
  });
}
```

The four steps are fixed. `cancelQueries` stops a request in flight from writing
an old value over the optimistic one. The snapshot is the value to restore.
`onError` restores it. `onSettled` invalidates, so the server decides the final
value.

Give the user an optimistic value only where the failure is rare and the
correction is visible. A write that fails often shows the user two answers.

The `useQueryClient()` closure above is the form to write. Recent v5 releases
also pass a context object with `context.client` to the callbacks. Read the
signature of the installed version before you use that form. NEVER mix the two
idioms in one hook.

### `useOptimistic` or the query cache

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| A `<form action>` submits to a Server Action, and one component shows the result | `useOptimistic`. Its value is local to the component and to the transition. | A second view reads the value, which the next row covers. | The value is lost on an unmount, and no other view sees it. |
| Two or more views read the value from the client cache | The Query cache, through `onMutate` and `setQueryData`. | One view reads the value, and the write goes through a Server Action. | The hook must cancel, snapshot, restore, and reconcile, which is four steps rather than one. |
| The value must survive the unmount of the component that changed it | The Query cache. | Never. `useOptimistic` holds no value past the transition. | The same four steps, and a cache entry that a failure must put back. |

`references/suspense-and-actions.md` owns `useOptimistic` and the transition
that it needs. The rule that divides them is the owner of the value, and not the
size of the change.

### The infinite query over DRF pagination

DRF sends the address of the next page. Read the next page param out of that
address, and never compute it. `references/api-client-and-request-safety.md`
states the same rule for a plain request, and it names the failure that a
computed offset produces.

| The DRF class | The envelope | The param to read from `next` |
| --- | --- | --- |
| `PageNumberPagination` | `{count, next, previous, results}` | `page` |
| `LimitOffsetPagination` | `{count, next, previous, results}` | `offset` |
| `CursorPagination` | `{next, previous, results}`, with no `count` | `cursor`, which is opaque |

```ts
// Correct: one rule serves all three classes.
import { infiniteQueryOptions } from "@tanstack/react-query";

function paramFrom(next: string | null, name: string): string | undefined {
  if (next === null) return undefined;
  return new URL(next).searchParams.get(name) ?? undefined;
}

export function ordersInfiniteOptions(search: string) {
  return infiniteQueryOptions({
    queryKey: [...orderKeys.all, "infinite", { search }] as const,
    queryFn: ({ pageParam, signal }) =>
      listOrders({ search, page: Number(pageParam) }, signal),
    initialPageParam: "1",
    getPreviousPageParam: (firstPage) => paramFrom(firstPage.previous, "page"),
    getNextPageParam: (lastPage) => paramFrom(lastPage.next, "page"),
  });
}
```

The property order matters for the type inference. Write `queryFn`, then
`getPreviousPageParam`, then `getNextPageParam`. The
`@tanstack/eslint-plugin-query` rule `infinite-query-property-order` reports the
other orders. Every other property is order-insensitive.

`initialPageParam` and `getNextPageParam` are both required in v5. A page param
of `undefined` means that no further page exists, and it sets `hasNextPage` to
false. `CursorPagination` sends no `count`, so give that endpoint no control
that jumps to a numbered page.

The column model, the row model, and the scroll container of a table are
`references/data-table-and-server-driven-state.md`.

### The failure reaches the error channel

The `queryFn` throws, so `error` holds an `ApiError` and `isError` is true.
A `queryFn` that returns a failed response puts the failure in `data`. The view
then renders the error body as if it were a record.

| The status | What the view does | It reverses when | The cost |
| --- | --- | --- | --- |
| 400 with field errors | Render each message beside its field. Do not retry. | Never. The answer is deterministic, so a second request returns it again. | The view needs a map from a field name to a control. |
| 401 or 403 | Render the state that the route needs. `references/session-and-token-lifecycle.md` owns the refresh, and `references/route-protection-and-permissions.md` owns the redirect. | The application can refresh the session, so the request repeats once after it. | Each route states its own answer, so two routes can answer one status in two ways. |
| 404 | Render the empty or missing state of that view. | The 404 means a deleted record that the cache still lists, so the list also needs the write. | The view needs a designed empty state beside its error state. |
| 429, 502, 503, and 504 | Retry under the rule above, and obey `Retry-After`. | The endpoint is not idempotent, so a repeat changes data. | The user waits for the backoff periods, and the backend takes more load. |
| A `TypeError` or a `DOMException` | Let it reach the error boundary. Neither carries a status. | The abort was deliberate, so the view discards it and renders nothing. | The whole boundary renders its fallback, not one part of the view. |

`references/api-client-and-request-safety.md` owns the shapes and the
normalizer. `references/suspense-and-actions.md` owns the rule that a validation
error returns as state and never throws. The map from a field name to a form
control is `references/form-submission-and-server-errors.md`.

### Polling needs a reason and a stop condition

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| A push transport is available | Use it, and write the message into the cache with `setQueryData`. `references/push-transport-and-connection.md` owns the transport. | The transport drops often enough that the view falls behind the server. | A connection for each client, and code that reconciles a missed message. |
| A job settles, such as an export or a build | `refetchInterval` as a function that returns `false` when the job is done. | The job never reports a terminal state, so the function has no stop condition. | One request for each period, for each client that watches the job. |
| The resource changes on the server, and no push transport exists | `refetchInterval`, with a comment that states the period and the reason. | A push transport lands, which the first row covers. | One request for each period for the life of the tab, and the battery that it costs. |
| The tab is in the background | Keep the default `refetchIntervalInBackground` of `false`. Query then stops the poll. | The value must stay fresh while the tab is hidden, such as a live alert. | The data is stale when the user returns, so the first paint shows the old value. |

```ts
// Correct: the poll stops itself.
refetchInterval: (query) => (query.state.data?.status === "done" ? false : 5_000),
```

An interval with no stop condition runs for the life of the tab. It costs the
battery of the device and a request for each user for each period.

### The lint gate

Add `@tanstack/eslint-plugin-query` to the flat config array.
`references/lint-format-and-scripts.md` owns that array and the position of
each block.

| The rule | What it reports |
| --- | --- |
| `exhaustive-deps` | A `queryFn` that reads a value the key does not encode |
| `stable-query-client` | A `QueryClient` that a render creates again on each pass |
| `no-rest-destructuring` | A rest destructure of the hook result, which subscribes to every field |
| `infinite-query-property-order` | The property order that breaks the type inference |
| `prefer-query-options` | A hook call that takes an inline object rather than an options object |

### Version discipline

Read the installed version before you write code. TanStack Query 5.101 is the
floor of this stack. Version 5 needs React 18 or later and TypeScript 4.7 or
later, and it takes one object argument for every hook.

Every idiom in the left column below is alive only in legacy code. Rewrite each
v4 idiom that a generator or an older file produces.

| Version 4 | Version 5 |
| --- | --- |
| `cacheTime` | `gcTime` |
| `useErrorBoundary` | `throwOnError` |
| `keepPreviousData: true` | `placeholderData: keepPreviousData` |
| `status: 'loading'`, `isInitialLoading` | `status: 'pending'`, `isLoading` |
| `onSuccess`, `onError`, `onSettled` on `useQuery` | The `QueryCache` callbacks on the client |
| An infinite query with no `initialPageParam` | `initialPageParam` and `getNextPageParam`, both required |
| Two positional arguments to a hook | One object argument |

The `useQuery` callbacks are gone. A side effect that ran in `onSuccess` has
three new homes. They are the render, an effect with a stated system outside
React, and the `QueryCache` `onError` on the `QueryClient`. That callback takes
the signature `(error, query)`.

## Verification

```bash
# 1. Read the installed version before you write code.
node -p "require('@tanstack/react-query/package.json').version"

# 2. Find an inline queryFn outside a feature api module. Read every hit.
rg -n 'queryFn:' src/app src/components src/features --glob '!**/api/**'

# 3. Find a QueryClient in module scope. This must print nothing.
rg -n '^(const|let) [A-Za-z]+ = new QueryClient\(' src/

# 4. Find a query key that is a literal array rather than a factory call.
rg -n 'queryKey: \[' src/

# 5. Find a mutation with no reconciliation. Read every hit.
rg -n -A12 'useMutation\(' src/ | rg -v 'invalidateQueries|setQueryData|refetchQueries'

# 6. Find an onMutate with no onError. This must print nothing.
rg -l 'onMutate' src/ | xargs rg --files-without-match 'onError'

# 7. Find an unbounded invalidation. This must print nothing.
rg -n 'invalidateQueries\(\)' src/

# 8. Find a refetchInterval. Each hit needs a comment and a stop condition.
rg -n -B2 'refetchInterval' src/

# 9. Find a v4 idiom. This must print nothing.
rg -n 'cacheTime|useErrorBoundary|keepPreviousData: true|isInitialLoading' src/

# 10. The lint gate. It exits 0 and prints nothing.
pnpm exec eslint . --max-warnings=0

# 11. Confirm the rollback. Fail the write in the network panel, then read the
#     row. It returns to the committed value.

# 12. Confirm the prefetch. Load the route, and count the requests for that
#     resource in the network panel. There is one.
```

## Review checklist

- [ ] Is every server resource read through a query hook or an RSC prop, and
      never held in `useState` or a store?
- [ ] Does each query have one `queryOptions` object, in the feature `api`
      folder?
- [ ] Does every query module leave out `server-only`?
- [ ] Does every key come from a factory, and encode every input that changes
      the result?
- [ ] Does every `queryOptions` object state a `staleTime`?
- [ ] Is the `QueryClient` built for each request on the server, and once in the
      browser?
- [ ] Does the server `QueryClient` set `staleTime` above zero?
- [ ] Does the `retry` rule leave a deterministic failure unrepeated?
- [ ] Do the server prefetch and the client hook call the same `queryOptions`
      function?
- [ ] Is every prefetch outside every `"use cache"` scope?
- [ ] Does every data view render loading, error, empty, and ready, with a
      designed empty state?
- [ ] Does the first paint read `isPending` rather than `isFetching`?
- [ ] Does every suspending read have a `<Suspense>` boundary above it?
- [ ] Does every mutation state an invalidation, a cache write, or a reason for
      neither?
- [ ] Is every invalidation limited to the prefix that the write changed?
- [ ] Does every optimistic update cancel, snapshot, restore on a failure, and
      reconcile at the end?
- [ ] Does every next page param come from the `next` URL rather than from a
      computation?
- [ ] Do the infinite query properties run `queryFn`, `getPreviousPageParam`,
      `getNextPageParam`?
- [ ] Does every `queryFn` throw, so the failure reaches the `error` state?
- [ ] Does every `refetchInterval` carry a reason and a stop condition?
- [ ] Does the lint array hold `@tanstack/eslint-plugin-query`?
- [ ] Is every version 4 idiom rewritten to its version 5 form?

## Handoffs

- Where a value lives when the backend does not own it, the URL as a store, and
  the client store → `references/client-and-url-state.md`.
- The page that prefetches, the `HydrationBoundary`, and the Server Action that
  mutates on the server → `references/data-access-and-mutations.md`.
- The one typed client, the timeout, the retry rule, and `normalizeApiError` →
  `references/api-client-and-request-safety.md`.
- The `Paginated<T>` type, the DRF error envelopes, and the parse at the
  boundary → `references/boundary-validation-and-api-types.md`.
- The generated `paths` types and the drift gate →
  `references/openapi-schema-and-codegen.md`. The serializer and the pagination
  class belong to the backend.
- The `<Suspense>` boundary, the error boundary, and `useOptimistic` →
  `references/suspense-and-actions.md`.
- The server cache, `"use cache"`, `updateTag`, and the Router Cache →
  `references/caching-and-revalidation.md`.
- The `"use client"` directive that a hook forces onto a component →
  `references/server-and-client-components.md`.
- The lint config array that holds the plugin →
  `references/lint-format-and-scripts.md`.
- The state that no query owns, and the effect that reads a browser API →
  `references/state-and-effects.md`.
- The 401 that starts a token refresh, the `queryClient.clear()` on a logout,
  and the status that ends a session →
  `references/session-and-token-lifecycle.md`. This file owns only the 401 as
  an error state, and the retry rule here never repeats a refresh. The
  redirect after a 401, and the tenant that a key carries, are
  `references/route-protection-and-permissions.md`.
- The connection that pushes data, its reconnect, and its close code →
  `references/push-transport-and-connection.md`. The event that writes into
  this cache, and the order of that write against the invalidation, are
  `references/live-events-and-cache-merge.md`.
- The field-level map from a DRF 400 to a form control, and the submit that
  invalidates a key here after a success →
  `references/form-submission-and-server-errors.md`.
- The column model, the row model, and the virtualiser over an infinite query →
  `references/data-table-and-server-driven-state.md`.
- The upload progress of a mutation that carries a file →
  `references/file-upload-and-transport.md`.
- The three cases behind an empty state, and the words in an error state →
  `references/error-and-empty-state-copy.md`.
- The request count and the payload cost of a cache decision →
  `references/performance-budgets-and-measurement.md`.
- The MSW handlers → `references/network-mocks-and-contract-tests.md`. The
  test harness → `references/test-strategy-and-component-tests.md`. The
  assertions that prove this domain are the rollback after a failure, the empty
  state, and one `QueryClient` for each test.
- The N+1 query and the latency behind a slow endpoint → the backend.
