# Semantics and accessible names

React 19.2.6 or later, Next.js 16.3, WCAG 2.2 Level AA, WAI-ARIA 1.2, and the
ARIA Authoring Practices Guide. This file owns the element that a component
renders, the role that the element reports, and the name that a screen reader
reads. The subjects are the native element, the five rules of ARIA, the
accessible name, and the landmarks and the headings of a page. They also
include the state that a control reports, and the three ways to hide an
element. The last subjects are the alternative text of an image, the language
of a passage, and the label on a field.

The tab path, the focus, and the announcement are
`references/keyboard-focus-and-live-regions.md`. The contrast, the target size,
and the reflow are `references/visual-and-motor-criteria.md`. The tooling and
the conformance target are `references/wcag-conformance-and-verification.md`.

## Principle

A native element carries a role, a keyboard behavior, and a set of states.
An attribute list on a generic element carries the report of those things, and
none of the behavior.

ARIA changes what a control says about itself. It never changes what the
control does.

Every control has a name. A name that only a sighted user can infer from a
picture is not a name.

Structure is a promise to a reader who does not see the page. A heading that
only makes text large breaks that promise.

An image says something, or it says nothing. Both cases need a decision, and
the absence of a decision is the third case, which is a defect.

Hidden means hidden to everybody, or it means hidden to nobody. An element that
one input method reaches and another does not is a trap.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The native element first

```tsx
// Wrong: a div carries the behavior of a button.
// Failure: the element has no role, no place in the tab order, and no
// activation from the Enter key or the Space key. A keyboard user cannot
// reach it, and a screen reader reports plain text. The task fails.
<div className="rounded-md bg-primary px-4 py-2" onClick={submit}>
  Save
</div>
```

```tsx
// Correct: the native element carries the role, the keyboard, and the state.
<button type="button" className="rounded-md bg-primary px-4 py-2" onClick={submit}>
  Save
</button>
```

A `role="button"` beside a hand-written `keydown` handler is the same defect
with more code. It reports the role, and it still needs the tab stop, the two
activation keys, the disabled state, and the focus indicator.

| The intent of the user | The element | The reason |
| --- | --- | --- |
| A move to another address | `<Link href>`, which renders an `<a href>` | It reports a link, it carries a URL, and the browser can open it in a new tab. |
| A change of state on this page | `<button type="button">` | It reports a button, and the Space key activates it. |
| A submit of the form around it | `<button type="submit">` | The Enter key submits from any field in the form. |

An `<a>` element with no `href` attribute is not focusable, and it reports no
link. NEVER use it as a control.

`references/component-composition.md` owns the shape of the component around
these elements, and the prop that renders a primitive as another element. That
file states the rule that no interactive element nests inside another one.

### The five rules of ARIA

1. Use the native element or attribute where it carries the semantics and the
   behavior that the pattern needs.
2. Do not change the native semantics of an element, unless the pattern makes
   it necessary.
3. Make every interactive ARIA control operable with a keyboard.
4. Never put `role="presentation"` or `aria-hidden="true"` on a focusable
   element.
5. Give every interactive element an accessible name.

No ARIA is better than bad ARIA. A wrong role reports a promise that the
component does not keep, and the user trusts the report.

```tsx
// Wrong: an aria-label on a generic element.
// Failure: a div has no role, so the name computation produces nothing for
// most tools. The label is invisible to the screen reader and to the eye.
<div aria-label="Close" onClick={close}>×</div>
```

```tsx
// Correct: the element that takes the name is the element with the role.
<button type="button" aria-label="Close" onClick={close}>
  <span aria-hidden="true">×</span>
</button>
```

Add an ARIA attribute only where a named APG pattern requires it. Write the
pattern name in the component, so a reviewer can check the attribute against
it. `references/keyboard-focus-and-live-regions.md` owns the keyboard half of
each pattern.

### The accessible name

