---
name: frontend-production-engineer
description: >-
  Frontend engineering for Next.js and TypeScript against a Django and DRF
  backend. Use whenever frontend work is planned, written, or reviewed and
  the task touches App Router files (app/, layout.tsx, page.tsx, proxy.ts,
  next.config.ts, tsconfig.json), the "use client" or "use server"
  boundary, await params or cookies(), "use cache" and revalidateTag, a
  Server Action, a Route Handler, a hydration error, a React hook
  (useState, useEffect, useActionState), a <Suspense> or error boundary,
  the React Compiler, a type or a cast (any, unknown, a branded id, a
  discriminated union), a Zod schema (safeParse, z.infer), the DRF
  contract (an OpenAPI schema, drf-spectacular, a generated client, an
  error or pagination envelope, CORS, CSRF), or the project setup
  (package.json, eslint.config.ts, Prettier, pnpm and the lockfile, a path
  alias, a barrel file, a git hook, a monorepo), and the agent has to
  verify the installed versions, plan first, and hold the work to a
  definition of done, even if "frontend" is never used.
license: MIT
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
metadata:
  author: n-shadloo
  version: 1.4.0
---

# frontend-production-engineer

A frontend engineering skill. It plans, writes, and reviews production Next.js
and TypeScript against a Django / Django REST Framework backend. It holds the
work to a stated definition of done, and it does not stop at code that renders.
Scope is the browser-facing application. That scope is routing and rendering,
the React component tree, and the typed contract with the backend. It also
holds the non-functional guarantees a shipped frontend owes its users.

It does not cover the Django backend itself. Server-side code, the ORM, the
database, backend security, and backend performance belong to the author's
`secure-code-auditor` and `django-performance-optimizer` skills. This skill
defers to them. It does not restate their material. Other agents (Codex,
Cursor, Gemini CLI) reuse its canonical content through `AGENTS.md`, with
Claude as the primary integration.

## How the reference material is organized

Domain depth lives in `references/`, never in this file. Every reference file
has two layers:

1. **Principle** — the concern, why it matters, and the correct approach,
   stated framework-agnostically so it survives the next major version.
2. **Pinned-stack depth** — the concrete Next.js, React, TypeScript, and
   Django/DRF-contract implementation. It holds runnable code, `// Wrong:` and
   `// Correct:` pairs that name the failure each wrong version produces, and a
   binary review checklist. This is where the depth lives.

Load only the file(s) relevant to the concern in front of you.

