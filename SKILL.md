---
name: frontend-production-engineer
description: >-
  Production-grade frontend engineering for Next.js and TypeScript against a
  Django and Django REST Framework backend. Use whenever frontend work is
  planned, written, or reviewed in a Next.js repository and the task touches
  App Router files (app/, layout.tsx, page.tsx, route.ts, proxy.ts,
  next.config.ts, tsconfig.json, package.json), the "use client" or
  "use server" boundary, await params, searchParams, or cookies(),
  "use cache" and revalidateTag, a Server Action, a Route Handler, a
  hydration error, React components, or a DRF endpoint being consumed, and
  the agent has to establish the installed versions, plan before editing,
  keep the diff minimal, and hold the result to a stated definition of done
  rather than guessing at an API, even if "frontend" is never used.
  Review-time audits existing code and reports what fails; write-time applies
  the defaults while generating code. Next.js 16 and React 19 are the pinned
  baseline; the router below lists the integrated material and grows one
  domain per release.
license: MIT
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
metadata:
  author: n-shadloo
  version: 1.0.0
---

# frontend-production-engineer

A frontend engineering skill. It plans, writes, and reviews production Next.js
and TypeScript against a Django / Django REST Framework backend, holding the
work to a stated definition of done instead of stopping at code that renders.
Scope is the browser-facing application: routing and rendering, the React
component tree, the typed contract with the backend, and the non-functional
guarantees a shipped frontend owes its users. It does not cover the Django
backend itself — server-side code, the ORM, the database, backend security,
and backend performance belong to the author's `secure-code-auditor` and
`django-performance-optimizer` skills, and this skill defers to them rather
than restating their material. Its canonical content is reused by other agents
(Codex, Cursor, Gemini CLI) via `AGENTS.md`, with Claude as the primary
integration.

## How the reference material is organized

Domain depth lives in `references/`, never in this file. Every reference file
has two layers:

1. **Principle** — the concern, why it matters, and the correct approach,
   stated framework-agnostically so it survives the next major version.
2. **Pinned-stack depth** — the concrete Next.js, React, TypeScript, and
   Django/DRF-contract implementation: runnable code, `// Wrong:` and
   `// Correct:` pairs naming the failure each wrong version produces, and a
   binary review checklist. This is where the depth lives.

Load only the file(s) relevant to the concern in front of you.