The name computation reads the sources in one order. `aria-labelledby` wins
over `aria-label`. `aria-label` wins over the content of the element. The
content wins over `title`.

Prefer visible text. Every user reads it, and it needs no assistive technology
to exist. Take `sr-only` text where the design allows no visible name. Take
`aria-label` last.

```tsx
// Wrong: an icon-only button with no name, and a title as the only name.
// Failure: the first button reports "button" and nothing else. The second
// reports its name only after a hover, which a keyboard and a touch screen
// never produce.
<button onClick={remove}><Trash2 /></button>
<button onClick={edit} title="Edit order"><Pencil /></button>
```

```tsx
// Correct: real text names the control, and the icon says nothing.
import { Pencil, Trash2 } from "lucide-react";

<button type="button" onClick={remove}>
  <Trash2 aria-hidden="true" className="size-4" />
  <span className="sr-only">Delete order</span>
</button>
<button type="button" onClick={edit} aria-label="Edit order">
  <Pencil aria-hidden="true" className="size-4" />
</button>
```

NEVER use `title` as the only name of a control. A pointer produces it, and a
keyboard and a touch screen do not.

`aria-describedby` adds a description after the name. Use it for a hint and for
an error message. `aria-details` points at a longer explanation elsewhere in
the document.

### The landmarks and the headings

One `<main>` element per page. A `<nav>` element takes an `aria-label` where
the page holds more than one. A `<section>` element becomes a landmark only
where it carries an accessible name.

```tsx
// Correct: src/app/layout.tsx states the language and the landmarks.
export default function RootLayout({ children }: LayoutProps<"/">) {
  return (
    <html lang="en">
      <body>
        <header>
          <nav aria-label="Primary">…</nav>
        </header>
        <main id="main-content">{children}</main>
        <footer>…</footer>
      </body>
    </html>
  );
}
```

One `<h1>` per page, and no level that the page skips. A heading states the
structure of the document, and the class states the appearance.

```tsx
// Wrong: a styled div stands in for a heading.
// Failure: the section has no entry in the heading list. A screen reader user
// who navigates by heading passes over the whole block.
<div className="text-2xl font-bold">Recent orders</div>
```

```tsx
// Correct: the heading carries the level, and the class carries the size.
<h2 className="text-2xl font-bold">Recent orders</h2>
```

A `<table>` element takes a `<caption>`, a `<thead>`, a `<tbody>`, and a
`scope` attribute on every `<th>`. A `<figure>` element takes a
`<figcaption>`. A list of items is a `<ul>` or an `<ol>`, and a set of terms is
a `<dl>`.

### The state that a control reports

| The state | The attribute | The element that carries it |
| --- | --- | --- |
| Open or closed | `aria-expanded` | The control that toggles the region, never the region |
| The region that a control toggles | `aria-controls` | The control, with the `id` of the region |
| The current page in a set of links | `aria-current="page"` | The link, never its wrapper |
| The selected tab or option | `aria-selected` | The tab or the option |
| A checked box or a checked option | `aria-checked` | The element with the checkbox or the radio role |
| A toggle that stays pressed | `aria-pressed` | The button |
| A field that failed validation | `aria-invalid` | The field |
| A field that the form requires | `aria-required` | The field |
| A region that is loading a value | `aria-busy` | The region, never the whole page |

`disabled` and `aria-disabled` are two different decisions.

| The condition | The attribute | What the user gets |
| --- | --- | --- |
| The control is not operable, and the user needs no explanation | `disabled` | The control leaves the tab order, and the browser blocks the event. |
| The control is not operable, and the user must find out why | `aria-disabled="true"` | The control stays in the tab order and reports its state. The handler must refuse the action itself. |

A form that disables its submit button hides the reason for the refusal. Keep
the button operable, and report the errors on the submit.
`references/keyboard-focus-and-live-regions.md` owns that report. A submit that
is already running is the one exception. It is the first row of the table
above: the refusal lasts for one request, and the label of the button states
the reason. `references/form-submission-and-server-errors.md` holds that case.