| Concern | Reference file |
|---|---|
| The route tree and the request — `app/`, `layout.tsx`, `page.tsx`, `template.tsx`, `loading.tsx`, `error.tsx`, `global-error.tsx`, `not-found.tsx`, `forbidden.tsx`, `unauthorized.tsx`, `default.tsx`, `route.ts`, `instrumentation.ts`; the folder tokens `(group)`, `_folder`, `[param]`, `[...slug]`, `[[...slug]]`, `@slot`, `(.)`, `(..)`, `(...)`; a parallel or intercepting route, a modal that breaks on reload; `await params`, `await searchParams`, `cookies()`, `headers()`, `draftMode()`, `PageProps`, `LayoutProps`, `next typegen`, `redirect()`, `notFound()`, `generateStaticParams`; `middleware.ts` renamed to `proxy.ts`, the `proxy` export, `skipProxyUrlNormalize`, a redirect or a locale rule at the network boundary, an auth check that belongs in a layout, CVE-2025-29927; `next.config.ts` keys — `typedRoutes`, `turbopack`, `images.remotePatterns`, `serverRuntimeConfig`, `experimental.ppr`, `next/legacy/image`, `next lint`; `NEXT_PUBLIC_` variables; Node 20.9, the Next 15 to 16 upgrade, a codemod, `@next-codemod-error`; the errors "Cannot access Request information synchronously", "Dynamic APIs are Asynchronous", "middleware ... renamed to proxy". Not here: `useQuery` and `staleTime` (domain 06), CSP and header values (domain 17), metadata content (domain 18), locale message files (domain 19). | `references/app-router-structure.md` |
| The boundary inside a route — `"use client"`, `"use server"`, a Server Component, a Client Component, a directive on a layout or a page, a provider that needs the client, a client leaf, `server-only`, `client-only`; a prop that fails to serialize, a class instance or a callback across the boundary, "Only plain objects can be passed"; a streamed promise, `use()`, the `"use client"` directive on an `error.tsx`, `useSearchParams()`; a fetch in `useEffect` that belongs on the server; a hydration error, "Text content does not match server-rendered HTML", `suppressHydrationWarning`, `typeof window`, `localStorage` in a render. Not here: which component holds the state (`references/state-and-effects.md`), the granularity of a boundary inside a route, the shape of a fallback, and the React 19 Actions (`references/suspense-and-actions.md`), the client cache (domain 06), bundle bytes and INP (domain 16), the accessible name of a control (domain 10). | `references/server-and-client-components.md` |
| The call to the backend — a data access layer, a DAL module, where a fetch belongs, a Server Component fetch, a Route Handler, a BFF or a proxy in front of Django, a browser call to Django, CORS on a session cookie; a Server Action, `<form action>`, an action that must authorize, validate, mutate, invalidate, and redirect in that order, a `redirect()` swallowed by a catch; a generated client, `openapi-typescript`, a drf-spectacular schema, a renamed serializer field, the DRF pagination envelope `{count, next, previous, results}`, a DRF error envelope; `prefetchQuery`, `dehydrate`, `HydrationBoundary`, a client that refetches after a prefetch; two sources for one resource. Not here: the schema generation config and the contract itself (domain 05), the query keys and the mutation state (domain 06), the field-level error mapping in a form (domain 11), the Django query behind a slow endpoint (sibling skill `django-performance-optimizer`). | `references/data-access-and-mutations.md` |
| The lifetime of a response — `cacheComponents`, Cache Components, the previous caching model, `"use cache"`, `cacheLife`, `cacheTag`, `unstable_cache`, `fetch` `cache` and `next.revalidate` and `next.tags`, route segment `revalidate` and `dynamic`; `revalidateTag`, `updateTag`, `refresh()`, `revalidatePath`, `router.refresh()`, the Router Cache, read-your-writes, stale data after a mutation; Partial Prerendering, a static shell with a dynamic hole, a route that is unexpectedly dynamic or unexpectedly static, the build report symbols, `connection()`, `unstable_noStore`, `Math.random()` or `new Date()` on a route; one user seeing another user's data. Not here: `staleTime` and the client query cache (domain 06), the `Cache-Control` header and the CDN (domain 22), whether the data may be cached at all (domain 17), the Django-side cache (sibling skill `django-performance-optimizer`). | `references/caching-and-revalidation.md` |
| The compiler and the gates that prove it — `tsconfig.json`, `compilerOptions`, `strict`, `strictNullChecks`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `verbatimModuleSyntax`, `isolatedModules`, `moduleResolution`, `moduleDetection`, `noImplicitOverride`, `noPropertyAccessFromIndexSignature`, `skipLibCheck`, `baseUrl`, `paths`, the `next` plugin; `next-env.d.ts`, a `*.d.ts` file, a generated type that someone edited by hand; `tsc --noEmit`, a typecheck that CI runs and the build does not, `typescript.ignoreBuildErrors`, `useTypeScriptCli`, a green build with a red editor; `eslint.config.ts`, typescript-eslint, `strictTypeChecked`, `stylisticTypeChecked`, `projectService`, `disableTypeChecked`, `no-explicit-any`, `no-unsafe-assignment`, `no-floating-promises`, `no-misused-promises`, `switch-exhaustiveness-check`, `ban-ts-comment`; `@ts-ignore`, `@ts-expect-error`, a suppression with no reason; `vitest --typecheck`, `expectTypeOf`, a `*.test-d.ts` file, `type-coverage`; a slow editor, a slow compiler, `--extendedDiagnostics`, `--generateTrace`, the errors `TS2589` and `TS1484`; TypeScript 5.9, 6.0, or 7.0. Not here: the folder layout and the Prettier config (domain 04), the test runner and the contract test (domain 20), the CI pipeline and the Docker build (domain 22). | `references/typescript-config-and-enforcement.md` |
| The vocabulary that models a value — `any`, `unknown`, `never`, a cast, `as`, a non-null `!`, a value that is possibly undefined, the errors `TS2322`, `TS2345`, `TS2532`, `TS18048`, `TS7053`, `TS2366`; a discriminated union, a status flag beside a data field, an optional-everything model of a serializer, `assertNever`, an exhaustive `switch`, a `Result` type, "make illegal states unrepresentable"; a branded id, `.brand()`, a `UserId` passed where an `OrderId` belongs; `satisfies`, `as const`, `as const satisfies`, a config object that loses its literal type; `enum` against a const object and a derived union; `keyof`, `typeof`, `infer`, a mapped type, a conditional type, `Prettify`, `NoInfer`, a type predicate, `asserts`; `type` against `interface`, `declare module`, a package with wrong types; `React.FC`, `PropsWithChildren`, `ComponentProps`, `forwardRef`, `Ref`, `RefObject`, `MutableRefObject`, `useRef` with no argument, a context read through a `!`, `React.JSX`. Not here: the composition and the list key (`references/component-composition.md`), the render rules and the effect rules (`references/state-and-effects.md`), the theme object behind a `satisfies` (domain 09), the form state type and the field error map (domain 11). | `references/type-modeling-and-narrowing.md` |
| A value that enters the program — `res.json()` cast to a type, a `fetch` response, `searchParams` read as data, `localStorage`, `sessionStorage`, `process.env`, a webhook body, `FormData`; "parse at the boundary", "type the response", a run-time crash on data that TypeScript called typed; Zod 4 — `z.object`, `safeParse`, `z.infer`, `z.input`, `z.output`, `z.email`, `z.discriminatedUnion`, `.brand()`, `.superRefine()`, `.check()`, `z.prettifyError`, `z.treeifyError`, `z.flattenError`, `error.issues`; the Zod 3 calls that Zod 4 removed — `error.errors`, `.format()`, `.flatten()`, `z.string().email()`, `z.nativeEnum()`, `.merge()`, `.ip()`, `.cidr()`; the generated `paths` and `components` types, `openapi-typescript`, a type hand-copied from the OpenAPI schema, two sources for one shape; a DRF field that is `required=False`, `allow_null=True`, or `blank=True`, a `nullable: true` that the generated type does not carry, `null` at run time where the type promised a value, the pagination envelope typed as an array, the DRF error body `{field: [msg]}`, `drf-standardized-errors` and its `type` and `attr` fields; `@t3-oss/env-nextjs`, an environment variable that is `undefined` in the browser. Not here: the drf-spectacular config and the codegen command (domain 05), the query keys and the mutation state (domain 06), the resolver and the field array in a form (domain 11), the MSW handler and the contract test (domain 20), the server-side serializer (sibling skill `django-api-contract`). | `references/boundary-validation-and-api-types.md` |
| The shape of a component and the way that parts compose — a God component, a component file past 200 lines, more than five `useState` calls, a decomposition, "single responsibility"; three or more boolean props that change the layout, composition over configuration, `children`, a slot, a named part, a compound component, `Tabs.List`, `Tabs.Panel`, a render prop, a polymorphic component, `asChild`, `Slot`, the Base UI `render` prop, `useRender`, `data-slot`; controlled against uncontrolled, `value` and `onChange` against `defaultValue`, `useControllableState`, "A component is changing an uncontrolled input to be controlled"; `ref` as a prop against `forwardRef` in new code, `useImperativeHandle`, a parent that reaches into a child; a list `key`, an index key, "Each child in a list should have a unique key prop", a row that keeps the state of another row after a sort or a filter; a headless hook, a custom hook that returns markup; shadcn/ui on Base UI, Radix Primitives, `usehooks-ts`, `prop-types`. Not here: where the state itself lives and whether an effect is correct (`references/state-and-effects.md`), the `<Suspense>` boundary and the Action hooks (`references/suspense-and-actions.md`), `React.FC` and `PropsWithChildren` and `ComponentProps` (`references/type-modeling-and-narrowing.md`), the classes and the tokens on a part (domain 09), the ARIA role and the focus order (domain 10), the virtualiser for a long list (domain 12). | `references/component-composition.md` |
| Where a value lives and when an effect is correct — `useState`, `useReducer`, `useEffect`, `useLayoutEffect`, `useContext`, `createContext`, `useSyncExternalStore`, `useEffectEvent`, `useMemo`, `useCallback`, `memo`, `<Profiler>`; colocate state, lift state up, prop drilling, a global store, a derived value, a filtered list held in state, an effect that sets state, an effect that resets state when a prop changes, a `key` that remounts a component, five `useState` calls for five form fields, a context that holds a list from the backend; a subscription, `window.matchMedia`, a browser API read during a render, a socket that reconnects on an unrelated change, a stale closure, a dependency array that lies; Strict Mode, `reactStrictMode`, an effect that runs twice in development, a missing cleanup; the Rules of React, purity, a mutation during a render, `eslint-plugin-react-hooks`, `rules-of-hooks`, `exhaustive-deps`, `set-state-in-render`, `set-state-in-effect`, `reactCompiler`, `babel-plugin-react-compiler`, `"use memo"`, `"use no memo"`, `compilationMode`, a compiler bailout, `eslint-plugin-react-compiler`; CVE-2025-55182 and the React 19.2.1 floor; the errors "Rendered more hooks than during the previous render", "Cannot update a component while rendering a different component", "Too many re-renders", "Maximum update depth exceeded", "Missing getServerSnapshot, which is required for server-rendered content", "Hooks can only be called inside the body of a function component". Not here: `useQuery` and `staleTime` and the query cache (domain 06), the shape of the component around the state (`references/component-composition.md`), the boundary and the Action (`references/suspense-and-actions.md`), the `"use client"` directive itself (`references/server-and-client-components.md`), the type-aware lint config (`references/typescript-config-and-enforcement.md`), the INP that a re-render costs (domain 16). | `references/state-and-effects.md` |
| The boundary that renders while a value is absent, and the Action that changes it — `<Suspense>`, a fallback, a skeleton that does not match the content, a page that shows one spinner, a section boundary, a widget boundary, layout shift when the data arrives; `ErrorBoundary`, `react-error-boundary`, `fallbackRender`, `resetErrorBoundary`, `useErrorBoundary`, a panel that must fail alone; `use(` on a promise, a promise created in a render, a fallback that never resolves, `use()` inside a `try`; a React 19 Action, `useActionState`, `useFormStatus`, `useOptimistic`, `startTransition`, `useTransition`, `formAction`, a submit button that must disable while pending, an optimistic value that must roll back; a DRF 400 rendered beside a field, a validation error that was thrown, a 5xx that must reach a boundary; `<Activity>`, `visible` and `hidden`, hidden UI that must keep its state, a tab panel that loses a scroll position; `<title>` or `<meta>` rendered from a component, `preload`, `preinit`, `prefetchDNS`, `preconnect`. Not here: `loading.tsx` and `error.tsx` as route files (`references/app-router-structure.md`), the body of the Server Action and its authorize, validate, mutate, invalidate, redirect order (`references/data-access-and-mutations.md`), the DRF error envelope shapes (`references/boundary-validation-and-api-types.md`), the mutation state in the query cache (domain 06), the resolver and the field array (domain 11), the choreography of a transition (domain 14), the words in the message (domain 15). | `references/suspense-and-actions.md` |
| Where a file goes and which module may import it — `src/`, `src/app`, `src/features`, `src/components/ui`, `src/components/common`, `src/lib`, `src/server`, `src/config`, feature-first, layer-first, colocation, a shared layer, dependency direction, a public API, a private folder that holds too much; a barrel file, `index.ts`, `export *`, a global barrel, a deep relative import `../../../..`, the `@/*` path alias, `optimizePackageImports`, `transpilePackages`; `eslint-plugin-boundaries`, `boundaries/element-types`, `no-restricted-paths`, `dependency-cruiser`, `.dependency-cruiser.js`, `sheriff`, `sheriff.config.ts`, a cycle, an orphan module; `src/api/generated`, `api:generate`, a generated client that someone edited, the schema artifact, the drift check; a monorepo, a workspace, `pnpm-workspace.yaml`, `turbo.json`, Turborepo, Nx; a repository restructure inside a feature task; the errors "Importing elements of type ... is not allowed", "is not allowed in elements of type", "Cannot find module" from a missing alias. Not here: the folder tokens `(group)`, `_folder`, and `@slot`, and the route files (`references/app-router-structure.md`), the `server-only` and `client-only` guards (`references/server-and-client-components.md`), `tsconfig.json` and the `paths` value (`references/typescript-config-and-enforcement.md`), the rest of the lint config array (`references/lint-format-and-scripts.md`), the drf-spectacular config and the generator (domain 05), the tokens that `components/ui` renders (domain 09). | `references/directory-and-module-boundaries.md` |
| The checks over the code and the commands that run them — `package.json` scripts, `dev`, `build`, `lint`, `lint:fix`, `format`, `typecheck`, `test`, `test:watch`, `test:e2e`, `analyze`, `pnpm check`, `pnpm dlx`, `--max-warnings=0`, a warning that CI accepts; `eslint.config.ts`, `eslint.config.mjs`, a flat config, `defineConfig`, `eslint-config-next`, `core-web-vitals`, `eslint-config-next/typescript`, an `ignores` entry, a lint rule that reports nothing; `next lint`, the `eslint` key in `next.config.ts`, `next-lint-to-eslint-cli`, a CI step that lints nothing; Prettier, `.prettierrc`, Biome, `biome.json`, `prettier-plugin-tailwindcss`, `tailwindStylesheet`, `eslint-plugin-tailwindcss`, `eslint-plugin-better-tailwindcss`, `.editorconfig`; `eslint-disable`, `eslint-disable-next-line`, a suppression with no reason; `next typegen` before a typecheck; `AGENTS.md`, `.cursorrules`, `.cursor/rules`, `.vscode/settings.json`, `.vscode/extensions.json`; the errors "Parsing error: ESLint was configured to run on", "next lint is deprecated and will be removed in Next.js 16", "It looks like you're trying to use tailwindcss directly as a PostCSS plugin". Not here: `projectService`, `strictTypeChecked`, and `tsc --noEmit` (`references/typescript-config-and-enforcement.md`), `eslint-plugin-react-hooks` and `reactCompiler` (`references/state-and-effects.md`), the boundaries block (`references/directory-and-module-boundaries.md`), the hook that calls a script (`references/dependencies-and-git-workflow.md`), the Tailwind theme (domain 09), the test layout and the coverage threshold (domain 20), the CI workflow (domain 22). | `references/lint-format-and-scripts.md` |
| What enters the repository and how a change leaves it — pnpm, npm, yarn, bun, `pnpm install --frozen-lockfile`, `pnpm-lock.yaml`, a lockfile conflict, a hand-edited lockfile, `packageManager`, Corepack, `corepack enable`, Volta, `.nvmrc`, `engines.node`, `.npmrc`, `overrides`; `minimumReleaseAge`, `minimumReleaseAgeExclude`, a cooldown, `onlyBuiltDependencies`, `allowBuilds`, `ignore-scripts`, a lifecycle script, `pnpm audit`, a new dependency and its justification; Renovate, `renovate.json`, Dependabot, `.github/dependabot.yml`, `automerge`, `matchUpdateTypes`, `fetch-metadata`; lefthook, `lefthook.yml`, Husky, `.husky/`, `lint-staged`, `stage_fixed`, a `prepare` script, a hook that never fires; commitlint, `commitlint.config.js`, a Conventional Commit, a commit-msg hook; `.gitignore`, a committed `.env.local`, gitleaks, `.gitleaks.toml`, a secret in the history; `CODEOWNERS`, `.gitattributes`, `linguist-generated`. Not here: the body of a script that a hook calls (`references/lint-format-and-scripts.md`), the folder that a new file goes into (`references/directory-and-module-boundaries.md`), the `NEXT_PUBLIC_` prefix (`references/app-router-structure.md`), whether a package is malicious (domain 17), the CI workflow and the deploy (domain 22), the server-side secret storage (sibling skill `secure-code-auditor`). | `references/dependencies-and-git-workflow.md` |
| The schema that the types come from — drf-spectacular, `SPECTACULAR_SETTINGS`, `COMPONENT_SPLIT_REQUEST`, `COMPONENT_SPLIT_PATCH`, `OAS_VERSION`, `@extend_schema`, `@extend_schema_field`, `@extend_schema_serializer`, `OpenApiExample`, `OpenApiParameter`, `ENUM_NAME_OVERRIDES`, `ENUM_ADD_EXPLICIT_BLANK_NULL_CHOICE`, `schema.yml`, `openapi.json`, `/api/schema/`, `drf-yasg` and Swagger 2.0, django-ninja and OpenAPI 3.1; `openapi-typescript`, `openapi-fetch`, Orval, `@hey-api/openapi-ts`, `@kubb/*`, `swagger-typescript-api`, `openapi-generator`, `operationId`, the `paths` type, `components["schemas"]`; `api:generate`, a client generated from a live URL, a stale schema in production, `oasdiff`, `openapi-diff`, a breaking change on an enum, an enum emitted as a TypeScript `enum`; a `SerializerMethodField` typed `any`, `XRequest` against `X`, a response field that became optional, upstream issue #810; camelCase against snake_case, `djangorestframework-camel-case`, `camelize_serializer_fields`, `humps`, `ts-case-convert`. Not here: the types that a DRF construct produces and the parse over them (`references/boundary-validation-and-api-types.md`), the output folder and its `.gitignore` entry (`references/directory-and-module-boundaries.md`), the `package.json` script surface (`references/lint-format-and-scripts.md`), the client that sends a request (`references/api-client-and-request-safety.md`), the serializer, the viewset, and the deprecation of a field (sibling skill `django-api-contract`). | `references/openapi-schema-and-codegen.md` |
| The request itself, and the failure that comes back — `apiClient`, `createClient`, `endpoints.ts`, a base URL, a `fetch("/api/...")` literal in a component, `DJANGO_URL` against `NEXT_PUBLIC_API_BASE_URL`, an ECONNREFUSED from a server fetch inside Docker; a trailing slash, `APPEND_SLASH`, a 301 that drops a POST body, "you called this URL via POST"; `AbortController`, `AbortSignal.timeout`, `AbortSignal.any`, a request that never ends, a `DOMException` against a `TypeError`; a retry, an exponential backoff, `Idempotency-Key`, `Retry-After`, a 429 loop, a throttle, a duplicate row from a retried POST; `normalizeApiError`, `ApiError`, `fieldErrors`, `retryable`, `ErrorDetail`, `non_field_errors`, `detail`, `[object Object]` in a toast; a 204 with no body, "Unexpected end of JSON input", a 500 that returns HTML; the `next` and `previous` URLs, a computed page offset, `CursorPagination` with no `count`; `FormData`, a multipart boundary, an empty `request.FILES`. Not here: which module holds the call and the order inside a Server Action (`references/data-access-and-mutations.md`), the `Paginated<T>` type and the parse (`references/boundary-validation-and-api-types.md`), the schema and the generator (`references/openapi-schema-and-codegen.md`), CORS and the CSRF header (`references/cross-origin-and-bff-proxy.md`), `queryKey` and `staleTime` (domain 06), the token refresh loop (domain 07), the upload progress bar (domain 13). | `references/api-client-and-request-safety.md` |
| The origin boundary that the browser enforces — `django-cors-headers`, `CORS_ALLOWED_ORIGINS`, `CORS_ALLOW_CREDENTIALS`, `CORS_ALLOW_HEADERS`, a preflight, an `OPTIONS` request, "No 'Access-Control-Allow-Origin' header is present", a wildcard beside `credentials: "include"`, two `Access-Control-Allow-Origin` values, a request that never reaches the Django log; `X-CSRFToken`, the `csrftoken` cookie, `ensure_csrf_cookie`, `CSRF_HEADER_NAME`, `CSRF_TRUSTED_ORIGINS` and its scheme, "CSRF Failed: CSRF token missing", `SESSION_COOKIE_SAMESITE`, `httpOnly`, `Secure`, a cookie that the browser drops on a cross-site call; a BFF, a rewrite in front of Django, a Route Handler proxy, a fixed upstream host, SSRF, `169.254.169.254`, a `?target=` parameter, a body size cap on a proxy. Not here: `proxy.ts`, its permitted work, and CVE-2025-29927 (`references/app-router-structure.md`), the choice between a Server Component fetch, a Route Handler, and a browser call (`references/data-access-and-mutations.md`), the timeout and the error shape (`references/api-client-and-request-safety.md`), the session strategy and the redirect after a 401 (domain 07), the CSP and the response headers (domain 17), the DRF permission class and the server settings (sibling skill `secure-code-auditor`). | `references/cross-origin-and-bff-proxy.md` |

