# Interface copy and voice

Next.js 16.3, React 19.2.6 or later, TypeScript 5.9, `next-intl` 4.13.6, WCAG
2.2 Level AA. This file owns the words that a person reads on a control and
beside it. The subjects are the outcome that a button names, the one verb that
one act keeps, and the confirmation before a destructive act. They also include
the accessible name that holds the visible label, the hint that arrives before
the error, and the link that carries a destination. The last subjects are the
case of a label and the pattern that takes a decision from the reader.

The message after a failure and the copy of an empty view are
`references/error-and-empty-state-copy.md`. The key, the count, and the second
locale are `references/message-catalog-and-plurals.md`.

## Principle

A person reads the interface to decide what to do next. A word that names no
outcome leaves that decision unmade.

One act carries one name. A product that calls one act "Save" on one screen and
"Apply" on another asks the reader to hold two models of one thing.

A control that destroys work owes the reader the scope of the loss before the
press. The reader cannot undo a press that nobody explained.

The visible text of a control is also its handle. A person who speaks to the
computer says the word on the screen, so the name in the accessibility tree
must hold that word.

A constraint that only a failure teaches is a constraint that the product hid.
The hint costs one line, and the rejected submit costs the whole form.

A pattern that takes a decision from the reader is a defect, whatever revenue
it produces.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark.

### Every control names its outcome

```tsx
// Wrong: the label names the mechanism, and not the result.
// Failure: a reader on speech input says "click Save changes", and the browser
// finds no control with that name. The press is unreachable by voice. The
// message after it reads "Success", so no word ties the press to the result.
<button type="submit">Submit</button>
```

```tsx
// Correct: the label is a verb and an object, and the result repeats the verb.
import { useTranslations } from "next-intl";

export function SaveProfileButton() {
  const t = useTranslations("profile");
  // profile.save = "Save changes"
  // profile.saved = "Changes saved"
  return <button type="submit">{t("save")}</button>;
}
```

Write a verb and an object. The object drops only where the verb alone names
the outcome, such as `Print`.

| The word | What it fails to state | The replacement |
| --- | --- | --- |
| `Submit` | Which act the press performs | The verb and the object of that act |
| `OK` | Which of the two answers the reader took | The verb of the act |
| `Yes` and `No` | Which act the reader confirmed | The verb of the act, and `Cancel` |
| `Click here` | Where the link goes | The name of the destination |

### One verb for one act

| The surface | What it holds |
| --- | --- |
| The control | The verb and the object — `Publish article` |
| The confirmation | The same verb, the consequence, and the scope |
| The result message | The past form of the same verb — `Article published` |
| The state on the screen | The same word — a `Published` mark on the row |

Hold one term for one concept. A product that says "workspace" in the
navigation and "project" in the settings has two names for one thing, and the
reader must learn both.

Record each term and each verb in a glossary. Review every new string against
that glossary before the merge. `references/message-catalog-and-plurals.md`
holds the command that extracts every string into one flat list, which is the
input to that review.

### The destructive confirmation

```tsx
// Wrong: the question carries the risk, and the controls carry nothing.
// Failure: the reader reads two words with no scope. A reader who returns to
// the tab after an interruption cannot tell what "Yes" deletes, and the act
// has no undo.
<div role="alertdialog">
  <h2>Are you sure?</h2>
  <button type="button" onClick={confirm}>Yes</button>
  <button type="button" onClick={close}>No</button>
</div>
```

```tsx
// Correct: the title states the decision, the body states the scope, and the
// confirm control repeats the destructive verb.
<div role="alertdialog" aria-labelledby="delete-title" aria-describedby="delete-body">
  <h2 id="delete-title">{t("delete.title", { count })}</h2>
  <p id="delete-body">{t("delete.consequence", { count })}</p>
  <button type="button" onClick={confirm}>{t("delete.confirm", { count })}</button>
  <button type="button" onClick={close}>{t("delete.cancel")}</button>
</div>
// delete.title = "Delete {count, plural, one {# file} other {# files}}?"
// delete.consequence =
//   "{count, plural, one {The file} other {The files}} cannot be restored."
// delete.confirm = "Delete {count, plural, one {# file} other {# files}}"
// delete.cancel = "Keep the files"
```

CAUTION: an act with no undo needs its scope in the body of the confirmation.
State the count and the object. `Delete 3 files` names both, and
`Delete selected` names neither.

`references/keyboard-focus-and-live-regions.md` owns the focus inside the
dialog and the trap around it. `references/component-composition.md` owns the
primitive. This file owns the three strings.

### The accessible name holds the visible label

```tsx
// Wrong: the visible word and the accessible name disagree.
// Failure: a reader on Voice Control or Dragon says "click Delete", and the
// browser finds no control named Delete. The control is unreachable by speech,
// and the reader has no way to learn the hidden name.
<button aria-label="Remove item">Delete</button>
```

```tsx
// Correct: the accessible name starts with the visible word, and adds the
// scope that the row supplies.
<button aria-label="Delete invoice 4021">Delete</button>
```