### The three ways to hide an element

| The intent | The mechanism | The effect |
| --- | --- | --- |
| Hide it from everybody | `hidden`, or a conditional render | It leaves the layout, the tab order, and the accessibility tree. |
| Hide it from a screen reader, and keep it on the screen | `aria-hidden="true"` | It leaves the accessibility tree alone. It stays focusable, which is the defect below. |
| Hide it from the screen, and keep it for a screen reader | `sr-only` | It stays in the accessibility tree and in the tab order. |
| Hide a whole background subtree behind a modal | `inert` | The subtree leaves the tab order and the accessibility tree, and the pointer cannot reach it. |

```tsx
// Wrong: aria-hidden covers a subtree that holds a control.
// Failure: the Tab key reaches the button, and the screen reader reports
// nothing when it arrives. The user has focus on an element that does not
// exist for them.
<div aria-hidden="true">
  <button type="button" onClick={close}>Close</button>
</div>
```

```tsx
// Correct: inert removes the subtree from every input method at once.
<div inert>
  <button type="button" onClick={close}>Close</button>
</div>
```

`role="presentation"` and `role="none"` strip the semantics of one element.
Take them only where a native element carries markup that the pattern does not
need. They never apply to a focusable element.

### The image, the icon, and the SVG

| The kind of image | The rule | The markup |
| --- | --- | --- |
| Decorative | It carries no information that the text does not already carry. | `alt=""` on an image, `aria-hidden="true"` on an inline icon |
| Informative | The alternative text states the information, and not the picture. | `alt="Payment failed"` |
| Functional | The alternative text states the action, and not the picture. | A link or a button whose name comes from the image |
| Complex | The page carries the same data in another form. | A caption, a table, or a text summary beside it |

```tsx
// Wrong: the alternative text repeats the file name.
// Failure: a screen reader reads "chart dot png". The user learns nothing,
// and the information in the image reaches nobody.
<Image src="/chart.png" alt="chart.png" width={640} height={360} />
```

```tsx
// Correct: the decorative image says nothing, and the functional image names
// the action of the control around it.
import Image from "next/image";

<Image src="/pattern.svg" alt="" width={64} height={64} />
<Link href="/orders">
  <Image src="/logo.svg" alt="Orders home" width={32} height={32} />
</Link>
```

An inline SVG that carries information takes `role="img"` and a `<title>`
element as its first child. A decorative inline SVG takes `aria-hidden="true"`.
Every icon of `lucide-react` is decorative by default, so the control around it
must carry the name.

`references/charts-and-visual-encoding.md` owns the text alternative of a
chart. `references/data-table-and-server-driven-state.md` owns the header cells
and the caption of a data table.

### The language of the document and of a passage

```tsx
// Correct: the document states its language, and the passage states its own.
<html lang="en">
…
<p>
  The Persian word for an order is <span lang="fa" dir="rtl">سفارش</span>.
</p>
```

A screen reader chooses its pronunciation rules from `lang`. A missing `lang`
on `<html>` produces the wrong voice for the whole document. A missing `lang`
on a foreign passage produces the wrong voice for that passage.

Set `dir` where the direction of a passage differs from the direction of the
document. Domain 19 `internationalization-and-rtl` owns the locale route and
the direction of the whole document. It is not integrated yet.
`references/layout-and-typography.md` owns the logical property that makes the
layout mirror correctly.

### The label, the hint, and the error on a field

A placeholder is a hint. It disappears when the user types, and it is never a
label.

```tsx
// Wrong: the placeholder is the label.
// Failure: the name of the field disappears at the first keystroke. A screen
// reader reports an unnamed field, and a user who forgets the prompt has no
// way back to it.
<input name="email" placeholder="Email" />
```