| Concern | Reference file |
|---|---|
| The route tree and the request — `app/`, `layout.tsx`, `page.tsx`, `template.tsx`, `loading.tsx`, `error.tsx`, `global-error.tsx`, `not-found.tsx`, `forbidden.tsx`, `unauthorized.tsx`, `default.tsx`, `route.ts`, `instrumentation.ts`; the folder tokens `(group)`, `_folder`, `[param]`, `[...slug]`, `[[...slug]]`, `@slot`, `(.)`, `(..)`, `(...)`; a parallel or intercepting route, a modal that breaks on reload; `await params`, `await searchParams`, `cookies()`, `headers()`, `draftMode()`, `PageProps`, `LayoutProps`, `next typegen`, `redirect()`, `notFound()`, `generateStaticParams`; `middleware.ts` renamed to `proxy.ts`, the `proxy` export, `skipProxyUrlNormalize`, a redirect or a locale rule at the network boundary, an auth check that belongs in a layout, CVE-2025-29927; `next.config.ts` keys — `typedRoutes`, `turbopack`, `images.remotePatterns`, `serverRuntimeConfig`, `experimental.ppr`, `next/legacy/image`, `next lint`; `NEXT_PUBLIC_` variables; Node 20.9, the Next 15 to 16 upgrade, a codemod, `@next-codemod-error`; the errors "Cannot access Request information synchronously", "Dynamic APIs are Asynchronous", "middleware ... renamed to proxy". Not here: `useQuery` and `staleTime` (domain 06), CSP and header values (domain 17), metadata content (domain 18), locale message files (domain 19). | `references/app-router-structure.md` |
| The boundary inside a route — `"use client"`, `"use server"`, a Server Component, a Client Component, a directive on a layout or a page, a provider that needs the client, a client leaf, `server-only`, `client-only`; a prop that fails to serialize, a class instance or a callback across the boundary, "Only plain objects can be passed"; `<Suspense>`, a streamed promise, `use()`, a skeleton, `reset()`, an error boundary, `useSearchParams()`; a fetch in `useEffect` that belongs on the server; a hydration error, "Text content does not match server-rendered HTML", `suppressHydrationWarning`, `typeof window`, `localStorage` in a render. Not here: hook composition and the React 19 API surface (domain 03), the client cache (domain 06), bundle bytes and INP (domain 16), the accessible name of a control (domain 10). | `references/server-and-client-components.md` |
| The call to the backend — a data access layer, a DAL module, where a fetch belongs, a Server Component fetch, a Route Handler, a BFF or a proxy in front of Django, a browser call to Django, CORS on a session cookie; a Server Action, `<form action>`, an action that must authorize, validate, mutate, invalidate, and redirect in that order, a `redirect()` swallowed by a catch; a generated client, `openapi-typescript`, a drf-spectacular schema, a renamed serializer field, the DRF pagination envelope `{count, next, previous, results}`, a DRF error envelope; `prefetchQuery`, `dehydrate`, `HydrationBoundary`, a client that refetches after a prefetch; two sources for one resource. Not here: the schema generation config and the contract itself (domain 05), the query keys and the mutation state (domain 06), the field-level error mapping in a form (domain 11), the Django query behind a slow endpoint (sibling skill `django-performance-optimizer`). | `references/data-access-and-mutations.md` |
| The lifetime of a response — `cacheComponents`, Cache Components, the previous caching model, `"use cache"`, `cacheLife`, `cacheTag`, `unstable_cache`, `fetch` `cache` and `next.revalidate` and `next.tags`, route segment `revalidate` and `dynamic`; `revalidateTag`, `updateTag`, `refresh()`, `revalidatePath`, `router.refresh()`, the Router Cache, read-your-writes, stale data after a mutation; Partial Prerendering, a static shell with a dynamic hole, a route that is unexpectedly dynamic or unexpectedly static, the build report symbols, `connection()`, `unstable_noStore`, `Math.random()` or `new Date()` on a route; one user seeing another user's data. Not here: `staleTime` and the client query cache (domain 06), the `Cache-Control` header and the CDN (domain 22), whether the data may be cached at all (domain 17), the Django-side cache (sibling skill `django-performance-optimizer`). | `references/caching-and-revalidation.md` |

The operating doctrine is integrated but has no row, because it is always in
effect and lives in this file rather than in `references/`. Each release
integrates one domain and adds its rows. Write each row as the trigger
vocabulary that routes to the file. Give the concrete nouns and verbs that a
task can contain, and the near-misses that belong to a neighbouring domain.
Never write a row as a summary of the file. A vague row is a domain that never
loads.

Cross-references between reference files are deliberate, and the seams are
recorded here as the domains land: which file owns a concern, and which files
defer to it rather than repeating it. The four files of
`nextjs-app-router-architecture` split on one line each.
`app-router-structure` owns the route tree, the request, and `proxy.ts`.
`server-and-client-components` owns the boundary inside a route.
`data-access-and-mutations` owns the call to the backend.
`caching-and-revalidation` owns the lifetime of the response. A rule about a
route file lives in the first file only. A rule about a `"use client"` leaf
lives in the second file only.

## Mode selection

Both modes are bound by the same standing rules. Plan before editing, and
state the success criteria the work will be checked against. Verify the
installed versions from `package.json`, `next.config.ts`, and `tsconfig.json`
before generating code — never mix Next 15 and Next 16 idioms in one file, and
never write against a version the repository does not have. Never invent an
API: if you cannot confirm that a function, prop, or option exists in the
installed version, say so instead of guessing. Keep the diff minimal, so every
changed line traces to the request. Run the commands rather than asserting
their result.

