# Data rights and privacy surfaces

Next.js 16.3, React 19.2, TanStack Query 5.101, Django and DRF. This file owns
the parts of the interface that the law makes the product show. The subjects are
the five rights that a reader may exercise, and the flow that starts each one
against Django. They also include the export that a reader takes away, the
deletion that nothing undoes, and the retention window that the interface states.
The last subjects are the policy page and its version, the age gate, and the
place where the data sits.

The consent record, the gate, and the cookie inventory are
`references/consent-gate-and-cookie-inventory.md`. The event and the analytics
module are `references/event-taxonomy-and-tracking-plan.md`. The words in a
notice, and the message that a refusal produces, are
`references/interface-copy-and-voice.md` and
`references/error-and-empty-state-copy.md`.

## Principle

A right that no control exercises is a right on paper. The product owes the
reader a path, and the path is a part of the interface.

An export and a deletion are slow, because the server reads many tables. Treat
each one as a job with a state, and never as a request that answers at once.

A deletion that nothing undoes needs a control that nobody presses by accident.
The cost of a wrong press is the whole account.

A retention window that the interface never states is a promise that the reader
cannot check. State the window in words beside the data.

A policy page is a versioned product surface. The consent record names a version,
so the page must be able to state which one the reader read.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The right, and the surface that serves it

GDPR Articles 12 to 22 state the rights below. Each row names the surface that
the frontend renders, and the Django endpoint behind it.

| The right | What the reader does | The surface | The state that the interface shows |
| --- | --- | --- | --- |
| Access and portability | Asks for a copy of their data | A control in the account settings | Requested, in progress, ready, expired |
| Rectification | Corrects a wrong value | The account form that already exists | The four states of a form submit |
| Deletion | Ends the account and its data | A separate, confirmed control | Requested, in a grace window, deleted |
| Objection | Refuses one purpose, and keeps the account | The consent control, and a purpose list | The current answer, and the time of it |
| Restriction | Asks the product to hold the data and stop the use | A control beside the objection, where the product supports it | Requested, in effect |

NEVER build a right into a support form alone. A form that a person reads is a
delay, and it gives no state back to the reader.

`references/form-schema-and-field-binding.md` owns the account form that serves
the rectification. This file owns the four rows beside it.

### The export is a job, and the interface holds its state

```tsx
// src/app/(app)/settings/privacy/export-panel.tsx
// Correct: the control starts a job, and the panel renders each state of it.
"use client"; // reason: the panel polls a job state and renders a mutation

import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { exportRequestOptions, requestExport } from "@/lib/dal/privacy";

export function ExportPanel() {
  const client = useQueryClient();
  const request = useQuery(exportRequestOptions());
  const start = useMutation({
    mutationFn: requestExport,
    onSuccess: () => client.invalidateQueries({ queryKey: ["privacy", "export"] }),
  });

  if (request.isPending) return <ExportSkeleton />;
  if (request.isError) return <ExportError onRetry={() => request.refetch()} />;

  switch (request.data?.status) {
    case "running":
      return <p>We are building your copy. This takes up to 24 hours.</p>;
    case "ready":
      return <a href={request.data.downloadPath} download>Download your data</a>;
    default:
      return (
        <button type="button" onClick={() => start.mutate()} disabled={start.isPending}>
          Request a copy of my data
        </button>
      );
  }
}
```

```tsx
// Wrong: the control awaits the whole export.
// Failure: the account holds 40 000 rows, so the endpoint runs for two minutes.
// The proxy closes the request at 60 seconds, the reader sees a failed request,
// and they press the control again. The server then builds the file twice.
const onClick = async () => {
  const file = await fetch("/api/privacy/export/").then((r) => r.blob());
  saveAs(file);
};
```

Four states cover the panel, and the code renders each one.
`references/server-state-and-query-cache.md` owns the key, the cache entry, and
the four states. This file owns the job states that the panel maps onto them.

The download itself is a file that leaves the application for a reader.
`references/served-content-and-downloads.md` owns the disposition header, the
separate origin, and the expiry of the link.

`references/data-access-and-mutations.md` owns the module that holds the call.
The DRF endpoint, its status codes, and the shape of the job record belong to
the backend.

### The deletion states its scope, and it asks twice

```tsx
// Correct: the reader types the confirmation, and the copy names the scope.
"use client"; // reason: the control compares the typed value against the account

export function DeleteAccountForm({ email }: { email: string }) {
  const [typed, setTyped] = useState("");

  return (
    <form action={requestDeletion}>
      <h2>Delete this account</h2>
      <p>
        This removes your account, your 42 projects, and every file in them. We
        hold the record for 30 days, and then it is gone.
      </p>
      <label htmlFor="confirm">Type {email} to confirm</label>
      <input id="confirm" name="confirm" value={typed} onChange={(e) => setTyped(e.target.value)} />
      <button type="submit" disabled={typed !== email}>Delete my account</button>
    </form>
  );
}
```

```tsx
// Wrong: one press ends the account.
// Failure: the reader presses the control beside "Change password" and loses
// every project. The copy names no scope, so they did not know what the press
// covered, and no window lets them undo it.
<button onClick={() => deleteAccount()}>Delete</button>
```