```tsx
// Correct: the label names the field, and the hint and the error describe it.
const hintId = "email-hint";
const errorId = "email-error";
const describedBy = [hintId, error ? errorId : undefined]
  .filter((id) => id !== undefined)
  .join(" ");

<label htmlFor="email">
  Email <span aria-hidden="true">*</span>
  <span className="sr-only">(required)</span>
</label>
<input
  id="email"
  name="email"
  type="email"
  autoComplete="email"
  required
  aria-required="true"
  aria-invalid={error !== undefined}
  aria-describedby={describedBy}
/>
<p id={hintId}>We send the receipt to this address.</p>
{error === undefined ? null : <p id={errorId}>{error}</p>}
```

Four rules hold on every field. The label carries `htmlFor`, and the field
carries the matching `id`. The required mark reaches the eye and the screen
reader. The error message is one of the `aria-describedby` targets. The
`aria-invalid` attribute is true only while the error is present.

Group a set of radio buttons or a set of related fields in a `<fieldset>` with
a `<legend>`. The legend names the group, and the label names each option.

The `autocomplete` attribute lets the browser fill a known field. It carries a
value from a fixed token list, and it helps a user who cannot type an address
a second time.

An authentication step must never make the user solve a cognitive test with no
alternative. Allow paste in every one-time-code field and in every password
field.

```tsx
// Wrong: the one-time-code field blocks paste.
// Failure: the user must hold the code in memory and type it. That is a
// cognitive function test, and criterion 3.3.8 forbids it with no
// alternative. A password manager also cannot fill the field.
<input name="otp" onPaste={(event) => event.preventDefault()} />
```

```tsx
// Correct: paste works, and the browser can offer the code it received.
<input name="otp" inputMode="numeric" autoComplete="one-time-code" />
```

A multi-step flow must not ask for the same value twice. Carry the value
forward, or offer the earlier answer as a choice.
`references/multi-step-forms-and-unsaved-work.md` holds the mechanism.

Warn the user before a session ends, and let the user extend it.
`references/session-and-token-lifecycle.md` owns the session that ends, and
this file owns the warning that the user must be able to act on.

`references/form-schema-and-field-binding.md` owns the resolver and the field
array, and `references/multi-step-forms-and-unsaved-work.md` owns the
multi-step mechanics. This file owns the label, the description, and the state
that each field reports. Those files supply the value that
`aria-invalid` and `aria-describedby` carry, and this file states the wiring.

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| A control that the Tab key never reaches | A `div` or a `span` carries the click handler | Tab through the surface, and count the stops | Render a `<button>` |
| A screen reader reports "button" with no name | An icon-only control with no text and no `aria-label` | The axe rule for a button name | Add `sr-only` text, or `aria-label` |
| A name appears only after a hover | `title` is the only name source | Tab to the control, and listen | Add a real name |
| A label that no tool associates with a field | The `htmlFor` value and the `id` value differ | The axe rule for a form label | Correct the two values |
| Focus lands on an element that the screen reader does not report | `aria-hidden` on an ancestor of a control | Tab into the hidden subtree | Replace `aria-hidden` with `inert` |
| A heading list that skips the whole section | A styled `div` stands in for a heading | Read the heading outline of the page | Render the heading element |
| A refusal with no reason | The submit button carries `disabled` | Submit the form with an empty field | Keep the button operable, and report the errors |
| A form that a password manager cannot fill | No `autocomplete` token on the field | Read the field with the manager open | Add the token |

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

shadcn/ui on Base UI supplies a primitive for most APG patterns, and the
primitive carries the roles and the states. Confirm the installed primitive
before you hand-build a control. A hand-built control owns the whole pattern,
and `references/component-styles-and-variants.md` states that the primitive is
the unit of style.

## Verification