Criterion 2.5.3 is Level A. The accessible name contains the text that the
control presents visually. Put that text at the start of the name, which is the
documented best practice.

An `aria-label` that states the same thing in other words is a defect, however
clear the other words are.

A control with no visible text is outside this criterion. Give it a name that
holds the word a speech reader says for it. Where a visible tooltip supplies
that word, the name must hold the tooltip word without a change.

`references/semantics-and-accessible-names.md` owns the name computation and
the order of its sources. That domain holds a veto. This file owns the words in
the name.

### The hint arrives before the error

```tsx
// Wrong: the rule lives in a title attribute.
// Failure: a keyboard reader and a touch reader never produce a title, so both
// learn the rule only from a rejected submit. A pointer reader produces it
// after a pause, and it disappears at the first keystroke.
<input name="password" type="password" title="Must be at least 12 characters" />
```

```tsx
// Correct: the rule is persistent text, and the field points at it.
<label htmlFor="password">{t("password.label")}</label>
<input
  id="password"
  name="password"
  type="password"
  aria-describedby="password-hint"
/>
<p id="password-hint">{t("password.hint")}</p>
// password.label = "Password"
// password.hint = "Use at least 12 characters."
```

State a format constraint before the submit. The hint prevents the error, and
the error only reports it.

Drop the hint where the field needs none. A `First name` field with a hint
costs the reader a line and teaches nothing.

A placeholder holds an example, and never a label and never an instruction. It
disappears at the first keystroke, so the reader loses it exactly where the
answer is half typed.

Where the product can compute a correction, put it in the message. Criterion
3.3.3 asks for that suggestion. A date in the future takes "Enter a date in the
past", and a free-text name field takes no suggestion.

`references/semantics-and-accessible-names.md` owns the label wiring, and it
states that a placeholder is never a label.
`references/keyboard-focus-and-live-regions.md` owns the three conditions that
content on hover or on focus must meet. This file owns the decision before
them: essential information is persistent text, and never a hover.

### Sentence case, and the length that a control holds

Write a label in sentence case. A capital letter on each word slows the reader,
and it carries no information.

Leave the trailing period off a label, a control, and a heading. Keep it on a
sentence in a hint and in a body paragraph.

A translated label is longer than the English one. Where a translated label
overflows its control, shorten the object and keep the verb.
`references/message-catalog-and-plurals.md` owns the pseudo-locale that proves
the layout, and `references/layout-and-typography.md` owns the box.

### The link that carries a destination

```tsx
// Wrong: the link text names no destination.
// Failure: a reader who lists the links of the page reads "click here" three
// times. That list is the whole context, so the reader cannot choose one.
// Criterion 2.4.4 fails.
<p>To read the amounts, <a href="/invoices/4021">click here</a>.</p>
```

```tsx
// Correct: the link text names the destination, and the sentence around it
// adds nothing that the link needs.
<a href="/invoices/4021">{t("invoice.view")}</a>
// invoice.view = "View invoice 4021"
```

Criterion 2.4.4 is Level A. `Read more` and `Learn more` repeat across a page,
and they fail in the same way.

### No instruction rests on one sense

```tsx
// Wrong: the instruction names a color and a position.
// Failure: a reader on a screen reader hears no color and no position. A
// reader with a color vision deficiency sees no green. A narrow screen moves
// the control, so "on the right" is also wrong for a sighted reader.
<p>Press the green button on the right to publish.</p>
```

```tsx
// Correct: the instruction names the control by its label.
<p>{t("publish.help")}</p>
// publish.help = "Select Publish article to make the page public."
```

Criteria 1.3.3 and 1.4.1 forbid an instruction that rests on shape, size,
position, sound, or color alone. `references/visual-and-motor-criteria.md` owns
the criterion verdict and the color pair that carries meaning. That domain
holds a veto. This file owns the sentence.

### The pattern that takes a decision from the reader

| The pattern | What it does | The replacement |
| --- | --- | --- |
| Confirmshaming | The decline control shames the reader | A neutral verb on both controls |
| A pre-checked consent | The reader agrees by doing nothing | An unchecked control, and an explicit press |
| A false deadline | A timer that resets on each visit | The real deadline, or none |
| A disguised advertisement | An advertisement in the shape of a control | A label that names the advertiser |
| A hidden cost | A charge that appears at the last step | The whole total at the first step |

Ship every consent control unchecked. A pre-checked control records an
agreement that nobody made.

