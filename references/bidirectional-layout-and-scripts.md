# Bidirectional layout and scripts

Unicode UAX #9, Tailwind CSS v4.3, Next.js 16.3, React 19.2.6 or later,
TypeScript 5.9. This file owns the direction of a surface and the script inside
it. The subjects are the `dir` attribute of the document, the value whose
direction nobody can predict, and the element that must not mirror. They also
include the icon that flips and the property that no logical rule reaches. The
last subjects are the font for a non-Latin script, the joining behavior of
Arabic, the zero-width non-joiner, and the cut that a truncation makes.

The authoring rule for a logical CSS property is
`references/layout-and-typography.md`. The `lang` attribute, and `dir` on a
passage whose language the author knows, are
`references/semantics-and-accessible-names.md`. The isolate around an argument
inside a message is `references/message-catalog-and-plurals.md`.

## Principle

Direction is two decisions. The document takes one direction, and each run of
text inside it takes its own. One attribute settles the first. The
bidirectional algorithm settles the second, and it needs help where a value
carries no direction of its own.

A mirrored layout is correct, and a mirrored meaning is not. Reading order
mirrors. Time, a clock, a logo, and a number do not.

A script decides the shape of a letter. Arabic joins its letters into a word,
so a property that adds space between two letters destroys the word rather
than spacing it.

A character is not a letter, and a letter is not a grapheme. A cut at a code
unit produces a broken glyph and a lost mark.

Two writing systems spell one sound with two code points. A reader types one of
them, the database holds the other, and a raw comparison finds nothing.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark.

### The document states its direction

```tsx
// Wrong: the attribute sits on the body, and the value is a constant.
// Failure: the html element keeps the default direction, so the scrollbar
// stays on the left and a fixed element positions against the wrong edge. A
// Persian reader reads a mirrored page inside a left-to-right frame.
<html lang="fa">
  <body dir="rtl">{children}</body>
</html>
```

```ts
// Correct: src/i18n/direction.ts derives the direction from the locale.
const RTL_LANGUAGES = new Set(["ar", "fa", "he", "ur", "ps", "sd", "ckb"]);

export function getLangDir(locale: string): "ltr" | "rtl" {
  return RTL_LANGUAGES.has(new Intl.Locale(locale).language) ? "rtl" : "ltr";
}
```

```tsx
// Correct: the root element carries both attributes, from the resolved locale.
<html lang={locale} dir={getLangDir(locale)}>
  <body>{children}</body>
</html>
```

Put `dir` on `<html>`, beside `lang`. The two attributes belong to one element.
`references/semantics-and-accessible-names.md` owns `lang`, and that domain
holds a veto. `references/locale-routing-and-catalogs.md` owns the locale that
this helper reads.

Read the language subtag, and never the whole tag. The tag `fa-IR` and the tag
`fa` both resolve to Persian, and a set that holds the full tags misses one of
them.

### A value of unknown direction needs `dir="auto"`

```tsx
// Wrong: the interface states the direction of a value that a person typed.
// Failure: the interface runs in English, and a reader posts a comment in
// Persian. The comment renders left to right, so the sentence starts at the
// wrong edge and its full stop lands at the left. A file name in Arabic breaks
// the row around it in the same way.
<p>{comment.body}</p>
<span>{file.name}</span>
```

```tsx
// Correct: the element resolves its own direction from its first strong
// character, and the inline value carries an isolate.
<p dir="auto">{comment.body}</p>
<span>
  The file <bdi>{file.name}</bdi> is ready.
</span>
```

Take `dir="auto"` on a block that holds one value from a person. Take `<bdi>`
where that value sits inside a sentence that the product wrote. The element
isolates the run, so the punctuation of the sentence stays at the correct end.

Five values carry no direction that the product can predict: a name, a comment
body, a file name, a search term, and a record title that a user authored.

`references/message-catalog-and-plurals.md` owns the isolate that an ICU
argument receives, which covers a value that reaches the screen through a
message. Take `<bdi>` for a value that reaches the markup outside one.

An input holds the same problem. Take `dir="auto"` on a text input whose
content the product cannot predict, so the caret starts at the correct edge.