The operating doctrine is integrated but has no row, because it is always in
effect and lives in this file rather than in `references/`. Each release
integrates one domain and adds its rows. Write each row as the trigger
vocabulary that routes to the file. Give the concrete nouns and verbs that a
task can contain, and the near-misses that belong to a neighbouring domain.
Never write a row as a summary of the file. A vague row is a domain that never
loads.

Cross-references between reference files are deliberate. This file records the
seams as the domains land: which file owns a concern, and which files defer to
it rather than repeat it.

The four files of `nextjs-app-router-architecture` split on one line each.
`app-router-structure` owns the route tree, the request, and `proxy.ts`.
`server-and-client-components` owns the boundary inside a route.
`data-access-and-mutations` owns the call to the backend.
`caching-and-revalidation` owns the lifetime of the response. A rule about a
route file lives in the first file only, and a rule about a `"use client"` leaf
lives in the second.

The three files of `typescript-type-system-discipline` split the same way.
`typescript-config-and-enforcement` owns the compiler flags and the gates.
`type-modeling-and-narrowing` owns the vocabulary that models a value inside
the program. `boundary-validation-and-api-types` owns every value that enters
from outside the program, and the types that a DRF response produces. A rule
about a `tsconfig.json` flag lives in the first file only, and a rule about a
schema lives in the third.

