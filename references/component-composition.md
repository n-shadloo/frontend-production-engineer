# Component composition

React 19.2.6 or later, `@types/react` 19, shadcn/ui on Base UI. This file owns
the shape of a component and the way that components compose. The subjects are
decomposition, `children` and slots, and the compound component. They also
include the controlled and uncontrolled choice, `ref` as a prop, the list key,
and the custom hook.

Where a value lives, and when an effect is correct, is
`references/state-and-effects.md`. The boundary that renders while a value is
absent is `references/suspense-and-actions.md`. The types of the props are
`references/type-modeling-and-narrowing.md`.

## Principle

A component has one responsibility. A component with several responsibilities
is hard to review, and each change to it risks the others.

A boolean prop that changes the layout is a request for a different structure.
Give the caller the structure, and the prop disappears.

The caller composes the markup. The component supplies the behavior and the
state that the parts share.

A component that owns its value is uncontrolled. A component whose parent owns
the value is controlled. An application component picks one. A library
component supports both.

Data goes down and events go up. A parent that reaches into a child breaks that
direction, and the call site shows no sign of it.

A key tells React which item a component belongs to. A position is not an
identity.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### Two thresholds start a decomposition

A component file of more than 200 lines is over the threshold. A component with
more than five `useState` calls is over the threshold. Decompose it before you
add to it. These two numbers are review thresholds, not a specification. Read
them as the point at which a second reader can no longer follow the component.

Split by responsibility, never by line count. Move a group of related state and
its handlers into a custom hook. Move a self-contained part of the markup into
its own component. A page file that fetches data, transforms it, holds the form
state, and renders four panels is four things.

### Composition, not configuration

```tsx
// Wrong: three boolean props decide the structure.
// Failure: the three props give eight combinations. The design supports two
// of them, and no reader can tell which two. Each new variant adds a fourth
// prop and doubles the count again.
type CardProps = {
  title: string;
  withHeader?: boolean;
  withFooter?: boolean;
  compact?: boolean;
};

function Card({ title, withHeader, withFooter, compact }: CardProps) {
  return (
    <section data-compact={compact}>
      {withHeader && <h2>{title}</h2>}
      <CardBody />
      {withFooter && <CardPagination />}
    </section>
  );
}
```

```tsx
// Correct: the caller composes the parts that it needs.
import type { PropsWithChildren } from "react";

function Card({ children }: PropsWithChildren) {
  return <section>{children}</section>;
}
function CardHeader({ children }: PropsWithChildren) {
  return <header>{children}</header>;
}
function CardFooter({ children }: PropsWithChildren) {
  return <footer>{children}</footer>;
}

// The call site states the structure, and the component states none of it.
<Card>
  <CardHeader>
    <h2>Orders</h2>
  </CardHeader>
  <OrdersTable />
  <CardFooter>
    <Pagination />
  </CardFooter>
</Card>;
```

Three or more boolean props that change the layout is the signal. Give the
caller `children`, a named slot prop, or a set of parts.

### The compound component

```tsx
// Correct: the parts share state through a context, and the caller composes
// the markup.
import { createContext, use, useMemo, useState } from "react";

type TabsContextValue = { active: string; setActive: (id: string) => void };
const TabsContext = createContext<TabsContextValue | null>(null);

function Tabs({
  defaultValue,
  children,
}: {
  defaultValue: string;
  children: React.ReactNode;
}) {
  const [active, setActive] = useState(defaultValue);
  const value = useMemo(() => ({ active, setActive }), [active]);
  return <TabsContext value={value}>{children}</TabsContext>;
}

function useTabs(): TabsContextValue {
  const context = use(TabsContext);
  if (context === null) throw new Error("Tabs.* must render inside <Tabs>");
  return context;
}

function TabsList({ children }: { children: React.ReactNode }) {
  return <div role="tablist">{children}</div>;
}

function Tab({ id, children }: { id: string; children: React.ReactNode }) {
  const { active, setActive } = useTabs();
  return (
    <button role="tab" aria-selected={active === id} onClick={() => setActive(id)}>
      {children}
    </button>
  );
}

function TabPanel({ id, children }: { id: string; children: React.ReactNode }) {
  const { active } = useTabs();
  return active === id ? <div role="tabpanel">{children}</div> : null;
}

Tabs.List = TabsList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;
```

The caller decides the order of the parts and the markup around them. The hook
throws when a part renders outside the parent, which is the rule that
`references/type-modeling-and-narrowing.md` states for every context hook. The
`useMemo` on the context value is an escape hatch for a project that does not
enable the React Compiler; `references/state-and-effects.md` rules on it. The
`role` and the `aria-selected` values above are the minimum that the pattern
needs. The accessible names are
`references/semantics-and-accessible-names.md`, and the full keyboard behavior
is `references/keyboard-focus-and-live-regions.md`. That domain holds a veto.

