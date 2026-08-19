# Route protection and permissions

Next.js 16.3, React 19.2.6 or later, against a Django and DRF backend. This
file owns the code that decides whether a request may proceed. The subjects
are the enforcement layers, the data access layer that reads the session, and
the gate inside a Server Action. They also include the answer to a missing
session, the validated redirect, and the permission model that the UI
reflects.

The credential itself, its storage, its refresh, and the logout are
`references/session-and-token-lifecycle.md`. `proxy.ts` and the route files
are `references/app-router-structure.md`. The module that calls the backend,
and the order inside a Server Action, are
`references/data-access-and-mutations.md`.

## Principle

Authorization is a decision of the server. The interface reflects that
decision, and a reflection is not an enforcement.

A gate that runs before the router resolves the route reaches no data layer,
and a forged header can skip it. Put the gate where the data is.

A public endpoint is public whatever renders it. A function that the network
can reach must check the caller itself.

The absence of a control is a comfort, not a boundary. Hide a control that the
user may not use, and expect the request anyway.

A value that one identity may read must never enter a payload that another
identity receives.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The four layers, in trust order

| Layer | Its job | May it be the only gate? |
| --- | --- | --- |
| `proxy.ts` | A redirect on the presence of a cookie, as a comfort for the user | No. CVE-2025-29927 and CVE-2026-64642 both skip it. |
| The page, on the server | A check that stops the render, so the user meets the correct route | No. It is frontend code, and it reads what the frontend has. |
| The data access layer, through `verifySession()` and React `cache()` | The real gate of the frontend, before any data returns | Yes, for the data that the frontend itself returns |
| The Django permission | The authoritative decision | Yes. This is the boundary of record. |

Read the table downward, and add each layer. The layers do not replace one
another. The data access layer is the gate that the frontend owns, and the
Django permission is the gate that decides. A rule that stops at the data
access layer is a rule that trusts frontend code with a security decision.

`secure-code-auditor` owns the DRF permission class and every server-side check.
This file owns the three frontend layers above it.

### The data access layer reads the session once

```tsx
// Wrong: the page trusts the redirect that proxy.ts performed.
// Failure: CVE-2025-29927 and CVE-2026-64642 are both bypasses of that file.
// A request that carries the forged header reaches this page with no session,
// the protected data renders, and the RSC payload streams it to the attacker.
export default async function DashboardPage() {
  return <Dashboard />;
}
```

```ts
// Correct: src/lib/dal/session.ts holds one memoised read for each request.
import "server-only";
import { cache } from "react";
import { cookies } from "next/headers";
import { z } from "zod";

// This response decides every later gate, so the value is proven at the
// boundary rather than taken on trust.
const SessionSchema = z.object({
  id: z.number(),
  email: z.email(),
  role: z.string(),
  permissions: z.array(z.string()),
});

export type Session = z.infer<typeof SessionSchema>;

export const verifySession = cache(async (): Promise<Session | null> => {
  const store = await cookies(); // async in Next 16
  if (!store.has("sessionid")) return null;

  const response = await fetch(`${process.env.DJANGO_URL}/api/auth/me/`, {
    headers: { cookie: store.toString() },
    cache: "no-store", // per-user data: state the intent, never inherit it
    signal: AbortSignal.timeout(10_000),
  });
  if (!response.ok) return null;

  const parsed = SessionSchema.safeParse(await response.json());
  return parsed.success ? parsed.data : null;
});
```

```tsx
// Correct: app/(app)/dashboard/page.tsx gates the page, and the session
// arrives as a prop.
import { redirect } from "next/navigation";
import { verifySession } from "@/lib/dal/session";

export default async function DashboardPage() {
  const session = await verifySession();
  if (session === null) redirect("/login?next=/dashboard");
  return <Dashboard session={session} />;
}
```

React `cache()` gives one read for each request. Ten server components that
call `verifySession()` in one render produce one call to Django. Without it,
each component sends its own request, and the render pays for all of them.

Four rules bind the module. It imports `server-only`, so a client import
fails the build. It states the cache intent on the `fetch`, because a session
read must never enter a shared cache. It parses the response, because the
session is the most trust-sensitive value that the frontend reads. It returns
`null` rather than throwing, so each caller decides its own answer.
`references/data-access-and-mutations.md` owns the rest of the rules for a
data access module, and
`references/boundary-validation-and-api-types.md` rules on the parse that a
trust-sensitive response needs.

### Gate the page, not the layout