The three files of `react-component-architecture` split on the same rule.
`component-composition` owns the shape of a component and the way that parts
compose. `state-and-effects` owns where a value lives, when an effect is
correct, and the Rules of React that the compiler depends on.
`suspense-and-actions` owns the boundary that renders while a value is absent,
and the Action that changes a value. A rule about a slot lives in the first
file only, and a rule about a dependency array lives in the second.

The three files of `project-structure-and-tooling` split on the same rule.
`directory-and-module-boundaries` owns the place where a file goes, and the
rule for which module may import it. `lint-format-and-scripts` owns the checks
that run over the code, and the commands that run them.
`dependencies-and-git-workflow` owns what enters the repository, and how a
change leaves it. A rule about a barrel file lives in the first file only, and
a rule about a lockfile lives in the third.

The three files of `django-drf-api-contract` split on the life of one request.
`openapi-schema-and-codegen` owns the schema that produces the types, and the
generator that reads it. `api-client-and-request-safety` owns the one client
that sends the request, and the normalizer that gives every failure one shape.
`cross-origin-and-bff-proxy` owns what the browser does when the two origins
differ. A rule about a generator lives in the first file only, and a rule about
a timeout lives in the second.

Six seams cross the domains. Where domain 01 and domain 02 meet, domain 01
owns the route file and the awaited request data. Domain 02 owns the types that
the route file uses. Where domain 01 and domain 03 meet, domain 03 decides
which component holds the state. Domain 01 states the boundary.