Three rules hold. The copy names what the deletion covers, in counted nouns. The
control needs a typed confirmation that the reader cannot press by accident. The
interface states the grace window, and it states what happens at the end of it.

`references/interface-copy-and-voice.md` owns the words on the control and in the
body. `references/keyboard-focus-and-live-regions.md` owns the focus that a
confirmation dialog takes and returns. That domain holds a veto.

WARNING: a deletion signs the reader out. Clear the query cache and the analytics
identity in the same step, or the next reader on that device inherits both.
`references/server-state-and-query-cache.md` owns the cache clear, and
`references/event-taxonomy-and-tracking-plan.md` owns `resetIdentity`.

### The retention window is a sentence

State the window beside the data that it covers, in the interface, and not in
the policy page alone.

| The data | Where the sentence belongs | An example of the sentence |
| --- | --- | --- |
| A deleted account | The confirmation, before the press | We hold the record for 30 days, and then it is gone. |
| An export file | The panel that offers the download | The link works for 7 days. |
| An activity log | The head of the log view | This view holds the last 90 days. |
| An analytics event | The privacy page | We keep a product event for 14 months. |

A window that the interface never states forces the reader to read a policy page
to learn what the product does with their work. State it where they are.

`references/error-and-empty-state-copy.md` owns the sentence in an empty view.
This file owns the four rows above.

### The policy page renders the inventory, and it names its version

```tsx
// src/app/(marketing)/cookies/page.tsx
// Correct: one list produces the page, and the version ties to the record.
import { COOKIE_INVENTORY } from "@/lib/cookie-inventory";
import { POLICY_VERSION } from "@/lib/consent";

export default function CookiePolicyPage() {
  return (
    <main>
      <h1>Cookies on this site</h1>
      <p>This policy is version {POLICY_VERSION}.</p>
      <table>
        <caption>Every cookie that this site sets</caption>
        <thead>
          <tr><th scope="col">Name</th><th scope="col">Purpose</th><th scope="col">Days</th><th scope="col">Category</th></tr>
        </thead>
        <tbody>
          {COOKIE_INVENTORY.map((cookie) => (
            <tr key={cookie.name}>
              <td>{cookie.name}</td>
              <td>{cookie.purpose}</td>
              <td>{cookie.days}</td>
              <td>{cookie.category}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </main>
  );
}
```

```tsx
// Wrong: the page is prose, and a release changed the cookies.
// Failure: the application dropped one vendor and added another six months ago.
// The page still names the old one and misses the new one, so a public claim of
// the product is false and the browser holds the evidence.
<p>We use a session cookie and one analytics cookie.</p>
```

The policy page, the terms, and the data-processing notice are product surfaces.
Each one carries a version, and a purpose change moves that version.
`references/consent-gate-and-cookie-inventory.md` states that a moved version
asks the consent question again.

`references/data-table-and-server-driven-state.md` owns a dense table with a
server behind it. A static list of ten rows needs none of that machinery.
`references/semantics-and-accessible-names.md` owns the `caption` and the scope
of a header cell. That domain holds a veto.

`references/crawl-and-index-control.md` owns whether a crawler may index the
page. A policy page is indexable, and a settings page is not.

### The age gate, where the product needs one

Build an age gate only where a stated requirement names one. A gate that no
requirement asks for collects a birth date, which is personal data that the
product then holds with no purpose.

Two rules hold where the gate exists. It asks for the least that answers the
question, which is a year rather than a full date. It records the answer in the
same record as the consent, so one clear removes both.

### The place where the data sits

The reader may ask where their data is. A hosted vendor stores it in a region
that the vendor chooses, and that region may differ from the region of the
product.

State the answer on the privacy page. Name each processor, name what it receives,
and name the region where it holds the data. A self-hosted tool makes the row
short, and it is one more reason to prefer one.

`references/event-taxonomy-and-tracking-plan.md` owns the vendor table and the
self-hosted choice. This file owns the disclosure.

### The libraries

This domain needs no package. The surfaces are routes, forms, and a table, and
the repository already holds the material for each one.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `@tanstack/react-query` | The export job state, and the poll behind it. `references/server-state-and-query-cache.md` owns the key. | 5.101 | Current | TanStack, active | None |
| Recommend | The account form that the project already has | The rectification needs no second surface. | — | — | — | — |
| Reject | A package that generates a privacy policy | The text is a legal document of this product, and a generated one states things that the product does not do. | — | — | — | — |
| Reject | A support form as the only path to a right | It gives the reader no state, and it adds a person to every request. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| An export request fails after a minute | The control awaits the whole build | Request an export on a large account, and read the network panel | Start a job, and poll its state |
| The server builds one export three times | The failed request gave no state, so the reader pressed again | Read the job table for one account | Render the running state, and disable the control |
| A reader deletes their account by accident | One press ends it, and no copy names the scope | Read the control and its neighbors | Add a typed confirmation, and name the scope in counted nouns |
| The next reader on a device sees the previous account | The deletion cleared no cache and no identity | Delete an account, and read the query cache and the vendor identity | Clear both in the same step |
| The cookie page names a cookie that nothing sets | The page is prose | Read `document.cookie` against the page | Render the page from the inventory |
| The consent banner never returns after a purpose change | The policy version did not move | Compare the version against the previous release | Move `POLICY_VERSION` for a purpose change |
| A reader cannot find how to get their data | The right lives in a support article | Walk the settings from the account menu | Put a control in the account settings |
| A crawler indexes the account settings | No route rule covers the segment | Request the route, and read the robots rule | `references/crawl-and-index-control.md` owns the rule |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next.js 16 makes `params` and `searchParams` async | A settings page that reads either one with no `await` | `references/app-router-structure.md` owns the await |
| React 19 renamed `useFormState` to `useActionState` | `rg -n 'useFormState\b' src/` reports a hit on the deletion form | `references/suspense-and-actions.md` owns the hook |
| TanStack Query 5 takes an object argument | `useQuery(key, fn)` in the export panel | Take the object form, and the `queryOptions` factory |