A layout renders once and then persists across a navigation inside its
segment. A check in a layout therefore runs on the first render and not on
each later one. Put the check in the page.

```tsx
// Wrong: only the layout verifies the session.
// Failure: the layout does not re-render on a client-side navigation inside
// the segment, so a session that ends mid-visit is never noticed. The user
// keeps every route under the layout until a full reload.
export default async function AppLayout({ children }: { children: React.ReactNode }) {
  const session = await verifySession();
  if (session === null) redirect("/login");
  return <Shell>{children}</Shell>;
}
```

```tsx
// Correct: each page under the segment verifies, and the memo makes the cost
// one call for each request.
export default async function SettingsPage() {
  const session = await verifySession();
  if (session === null) redirect("/login?next=/settings");
  return <Settings session={session} />;
}
```

Keep the route groups `(auth)` and `(app)` for the shared shell and for the
composition. `references/app-router-structure.md` owns the folder tokens. A
group is a layout decision, and it is never an authorization decision.

NEVER gate the login route itself. A gate on `/login` and a redirect from
`/dashboard` produce a loop of 307 responses, and the application is unusable.
A cookie that passes the presence test in `proxy.ts` and then fails
`verifySession()` produces the same loop. Clear the invalid cookie on the
login route, and the loop ends.

### Every Server Action authenticates inside itself

```ts
// Wrong: the action takes the identity from its caller.
// Failure: a Server Action is a public HTTP endpoint, and CVE-2026-64643
// discloses its id through the client chunks. An attacker posts any userId to
// that endpoint and deletes the project of another person.
"use server";

export async function deleteProject(userId: string, projectId: string) {
  await api.DELETE("/api/projects/{id}/", { params: { path: { id: projectId } } });
}
```

```ts
// Correct: the action reads the identity from the session, and it ignores
// every argument that claims one.
"use server";

import { verifySession } from "@/lib/dal/session";
import { deleteProjectFor } from "@/lib/dal/projects";

export async function deleteProject(projectId: string) {
  const session = await verifySession(); // 1. authorize inside the action
  if (session === null) return { error: "unauthorized" };

  await deleteProjectFor(session, projectId); // Django checks the ownership
  return {};
}
```

The rule covers every `"use server"` function and every Route Handler. Next.js
states it directly: never assume an authentication claim at the `"use server"`
or the `"use cache"` boundary, and always authenticate inside the boundary. A
comment such as "this action runs from a protected page only" is not a gate.

An argument that names an identity is the signal. NEVER take a `userId`, a
`role`, a `tenantId`, or an `isAdmin` flag as a parameter of a Server Action.
Read each one from the session. `references/data-access-and-mutations.md` owns
the order inside the action, which is authorize, validate, mutate, invalidate,
and redirect.

### Choose the answer to a missing session

| The situation | The call | The caveat |
| --- | --- | --- |
| No session, and the user needs the login route | `redirect('/login?next=...')` | Stable. Validate the `next` value. |
| No session, and the route renders a 401 in place | `unauthorized()`, with an `unauthorized.tsx` file | EXPERIMENTAL in 16.3. It needs `authInterrupts: true`, and Next.js does not recommend it for production. |
| A session exists, and the identity lacks the permission | `forbidden()`, with a `forbidden.tsx` file | EXPERIMENTAL in 16.3. It needs `authInterrupts: true`. The status is 200 where the stream already started. |

`redirect()` is the only stable answer of the three. Take `unauthorized()` and
`forbidden()` only where the project accepts an experimental flag, and record
the decision. Confirm the flag and the two calls against the installed version
before you write the code.

Render an explanation on a 403, whichever call produces it. A blank page after
a permission failure is a support request, and the user cannot act on it.
`references/error-and-empty-state-copy.md` owns the words.

### The redirect target is validated

```ts
// Wrong: the redirect follows the query string.
// Failure: an open redirect. A ?next= value that holds an absolute URL to
// another host starts the session and then sends the user to the attacker,
// and the address bar shows the real application until the last step. The
// protocol-relative form //evil.example passes a naive startsWith("/") test.
redirect(searchParams.next ?? "/");
```

```ts
// Correct: src/lib/auth/safe-next.ts accepts a same-origin relative path only.
export function safeNext(next: string | undefined): string {
  if (next === undefined) return "/";
  if (!next.startsWith("/")) return "/"; // an absolute URL to another host
  if (next.startsWith("//")) return "/"; // a protocol-relative URL

  // The string tests above pass "/\evil.example". The URL parser reads the
  // backslash as a second slash, so the value resolves to another origin.
  const origin = process.env.APP_ORIGIN;
  if (origin === undefined) return "/"; // no origin to compare against
  let parsed: URL;
  try {
    parsed = new URL(next, origin);
  } catch {
    return "/";
  }
  return parsed.origin === new URL(origin).origin ? next : "/";
}
```

