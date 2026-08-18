# State and effects

React 19.2.6 or later, React Compiler 1.0, `eslint-plugin-react-hooks` 7.1.1,
Next.js 16.3. This file owns where a value lives and when an effect is correct.
The subjects are state placement, `useState` against `useReducer`, the derived
value, and the context. They also include the rules for an effect, the Rules of
React, and the compiler that depends on them.

The shape of the component is `references/component-composition.md`. The
boundary that renders while a value is absent is
`references/suspense-and-actions.md`. The `"use client"` directive that state
forces onto a component is `references/server-and-client-components.md`.

## Principle

A value that the program can compute is not state. Compute it during the
render.

An effect synchronizes React with a system outside React. An effect that only
computes a value is a second render pass and a stale first frame.

State belongs to the lowest component that needs it. A value that sits higher
than its use re-renders a subtree that does not care about it.

A render is a pure function of props, state, and context. A compiler optimises
only what is pure, so purity is a build concern and not a matter of taste.

Data from the server is not the state of a component. It has an owner outside
the component tree, and it goes stale on its own schedule.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### Where the value lives

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| The program can derive the value from props or state that already exist | Compute it during the render. Add no `useState`, and add no effect. | A measurement shows that the computation fails the render budget. | The computation runs on each render. |
| Exactly one component reads the value | Keep it in that component. | A second component reads it, which the next row covers. | The value is lost when the component unmounts. |
| Two or more sibling components read the value | Lift it to the lowest common ancestor, and pass it down. | The ancestor is far above the readers, so the prop crosses components that ignore it. | Each change re-renders the ancestor and its whole subtree. |
| A deep subtree reads it, and it changes rarely | A context. Split the value and the dispatch into two contexts. | The value changes many times each second, so a store with a selector serves better. | Two providers in the tree, and a re-render of every consumer of the value context. |
| The value came from the backend | `references/server-state-and-query-cache.md` owns it. NEVER hold it in `useState`, and never in a context. | Never. Nothing in `useState` or a context revalidates. | The view must render a loading state, an error state, and an empty state. |
| One component holds more than five `useState` calls | Consolidate into one object, or into a `useReducer`. Then decompose the component. | The five values are independent, and no action changes two of them together. | One dispatch and one action type replace five setters, so a single field update is longer to write. |
| A global store | Write the reason first. A store that no written reason supports is a finding. | Never. The reason is the gate on the store. | A written reason precedes every store, and the review reads it. |

### `useState` or `useReducer`

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| Independent primitive values | One `useState` for each value. | The count passes five, or two of the values start to change together. | Each new value adds a setter, and a reset must name every one of them. |
| The next value depends on the previous one, across several actions | `useReducer`, with a discriminated union for the action type. | One action is left, so the reducer states one transition. | An action type, a reducer, and a dispatch for a value that one setter held. |
| Several values change together | `useReducer`, so one dispatch moves all of them. | The values become independent, which the first row covers. | The same. A single field update goes through an action. |
| The transitions need a unit test on their own | `useReducer`. The reducer is a pure function, so a test calls it directly. | The test of the component covers the transitions. | A test file for the reducer, beside the test of the component. |

`references/type-modeling-and-narrowing.md` holds the rules for the
discriminated union and for the exhaustive `switch` that reads it.

### A derived value is computed, never stored

```tsx
// Wrong: an effect copies a computed value into state.
// Failure: the component renders once with the empty string, then the effect
// runs, then it renders again. The user sees the empty frame, and the lint
// rule set-state-in-effect reports the line.
function FullName({ firstName, lastName }: { firstName: string; lastName: string }) {
  const [fullName, setFullName] = useState("");
  useEffect(() => {
    setFullName(`${firstName} ${lastName}`);
  }, [firstName, lastName]);
  return <p>{fullName}</p>;
}
```

