# Client state and URL state

nuqs 2.9, Zustand 5.0, React 19.2.4 or later, Next.js 16.3. This file owns
where a value lives when the backend does not own it. The subjects are the
state taxonomy and the single-owner rule. They also include the URL as the
store for shareable state, and the client store with its lifetime on the
server.

Every value that the backend owns is
`references/server-state-and-query-cache.md`. When an effect is correct, and
which component holds a local value, is `references/state-and-effects.md`. The
`"use client"` directive that a hook forces onto a component is
`references/server-and-client-components.md`.

## Principle

Every value has exactly one owner. Two owners produce two answers, and the user
sees whichever one rendered last.

The URL is a store that the user controls. A value that the user expects a
link, a bookmark, or the back button to carry belongs in it.

A store is a claim that no component owns the value. Write the claim first. A
store that no written reason supports is a habit.

A value that the program can compute is not state. `references/state-and-effects.md`
holds that rule and the effect that breaks it.

State that the server rendered must match the state that the browser holds on
the first paint. A store that reads the device before the hydration breaks that
match.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The taxonomy decides the owner

| The value | Where it lives | It reverses when | The cost |
| --- | --- | --- | --- |
| It came from Django or DRF | The query cache, or an RSC prop. `references/server-state-and-query-cache.md` | Never. A second owner of a backend value gives the user two answers. | The view must render a loading state, an error state, and an empty state. |
| The user expects a link, a bookmark, or the back button to restore it — a filter, a sort, a page, a tab, a search term, an open record | The URL, as a search param | The value is a secret, or it is too large for an address bar. | Every change writes a history entry, and every read passes through a parser. |
| It is UI only, and one component reads it — an open menu, a hover, a step index | `useState` or `useReducer`, in that component | A second component reads the value, so the value lifts to the common ancestor. | The value is lost on an unmount, and no link restores it. |
| It is UI only, it changes rarely, and a deep subtree reads it — a theme, a collapsed sidebar | A context, with the value and the dispatch split | The value changes many times each second, which the next row covers. | Each change re-renders every consumer of the value context. |
| It changes many times each second, or code outside React must read it, or it must survive a route change | A client store | One component reads the value, or the backend owns it. | A module, a provider for a request-specific value, and a selector at each read. |
| The program can derive it from a value that already exists | Nothing. Compute it during the render. | A measurement shows that the computation fails the render budget. | The computation runs on each render. |
| It is the value of a form field, before the submit | The form library. `references/form-schema-and-field-binding.md` | The field value must also appear in a link, which the URL row covers. | A dependency, and one more model of the same value. |

Read the table from the top. The first row that matches decides the owner, and
no value takes two rows.

### Shareable state goes in the URL

```tsx
// Wrong: the filter lives in the component.
// Failure: a copied link opens the unfiltered list, the back button leaves
// the filter on the screen and changes nothing, and a reload loses the work
// that the user did to reach this view.
const [search, setSearch] = useState("");
const [page, setPage] = useState(1);
```

```tsx
// Correct: the URL holds the value, and the query key derives from it.
"use client"; // reason: useQueryState reads and writes the address bar
import { parseAsInteger, parseAsString, useQueryStates } from "nuqs";

export function useOrderFilters() {
  const [filters, setFilters] = useQueryStates({
    search: parseAsString.withDefault(""),
    page: parseAsInteger.withDefault(1),
  });
  return { filters, setFilters };
}
```

`useQueryStates` writes both values in one navigation, so a change of the search
term and a reset of the page produce one history entry. `useQueryState` holds
one value, and two calls to it produce two entries.

Pass the result of this hook into `ordersListOptions(filters)`. The URL then
decides the cache key, and the two never disagree.
`references/server-state-and-query-cache.md` owns the key.

A search box that writes on every keystroke produces one history entry for each
letter. Give that input `limitUrlUpdates: debounce(ms)`, or wrap the write in a
transition. The address bar then follows the typing rather than records it.

### `useSearchParams` needs a `<Suspense>` boundary

```
Missing Suspense boundary with useSearchParams
```

