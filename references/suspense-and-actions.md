# Suspense and Actions

React 19.2.1 or later, Next.js 16.3, against a Django and DRF backend. This
file owns the boundary that renders while a value is absent, and the Action
that changes a value. The subjects are the granularity of a `<Suspense>`
boundary, the error boundary inside the tree, `use()` on a promise,
`useActionState`, `useFormStatus`, `useOptimistic`, and `<Activity>`.

The shape of the component is `references/component-composition.md`. Where the
state of a component lives is `references/state-and-effects.md`. The route files
`loading.tsx` and `error.tsx` are `references/app-router-structure.md`. The body
of the Server Action is `references/data-access-and-mutations.md`.

## Principle

A boundary decides what the user sees while a value is absent. Put the boundary
where the wait is, and not at the root.

A fallback with a different shape from the content moves the page when the
content arrives. Give the fallback the box of the content.

An expected failure is a value, and an unexpected failure is a throw. The two
need different code, and a program that treats them the same loses one of them.

An optimistic value is a promise to the user. React holds it until the Action
settles, and then the committed value replaces it.

A promise that a render creates is a new promise on each render. A component
that reads it waits forever.

## Pinned-stack depth

### Choose the granularity

| Condition | Action |
| --- | --- |
| The whole route waits for one value | One `<Suspense>` at the page, plus the `error.tsx` of the segment. |
| Independent sections finish at different times | One boundary for each section, so the fast content paints first. |
| One widget can fail, and the rest of the page stays useful | A local `<Suspense>` and a local error boundary around that widget. |

```tsx
// Wrong: one boundary holds the whole route.
// Failure: the slowest value decides the whole page. The header and the
// filters wait for the report, and a failure in the report replaces the page
// that the user can still act on.
<Suspense fallback={<FullPageSpinner />}>
  <Header />
  <Filters />
  <SlowReport />
</Suspense>
```

```tsx
// Correct: the boundary sits on the part that waits.
<>
  <Header />
  <Filters />
  <Suspense fallback={<ReportSkeleton rows={8} />}>
    <SlowReport />
  </Suspense>
</>
```

Give the fallback the same box as the content. A skeleton of eight rows for a
table of eight rows holds the layout. A spinner of a different size moves every
element below it when the data arrives, which the user reads as a fault.

### The widget boundary recovers on its own

`error.tsx` catches a failure for a whole route segment. A widget inside the
route needs a boundary in the tree, so that the failure of one panel leaves the
rest of the page usable. `react-error-boundary` supplies that component and the
hooks around it.

```tsx
// Correct: the panel fails alone, and the user retries the panel alone.
"use client"; // reason: an error boundary needs client state
import { ErrorBoundary } from "react-error-boundary";
import { Suspense } from "react";

export function RevenuePanel({ promise }: { promise: Promise<Revenue> }) {
  return (
    <ErrorBoundary
      fallbackRender={({ resetErrorBoundary }) => (
        <div role="alert">
          <p>The revenue did not load.</p>
          <button onClick={resetErrorBoundary}>Try again</button>
        </div>
      )}
    >
      <Suspense fallback={<RevenueSkeleton />}>
        <Revenue promise={promise} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

Confirm that the retry works. A boundary whose button changes nothing is a dead
end for the user. Put the error boundary outside the `<Suspense>`, so that a
throw during the retry reaches it.

### `use()` needs a promise that already exists

```tsx
// Wrong: the consumer creates the promise.
// Failure: the render calls fetchUser again, which suspends again, which
// renders again. The fallback never resolves, and the browser sends a request
// on every attempt.
function UserName({ id }: { id: string }) {
  const user = use(fetchUser(id));
  return <span>{user.name}</span>;
}
```

```tsx
// Correct: a parent creates the promise once, and the consumer reads it.
function Page({ id }: { id: string }) {
  const userPromise = useMemo(() => fetchUser(id), [id]);
  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserName userPromise={userPromise} />
    </Suspense>
  );
}