```tsx
// Correct: the value is an expression in the render.
function FullName({ firstName, lastName }: { firstName: string; lastName: string }) {
  const fullName = `${firstName} ${lastName}`;
  return <p>{fullName}</p>;
}
```

A filtered list, a total, a formatted label, and a validity flag are all derived
values. None of them is state.

### One object or one reducer, never one state per field

```tsx
// Wrong: one useState for each field.
// Failure: five setters keep five values that belong to one record. A reset
// has to remember all five, a new field adds a sixth, and two fields drift
// apart when only one of them updates.
const [name, setName] = useState("");
const [email, setEmail] = useState("");
const [phone, setPhone] = useState("");
const [company, setCompany] = useState("");
const [role, setRole] = useState("");
```

```tsx
// Correct: one reducer holds the record, and one dispatch moves it.
type ContactAction =
  | { type: "field"; field: keyof Contact; value: string }
  | { type: "reset" };

function contactReducer(state: Contact, action: ContactAction): Contact {
  switch (action.type) {
    case "field":
      return { ...state, [action.field]: action.value };
    case "reset":
      return empty;
    default:
      return assertNever(action);
  }
}

const [contact, dispatch] = useReducer(contactReducer, empty);
```

A form that posts to the backend has a third option, which is better than both.
A React 19 Action holds the values in the `<form>` and returns the result as
state. `references/suspense-and-actions.md` holds that pattern.

### A context holds UI state, never server data

```tsx
// Wrong: the context holds the list that the backend owns.
// Failure: nothing revalidates the list, so it goes stale and the user acts
// on old rows. Every write to the context re-renders the whole subtree under
// the provider, including the parts that read no user.
const UserContext = createContext<User[]>([]);
```

```tsx
// Correct: the context holds UI state, and the value and the dispatch are two
// contexts.
type ThemeState = { theme: "light" | "dark" };

const ThemeStateContext = createContext<ThemeState | null>(null);
const ThemeDispatchContext = createContext<React.Dispatch<ThemeAction> | null>(null);
```

A component that reads only the dispatch context does not re-render when the
value changes. Data from the backend goes through the query layer, which is
`references/server-state-and-query-cache.md`. The rule that a fetch in `useEffect`
belongs on the server is
`references/server-and-client-components.md`.

### An effect needs a system outside React

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| Transform a value for the render | Do it in the render. No effect. | A measurement shows that the transform fails the render budget. | The transform runs on each render. |
| Respond to a click, a submit, or a key | An event handler. No effect. | The system outside React starts the work, so no event exists to attach to. | The work runs only where the user acts, so a programmatic change does not start it. |
| Reset state when a prop changes | A `key` on the component, or a value computed in the render. No effect. | Part of the state must survive the reset, which a remount discards. | The `key` change remounts the component, so every value inside it is lost. |
| Read a browser API or an external store | `useSyncExternalStore`. | The store is inside React, so state or a context holds it. | A subscribe function, a client snapshot, and a server snapshot for each value. |
| Talk to a non-React widget, a socket, a timer, or an analytics endpoint | `useEffect`. | The work is a pure transform, which the first row covers. | A cleanup for each setup, and a second run of both under Strict Mode. |
| Measure the DOM before the browser paints | `useLayoutEffect`. | The measurement may appear one frame late, so `useEffect` serves. | The browser waits for the code before it paints, so the work is on the paint path. |
| Read a value inside an effect that must not re-trigger it | Wrap that code in `useEffectEvent`. | The effect must in fact re-run when that value changes. | One more function in the component, which no dependency array and no child may hold. |

Every effect has a cleanup that undoes its setup. The two are symmetric: a
subscribe pairs with an unsubscribe, and an open pairs with a close.

### Reset state with a `key`, not with an effect

```tsx
// Wrong: an effect clears the draft when the profile changes.
// Failure: the component renders the old draft once, then clears it. The next
// field that someone adds to the form is forgotten here, so it keeps the
// previous profile's value.
useEffect(() => {
  setDraft("");
}, [profileId]);
```

