# Error and empty state copy

Next.js 16.3, React 19.2.6 or later, TypeScript 5.9, `next-intl` 4.13.6,
`drf-standardized-errors`, WCAG 2.2 Level AA. This file owns what a person reads
when a request fails, and when a view holds nothing. The subjects are the map
from an error code onto a message, the control beside it, and the digest that
support needs. They also include the copy of `error.tsx`, of
`global-error.tsx`, and of `not-found.tsx`. The last subjects are the three
empty states and the channel that carries a failure.

The words on a control and the voice behind them are
`references/interface-copy-and-voice.md`. The key and the count are
`references/message-catalog-and-plurals.md`.

## Principle

A failure is a moment where the reader has lost control of the product. The
message returns that control, or it confirms the loss.

Three facts return it. They are what failed, what the reader does next, and the
control that does it. A message with the first fact alone is a report to
nobody.

The text that a server produces is written for the people who run the server.
It names a serializer, a constraint, or a table. It reaches the reader as noise,
or as a leak.

An empty view has three causes, and the reader cannot tell them apart. A list
that never held a row, a filter that excluded every row, and a request that
failed all render zero rows.

A message that only a toast carries expires. The reader who looked at the
keyboard has no second chance at it.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark.

### The map from a code onto a message

```tsx
// Wrong: the view renders the string that the server sent.
// Failure: the reader reads "Ensure this field has no more than 32
// characters." with no field named. The text names a serializer rule, it
// arrives in one language, and no locale can change it.
<p>{apiError.message}</p>
```

```ts
// Correct: src/features/orders/messages.ts maps the code onto a key, and an
// unknown code falls to a safe generic key.
import type { ApiError } from "@/lib/api/errors";

const MESSAGE_BY_CODE: Record<string, string> = {
  required: "errors.field.required",
  invalid: "errors.field.invalid",
  max_length: "errors.field.tooLong",
  unique: "errors.field.duplicate",
  throttled: "errors.request.throttled",
};

export function messageKey(error: ApiError): string {
  return MESSAGE_BY_CODE[error.code] ?? "errors.request.generic";
}
```

`references/api-client-and-request-safety.md` owns `ApiError`, and it states
that `code` holds the DRF code and never the message. This file owns the map
from that code onto a key that the reader can act on.

`ApiError.message` is the fallback that the normalizer built for a person who
reads a log. NEVER render it as the primary message of a view.

Two envelope shapes reach the client. The `drf-standardized-errors` shape
carries a machine `code` beside each error, and the native DRF shape carries a
human string alone. Key the map off `code` where the backend sends that shape.
Where the backend sends the native shape, key the map off the status and the
field name, and record that those messages are less exact. The sibling skill
`django-api-contract` owns the decision to publish the standardized envelope.

CAUTION: a renamed code on the server drops every message of that kind to the
generic key, and the reader loses the specific step. Assert in a test that every
code of the generated schema is a key of the map. The test fails the build on
the rename. `references/openapi-schema-and-codegen.md` owns the generated enum.

| The failure | What the message states | The control beside it |
| --- | --- | --- |
| A field is empty or malformed | The rule that the value broke | The focus on that field |
| A 401 | The session ended | A control that signs in again |
| A 403 | The permission that the account lacks | The person or the route that grants it |
| A 404 on a record | The record is gone | The list that holds the rest |
| A 409 | Another change landed first | A control that reads the record again |
| A 429 | The product accepted too many requests | The period that `Retry-After` states |
| A 5xx | The product failed, and the reader did not | A retry control, and the digest |
| A network failure | The request never reached the product | A retry control |

`references/form-submission-and-server-errors.md` owns the map from a field
path onto a control, and the region that holds a form-level message. This file
owns the words that each map produces.

### The production error object carries no message

```tsx
// Wrong: the boundary renders the message of the error.
// Failure: a production build replaces the message of a server error with a
// generic string, so this element renders that placeholder. The reader learns
// nothing, and the boundary offers no way forward.
"use client"; // reason: an error boundary needs client state

export default function Error({ error }: { error: Error & { digest?: string } }) {
  return <div>Oops. {error.message}</div>;
}
```

