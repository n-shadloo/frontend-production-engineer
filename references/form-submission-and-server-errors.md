# Form submission and server errors

React Hook Form 7.85, Zod 4.4, React 19.2.6 or later, Next.js 16.3, against a
Django and DRF backend. This file owns what happens after the user submits. The
subjects are the one submit path, the second click, and the pending state. They
also include the map from a DRF rejection onto a control, the form-level
message, and the status that each failure produces. The last subjects are the
values that survive a failure, the reset after a success, and the log that
carries no secret.

The schema and the bind of a control are
`references/form-schema-and-field-binding.md`. The `ApiError` that a failure
arrives as is `references/api-client-and-request-safety.md`, and the DRF
envelopes behind it are `references/boundary-validation-and-api-types.md`. The
error summary, the focus after a failed submit, and the live region are
`references/keyboard-focus-and-live-regions.md`, and that domain holds a veto.

## Principle

The client check is user experience. The server is the authority. Every submit
path handles a rejection of data that the client accepted.

An error that names a field belongs on that field. A message that names no
field belongs above the fields, in one region.

A message that disappears is not a report. A user who reads slowly, a user who
looks away, and a screen-reader user all lose it.

A submit that the user can send twice arrives twice. The backend then holds two
records, and the user reads one confirmation.

A failed submit keeps every value that the user typed. Work that the user did
is not the program's to discard.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The map from a DRF rejection onto a control

```ts
// Wrong: the failure becomes a toast.
// Failure: the server said which field is wrong, and the user reads a message
// that names no field. The toast then disappears, the field carries no error
// state, and a screen-reader user cannot reach the message again.
catch (error) {
  toast.error((error as Error).message);
}
```

```ts
// Correct: src/lib/forms/apply-server-errors.ts puts each message on its
// control, and returns the messages that belong to the form.
import type { FieldPath, FieldValues, UseFormSetError } from "react-hook-form";
import type { ApiError } from "@/lib/api/errors";

// A list serializer names a row by its index. The set holds one pattern for
// the whole array, so items.0.sku and items.7.sku both match items.*.sku.
function pattern(name: string): string {
  return name.replace(/\.\d+(?=\.|$)/g, ".*");
}

export function applyServerErrors<T extends FieldValues>(
  setError: UseFormSetError<T>,
  error: ApiError,
  fieldPaths: ReadonlySet<string>,
): string[] {
  const formLevel: string[] = [];
  let focused = false;

  for (const [name, messages] of Object.entries(error.fieldErrors ?? {})) {
    const message = messages[0];
    if (message === undefined) continue;
    if (!fieldPaths.has(pattern(name))) {
      formLevel.push(message); // non_field_errors, and any key with no control
      continue;
    }
    setError(
      name as FieldPath<T>,
      { type: "server", message },
      { shouldFocus: !focused },
    );
    focused = true;
  }

  return formLevel.length > 0 || focused ? formLevel : [error.message];
}
```

```ts
// Correct: the form states the paths that it can render.
const ORDER_FIELD_PATHS: ReadonlySet<string> = new Set([
  "email",
  "quantity",
  "note",
  "address.city", // a nested serializer names the field with a dot
  "items.*.sku", // a list serializer names the row with an index
]);
```

The set is the whole rule. A message on a path that this form does not render
reaches nobody, so it goes to the form-level region instead. `non_field_errors`
never matches a path, so it lands there without a special case.

`normalizeApiError` flattens the DRF body into `ApiError.fieldErrors` before
this function reads it. A nested serializer arrives as `address.city`, and a
list serializer arrives as `items.1.sku`.
`references/api-client-and-request-safety.md` owns that function, and
`references/boundary-validation-and-api-types.md` owns the envelopes that reach
it. Where the backend runs `drf-standardized-errors`, the `attr` field carries
the same dotted path, and the separator is a server setting.

DRF 3.17 changed the error output of a list serializer. Confirm the shape
against the deployed version before you trust an indexed path, with the request
in the verification block below. The sibling skill `django-api-contract` owns
the envelope on the server side, and this file changes nothing there.