function UserName({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);
  return <span>{user.name}</span>;
}
```

The `useMemo` here is the escape hatch that
`references/state-and-effects.md` permits: the identity of the promise is the
thing that must not change. A Server Component starts the promise and passes it
as a prop, which removes the memo and the client request together;
`references/server-and-client-components.md` holds that pattern.

Two more rules bind `use()`. It accepts a call inside a condition and inside a
loop, which no other hook does. NEVER call it inside a `try` block: an error
boundary catches the rejection, or the promise handles it with `.catch`.

### The Action carries the pending state

```tsx
// Correct: the hook wires the Server Action to the form and returns the
// result, the dispatcher, and the pending flag.
// app/posts/new/new-post-form.tsx
"use client"; // reason: useActionState holds the state that the Action returns
import { useActionState } from "react";
import { createPostAction } from "../actions";

export function NewPostForm() {
  const [state, formAction, isPending] = useActionState(createPostAction, {});
  return (
    <form action={formAction}>
      <label htmlFor="title">Title</label>
      <input id="title" name="title" required />
      {state.error && <p role="alert">{state.error}</p>}
      <SubmitButton />
    </form>
  );
}
```

```tsx
// Correct: useFormStatus reads the status of the form above it. It takes no
// argument, and it returns { pending, data, method, action }.
import { useFormStatus } from "react-dom";

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? "Saving" : "Save"}
    </button>
  );
}
```

The return of `useActionState` is three values in a fixed order: the state, the
dispatcher, and the pending flag. The action takes the previous state as its
first parameter and the payload as its second. Pass the dispatcher to
`<form action>`, never the action itself, because `<form action>` calls with one
argument. Call `useFormStatus` from a component inside the `<form>`, and not
from the component that renders the `<form>`.

The body of the Server Action, and the order authorize, validate, mutate,
invalidate, redirect, is `references/data-access-and-mutations.md`.

### An expected error is state, an unexpected error throws

| The response from DRF | What the Action does | Where the user reads it |
| --- | --- | --- |
| 400, a validation error on a field | Return it in the state | Beside the field, in the form |
| 400 or 409, a rule of the business, such as a quantity that is not available | Return it in the state | Beside the control that the user can change |
| 401 or 403 | Return it in the state, and redirect where the route requires a session | The form, or the sign-in route |
| 5xx, or a network failure | Throw | The nearest error boundary |

```ts
// Correct: the known error is a value, and the unknown error is a throw.
// app/posts/actions.ts
"use server";

export type State = { error?: string; fieldErrors?: Record<string, string[]> };

export async function createPostAction(
  _prev: State,
  formData: FormData,
): Promise<State> {
  const response = await postToDjango(formData);
  if (response.status === 400) {
    const { fieldErrors } = normalizeApiError(400, await response.json());
    return { fieldErrors };
  }
  if (!response.ok) {
    throw new Error(`posts: ${response.status}`); // the boundary renders
  }
  return {};
}
```

NEVER throw a validation error. A throw replaces the form with an error
boundary, so the user loses every value that they typed and the message names
no field. The shape of the DRF error body, and the `attr` field that names the
form field, are `references/boundary-validation-and-api-types.md`. The map from
a field name to a form control is domain 11 `forms-and-validation`.

### `useOptimistic` rolls back on its own

```tsx
// Correct: the optimistic value paints at once, and React restores the
// committed value when the Action rejects.
import { startTransition, useOptimistic } from "react";

function LikeButton({
  likes,
  likeAction,
}: {
  likes: number;
  likeAction: () => Promise<number>;
}) {
  const [optimisticLikes, addOptimistic] = useOptimistic(likes);
  return (
    <button
      onClick={() =>
        startTransition(async () => {
          addOptimistic(optimisticLikes + 1);
          await likeAction();
        })
      }
    >
      {optimisticLikes} likes
    </button>
  );
}
```

Call the optimistic setter inside a transition. A DRF 400 that the Action
rejects on therefore returns the button to the last committed value without any
rollback code. Give the user an optimistic value only where the failure is
rare and the correction is visible.

### `<Activity>` keeps the state of hidden UI

React 19.2 adds `<Activity>`, with the two modes `visible` and `hidden`. A
hidden subtree keeps its state. Use it where a conditional render would remount
the subtree. A remount loses a scroll position, a half-typed value, and an open
panel. A tab set, a wizard step, and a route that the user returns to are the
common cases.

Confirm the prop name and the mode values against the installed version before
you write the code. The React team states that more modes are planned, so read
the release notes of the installed version. The choreography of a transition
between two views is domain 14 `motion-and-interaction`.

### Document metadata and resource preloading

React 19 supports `<title>`, `<meta>`, and `<link>` rendered from inside a
component. It also makes `preload`, `preinit`, `prefetchDNS`, and `preconnect`
stable. This file records that the APIs exist. The content of the metadata, and
which mechanism a Next.js route uses for it, are domain 18 `seo-and-metadata`.
Which resource to preload, and the cost of the wrong one, are domain 16
`performance-and-web-vitals`.

## Verification

```bash
# 1. Read the installed React version before you write code.
node -p "require('react/package.json').version"