```tsx
// Correct: a mapped message, a control that recovers, and the digest behind a
// disclosure.
"use client"; // reason: an error boundary needs client state
import { useTranslations } from "next-intl";

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  const t = useTranslations("errors.boundary");
  return (
    <div role="alert">
      <h2>{t("title")}</h2>
      <p>{t("body")}</p>
      <button type="button" onClick={reset}>{t("retry")}</button>
      {error.digest === undefined ? null : (
        <details>
          <summary>{t("detailsLabel")}</summary>
          <code>{error.digest}</code>
        </details>
      )}
    </div>
  );
}
```

Next.js forwards a generic message and a `digest` property to the client in a
production build. The reason is a security precaution against a leak of the
exception text. The message is therefore readable in development and empty of
content in production.

The `digest` ties the screen of the reader to a line in the server log. It is
the one identifier that a support request can quote. Put it behind a
disclosure, so it reaches support and never leads the message.

`references/server-and-client-components.md` owns the directive on a boundary
file, and `references/app-router-structure.md` owns the route files.
`references/exposed-endpoints-and-destinations.md` owns the rule that exception
text must not reach the client. Domain 21 `observability-and-resilience` owns
the lookup from a digest into a log, and it is not integrated yet. The sibling
skill `secure-code-auditor` owns the server-side guarantee that no response body
carries a stack trace.

### `global-error.tsx` runs with no provider above it

```tsx
// Wrong: the root boundary reads a translation.
// Failure: global-error.tsx replaces the root layout, so the provider that the
// layout mounted is gone. The hook throws inside the boundary that must not
// throw, and the reader gets the default error page of the browser.
"use client";
import { useTranslations } from "next-intl";

export default function GlobalError() {
  const t = useTranslations("errors");
  return <html><body><p>{t("root")}</p></body></html>;
}
```

```tsx
// Correct: the last-resort copy is literal, and the element tree is complete.
"use client"; // reason: a root error boundary needs client state

export default function GlobalError({ reset }: { reset: () => void }) {
  return (
    <html lang="en">
      <body>
        <div role="alert">
          <h1>This page did not load.</h1>
          <p>Reload the page. Contact support if it fails again.</p>
          <button type="button" onClick={reset}>Reload</button>
        </div>
      </body>
    </html>
  );
}
```

`global-error.tsx` replaces the root layout, so every provider that the layout
mounted is absent. The translation provider, the theme provider, and the query
client are all gone.

Write the copy of this one file as a literal string, or mount the locale again
inside the file. This file is the documented exception to the rule that no
string sits in the markup, and
`references/message-catalog-and-plurals.md` states both.

The `<html>` element needs its `lang` attribute here as well, because the root
layout no longer supplies one.

### The three empty states

```tsx
// Wrong: one string covers three different situations.
// Failure: a reader who filtered to zero rows reads that the product holds no
// invoices, and looks for a defect. A reader whose request failed gets no
// retry control, and the view looks correct.
{invoices.length === 0 ? <p>No data available</p> : <InvoiceRows rows={invoices} />}
```

```tsx
// Correct: a union names the case, and each case carries its own action.
import Link from "next/link";
import { useTranslations } from "next-intl";
import { assertNever } from "@/lib/assert-never";

type EmptyCase =
  | { kind: "never" }
  | { kind: "filtered"; clearFilters: () => void }
  | { kind: "failed"; retry: () => void };

export function InvoicesEmpty({ state }: { state: EmptyCase }) {
  const t = useTranslations("invoices.empty");
  switch (state.kind) {
    case "never":
      return (
        <div>
          <p>{t("never.body")}</p>
          <Link href="/invoices/new">{t("never.action")}</Link>
        </div>
      );
    case "filtered":
      return (
        <div>
          <p>{t("filtered.body")}</p>
          <button type="button" onClick={state.clearFilters}>
            {t("filtered.action")}
          </button>
        </div>
      );
    case "failed":
      return (
        <div role="alert">
          <p>{t("failed.body")}</p>
          <button type="button" onClick={state.retry}>{t("failed.action")}</button>
        </div>
      );
    default:
      return assertNever(state);
  }
}
```

