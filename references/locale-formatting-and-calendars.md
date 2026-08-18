# Locale formatting and calendars

The `Intl` API of ECMA-402, `next-intl` 4.13.6, Next.js 16.3, React 19.2.6 or
later, TypeScript 5.9, against a Django and DRF backend. This file owns the
value that changes its form with the locale. The subjects are the instant that
a timestamp carries, the zone that renders it, and the relative time over it.
They also include the numbering system, the calendar, and the currency that a
locale selects. The last subjects are the collation behind a sort, the digits
that a reader types, and the display name of a language or a region.

The formatter inside a cell, on an axis, and in an export file is
`references/cell-formatting-and-export.md`. The message that holds a formatted
value is `references/message-catalog-and-plurals.md`. The locale that this
file reads is `references/locale-routing-and-catalogs.md`.

## Principle

A timestamp is one instant. A rendered time is that instant in one zone, and
the zone belongs to the reader.

A date string with no zone is not an instant. Two machines read it as two
different moments, and each machine believes its own answer.

A number and a date have no universal written form. The form is data of the
locale. No product holds that data, and the platform does.

A calendar is not a format. A Persian reader reads a different year, a
different month name, and a different month length. A translated month name on
a Gregorian date does not produce a Persian date.

A script writes its own digits. What a reader reads and what a program stores
are two decisions, and only the first one belongs to the locale.

An order over words is a property of a language. The code unit order of a
string is a property of a table, and it is not an order that a reader accepts.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark.

### Store the instant, and render it in a zone

```tsx
// Wrong: the code builds a Date from a string that carries no zone, and it
// renders with no zone.
// Failure: DRF sends "2026-08-18T09:30:00Z" for a UTC instant, and
// "2026-08-18T09:30:00" for a naive one. The naive form takes the zone of the
// machine, so the server in UTC and a browser in Tehran differ by 3 hours and
// 30 minutes. React reports a hydration mismatch, and one of the two times is
// wrong.
<time>{new Date(order.created).toLocaleString()}</time>
```

```tsx
// Correct: the instant carries its offset, and the render states the zone.
import { useFormatter } from "next-intl";

export function CreatedAt({ created }: { created: string }) {
  const format = useFormatter();
  return (
    <time dateTime={created}>
      {format.dateTime(new Date(created), {
        dateStyle: "medium",
        timeStyle: "short",
      })}
    </time>
  );
}
```

The `useFormatter` hook reads the locale and the `timeZone` of the request
config, so the server render and the browser render agree.
`references/locale-routing-and-catalogs.md` owns that config.

Require an offset on every datetime field in the contract. A DRF field with
`USE_TZ = True` serializes an aware datetime, and the string ends in `Z` or in
a numeric offset. A field that ends in neither is naive, and the frontend
cannot repair it. The sibling skill `django-api-contract` owns that decision on
the server.

Hold the zone that the reader chose. Read `Intl.DateTimeFormat()
.resolvedOptions().timeZone` in the browser to offer a default, and store the
answer with the profile. A zone read on the server is the zone of the server.

Put the machine-readable instant in the `dateTime` attribute of `<time>`. The
element then carries the value that a copy and a test can read.

### The formatter takes the resolved locale

```ts
// Wrong: the locale is a constant, and the currency is another constant.
// Failure: a reader in the Persian locale reads a date in the English form
// beside Persian words. A price in a second market renders with the symbol of
// the first one, so the number states the wrong amount of money.
const price = new Intl.NumberFormat("en-GB", {
  style: "currency",
  currency: "GBP",
});
```

```ts
// Correct: the locale comes from the request, and the currency comes with the
// amount.
import { getFormatter } from "next-intl/server";

const format = await getFormatter();
const label = format.number(order.totalMinor / 100, {
  style: "currency",
  currency: order.currency,
});
```

A currency is a property of the record, and never of the locale. One reader
looks at prices in two currencies, and the locale decides the placement of the
symbol and the grouping of the digits alone.

`references/cell-formatting-and-export.md` owns the rule that a formatter is
built once, that no formatted string is stored, and that a numeric column takes
`tabular-nums`. All three hold here.