Every redirect target that a request supplies passes through this function.
Both halves are needed. The string tests refuse a value such as
`https:evil.example`, which the parser resolves against the origin of the
application. The parse refuses `/\evil.example`, which the string tests pass.
The `next` parameter is a value from outside the program.
`references/boundary-validation-and-api-types.md` states the general rule that
covers it. `references/exposed-endpoints-and-destinations.md` states the rule
that every destination is parsed rather than matched.

### Protected data never enters a shared payload

A route that renders the name of a user, a balance, or a private record must
be dynamic. `references/caching-and-revalidation.md` states that a cache is
keyed by what the code passes to it, and never by who asked. This file states
the auth consequence.

```tsx
// Correct: a static shell, and the identity behind a boundary.
import { Suspense } from "react";

export default function Page() {
  return (
    <>
      <MarketingHeader />
      <Suspense fallback={<AccountSkeleton />}>
        <Account />
      </Suspense>
    </>
  );
}

async function Account() {
  const session = await verifySession(); // dynamic, and outside every cache
  if (session === null) redirect("/login?next=/account");
  return <AccountPanel session={session} />;
}
```

Three rules follow. NEVER call `verifySession()` inside a `"use cache"` scope.
A cache scope reads no request data, and the entry would serve the first user
to every later one. NEVER prefetch a per-user query into a payload
that a shared cache holds;
`references/server-state-and-query-cache.md` owns the prefetch, and
`references/data-access-and-mutations.md` owns the `HydrationBoundary` around
it. Read the RSC payload of each protected route with the cookies cleared, and
confirm that it holds no private value.

### The UI reflects a permission, it does not enforce one

```tsx
// Wrong: the hidden button is the whole control.
// Failure: the endpoint is open to every session. A user who reads the client
// chunks finds the Server Action id, calls it, and the delete succeeds.
{user.role === "admin" && <DeleteButton id={id} />}
```

```tsx
// Correct: the same test, stated as UX, with the server as the gate.
// src/features/auth/can.tsx
"use client"; // reason: the component reads the session from a context

export function Can({
  permission,
  children,
}: {
  permission: string;
  children: React.ReactNode;
}) {
  // UX only. Django decides. Every gated action fails server-side as well.
  const session = useSession();
  return session.permissions.includes(permission) ? <>{children}</> : null;
}
```

Four rules bind the permission model. The server shapes the roles, the groups,
and the permission list once, and it delivers them with the user payload. The
client holds that list in memory for the render, and never in `localStorage`;
`references/session-and-token-lifecycle.md` states that rule. Each check is a
pure function over the session, so a test calls it with no component. Every
gated action still answers 403 from the server, and the view renders that 403
rather than a blank panel.

Hide a control that the identity may never use. Disable a control that the
identity may use later, and state the condition beside it. The user then reads
a reason rather than an absence.

### One tenant, one cache key

A multi-tenant application holds the active organization in a URL segment, or
in its own cookie. Every query key carries it, or one tenant reads the cache
entry of another after a switch.

```ts
// Correct: the tenant is the first level of the key factory.
export const projectKeys = {
  all: (tenant: TenantId) => ["tenant", tenant, "projects"] as const,
  lists: (tenant: TenantId) => [...projectKeys.all(tenant), "list"] as const,
};
```

`references/server-state-and-query-cache.md` owns the key factory and states
that a key encodes every input that changes the result. The tenant is such an
input. A switch of the tenant also clears the entries of the previous one, and
`queryClient.clear()` is the simplest correct answer.

## Verification