Where domain 04 and domain 01 meet, domain 01 owns the folder tokens inside
`app/`. Domain 04 owns every folder outside `app/`. Where domain 04 and domain
02 meet, domain 02 owns `tsconfig.json` and the type-aware lint rules. Domain
04 owns the config array that holds them, and the scripts that run them.

Where domain 05 and domain 01 meet, domain 01 owns the module that holds a
call, and it owns `proxy.ts`. Domain 05 owns the client that the module calls,
and the proxy Route Handler in front of Django. Where domain 05 and domain 02
meet, domain 02 owns the type of a value and the parse that proves it. Domain
05 owns the schema that produces the type, and the command that generates it.

## Mode selection

The same standing rules bind both modes. Plan before you edit, and state the
success criteria that the work must meet. Verify the installed versions from
`package.json`, `next.config.ts`, and `tsconfig.json` before you generate code.
Never mix Next 15 and Next 16 idioms in one file. Never write against a version
that the repository does not have.

Never invent an API. Where you cannot confirm that a function, prop, or option
exists in the installed version, say so rather than guess. Keep the diff
minimal, so every changed line traces to the request. Run the commands, and
never assert their result.

**Review-time.** Trigger on three conditions. The user asks to review, audit,
or "check" existing frontend code. The user pastes a component and asks whether
it is correct. The user finished a feature and asks for a review of it.
Behavior:

- Treat the codebase as **read-only**. Do not edit, refactor, or "fix in
  place" unless the user explicitly asks you to apply fixes afterward.
- Establish the installed versions first. A finding written against the wrong
  major version is noise.
- Investigate before you flag. Confirm the render path, the boundary the
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
  risk. Never hide it.
- Run the definition-of-done checks before you report the work complete.

**If it is ambiguous,** apply the write-time guardrails as you write, and offer
a review afterward.

## Definition of done

Resolve the domains in this order on any feature task. The standing rules above
come first, and they always apply. The foundations come next, for where files
go and how they are typed. The backend contract comes before any code that
touches Django. The domain files that match the feature come after it. The
non-functional domains come last, as a review pass before done.

At 1.4.0 the integrated material is the standing rules,
`nextjs-app-router-architecture`, `typescript-type-system-discipline`,
`react-component-architecture`, `project-structure-and-tooling`, and
`django-drf-api-contract`. The rest of the order applies as the router grows.

Work is not done because it renders. It is done when all of the following
hold. Run the command for each one, and never establish it by inspection:

- The versions were verified before any code, and no file mixes idioms from
  two major versions.
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

`frontend-security` and `accessibility-wcag` are absolute. A keyboard trap or
an interactive control with no accessible name is a failed task. An unescaped
injection sink or a secret that reaches the client is a failed task. Neither is
a follow-up ticket.

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