### What mirrors, and what must not

| The element | Under `dir="rtl"` | Why |
| --- | --- | --- |
| The reading order, the columns, and the navigation | It mirrors | Reading runs from the right |
| A back arrow, a forward arrow, a chevron, and a breadcrumb separator | It mirrors | Each one points along the reading order |
| An undo icon, a redo icon, an indent icon, and a reply icon | It mirrors | Each one describes a move along the text |
| A progress bar and a stepper | It mirrors | Progress follows the reading order |
| A play button, a fast-forward icon, and a rewind icon | It does not mirror | Playback runs one way in every language |
| A timeline, a chart with a time axis, and a video scrubber | It does not mirror | Time runs one way |
| A clock face and a stopwatch icon | It does not mirror | The hands turn one way |
| A logo and a brand mark | It does not mirror | It is a fixed image of a name |
| A code block, a terminal, and a diff | It does not mirror | The syntax is left to right |
| A number, a phone number, and a version string | It does not mirror | Digits run left to right in every script |
| A checkmark, a search icon, and a settings icon | It does not mirror | Neither one points along an axis |

```tsx
// Wrong: every icon flips with the layout.
// Failure: the play button points backwards in a Persian interface, and the
// reader presses it to rewind. The logo renders as its own mirror image.
<div className="rtl:-scale-x-100">…</div>
```

```tsx
// Correct: the flip sits on the icons that describe a direction.
<ChevronRight className="size-4 rtl:-scale-x-100" aria-hidden="true" />
<Play className="size-4" aria-hidden="true" />
```

A chart with a category axis mirrors with the layout. A chart with a time axis
keeps its direction, and its labels take the locale.
`references/charts-and-visual-encoding.md` owns the encoding, and
`references/locale-formatting-and-calendars.md` owns the axis label.

### The properties that no logical rule reaches

`references/layout-and-typography.md` requires `ms-`, `me-`, `ps-`, `pe-`,
`start-`, `end-`, `border-s`, and `text-start` in every new component. Those
utilities settle the box. Five things stay physical, and each one needs a
decision:

| The property | The failure under `dir="rtl"` | The fix |
| --- | --- | --- |
| `transform: translateX()` | A drawer slides in from the wrong edge, and an exit animation leaves through the wrong one | Flip the sign with an `rtl:` variant, or animate a logical inset |
| `box-shadow` and `text-shadow` | The light appears to come from the other side, which is a physical fact and not a reading one | Keep it, unless the design states a direction of light |
| `background-position` and a linear gradient angle | A gradient that leads the eye now leads it backwards | Flip it with an `rtl:` variant where it carries meaning |
| A scroll position set in code | A carousel starts at the wrong end, and `scrollLeft` is negative in some engines | Read `scrollLeft` against the computed direction, or take `scrollIntoView` |
| A hard-coded `left` or `right` on a portal or a tooltip | The floating element opens off the edge of the screen | Take the placement API of the primitive, which reads the direction |

```tsx
// Wrong: the sheet always slides in from the left.
// Failure: under dir="rtl" the panel is anchored to the right edge, and the
// animation still starts 100 percent to the left. The panel crosses the whole
// screen before it settles.
<div className="-translate-x-full transition-transform data-[open]:translate-x-0" />
```

```tsx
// Correct: the sign follows the direction.
<div className="-translate-x-full rtl:translate-x-full transition-transform data-[open]:translate-x-0" />
```

`references/motion-primitives-and-reduced-motion.md` owns the animation itself
and the reduced-motion override, and
`references/view-transitions-and-animation-libraries.md` owns a shared-element
transition. Both hold under either direction.

The `shadcn` CLI converts a physical utility to its logical equivalent when it
adds a component. Read the generated file, because the conversion covers the
box and not the five rows above.
`references/component-styles-and-variants.md` owns the classes on a part.

### The font for a non-Latin script

```ts
// Wrong: one Latin font serves every locale.
// Failure: the Persian text falls to a system font that the design never
// chose. The metrics differ from the Latin family, so the line height is
// wrong, the baseline shifts, and the layout moves when the fallback resolves.
export const sans = Inter({ subsets: ["latin"], variable: "--font-sans" });
```