The message appears in `next build` and never in `next dev`. The cause is a
component that reads the query string inside a route that Next prerenders as
static. Wrap the consumer in `<Suspense>`, or call `await connection()` in the
page so the route becomes dynamic. `references/caching-and-revalidation.md` owns
`connection()`, and `references/suspense-and-actions.md` owns the shape of the
fallback.

The server `searchParams` prop is a different value, and
`references/app-router-structure.md` states that a page which reads it is
dynamic. Read the query string on the server for the first render. Read it
with nuqs in the client component that changes it.

### nuqs

| Task | The call |
| --- | --- |
| One value in the URL | `useQueryState("q", parseAsString.withDefault(""))` |
| Several values in one navigation | `useQueryStates({ ... })` |
| A typed parser | `parseAsString`, `parseAsInteger`, `parseAsFloat`, `parseAsBoolean`, `parseAsIsoDate` |
| A default that the URL omits | `.withDefault(value)` |
| A read of the same params on the server | `createLoader`, or `createSearchParamsCache` |
| A limit on the write rate | `limitUrlUpdates: throttle(ms)`, or `limitUrlUpdates: debounce(ms)` |

Take the parser from the parser exports. Version 1 of nuqs held one
`queryTypes` object. Version 2 replaced it with one export for each parser, so
the bundle carries only the parsers that the code names. An import of
`queryTypes` is a version 1 file that no longer compiles, so `queryTypes` is
alive only in legacy code.

nuqs 2 ships as ESM only. A build that needs CommonJS needs a different package.

A value in the URL is a value from outside the program, so it needs the same
treatment as a response body.
`references/boundary-validation-and-api-types.md` owns the parse, and it states
what a page number of `abc` costs without one.

### When a store is justified

| Condition | Decision | It reverses when | The cost |
| --- | --- | --- | --- |
| Two sibling components read the value, and no natural parent holds it | A context first. A store where the value changes many times each second. | A natural parent appears in the tree, so the value lifts into it. | A provider in the tree, and a re-render of every consumer on each change. |
| The value crosses a route change, such as a wizard that spans pages | A store. | The steps move into one route, so one component holds the value. | The value survives a route that no longer needs it, until code clears it. |
| A context re-renders a large subtree on each change | A store, read through a selector. | The measurement states that the re-render is inside the budget. | A store module and a selector at each read site. |
| Code outside React reads the value, such as an event handler or a socket callback | A store, read through `getState()`. | The code moves inside React, so a hook reads the value. | A read through `getState()` does not re-render, so a stale read is possible. |
| The value came from the backend | NEVER a store. The query cache owns it. | Never. Nothing in a store revalidates. | The view must render the four states that a query produces. |
| One component reads the value | NEVER a store. `useState` owns it. | A second component reads the value, which the first row covers. | The value is lost when the component unmounts. |

Zustand is the default store of this stack. Reach for Jotai where the state is
many small independent values. Reach for Valtio where the model is a deep
mutable object. Keep Redux Toolkit only where it is already installed.

### A store at module scope leaks between requests

```ts
// Wrong: create() runs when the module loads.
// Failure: the server loads the module once, so every request shares one
// store. A value that belongs to one user reaches the next user, and the
// first render of each request starts from whatever the previous request
// left. The defect appears in production only.
import { create } from "zustand";

export const useCartStore = create<CartState>()((set) => ({
  items: [],
  add: (item) => set((state) => ({ items: [...state.items, item] })),
}));
```

```tsx
// Correct: a factory, and one store for each request, held in a context.
// src/features/cart/store.ts
import { createStore } from "zustand";

export function createCartStore(initial: CartState["items"] = []) {
  return createStore<CartState>()((set) => ({
    items: initial,
    add: (item) => set((state) => ({ items: [...state.items, item] })),
  }));
}
```

```tsx
// src/features/cart/store-provider.tsx
"use client"; // reason: the provider holds the store instance
import { createContext, use, useRef } from "react";
import { useStore } from "zustand";
import { createCartStore } from "./store";

type CartStore = ReturnType<typeof createCartStore>;

const CartStoreContext = createContext<CartStore | null>(null);

export function CartStoreProvider({
  children,
  initial,
}: {
  children: React.ReactNode;
  initial: CartState["items"];
}) {
  const storeRef = useRef<CartStore>(null);
  storeRef.current ??= createCartStore(initial);
  return <CartStoreContext value={storeRef.current}>{children}</CartStoreContext>;
}

export function useCartStore<T>(selector: (state: CartState) => T): T {
  const store = use(CartStoreContext);
  if (store === null) throw new Error("useCartStore must render inside CartStoreProvider");
  return useStore(store, selector);
}
```