Hold money as an integer of minor units.
`references/type-modeling-and-narrowing.md` requires the brand over that value.

### The relative time needs a stated present

```tsx
// Wrong: the relative label reads the clock at render time.
// Failure: the server renders "2 minutes ago" and the browser renders
// "3 minutes ago" one second later. React reports a hydration mismatch. The
// label then never updates, because no state changes after the first paint.
<span>{rtf.format(-minutesSince(created), "minute")}</span>
```

```tsx
// Correct: one present reaches both renders, and an interval advances it.
"use client"; // it advances the label on an interval

import { useFormatter, useNow } from "next-intl";

export function Ago({ created }: { created: string }) {
  const now = useNow({ updateInterval: 60_000 });
  const format = useFormatter();
  return <span>{format.relativeTime(new Date(created), now)}</span>;
}
```

Supply `now` in the request config for the server render. Both machines then
compute the label from one instant.

An absolute timestamp needs no present, and it never drifts. Prefer it for a
record that a reader audits, and take the relative form for a feed.

### The numbering system is a display decision

```ts
// Wrong: the code strips every character that is not an ASCII digit.
// Failure: a Persian reader types ۱۲۳۴ in an amount field. The filter removes
// all four characters, the field submits an empty value, and the form reports
// that the amount is required. The reader retypes the same digits.
const amount = raw.replace(/[^0-9]/g, "");
```

```ts
// Correct: src/lib/digits.ts maps every digit form onto ASCII.
const EASTERN_ARABIC = /[۰-۹]/g; // Persian ۰ to ۹
const ARABIC_INDIC = /[٠-٩]/g; // Arabic ٠ to ٩

export function toLatinDigits(value: string): string {
  return value
    .replace(EASTERN_ARABIC, (d) => String(d.charCodeAt(0) - 0x06f0))
    .replace(ARABIC_INDIC, (d) => String(d.charCodeAt(0) - 0x0660));
}
```

Run the map before the schema parses the value, and before the request carries
it. `references/form-schema-and-field-binding.md` owns the field, and
`references/boundary-validation-and-api-types.md` requires the parse.

Three decisions stay apart:

| The decision | The answer |
| --- | --- |
| The digits that a reader reads | The numbering system of the locale, through `Intl` |
| The digits that a reader types | Any form. The application normalizes them |
| The digits that the program stores and sends | ASCII, always |

Select the system with the `-u-nu-` extension on the locale tag. The tag
`fa-IR` renders ۱۲۳۴, and the tag `fa-IR-u-nu-latn` renders 1234. Take the
Latin form for an identifier, a version number, and a code that a reader
copies.

### The calendar is a separate axis

```tsx
// Wrong: the code renders a Gregorian date with Persian month names.
// Failure: the reader reads "18 اوت 2026". A Persian reader expects 27
// Mordad 1405. The year, the month, and the day are all different values, and
// no translation of a month name produces them.
{format.dateTime(new Date(created), { dateStyle: "long" })}
```

```tsx
// Correct: the calendar is stated on the locale tag.
const jalali = new Intl.DateTimeFormat("fa-IR-u-ca-persian", {
  dateStyle: "long",
  timeZone: "Asia/Tehran",
});

<time dateTime={created}>{jalali.format(new Date(created))}</time>
```

`Intl` supports `gregory`, `persian`, `islamic`, `islamic-umalqura`,
`japanese`, `buddhist`, and more. The `fa` locale resolves to `persian` by
default, so state `-u-ca-gregory` where the product needs the Gregorian form in
a Persian interface.

Read the resolved calendar rather than the requested one. A runtime that lacks
the requested calendar falls back in silence:

```ts
const resolved = new Intl.DateTimeFormat("fa-IR-u-ca-persian").resolvedOptions();
// resolved.calendar === "persian" on a runtime that supports it
```

A date picker is a component, and `Intl` formats but never parses. Take a
picker that holds the calendar, and convert at the boundary of the request. The
API takes one instant in ISO 8601, whatever calendar the reader selected.
`references/form-schema-and-field-binding.md` owns the control, and
`references/component-composition.md` owns its shape.