```ts
// Correct: src/app/fonts.ts declares a family for each script.
import localFont from "next/font/local";
import { Inter } from "next/font/google";

export const latin = Inter({
  subsets: ["latin"],
  display: "swap",
  variable: "--font-latin",
});

export const arabic = localFont({
  src: [{ path: "./Vazirmatn[wght].woff2", weight: "100 900", style: "normal" }],
  display: "swap",
  variable: "--font-arabic",
  fallback: ["Tahoma", "Arial"],
  adjustFontFallback: false,
});
```

Take a variable font for an Arabic-script family. A family of static weights
costs several files, and each one carries the whole script.

Subset the file before you add it. A font file with no subset holds the Arabic,
Persian, and Urdu ranges with every presentation form. It is far larger than a
Latin file of the same design.
`references/performance-budgets-and-measurement.md` owns the byte budget.

Set `adjustFontFallback: false` for a non-Latin family, and state a `fallback`
family that covers the script. The automatic metric override is computed
against a Latin fallback, and it moves the layout for a script whose metrics
differ. `references/layout-and-typography.md` owns the option table and the
reserved box, and `references/paint-and-interaction-cost.md` owns the layout
shift budget.

Arabic script needs more leading than Latin at the same size. Its ascenders and
its descenders reach further, so a line height tuned for Latin makes the lines
touch. Raise the line height for the Arabic family in the theme.
`references/design-tokens-and-theming.md` owns the token.

A Latin brand name inside a Persian sentence renders in the Latin family. Order
the family list so the Arabic family comes first and the Latin family follows,
and each script then reaches the file that holds it.

### Letter spacing destroys an Arabic word

```tsx
// Wrong: the tracking utility applies to every locale.
// Failure: Arabic script joins its letters. The property inserts a gap at each
// join, so a word breaks into disconnected letter shapes. A reader of Persian
// reads it as damage, and not as a style.
<h1 className="text-3xl font-bold tracking-tight">{t("title")}</h1>
```

```tsx
// Correct: the tracking applies to the Latin script alone.
<h1 className="text-3xl font-bold tracking-tight rtl:tracking-normal">
  {t("title")}
</h1>
```

Set the tracking token to zero for the Arabic-script theme, so no component
needs the variant. A negative tracking is a Latin display convention, and no
Arabic-script design uses it.

The same rule covers `font-stretch`, a synthetic bold, and a synthetic italic.
Arabic script has no italic form, so a synthetic slant is a defect.

### The zero-width non-joiner survives every step

The character U+200C is a letter of written Persian. It separates two letters
inside one word and stops them from joining. The word می‌رود holds one, and
میرود without it is a different spelling.

Four steps delete it, and each one is a defect:

1. A sanitiser that strips every invisible character.
2. A validation rule that rejects a character outside a stated range.
3. A trim or a collapse of whitespace that treats it as a space.
4. A truncation that cuts inside the sequence.

```ts
// Wrong: the rule removes every zero-width character.
// Failure: every Persian compound verb loses its separator, and the stored
// value is a misspelling that the reader cannot correct through the interface.
const clean = value.replace(/[\u200B-\u200F\uFEFF]/g, "");
```

```ts
// Correct: the rule removes the zero-width characters that carry no meaning,
// and it keeps U+200C.
const clean = value.replace(/[\u200B\u200E\u200F\uFEFF]/g, "");
```

`references/untrusted-markup-and-injection.md` owns the sanitiser over markup,
and that domain holds a veto. This rule concerns a plain text value, and it
never relaxes an escape.

### The search normalizes, and the display does not

```ts
// Wrong: the filter compares the raw strings.
// Failure: the record holds علی with the Persian letter ی (U+06CC). The reader
// types علي with the Arabic letter ي (U+064A), which the Arabic keyboard
// produces. The two strings differ, and the search returns nothing.
const hits = people.filter((p) => p.name.includes(term));
```