A store with no request-specific value, such as a theme flag, is safe at module
scope. Any store that holds a value of one user takes the factory above.

Read the store through a selector, and never through the whole state object. A
component that takes the whole object re-renders on every change to any field.
Where the selector returns a new object or a new array, wrap it in `useShallow`,
so the comparison reads the fields and not the identity.

The rule that a context hook throws outside its provider is
`references/type-modeling-and-narrowing.md`. The compound component that shares
state the same way is `references/component-composition.md`.

### `persist` and the first paint

The `persist` middleware reads `localStorage`. The server has no
`localStorage`, so the server render uses the default value and the first client
render uses the stored value. React reports a hydration error, and the theme
flashes.

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| The value is a preference that the server must render correctly, such as a theme | Store it in a cookie, read the cookie on the server, and pass the value into the provider. | The value is larger than a cookie permits, or a request must not carry it. | The cookie travels on every request, and the route that reads it becomes dynamic. |
| The value may appear one frame late | Read the stored value after the mount, and render the default before it. | The frame is visible enough that a user reads it as a fault. | The user sees the default value for one frame. |
| The value is a draft that only the browser needs | `persist`, with no server render of the value. | The server starts to render the value, which the first row then covers. | The value is bound to one browser, and it does not follow the user to a second device. |

`references/server-and-client-components.md` owns the hydration error and the
rest of its causes.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those four facts on 16 August 2026.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `nuqs` 2.9 | Typed URL state. It supplements `useSearchParams`, and it does not replace it. | 2.9.5 | 5 Aug 2026 | Active. The repository takes commits each week. | None |
| Recommend | `zustand` 5.0 | The default store. Use `createStore` and a provider for any request-specific value. | 5.0.15 | 13 Aug 2026 | Active. Three open issues on the repository. | None |
| Conditional | `jotai` 2.20 | Many small independent values, each with its own subscribers. | 2.20.2 | 14 Jul 2026 | Active. A 3.0 alpha line runs beside it. | None |
| Conditional | `valtio` 2.3 | A deep mutable model that a proxy tracks. | 2.3.2 | 1 May 2026 | Active, at a slower release rate than its two siblings. | None |
| Conditional | `immer` 11.1 | The Zustand middleware for a nested update. Add it only where the spread is unreadable. | 11.1.17 | 16 Aug 2026 | Active. | None. Version 9.0.6 patched the three advisories from 2021. |
| Conditional | `@reduxjs/toolkit` 2.12 | Only where Redux is already installed. RTK Query then holds the server state, and TanStack Query does not. | 2.12.0 | 15 May 2026 | Active. | None |
| Conditional | `use-debounce` 10.1 | A debounce that nuqs `limitUrlUpdates` and a transition do not cover. | 10.1.1 | 29 Mar 2026 | Active. The repository takes commits, and the release rate is low. | None |
| Audit-only | `swr` 2.5 | An existing install. TanStack Query is the default of this stack. | 2.5.1 | 12 Aug 2026 | Active. | None |
| Reject | `@apollo/client` | It serves GraphQL. This stack talks to DRF over REST. | 4.2.12 | 14 Aug 2026 | Active. | None |

`references/dependencies-and-git-workflow.md` owns the rule that a new
dependency states its replacement, its size, and its maintenance status.

### Version discipline

Read the installed versions before you write code.

Zustand 5 is the pin. The `create` call takes the curried form
`create<T>()(...)` under `strict`. A module-level `create` that holds a
request-specific value is the version 4 idiom that the App Router breaks. That
idiom is alive only in legacy code.

nuqs 2 is the pin. Rewrite `import { queryTypes } from "nuqs"` to the parser
exports. `createLoader` and `createSearchParamsCache` need nuqs 2.3 or later.
The `throttleMs` option is deprecated. Rewrite it to
`limitUrlUpdates: throttle(ms)`, or to `limitUrlUpdates: debounce(ms)` where the
value comes from a keystroke.

