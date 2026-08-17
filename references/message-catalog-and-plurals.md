# Message catalog and plurals

`next-intl` 4.13.6, ICU MessageFormat, the Unicode CLDR plural rules, Unicode
UAX #9, TypeScript 5.9, React 19.2.6 or later. This file owns the string as
data. The subjects are the key that holds a string, the ICU body behind that
key, and the count that a plural rule reads. They also include the list that a
formatter joins, and the user value that needs an isolate. The last subjects are
the pseudo-locale that proves the layout, and the tools that audit the whole
set.

The words themselves are `references/interface-copy-and-voice.md` and
`references/error-and-empty-state-copy.md`.

## Principle

A string in the markup is a string that one language holds. A string behind a
key is data, and a second language supplies its own.

A sentence built from parts is a sentence in one grammar. The parts take a
different order in a different language, so the join is the defect and the
translation cannot repair it.

A count changes the noun beside it. English has two forms, and other languages
have as many as six. A number placed next to a fixed noun is wrong in most of
them.

Text runs in two directions. A name in a right-to-left script inside a
left-to-right sentence moves the punctuation around it, unless an isolate holds
it.

A translated string is longer than the English string that the design used. A
layout that fits the shortest language proves nothing.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark.

### No user-facing string sits in the markup

```tsx
// Wrong: the words sit in the markup.
// Failure: the string reaches no translator and no review. The first locale
// after English needs an edit in every component that holds one, and a
// glossary audit has nothing to read.
<button type="submit">Save changes</button>
```

```tsx
// Correct: the component holds the key, and the catalog holds the words.
import { useTranslations } from "next-intl";

export function SaveButton() {
  const t = useTranslations("profile");
  return <button type="submit">{t("save")}</button>;
}
```

Name a key for the place that reads it and for the thing that it says.
`profile.save` reads well, and `label1` does not.

Two exceptions hold. `global-error.tsx` runs with no provider above it, and its
copy is literal for that reason. A test fixture and a component story are not
the product. `references/error-and-empty-state-copy.md` owns the first
exception.

### One key holds one whole sentence

```tsx
// Wrong: a sentence built from two keys and a value.
// Failure: German moves the verb to the end, and a right-to-left language
// reverses the order of the parts. The join fixes an English word order that
// no translation can undo.
<p>{t("deleted")} {name} {t("fromProject")}</p>
```

```tsx
// Correct: one key holds the sentence, and the value is an argument.
<p>{t("deletedFromProject", { name })}</p>
// deletedFromProject = "{name} is deleted from the project."
```

NEVER build a sentence from two keys and a template string. The order of the
parts belongs to the grammar of each language.

### A count takes an ICU plural

```tsx
// Wrong: the count sits beside a fixed noun.
// Failure: the English reads "1 items". Arabic needs six forms of the noun,
// and this sentence holds one, so the noun is wrong for most counts.
<p>{count + " items selected"}</p>
```

```tsx
// Correct: the message holds the rule, and the count is an argument.
<p>{t("selection", { count })}</p>
// selection = "{count, plural,
//   =0 {No items are selected}
//   one {# item is selected}
//   other {# items are selected}}"
```

CLDR defines six plural categories. English uses two of them, and Arabic uses
all six.

| The category | The counts at which Arabic uses it |
| --- | --- |
| `zero` | 0 |
| `one` | 1 |
| `two` | 2 |
| `few` | 3 to 10 |
| `many` | 11 to 99 |
| `other` | 100 and above |

A message that holds `one` and `other` alone is correct for English and wrong
for a language that needs more. Read the categories that each target locale
needs from CLDR, and write a branch for each one.

The `=0` branch is an exact match on the number, and it is not the `zero`
category. Take `=0` where the empty case needs a different sentence, such as
"No items are selected".

The `#` token renders the count with the number format of the locale. Take `#`
inside a plural branch, and never the raw argument beside it.

Take `selectordinal` for a rank, and `select` for a fixed set of values such as
a status.

### A list of values takes `Intl.ListFormat`

```ts
// Wrong: the join holds an English separator and an English conjunction.
// Failure: the separator and the word before the last item differ by locale.
// The output carries "and" inside a sentence in another language.
const label = names.slice(0, -1).join(", ") + " and " + names.at(-1);
```

```ts
// Correct: the platform formatter holds the rule for each locale.
const listFormat = new Intl.ListFormat(locale, {
  style: "long",
  type: "conjunction",
});
const label = listFormat.format(names);
```

Build the formatter once, outside the render.
`references/cell-formatting-and-export.md` states that rule for
`Intl.NumberFormat` and `Intl.DateTimeFormat`, and it holds here.

Take `type: "disjunction"` for a list that ends in "or". Take `type: "unit"`
for a list with no word before the last item.