```ts
// Correct: src/lib/persian.ts folds the forms that a reader treats as one.
const ARABIC_YEH = /ي/g; // ي to ی
const ARABIC_KAF = /ك/g; // ك to ک
const TATWEEL = /\u0640/g; // the joining extension
const HARAKAT = /[\u064B-\u0652]/g; // the vowel marks
const ZWNJ = /\u200C/g; // the zero-width non-joiner

export function foldForSearch(value: string): string {
  return value
    .normalize("NFC")
    .replace(ARABIC_YEH, "ی")
    .replace(ARABIC_KAF, "ک")
    .replace(TATWEEL, "")
    .replace(HARAKAT, "")
    .replace(ZWNJ, "")
    .toLowerCase();
}

const hits = people.filter((p) =>
  foldForSearch(p.name).includes(foldForSearch(term)),
);
```

Fold for the comparison, and never for the value. The stored name and the
rendered name keep every character that the person typed. A fold that reaches
the database rewrites a name that its owner chose.

The fold removes the ZWNJ for the comparison alone, so a reader who omits it
still finds the record.

A server-side search folds on the server, over the same rules. State the need,
and never write the query here. The sibling skill `django-performance-optimizer`
owns the index behind it.

`references/locale-formatting-and-calendars.md` owns `Intl.Collator`, which
gives the order of a language. A collator does not fold ی against ي, because
the two are letters of two languages.

### A cut lands between graphemes

```ts
// Wrong: the truncation counts code units.
// Failure: the cut lands inside a surrogate pair or before a combining mark.
// The last glyph renders as a replacement character, and an emoji becomes two
// broken halves. In Persian the cut can also remove a ZWNJ and join two
// letters that must stay apart.
const short = value.slice(0, 40) + "…";
```

```ts
// Correct: the segmenter counts what a reader counts.
const segmenter = new Intl.Segmenter(locale, { granularity: "grapheme" });

export function truncate(value: string, max: number): string {
  const units = [...segmenter.segment(value)];
  if (units.length <= max) return value;
  return units.slice(0, max).map((unit) => unit.segment).join("") + "…";
}
```

Prefer the CSS answer where a box decides the length. `text-overflow: ellipsis`
cuts at the correct edge under either direction, and it needs no count. Take
the segmenter where a stated count decides, such as a summary field or a
character limit beside an input.

Build the segmenter once, outside the render.

A grapheme count is also the correct count for a character limit that a reader
sees. The `length` of a string counts code units, so it reports 4 for a value
that the reader typed as one emoji.

### Verify both directions

Add a direction toggle to the development build, and read every primary surface
in each direction. The toggle sets `dir` on `<html>`, so it reproduces the
resolved state with no locale switch.

Take a visual snapshot of each primary surface in each direction, and compare
the two. A mirrored logo, a chevron that did not flip, and a panel that slides
from the wrong edge are visible in an image. Each one is invisible in the
markup.

`references/message-catalog-and-plurals.md` owns the pseudo-locale that proves
the length of a translated string. Run that pass and this one together, because
a longer string and a mirrored box break the same control.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry supplied
those facts on 18 August 2026. A cell that holds no date is a package with a
current registry entry on that date. This file does not state an exact release
date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | The `rtl:` and `ltr:` variants of Tailwind v4.3 | They read the `dir` attribute, and they need no plugin and no build step. | Tailwind CSS v4.3 | Current | Active | None |
| Recommend | `Intl.Segmenter` | The platform grapheme boundary. It needs no package. | Platform | — | Platform | — |
| Recommend | `next/font/local` | It self-hosts the script font, and it generates the CSS variable. | Ships with Next 16.3 | — | Active | None |
| Recommend | Vazirmatn, as a variable font | A Persian and Arabic family with a variable weight axis and an open licence. Estedad and Sahel are alternatives of the same class. | Current | Current | Active | None |
| Conditional | `postcss-logical` | Only where a legacy stylesheet holds physical properties that no one can edit by hand. New code needs the logical utilities. | Current | Current | Active | None |
| Reject | `rtlcss` as a second stylesheet | It builds a mirrored copy of the whole stylesheet. Two files then drift, and the logical properties make it unnecessary. | Current | Current | Active | — |
| Reject | `letter-spacing` on Arabic script | It breaks the joins, so a word renders as disconnected letters. | — | — | — | — |
| Reject | A `dir` attribute on `<body>` alone | The root element keeps the default direction, so the scrollbar and a fixed element take the wrong edge. | — | — | — | — |
| Reject | A stored value that a search fold rewrote | The fold serves a comparison. A name belongs to the person who typed it. | — | — | — | — |