```bash
# 1. Find a generic element with a click handler. Read every hit.
rg -n '<(div|span|li|td)[^>]*onClick' -g '*.tsx' src/

# 2. Find a manual role on a control. Read every hit, and check the pattern.
rg -n 'role="(button|link|checkbox|tab|dialog|menu)"' -g '*.tsx' src/

# 3. Find a placeholder, and confirm that a label sits beside each one.
rg -n 'placeholder=' -g '*.tsx' src/

# 4. Find an aria-hidden over a focusable element. This must print nothing.
rg -n 'aria-hidden[^>]*(onClick|href=|tabIndex)' -g '*.tsx' src/

# 5. Find a title attribute on a control. Read every hit.
rg -n '<(button|a)[^>]*title=' -g '*.tsx' src/

# 6. Confirm one h1 and one main per route.
rg -c '<h1' -g '*.tsx' src/app/
rg -n '<main' -g '*.tsx' src/

# 7. Read every image, and decide the alternative text of each one.
rg -n '<Image|<img' -g '*.tsx' src/

# 8. Confirm the language of the document.
rg -n '<html' -g '*.tsx' src/app/

# 9. The lint gate reports the static failures of this file.
pnpm lint

# 10. Read one surface with a screen reader, and confirm the name, the role,
#     and the state of every control on it.
```

## Review checklist

- [ ] Does every control render a native element, rather than a generic
      element with a handler?
- [ ] Does every interactive element carry an accessible name?
- [ ] Is `title` absent as the only name of a control?
- [ ] Does every ARIA attribute map to a requirement of a named APG pattern?
- [ ] Does the page carry one `<h1>`, one `<main>`, and no skipped heading
      level?
- [ ] Does every `<nav>` element carry a name where the page holds more than
      one?
- [ ] Is `aria-hidden` absent from every focusable element and from every
      ancestor of one?
- [ ] Does every image carry a decision — an empty `alt`, an informative
      `alt`, or a functional name?
- [ ] Does `<html>` carry `lang`, and does every foreign passage carry its
      own?
- [ ] Does every field carry a programmatic label that is not a placeholder?
- [ ] Does every hint and every error reach the field through
      `aria-describedby`?
- [ ] Does `aria-invalid` follow the presence of the error, rather than stay
      fixed?
- [ ] Does every group of related fields sit in a `<fieldset>` with a
      `<legend>`?
- [ ] Does every one-time-code field and every password field accept paste?

## Handoffs

- The tab order, the focus indicator, the focus on a route change, the modal,
  and the announcement of a change →
  `references/keyboard-focus-and-live-regions.md`.
- The contrast ratio of a name against its background, the target size of a
  control, and the reflow of a form →
  `references/visual-and-motor-criteria.md`.
- The conformance target, the axe rules, the lint plugin, and the manual pass
  → `references/wcag-conformance-and-verification.md`.
- The shape of the component, its parts, and the prop that renders it as
  another element → `references/component-composition.md`.
- The classes on a part, `cn()`, and the variant API →
  `references/component-styles-and-variants.md`.
- The logical property that mirrors a layout, and the type scale →
  `references/layout-and-typography.md`.
- The session that ends, and the credential behind it →
  `references/session-and-token-lifecycle.md`.
- The gate that refuses a request, and the permission that the UI reflects →
  `references/route-protection-and-permissions.md`.
- The resolver, the field array, and the source of the error value →
  `references/form-schema-and-field-binding.md` and
  `references/form-submission-and-server-errors.md`.
- The step, the guard over unsaved work, and the answer that the flow must not
  request twice → `references/multi-step-forms-and-unsaved-work.md`.
- The text alternative of a chart → `references/charts-and-visual-encoding.md`.
- The header cells and the caption of a data table →
  `references/data-table-and-server-driven-state.md`.
- The bytes of an image, and the video player →
  `references/image-and-video-delivery.md`.
- The words inside a label, a hint, and an error message → domain 15
  `ux-writing-and-content-design`. Not integrated yet.
- The locale route, the message catalog, and the direction of the document →
  domain 19 `internationalization-and-rtl`. Not integrated yet.
