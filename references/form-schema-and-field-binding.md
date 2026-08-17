# Form schema and field binding

React Hook Form 7.85, Zod 4.4, `@hookform/resolvers` 5.9, React 19.2.6 or
later, Next.js 16.3, against a Django and DRF backend. This file owns the
schema that one form stands on, and the bind of a control to that schema. The
subjects are the approach decision, `useForm` with `zodResolver`, and the
`z.input` and `z.output` generics. They also include `register` against
`Controller`, the read of a field value, and the moment that validation starts.
The last subjects are the field array, and the input that no native control
covers.

The submit, the server error, and the map onto a control are
`references/form-submission-and-server-errors.md`. The step, the draft, and
the exit guard are `references/multi-step-forms-and-unsaved-work.md`. The Zod 4
API and the DRF envelopes are `references/boundary-validation-and-api-types.md`.
The `<label>`, the hint, and the error semantics of a field are
`references/semantics-and-accessible-names.md`, and that domain holds a veto.

## Principle

One form stands on one schema. The schema produces the types and the rules
together. Two sources for one rule drift apart, and the drift reaches the user
as a field that the client accepts and the server refuses.

A value has two types. One is what the user types, and the other is what the
schema produces. A model with one type breaks where a rule fills a missing
value or converts one.

A control that the browser already holds needs no wrapper. A control that holds
its own value needs one, because the form cannot read it in any other way.

A message that arrives at the first character punishes the user for a value
that is not finished. Wait for the user to leave the field.

The client check is a courtesy. It is fast, and it is not authority. The server
decides.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### Choose the approach

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| The form has conditional fields, an array of rows, or a control that holds its own value | React Hook Form with `zodResolver`. | The route must work with no JavaScript, which the next row covers. | Two dependencies, a client boundary, and a schema that the server repeats. |
| The submit must work before the JavaScript arrives, or the route is server-first | A React 19 Action on `<form action>`, with the schema on the server. `references/suspense-and-actions.md` owns `useActionState` and `useFormStatus`. | The form needs instant cross-field feedback, which the first row covers. | Each keystroke rule costs a round trip, or a second schema on the client. |
| The user needs the first row's feedback, and the submit must redirect or revalidate on the server | React Hook Form for the fields, and a Server Action or the typed client for the submit. | Nothing here needs the client library, so the second row is simpler. | One schema in two runtimes, and a test for each path. |
| The form only reads a list — a search, a filter, or a sort | `<Form>` from `next/form` with a string `action`, and `nuqs` for the values. `references/client-and-url-state.md` owns the parsers. | The form writes a record, which the rows above cover. | The values sit in the URL, so nothing secret can enter this form. |

A search form is not a form problem. It is a URL problem, and the URL owns the
value. Never hold a filter in `useState` inside a form.

### One schema, and the two types that it produces

```ts
// src/features/orders/schema.ts
import { z } from "zod";

export const orderFormSchema = z.object({
  email: z.email(),
  quantity: z.number().int().positive(),
  note: z.string().max(500).default(""),
});
```

```ts
// Wrong: one type for a schema that produces two.
// Failure: .default("") makes note optional on the input side and present on
// the output side. z.infer gives the output type, so the generic and the
// resolver disagree. TypeScript reports the field as required where the form
// leaves it empty, and the developer reaches for a cast to silence it.
const form = useForm<z.infer<typeof orderFormSchema>>({
  resolver: zodResolver(orderFormSchema),
});
```

```ts
// Correct: the input type binds the fields, and the output type reaches the
// submit handler. The middle generic is the resolver context, which this form
// does not use.
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";

const form = useForm<
  z.input<typeof orderFormSchema>,
  unknown,
  z.output<typeof orderFormSchema>
>({
  resolver: zodResolver(orderFormSchema),
  mode: "onTouched",
  reValidateMode: "onChange",
  defaultValues: { email: "", note: "" },
});
```

Confirm the order of the three generics against the installed version before
you write the code. The pair `z.input` and `z.output` is only necessary where a
rule changes the value. A schema with no `.default()`, no `.transform()`, and
no `.catch()` has one type, and `z.infer` serves it.

Give every field a `defaultValues` entry. A field that starts as `undefined`
and later carries a string moves from uncontrolled to controlled, and React
reports it in the console.

| The intent | The Zod 4 API | What it does to the form |
| --- | --- | --- |
| The value can be absent | `.optional()` | The field is optional on the input type. |
| The value can be `null` | `.nullable()` | The field accepts `null`, which is not the same as absent. |
| Fill the value when it is missing | `.default(v)` | The field is optional on the input type, and present on the output type. |
| A rule over two fields | `.refine()` or `.superRefine()` | The message needs a `path`, or it lands on the form and not on a field. |
| The shape depends on a choice | `z.discriminatedUnion()` | Each branch validates only its own fields. |