```tsx
// Correct: the key changes, so React discards the old component and its whole
// state.
<ProfileForm key={profileId} profileId={profileId} />
```

### `useEffectEvent` reads the latest value without a re-trigger

```tsx
// Wrong: a value that the effect reads sits in the dependency array.
// Failure: theme is in the deps, so a change of theme disconnects the socket
// and connects it again. The user loses the messages that arrive between the
// two.
useEffect(() => {
  const connection = createConnection(serverUrl, roomId);
  connection.on("connected", () => showNotification("Connected", theme));
  connection.connect();
  return () => connection.disconnect();
}, [roomId, theme]);
```

```tsx
// Correct: the non-reactive read moves into an Effect Event, which is never a
// dependency.
const onConnected = useEffectEvent(() => {
  showNotification("Connected", theme);
});

useEffect(() => {
  const connection = createConnection(serverUrl, roomId);
  connection.on("connected", () => onConnected());
  connection.connect();
  return () => connection.disconnect();
}, [roomId]);
```

`useEffectEvent` is stable since React 19.2. Three rules bind it. Declare it in
the same component or hook as the effect that calls it. NEVER list it in a
dependency array. NEVER pass it to a child, to a prop, or to a context.

### `useSyncExternalStore` for a browser API

```ts
// Correct: no effect, no hydration mismatch, and it is safe under a
// concurrent render.
import { useSyncExternalStore } from "react";

const query = "(max-width: 600px)";

function subscribe(callback: () => void) {
  const list = window.matchMedia(query);
  list.addEventListener("change", callback);
  return () => list.removeEventListener("change", callback);
}

export function useIsNarrow(): boolean {
  return useSyncExternalStore(
    subscribe,
    () => window.matchMedia(query).matches, // the client snapshot
    () => false, // the server snapshot, which the render on the server needs
  );
}
```

The third argument is required wherever the component renders on the server.
Without it the prerender stops with "Missing getServerSnapshot, which is
required for server-rendered content". Return a safe default that matches the
first client render.

### Strict Mode proves the cleanup

Next.js sets `reactStrictMode` to true. Keep it on. React then mounts,
unmounts, and mounts each component again in development, so an effect with a
missing cleanup runs its setup twice. Two connections, two timers, or two
analytics events in development, and one in production, is the report of a
missing cleanup. NEVER turn Strict Mode off to hide it. `next.config.ts` itself
is `references/app-router-structure.md`.

### The Rules of React, and the compiler that depends on them

A render reads props, state, and context, and it returns JSX. During a render,
these are the things that it never does:

- It mutates props, state, context, or a value in module scope.
- It reads a ref, or writes one.
- It calls the setter of another component.
- It runs a side effect, such as a fetch, a timer, or a write to storage.
- It produces a value that is not deterministic, such as `Math.random()` or
  `new Date()`. `references/server-and-client-components.md` states what that
  costs at the hydration.

```tsx
// Wrong: the render writes to a value in module scope.
// Failure: the render is impure, so the compiler skips this component and the
// component loses its memoisation with no message. The array also grows on
// every render, and Strict Mode makes it grow twice in development.
const seen: string[] = [];

function Row({ order }: { order: Order }) {
  seen.push(order.id);
  return <td>{order.id}</td>;
}
```

```tsx
// Correct: the render computes and returns. The record of what the user saw
// goes to an effect, which is the code that talks to a system outside React.
function Row({ order }: { order: Order }) {
  useEffect(() => {
    trackImpression(order.id);
    return () => cancelImpression(order.id);
  }, [order.id]);
  return <td>{order.id}</td>;
}
```

Enable the compiler in `next.config.ts`. It is stable in Next 16, and it is off
by default.

```ts
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = { reactCompiler: true };

export default nextConfig;
```

