---
name: frontend-production-engineer
description: >-
  Frontend engineering for Next.js and TypeScript against a Django and
  DRF backend. Use whenever frontend work is planned, written, or
  reviewed and the task touches App Router files (app/, layout.tsx,
  page.tsx, proxy.ts, next.config.ts, tsconfig.json), the "use client"
  or "use server" boundary, await params or cookies(), "use cache" and
  revalidateTag, a Server Action, a React hook (useState, useEffect),
  a cast (any), a Zod schema (safeParse), the DRF contract (an OpenAPI
  schema, drf-spectacular, a generated client, a pagination envelope,
  CORS, CSRF), the client cache and state (TanStack Query, useQuery,
  staleTime, a search param), live data (a WebSocket, server-sent
  events, a reconnect, a pushed event), auth (login, logout, a
  session, an httpOnly cookie, a token refresh, a protected route, 401
  or 403, a permission), or the project setup (package.json,
  eslint.config.ts), and the agent has to verify the installed
  versions, plan first, and hold the work to a definition of done,
  even if "frontend" is never used.
license: MIT
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
metadata:
  author: n-shadloo
  version: 1.7.0
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
| The route tree and the request — `app/`, `layout.tsx`, `page.tsx`, `template.tsx`, `loading.tsx`, `error.tsx`, `global-error.tsx`, `not-found.tsx`, `forbidden.tsx`, `unauthorized.tsx`, `default.tsx`, `route.ts`, `instrumentation.ts`; the folder tokens `(group)`, `_folder`, `[param]`, `[...slug]`, `[[...slug]]`, `@slot`, `(.)`, `(..)`, `(...)`; a parallel or intercepting route, a modal that breaks on reload; `await params`, `await searchParams`, `cookies()`, `headers()`, `draftMode()`, `PageProps`, `LayoutProps`, `next typegen`, `redirect()`, `notFound()`, `generateStaticParams`; `middleware.ts` renamed to `proxy.ts`, the `proxy` export, `skipProxyUrlNormalize`, a redirect or a locale rule at the network boundary, an auth check that belongs in a layout, CVE-2025-29927; `next.config.ts` keys — `typedRoutes`, `turbopack`, `images.remotePatterns`, `serverRuntimeConfig`, `experimental.ppr`, `next/legacy/image`, `next lint`; `NEXT_PUBLIC_` variables; Node 20.9, the Next 15 to 16 upgrade, a codemod, `@next-codemod-error`; the errors "Cannot access Request information synchronously", "Dynamic APIs are Asynchronous", "middleware ... renamed to proxy". Not here: `useQuery` and `staleTime` (`references/server-state-and-query-cache.md`), CSP and header values (domain 17), metadata content (domain 18), locale message files (domain 19). | `references/app-router-structure.md` |
| The boundary inside a route — `"use client"`, `"use server"`, a Server Component, a Client Component, a directive on a layout or a page, a provider that needs the client, a client leaf, `server-only`, `client-only`; a prop that fails to serialize, a class instance or a callback across the boundary, "Only plain objects can be passed"; a streamed promise, `use()`, the `"use client"` directive on an `error.tsx`, `useSearchParams()`; a fetch in `useEffect` that belongs on the server; a hydration error, "Text content does not match server-rendered HTML", `suppressHydrationWarning`, `typeof window`, `localStorage` in a render. Not here: which component holds the state (`references/state-and-effects.md`), the granularity of a boundary inside a route, the shape of a fallback, and the React 19 Actions (`references/suspense-and-actions.md`), the client cache (`references/server-state-and-query-cache.md`), bundle bytes and INP (domain 16), the accessible name of a control (domain 10). | `references/server-and-client-components.md` |
| The call to the backend — a data access layer, a DAL module, where a fetch belongs, a Server Component fetch, a Route Handler, a BFF or a proxy in front of Django, a browser call to Django, CORS on a session cookie; a Server Action, `<form action>`, an action that must authorize, validate, mutate, invalidate, and redirect in that order, a `redirect()` swallowed by a catch; a generated client, `openapi-typescript`, a drf-spectacular schema, a renamed serializer field, the DRF pagination envelope `{count, next, previous, results}`, a DRF error envelope; `prefetchQuery`, `dehydrate`, `HydrationBoundary`, a client that refetches after a prefetch; two sources for one resource. Not here: the schema generation config and the contract itself (domain 05), the query keys and the mutation state (`references/server-state-and-query-cache.md`), the field-level error mapping in a form (domain 11), the Django query behind a slow endpoint (sibling skill `django-performance-optimizer`). | `references/data-access-and-mutations.md` |
| The lifetime of a response — `cacheComponents`, Cache Components, the previous caching model, `"use cache"`, `cacheLife`, `cacheTag`, `unstable_cache`, `fetch` `cache` and `next.revalidate` and `next.tags`, route segment `revalidate` and `dynamic`; `revalidateTag`, `updateTag`, `refresh()`, `revalidatePath`, `router.refresh()`, the Router Cache, read-your-writes, stale data after a mutation; Partial Prerendering, a static shell with a dynamic hole, a route that is unexpectedly dynamic or unexpectedly static, the build report symbols, `connection()`, `unstable_noStore`, `Math.random()` or `new Date()` on a route; one user seeing another user's data. Not here: `staleTime` and the client query cache (`references/server-state-and-query-cache.md`), the `Cache-Control` header and the CDN (domain 22), whether the data may be cached at all (domain 17), the Django-side cache (sibling skill `django-performance-optimizer`). | `references/caching-and-revalidation.md` |
| The compiler and the gates that prove it — `tsconfig.json`, `compilerOptions`, `strict`, `strictNullChecks`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `verbatimModuleSyntax`, `isolatedModules`, `moduleResolution`, `moduleDetection`, `noImplicitOverride`, `noPropertyAccessFromIndexSignature`, `skipLibCheck`, `baseUrl`, `paths`, the `next` plugin; `next-env.d.ts`, a `*.d.ts` file, a generated type that someone edited by hand; `tsc --noEmit`, a typecheck that CI runs and the build does not, `typescript.ignoreBuildErrors`, `useTypeScriptCli`, a green build with a red editor; `eslint.config.ts`, typescript-eslint, `strictTypeChecked`, `stylisticTypeChecked`, `projectService`, `disableTypeChecked`, `no-explicit-any`, `no-unsafe-assignment`, `no-floating-promises`, `no-misused-promises`, `switch-exhaustiveness-check`, `ban-ts-comment`; `@ts-ignore`, `@ts-expect-error`, a suppression with no reason; `vitest --typecheck`, `expectTypeOf`, a `*.test-d.ts` file, `type-coverage`; a slow editor, a slow compiler, `--extendedDiagnostics`, `--generateTrace`, the errors `TS2589` and `TS1484`; TypeScript 5.9, 6.0, or 7.0. Not here: the folder layout and the Prettier config (domain 04), the test runner and the contract test (domain 20), the CI pipeline and the Docker build (domain 22). | `references/typescript-config-and-enforcement.md` |
| The vocabulary that models a value — `any`, `unknown`, `never`, a cast, `as`, a non-null `!`, a value that is possibly undefined, the errors `TS2322`, `TS2345`, `TS2532`, `TS18048`, `TS7053`, `TS2366`; a discriminated union, a status flag beside a data field, an optional-everything model of a serializer, `assertNever`, an exhaustive `switch`, a `Result` type, "make illegal states unrepresentable"; a branded id, `.brand()`, a `UserId` passed where an `OrderId` belongs; `satisfies`, `as const`, `as const satisfies`, a config object that loses its literal type; `enum` against a const object and a derived union; `keyof`, `typeof`, `infer`, a mapped type, a conditional type, `Prettify`, `NoInfer`, a type predicate, `asserts`; `type` against `interface`, `declare module`, a package with wrong types; `React.FC`, `PropsWithChildren`, `ComponentProps`, `forwardRef`, `Ref`, `RefObject`, `MutableRefObject`, `useRef` with no argument, a context read through a `!`, `React.JSX`. Not here: the composition and the list key (`references/component-composition.md`), the render rules and the effect rules (`references/state-and-effects.md`), the theme object behind a `satisfies` (domain 09), the form state type and the field error map (domain 11). | `references/type-modeling-and-narrowing.md` |
| A value that enters the program — `res.json()` cast to a type, a `fetch` response, `searchParams` read as data, `localStorage`, `sessionStorage`, `process.env`, a webhook body, `FormData`; "parse at the boundary", "type the response", a run-time crash on data that TypeScript called typed; Zod 4 — `z.object`, `safeParse`, `z.infer`, `z.input`, `z.output`, `z.email`, `z.discriminatedUnion`, `.brand()`, `.superRefine()`, `.check()`, `z.prettifyError`, `z.treeifyError`, `z.flattenError`, `error.issues`; the Zod 3 calls that Zod 4 removed — `error.errors`, `.format()`, `.flatten()`, `z.string().email()`, `z.nativeEnum()`, `.merge()`, `.ip()`, `.cidr()`; the generated `paths` and `components` types, `openapi-typescript`, a type hand-copied from the OpenAPI schema, two sources for one shape; a DRF field that is `required=False`, `allow_null=True`, or `blank=True`, a `nullable: true` that the generated type does not carry, `null` at run time where the type promised a value, the pagination envelope typed as an array, the DRF error body `{field: [msg]}`, `drf-standardized-errors` and its `type` and `attr` fields; `@t3-oss/env-nextjs`, an environment variable that is `undefined` in the browser. Not here: the drf-spectacular config and the codegen command (domain 05), the query keys and the mutation state (`references/server-state-and-query-cache.md`), the resolver and the field array in a form (domain 11), the MSW handler and the contract test (domain 20), the server-side serializer (sibling skill `django-api-contract`). | `references/boundary-validation-and-api-types.md` |
| The shape of a component and the way that parts compose — a God component, a component file past 200 lines, more than five `useState` calls, a decomposition, "single responsibility"; three or more boolean props that change the layout, composition over configuration, `children`, a slot, a named part, a compound component, `Tabs.List`, `Tabs.Panel`, a render prop, a polymorphic component, `asChild`, `Slot`, the Base UI `render` prop, `useRender`, `data-slot`; controlled against uncontrolled, `value` and `onChange` against `defaultValue`, `useControllableState`, "A component is changing an uncontrolled input to be controlled"; `ref` as a prop against `forwardRef` in new code, `useImperativeHandle`, a parent that reaches into a child; a list `key`, an index key, "Each child in a list should have a unique key prop", a row that keeps the state of another row after a sort or a filter; a headless hook, a custom hook that returns markup; shadcn/ui on Base UI, Radix Primitives, `usehooks-ts`, `prop-types`. Not here: where the state itself lives and whether an effect is correct (`references/state-and-effects.md`), the `<Suspense>` boundary and the Action hooks (`references/suspense-and-actions.md`), `React.FC` and `PropsWithChildren` and `ComponentProps` (`references/type-modeling-and-narrowing.md`), the classes and the tokens on a part (domain 09), the ARIA role and the focus order (domain 10), the virtualiser for a long list (domain 12). | `references/component-composition.md` |
| Where a value lives and when an effect is correct — `useState`, `useReducer`, `useEffect`, `useLayoutEffect`, `useContext`, `createContext`, `useSyncExternalStore`, `useEffectEvent`, `useMemo`, `useCallback`, `memo`, `<Profiler>`; colocate state, lift state up, prop drilling, a global store, a derived value, a filtered list held in state, an effect that sets state, an effect that resets state when a prop changes, a `key` that remounts a component, five `useState` calls for five form fields, a context that holds a list from the backend; a subscription, `window.matchMedia`, a browser API read during a render, a socket that reconnects on an unrelated change, a stale closure, a dependency array that lies; Strict Mode, `reactStrictMode`, an effect that runs twice in development, a missing cleanup; the Rules of React, purity, a mutation during a render, `eslint-plugin-react-hooks`, `rules-of-hooks`, `exhaustive-deps`, `set-state-in-render`, `set-state-in-effect`, `reactCompiler`, `babel-plugin-react-compiler`, `"use memo"`, `"use no memo"`, `compilationMode`, a compiler bailout, `eslint-plugin-react-compiler`; CVE-2025-55182 and the React 19.2.4 floor; the errors "Rendered more hooks than during the previous render", "Cannot update a component while rendering a different component", "Too many re-renders", "Maximum update depth exceeded", "Missing getServerSnapshot, which is required for server-rendered content", "Hooks can only be called inside the body of a function component". Not here: `useQuery` and `staleTime` and the query cache (`references/server-state-and-query-cache.md`), the shape of the component around the state (`references/component-composition.md`), the boundary and the Action (`references/suspense-and-actions.md`), the `"use client"` directive itself (`references/server-and-client-components.md`), the type-aware lint config (`references/typescript-config-and-enforcement.md`), the INP that a re-render costs (domain 16). | `references/state-and-effects.md` |
| The boundary that renders while a value is absent, and the Action that changes it — `<Suspense>`, a fallback, a skeleton that does not match the content, a page that shows one spinner, a section boundary, a widget boundary, layout shift when the data arrives; `ErrorBoundary`, `react-error-boundary`, `fallbackRender`, `resetErrorBoundary`, `useErrorBoundary`, a panel that must fail alone; `use(` on a promise, a promise created in a render, a fallback that never resolves, `use()` inside a `try`; a React 19 Action, `useActionState`, `useFormStatus`, `useOptimistic`, `startTransition`, `useTransition`, `formAction`, a submit button that must disable while pending, an optimistic value that must roll back; a DRF 400 rendered beside a field, a validation error that was thrown, a 5xx that must reach a boundary; `<Activity>`, `visible` and `hidden`, hidden UI that must keep its state, a tab panel that loses a scroll position; `<title>` or `<meta>` rendered from a component, `preload`, `preinit`, `prefetchDNS`, `preconnect`. Not here: `loading.tsx` and `error.tsx` as route files (`references/app-router-structure.md`), the body of the Server Action and its authorize, validate, mutate, invalidate, redirect order (`references/data-access-and-mutations.md`), the DRF error envelope shapes (`references/boundary-validation-and-api-types.md`), the mutation state in the query cache (`references/server-state-and-query-cache.md`), the resolver and the field array (domain 11), the choreography of a transition (domain 14), the words in the message (domain 15). | `references/suspense-and-actions.md` |
| Where a file goes and which module may import it — `src/`, `src/app`, `src/features`, `src/components/ui`, `src/components/common`, `src/lib`, `src/server`, `src/config`, feature-first, layer-first, colocation, a shared layer, dependency direction, a public API, a private folder that holds too much; a barrel file, `index.ts`, `export *`, a global barrel, a deep relative import `../../../..`, the `@/*` path alias, `optimizePackageImports`, `transpilePackages`; `eslint-plugin-boundaries`, `boundaries/element-types`, `no-restricted-paths`, `dependency-cruiser`, `.dependency-cruiser.js`, `sheriff`, `sheriff.config.ts`, a cycle, an orphan module; `src/api/generated`, `api:generate`, a generated client that someone edited, the schema artifact, the drift check; a monorepo, a workspace, `pnpm-workspace.yaml`, `turbo.json`, Turborepo, Nx; a repository restructure inside a feature task; the errors "Importing elements of type ... is not allowed", "is not allowed in elements of type", "Cannot find module" from a missing alias. Not here: the folder tokens `(group)`, `_folder`, and `@slot`, and the route files (`references/app-router-structure.md`), the `server-only` and `client-only` guards (`references/server-and-client-components.md`), `tsconfig.json` and the `paths` value (`references/typescript-config-and-enforcement.md`), the rest of the lint config array (`references/lint-format-and-scripts.md`), the drf-spectacular config and the generator (domain 05), the tokens that `components/ui` renders (domain 09). | `references/directory-and-module-boundaries.md` |
| The checks over the code and the commands that run them — `package.json` scripts, `dev`, `build`, `lint`, `lint:fix`, `format`, `typecheck`, `test`, `test:watch`, `test:e2e`, `analyze`, `pnpm check`, `pnpm dlx`, `--max-warnings=0`, a warning that CI accepts; `eslint.config.ts`, `eslint.config.mjs`, a flat config, `defineConfig`, `eslint-config-next`, `core-web-vitals`, `eslint-config-next/typescript`, an `ignores` entry, a lint rule that reports nothing; `next lint`, the `eslint` key in `next.config.ts`, `next-lint-to-eslint-cli`, a CI step that lints nothing; Prettier, `.prettierrc`, Biome, `biome.json`, `prettier-plugin-tailwindcss`, `tailwindStylesheet`, `eslint-plugin-tailwindcss`, `eslint-plugin-better-tailwindcss`, `.editorconfig`; `eslint-disable`, `eslint-disable-next-line`, a suppression with no reason; `next typegen` before a typecheck; `AGENTS.md`, `.cursorrules`, `.cursor/rules`, `.vscode/settings.json`, `.vscode/extensions.json`; the errors "Parsing error: ESLint was configured to run on", "next lint is deprecated and will be removed in Next.js 16", "It looks like you're trying to use tailwindcss directly as a PostCSS plugin". Not here: `projectService`, `strictTypeChecked`, and `tsc --noEmit` (`references/typescript-config-and-enforcement.md`), `eslint-plugin-react-hooks` and `reactCompiler` (`references/state-and-effects.md`), the boundaries block (`references/directory-and-module-boundaries.md`), the hook that calls a script (`references/dependencies-and-git-workflow.md`), the Tailwind theme (domain 09), the test layout and the coverage threshold (domain 20), the CI workflow (domain 22). | `references/lint-format-and-scripts.md` |
| What enters the repository and how a change leaves it — pnpm, npm, yarn, bun, `pnpm install --frozen-lockfile`, `pnpm-lock.yaml`, a lockfile conflict, a hand-edited lockfile, `packageManager`, Corepack, `corepack enable`, Volta, `.nvmrc`, `engines.node`, `.npmrc`, `overrides`; `minimumReleaseAge`, `minimumReleaseAgeExclude`, a cooldown, `onlyBuiltDependencies`, `allowBuilds`, `ignore-scripts`, a lifecycle script, `pnpm audit`, a new dependency and its justification; Renovate, `renovate.json`, Dependabot, `.github/dependabot.yml`, `automerge`, `matchUpdateTypes`, `fetch-metadata`; lefthook, `lefthook.yml`, Husky, `.husky/`, `lint-staged`, `stage_fixed`, a `prepare` script, a hook that never fires; commitlint, `commitlint.config.js`, a Conventional Commit, a commit-msg hook; `.gitignore`, a committed `.env.local`, gitleaks, `.gitleaks.toml`, a secret in the history; `CODEOWNERS`, `.gitattributes`, `linguist-generated`. Not here: the body of a script that a hook calls (`references/lint-format-and-scripts.md`), the folder that a new file goes into (`references/directory-and-module-boundaries.md`), the `NEXT_PUBLIC_` prefix (`references/app-router-structure.md`), whether a package is malicious (domain 17), the CI workflow and the deploy (domain 22), the server-side secret storage (sibling skill `secure-code-auditor`). | `references/dependencies-and-git-workflow.md` |
| The schema that the types come from — drf-spectacular, `SPECTACULAR_SETTINGS`, `COMPONENT_SPLIT_REQUEST`, `COMPONENT_SPLIT_PATCH`, `OAS_VERSION`, `@extend_schema`, `@extend_schema_field`, `@extend_schema_serializer`, `OpenApiExample`, `OpenApiParameter`, `ENUM_NAME_OVERRIDES`, `ENUM_ADD_EXPLICIT_BLANK_NULL_CHOICE`, `schema.yml`, `openapi.json`, `/api/schema/`, `drf-yasg` and Swagger 2.0, django-ninja and OpenAPI 3.1; `openapi-typescript`, `openapi-fetch`, Orval, `@hey-api/openapi-ts`, `@kubb/*`, `swagger-typescript-api`, `openapi-generator`, `operationId`, the `paths` type, `components["schemas"]`; `api:generate`, a client generated from a live URL, a stale schema in production, `oasdiff`, `openapi-diff`, a breaking change on an enum, an enum emitted as a TypeScript `enum`; a `SerializerMethodField` typed `any`, `XRequest` against `X`, a response field that became optional, upstream issue #810; camelCase against snake_case, `djangorestframework-camel-case`, `camelize_serializer_fields`, `humps`, `ts-case-convert`. Not here: the types that a DRF construct produces and the parse over them (`references/boundary-validation-and-api-types.md`), the output folder and its `.gitignore` entry (`references/directory-and-module-boundaries.md`), the `package.json` script surface (`references/lint-format-and-scripts.md`), the client that sends a request (`references/api-client-and-request-safety.md`), the serializer, the viewset, and the deprecation of a field (sibling skill `django-api-contract`). | `references/openapi-schema-and-codegen.md` |
| The request itself, and the failure that comes back — `apiClient`, `createClient`, `endpoints.ts`, a base URL, a `fetch("/api/...")` literal in a component, `DJANGO_URL` against `NEXT_PUBLIC_API_BASE_URL`, an ECONNREFUSED from a server fetch inside Docker; a trailing slash, `APPEND_SLASH`, a 301 that drops a POST body, "you called this URL via POST"; `AbortController`, `AbortSignal.timeout`, `AbortSignal.any`, a request that never ends, a `DOMException` against a `TypeError`; a retry, an exponential backoff, `Idempotency-Key`, `Retry-After`, a 429 loop, a throttle, a duplicate row from a retried POST; `normalizeApiError`, `ApiError`, `fieldErrors`, `retryable`, `ErrorDetail`, `non_field_errors`, `detail`, `[object Object]` in a toast; a 204 with no body, "Unexpected end of JSON input", a 500 that returns HTML; the `next` and `previous` URLs, a computed page offset, `CursorPagination` with no `count`; `FormData`, a multipart boundary, an empty `request.FILES`. Not here: which module holds the call and the order inside a Server Action (`references/data-access-and-mutations.md`), the `Paginated<T>` type and the parse (`references/boundary-validation-and-api-types.md`), the schema and the generator (`references/openapi-schema-and-codegen.md`), CORS and the CSRF header (`references/cross-origin-and-bff-proxy.md`), `queryKey` and `staleTime` (`references/server-state-and-query-cache.md`), the single-flight token refresh (`references/session-and-token-lifecycle.md`), the upload progress bar (domain 13). | `references/api-client-and-request-safety.md` |
| The origin boundary that the browser enforces — `django-cors-headers`, `CORS_ALLOWED_ORIGINS`, `CORS_ALLOW_CREDENTIALS`, `CORS_ALLOW_HEADERS`, a preflight, an `OPTIONS` request, "No 'Access-Control-Allow-Origin' header is present", a wildcard beside `credentials: "include"`, two `Access-Control-Allow-Origin` values, a request that never reaches the Django log; `X-CSRFToken`, the `csrftoken` cookie, `ensure_csrf_cookie`, `CSRF_HEADER_NAME`, `CSRF_TRUSTED_ORIGINS` and its scheme, "CSRF Failed: CSRF token missing", `SESSION_COOKIE_SAMESITE`, `httpOnly`, `Secure`, a cookie that the browser drops on a cross-site call; a BFF, a rewrite in front of Django, a Route Handler proxy, a fixed upstream host, SSRF, `169.254.169.254`, a `?target=` parameter, a body size cap on a proxy. Not here: `proxy.ts`, its permitted work, and CVE-2025-29927 (`references/app-router-structure.md`), the choice between a Server Component fetch, a Route Handler, and a browser call (`references/data-access-and-mutations.md`), the timeout and the error shape (`references/api-client-and-request-safety.md`), the session strategy and the cookie prefixes (`references/session-and-token-lifecycle.md`), the redirect after a 401 (`references/route-protection-and-permissions.md`), the CSP and the response headers (domain 17), the DRF permission class and the server settings (sibling skill `secure-code-auditor`). | `references/cross-origin-and-bff-proxy.md` |
| The cache that holds server state — `@tanstack/react-query`, `queryOptions`, `infiniteQueryOptions`, `useQuery`, `useSuspenseQuery`, `useInfiniteQuery`, `useMutation`, `QueryClient`, `QueryClientProvider`, `staleTime`, `gcTime`, `select`, `enabled`, `initialData`, `placeholderData`, `keepPreviousData`, `isPending` against `isFetching` against `isLoading`; a key factory, a static key over a filtered list, a list that flickers, an inline `queryFn` in a component, a `QueryClient` at module scope, one user who sees another user's rows; `prefetchQuery`, `dehydrate`, `HydrationBoundary`, a refetch straight after the hydration; `invalidateQueries`, `refetchQueries`, `setQueryData`, `cancelQueries`, `onMutate`, `onSettled`, an optimistic value with no rollback, a saved record that the list does not show; `initialPageParam`, `getNextPageParam`, `getPreviousPageParam`, DRF `PageNumberPagination`, `LimitOffsetPagination`, and `CursorPagination` behind an infinite query; `refetchInterval`, a poll with no stop condition; the loading, empty, error, and ready states of a view; `cacheTime`, `keepPreviousData: true`, `useErrorBoundary`, and `isInitialLoading` from version 4; `@tanstack/eslint-plugin-query`; the errors "No QueryClient set, use QueryClientProvider to set one" and "Missing queryFn". Not here: the taxonomy, the URL, and the store (`references/client-and-url-state.md`), the page that prefetches (`references/data-access-and-mutations.md`), the typed client and `ApiError` (`references/api-client-and-request-safety.md`), the server cache and `updateTag` (`references/caching-and-revalidation.md`), the token refresh after a 401 (`references/session-and-token-lifecycle.md`), the push transport (`references/push-transport-and-connection.md`), the event that writes into this cache (`references/live-events-and-cache-merge.md`), the field error map (domain 11), the column model (domain 12). | `references/server-state-and-query-cache.md` |
| Where a value lives when the backend does not own it — the state taxonomy, one owner for one value, a filter or a sort or a page or a tab or a search term held in `useState`, a copied link that loses the view, a back button that changes nothing; `useSearchParams`, `nuqs`, `useQueryState`, `useQueryStates`, `parseAsString`, `parseAsInteger`, `.withDefault()`, `throttleMs`, `createLoader`, `queryTypes` from version 1, the error "Missing Suspense boundary with useSearchParams"; Zustand, `create`, `createStore`, `useStore`, `useShallow`, `persist`, `getState()`, a store at module scope that leaks between requests, a store that holds a list from the backend, a theme that flashes on the first paint; Jotai, Valtio, Redux Toolkit, RTK Query, `swr`. Not here: every value that the backend owns (`references/server-state-and-query-cache.md`), the derived value and the effect (`references/state-and-effects.md`), the hydration error itself (`references/server-and-client-components.md`), the server `searchParams` prop (`references/app-router-structure.md`), the parse over a URL value (`references/boundary-validation-and-api-types.md`), the field value before a submit (domain 11), the sort model of a table (domain 12), the locale segment (domain 19). | `references/client-and-url-state.md` |
| The credential and how long it lives — login, logout, sign-in, register, a session, `SessionAuthentication`, an access token, a refresh token, a bearer token, `Authorization`, `AUTH_HEADER_TYPES`; `httpOnly`, `Secure`, `SameSite=Lax` against `Strict` against `None`, `Max-Age`, `Domain`, `Path`, the `__Host-` and `__Secure-` prefixes, the 4 KB cookie ceiling, a `Set-Cookie` that a proxy rewrote, the console message `Cookie "myCookie" rejected because it has the "SameSite=None" attribute but is missing the "secure" attribute`; `localStorage.setItem('access', token)`, a token or a permission list in web storage; a silent refresh, a single-flight refresh, concurrent 401s, a refresh storm, `ROTATE_REFRESH_TOKENS`, `BLACKLIST_AFTER_ROTATION`, token rotation, reuse detection, a blacklist, `SIGNING_KEY`, `LEEWAY`, clock skew, an `exp` decoded on the client, a session that ends mid-visit, a session that a 500 ends; `queryClient.clear()`, `BroadcastChannel`, a cross-tab logout, a back navigation that shows the previous user; `djangorestframework-simplejwt`, `TokenObtainPairView`, `TokenRefreshView`, `/api/token/refresh/`, `dj-rest-auth`, `django-allauth` headless, `_allauth`, `X-Session-Token`, `iron-session`, `sealData`, Auth.js, NextAuth, `knox`, DRF `TokenAuthentication`, PKCE, the OAuth implicit flow, RFC 6749 and RFC 7636. Not here: the gate on a route, a Server Action, or a control (`references/route-protection-and-permissions.md`), the `X-CSRFToken` exchange and the CORS preflight (`references/cross-origin-and-bff-proxy.md`), `normalizeApiError` and the retry rule (`references/api-client-and-request-safety.md`), the cache that a logout clears (`references/server-state-and-query-cache.md`), the carrier on a socket handshake (`references/push-transport-and-connection.md`), the login form fields (domain 11), the CSP (domain 17), the DRF settings and the password hash (sibling skill `secure-code-auditor`). | `references/session-and-token-lifecycle.md` |
| Who may proceed, and what the interface shows them — `verifySession`, `getCurrentUser`, a data access layer, React `cache()` for one session read, a protected route, a page gate against a layout gate, a route group `(auth)` and `(app)`; an auth check in `proxy.ts` alone, CVE-2025-29927, CVE-2026-64642, a forged `x-middleware-subrequest` header; a Server Action that takes a `userId`, a `role`, a `tenantId`, or an `isAdmin` argument, `curl` against a Server Action with no cookie, CVE-2026-64643 and a Server Function endpoint that the client chunks disclose, "authenticate within the boundary"; `redirect('/login?next=')`, `unauthorized()`, `forbidden()`, `unauthorized.tsx`, `forbidden.tsx`, `authInterrupts`, an experimental auth interrupt, a 307 loop between `/login` and `/dashboard`, an open redirect through a `?next=` value that names another host, or `?next=//evil.example`; RBAC, a role, a group, a permission list, an object permission, a feature flag, `<Can>`, `usePermission`, a hidden button that is the whole gate, a blank page in place of a 403 explanation; a multi-tenant application, an active organization, cross-tenant cache bleed; a cached page that renders the name of a user, protected data in an RSC payload or a prefetch. Not here: the credential, the cookie attributes, and the refresh (`references/session-and-token-lifecycle.md`), the permitted work of `proxy.ts` and the route files themselves (`references/app-router-structure.md`), the rest of the data access module and the Server Action order (`references/data-access-and-mutations.md`), the `"use cache"` rules (`references/caching-and-revalidation.md`), the key factory (`references/server-state-and-query-cache.md`), the focus on a 403 message (domain 10), the Server Action supply chain (domain 17), the DRF permission class and the object-level check (sibling skill `secure-code-auditor`). | `references/route-protection-and-permissions.md` |
| The connection that pushes data, and how long it lives — live, realtime, a notification feed, a chat, a progress feed, push, subscribe; `WebSocket`, `new WebSocket()`, `EventSource`, `ws://`, `wss://`, `Sec-WebSocket-Protocol`, a subprotocol that carries a token, a token in a socket URL, a single-use ticket, an `Origin` check on a handshake; `text/event-stream`, `Last-Event-ID`, the `data:` and `retry:` frame fields, `ReadableStream`, `TextDecoderStream`, NDJSON, one response read in parts, an `AbortController` over a stream; `StreamingHttpResponse`, an async iterator under ASGI, Django Channels, `ProtocolTypeRouter`, `URLRouter`, `AsyncJsonWebsocketConsumer`, a channel layer, `group_send`; a reconnect, a reconnect storm, an exponential backoff, jitter, a heartbeat, a ping and a pong, a zombie connection, a socket that dies every 60 seconds or every 100 seconds, a `readyState` that stays `OPEN` with nothing on it; close code 1000, 1001, 1006, 1008, 1011, 1012, 1013, 4001, 4003, a 1006 that logs the user out, the error "closed before the connection is established"; `document.hidden`, `visibilitychange`, a connection in a background tab; a socket in a component body, a socket for each list row, the six-connection limit of HTTP/1.1, `BroadcastChannel` for one connection over many tabs; `partysocket`, `reconnecting-websocket`, `@microsoft/fetch-event-source`, `socket.io`, WebTransport, `experimental_streamedQuery`; `proxy_http_version 1.1`, `Upgrade`, `Connection: upgrade`, `proxy_read_timeout`, `proxy_buffering off`, `X-Accel-Buffering`, a Cloudflare idle close, events that stream in development and batch in production. Not here: the frame that arrives and the cache that it writes into (`references/live-events-and-cache-merge.md`), `refetchInterval` and the poll (`references/server-state-and-query-cache.md`), where the credential lives and the refresh over it (`references/session-and-token-lifecycle.md`), the deadline and the `ApiError` of an ordinary request (`references/api-client-and-request-safety.md`), the effect rules and `useSyncExternalStore` (`references/state-and-effects.md`), the same-origin rewrite (`references/cross-origin-and-bff-proxy.md`), the politeness of a live announcement (domain 10), the upload progress bar (domain 13), the Nginx file itself (domain 22), the consumer and the queue (sibling skill `django-async-jobs`). | `references/push-transport-and-connection.md` |
| The message that arrives, and what it changes — an event envelope, a `type` discriminator, an `origin` field, a `payload` from a serializer; `onmessage`, `JSON.parse(event.data)`, a malformed frame, a frame that is not JSON, an event type that the build cannot name, a feed that stops with the connection still open, a handler that threw; `z.discriminatedUnion` over an event, `safeParse` on a frame, a dropped-frame counter, `assertNever` in an event switch; `setQueryData` from a pushed event, `invalidateQueries` after a push, `removeQueries`, an invalidation that never refetches, an `isInvalidated` mark that a later write cleared, pushed rows held in `useState`; a ticker, a cursor feed, a render thrash, a coalesce into one animation frame, the React Profiler commit count; a duplicate row after your own edit, the echo of an optimistic write, `X-Origin-Id`; a resync after a reconnect, a gap in the feed, a `Last-Event-ID` replay; a renamed event type that stops the screen in silence, a channel layer that is down and a view that renders empty. Not here: the transport, the handshake, the reconnect, the heartbeat, and the close code (`references/push-transport-and-connection.md`), the key factory, the invalidation of a mutation, and the optimistic rollback (`references/server-state-and-query-cache.md`), the parse doctrine and the DRF envelopes (`references/boundary-validation-and-api-types.md`), `assertNever` itself (`references/type-modeling-and-narrowing.md`), the generated payload types (`references/openapi-schema-and-codegen.md`), the live row in a table (domain 12), the metric behind the counter (domain 21), the event as a versioned published surface (sibling skill `django-api-contract`). | `references/live-events-and-cache-merge.md` |