### Render the component as another element

A component that must render as a different element takes the element from the
caller. It never duplicates the wrapper, and it never nests a second
interactive element inside the first.

| The library | The prop |
| --- | --- |
| Base UI, which shadcn/ui uses by default | The `render` prop, built on `useRender` |
| Radix Primitives | `asChild`, built on `Slot` |

```tsx
// Wrong: the button wraps a link.
// Failure: the markup nests an interactive element inside another one. A
// keyboard user reaches two stops for one control, and the accessible name
// comes from the wrong element.
<Button>
  <a href="/orders">Orders</a>
</Button>
```

```tsx
// Correct: the caller supplies the element, and the component merges its
// props and its ref into it. This is the Base UI form, which shadcn/ui takes.
<Button render={<a href="/orders" />}>Orders</Button>

// On a Radix codebase the same call is:
// <Button asChild><a href="/orders">Orders</a></Button>
```

Confirm the prop name in the installed library before you write the code. The
two libraries do not use one name for this.

### Controlled or uncontrolled

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| The parent must read the value, or drive it | Controlled. Take `value` and `onChange`. | The parent only reads the final value, which the next row covers. | Every keystroke re-renders the parent, and the parent must hold the state. |
| The component owns the value, and the parent needs only the final one | Uncontrolled. Take `defaultValue`, and read the value from the form. | The parent must reset or preset the value while the component is mounted. | No code outside the component can read the value before the submit. |
| A library component must serve both callers | Take `value`, `defaultValue`, and `onChange`, and choose between them at run time. | The component is application code with one caller, so one mode serves it. | One more hook, and a run-time branch that needs a test for each mode. |

NEVER move a component from uncontrolled to controlled while it is mounted.
React reports "A component is changing an uncontrolled input to be controlled",
and the input loses its value. An input whose `value` starts as `undefined` and
becomes a string is this failure.

```ts
// Correct: one hook serves both callers. The prop wins where the caller
// passes it, and the internal state serves every other caller.
import { useState } from "react";

export function useControllableState<T>({
  value,
  defaultValue,
  onChange,
}: {
  value?: T;
  defaultValue: T;
  onChange?: (next: T) => void;
}) {
  const [internal, setInternal] = useState(defaultValue);
  const isControlled = value !== undefined;
  const current = isControlled ? value : internal;

  const setValue = (next: T) => {
    if (!isControlled) setInternal(next);
    onChange?.(next);
  };

  return [current, setValue] as const;
}
```

### `ref` is a prop, and a child is not a remote control

React 19 makes `ref` an ordinary prop on a function component. `forwardRef` is
alive only in legacy code. NEVER write `forwardRef` in new code. `references/type-modeling-and-narrowing.md` holds the
pair and the types.

```tsx
// Wrong: the parent reaches into the child to clear it.
// Failure: two places now own one value. The call site cannot tell what
// clear() resets, a rename inside the child breaks the parent in silence, and
// a test of the parent has to mount the child to reach the behavior.
function SearchBox({ ref }: { ref?: React.Ref<{ clear: () => void }> }) {
  const [query, setQuery] = useState("");
  useImperativeHandle(ref, () => ({ clear: () => setQuery("") }), []);
  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
```

```tsx
// Correct: the parent owns the value, so it clears the value itself.
function SearchBox({
  query,
  onQueryChange,
}: {
  query: string;
  onQueryChange: (next: string) => void;
}) {
  return <input value={query} onChange={(e) => onQueryChange(e.target.value)} />;
}
```

`useImperativeHandle` is a last resort. Keep it for an imperative browser
behavior that no value describes, such as `focus()`, `play()`, or `scrollTo()`.
Each remaining call carries a comment that states why lifted state does not
serve. A remount by `key` clears state without any handle at all;
`references/state-and-effects.md` holds that pattern.

### Every list item has a stable key

```tsx
// Wrong: the index is the key.
// Failure: after a sort or a filter, React matches by position. The third row
// keeps the second row's open menu and its half-typed input, and the user
// edits the wrong record.
{rows.map((row, i) => (
  <Row key={i} row={row} />
))}
```

```tsx
// Correct: the key is the identity of the record.
{rows.map((row) => (
  <Row key={row.id} row={row} />
))}
```

An index is permitted only on a list that is static, never sorted, never
filtered, and never reordered. The identity comes from the domain, so take it
from the backend record rather than from a value that the render computes.

### A custom hook names a behavior

A custom hook is a function that calls hooks. It holds behavior and it returns
values. It renders nothing. Extract one where a component holds two groups of
state with no relation between them. Extract one where two components repeat
the same sequence of hooks.

Copy a small hook into the repository rather than add a dependency for it. Take
a dependency only where the hook is hard to copy, such as a hook around the
intersection observer.

