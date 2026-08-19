# Keyboard, focus, and live regions

React 19.2.6 or later, Next.js 16.3, WCAG 2.2 Level AA, WAI-ARIA 1.2, and the
ARIA Authoring Practices Guide. This file owns the path that a keyboard takes
through a surface, the place that focus goes, and the change that a screen
reader hears. The subjects are the tab order, the focus indicator, the keyboard
contract of a pattern, and the modal. They also include the focus on a route
change, the skip link, and the keyboard trap. The last subjects are the single
announcer of the application, and the error summary of a form.

The element, the role, and the name are
`references/semantics-and-accessible-names.md`. The contrast of the focus
indicator and the size of a target are
`references/visual-and-motor-criteria.md`. The tooling that proves this file is
`references/wcag-conformance-and-verification.md`.

## Principle

Everything that a pointer can do, a keyboard can do. A feature that only a
mouse reaches does not exist for a part of the users.

Focus is the cursor of a keyboard user. It is always visible, and it is always
somewhere that the user chose or that the application chose on purpose.

The DOM order is the tab order. A visual order that the DOM does not carry
sends the user through the page in a sequence that nothing on the screen
explains.

A change that a sighted user sees is a change that a screen reader user hears.
A change that only the pixels report did not happen for that user.

An announcement on every frame is silence. The user hears noise, and the
important message arrives inside it.

A control that focus can enter and cannot leave is a trap. The user loses the
whole page, and not only the control.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The tab order is the DOM order

```tsx
// Wrong: a positive tabindex reorders the tab sequence.
// Failure: every positive value moves ahead of every element in the document
// order. One number on one control changes the sequence of the whole page,
// and the next author cannot predict it.
<input tabIndex={3} />
```

```tsx
// Correct: the DOM carries the order, and the CSS carries the appearance.
<input />
```

NEVER write a positive `tabIndex` value. Write `tabIndex={-1}` only where the
application moves focus to the element in code. The user must not reach that
element with the Tab key.

Order the markup in the order that the user reads it. Where the design needs a
different visual order, `references/layout-and-typography.md` owns the grid and
the flex properties that produce it without a DOM change.

### The focus indicator

The indicator is part of the design. `references/component-styles-and-variants.md`
owns the ring token and the rule that `outline-none` never stands alone. This
file owns the three criteria that the indicator must meet.

1. The indicator has a contrast ratio of at least 3:1 against the colors next
   to it. That ratio holds in the focused state and in the unfocused state.
2. The indicator is large enough to see. A one-pixel ring in a low-contrast
   color meets no criterion.
3. A sticky header, a sticky footer, or a floating panel never hides the
   focused control. Criterion 2.4.11 fails where the element is fully hidden.

```css
/* Correct: the scroll container reserves the height of the sticky header, so
   a focused control never lands behind it. */
html {
  scroll-padding-top: var(--header-height);
}
```

`references/visual-and-motor-criteria.md` owns the measurement of the 3:1
ratio. Tab through every surface, and watch the indicator. A control that you
cannot find is a control that a keyboard user cannot use.

### The keyboard contract of a pattern

The ARIA Authoring Practices Guide defines the keys of each pattern. This file
does not restate them. Read the pattern, and implement every key that it lists.

An application interface needs these patterns. They are the menu, the menubar,
the listbox, the combobox, and the tabs. The rest are the accordion, the
dialog, the alert dialog, the tree, and the grid. The last four are the slider,
the toolbar, the disclosure, and the carousel.

| The condition | The action | It reverses when | The cost |
| --- | --- | --- | --- |
| The project holds a primitive for the pattern | Use it. shadcn/ui on Base UI implements the pattern, and the primitive owns the keys. | The design needs behavior that the primitive does not expose. | The variants of the primitive, which `references/component-styles-and-variants.md` covers. |
| No primitive exists, and the pattern is in the APG | Implement every key that the pattern lists, and write the key map in a comment above the component. | A primitive lands for the pattern. | The component owns a contract that a library would maintain. |
| The interaction matches no APG pattern | Compose it from native elements and from links. Never invent a role. | Never. | Nothing. |