The operating doctrine is integrated but has no row, because it is always in
effect and lives in this file rather than in `references/`. Each release
integrates one domain and adds its rows. Write each row as trigger vocabulary,
never as a summary. A vague row is a domain that never loads.

A domain of more than one file splits on one line. The leading phrase of a row
states what the file owns, and its `Not here` clause states what it does not.

Fourteen seams cross the domains, and this table settles each one.

| The seam | Who owns what |
|---|---|
| 01 ↔ 02 | 01 owns the route file and the awaited request data. 02 owns the types that the route file uses. |
| 01 ↔ 03 | 03 decides which component holds the state. 01 states the boundary. |
| 01 ↔ 04 | 01 owns the folder tokens inside `app/`. 04 owns every folder outside `app/`. |
| 02 ↔ 04 | 02 owns `tsconfig.json` and the type-aware lint rules. 04 owns the config array that holds them, and the scripts that run them. |
| 01 ↔ 05 | 01 owns the module that holds a call, and `proxy.ts`. 05 owns the client that the module calls, and the proxy Route Handler in front of Django. |
| 02 ↔ 05 | 02 owns the type of a value and the parse that proves it. 05 owns the schema that produces the type, and the command that generates it. |
| 03 ↔ 06 | 03 owns every value that a component computes or holds. 06 owns every value that the backend produces, and the URL that carries a view. |
| 05 ↔ 06 | 05 owns the request, the timeout, and the `ApiError`. 06 owns the key, the cache entry, and the retry rule over that `ApiError`. |
| 01 ↔ 07 | 01 owns `proxy.ts`, the route files, and their permitted work. 07 owns the enforcement layers, and the rule that no one of them is the only gate. |
| 05 ↔ 07 | 05 owns the CSRF exchange, the preflight, and the cookie on a cross-origin request. 07 owns the session that the cookie carries, and its whole lifetime. |
| 06 ↔ 07 | 06 owns the key and the cache entry. 07 owns the identity that scopes the key, and the clear that a logout performs. |
| 05 ↔ 08 | 05 owns the fields inside an event payload, and the schema that types them. 08 owns the envelope around them, and the parse over it. |
| 06 ↔ 08 | 06 owns the key, the cache entry, and the poll. 08 owns the event that writes into that entry, and the reason not to poll. |
| 07 ↔ 08 | 07 owns the credential and its whole lifetime. 08 owns the carrier that a handshake takes, and the close code that ends a connection. |

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