Domain 23 `analytics-privacy-and-consent` owns the lawful basis of a consent
string and the mechanism behind it. It is not integrated yet. This file owns
the plain language and the five rows above.

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| A speech reader cannot press a control | The accessible name does not hold the visible word | Read the name in the accessibility panel beside the visible text | Start the name with the visible word |
| The reader cannot tell what a confirmation deletes | The title asks a question, and the controls say `Yes` and `No` | Read the dialog with the rest of the page covered | State the scope in the body, and the verb on the control |
| The reader repeats a rejected submit | The format rule lives in the error alone | Complete the form with no prior knowledge of it | Move the rule into a persistent hint |
| A link list carries no information | The link text is `click here` | Read the link list of the page with a screen reader | Name the destination in the link text |
| A label overflows its control in one locale | The translated string is longer than the box | Render the pseudo-locale | Shorten the object, and keep the verb |
| Two screens name one act differently | No glossary holds the verb | Extract every string, and read the flat list | Take one verb, and correct the others |
| The reader agrees to a term with no press | The consent control ships checked | Read the default `checked` value of each consent input | Ship it unchecked |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| React 19 renamed `useFormState` to `useActionState`, so one state value holds the result and the error | `rg -n 'useFormState\b' src/` reports a hit | `references/suspense-and-actions.md` owns the hooks. The copy moves out of a prop and into that state. |
| `useFormStatus` supplies the pending label inside the submit control | A `pending` prop that a form passes down to its control | Read the status inside the control, and hold both labels in the catalog |

## Verification

```bash
# 1. Find a banned control label. Each hit is a defect.
rg -n -i '>[[:space:]]*(Submit|OK|Yes|No|Click here)[[:space:]]*<' -g '*.tsx' src/

# 2. Find a confirmation that asks a question and names no act.
rg -n -i 'are you sure' -g '*.tsx' -g '*.json' src/

# 3. Find an aria-label beside visible text. Read every hit, and confirm that
#    the name starts with the visible word.
rg -n -A2 'aria-label=' -g '*.tsx' src/

# 4. Find a title attribute on a control. Each hit moves into persistent text.
rg -n '<(button|a|input|select|textarea)[^>]* title=' -g '*.tsx' src/

# 5. Find link text that names no destination.
rg -n -i '>[[:space:]]*(click here|read more|learn more|here)[[:space:]]*<' \
  -g '*.tsx' src/

# 6. Find an instruction that rests on a color, a shape, or a position. Run it
#    over the message files of the project.
rg -n -i '\b(green|red|blue|left|right|above|below|round|square)\b' \
  -g '*.json' src/

# 7. Find a consent control that ships checked. Read every hit.
rg -n 'defaultChecked|checked=\{true\}' -g '*.tsx' src/

# 8. Read the flat list of strings, and confirm one verb for one act.
#    references/message-catalog-and-plurals.md holds the extract command.

# 9. Read one surface with the screen covered. Every control states its outcome
#    from its name alone.
```

## Review checklist

- [ ] Does every control name its outcome with a verb and an object?
- [ ] Are `Submit`, `OK`, `Yes`, `No`, and `Click here` absent from every
      control?
- [ ] Does one act keep one verb across the control, the confirmation, the
      result, and the state on the screen?
- [ ] Does every destructive confirmation state the consequence and the scope?
- [ ] Does the confirm control of a destructive act repeat the destructive verb?
- [ ] Does every accessible name hold the visible label of its control, at the
      start of the name?
- [ ] Does every format constraint reach the reader before the submit?
- [ ] Is every hint persistent text, rather than a `title` attribute or a hover?
- [ ] Does every placeholder hold an example, and never a label and never an
      instruction?
- [ ] Is every label sentence case, with no trailing period?
- [ ] Does every link text name its destination with no sentence around it?
- [ ] Does every instruction name a control, rather than a color, a shape, or a
      position?
- [ ] Does one glossary hold the term for each concept, with no synonym beside
      it?
- [ ] Is every consent control unchecked until the reader presses it?
- [ ] Are confirmshaming, a false deadline, and a disguised advertisement
      absent?

## Handoffs

- The name computation, the label wiring, and the alternative text of an image
  → `references/semantics-and-accessible-names.md`. That domain holds a veto.
- The three conditions of content on hover or on focus, the announcer, and the
  error summary → `references/keyboard-focus-and-live-regions.md`. That domain
  holds a veto.
- The criterion verdict, the color pair that carries meaning, and the target
  size → `references/visual-and-motor-criteria.md`. That domain holds a veto.
- The message after a failure, the empty view, and the recovery control →
  `references/error-and-empty-state-copy.md`.
- The key, the ICU body, the count, and the pseudo-locale →
  `references/message-catalog-and-plurals.md`.
- The primitive behind a dialog, its parts, and its `render` prop →
  `references/component-composition.md`.
- The classes on a control, and the variant that a state selects →
  `references/component-styles-and-variants.md`.
- The box that holds a label, and the type scale over it →
  `references/layout-and-typography.md`.
- The resolver, the field array, and the moment that validation runs →
  `references/form-schema-and-field-binding.md`.
- `useActionState`, `useFormStatus`, and the pending state of a submit →
  `references/suspense-and-actions.md`.
- The locale route, the direction of the document, and the file that holds the
  catalog → domain 19 `internationalization-and-rtl`. Not integrated yet.
- The lawful basis of a consent string, and the consent mechanism → domain 23
  `analytics-privacy-and-consent`. Not integrated yet.