**`typescript-type-system-discipline` — integrated, blocking.** The task fails
when any one of these holds:

- `tsc --noEmit` does not exit 0, or a suppression was added to make it do so.
- `tsconfig.json` is missing `strict`, `noUncheckedIndexedAccess`,
  `isolatedModules`, or `verbatimModuleSyntax`.
- The code holds an `any`, an `@ts-ignore`, or an `@ts-expect-error` with no
  description.
- A value from the network, the URL, storage, the environment, a form, or a
  webhook reaches the program without a parse.
- A cast is present that is neither a const assertion nor a narrow with a
  comment that proves it.
- A non-null `!` stands on a value that can be absent.
- A union `switch` has no `default` that calls `assertNever`.
- An id, a token, or a money amount carries a bare `string` or `number` type.
- A response shape is hand-copied from the OpenAPI schema rather than
  generated, or a paginated endpoint is typed as an array.
- The lint gate does not run with type information.
- A component is typed as `React.FC`, or a new component uses `forwardRef`.

**`react-component-architecture` — integrated, not blocking.** This domain
holds no veto. Its rules are findings on a review pass, and one of them fails a
task only where a blocking domain fails with it. Report each of these:

- An effect whose only work is to compute a value from props or state.
- Data from the backend held in `useState`, or held in a context.
- An index key on a list that sorts, filters, or reorders.
- A component with more than five `useState` calls, or a component file past
  200 lines.
