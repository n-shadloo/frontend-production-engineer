# Multi-step forms and unsaved work

React Hook Form 7.85, Zod 4.4, Next.js 16.3, `nuqs` 2.9, React 19.2.6 or later,
against a Django and DRF backend. This file owns a form that crosses more than
one screen, and the work that must survive a navigation. The subjects are the
place that holds the step, and the schema for each step. They also include the
value that an unmounted field keeps, and the guard over each way out of the
page. The last subject is the answer that the user must not give twice.

The schema and the bind of a control are
`references/form-schema-and-field-binding.md`. The submit and the server error
are `references/form-submission-and-server-errors.md`. The URL parser and the
store are `references/client-and-url-state.md`.

## Principle

A step index in memory is work that a reload destroys. The user reads a
five-step flow as one task, and the browser reads it as one page.

A guard that covers one way out is not a guard. A page has several exits, and
the user takes whichever one is nearest.

A user who answered a question has answered it. Carry the answer forward, or
offer it as a choice.

A draft that leaves the device carries whatever the user typed. Decide what may
enter it before you write it.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The step lives in the URL

```tsx
// Wrong: the step lives in memory alone.
// Failure: a reload, a back navigation, and a shared link all return the user
// to the first step. Every value that the user typed goes with it, and the
// browser gives no warning before it happens.
const [step, setStep] = useState(0);
```

```tsx
// Correct: the URL carries the step, so the back button and a reload work.
"use client"; // reason: the step control reads and writes the URL
import { parseAsInteger, useQueryState } from "nuqs";

const [step, setStep] = useQueryState("step", parseAsInteger.withDefault(1));
```

`references/client-and-url-state.md` owns the parsers, the store, and the rule
that the URL owns a value which a link must reproduce. It states that a value
which crosses a route change belongs in a store, and a wizard that spans pages
is exactly that case.

Never put a password, a payment value, or a personal identifier in the URL. The
browser history, the server log, and the referrer header all keep it.

### One `useForm` for the flow, and one schema for each step

Hold the whole flow in one `useForm` call where the steps sit in one route. The
default `shouldUnregister` of `false` keeps the value of a field that unmounts,
so a step that leaves the screen keeps what the user typed.
`references/form-schema-and-field-binding.md` states that default.

```ts
// Correct: the step validates its own fields, and the flow advances after it.
// The array holds the literal field names, so the resolver types them, and
// noUncheckedIndexedAccess forces the guard on a step that no entry covers.
const stepFields = [
  ["email", "name"],
  ["street", "city"],
] as const;

async function next(): Promise<void> {
  const fields = stepFields[step - 1];
  if (fields === undefined) return;
  const valid = await form.trigger(fields);
  if (valid) await setStep(step + 1);
}
```

The final submit validates the whole record, and the server validates it again.
A per-step check is user experience, and it is never the authority.
`references/form-submission-and-server-errors.md` holds that rule.

A step whose fields depend on an earlier answer takes a
`z.discriminatedUnion()`, so each branch validates only its own fields.

Move focus to the heading of the new step. A flow that advances in silence
leaves a screen-reader user on a control that no longer exists.
`references/keyboard-focus-and-live-regions.md` owns that move, and it holds a
veto.

### Each way out of the page needs its own guard

| The exit | The mechanism | What it does not cover |
| --- | --- | --- |
| A click on a `next/link` `<Link>` | `onNavigate`, with `event.preventDefault()` inside it. Next.js 15.3 added the prop. | A click with a modifier key, a new tab, an address outside the application, the back and forward buttons, and a reload. |
| A tab close, a reload, and a navigation to another site | The `beforeunload` event, with `event.preventDefault()` in the handler. | Every navigation inside the application. The browser also shows its own message, and it ignores a custom one. |
| A `router.push` call from code | Nothing. Next.js 16 publishes no API that intercepts it. Ask the user before the call instead. | Itself. A community package patches the router, and a patch of a framework internal breaks on an upgrade. |
| Most navigations inside the application | The Navigation API `navigate` event, with `intercept()`. It reached Baseline "newly available" in January 2026. | A traverse. No browser cancels a back or a forward navigation through this API yet. |

Take `onNavigate` and `beforeunload` together. That pair covers the links and
the exits that leave the application. Add the Navigation API only as a third
layer, and never as the only guard.

```tsx
// Wrong: the guard covers the tab close alone.
// Failure: beforeunload never fires for a navigation inside the application.
// The user clicks a link, the page changes, and the work goes with no warning.
// The most common exit is the one that this guard misses.
useEffect(() => {
  window.addEventListener("beforeunload", warn);
  return () => window.removeEventListener("beforeunload", warn);
}, []);
```