The router table above is the integrated material at 1.7.0, and it is the
authoritative list. The rest of the order applies as the router grows.

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
one of them is not complete, however finished the feature looks. Every other
integrated domain holds no veto. Its rules are findings on a review pass, and
one of them fails a task only where a blocking domain fails with it.

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

**`react-component-architecture` — integrated, not blocking.** Report
each of these:

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

**`project-structure-and-tooling` — integrated, not blocking.** Report
each of these:

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

**`authentication-and-authorization` — integrated, blocking.** The task fails
when any one of these holds:

- A token, a refresh token, or a permission list is written to `localStorage`
  or `sessionStorage`.
- The refresh token is anywhere other than an `httpOnly; Secure; SameSite`
  cookie, or the session cookie carries no `__Host-` or `__Secure-` prefix.
- A `SameSite=None` cookie carries no `Secure`.
- A protected page trusts the redirect of `proxy.ts`, and calls no
  `verifySession()`.
- The session read is not memoised with React `cache()`, so one request
  produces more than one read.
- The gate sits in a layout rather than in the page.
- A Server Action or a Route Handler does not verify the session inside its
  own body, or it takes an identity as a parameter.
- The refresh is not single-flight, so concurrent 401s send more than one
  refresh request.
- A 5xx ends the session, or a 401 from the refresh endpoint is retried.
- The logout leaves the server session, the query cache, another tab, or the
  Router Cache holding the previous identity.