NEVER invent a keyboard map. A pattern with keys that no specification defines
teaches the user nothing that the next screen reuses.

Two mechanisms move the active item inside a composite control.

| The mechanism | How it works | Where it fits |
| --- | --- | --- |
| Roving `tabindex` | One item carries `tabIndex={0}` and the rest carry `tabIndex={-1}`. The arrow keys move real DOM focus, and the values move with it. | A composite whose items are the focus targets, such as a toolbar or a tab list |
| `aria-activedescendant` | Focus stays on the container. The container points at the active option through its `id`. | A composite where focus must stay in a text field, such as a combobox |

The composite is one stop in the tab sequence, under either mechanism. The Tab
key leaves the composite, and the arrow keys move inside it.

### The modal

| The condition | The action |
| --- | --- |
| The project holds a dialog primitive | Use it. It implements the pattern, including the trap and the return of focus. |
| No primitive, and the dialog is simple | The native `<dialog>` element with `showModal()`. The browser supplies the trap, the top layer, and the Escape key. |
| A hand-built overlay | Own the whole contract: the initial focus, the trap, the return of focus, `inert` on the background, and the Escape key. |

```tsx
// Wrong: a hand-built overlay with no focus contract.
// Failure: focus stays behind the overlay, the Tab key walks the page under
// it, and the close returns focus to <body>. The user restarts the whole tab
// path to reach the control that opened the dialog.
{open ? (
  <div className="fixed inset-0 bg-black/50">
    <div className="rounded-md bg-card p-6">…</div>
  </div>
) : null}
```

```tsx
// Correct: the browser traps the focus, and the close returns it.
"use client"; // reason: the dialog needs a ref and the imperative API
import { useRef } from "react";

export function ConfirmDialog({ onConfirm }: { onConfirm: () => void }) {
  const dialog = useRef<HTMLDialogElement>(null);
  const opener = useRef<HTMLButtonElement>(null);

  return (
    <>
      <button type="button" ref={opener} onClick={() => dialog.current?.showModal()}>
        Delete order
      </button>
      <dialog
        ref={dialog}
        aria-labelledby="confirm-title"
        onClose={() => opener.current?.focus()}
      >
        <h2 id="confirm-title">Delete this order?</h2>
        <button type="button" onClick={() => dialog.current?.close()}>
          Cancel
        </button>
        <button
          type="button"
          onClick={() => {
            onConfirm();
            dialog.current?.close();
          }}
        >
          Delete
        </button>
      </dialog>
    </>
  );
}
```

Four rules hold on every modal. Focus enters the dialog when it opens, and it
lands on the first control or on the heading. Focus stays inside the dialog
while it is open. Focus returns to the control that opened it on close. The
background is `inert`, so no input method reaches it.
`references/semantics-and-accessible-names.md` owns `inert`.

The scroll lock must not move the page. `references/layout-and-typography.md`
owns `scrollbar-gutter`, which reserves the width of the scrollbar.

### The focus on a route change

A client navigation replaces the content of the page and leaves focus where it
was. The user hears nothing, and the Tab key continues from a control that the
old page held.

| The action | What the user gets | The cost |
| --- | --- | --- |
| Move focus to the `<h1>` of the new page | The screen reader reads the new heading, and the tab path restarts at the top of the content. | The visible focus indicator appears on a heading, which some designs must style. |
| Announce the title of the new page in a live region | The screen reader reads the new title, and focus stays where the user left it. | A user who tabs after the navigation continues from the old position. |

Choose one, and use it on every route. Confirm what the installed version of
Next.js already announces on a client navigation before you add a second
announcer. Two announcements of one navigation are worse than one.

```tsx
// Correct: src/components/a11y/route-focus.tsx moves focus on a real
// navigation, and never on the first render.
"use client"; // reason: the effect reads the pathname and moves the focus
import { usePathname } from "next/navigation";
import { useEffect, useRef } from "react";

export function RouteFocus() {
  const pathname = usePathname();
  const previous = useRef(pathname);

  useEffect(() => {
    if (previous.current === pathname) return;
    previous.current = pathname;
    const heading = document.querySelector<HTMLElement>("main h1");
    if (heading === null) return;
    heading.tabIndex = -1;
    heading.focus();
  }, [pathname]);

  return null;
}
```