Prefer the DRF `code` over the message where the form must branch. The message
is translated, and a branch on translated text breaks under a second locale.

### The submit handler

```tsx
// Correct: one path, and every failure lands somewhere the user can read.
"use client"; // reason: the submit handler holds the form state

const [formErrors, setFormErrors] = useState<readonly string[]>([]);

const onSubmit = form.handleSubmit(async (values) => {
  setFormErrors([]);
  try {
    const order = await createOrder(values);
    form.reset(toFormValues(order)); // the committed values, after a success
    await queryClient.invalidateQueries({ queryKey: orderKeys.lists() });
  } catch (error) {
    if (!isApiError(error)) throw error; // not from the API: the boundary renders
    setFormErrors(applyServerErrors(form.setError, error, ORDER_FIELD_PATHS));
  }
});
```

`isApiError` sits beside `normalizeApiError`, and
`references/server-state-and-query-cache.md` holds it. A value that is not an
`ApiError` is a defect in the program, and it belongs at the error boundary.

`references/server-state-and-query-cache.md` owns the key that the success
invalidates, and it owns the mutation state where the submit runs through
`useMutation`. In a Server Action the order is authorize, validate, mutate,
invalidate, redirect, and `references/data-access-and-mutations.md` owns it.

### Each status, and what the form does

| The status | What the form does | It reverses when | The cost |
| --- | --- | --- | --- |
| 400 with a field dictionary | Put each message on its control, and move focus to the first one. Never retry. | Never. The answer is deterministic, so a repeat returns it again. | The form needs a path for every field that the serializer can name. |
| 400 with `non_field_errors` | Render it above the fields, in one alert region. | Never. No control carries the message. | The form needs a region that no field owns. |
| 409 | State the conflict, keep every value, and offer a reload of the record. | Never. A blind repeat writes over the other change. | The view needs the other version, or a path to fetch it. |
| 422 | Treat it as the 400 row. | The backend gives 422 a different meaning, which the code states. | None beyond the 400 row. |
| 429 | Read `Retry-After`, hold the submit for that period, and count the period down. | Nothing. The limit is the answer. | The user waits, and the form reports the wait. |
| 5xx, or a network failure | Render one form-level message, keep every value, and leave the submit operable. | Never on a submit. A throw discards the work that the user did. | No error boundary sees the failure, so the report here must be complete. |

The last row is the reversal that `references/suspense-and-actions.md` names in
its own table. A 5xx throws where the user loses nothing. A submit carries
values that the user typed, so it reports at the form level instead.

`references/api-client-and-request-safety.md` owns `Retry-After`, the retry
rule, and the `Idempotency-Key` that a repeated POST needs. Never retry a POST
without that key.

### One submit, once

```tsx
// Wrong: nothing stops a second click.
// Failure: two clicks send two POSTs. The backend writes two records, and the
// user reads one confirmation.
<button type="submit">Save</button>
```

```tsx
// Wrong: the button reports an invalid form by refusing to work.
// Failure: the button is inert, and it states no reason. A read of isValid
// also validates every field, so the rules run before the user types.
<button type="submit" disabled={!form.formState.isValid}>Save</button>
```

```tsx
// Correct: the button works until the submit runs, and it refuses the second
// run while the first is open.
<button type="submit" disabled={form.formState.isSubmitting}>
  {form.formState.isSubmitting ? "Saving" : "Save"}
</button>
```

`references/semantics-and-accessible-names.md` states that a disabled submit
button hides the reason for a refusal. The pending state is the one exception
that the rule permits. The refusal lasts for one request, and the label states
the reason while it lasts.

A React 19 Action reads the same state from `useFormStatus`, which
`references/suspense-and-actions.md` owns.

### The values survive the failure

```ts
// Wrong: the form resets whatever happened.
// Failure: a 500 clears every value, so the user types the whole record again
// to retry a request that the backend, not the user, failed.
} finally {
  form.reset();
}
```