### The component libraries

The table gives each library its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those four facts on 16 August 2026.

| Tier | Library | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | shadcn/ui on Base UI | Base UI is the default since July 2026. It supplies the headless primitives, the `render` prop, and `data-slot`. The Radix path is the `-b radix` flag. The classes and the tokens are `references/component-styles-and-variants.md` and `references/design-tokens-and-theming.md`. | `shadcn` 4.18.0, and `@base-ui/react` 1.7.0 | 13 Aug 2026, and 4 Aug 2026 | Both active. The Base UI package name `@base-ui-components/react` is deprecated, and it is alive only in legacy code. | None on either package |
| Conditional | Radix Primitives | Use it where a Radix codebase exists, or where you need Context Menu, Hover Card, or Toast. It supplies `asChild` and `Slot`. Radix development has less focus now. Current but in decline. | `radix-ui` 1.6.7 | 24 Jul 2026 | Active. The repository takes commits, and it holds a large open-issue count. | None |
| Conditional | `usehooks-ts`, `react-use` | Prefer a copy into the repository. `react-use` gets less maintenance. Both are current but in decline. | `usehooks-ts` 3.1.1, and `react-use` 17.6.1 | 5 Feb 2025, and 10 Jun 2026 | `usehooks-ts` has taken no release for 18 months. `react-use` releases rarely, and it holds over 600 open issues. | None on either package |
| Reject | `prop-types` | React 19 ignores the checks, and it reports nothing. Use TypeScript. Alive only in legacy code. | 15.8.1 | 5 Jan 2022 | Unmaintained. The repository is archived. | None |

## Verification

```bash
# 1. Find every component file that is over the size threshold.
fd -e tsx . src | xargs wc -l | sort -rn | head -20

# 2. Find every component with more than five useState calls.
rg -c 'useState' -g '*.tsx' src/ | awk -F: '$2 > 5'

# 3. Find forwardRef. This must print nothing.
rg -n 'forwardRef' src/

# 4. Find an index key. Read every hit.
rg -n 'key=\{\s*(i|idx|index)\s*\}' -g '*.tsx' src/

# 5. Find useImperativeHandle. Each hit carries a comment that states why.
rg -n 'useImperativeHandle' src/

# 6. Find prop-types, which React 19 ignores. This must print nothing.
rg -n 'propTypes|from "prop-types"' src/

# 7. Find a component with three or more boolean props. Read every hit.
rg -n -B1 -A8 '^\s*(type|interface) \w+Props' -g '*.tsx' src/ | rg '\?: boolean'
```

## Review checklist

- [ ] Does every component have one responsibility?
- [ ] Is every component file at 200 lines or fewer, and every component at
      five `useState` calls or fewer?
- [ ] Does every layout variant come from composition rather than from three or
      more boolean props?
- [ ] Does a compound component share its state through a context rather than
      through cloned children?
- [ ] Does every context hook throw when it renders outside its provider?
- [ ] Does a component that renders as another element take that element from
      the caller?
- [ ] Is every control either controlled or uncontrolled for its whole life?
- [ ] Does a component that serves both callers use one hook to choose between
      the prop and the internal state?
- [ ] Does every new component take `ref` as a plain prop, with no
      `forwardRef`?
- [ ] Does every `useImperativeHandle` carry a comment that states why lifted
      state does not serve?
- [ ] Does every list key come from the record, and never from the index?
- [ ] Does every custom hook return values and render nothing?
- [ ] Is `prop-types` absent from the repository?

## Handoffs

- Where the state of a component lives, when an effect is correct, and the
  memoisation rule → `references/state-and-effects.md`.
- The Suspense boundary, the error boundary, and the React 19 Action →
  `references/suspense-and-actions.md`.
- The props types, the generics, `PropsWithChildren`, `ComponentProps`, and the
  `ref` types → `references/type-modeling-and-narrowing.md`.
- The `"use client"` directive on a leaf, and what a prop must serialize to
  cross the boundary → `references/server-and-client-components.md`.
- The classes on a part, `cn()`, and the variant API →
  `references/component-styles-and-variants.md`. The tokens behind them are
  `references/design-tokens-and-theming.md`.
- The ARIA roles of a compound component →
  `references/semantics-and-accessible-names.md`. Its keyboard behavior and
  its focus order → `references/keyboard-focus-and-live-regions.md`. That
  domain holds a veto.
- The field binding and the resolver on a form control →
  `references/form-schema-and-field-binding.md`. This file owns the shape of
  the field component, and that file owns the bind of its value.
- The virtualiser for a long list, and the column definitions →
  `references/data-table-and-server-driven-state.md`.
- The bundle bytes that a component adds →
  `references/client-bundle-and-third-party-scripts.md`.