## Verification

```bash
# 1. Read the installed versions before you write code.
node -p "require('zustand/package.json').version"
node -p "require('nuqs/package.json').version"

# 2. Find a filter, a sort, or a page in component state. Read every hit.
rg -n 'useState.*(search|query|filter|sort|page|tab)' -g '*.tsx' src/

# 3. Find a store at module scope. Each hit must hold no request-specific
#    value.
rg -n 'create[A-Za-z]*\(' -g '*.ts' -g '*.tsx' src/ | rg 'zustand|createStore'

# 4. Find a store that holds data from the backend. This must print nothing.
rg -n -A10 'createStore|create<' src/ | rg 'api\.|fetch\(|results'

# 5. Find a nuqs version 1 import and the deprecated write-rate option. Each
#    one must print nothing.
rg -n 'queryTypes' src/
rg -n 'throttleMs' src/

# 6. Find a useSearchParams consumer with no boundary above it. Read every hit.
rg -l 'useSearchParams' -g '*.tsx' src/

# 7. Build, which is the only place the Suspense message appears.
pnpm build

# 8. Confirm the URL. Change a filter, copy the address, open it in a new tab,
#    and read the same view. Then use the back button and read the previous
#    view.

# 9. Confirm the first paint. Load a route with a persisted value and read the
#    console for a hydration error.
```

## Review checklist

- [ ] Does every value have exactly one owner from the taxonomy table?
- [ ] Is every shareable control — filter, sort, page, tab, search — written to
      the URL?
- [ ] Does the query key derive from the URL value rather than from a second
      copy of it?
- [ ] Does a search input throttle its write, or wrap it in a transition?
- [ ] Is every `useSearchParams` consumer inside a `<Suspense>` boundary, or on
      a dynamic route?
- [ ] Does every value read from the URL pass through a parser?
- [ ] Does every store carry a written reason?
- [ ] Is every store free of data that the backend owns?
- [ ] Does every store that holds a request-specific value use `createStore` and
      a provider, rather than a module-level `create`?
- [ ] Does every store read go through a selector, with `useShallow` where the
      selector returns an object?
- [ ] Does every store context hook throw outside its provider?
- [ ] Does a persisted value that the server renders come from a cookie rather
      than from `localStorage`?
- [ ] Is every nuqs version 1 import rewritten to a parser export?

## Handoffs

- Every value that the backend owns, the cache key, and the mutation →
  `references/server-state-and-query-cache.md`.
- The derived value, the effect that is justified, the context split, and
  `useSyncExternalStore` → `references/state-and-effects.md`.
- The `"use client"` directive on a provider, and the hydration error →
  `references/server-and-client-components.md`.
- The server `searchParams` prop, and the route that it makes dynamic →
  `references/app-router-structure.md`.
- `connection()`, and the route that must run for each request →
  `references/caching-and-revalidation.md`.
- The `<Suspense>` boundary and the shape of the fallback →
  `references/suspense-and-actions.md`.
- The parse over a value from the URL or from storage →
  `references/boundary-validation-and-api-types.md`.
- The context hook that throws outside its provider →
  `references/type-modeling-and-narrowing.md`.
- The place of a store module inside a feature →
  `references/directory-and-module-boundaries.md`.
- The rule for a new dependency →
  `references/dependencies-and-git-workflow.md`.
- The session value, and the rule that a token and a permission list never
  reach `localStorage` → `references/session-and-token-lifecycle.md`. The role
  flag that the UI reads is
  `references/route-protection-and-permissions.md`.
- The form field value before the submit →
  `references/form-schema-and-field-binding.md`. The step that this file's
  parsers carry, and the guard over unsaved work, are
  `references/multi-step-forms-and-unsaved-work.md`.
- The column visibility and the sort model of a table →
  `references/data-table-and-server-driven-state.md`.
- The theme token behind a stored preference, and the class that must reach
  `<html>` before the first paint →
  `references/design-tokens-and-theming.md`.
- The locale segment in the address → domain 19
  `internationalization-and-rtl`. Not integrated yet.
- The re-render cost of a store read, and the INP that it produces → domain 16
  `performance-and-web-vitals`. Not integrated yet.