`reactCompiler` is a top-level key, and not a key under `experimental`. Add
`babel-plugin-react-compiler` as a development dependency. Adopt the compiler
in one step where the lint gate is already clean. Where it is not, set
`compilationMode: 'annotation'`, then mark a file or a component with
`"use memo"`, and opt a file out with `"use no memo"`. Measure the build time
in CI before the change and after it. React publishes no figure for the build
cost.

```ts
// eslint.config.ts — the flat config of eslint-plugin-react-hooks v7.
import reactHooks from "eslint-plugin-react-hooks";
import { defineConfig } from "eslint/config";

export default defineConfig([reactHooks.configs.flat.recommended]);
```

The recommended preset carries `rules-of-hooks`, `exhaustive-deps`,
`set-state-in-render`, `set-state-in-effect`, and the rules for refs, purity,
and immutability. It surfaces the compiler diagnostics even in a project that
has not adopted the compiler. Treat the preset as a build gate, not as advice.
The type-aware TypeScript rules that run beside it are
`references/typescript-config-and-enforcement.md`.

### Do not hand-memoise

A hand-written `useMemo`, `useCallback`, or `memo` is current but in decline.
The React Compiler replaces it in every ordinary case.

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| Every ordinary case | Add nothing. The compiler memoises. | The project does not enable the compiler, and a measurement proves a cascade. | Nothing memoises until the compiler is on, so a project without it re-renders more. |
| A value that an effect lists as a dependency, and that must keep one identity | Keep `useMemo` or `useCallback`. State the reason in a comment. | The value leaves the dependency array, such as through `useEffectEvent`. | A dependency array that a later reader must keep correct by hand. |
| A promise that `use()` reads | Keep `useMemo`, so the render does not create a second promise. `references/suspense-and-actions.md` | A Server Component starts the promise and passes it as a prop. | One memo for each promise, and a client request that the server could have made. |
| A context provider value, in a project without the compiler | `useMemo` on the value. Delete it when the project enables the compiler. | The project enables the compiler, and the line is then deleted. | One memo for each provider, which stays after it is no longer needed. |
| The compiler skipped a component that is hot | Correct the rule violation first. Measure again before you add a memo. | The violation is in third-party code that this repository cannot change. | The correction is a rewrite of the component, not one added line. |
| A manual memo in code that already exists | Leave it. Its removal can change what the compiler emits. | The memo is provably wrong, such as a dependency array that lies. | The file holds a memo that the compiler makes unnecessary. |

NEVER add `useMemo`, `useCallback`, or `memo` without a measurement. Use the
React Profiler, or the Performance Tracks of React 19.2, to prove the cascade
first.

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the floor. The advisories of December 2025 and January 2026
describe a family of remote code execution and denial of service defects in
React Server Components. CVE-2025-55182 is the first of them, with a CVSS
score of 10.0. Release 19.2.4 fixed CVE-2026-23864, and CVE-2026-23870 then
marked 19.2.0 through 19.2.5 vulnerable, which moved the floor to 19.2.6. This
family produces a new advisory often, so read the advisory database before each
release rather than this line.

`eslint-plugin-react-hooks` 7.1.1 is the pin. Version 6.1.0 holds two defects
that this stack meets: its `recommended` preset fails inside a flat-config
array, and the plugin crashes against Zod 4. Version 7 supports ESLint 9 and
ESLint 10, and its peer range accepts Zod 3.25 and Zod 4.

`babel-plugin-react-compiler` 1.0 is the pin. Remove
`eslint-plugin-react-compiler` where a project still has it, because its rules
moved into `eslint-plugin-react-hooks`. That plugin is alive only in legacy
code.

The five idioms in the next paragraph are alive only in legacy code. Rewrite
the stale idioms that a generator produces.

`forwardRef(` becomes a `ref` prop. `ReactDOM.render(` becomes `createRoot`. A
`propTypes` or a `defaultProps` assignment on a function component becomes a
type and a default parameter. `experimental_useEffectEvent` becomes
`useEffectEvent`. A `next lint` script becomes a direct call to ESLint.