Never send a Jalali year to Django. The server holds one calendar, and the
conversion belongs to the client that presented the other one.

### The sort order belongs to the language

```ts
// Wrong: the sort compares code units.
// Failure: the order puts every uppercase word before every lowercase word,
// and it puts "Ärger" after "Zeit". A Persian list orders by code point, which
// no reader of that language recognises as alphabetical.
const sorted = [...names].sort();
```

```ts
// Correct: one collator holds the rule of the locale.
const collator = new Intl.Collator(locale, {
  sensitivity: "base",
  numeric: true,
});

const sorted = [...names].sort((a, b) => collator.compare(a, b));
```

Build the collator once, outside the render. The constructor is expensive, and
a sort calls `compare` for every pair.

Take `sensitivity: "base"` for a search, so a difference of case and of accent
matches. Take `numeric: true`, so "Item 2" sorts before "Item 10".

A server-side sort and a client-side sort must not disagree. Sort on one side
only. Where the backend orders the page, the table renders that order and adds
none of its own. `references/data-table-and-server-driven-state.md` owns the
sort model, and the sibling skill `django-performance-optimizer` owns the index
behind the ordering field.

The normalization of Persian and Arabic characters before a comparison is
`references/bidirectional-layout-and-scripts.md`. A collator does not settle
ی against ي, because the two are different letters of two languages.

### The name of a language and of a region

```ts
// Correct: the platform holds the names, and no catalog repeats them.
const languages = new Intl.DisplayNames([locale], { type: "language" });
const regions = new Intl.DisplayNames([locale], { type: "region" });

languages.of("fa"); // "Persian" under en, and "فارسی" under fa
regions.of("IR"); // "Iran" under en
```

Never hold a language list or a country list in the catalog. The platform holds
about 300 regions and about 8,000 languages, and each one changes name with the
locale.

The switcher writes each language name in that language, and it therefore
passes the target locale to the constructor.
`references/locale-routing-and-catalogs.md` owns the switcher.

### The values that vary and that no formatter covers

| The value | What varies | The rule |
| --- | --- | --- |
| A phone number | The grouping, the country code, and the length | Store E.164. Format for display with a library, and never with a regular expression |
| A postal address | The field order, the presence of a state, and the postcode form | Take the field set of the country. Never assume a state field |
| A personal name | The order of the parts, and the count of the parts | Hold one full name field. A required surname field rejects a real reader |
| The first day of the week | Saturday, Sunday, or Monday | Read `Intl.Locale.prototype.getWeekInfo` where the runtime supports it. State the fallback |
| The weekend | Friday and Saturday in Iran, and Saturday and Sunday elsewhere | Read the same week info. A hard-coded weekend marks the wrong cells |
| A unit of measure | The system, and the written form | `Intl.NumberFormat` with `style: "unit"` |

`references/form-schema-and-field-binding.md` owns the schema over each of
these fields, and `references/interface-copy-and-voice.md` owns the label.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry supplied
those facts on 18 August 2026. A cell that holds no date is a package with a
current registry entry on that date. This file does not state an exact release
date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `Intl.DateTimeFormat`, `Intl.NumberFormat`, `Intl.RelativeTimeFormat`, `Intl.Collator`, `Intl.ListFormat`, `Intl.DisplayNames`, `Intl.PluralRules`, `Intl.Segmenter` | The platform holds the locale data. Reach for each one before any package. | Platform | — | Platform | — |
| Recommend | `next-intl` 4.13.6 | `useFormatter` and `getFormatter` read the locale and the zone of the request, so the two renders agree. | 4.13.6 | 10 Aug 2026 | Active, on a weekly cadence | None |
| Conditional | `date-fns-jalali` | Only where the product renders the Jalali calendar and needs arithmetic over it. It mirrors the `date-fns` API. | Current | Current | Active | None |
| Conditional | `jalaali-js` | Only for the conversion by itself, with no date library. It is small and it has one job. | Current | Current | Active | None |
| Conditional | `react-multi-date-picker` | Only where the product needs a Jalali picker. It carries its own calendar and its own locale data. | Current | Current | Active | None |
| Conditional | `libphonenumber-js` | Only where the product formats or validates a phone number. Take the `min` metadata build where the country set is known. | Current | Current | Active | None |
| Conditional | `@date-fns/tz` or `date-fns-tz` | Only for zone arithmetic that `Intl` cannot express, such as the start of a day in another zone. | Current | Current | Active | None |
| Audit only | `luxon` | Alive in a project that standardised on it. It wraps `Intl`, and it adds about 70 kB. New code needs neither. | Current | Current | Active | None |
| Reject | `toLocaleDateString()` and `toLocaleString()` with no arguments | The platform default of the machine decides the form, so the server and the browser disagree. | — | — | — | — |
| Reject | A hand-written Jalali conversion | The leap year rule of the Persian calendar is not a modulus, and a hand-written table drifts. | — | — | — | — |
| Reject | A hard-coded month name array | It holds one calendar and one language, and `Intl` holds every one of them. | — | — | — | — |