- A suspending component with no fallback, or with no reachable error boundary.
- A fallback whose shape differs from the content that replaces it.
- A promise that `use()` reads and the same render creates.
- A validation error that throws rather than returns as state.
- A `useEffectEvent` value inside a dependency array, or passed to a child.
- An effect with no cleanup, which Strict Mode exposes in development.
- A hand-written `useMemo`, `useCallback`, or `memo` with no stated
  measurement, in a project that enables the React Compiler.
- Three or more boolean props that change the layout of one component.

**`project-structure-and-tooling` — integrated, not blocking.** This domain
holds no veto. Its rules are findings on a review pass, and one of them fails a
task only where a blocking domain fails with it. Report each of these:

- Application code outside `src/`, or a config file inside it.
- A file in a shared folder that one consumer uses.
- An import of another feature's internal path, or a relative path of three
  parent steps or more.
- A barrel outside a feature, or a feature barrel that the same feature
  imports.
- A lint invocation with no `--max-warnings=0`, or `next lint` still in a
  script or in CI.
- A flat config that takes `eslint-config-next/core-web-vitals` and not
  `eslint-config-next/typescript`.
- An `eslint-disable` with no reason after two hyphens, or a file-level
  disable.
- A CI install with no `--frozen-lockfile`, or a hand-edited lockfile.
- `packageManager`, `engines.node`, and `.nvmrc` that disagree, or pnpm 11 on
  Node 20.
- No `minimumReleaseAge`, or a new dependency with no stated replacement,
  size, and maintenance status.
- A hand-edited file under `src/api/generated/`, or CI that proves the
  generated client against neither the compiler nor a diff.
- A pre-commit, pre-push, or commit-msg hook that a fresh clone does not
  install, or a tracked `.env` file.
- A file move that the request did not ask for.

**`django-drf-api-contract` — integrated, blocking.** The task fails when any
one of these holds:

- The schema is absent, and the code proceeds on a guessed response shape.
- `COMPONENT_SPLIT_REQUEST` is `False`, so one component describes both a
  request and a response.
- CI does not run `api:generate` and then the typecheck, so a renamed
  serializer field reaches production.
- A URL literal for the backend sits outside the client module.
- A trailing slash on a write does not match the backend.
- A request carries no timeout, or no abort signal.
- A POST or a PATCH is retried with no idempotency key.
- A failure reaches a component in the shape the server sent, rather than as
  an `ApiError`.
- A 400 field dictionary renders in a toast rather than beside the field.
- One base URL serves both the server and the browser.
- An API token, or the internal address of Django, sits in a `NEXT_PUBLIC_`
  variable.
- A proxy Route Handler takes its destination host from the request.
- A write under session auth carries no `X-CSRFToken`, or a credentialed
  response carries a wildcard origin.
- Two modules disagree on the case convention.

**Conflict rule.** security > accessibility > correctness > performance >
developer convenience. No level trades down to satisfy a level above it. When
a requirement appears to force such a trade, say so and stop rather than take
it silently.