| The case | What the reader learns | The action beside it |
| --- | --- | --- |
| The list never held a row | What the list holds once it fills | The control that makes the first row |
| A filter excluded every row | Which filter to relax | The control that clears the filters |
| The request failed | The product failed to read the list | The control that repeats the request |

`references/server-state-and-query-cache.md` owns the four states of a data
view, and it states that the empty state is designed. This file states that the
empty state is three states.

`references/push-transport-and-connection.md` adds a fifth state. An empty list
under a dropped connection is not an empty list. Write that copy as a degraded
state, and never as the case where the list never held a row.

### The route that has no record, and the route that refuses

A `not-found.tsx` file renders for a `notFound()` call and for a path that no
segment matches. State what the reader looked for, and offer the route back.
NEVER blame the reader for a link that the product produced.

A permission failure needs an explanation. A blank page after a 403 becomes a
support request, because the reader cannot act on it. State the permission that
the account lacks, and name the person or the route that grants it.
`references/route-protection-and-permissions.md` owns the gate that produces the
403, and that domain holds a veto.

CAUTION: a message about a record that the reader may not read confirms that
the record exists. "Invoice 4021 belongs to another account" is a disclosure.
Write one message for a missing record and for a refused record.

### The channel that carries a failure

```tsx
// Wrong: a toast is the only report of a failed submit.
// Failure: the toast disappears after a few seconds. A reader who looked at
// the keyboard reads nothing, and no field on the form carries the error.
onError: (error) => toast(t(messageKey(error)))
```

```tsx
// Correct: the message stays in the view, and one announcer reads it once.
onError: (error) => {
  setFormError(t(messageKey(error)));
  announce(t(messageKey(error)));
}
```

A toast reports a result that the reader may ignore. It never reports a failure
that the reader must act on.

Write the failure into the view, beside the control that produced it. Where a
toast repeats that text, the reader reads one message twice.

`references/keyboard-focus-and-live-regions.md` owns the announcer, the
politeness level, and the rule that the region mounts before its message. That
domain holds a veto. This file owns the choice of channel and the words in it.

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| The production error page carries no message | The boundary renders `error.message`, which the build replaced | Run the production build, and trigger a server error | Render a mapped key, and put the digest behind a disclosure |
| The root boundary throws inside itself | A provider hook runs where the root layout is gone | Trigger a throw in the root layout of a production build | Write literal copy, or mount the locale in the file |
| Every failure reads the same | A renamed code falls through to the generic key | Assert the generated enum against the keys of the map | Add the code, and keep the test |
| The reader looks for a defect in an empty list | One string covers three empty cases | Filter one list to zero rows | Split the case into a union, and write three messages |
| A failed request looks like an empty list | The view holds no failed case | Break the request in the network panel | Add the failed case and its retry control |
| The reader cannot act on a message | The message states the cause, and offers no control | Read each message, and look for the control beside it | Add the control that fixes the stated cause |
| The message disappears before the reader reads it | A toast is the only channel | Complete the flow with the eyes on the keyboard | Render the message in the view |
| A support request carries no identifier | The digest never reaches the screen | Read the production error page | Put the digest behind a disclosure |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next.js 16.3 forwards a generic message and a `digest` for a server error in production | `rg -n 'error\.message' -g 'error.tsx' -g 'global-error.tsx' src/` reports a hit | Render a mapped key, and put the digest behind a disclosure |
| React 19 returns the error of an Action inside the state of `useActionState` | `rg -n 'useFormState\b' src/` reports a hit | `references/suspense-and-actions.md` owns the hooks and the expected-error rule |

## Verification