```bash
# 1. Find a protected page with no session check. Read every hit.
rg --files-without-match 'verifySession|getSession' -g 'page.tsx' src/app

# 2. Confirm that proxy.ts holds no authorization. This must print nothing.
rg -nE 'jwtVerify|jsonwebtoken|decode\(|\.role|verifySession|DJANGO_URL' proxy.ts src/proxy.ts

# 3. Find a Server Action that takes an identity as a parameter. Read every
#    hit.
rg -n -A3 '"use server"' src/ | rg -i 'userId|role|tenantId|isAdmin|isStaff'

# 4. Confirm that every Server Action verifies the session.
rg -l '"use server"' -g '*.ts' src/ | xargs rg --files-without-match 'verifySession'

# 5. Find a session read inside a cache scope. This must print nothing.
rg -n -A12 '"use cache"' -g '*.ts' -g '*.tsx' src/ | rg 'verifySession|cookies\('

# 6. Find a redirect that follows a request value. Each hit passes through
#    safeNext.
rg -n 'redirect\(' src/ | rg -i 'next|returnTo|callback'

# 7. Confirm that a protected route leaks nothing with no cookie.
curl -s "$APP_ORIGIN/dashboard" | rg -i 'user@|role|balance' && echo FAIL || echo PASS

# 8. Confirm that a Server Action refuses an unauthenticated caller. Read the
#    action id from the client chunk, then post to it with no cookie. Expect a
#    401 or a 403, and never a 200 with a mutation.

# 9. Confirm that no protected route is static. Build, and read the render
#    mode of each one.
pnpm build
```

## Review checklist

- [ ] Does every protected page call `verifySession()`, rather than trust
      `proxy.ts`?
- [ ] Does `proxy.ts` test the presence of a cookie only, with no decode, no
      data call, and no role check?
- [ ] Is the session read memoised with React `cache()`, so one request
      produces one read?
- [ ] Does the session module import `server-only`?
- [ ] Does the check sit in the page rather than in the layout?
- [ ] Is the login route free of every gate, so no redirect loop is possible?
- [ ] Does every Server Action and every Route Handler verify the session
      inside its own body?
- [ ] Is every Server Action free of a parameter that names an identity?
- [ ] Does the project record its decision on `authInterrupts`, where it uses
      `unauthorized()` or `forbidden()`?
- [ ] Does every 403 render an explanation rather than a blank page?
- [ ] Does every redirect target pass through the same-origin check?
- [ ] Is `verifySession()` absent from every `"use cache"` scope?
- [ ] Is every route that renders per-user data reported as dynamic by the
      build?
- [ ] Does the RSC payload of each protected route hold no private value with
      the cookies cleared?
- [ ] Does the server deliver the permission list, and does the client hold it
      in memory only?
- [ ] Does every gated control carry a comment that states that the gate is
      UX?
- [ ] Does every cache key carry the tenant, in a multi-tenant application?

## Handoffs

- The credential, the cookie attributes, the single-flight refresh, the
  logout, and the status that ends a session →
  `references/session-and-token-lifecycle.md`.
- `proxy.ts`, its permitted work, CVE-2025-29927, the route files
  `unauthorized.tsx` and `forbidden.tsx`, and the route groups →
  `references/app-router-structure.md`.
- The rest of the rules for a data access module, and the order inside a
  Server Action → `references/data-access-and-mutations.md`.
- The `"use cache"` scope, the request data that it may not read, and the
  Router Cache → `references/caching-and-revalidation.md`.
- The prefetch, the `HydrationBoundary`, the key factory, and the cache that a
  tenant switch clears → `references/server-state-and-query-cache.md`.
- The parse over the `next` parameter and over the session payload →
  `references/boundary-validation-and-api-types.md`.
- The `"use client"` directive on the permission component, and the
  server-to-client prop → `references/server-and-client-components.md`.
- The error boundary that renders a failed session read →
  `references/suspense-and-actions.md`.
- The CSRF token that a session write carries →
  `references/cross-origin-and-bff-proxy.md`.
- The handshake that must authenticate from the same session, and the close
  codes 4001 and 4003 → `references/push-transport-and-connection.md`. This
  file states that the connection never trusts an identity that the client
  sends.
- The Server Action as a public endpoint, the origin check over it, and the rule
  that every destination is parsed →
  `references/exposed-endpoints-and-destinations.md`. The Content Security
  Policy and the header set are `references/security-headers-and-csp.md`. The
  judgment of an injection sink is
  `references/untrusted-markup-and-injection.md`. The supply chain of a
  dependency is `references/secret-boundary-and-supply-chain.md`. That domain
  holds a veto.
- The focus that moves to a 403 message, and the keyboard path to a gated
  control → `references/keyboard-focus-and-live-regions.md`. That domain holds
  a veto.
- The words of a 403 explanation → `references/error-and-empty-state-copy.md`.
  The words on a disabled control → `references/interface-copy-and-voice.md`.
- The test that posts to a Server Action with no session →
  `references/end-to-end-journeys-and-flake-control.md`.
- The DRF permission class, the object-level check, the impersonation procedure,
  and every server-side gate → `secure-code-auditor`. This file owns the three
  frontend layers above it.