```ts
// Correct: the reset runs after a success, and it takes the committed values.
form.reset(toFormValues(order));
```

Reset with the record that the server returned, and never with the values that
the client sent. The two differ where the backend normalizes a value, and the
form then shows what the user typed rather than what the backend stored.

### The report reaches the user

React Hook Form moves focus to the first invalid field on a failed client
check, through the default `shouldFocusError`. The `applyServerErrors` function
above does the same for a server rejection, with `shouldFocus` on the first
field alone.

The alternative is an error summary that takes focus and links to each field.
Take one of the two, and use it on every form of the product.
`references/keyboard-focus-and-live-regions.md` owns the summary, the focus
rule, and the live region. This file owns the list of failures that both read.

Render the form-level messages in one region above the fields, and give that
region the `alert` role. A toast is never the only report of a validation
error.

```tsx
// Correct: the form-level messages sit above the fields, in one region.
{formErrors.length > 0 && (
  <div role="alert">
    <ul>
      {formErrors.map((message) => (
        <li key={message}>{message}</li>
      ))}
    </ul>
  </div>
)}
```

### The log carries no secret

```ts
// Wrong: the failed submit logs what the user typed.
// Failure: a password, a token, and a personal value reach the browser console
// and every log sink behind it. None of the three belongs there, and a log
// that a sink already holds cannot be recalled.
console.error("submit failed", values);
```

```ts
// Correct: the log carries the status and the names, and never the values.
console.error("submit failed", {
  status: error.status,
  code: error.code,
  fields: Object.keys(error.fieldErrors ?? {}),
});
```

### The Action path

```ts
// Correct: the Action returns the messages and the values as state, so a
// submit with no JavaScript renders the form again with the user's work in it.
// app/orders/actions.ts
"use server";
import { z } from "zod";
import { orderFormSchema } from "@/features/orders/schema";

// FormData carries text, so the server schema converts what the form sent.
const actionSchema = orderFormSchema.extend({
  quantity: z.coerce.number().int().positive(),
});

export type OrderFormState = {
  values?: { email: string; quantity: string; note: string };
  fieldErrors?: Record<string, string[]>;
  formErrors?: string[];
};

export async function createOrderAction(
  _previous: OrderFormState,
  formData: FormData,
): Promise<OrderFormState> {
  const values = {
    email: String(formData.get("email") ?? ""),
    quantity: String(formData.get("quantity") ?? ""),
    note: String(formData.get("note") ?? ""),
  };

  const parsed = actionSchema.safeParse(values);
  if (!parsed.success) {
    const flat = z.flattenError(parsed.error);
    return { values, fieldErrors: flat.fieldErrors, formErrors: flat.formErrors };
  }

  // authorize, mutate, invalidate, redirect: data-access-and-mutations.md
}
```

Return the values with the state. A form that renders from the Action state
alone loses every value on an error. A submit with no JavaScript is exactly
that case.

`z.flattenError` gives `{ formErrors, fieldErrors }` in Zod 4. It replaces
`.flatten()`, which Zod 3 held.

The Action verifies the session in its own body, whatever the form does.
`references/route-protection-and-permissions.md` holds that rule, and it holds
a veto.

## Verification

```bash
# 1. Find a toast in a catch that sets no field error. Read every hit.
rg -n -B6 'toast[.(]' -g '*.ts*' src/ | rg -B4 'catch'

# 2. Find a submit button that binds no pending state. Read every hit.
rg -n -A2 'type="submit"' -g '*.tsx' src/

# 3. Find a submit button that binds validity. This prints nothing.
rg -n 'disabled=\{!.*isValid' -g '*.tsx' src/

# 4. Find a reset in a catch or a finally. This prints nothing.
rg -n -A4 '\} (catch|finally)' -g '*.ts*' src/ | rg 'reset\('

# 5. Find a log that carries a value from a form. This prints nothing.
rg -n 'console\.(log|error|warn).*(password|token|secret|formData|values)' \
  -g '*.ts*' src/

# 6. Read the DRF answer to an empty body, and confirm the field dictionary.
curl -sS -X POST -H 'Content-Type: application/json' -d '{}' "$ORDERS_URL"

# 7. Read the answer of the deployed DRF version to a bad row in a list, and
#    confirm whether the shape is indexed or a dictionary.
curl -sS -X POST -H 'Content-Type: application/json' \
  -d '{"items":[{"sku":"ok"},{"sku":""}]}' "$ORDERS_URL"

# 8. Read the 429 answer, and confirm the Retry-After header.
curl -sSi -X POST "$ORDERS_URL"

# 9. Click the submit button twice. Confirm one request in the network panel.

# 10. Fail the network in the middle of a submit. Confirm that every value
#     stays in the form, and that the submit still works.
```