`references/layout-and-typography.md` owns the `next/font` option table, and
`references/untrusted-markup-and-injection.md` owns the markup sanitiser. Do
not add a second row for either one.

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| The page mirrors, and the scrollbar stays on the wrong side | `dir` sits on `<body>` | Read the `html` element in the inspector | Put `dir` on `<html>`, beside `lang` |
| A Persian comment starts at the wrong edge inside an English page | The block states no direction | Post a comment in a right-to-left script | Take `dir="auto"` on the block |
| The full stop of a sentence lands at the wrong end | An interpolated value carries no isolate | Render a name in a right-to-left script inside a sentence | Take `<bdi>`, or pass the value as an ICU argument |
| The play button points backwards | A blanket flip covers every icon | Read the media controls under `dir="rtl"` | Flip the directional icons alone |
| The logo renders as its own mirror image | The same blanket flip | Read the header under `dir="rtl"` | Remove the flip from the brand mark |
| A drawer crosses the whole screen before it settles | A `translateX` sign that the direction does not reach | Open the drawer under `dir="rtl"` | Flip the sign with an `rtl:` variant |
| A Persian word renders as disconnected letters | A tracking utility applies to the Arabic script | Read a heading in Persian | Set the tracking to zero for that theme |
| The layout moves when the Persian font arrives | The automatic fallback metrics assume a Latin family | Throttle the network, and watch the first paint | Set `adjustFontFallback: false`, and state a fallback family |
| A Persian compound verb is stored as a misspelling | A rule stripped U+200C | Save a value that holds one, and read it back | Keep U+200C, and strip the other zero-width characters |
| A search for a Persian name returns nothing | The two spellings of one letter differ | Search with an Arabic keyboard for a record typed on a Persian one | Fold both sides before the comparison |
| The last glyph of a truncated string is a replacement character | The cut counts code units | Truncate a string that ends in an emoji or a combining mark | Cut on a grapheme boundary |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Tailwind v4 ships the logical utilities and the `rtl:` variant | A `tailwindcss-rtl` plugin, or an `rtlcss` build step | Delete the plugin, and take the built-in utilities |
| Tailwind v4 reads the direction from the `dir` attribute | A `dir` class on a wrapper, which the variant does not read | Set the attribute on `<html>` |
| `Intl.Segmenter` is on the baseline runtimes | A `split("")` over text, or a `slice` on a user value | Take a segmenter for every count and every cut |
| `Intl.Locale.prototype.getTextInfo` is not on every baseline runtime | A call with no feature test | Take the declared language set until the method reaches the stated baseline |

## Verification

```bash
# 1. Read the installed versions before you write code.
cat package.json | rg '"tailwindcss"|"next"|"react"'

# 2. Confirm that the root element carries both attributes. Read the hit.
rg -n '<html' -g '*.tsx' src/

# 3. Find a dir attribute on the body element. This must print nothing.
rg -n '<body[^>]*dir=' -g '*.tsx' src/

# 4. Find a physical direction utility in new component code. Read every hit.
rg -n 'class[Nn]ame="[^"]*\b(ml-|mr-|pl-|pr-|left-|right-|text-left|text-right)' \
  -g '*.tsx' src/

# 5. Find a blanket mirror on a container. Each hit must name the icons.
rg -n 'rtl:-scale-x-100' -g '*.tsx' src/

# 6. Find a translateX with no rtl variant beside it. Read every hit.
rg -n 'translate-x-' -g '*.tsx' src/ | rg -v 'rtl:'

# 7. Find a rule that strips U+200C. Read every hit.
rg -n '200B-\\?u200F|u200C' -g '*.ts' -g '*.tsx' src/

# 8. Find a truncation that counts code units. Read every hit.
rg -n '\.slice\(0,\s*\d+\)|\.substring\(0,\s*\d+\)' -g '*.ts' -g '*.tsx' src/

# 9. Find a tracking utility on a heading. Each hit needs the rtl override or
#    a zero token.
rg -n 'tracking-(tight|tighter|wide|wider|widest)' -g '*.tsx' src/

# 10. Find a component that renders a user value with no direction decision.
rg --files-without-match 'dir="auto"|<bdi' \
  -g 'src/features/**/*Comment*.tsx' src/

# 11. Confirm one font declaration for each script, and read its options.
rg -n -A8 'localFont\(|from "next/font' -g '*.ts' src/app/fonts.ts

# 12. Set dir="rtl" on <html>, and read every primary surface. No layout
#     breaks, every directional icon mirrors, and no logo and no media
#     control mirrors.

# 13. Take a snapshot of each primary surface in each direction, and compare
#     the two images.

# 14. Type a Persian compound verb into every text field, save it, and read it
#     back. The separator survives.

# 15. Search for one record with each spelling of the same letter. Both
#     searches find it.
```