## Verification

```bash
# 1. Confirm that a control exists for each right. Expect four routes.
rg -l 'export|deletion|rectif|objection' src/app/**/settings

# 2. Confirm that the export is a job with a state. Expect the poll.
rg -n -A6 'exportRequestOptions' src/lib/dal/privacy.ts

# 3. Find a deletion with one press. Each hit is a defect.
rg -n -B6 'deleteAccount|requestDeletion' src/ | rg -v 'confirm'

# 4. Confirm that the deletion clears the cache and the identity.
rg -n -A8 'requestDeletion' src/ | rg 'clear\(\)|resetIdentity'

# 5. Confirm that the policy page renders the inventory. Expect the import.
rg -n 'COOKIE_INVENTORY' src/app

# 6. Find a cookie list written as prose. Read every hit.
rg -n -i 'we use .*(cookie|analytics)' -g '*.tsx' src/app

# 7. Confirm that the page states its version. Expect the constant.
rg -n 'POLICY_VERSION' src/app

# 8. Confirm that the settings routes carry a robots rule.
rg -n -A4 'settings' src/app/robots.ts

# 9. Request an export on a large account. The control answers at once, and the
#    panel then reports the running state.

# 10. Complete a deletion on a test account. The application signs out, the
#     query cache is empty, and the vendor identity is reset.

# 11. Read the privacy page. Each processor, what it receives, and its region
#     appear on it.

# 12. Run the axe pass over the cookie policy table and the deletion form.
pnpm test:a11y
```

## Review checklist

- [ ] Does a control in the account settings start an export?
- [ ] Does the export run as a job, with a state that the panel renders?
- [ ] Does the panel render all four states of the query?
- [ ] Does the download link state how long it works?
- [ ] Does the deletion control name what it covers, in counted nouns?
- [ ] Does the deletion need a typed confirmation?
- [ ] Does the interface state the grace window and what follows it?
- [ ] Does a deletion clear the query cache and reset the analytics identity?
- [ ] Does the interface state a retention window beside the data that it covers?
- [ ] Does the cookie policy page render the inventory file rather than prose?
- [ ] Does the policy page state its version, and does that version match the
      consent record?
- [ ] Does the privacy page name each processor, what it receives, and its
      region?
- [ ] Is an age gate absent unless a stated requirement names one?
- [ ] Does a robots rule keep the settings routes out of the index?

## Handoffs

- The consent record, the policy version that it names, the gate, and the cookie
  inventory file → `references/consent-gate-and-cookie-inventory.md`.
- The event, the analytics module, `resetIdentity`, and the vendor table →
  `references/event-taxonomy-and-tracking-plan.md`.
- The words on a control, the counted noun, and the sentence in an empty view →
  `references/interface-copy-and-voice.md` and
  `references/error-and-empty-state-copy.md`.
- The key, the cache entry, the four states, and the clear on a sign-out →
  `references/server-state-and-query-cache.md`. The module that holds the call
  and the Server Action order → `references/data-access-and-mutations.md`.
- The account form, its schema, and the server error map →
  `references/form-schema-and-field-binding.md` and
  `references/form-submission-and-server-errors.md`. The pending state of a
  submit → `references/suspense-and-actions.md`.
- The download itself, its disposition header, its separate origin, and its
  expiry → `references/served-content-and-downloads.md`.
- The focus that a confirmation dialog takes and returns, and the announcement of
  a state change → `references/keyboard-focus-and-live-regions.md`. The
  `caption`, the header scope, and the accessible name →
  `references/semantics-and-accessible-names.md`. That domain holds a veto.
- The robots rule over a settings route, and the index decision for a policy page
  → `references/crawl-and-index-control.md`.
- The route group and the segment that hold these pages →
  `references/app-router-structure.md`.
- The browser test that walks an export and a deletion →
  `references/end-to-end-journeys-and-flake-control.md`.
- The endpoint that starts each job, its status codes, the shape of the job
  record, and the erasure → the backend and its security review. This file
  owns the surface that starts each one and the state that it renders.
- The background worker that builds the export → the backend.