`references/state-and-effects.md` owns the effect rules that this component
follows. The guard on the previous value is what keeps the effect quiet under
Strict Mode, which runs it twice in development.

### The skip link and the bypass block

```tsx
// Correct: the first tab stop of the document skips the navigation.
<a
  href="#main-content"
  className="sr-only focus:not-sr-only focus:absolute focus:left-2 focus:top-2 focus:z-50 focus:rounded-md focus:bg-background focus:px-4 focus:py-2"
>
  Skip to content
</a>
```

The link is the first element in the body, and its target is the `id` of the
`<main>` element. The link is invisible until it takes focus, and then it is
fully visible.

The landmarks of the page are the second bypass mechanism. A screen reader user
moves between them directly.
`references/semantics-and-accessible-names.md` owns the landmarks.

### The keyboard trap and the shortcut

Detect a trap in one pass. Put focus on the first control of the surface. Press
the Tab key until focus returns to the address bar. A control that never
releases focus is a trap.

A single-character shortcut is a trap of another kind. A user of a speech input
tool produces characters that the application reads as commands.

Give a single-character shortcut one of three properties. The user can turn it
off. The user can remap it to a key with a modifier. It works only while the
control that owns it has focus.

Publish the shortcut list in a dialog that a documented key opens. A shortcut
that nobody can find serves nobody.

### Content on hover or on focus

Content that appears on a hover or on a focus meets three conditions.

1. **Dismissible.** The Escape key removes it, and the pointer stays where it
   is.
2. **Hoverable.** The pointer can move onto the new content without it
   disappearing.
3. **Persistent.** It stays until the user removes it, moves the pointer away,
   or the information becomes invalid.

A tooltip that only a hover produces fails for a keyboard user. Bind it to the
focus as well, or move the text into the accessible description.

### One announcer for the whole application

A live region reports a change to a screen reader with no move of focus. The
region must be in the DOM before the message arrives. A region that mounts with
its text announces nothing.

```tsx
// Wrong: the region mounts together with its message.
// Failure: the screen reader has nothing to observe, because the region and
// the text arrive in one update. The toast is silent.
{message === undefined ? null : <div role="status">{message}</div>}
```

```tsx
// Correct: src/components/a11y/announcer.tsx mounts once at the root, and
// only the text changes.
"use client"; // reason: the announcer holds the message and updates it
import {
  createContext,
  useCallback,
  useContext,
  useState,
  type PropsWithChildren,
} from "react";

type Politeness = "polite" | "assertive";
type Announce = (message: string, politeness?: Politeness) => void;

const AnnounceContext = createContext<Announce | null>(null);

export function AnnouncerProvider({ children }: PropsWithChildren) {
  const [polite, setPolite] = useState("");
  const [assertive, setAssertive] = useState("");

  const announce = useCallback<Announce>((message, politeness = "polite") => {
    if (politeness === "assertive") setAssertive(message);
    else setPolite(message);
  }, []);

  return (
    <AnnounceContext value={announce}>
      {children}
      <div className="sr-only" role="status" aria-live="polite" aria-atomic="true">
        {polite}
      </div>
      <div className="sr-only" role="alert" aria-live="assertive" aria-atomic="true">
        {assertive}
      </div>
    </AnnounceContext>
  );
}

export function useAnnounce(): Announce {
  const announce = useContext(AnnounceContext);
  if (announce === null) {
    throw new Error("useAnnounce needs an AnnouncerProvider above it");
  }
  return announce;
}
```

| The politeness | The role | What it is for |
| --- | --- | --- |
| `polite` | `status` | A result, a saved record, a count of rows, a change of connection status |
| `assertive` | `alert` | An error that stops the user, and a message that expires |
| A log of messages in order | `log` | A chat feed and an event feed, where the order carries meaning |

Take `assertive` rarely. It interrupts the user in the middle of a sentence.
Every message that can wait is `polite`.