### A user value inside a sentence needs an isolate

```tsx
// Wrong: the value reaches the sentence through a template string.
// Failure: a name in a right-to-left script moves the punctuation of the
// left-to-right sentence. The full stop lands at the wrong end, and the
// sentence reads as broken.
<p>{`The file ${fileName} is deleted.`}</p>
```

```tsx
// Correct: the ICU formatter isolates each argument.
<p>{t("deletedFile", { fileName })}</p>
// deletedFile = "The file {fileName} is deleted."
```

An ICU formatter wraps each argument in the Unicode isolate characters U+2068
and U+2069. The isolate stops the direction of the argument from reaching the
text around it. Unicode UAX #9 defines that behavior.

Where a value reaches the markup outside a message, wrap it by hand in the same
two characters. A name, a file name, a search term, and a comment are each a
value whose direction the product cannot predict.

### The pseudo-locale proves the layout

Render the interface in a pseudo-locale before a translator writes one word. A
pseudo-locale expands each string by about 40 percent, and it marks each
character.

Two failures appear at once. A control that clips its label fails the layout,
and a string that stays in plain English never reached the catalog.

Take a visual snapshot of each primary surface in that locale. Shorten the
object of a label that overflows, and keep the verb.
`references/interface-copy-and-voice.md` owns the shortened label, and
`references/layout-and-typography.md` owns the box.

### The tools that audit the whole set

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `next-intl` 4.13.6 | The ICU renderer on this stack. It reads `plural`, `select`, and `selectordinal`, and it isolates each argument. Add no second ICU runtime. | 4.13.6 | 10 Aug 2026 | Active, on a weekly cadence | None |
| Recommend | `@formatjs/cli` 6.16.16 | It extracts every message into one flat file, which is the input to a glossary review. | 6.16.16 | Aug 2026 | Active | None |
| Recommend | `eslint-plugin-formatjs` 6.4.20 | It reports an ICU syntax error and a missing message id at lint time. | 6.4.20 | Aug 2026 | Active | None |
| Recommend | `Intl.ListFormat` | The platform formatter for a list of values. It needs no package. | Platform | — | Platform | — |
| Conditional | `pseudo-localization` 3.1.1 | The pseudo-locale generator. It is the common choice, and its release cadence is slow. | 3.1.1 | Current | Slow, and stable | None |
| Conditional | `vale` 3.17.1 | Only where CI reviews the copy as prose. It holds the banned words, the case rule, and the house glossary. | 3.17.1 | Current | Active | None |
| Conditional | `cspell` 10.0.1 | Only for a spell check over the catalog strings. | 10.0.1 | Current | Active | None |
| Conditional | `eslint-plugin-i18next` 6.1.3 | Only where the project stands on i18next. It fails the build on a string in the markup. | 6.1.3 | Current | Active | None |
| Conditional | `@lingui/core` 6.6.0 | Only where the project already holds Lingui catalogs. | 6.6.0 | Current | Active | None |
| Audit only | `i18next` with `react-i18next` | Alive in a project that standardised on it. This stack takes `next-intl`. | 26.3.6, and 17.0.11 | Current | Active | None |
| Audit only | `textlint` 15.7 | An alternative to `vale`. Keep it where it is already wired. | 15.7 | Current | Active | None |
| Reject | `write-good` | The last release is 2020, and it holds no rule config that CI can read. | 1.0.8 | 2020 | Stopped | — |
| Reject | `alex` | Dormant since 2023. A `vale` style covers the same check. | 11.0.1 | 2023 | Stopped | — |
| Reject | A count joined to a noun | It is wrong in every language that needs more than two forms. | — | — | — | — |

`references/wcag-conformance-and-verification.md` owns
`eslint-plugin-jsx-a11y`, and that domain requires it. Do not add a second row
for it here.

Domain 19 `internationalization-and-rtl` owns the file that holds the catalog,
and the route that carries the locale. It also owns the provider that loads a
message, and the layout that mirrors. It is not integrated yet. This file owns
the key, the body of the message, and the rule that no string sits in the
markup.

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| The first locale needs an edit in every component | The strings sit in the markup | Read the flat list of extracted strings against the screen | Move each string behind a key |
| The English reads "1 items" | The count joins a fixed noun | Render the counts 0, 1, and 2 | Take an ICU `plural` |
| The noun is wrong in one locale | The message holds `one` and `other` alone | Render the counts 0, 2, 5, and 11 in that locale | Add the categories that CLDR states for it |
| A sentence reads as broken in one locale | Two keys and a template string build it | Read the extracted list for a key that holds half a sentence | Hold the sentence in one key, with arguments |
| A list carries an English conjunction | A manual join builds it | Render the list in a second locale | Take `Intl.ListFormat` |
| The full stop lands at the wrong end | A user value reaches the sentence with no isolate | Render a name in a right-to-left script inside the sentence | Pass the value as an ICU argument |
| A control clips its label in one locale | The design measured the English string | Render the pseudo-locale | Shorten the object, and keep the verb |
| The root boundary shows an untranslated screen | `global-error.tsx` lost the provider | Trigger a throw in the root layout | Write literal copy in that one file |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| `next-intl` 4.13.6 renders an ICU message with no extra runtime | A second ICU library beside it in `package.json` | Delete the second runtime, and move the messages into the catalog |
| `Intl.MessageFormat` is a TC39 proposal, and it is not in the runtime baseline | A message call against a platform API that the installed Node and the installed browsers do not carry | Stay on the library parser until the proposal reaches the stated baseline |