## Verification

```bash
# 1. Read the installed versions before you write code.
node -p "require('react/package.json').version"
node -p "require('eslint-plugin-react-hooks/package.json').version"

# 2. The lint gate. It exits 0 and prints nothing.
pnpm exec eslint . --max-warnings=0

# 3. Build, then read the compiler diagnostics for the files that changed.
#    Each skipped component is a rule violation to correct.
pnpm build

# 4. Find an effect whose only work is a setter. Read every hit.
rg -n -A4 'useEffect\(' -g '*.tsx' src/ | rg 'set[A-Z]'

# 5. Find a useEffectEvent value inside a dependency array. This must print
#    nothing.
rg -n -A6 'useEffectEvent' -g '*.tsx' src/ | rg '\}, \[.*on[A-Z]'

# 6. Find a global store. Each one needs a written reason.
rg -n 'createStore|zustand|redux|jotai' src/

# 7. Confirm that Strict Mode is on, then run the application in development
#    and count the network calls, the timers, and the subscriptions.
rg -n 'reactStrictMode' next.config.ts
pnpm dev

# 8. Confirm that the effects run in matched pairs in a production build.
pnpm build && pnpm start
```

## Review checklist

- [ ] Is every derived value computed during the render rather than stored?
- [ ] Is every effect justified by a system outside React?
- [ ] Does every effect have a cleanup that undoes its setup?
- [ ] Does state sit in the lowest component that reads it?
- [ ] Is data from the backend absent from `useState` and from every context?
- [ ] Does every context hold UI state, with the value and the dispatch split?
- [ ] Does a component with more than five `useState` calls use one object or
      one reducer?
- [ ] Does a prop change reset state through a `key` rather than through an
      effect?
- [ ] Is every `useEffectEvent` absent from every dependency array, and absent
      from every child, prop, and context?
- [ ] Does every `useSyncExternalStore` call pass the third argument?
- [ ] Is `reactStrictMode` on, and does the development run produce no
      duplicate side effect?
- [ ] Does every render avoid a mutation, a ref read, a side effect, and a
      value that is not deterministic?
- [ ] Does `reactCompiler` sit at the top level of `next.config.ts`?
- [ ] Does the lint gate run `eslint-plugin-react-hooks` version 7 or later?
- [ ] Does every hand-written `useMemo`, `useCallback`, and `memo` state its
      measurement or its escape hatch?
- [ ] Is the compiler free of a bailout on every file that this change touched?

## Handoffs

- The shape of the component that holds the state, and the custom hook that
  groups it → `references/component-composition.md`.
- The Suspense boundary, the React 19 Action, and the optimistic value →
  `references/suspense-and-actions.md`.
- The `"use client"` directive that state forces onto a component, and the
  fetch in `useEffect` that belongs on the server →
  `references/server-and-client-components.md`.
- The discriminated union for a reducer action, and the exhaustive `switch` →
  `references/type-modeling-and-narrowing.md`.
- The type-aware lint config that runs beside the React rules, and the CI gates
  → `references/typescript-config-and-enforcement.md`.
- The `next.config.ts` keys other than `reactCompiler` →
  `references/app-router-structure.md`.
- The query cache, the query keys, `staleTime`, and the mutation state →
  `references/server-state-and-query-cache.md`. That file owns every value that
  the backend produces.
- The URL as a store for state that must survive a reload, and the client store
  → `references/client-and-url-state.md`.
- The connection behind a subscription, the provider that holds it, and the
  reconnect → `references/push-transport-and-connection.md`. This file owns the
  rules for an effect, and that file owns the connection inside one.
- The render cost that a state change produces, and the INP that it costs →
  `references/paint-and-interaction-cost.md`.
- The test for a reducer and for a custom hook →
  `references/test-strategy-and-component-tests.md`.