```ts
// Correct: a cross-field rule names the field that it marks.
const passwordSchema = z
  .object({ password: z.string().min(12), confirm: z.string() })
  .refine((value) => value.password === value.confirm, {
    path: ["confirm"], // without this, the message reaches no field
    error: "The two entries are different.",
  });
```

Zod 4 takes one `error` parameter. It replaces `message`, `invalid_type_error`,
`required_error`, and `errorMap`. `references/boundary-validation-and-api-types.md`
holds the full list of the Zod 3 calls that Zod 4 removed.

A number input returns a string. Convert it with `valueAsNumber` on `register`,
or with `z.coerce.number()` in the schema. Take one of the two, and never both.

### `register` for a native control, `Controller` for a controlled one

| The control | The API | The reason |
| --- | --- | --- |
| `<input>`, `<textarea>`, `<select>` | `register` | The DOM holds the value, so no render runs for a keystroke. |
| A select, a combobox, a date picker, an OTP field, a phone field, a rich-text field | `Controller` | The control holds its own value, and only a subscription reaches it. |
| A field component that more than one form reuses | `useController` | The hook gives the same value inside the component, with no render prop. |

```tsx
// Wrong: a Controller around a native input.
// Failure: the form renders again on each keystroke, which register avoids.
// The wrapper adds code, and it returns the same result.
<Controller
  name="email"
  control={form.control}
  render={({ field }) => <input {...field} />}
/>
```

```tsx
// Correct: register binds the native control, and the DOM holds the value.
<input
  id="email"
  type="email"
  autoComplete="email"
  aria-invalid={form.formState.errors.email !== undefined}
  {...form.register("email")}
/>
<input
  id="quantity"
  type="number"
  {...form.register("quantity", { valueAsNumber: true })}
/>
```

```tsx
// Correct: Controller binds a control that holds its own value.
<Controller
  name="country"
  control={form.control}
  render={({ field, fieldState }) => (
    <CountrySelect
      value={field.value}
      onValueChange={field.onChange}
      onBlur={field.onBlur}
      aria-invalid={fieldState.invalid}
    />
  )}
/>
```

Pass `onBlur` to the control. The mode `onTouched` needs the blur event, and a
control that never reports it never validates until the submit.

`references/component-composition.md` owns the shape of a field component, and
`references/component-styles-and-variants.md` owns the classes that its error
state selects. The `id`, the `htmlFor`, and the `aria-describedby` of a field
are `references/semantics-and-accessible-names.md`.

### Read a value without a render of the whole form

| The need | The API | The render cost |
| --- | --- | --- |
| Read the value once, inside a handler | `getValues()` | None. It does not subscribe. |
| Show a value in one child, and follow it | `useWatch()` in that child | That child alone. |
| Follow a value at the root of the form | `watch()` | The whole form, on each keystroke. |

```ts
// Wrong: the root of the form subscribes to every value.
// Failure: each keystroke renders every field, every wrapper, and every list
// row. A long form drops frames while the user types.
const values = form.watch();
```

```ts
// Correct: the child that shows the value subscribes to that value alone.
const quantity = useWatch({ control: form.control, name: "quantity" });
```

### The moment that validation starts

```ts
// Wrong: validation starts at the first character.
// Failure: the user types "n" in an email field and reads "Enter a valid email
// address". The message is correct and useless, and it repeats on each
// keystroke until the value is complete.
useForm({ mode: "onChange" });
```

```ts
// Correct: the first check runs at the blur, and the correction is live.
useForm({ mode: "onTouched", reValidateMode: "onChange" });
```

The mode `onBlur` is the alternative. It checks at the blur every time, and it
does not follow a correction while the user repairs the field. Take
`onTouched`, and take `onBlur` only where a check is expensive.

`formState.isValid` comes from a validation result, so it needs a resolver or a
validation mode. A read of `isValid` validates the whole form. Never bind
it to the `disabled` attribute of a submit button;
`references/form-submission-and-server-errors.md` holds that rule and its
reason.

### The field array

```tsx
// Correct: the row key comes from the field id, and never from the index.
const { fields, append, remove } = useFieldArray({
  control: form.control,
  name: "items",
});

{fields.map((field, index) => (
  <fieldset key={field.id}>
    <legend>Item {index + 1}</legend>
    <input {...form.register(`items.${index}.sku`)} />
    <input
      type="number"
      {...form.register(`items.${index}.quantity`, { valueAsNumber: true })}
    />
    <button type="button" onClick={() => remove(index)}>
      Remove item {index + 1}
    </button>
  </fieldset>
))}
```

An index key moves the value of one row into the box of another after a remove.
`references/component-composition.md` owns that rule for every list.