**Review-time.** Trigger when the user asks to review, audit, or "check"
existing frontend code; pastes a component and asks whether it is correct; or
has just finished a feature and wants it looked at. Behavior:

- Treat the codebase as **read-only**. Do not edit, refactor, or "fix in
  place" unless the user explicitly asks you to apply fixes afterward.
- Establish the installed versions first. A finding written against the wrong
  major version is noise.
- Investigate before flagging. Confirm the render path, the boundary the
  component actually sits on, and whether the code is reachable at runtime.
  Do not pattern-match a keyword into a finding.
- Report findings ordered by severity, each with a location, the concrete
  failure a user would experience, and a fix. End with what you did *not*
  review.

**Write-time.** Trigger when you are generating or modifying frontend code for
a feature. Behavior:

- Apply the defaults from the relevant reference file(s) as you write.
- Take types from the backend schema, never from imagination. A response shape
  you guessed is a bug with a delay on it.
- Note the choices a reviewer would want to see. If a requirement forces a
  pattern this skill would otherwise reject, say so and describe the residual
  risk rather than hiding it.
- Run the definition-of-done checks before reporting the work complete.

**If it is ambiguous,** apply write-time guardrails while coding and offer to
run a review afterward.

## Definition of done

Resolve domains in this order on any feature task: the standing rules above,
always; then the foundations, for where files go and how they are typed; then
the backend contract, before any code that touches Django; then whichever
domain files match the feature; then the non-functional domains, as a review
pass before done. At 1.0.0 the standing rules and
`nextjs-app-router-architecture` are integrated, so the rest of the order
fills in as the router grows.

Work is not done because it renders. It is done when all of the following
hold, each established by running the command rather than by inspection:

- The installed versions were verified before any code was written, and no
  file mixes idioms from two major versions.
- TypeScript compiles clean, with no suppression added to make it do so.
- Lint passes.
- The tests covering the change pass.
- The production build succeeds.
- The diff contains nothing the request did not ask for.

**Blocking domains.** Once integrated, `nextjs-app-router-architecture`,
`typescript-type-system-discipline`, `django-drf-api-contract`,
`authentication-and-authorization`, `accessibility-wcag`, `frontend-security`,
and `testing-and-quality` each hold a veto over completion. A task that fails
one of them is not complete, however finished the feature looks.
`frontend-security` and `accessibility-wcag` are absolute: a keyboard trap, an
interactive control with no accessible name, an unescaped injection sink, or a
secret that reaches the client is a failed task, not a follow-up ticket.

**`nextjs-app-router-architecture` — integrated, blocking.** The task fails
when any one of these holds:

- A route cannot state its render mode, its data source, its cache strategy,
  and its invalidation trigger.
- A `"use client"` directive sits on `app/layout.tsx` or on a page shell, or
  carries no one-line reason above it.
- Code reads `params`, `searchParams`, `cookies()`, `headers()`, or
  `draftMode()` without `await`, or an `@next-codemod-error` or
  `UnsafeUnwrapped` marker remains.
- `middleware.ts` is still present, and the Edge runtime is not a stated
  requirement.
- `proxy.ts` verifies a session, calls a database, or is the only place that
  authorizes a route.
- A Server Action does not verify the session inside the action.
- A response that depends on the user is reachable from a static route, or
  from a `"use cache"` scope.
- A `"use cache"` scope declares no `cacheTag`, or no `cacheLife`.
- A dynamic segment has no `loading.tsx` or `<Suspense>` with a real
  skeleton, or a fallible segment has no `error.tsx` with a working `reset()`.
- A parallel route slot has no `default.tsx`.
- The build report contradicts the declared render mode of a route.

**Conflict rule.** security > accessibility > correctness > performance >
developer convenience. No level trades down to satisfy a level above it. When
a requirement appears to force such a trade, say so and stop rather than take
it silently.
