# Server and Client Components

Next.js 16.3 baseline, React 19.2.6 or later. This file owns the boundary
between the server tree and the client tree. The subjects are the placement of
`"use client"` and what crosses the boundary. They also include the way the
server streams a value to the client, and the cause of a hydration error.

The route tree around the boundary is `references/app-router-structure.md`. The
source of the data is `references/data-access-and-mutations.md`. The cache of
that data is `references/caching-and-revalidation.md`.

## Principle

A component that renders on the server costs the user no bytes. A component
that renders on the client costs the user the component, its imports, and the
imports of those imports. The boundary is therefore a budget line, not a
style choice.

The boundary is one-directional. A server tree can hold a client leaf. A
client tree cannot hold a server component that it imports, because the import
makes the module part of the client bundle.

Props that cross the boundary travel over the wire. They must serialize.

The server and the client must produce the same first render. A value that
differs between the two breaks the hydration.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The decision

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| Needs `useState`, `useEffect`, `useReducer`, `useContext`, an event handler, or a browser API | Client Component. Put `"use client"` on that leaf. | The state can move to the URL or to the server, so no hook is left. | The component, its imports, and their imports all ship to the browser. |
| Only reads data, reads a secret, or renders markup | Server Component. Add no directive. | The markup needs one event handler, which the first row then covers. | No part of it can hold state, so an interaction needs a new client leaf around it. |
| An interactive shell around mostly static content | Keep the parent on the server. Pass the static subtree as `children`. | The shell must read a value out of the subtree, which a prop cannot carry. | The call site holds the composition, so the shell no longer states its own structure. |
| Needs a package that touches `window` | Client Component. Isolate it in the smallest wrapper. | The package ships a server-safe entry point. | One more file for each such package, and the bytes of the package itself. |
| Reads the query string for display only | Client Component with `useSearchParams()`. The server `searchParams` prop makes the whole route dynamic. | The route is dynamic for another reason, so the prop costs nothing more. | The consumer needs a `<Suspense>` boundary above it, or the build stops. |

Every `"use client"` carries a one-line reason comment directly above it. A
directive with no reason is a finding.

### The directive belongs on the leaf

```tsx
// Wrong: the directive sits on the page.
// Failure: the whole route becomes a client tree. Every child ships
// JavaScript, the server data fetch is no longer available, and getData()
// must move into a useEffect with a spinner in front of it.
"use client";
export default function Page() {
  const [open, setOpen] = useState(false);
  return (
    <div>
      {/* a large static subtree */}
      <button onClick={() => setOpen(!open)}>Toggle</button>
    </div>
  );
}
```

```tsx
// Correct: the page stays a Server Component.
// app/page.tsx
import { Toggle } from "./toggle";
import { getData } from "@/lib/dal/data";

export default async function Page() {
  const data = await getData();
  return (
    <div>
      {/* the same subtree, rendered on the server */}
      <Toggle label={data.label} />
    </div>
  );
}
```

```tsx
// app/toggle.tsx
"use client"; // reason: useState and an onClick handler
import { useState } from "react";

export function Toggle({ label }: { label: string }) {
  const [open, setOpen] = useState(false);
  return <button onClick={() => setOpen((v) => !v)}>{open ? "Close" : label}</button>;
}
```

NEVER put `"use client"` on `app/layout.tsx` or on a page shell. A provider
that needs the client goes into its own small wrapper, and the layout renders
that wrapper.

### A Server Component crosses as a prop, never as an import

```tsx
// Wrong: a Client Component imports a Server Component.
// Failure: the server module joins the client bundle. Its server-only
// imports throw, and its secrets are now in browser-readable code.
"use client";
import ServerData from "./server-data";

export function Panel() {
  return <div><ServerData /></div>;
}
```

```tsx
// Correct: the server parent passes the rendered output as a prop.
// app/page.tsx
import { Panel } from "./panel";
import ServerData from "./server-data";

export default function Page() {
  return <Panel><ServerData /></Panel>;
}
```

```tsx
// app/panel.tsx
"use client"; // reason: the panel holds open/closed state
export function Panel({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>;
}
```