Set `aria-atomic="true"` where the region must read the whole message. Leave it
absent where only the new part matters.

Four sources send messages to this one announcer. They are the toast, the
result of a form submit, the result of an asynchronous request, and the pushed
event of a live connection. One announcer keeps them in one queue.

`references/live-events-and-cache-merge.md` owns the pushed event itself, and
`references/push-transport-and-connection.md` owns the status of the
connection. This file owns the politeness of the message that reports either
one.

Announce a load and a progress value at intervals, and never on every update. A
progress bar that announces each percent produces a wall of speech. Announce
the start once, the end once, and the value at a stated period between them.

`references/server-state-and-query-cache.md` owns the four states of a data
view. Announce the change between two of those states, and never the render of
each one.

### The error summary and the focus on a submit

A form that fails validation must tell the user, move the user to the problem,
and name each field that failed.

```tsx
// Correct: the summary takes focus on a failed submit, and each entry links
// to the field that failed.
"use client"; // reason: the summary takes focus after a submit
import { useEffect, useRef } from "react";

type FieldError = { readonly field: string; readonly message: string };

export function ErrorSummary({ errors }: { errors: readonly FieldError[] }) {
  const summary = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (errors.length > 0) summary.current?.focus();
  }, [errors]);

  if (errors.length === 0) return null;

  return (
    <div ref={summary} tabIndex={-1} role="alert" aria-labelledby="error-summary-title">
      <h2 id="error-summary-title">The form has {errors.length} errors</h2>
      <ul>
        {errors.map((error) => (
          <li key={error.field}>
            <a href={`#${error.field}`}>{error.message}</a>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

The alternative is a move of focus to the first field that failed. Choose one
of the two, and use it on every form of the product.

A toast is never the only report of a validation error. It disappears, it sits
far from the field, and a screen reader user cannot return to it.
`references/boundary-validation-and-api-types.md` owns the DRF error envelope
that produces the field list, and
`references/suspense-and-actions.md` owns the Action state that holds it.
`references/form-submission-and-server-errors.md` owns the map that puts each
message on its control, and it states that this file owns the report.

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| The tab path jumps across the page | A positive `tabIndex` value | Search for `tabIndex={` with a positive number | Remove the value, and order the DOM |
| Focus disappears at one control | The indicator is absent, or a sticky header hides it | Tab through the surface, and watch | Restore the ring, or set `scroll-padding-top` |
| The Tab key never leaves a widget | A key handler stops the event | Tab from the first control to the address bar | Release the Tab key from the handler |
| Focus returns to the top of the page after a dialog closes | The close never returned focus to the opener | Open a dialog from a control low on the page, and close it | Return focus in the `onClose` handler |
| A navigation announces nothing | The route change moved no focus, and posted no message | Navigate with a screen reader open | Move focus to the heading, or announce the title |
| A toast is silent | The live region mounted together with its text | Trigger the toast with a screen reader open | Mount the region once at the root |
| The same message announces once, then never again | The text did not change, so the region reported no update | Send one message twice | Clear the region, and then set the new text |
| A screen reader talks over every action | The messages are `assertive`, or the progress announces each value | Complete one flow with a screen reader open | Move the messages to `polite`, and announce progress at intervals |
| A tooltip that a keyboard user never sees | The content binds to the hover alone | Tab to the control, and watch | Bind it to the focus, or move it into the description |

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

React 19 renders a context directly, as `<Context value={…}>`. The
`.Provider` form is deprecated, and
`references/component-composition.md` states the rule.

The native `<dialog>` element and the `inert` attribute both need a support
target that includes them. Confirm the target of the project before you drop a
hand-built trap.

## Verification