## Verification

```bash
# 1. Find text in the markup that no key holds. Read every hit.
rg -n '>[[:space:]]*[A-Za-z][A-Za-z ,.-]{3,}[[:space:]]*<' -g '*.tsx' src/

# 2. Find a count joined to a noun. Each hit takes an ICU plural.
rg -n 'count[[:space:]]*\+|\+[[:space:]]*.[[:space:]]?(item|file|row|result)' \
  -g '*.tsx' src/

# 3. Find a sentence built from two keys. Read every hit.
rg -n '\{t\([^)]+\)\}[^<{]*\{t\(' -g '*.tsx' src/

# 4. Find a manual list join. Each hit takes Intl.ListFormat.
rg -n '\.join\(., .\)' -g '*.ts' -g '*.tsx' src/

# 5. Find a message that carries a count argument with no plural rule.
rg -n '\{count[^}]*\}' -g '*.json' src/ | rg -v 'plural'

# 6. Find an Intl formatter built inside a component. Read every hit.
rg -n 'new Intl\.' -g '*.tsx' src/

# 7. Extract every string into one flat list. Read the list for two verbs that
#    name one act.
npx @formatjs/cli extract 'src/**/*.{ts,tsx}' --out-file messages.extracted.json

# 8. Confirm that the root boundary holds no message hook. This prints the file.
rg --files-without-match 'useTranslations' -g 'global-error.tsx' src/

# 9. Render each primary surface in the pseudo-locale. No control clips its
#    label, and no string stays in plain English.

# 10. Render one sentence with a name in a right-to-left script. The
#     punctuation stays at the correct end of the sentence.

# 11. Render the counts 0, 1, 2, 5, and 11 in every target locale. The noun
#     beside each count is correct.
```

## Review checklist

- [ ] Does every user-facing string sit behind a catalog key?
- [ ] Does `global-error.tsx` hold the only literal copy in the product?
- [ ] Does one key hold each whole sentence, with no join across two keys?
- [ ] Does every interpolated count take an ICU `plural`?
- [ ] Does each plural message carry every category that the target locales
      need?
- [ ] Does an empty case take an exact `=0` branch where its sentence differs?
- [ ] Does every plural branch render the count with `#`?
- [ ] Does every list of values pass through `Intl.ListFormat`?
- [ ] Is every `Intl` formatter built outside the render?
- [ ] Does every interpolated user value reach the sentence as an ICU argument?
- [ ] Does a value outside a message carry the two isolate characters?
- [ ] Does a pseudo-locale at about 40 percent extra length exist for the
      project?
- [ ] Does every primary surface hold its layout in that pseudo-locale?
- [ ] Does each key name the place that reads it and the thing that it says?
- [ ] Does the lint gate report an ICU syntax error?

## Handoffs

- The words in each message, the voice, and the shortened label →
  `references/interface-copy-and-voice.md`.
- The message after a failure, and the literal copy of the root boundary →
  `references/error-and-empty-state-copy.md`.
- `Intl.NumberFormat`, `Intl.DateTimeFormat`, and the formatter built once →
  `references/cell-formatting-and-export.md`.
- The box that holds a label, and the type scale over it →
  `references/layout-and-typography.md`.
- The `lang` attribute and the `dir` attribute of a passage →
  `references/semantics-and-accessible-names.md`. That domain holds a veto.
- The lint config array, and the plugin that runs inside it →
  `references/lint-format-and-scripts.md`.
- The `eslint-plugin-jsx-a11y` rule set and the accessibility gate →
  `references/wcag-conformance-and-verification.md`. That domain holds a veto.
- The file that holds the catalog, the locale route, and the layout that mirrors
  → domain 19 `internationalization-and-rtl`. Not integrated yet.
- The test that renders each locale, and the snapshot over the pseudo-locale →
  domain 20 `testing-and-quality`. Not integrated yet.