`references/client-bundle-and-third-party-scripts.md` rejects `moment` and owns
the byte budget over a date library. Do not add a second row for it.

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| React reports a hydration mismatch on a timestamp | The render takes the zone of the machine | Read the two strings in the console message | State the locale and the zone, and take them from the request config |
| A time is 3 hours and 30 minutes out for one reader | The API sent a naive datetime | Read the raw field for a `Z` or an offset | Require an aware datetime in the contract |
| A relative label never updates after the first paint | No state advances the present | Leave the surface open for two minutes | Take `useNow` with an update interval |
| A form rejects an amount that the reader typed | The filter stripped the Persian digits | Type ۱۲۳۴ into the field | Normalize every digit form to ASCII before the parse |
| A price shows the wrong currency symbol | The formatter holds a constant currency | Render a record in a second currency | Read the currency from the record |
| A Persian date shows the Gregorian year | The tag carries no calendar, or the locale is not `fa` | Compare the rendered year against 1405 | State `-u-ca-persian` on the tag |
| A list order looks random to a reader | The sort compares code units | Sort a list that holds accents or a non-Latin script | Take one `Intl.Collator` for the locale |
| A country list is missing a country | A catalog holds the list | Search for a country that no key names | Take `Intl.DisplayNames` |
| The calendar of a picker does not match the rendered date | The picker holds its own calendar setting | Select a date, and read it back on the detail view | State one calendar for the surface, and convert at the request |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| The `Intl.NumberFormat` V3 options are on the baseline runtimes | A hand-written compact formatter, or a hand-written rounding helper | Take `notation`, `signDisplay`, `roundingMode`, and `roundingIncrement` |
| `Temporal` is a TC39 proposal, and it is not on the runtime baseline | A `Temporal.` call with no polyfill, or a polyfill in the client bundle with no measurement | Stay on `Date` with `Intl` for the render. Take a zone library for arithmetic that `Intl` cannot express |
| `Intl.Segmenter` is available on the baseline runtimes | A `split("")` over text, or a `length` read as a count of characters | Take a segmenter for a grapheme count and for a word count |
| `Intl.Locale.prototype.getWeekInfo` is not on every baseline runtime | A call with no feature test | Test for the method, and state the fallback for the first day of the week |

## Verification