## Review checklist

- [ ] Does `<html>` carry both `lang` and `dir`, from the resolved locale?
- [ ] Does the direction helper read the language subtag rather than the full
      tag?
- [ ] Does every block that holds a value from a person carry `dir="auto"`?
- [ ] Does every user value inside a product sentence sit in a `<bdi>` element,
      or reach it as an ICU argument?
- [ ] Does the mirror apply to the directional icons alone?
- [ ] Does no logo, no media control, no clock, and no time axis mirror?
- [ ] Does every `translateX` that moves a panel carry a direction variant?
- [ ] Does every floating element take the placement API of its primitive?
- [ ] Does a font declaration exist for each script that the product renders?
- [ ] Is the non-Latin font subsetted, and does it state a fallback family?
- [ ] Does the non-Latin family set `adjustFontFallback` to `false`?
- [ ] Does the Arabic-script theme raise the line height?
- [ ] Is the tracking zero for the Arabic-script theme?
- [ ] Does every text rule keep U+200C?
- [ ] Does the search fold both sides, and does the stored value keep every
      character?
- [ ] Does every truncation cut on a grapheme boundary, or leave the cut to
      CSS?
- [ ] Is every segmenter built outside the render?
- [ ] Does each primary surface hold its layout under `dir="rtl"`?

## Handoffs

- The logical utilities, the container query, the type scale, the reserved box,
  and the `next/font` option table → `references/layout-and-typography.md`.
- The `lang` attribute, and `dir` on a passage whose language the author knows
  → `references/semantics-and-accessible-names.md`. That domain holds a veto.
- The isolate around an ICU argument, the pseudo-locale, and the plural rule →
  `references/message-catalog-and-plurals.md`.
- The locale that the direction helper reads, and the root layout that carries
  both attributes → `references/locale-routing-and-catalogs.md`.
- `Intl.Collator`, the calendar, and the digits that a reader types →
  `references/locale-formatting-and-calendars.md`.
- The tokens behind a line height and a tracking value →
  `references/design-tokens-and-theming.md`. The classes on a part, and the
  variant API → `references/component-styles-and-variants.md`.
- The animation that moves a panel, and the reduced-motion override →
  `references/motion-primitives-and-reduced-motion.md` and
  `references/view-transitions-and-animation-libraries.md`. The drag and the
  scroll → `references/gesture-and-scroll-interaction.md`.
- The sanitiser over markup → `references/untrusted-markup-and-injection.md`.
  That domain holds a veto.
- The byte budget over a font file →
  `references/performance-budgets-and-measurement.md`. The layout shift that a
  late font produces → `references/paint-and-interaction-cost.md`.
- The encoding of a chart, and its text alternative →
  `references/charts-and-visual-encoding.md`. The markup of a data table →
  `references/data-table-and-server-driven-state.md`.
- The schema over a text field, and the control that binds to it →
  `references/form-schema-and-field-binding.md`.
- The snapshot test in each direction →
  `references/end-to-end-journeys-and-flake-control.md`.
- The index behind a folded search on the server → the sibling skill
  `django-performance-optimizer`.