### What serializes across the boundary

Strings, numbers, booleans, arrays, plain objects, `Date`, `Map`, `Set`,
promises, and JSX cross the boundary. A class instance, a function, and a
method on an object do not.

```tsx
// Wrong: a class instance and a callback cross the boundary.
// Failure: the render fails with a serialization error at run time. The
// component works in development until the prop first carries a real
// instance.
<PriceTag money={new Money(1200, "EUR")} format={(v) => v.toFixed(2)} />
```

```tsx
// Correct: plain data crosses, and the client reconstructs what it needs.
<PriceTag amountMinor={1200} currency="EUR" />
```

A function crosses the boundary only as a Server Action, which is declared in
a `"use server"` file. See `references/data-access-and-mutations.md`.

### Stream a promise to the client and unwrap it with `use()`

```tsx
// Correct: the server starts the fetch, does not await it, and streams.
// app/post/[id]/page.tsx
import { Suspense } from "react";
import { Comments } from "./comments";
import { getComments } from "@/lib/dal/comments";

export default async function Page(props: PageProps<'/post/[id]'>) {
  const { id } = await props.params;
  const commentsPromise = getComments(id); // started, not awaited
  return (
    <Suspense fallback={<CommentsSkeleton />}>
      <Comments promise={commentsPromise} />
    </Suspense>
  );
}
```

```tsx
// app/post/[id]/comments.tsx
"use client"; // reason: use() unwraps the streamed promise here
import { use } from "react";
import type { Comment } from "@/lib/api/types";

export function Comments({ promise }: { promise: Promise<Comment[]> }) {
  const comments = use(promise);
  return <ul>{comments.map((c) => <li key={c.id}>{c.body}</li>)}</ul>;
}
```

The page returns its shell immediately. The comments arrive in a later chunk.

### The client fetch that belongs on the server

```tsx
// Wrong: the component fetches after it mounts.
// Failure: the user sees a spinner, the HTML holds no content, the request
// waits for hydration, and the endpoint is now public browser traffic.
"use client";
export function List() {
  const [items, setItems] = useState([]);
  useEffect(() => {
    fetch("/api/items").then((r) => r.json()).then(setItems);
  }, []);
  return <ul>{items.map((i) => <li key={i.id}>{i.name}</li>)}</ul>;
}
```

```tsx
// Correct: the server fetches, and the first paint holds the content.
import { getItems } from "@/lib/dal/items";

export default async function List() {
  const items = await getItems();
  return <ul>{items.map((i) => <li key={i.id}>{i.name}</li>)}</ul>;
}
```

A client component that must refetch after the mount is a different case. Fetch
it on the server first, then hydrate the client cache. That pattern is in
`references/data-access-and-mutations.md`.

### Enforce the boundary at build time

Import `server-only` at the top of every module that reads a secret, a
session, or the database. Import `client-only` in a module that touches
`window`. The build then fails on the wrong side of the boundary, instead of
the browser failing at run time.

```ts
// src/lib/dal/session.ts
import "server-only";
```

### An error boundary sits on the client side of the boundary

An error boundary holds state, so it is always a Client Component. The segment
file therefore carries the directive, and a `<Suspense>` fallback does not.

```tsx
// Correct: the segment error file carries the directive.
// app/dashboard/orders/error.tsx
"use client"; // reason: an error boundary needs client state

export default function Error({
  error,
  retry,
}: {
  error: Error & { digest?: string };
  retry: () => void;
}) {
  return (
    <div role="alert">
      <p>The orders did not load.</p>
      <button onClick={retry}>Try again</button>
    </div>
  );
}
```

`loading.tsx` and `error.tsx` as route files are
`references/app-router-structure.md`. Where a boundary belongs, the shape that
a fallback needs, and the retry that the boundary must perform are
`references/suspense-and-actions.md`.

### Hydration errors

The message is "Text content does not match server-rendered HTML". The cause
is always a difference between the server render and the first client render.