- A redirect target from a request reaches `redirect()` with no same-origin
  check.
- `verifySession()` runs inside a `"use cache"` scope, or a route that renders
  per-user data is reported as static, or a protected RSC payload holds a
  private value with the cookies cleared.
- A permission decision has no server-side gate behind it, or a 403 renders a
  blank page.

**`data-fetching-and-state` — integrated, not blocking.** Report each of these:

- Data from the backend held in `useState`, in a context, or in a store.
- A query defined in a component, rather than as one `queryOptions` object in
  the feature `api` folder.
- A query key that omits an input which changes the result.
- A `QueryClient` at module scope, or a server client whose `staleTime` is zero.
- One resource fetched by both a Server Component and a client query on one
  page.
- A mutation with no invalidation, no cache write, and no stated reason.
- An invalidation with no key, or one that reaches past the subtree the write
  changed.
- An optimistic update with no snapshot, no rollback, or no reconciliation.
- A data view with no designed empty state, or no error state.
- A next page param computed, rather than read from the DRF `next` URL.
- A `refetchInterval` with no stated period and no stop condition.
- A filter, sort, page, tab, or search term that the URL does not carry, or a
  `useSearchParams` consumer with no `<Suspense>` boundary above it.
- A store that holds a request-specific value at module scope.

**`realtime-and-streaming` — integrated, not blocking.** Report each of these:

- A `new WebSocket` or a `new EventSource` with no written reason to prefer it
  over a poll.
- More than one connection for one application, or a connection that a
  component body creates.
- An effect that opens a connection and returns no cleanup that closes it.
- A credential in the query string of a connection URL, with no single-use
  ticket and no comment.
- A handshake that carries a cookie, where nobody confirmed the `Origin` check
  on the backend.
- A frame that reaches the cache or the state with no `safeParse`, or a
  `JSON.parse` outside a `try`.
- An event type that the build cannot name and that throws, or one that no
  counter records.
- A reconnect at a fixed period, or one with no cap, no random part, or no
  resync.
- A heartbeat that is absent, or one whose period is above the shortest idle
  timeout on the path.
- A close code 1006, 1011, 1012, or 1013 that ends the session.
- A dropped connection that renders an empty state rather than a degraded one.
- A connection that stays open in a hidden tab with no stated reason.
- Pushed rows in a `useState` beside the query cache, or an optimistic write
  whose own echo the client applies a second time.

**Conflict rule.** security > accessibility > correctness > performance >
developer convenience. No level trades down to satisfy a level above it. When
a requirement appears to force such a trade, say so and stop rather than take
it silently.