The `id` that `useFieldArray` supplies is the key, and the field name still
carries the index. A server error on the second row therefore arrives as
`items.1.quantity`, which
`references/form-submission-and-server-errors.md` maps.

Name each remove button. "Remove" alone gives a screen-reader user a list of
identical buttons.

### An input that no native control covers

| The input | The package | The rule |
| --- | --- | --- |
| A one-time code | `input-otp` | Paste must work. `references/semantics-and-accessible-names.md` states the criterion, and it holds a veto. |
| A telephone number | `libphonenumber-js`, with `react-phone-number-input` for the control | Store E.164. Validate against the country, and never against a regular expression. |
| A date | `react-day-picker`, with `@internationalized/date` for the arithmetic | A date is not a `Date`. A time zone changes the day, so the calculation belongs in the library. |
| The strength of a password | `@zxcvbn-ts/core` | A meter is a hint for the user. The rule that refuses a password belongs to the server. |
| A mask over a value | `react-imask` | Take a mask only where it helps. A mask that rejects a paste or blocks a valid form of the value costs more than it gives. |
| A file | The schema states the type, the size, and the count | `references/file-upload-and-transport.md` owns the picker, the transport, and the progress. |

A masked field and a coerced field both hold two values: what the user sees and
what the form submits. State which one the schema validates.

### The libraries

The table gives each package its latest version, its last release date, and its
maintenance status. The research supplied those facts in August 2026, and it
supplied no advisory count for any of them, so this table states none. A cell
that holds no date is a package whose exact release date this material does not
state.

| Tier | Package | The rule | Latest version | Last release | Maintenance |
| --- | --- | --- | --- | --- | --- |
| Recommend | `react-hook-form` | The default for a form that the client owns. | 7.85.0 | 8 August 2026 | Active |
| Recommend | `@hookform/resolvers` | `zodResolver` for a Zod schema, and `standardSchemaResolver` for any Standard Schema validator. | 5.9.0 | Current | Active |
| Recommend | `zod` | The schema, the messages, and the types. | 4.4.3 | 4 May 2026 | Active |
| Recommend | React 19 Actions | The server-first form. No dependency, because React supplies the hooks. | — | — | — |
| Recommend | `next/form` | The search or filter form that navigates. No dependency, because Next.js supplies it. | — | — | — |
| Recommend | `input-otp` | The segmented one-time-code field. The cadence is slow, and the package is stable. | 1.4.2 | About two years old | Dormant cadence, no replacement |
| Recommend | `libphonenumber-js` with `react-phone-number-input` | The telephone number, parsed and validated to E.164. | 1.13.11, and 3.4.17 | Current | Active |
| Recommend | `react-day-picker` with `@internationalized/date` | The calendar, and the time-zone-correct arithmetic behind it. | 10.0.1, and 3.12.3 | Current | Active |
| Recommend | `@zxcvbn-ts/core` | The strength meter. It is the maintained TypeScript port. | 4.1.2 | Current | Active |
| Conditional | `@tanstack/react-form` | A framework-agnostic form layer with first-class async validation. Base UI integrates with it directly. State the reason to add a second form library. | 1.33.5 | Current | Active |
| Conditional | `@base-ui/react` `Field` and `Form` | The native-validation path. It binds to React Hook Form through `Controller`. `references/component-composition.md` owns the primitive itself. | 1.7.0 | Current | Active |
| Conditional | `@conform-to/react` | A form that must work with no JavaScript, on Server Actions. | 1.19.4 | Current | Active |
| Conditional | `valibot` | A smaller validator with the same Standard Schema contract. Audit it against the Zod that the repository already carries. | 1.4.2 | 28 June 2026 | Active |
| Audit-only | `vest` | A declarative validator that works. It is a second model of the same rules, so state why Zod does not serve. | 6.3.2 | Current | Active |
| Reject | `formik` | Classified inactive. Keep it in the files that hold it, and never take it for a new form. | 2.4.9 | About nine months old | Inactive |
| Reject | `zxcvbn` | Unmaintained for about nine years. `@zxcvbn-ts/core` replaces it. | 4.4.2 | About nine years old | Unmaintained |
| Reject | `cleave.js` | Near-dormant masking. `react-imask` replaces it. | — | — | Near-dormant |
| Reject | One `useState` for each field | It is a second model of the form, with no validation and no error state. `references/state-and-effects.md` holds the same rule. | — | — | — |

### Version discipline

Read the installed versions before you write code.

React Hook Form 7.85.0 is the stable line, and `latest` points at 7. Never
write a 7 idiom and an 8 idiom in one file.

The 8.0.0 pre-release changes the API in four ways. It passes an input ref in
place of a partial. It renames the `useFieldArray` `id` to `key`, and it
removes `keyName`. It renames the watch component. It no longer permits
`setValue` to write to a field array, so take `replace` on 8.