# 2. List every boundary, then compare the count against the components that
#    suspend. Each suspending component needs a fallback and a reachable
#    error boundary.
rg -n '<Suspense' -g '*.tsx' .
rg -n 'ErrorBoundary' -g '*.tsx' .

# 3. Find every use() call. The promise arrives as a prop or from a memo,
#    never from a call in the same render.
rg -n '\buse\(' -g '*.tsx' src/

# 4. Find a use() call inside a try block. This must print nothing.
rg -n -B4 '\buse\(' -g '*.tsx' src/ | rg 'try \{'

# 5. Find a form that receives the action rather than the dispatcher. Read
#    every hit.
rg -n 'action=\{' -g '*.tsx' src/

# 6. Submit a form with an invalid field. The message renders beside the
#    field, and the other values stay in the form.

# 7. Fail the network, then confirm that the optimistic value returns to the
#    committed value.

# 8. Load each route on a throttled connection, and confirm that no fallback
#    moves the content when it resolves.
```

## Review checklist

- [ ] Does every boundary sit on the part of the page that waits, rather than
      at the root?
- [ ] Does every suspending component have a `<Suspense>` fallback and a
      reachable error boundary?
- [ ] Does every fallback occupy the same box as the content that replaces it?
- [ ] Does every widget that can fail alone carry its own error boundary?
- [ ] Does every error boundary offer a retry that works?
- [ ] Does every promise that `use()` reads come from a prop, a memo, or a
      module, never from a call in the same render?
- [ ] Is every `use()` call outside a `try` block?
- [ ] Does every `<form action>` receive the dispatcher from `useActionState`
      rather than the action itself?
- [ ] Does `useFormStatus` run in a component inside the `<form>`?
- [ ] Does every expected error return as state rather than throw?
- [ ] Does every unexpected error throw to a boundary rather than return?
- [ ] Does every validation message render beside its field, with the typed
      values still in the form?
- [ ] Does every optimistic update run inside a transition?
- [ ] Does hidden UI that must keep its state use `<Activity>` rather than a
      conditional render?

## Handoffs

- The `loading.tsx` and `error.tsx` route files, and the `reset()` that the
  segment boundary exports → `references/app-router-structure.md`.
- The `"use client"` directive on a boundary, and the promise that a Server
  Component streams to the client →
  `references/server-and-client-components.md`.
- The body of the Server Action, and the order authorize, validate, mutate,
  invalidate, redirect → `references/data-access-and-mutations.md`.
- The DRF error envelope, the pagination envelope, and the parse at the
  boundary → `references/boundary-validation-and-api-types.md`.
- `normalizeApiError`, the `ApiError` shape that it returns, and the
  `fieldErrors` that a 400 produces →
  `references/api-client-and-request-safety.md`.
- The cache entry that a mutation invalidates, and read-your-writes →
  `references/caching-and-revalidation.md`.
- Where the state of a component lives, and the memoisation rule →
  `references/state-and-effects.md`.
- The mutation state, the retry, and the query that a client refetches →
  domain 06 `data-fetching-and-state`. Not integrated yet.
- The live region that announces a pending state, and the focus that moves to
  an error message → domain 10 `accessibility-wcag`. Not integrated yet. That
  domain holds a veto.
- The resolver, the field array, and the multi-step flow → domain 11
  `forms-and-validation`. Not integrated yet.
- The transition between two views, and the duration of an animation →
  domain 14 `motion-and-interaction`. Not integrated yet.
- The words in an error message, an empty state, and a pending label →
  domain 15 `ux-writing-and-content-design`. Not integrated yet.
- The content of the document metadata → domain 18 `seo-and-metadata`. Not
  integrated yet.