```bash
# 1. Find a positive tabIndex. This must print nothing.
rg -n 'tabIndex=\{[1-9]' -g '*.tsx' src/

# 2. Find a key handler that may swallow the Tab key. Read every hit.
rg -n 'onKeyDown|preventDefault\(\)' -g '*.tsx' src/

# 3. Confirm one announcer, mounted at the root.
rg -n 'aria-live|role="status"|role="alert"' -g '*.tsx' src/

# 4. Find a live region that mounts with its message. Read every hit.
rg -n '\{.*\?.*<div[^>]*(role="status"|aria-live)' -g '*.tsx' src/

# 5. Confirm the skip link, and confirm its target.
rg -n 'Skip to content|#main-content' -g '*.tsx' src/

# 6. Find a hand-built overlay with no dialog primitive. Read every hit.
rg -n 'fixed inset-0' -g '*.tsx' src/

# 7. Find a tooltip that binds to the hover alone. Read every hit.
rg -n 'onMouseEnter|onMouseOver' -g '*.tsx' src/

# 8. Complete the primary flow with no pointer. Unplug the mouse, and use the
#    keyboard alone from the first control to the confirmation.

# 9. Tab from the first control of each surface to the address bar. No
#    control holds the focus, and every stop shows a visible indicator.

# 10. Navigate between two routes with a screen reader open. The new page
#     announces itself exactly once.
```

## Review checklist

- [ ] Is every positive `tabIndex` value absent from the code?
- [ ] Does every interactive element take focus, and show a visible indicator?
- [ ] Does every focused control stay visible under a sticky header, a sticky
      footer, and a floating panel?
- [ ] Does every composite control implement the keys of its APG pattern?
- [ ] Is every composite control one stop in the tab sequence?
- [ ] Does every modal set the initial focus, trap the focus, and return it on
      close?
- [ ] Is the background of a modal `inert`?
- [ ] Does every route change move focus or announce the new page, in one way
      across the product?
- [ ] Does the document carry a skip link as its first tab stop?
- [ ] Can focus leave every widget with the Tab key alone?
- [ ] Does every single-character shortcut turn off, remap, or scope to a
      focused control?
- [ ] Does the application hold one announcer, mounted once at the root?
- [ ] Is every message `polite`, except the errors that stop the user?
- [ ] Does a progress value announce at intervals, rather than on each update?
- [ ] Does a failed submit move focus to the summary or to the first failed
      field?
- [ ] Is every hover-revealed content reachable by focus, and dismissible with
      the Escape key?

## Handoffs

- The role, the accessible name, the state attributes, and `inert` →
  `references/semantics-and-accessible-names.md`.
- The contrast ratio of the focus indicator, the target size, and the reflow →
  `references/visual-and-motor-criteria.md`.
- The axe rules, the lint plugin, and the manual pass →
  `references/wcag-conformance-and-verification.md`.
- The ring token, `outline-none`, and the variant API →
  `references/component-styles-and-variants.md`.
- `scrollbar-gutter`, the grid order, and the type scale →
  `references/layout-and-typography.md`.
- The effect rules, the cleanup, and `useSyncExternalStore` →
  `references/state-and-effects.md`.
- The context that a provider renders, and the compound component →
  `references/component-composition.md`.
- The Action state that holds the field errors, and the error boundary →
  `references/suspense-and-actions.md`.
- The four states of a data view, and the mutation that changes them →
  `references/server-state-and-query-cache.md`.
- The pushed event and the status of a connection →
  `references/live-events-and-cache-merge.md` and
  `references/push-transport-and-connection.md`.
- The route files and the client navigation →
  `references/app-router-structure.md`.
- The redirect on a refused request, and the message that replaces the page →
  `references/route-protection-and-permissions.md`.
- The resolver and the server rejection that produce the list behind an error
  summary → `references/form-schema-and-field-binding.md` and
  `references/form-submission-and-server-errors.md`.
- The step change that must move focus →
  `references/multi-step-forms-and-unsaved-work.md`.
- The keyboard path through a data grid, and the virtualiser →
  `references/data-table-and-server-driven-state.md`.
- The moment at which a focus move happens, after an entrance ends →
  `references/motion-primitives-and-reduced-motion.md`. The view transition
  itself → `references/view-transitions-and-animation-libraries.md`.
- The words of an announcement and of an error message, and the choice of
  channel → `references/error-and-empty-state-copy.md`. The words on a control
  → `references/interface-copy-and-voice.md`.