`@hookform/resolvers` 5 supports Zod 4, and `zodResolver` detects the Zod major
version by itself. The `standardSchemaResolver` export at
`@hookform/resolvers/standard-schema` accepts any validator that implements
Standard Schema v1.

`shouldUnregister` defaults to `false`. A field that unmounts keeps its value,
which is what a conditional field and a wizard step both need.
`references/multi-step-forms-and-unsaved-work.md` depends on this default.

Zod 4 needs TypeScript 5.5 or later. The repository pins 5.9.

## Verification

```bash
# 1. Read the installed versions before you write code.
node -p "require('react-hook-form/package.json').version"
node -p "require('zod/package.json').version"
node -p "require('@hookform/resolvers/package.json').version"

# 2. Find a useForm call that has no resolver. Read every hit.
rg -n -A6 'useForm[<(]' -g '*.tsx' src/ | rg -B2 -A4 'useForm' | rg -v resolver

# 3. Find validation that starts at the first character. This prints nothing.
rg -n "mode:\s*['\"]onChange['\"]" -g '*.ts*' src/

# 4. Find a Controller around a native input. Read every hit.
rg -n -A4 '<Controller' -g '*.tsx' src/ | rg '<(input|select|textarea)'

# 5. Find a watch() call at the root of a form. Read every hit.
rg -n '\bwatch\(\)' -g '*.tsx' src/

# 6. Find an index key on a field array row. This prints nothing.
rg -n -B3 'fields\.map' -A3 -g '*.tsx' src/ | rg 'key=\{index\}'

# 7. Find a form value type that ignores the input side. Read every hit.
rg -n 'useForm<z\.infer' -g '*.ts*' src/

# 8. Type the form. A resolver that disagrees with the generics fails here.
pnpm typecheck

# 9. Open the form. Type one character in each field, and confirm that no
#    message appears before the field loses focus.
```

## Review checklist

- [ ] Does one schema hold the rules and the types for this form?
- [ ] Does the `useForm` call carry a resolver?
- [ ] Does a schema with a `.default()`, a `.transform()`, or a `.catch()` use
      `z.input` and `z.output` in the generics?
- [ ] Does every field have a `defaultValues` entry?
- [ ] Does every cross-field rule carry a `path`, so its message reaches a
      field?
- [ ] Does every native control bind with `register` rather than `Controller`?
- [ ] Does every control that holds its own value bind with `Controller` or
      `useController`, and receive `onBlur`?
- [ ] Does a number field convert with `valueAsNumber` or with
      `z.coerce.number()`, and never with both?
- [ ] Is the validation mode `onTouched` or `onBlur`, and never `onChange` from
      the mount?
- [ ] Does every subscription to a field value use `useWatch` in the child
      rather than `watch()` at the root?
- [ ] Does every field array row take its key from the field `id` rather than
      the index?
- [ ] Does every repeated control in an array carry a name that states its row?
- [ ] Does the one-time-code field accept a paste?
- [ ] Does a telephone number reach the backend as E.164?
- [ ] Does a date calculation run in a date library rather than on a `Date`?
- [ ] Does the code hold one form library, and one validator?

## Handoffs

- The submit, the double submit, the DRF field-error map, and the values that
  survive a failure → `references/form-submission-and-server-errors.md`.
- The step in the URL, the draft, and the guard on an exit →
  `references/multi-step-forms-and-unsaved-work.md`.
- The Zod 4 API, the calls that Zod 3 held, and the DRF envelopes that a parse
  proves → `references/boundary-validation-and-api-types.md`.
- `useActionState`, `useFormStatus`, `useOptimistic`, and the rule that an
  expected error returns as state → `references/suspense-and-actions.md`.
- The `<label>`, the hint, the error, `aria-invalid`, `aria-describedby`, and
  the paste in a one-time-code field →
  `references/semantics-and-accessible-names.md`. That domain holds a veto.
- The shape of a field component, its parts, and the list key →
  `references/component-composition.md`.
- One reducer in place of one `useState` for each field →
  `references/state-and-effects.md`.
- The classes on a field control, and the variant that its error state selects
  → `references/component-styles-and-variants.md`.
- The URL parser behind a search form, and the store that a value crosses a
  route in → `references/client-and-url-state.md`.
- The file picker, the upload transport, and the progress →
  `references/file-upload-and-transport.md`. This file owns only the schema rule
  over a file.
- The words inside a label, a hint, and a validation message →
  `references/interface-copy-and-voice.md`. This file owns the key, and that
  file owns the text.
- The catalog key and the plural rule behind a translated message →
  `references/message-catalog-and-plurals.md`. The file that holds the catalog
  and the locale route → domain 19 `internationalization-and-rtl`. Not
  integrated yet.
- The test that fills a form by its accessible label, and the schema test →
  domain 20 `testing-and-quality`. Not integrated yet.