## Review checklist

- [ ] Does every submit path have a branch for a server rejection of data that
      the client accepted?
- [ ] Does every server field error reach its control, rather than a toast
      alone?
- [ ] Does `non_field_errors` reach a form-level region with the `alert` role?
- [ ] Does a message on a path that this form does not render reach the
      form-level region rather than nothing?
- [ ] Does a failed submit move focus once, to the first failed field or to the
      summary?
- [ ] Does the submit button bind `isSubmitting` rather than `isValid`?
- [ ] Does a repeated POST carry an `Idempotency-Key`?
- [ ] Does a 5xx keep every value, and leave the submit operable?
- [ ] Does a 429 read `Retry-After` and report the wait?
- [ ] Does a 409 keep the values and offer a way to see the other version?
- [ ] Does the reset run after a success alone, with the values that the server
      returned?
- [ ] Does a success invalidate the cache entry that the write changed?
- [ ] Does the code log a status and a field name, and never a field value?
- [ ] Does an Action return the submitted values with its error state?
- [ ] Does the Action verify the session inside its own body?

## Handoffs

- The schema, the resolver, the generics, and the bind of a control →
  `references/form-schema-and-field-binding.md`.
- The step, the draft, and the guard over an exit with unsaved work →
  `references/multi-step-forms-and-unsaved-work.md`.
- `normalizeApiError`, the `ApiError` shape, `Retry-After`, the retry rule, and
  the `Idempotency-Key` → `references/api-client-and-request-safety.md`.
- The DRF error envelopes, the `drf-standardized-errors` shape, and the parse
  at the boundary → `references/boundary-validation-and-api-types.md`.
- `useActionState`, `useFormStatus`, `useOptimistic`, and the rule that an
  expected error returns as state → `references/suspense-and-actions.md`.
- The order authorize, validate, mutate, invalidate, redirect inside a Server
  Action → `references/data-access-and-mutations.md`.
- The cache key that a success invalidates, the mutation state, and the
  optimistic write with its rollback →
  `references/server-state-and-query-cache.md`.
- The session that a Server Action verifies in its own body →
  `references/route-protection-and-permissions.md`. That domain holds a veto.
- The tag that a Server Action revalidates after a write →
  `references/caching-and-revalidation.md`.
- The error summary, the focus after a failed submit, and the live region that
  announces a result → `references/keyboard-focus-and-live-regions.md`. That
  domain holds a veto.
- The `aria-invalid` attribute and the `aria-describedby` that reaches the
  message → `references/semantics-and-accessible-names.md`. That file also
  holds the rule that a disabled submit button hides a refusal.
- The upload that a submit carries, and its progress →
  `references/file-upload-and-transport.md`.
- The words in an error message → `references/error-and-empty-state-copy.md`.
  The words in a confirmation → `references/interface-copy-and-voice.md`.
- The Content Security Policy over the page that holds the form, and the
  Server Action surface → domain 17 `frontend-security`. Not integrated yet.
- The MSW handler that returns a DRF 400, and the test that proves the message
  lands on the field → domain 20 `testing-and-quality`. Not integrated yet.
- The serializer, the status code, and the error envelope on the server → the
  sibling skill `django-api-contract`. This file changes nothing on the server.