| Cause | Fix |
| --- | --- |
| `new Date()`, `Math.random()`, or a locale format in the render | Compute the value on the server, and pass it as a prop |
| `typeof window !== 'undefined'` in the render | Move the branch into an Effect |
| `window.matchMedia`, `localStorage`, or another browser API in the render | Read it in an Effect, after the mount |
| Invalid HTML nesting, such as an `<a>` inside an `<a>` | Correct the markup |
| Extra whitespace around the React root inside the HTML | Remove the whitespace |
| Data that differs between the server and the client | Make one source of truth |

```tsx
// Wrong: the timestamp differs between the two renders.
// Failure: React reports a hydration mismatch, discards the server HTML for
// that subtree, and re-renders it on the client.
export function Stamp() {
  return <time>{new Date().toLocaleTimeString()}</time>;
}
```

```tsx
// Correct: the server decides the value, and the client formats it after
// the mount.
"use client"; // reason: the formatted time is client-only
import { useEffect, useState } from "react";

export function Stamp({ iso }: { iso: string }) {
  const [text, setText] = useState(iso);
  useEffect(() => setText(new Date(iso).toLocaleTimeString()), [iso]);
  return <time dateTime={iso}>{text}</time>;
}
```

`suppressHydrationWarning` is an escape hatch. It works one level deep, React
does not patch the mismatched text, and it hides the next real mismatch in the
same element. Use it on a single unavoidable leaf, and never on a subtree. The
one exception is the `<html>` element under a theme script. The attribute alone
differs there, so the one-level behavior is the behavior that the case needs.
`references/design-tokens-and-theming.md` holds that rule.

## Verification

```bash
# 1. List every "use client". Each hit needs a reason comment above it.
rg -n '^\s*["'"'"']use client["'"'"']' -g '*.tsx' -g '*.ts' .

# 2. Find the directive on a layout or a page shell. This must print nothing.
rg -l 'use client' -g '**/app/**/layout.tsx' -g '**/app/**/page.tsx' .

# 3. Confirm that every secret-reading module is server-only.
rg --files-without-match 'server-only' src/lib/dal

# 4. Build, then load each route and read the console for a hydration error.
pnpm build && pnpm start
```

## Review checklist

- [ ] Is every `"use client"` on a leaf, with a one-line reason above it?
- [ ] Is `app/layout.tsx` free of `"use client"`?
- [ ] Does every Client Component receive a Server Component as a prop rather
      than import it?
- [ ] Does every prop that crosses the boundary serialize?
- [ ] Does the server start each slow fetch, so the client unwraps a promise
      and never starts a new request?
- [ ] Is every `useEffect` fetch justified by a need to refetch after the
      mount?
- [ ] Does every module that reads a secret or a session import
      `server-only`?
- [ ] Does every `error.tsx`, and every error boundary in the tree, carry
      `"use client"`?
- [ ] Does the console report no hydration error on any route?
- [ ] Is `suppressHydrationWarning` limited to a single unavoidable leaf?

## Handoffs

- The props types, the generics, and the discriminated unions on a component →
  domain 02 `typescript-type-system-discipline`, in
  `references/type-modeling-and-narrowing.md`.
- Which component holds the state that makes a leaf a Client Component →
  domain 03 `react-component-architecture`, in
  `references/state-and-effects.md`. This file owns the directive and the
  boundary; that file owns the placement of the value.
- Composition, the slots, and the list key → `references/component-composition.md`.
- The granularity of a `<Suspense>` boundary inside a route, the React 19
  Actions, and the promise that `use()` reads →
  `references/suspense-and-actions.md`.
- The client cache, `useQuery`, and the mutation state →
  `references/server-state-and-query-cache.md`. The store and the URL value that
  a provider carries are `references/client-and-url-state.md`.
- The accessible name → `references/semantics-and-accessible-names.md`. The
  focus order, and the live region on a skeleton or an error boundary →
  `references/keyboard-focus-and-live-regions.md`. That domain holds a veto.
- The bundle bytes that a client leaf adds →
  `references/client-bundle-and-third-party-scripts.md`. The INP that it
  costs → `references/paint-and-interaction-cost.md`.
- The words in an error state, and the three cases behind an empty state →
  `references/error-and-empty-state-copy.md`.