```tsx
// Correct: one hook holds the browser exit, and the Link holds the in-page one.
"use client"; // reason: the guard listens to a browser event
import { useEffect } from "react";

export function useUnsavedGuard(blocked: boolean): void {
  useEffect(() => {
    if (!blocked) return;
    const warn = (event: BeforeUnloadEvent): void => {
      event.preventDefault(); // the browser shows its own message
    };
    window.addEventListener("beforeunload", warn);
    return () => window.removeEventListener("beforeunload", warn);
  }, [blocked]);
}
```

```tsx
// Correct: the link asks first, and it navigates after the answer.
<Link
  href="/orders"
  onNavigate={(event) => {
    if (!blocked) return;
    event.preventDefault();
    openLeaveDialog("/orders"); // it clears the block, then it pushes
  }}
>
  Orders
</Link>
```

The `onNavigate` handler runs at the moment of the click, so it cannot wait for
an answer. Cancel the navigation, open the dialog, and navigate from the dialog
after the user confirms. A synchronous `confirm()` also works, and it gives the
user a dialog that the product does not style.

`formState.isDirty` supplies the `blocked` value. Reset the form after a
successful submit, so the guard stops. Where more than one component reads the
value, hold it in one context, which `references/state-and-effects.md` covers.

The `beforeunload` prompt does not appear until the user interacts with the
page. That is a browser rule, and no code changes it.

An autosave replaces the guard only where every value has already reached the
server. A draft that sits in the browser alone is still unsaved work.

### Ask the user once

A flow must not ask for the same value twice.
`references/semantics-and-accessible-names.md` states the criterion. This file
states the mechanism. Read the earlier answer from the one form state or the
one store. Render it as a filled field, or offer it as a choice.

## Verification

```bash
# 1. Find a step index in memory. Read every hit.
rg -n 'useState\(0\)|useState<number>' -g '*.tsx' src/ | rg -i 'step|wizard'

# 2. Find a form that tracks dirty state with no guard. Read every hit.
rg -ln 'isDirty' -g '*.tsx' src/ | \
  xargs rg --files-without-match 'beforeunload|onNavigate'

# 3. Find a router.push in a form that can hold unsaved work. Read every hit.
rg -n 'router\.push' -g '*.tsx' src/

# 4. Find a secret in a URL parameter. This prints nothing.
rg -n 'useQueryState\(.*(password|token|card|ssn)' -g '*.ts*' src/

# 5. Reload the page in the middle of the flow. Confirm the step and the
#    values return.

# 6. Press the back button in the middle of the flow. Confirm that the step
#    moves back and that no value is lost.

# 7. Type in a field, then click a link inside the application. Confirm the
#    warning.

# 8. Type in a field, then close the tab. Confirm the warning of the browser.

# 9. Submit the flow, then repeat step 7. Confirm that no warning appears.
```

## Review checklist

- [ ] Does the URL or a store carry the step, rather than memory alone?
- [ ] Do the values of an earlier step survive a move forward and a move back?
- [ ] Does each step validate its own fields before the flow advances?
- [ ] Does the final submit validate the whole record?
- [ ] Does focus move to the new step after each advance?
- [ ] Does the flow warn before a click on a link inside the application?
- [ ] Does the flow warn before a tab close and before a reload?
- [ ] Does the code ask the user before every programmatic navigation away from
      unsaved work?
- [ ] Does the guard stop after a successful submit?
- [ ] Do the URL and any stored draft hold no password, no payment value, and
      no personal identifier?
- [ ] Does the flow ask for each answer once?

## Handoffs

- The schema, the resolver, and the bind of a control →
  `references/form-schema-and-field-binding.md`. That file also states the
  `shouldUnregister` default that this flow depends on.
- The submit, the server error map, and the reset after a success →
  `references/form-submission-and-server-errors.md`.
- The URL parser, the `nuqs` options, and the store that a value crosses a
  route in → `references/client-and-url-state.md`.
- The context that shares the block state, and the effect cleanup →
  `references/state-and-effects.md`.
- The `<Link>` component, the route files, and the client navigation →
  `references/app-router-structure.md`.
- The focus that a step change moves, and the message that announces it →
  `references/keyboard-focus-and-live-regions.md`. That domain holds a veto.
- The criterion that forbids a second request for the same answer →
  `references/semantics-and-accessible-names.md`. That domain holds a veto.
- The words in a warning dialog and in a step label → domain 15
  `ux-writing-and-content-design`. Not integrated yet.
- The test that reloads the page in the middle of the flow, and the test that
  presses the back button → domain 20 `testing-and-quality`. Not integrated
  yet.