```bash
# 1. Find a rendered server message. This prints nothing.
rg -n 'error\.message|apiError\.message|\.detail\}' -g '*.tsx' src/

# 2. Find an apologetic generic message. Each hit is a defect.
rg -n -i 'oops|something went wrong|unknown error|an error occurred' src/

# 3. Find one string over an empty list. Read every hit.
rg -n -i 'no data|no results|nothing here|no items' -g '*.tsx' -g '*.json' src/

# 4. Find a route error file that offers no control.
rg --files-without-match 'reset' -g 'error.tsx' -g 'global-error.tsx' src/

# 5. Find a message hook inside the root boundary. This prints nothing.
rg -n 'useTranslations|getTranslations' -g 'global-error.tsx' src/

# 6. Read the code map beside the generated enum. Every code of the schema is a
#    key of the map.
rg -n -A20 'MESSAGE_BY_CODE' src/

# 7. Build for production, and trigger a server error. The page states the
#    cause, offers a control, and holds the digest behind a disclosure.
pnpm build && pnpm start

# 8. Filter one list to zero rows. The copy names the filter, and a control
#    clears it.

# 9. Break one request in the network panel. The view states the failure, and a
#    control repeats the request.

# 10. Open one refused route. The page states the permission, and it names no
#     record that the account may not read.
```

## Review checklist

- [ ] Does every failure message state what failed and what the reader does
      next?
- [ ] Does every failure message sit beside the control that performs that step?
- [ ] Does the map from a code onto a key cover every code of the generated
      schema?
- [ ] Does an unknown code fall to a generic key, rather than to the server
      text?
- [ ] Is `ApiError.message` absent from every rendered view?
- [ ] Is `error.message` absent from `error.tsx` and from `global-error.tsx`?
- [ ] Does each error boundary hold the digest behind a disclosure?
- [ ] Does `global-error.tsx` render literal copy, with no provider hook in it?
- [ ] Does `global-error.tsx` render its own `<html lang>` and `<body>`?
- [ ] Is every empty view one of the three cases, with its own message?
- [ ] Does each empty case carry the control that its message names?
- [ ] Is a dropped connection reported as a degraded state, rather than as an
      empty list?
- [ ] Does the refused route state the permission that the account lacks?
- [ ] Does a message about a missing record name no record that the account may
      not read?
- [ ] Is a toast absent as the only channel for a failure?
- [ ] Does no toast repeat a message that the view already holds?

## Handoffs

- `ApiError`, its `code` field, the normalizer, and the retry rule →
  `references/api-client-and-request-safety.md`.
- The envelope shapes on the wire, and the union over them →
  `references/boundary-validation-and-api-types.md`.
- The map from a field path onto a control, and the form-level region →
  `references/form-submission-and-server-errors.md`.
- The generated enum of error codes, and the drift check over it →
  `references/openapi-schema-and-codegen.md`.
- The four states of a data view, and the designed empty state →
  `references/server-state-and-query-cache.md`.
- The fifth state that a dropped connection adds →
  `references/push-transport-and-connection.md`.
- The route files, and the boundary that each one holds →
  `references/app-router-structure.md`.
- The directive on a boundary file →
  `references/server-and-client-components.md`.
- The fallback, its shape, and the React 19 Action behind a pending state →
  `references/suspense-and-actions.md`.
- The gate that produces a 401 or a 403, and the redirect after it →
  `references/route-protection-and-permissions.md`. That domain holds a veto.
- The announcer, the politeness level, and the region that mounts first →
  `references/keyboard-focus-and-live-regions.md`. That domain holds a veto.
- The words on the control inside a message, and the voice →
  `references/interface-copy-and-voice.md`.
- The key, and the ICU body behind each message →
  `references/message-catalog-and-plurals.md`.
- The rule that exception text must not reach the client →
  `references/exposed-endpoints-and-destinations.md`. That domain holds a veto.
- The capture of the error, and the lookup from a digest into a log → domain 21
  `observability-and-resilience`. Not integrated yet.
- The guarantee that no response body carries exception text or a stack trace →
  sibling skill `secure-code-auditor`.
- The `code` field as a published contract, and the rename that breaks the map
  → sibling skill `django-api-contract`.