```bash
# 1. Read the installed versions before you write code.
cat package.json | rg '"next-intl"|"date-fns"|"luxon"|"libphonenumber-js"'

# 2. Find a locale-sensitive call with no arguments. This must print nothing.
rg -n 'toLocaleDateString\(\)|toLocaleTimeString\(\)|toLocaleString\(\)' \
  -g '*.ts' -g '*.tsx' src/

# 3. Find a date formatter that states no time zone. Read every hit.
rg -n -A5 'new Intl\.DateTimeFormat' -g '*.ts' -g '*.tsx' src/ | rg -v 'timeZone'

# 4. Find a formatter with a hard-coded locale tag. Read every hit.
rg -n 'Intl\.(DateTimeFormat|NumberFormat|Collator)\(["'"'"']' \
  -g '*.ts' -g '*.tsx' src/

# 5. Find a currency that a constant supplies. Read every hit.
rg -n 'currency:\s*["'"'"']' -g '*.ts' -g '*.tsx' src/

# 6. Find a digit filter that removes every non-ASCII digit. Read every hit.
rg -n 'replace\(/\[\^0-9\]|\[\^\\d\]' -g '*.ts' -g '*.tsx' src/

# 7. Find a bare sort over strings. Each hit takes a collator.
rg -n '\.sort\(\)' -g '*.ts' -g '*.tsx' src/

# 8. Find a hard-coded month name list or a country list. Read every hit.
rg -n 'January|Jan.*Feb.*Mar' -g '*.ts' -g '*.tsx' -g '*.json' src/

# 9. Confirm that the resolved calendar is the requested one.
node -e "console.log(new Intl.DateTimeFormat('fa-IR-u-ca-persian').resolvedOptions())"

# 10. Render a record in each target locale. The date, the number, the
#     currency, and the calendar are each correct for that locale.

# 11. Type the Persian digits ۱۲۳۴ into every numeric field. Each field
#     accepts them, and the request carries 1234.

# 12. Leave a surface with a relative label open for two minutes. The label
#     advances, and the console reports no hydration mismatch.
```

## Review checklist

- [ ] Does every datetime field from the API carry an offset or a `Z`?
- [ ] Does every date render state a time zone?
- [ ] Does every formatter take the locale from the request, rather than from a
      constant?
- [ ] Does every currency come from the record that holds the amount?
- [ ] Does the code hold money as an integer of minor units?
- [ ] Does a relative label read one stated present on both renders?
- [ ] Does a relative label advance after the first paint?
- [ ] Does every numeric input normalize the Persian and the Arabic digits to
      ASCII?
- [ ] Does every request and every stored value carry ASCII digits?
- [ ] Does a Persian surface state the calendar on the locale tag?
- [ ] Does the code read the resolved calendar rather than the requested one?
- [ ] Does every date leave for the API as one instant in ISO 8601?
- [ ] Does every sort over words pass through one `Intl.Collator`?
- [ ] Is the collator built once, outside the render?
- [ ] Do the language names and the region names come from
      `Intl.DisplayNames`?
- [ ] Does the code read the first day of the week rather than assume it?
- [ ] Is `Temporal` absent, or is its polyfill measured against the budget?

## Handoffs

- The locale, the request config, the `timeZone` field, and the switcher →
  `references/locale-routing-and-catalogs.md`.
- The formatter built once, the stored formatted string, `tabular-nums`, and
  the export file → `references/cell-formatting-and-export.md`.
- The ICU message that holds a formatted value, the plural rule, and
  `Intl.ListFormat` → `references/message-catalog-and-plurals.md`.
- The normalization of Persian and Arabic letters, the grapheme-safe
  truncation, and the direction of a number inside a sentence →
  `references/bidirectional-layout-and-scripts.md`.
- The schema over a date field, a phone field, or an amount field, and the
  control that binds to it → `references/form-schema-and-field-binding.md`.
  The parse over a value that enters the program →
  `references/boundary-validation-and-api-types.md`.
- The brand over a money amount → `references/type-modeling-and-narrowing.md`.
- The sort model of a table, and the column that holds a formatted value →
  `references/data-table-and-server-driven-state.md`. The axis label →
  `references/charts-and-visual-encoding.md`.
- The bytes of a date library, and the budget over them →
  `references/client-bundle-and-third-party-scripts.md`.
- The label beside a formatted value →
  `references/interface-copy-and-voice.md`.
- The shape of a date picker component →
  `references/component-composition.md`.
- The test over a rendered locale → domain 20 `testing-and-quality`. Not
  integrated yet.
- The aware datetime that the serializer emits, and the field that carries a
  currency code → the sibling skill `django-api-contract`. The index behind an
  ordering field → the sibling skill `django-performance-optimizer`.
