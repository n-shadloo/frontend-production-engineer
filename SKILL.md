---
name: frontend-production-engineer
description: >-
  Frontend engineering for Next.js and TypeScript on a Django and DRF
  backend. Use when frontend work is planned, written, or reviewed
  and touches App Router files (app/, proxy.ts), the "use client" or
  "use server" boundary, await params, a Server Action, a React hook,
  a cast (any), a Zod schema, the DRF contract (OpenAPI, CORS, CSRF),
  the client cache (TanStack Query, useQuery), live data (a
  WebSocket), auth (a session, a token), security (XSS, CSP, a nonce,
  dangerouslySetInnerHTML, SSRF, a CVE), styling (Tailwind, a token,
  dark mode, shadcn), accessibility (WCAG, ARIA, a keyboard, a screen
  reader, axe), forms (useForm, a field error), data surfaces (a
  table, a chart), media (an upload, next/image), motion (an
  animation), copy (a label, an error message, an empty state),
  performance (slow, LCP, INP, CLS, bundle size, a third-party
  script, Lighthouse), or the setup (package.json, tsconfig.json,
  eslint.config.ts), and the agent must verify the installed versions
  first, even if "frontend" is never used.
license: MIT
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
metadata:
  author: n-shadloo
  version: 1.16.0
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
| The route tree and the request — `app/`, `layout.tsx`, `page.tsx`, `template.tsx`, `loading.tsx`, `error.tsx`, `global-error.tsx`, `not-found.tsx`, `forbidden.tsx`, `unauthorized.tsx`, `default.tsx`, `route.ts`, `instrumentation.ts`; the folder tokens `(group)`, `_folder`, `[param]`, `[...slug]`, `[[...slug]]`, `@slot`, `(.)`, `(..)`, `(...)`; a parallel or intercepting route, a modal that breaks on reload; `await params`, `await searchParams`, `cookies()`, `headers()`, `draftMode()`, `PageProps`, `LayoutProps`, `next typegen`, `redirect()`, `notFound()`, `generateStaticParams`; `middleware.ts` renamed to `proxy.ts`, the `proxy` export, `skipProxyUrlNormalize`, a redirect or a locale rule at the network boundary, an auth check that belongs in a layout, CVE-2025-29927; `next.config.ts` keys — `typedRoutes`, `turbopack`, `images.remotePatterns`, `serverRuntimeConfig`, `experimental.ppr`, `next/legacy/image`, `next lint`; `NEXT_PUBLIC_` variables; Node 20.9, the Next 15 to 16 upgrade, a codemod, `@next-codemod-error`; the errors "Cannot access Request information synchronously", "Dynamic APIs are Asynchronous", "middleware ... renamed to proxy". Not here: `useQuery` and `staleTime` (`references/server-state-and-query-cache.md`), CSP and header values (`references/security-headers-and-csp.md`), metadata content (domain 18), locale message files (domain 19). | `references/app-router-structure.md` |
| The boundary inside a route — `"use client"`, `"use server"`, a Server Component, a Client Component, a directive on a layout or a page, a provider that needs the client, a client leaf, `server-only`, `client-only`; a prop that fails to serialize, a class instance or a callback across the boundary, "Only plain objects can be passed"; a streamed promise, `use()`, the `"use client"` directive on an `error.tsx`, `useSearchParams()`; a fetch in `useEffect` that belongs on the server; a hydration error, "Text content does not match server-rendered HTML", `suppressHydrationWarning`, `typeof window`, `localStorage` in a render. Not here: which component holds the state (`references/state-and-effects.md`), the granularity of a boundary inside a route, the shape of a fallback, and the React 19 Actions (`references/suspense-and-actions.md`), the client cache (`references/server-state-and-query-cache.md`), the bundle bytes of a client leaf (`references/client-bundle-and-third-party-scripts.md`), the INP that it costs (`references/paint-and-interaction-cost.md`), the accessible name of a control (`references/semantics-and-accessible-names.md`). | `references/server-and-client-components.md` |
| The call to the backend — a data access layer, a DAL module, where a fetch belongs, a Server Component fetch, a Route Handler, a BFF or a proxy in front of Django, a browser call to Django, CORS on a session cookie; a Server Action, `<form action>`, an action that must authorize, validate, mutate, invalidate, and redirect in that order, a `redirect()` swallowed by a catch; a generated client, `openapi-typescript`, a drf-spectacular schema, a renamed serializer field, the DRF pagination envelope `{count, next, previous, results}`, a DRF error envelope; `prefetchQuery`, `dehydrate`, `HydrationBoundary`, a client that refetches after a prefetch; two sources for one resource. Not here: the schema generation config and the contract itself (`references/openapi-schema-and-codegen.md`), the query keys and the mutation state (`references/server-state-and-query-cache.md`), the field-level error mapping in a form (`references/form-submission-and-server-errors.md`), the Django query behind a slow endpoint (sibling skill `django-performance-optimizer`). | `references/data-access-and-mutations.md` |
| The lifetime of a response — `cacheComponents`, Cache Components, the previous caching model, `"use cache"`, `cacheLife`, `cacheTag`, `unstable_cache`, `fetch` `cache` and `next.revalidate` and `next.tags`, route segment `revalidate` and `dynamic`; `revalidateTag`, `updateTag`, `refresh()`, `revalidatePath`, `router.refresh()`, the Router Cache, read-your-writes, stale data after a mutation; Partial Prerendering, a static shell with a dynamic hole, a route that is unexpectedly dynamic or unexpectedly static, the build report symbols, `connection()`, `unstable_noStore`, `Math.random()` or `new Date()` on a route; one user seeing another user's data. Not here: `staleTime` and the client query cache (`references/server-state-and-query-cache.md`), the `Cache-Control` header and the CDN (domain 22), whether the data may be cached at all (`references/exposed-endpoints-and-destinations.md`), the Django-side cache (sibling skill `django-performance-optimizer`). | `references/caching-and-revalidation.md` |
| The compiler and the gates that prove it — `tsconfig.json`, `compilerOptions`, `strict`, `strictNullChecks`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `verbatimModuleSyntax`, `isolatedModules`, `moduleResolution`, `moduleDetection`, `noImplicitOverride`, `noPropertyAccessFromIndexSignature`, `skipLibCheck`, `baseUrl`, `paths`, the `next` plugin; `next-env.d.ts`, a `*.d.ts` file, a generated type that someone edited by hand; `tsc --noEmit`, a typecheck that CI runs and the build does not, `typescript.ignoreBuildErrors`, `useTypeScriptCli`, a green build with a red editor; `eslint.config.ts`, typescript-eslint, `strictTypeChecked`, `stylisticTypeChecked`, `projectService`, `disableTypeChecked`, `no-explicit-any`, `no-unsafe-assignment`, `no-floating-promises`, `no-misused-promises`, `switch-exhaustiveness-check`, `ban-ts-comment`; `@ts-ignore`, `@ts-expect-error`, a suppression with no reason; `vitest --typecheck`, `expectTypeOf`, a `*.test-d.ts` file, `type-coverage`; a slow editor, a slow compiler, `--extendedDiagnostics`, `--generateTrace`, the errors `TS2589` and `TS1484`; TypeScript 5.9, 6.0, or 7.0. Not here: the folder layout (`references/directory-and-module-boundaries.md`), the Prettier config (`references/lint-format-and-scripts.md`), the test runner and the contract test (domain 20), the CI pipeline and the Docker build (domain 22). | `references/typescript-config-and-enforcement.md` |
| The vocabulary that models a value — `any`, `unknown`, `never`, a cast, `as`, a non-null `!`, a value that is possibly undefined, the errors `TS2322`, `TS2345`, `TS2532`, `TS18048`, `TS7053`, `TS2366`; a discriminated union, a status flag beside a data field, an optional-everything model of a serializer, `assertNever`, an exhaustive `switch`, a `Result` type, "make illegal states unrepresentable"; a branded id, `.brand()`, a `UserId` passed where an `OrderId` belongs; `satisfies`, `as const`, `as const satisfies`, a config object that loses its literal type; `enum` against a const object and a derived union; `keyof`, `typeof`, `infer`, a mapped type, a conditional type, `Prettify`, `NoInfer`, a type predicate, `asserts`; `type` against `interface`, `declare module`, a package with wrong types; `React.FC`, `PropsWithChildren`, `ComponentProps`, `forwardRef`, `Ref`, `RefObject`, `MutableRefObject`, `useRef` with no argument, a context read through a `!`, `React.JSX`. Not here: the composition and the list key (`references/component-composition.md`), the render rules and the effect rules (`references/state-and-effects.md`), the theme object behind a `satisfies` (`references/design-tokens-and-theming.md`), the form state type and the field error map (`references/form-schema-and-field-binding.md` and `references/form-submission-and-server-errors.md`). | `references/type-modeling-and-narrowing.md` |
| A value that enters the program — `res.json()` cast to a type, a `fetch` response, `searchParams` read as data, `localStorage`, `sessionStorage`, `process.env`, a webhook body, `FormData`; "parse at the boundary", "type the response", a run-time crash on data that TypeScript called typed; Zod 4 — `z.object`, `safeParse`, `z.infer`, `z.input`, `z.output`, `z.email`, `z.discriminatedUnion`, `.brand()`, `.superRefine()`, `.check()`, `z.prettifyError`, `z.treeifyError`, `z.flattenError`, `error.issues`; the Zod 3 calls that Zod 4 removed — `error.errors`, `.format()`, `.flatten()`, `z.string().email()`, `z.nativeEnum()`, `.merge()`, `.ip()`, `.cidr()`; the generated `paths` and `components` types, `openapi-typescript`, a type hand-copied from the OpenAPI schema, two sources for one shape; a DRF field that is `required=False`, `allow_null=True`, or `blank=True`, a `nullable: true` that the generated type does not carry, `null` at run time where the type promised a value, the pagination envelope typed as an array, the DRF error body `{field: [msg]}`, `drf-standardized-errors` and its `type` and `attr` fields; `@t3-oss/env-nextjs`, an environment variable that is `undefined` in the browser. Not here: the drf-spectacular config and the codegen command (`references/openapi-schema-and-codegen.md`), the query keys and the mutation state (`references/server-state-and-query-cache.md`), the resolver and the field array in a form (`references/form-schema-and-field-binding.md`), the MSW handler and the contract test (domain 20), the server-side serializer (sibling skill `django-api-contract`). | `references/boundary-validation-and-api-types.md` |
| The shape of a component and the way that parts compose — a God component, a component file past 200 lines, more than five `useState` calls, a decomposition, "single responsibility"; three or more boolean props that change the layout, composition over configuration, `children`, a slot, a named part, a compound component, `Tabs.List`, `Tabs.Panel`, a render prop, a polymorphic component, `asChild`, `Slot`, the Base UI `render` prop, `useRender`, `data-slot`; controlled against uncontrolled, `value` and `onChange` against `defaultValue`, `useControllableState`, "A component is changing an uncontrolled input to be controlled"; `ref` as a prop against `forwardRef` in new code, `useImperativeHandle`, a parent that reaches into a child; a list `key`, an index key, "Each child in a list should have a unique key prop", a row that keeps the state of another row after a sort or a filter; a headless hook, a custom hook that returns markup; shadcn/ui on Base UI, Radix Primitives, `usehooks-ts`, `prop-types`. Not here: where the state itself lives and whether an effect is correct (`references/state-and-effects.md`), the `<Suspense>` boundary and the Action hooks (`references/suspense-and-actions.md`), `React.FC` and `PropsWithChildren` and `ComponentProps` (`references/type-modeling-and-narrowing.md`), the classes and the tokens on a part (`references/component-styles-and-variants.md` and `references/design-tokens-and-theming.md`), the ARIA role and the accessible name (`references/semantics-and-accessible-names.md`), the focus order and the keyboard contract (`references/keyboard-focus-and-live-regions.md`), the virtualiser for a long list (`references/data-table-and-server-driven-state.md`). | `references/component-composition.md` |
| Where a value lives and when an effect is correct — `useState`, `useReducer`, `useEffect`, `useLayoutEffect`, `useContext`, `createContext`, `useSyncExternalStore`, `useEffectEvent`, `useMemo`, `useCallback`, `memo`, `<Profiler>`; colocate state, lift state up, prop drilling, a global store, a derived value, a filtered list held in state, an effect that sets state, an effect that resets state when a prop changes, a `key` that remounts a component, five `useState` calls for five form fields, a context that holds a list from the backend; a subscription, `window.matchMedia`, a browser API read during a render, a socket that reconnects on an unrelated change, a stale closure, a dependency array that lies; Strict Mode, `reactStrictMode`, an effect that runs twice in development, a missing cleanup; the Rules of React, purity, a mutation during a render, `eslint-plugin-react-hooks`, `rules-of-hooks`, `exhaustive-deps`, `set-state-in-render`, `set-state-in-effect`, `reactCompiler`, `babel-plugin-react-compiler`, `"use memo"`, `"use no memo"`, `compilationMode`, a compiler bailout, `eslint-plugin-react-compiler`; CVE-2025-55182, CVE-2026-23870, and the React 19.2.6 floor; the errors "Rendered more hooks than during the previous render", "Cannot update a component while rendering a different component", "Too many re-renders", "Maximum update depth exceeded", "Missing getServerSnapshot, which is required for server-rendered content", "Hooks can only be called inside the body of a function component". Not here: `useQuery` and `staleTime` and the query cache (`references/server-state-and-query-cache.md`), the shape of the component around the state (`references/component-composition.md`), the boundary and the Action (`references/suspense-and-actions.md`), the `"use client"` directive itself (`references/server-and-client-components.md`), the type-aware lint config (`references/typescript-config-and-enforcement.md`), the INP that a re-render costs (`references/paint-and-interaction-cost.md`). | `references/state-and-effects.md` |
| The boundary that renders while a value is absent, and the Action that changes it — `<Suspense>`, a fallback, a skeleton that does not match the content, a page that shows one spinner, a section boundary, a widget boundary, layout shift when the data arrives; `ErrorBoundary`, `react-error-boundary`, `fallbackRender`, `resetErrorBoundary`, `useErrorBoundary`, a panel that must fail alone; `use(` on a promise, a promise created in a render, a fallback that never resolves, `use()` inside a `try`; a React 19 Action, `useActionState`, `useFormStatus`, `useOptimistic`, `startTransition`, `useTransition`, `formAction`, a submit button that must disable while pending, an optimistic value that must roll back; a DRF 400 rendered beside a field, a validation error that was thrown, a 5xx that must reach a boundary; `<Activity>`, `visible` and `hidden`, hidden UI that must keep its state, a tab panel that loses a scroll position; `<title>` or `<meta>` rendered from a component, `preload`, `preinit`, `prefetchDNS`, `preconnect`. Not here: `loading.tsx` and `error.tsx` as route files (`references/app-router-structure.md`), the body of the Server Action and its authorize, validate, mutate, invalidate, redirect order (`references/data-access-and-mutations.md`), the DRF error envelope shapes (`references/boundary-validation-and-api-types.md`), the mutation state in the query cache (`references/server-state-and-query-cache.md`), the resolver and the field array (`references/form-schema-and-field-binding.md`), the choreography of a transition (`references/view-transitions-and-animation-libraries.md`), the words in the message (`references/error-and-empty-state-copy.md`). | `references/suspense-and-actions.md` |
| Where a file goes and which module may import it — `src/`, `src/app`, `src/features`, `src/components/ui`, `src/components/common`, `src/lib`, `src/server`, `src/config`, feature-first, layer-first, colocation, a shared layer, dependency direction, a public API, a private folder that holds too much; a barrel file, `index.ts`, `export *`, a global barrel, a deep relative import `../../../..`, the `@/*` path alias, `optimizePackageImports`, `transpilePackages`; `eslint-plugin-boundaries`, `boundaries/element-types`, `no-restricted-paths`, `dependency-cruiser`, `.dependency-cruiser.js`, `sheriff`, `sheriff.config.ts`, a cycle, an orphan module; `src/api/generated`, `api:generate`, a generated client that someone edited, the schema artifact, the drift check; a monorepo, a workspace, `pnpm-workspace.yaml`, `turbo.json`, Turborepo, Nx; a repository restructure inside a feature task; the errors "Importing elements of type ... is not allowed", "is not allowed in elements of type", "Cannot find module" from a missing alias. Not here: the folder tokens `(group)`, `_folder`, and `@slot`, and the route files (`references/app-router-structure.md`), the `server-only` and `client-only` guards (`references/server-and-client-components.md`), `tsconfig.json` and the `paths` value (`references/typescript-config-and-enforcement.md`), the rest of the lint config array (`references/lint-format-and-scripts.md`), the drf-spectacular config and the generator (`references/openapi-schema-and-codegen.md`), the tokens that `components/ui` renders (`references/design-tokens-and-theming.md`). | `references/directory-and-module-boundaries.md` |
| The checks over the code and the commands that run them — `package.json` scripts, `dev`, `build`, `lint`, `lint:fix`, `format`, `typecheck`, `test`, `test:watch`, `test:e2e`, `analyze`, `pnpm check`, `pnpm dlx`, `--max-warnings=0`, a warning that CI accepts; `eslint.config.ts`, `eslint.config.mjs`, a flat config, `defineConfig`, `eslint-config-next`, `core-web-vitals`, `eslint-config-next/typescript`, an `ignores` entry, a lint rule that reports nothing; `next lint`, the `eslint` key in `next.config.ts`, `next-lint-to-eslint-cli`, a CI step that lints nothing; Prettier, `.prettierrc`, Biome, `biome.json`, `prettier-plugin-tailwindcss`, `tailwindStylesheet`, `eslint-plugin-tailwindcss`, `eslint-plugin-better-tailwindcss`, `.editorconfig`; `eslint-disable`, `eslint-disable-next-line`, a suppression with no reason; `next typegen` before a typecheck; `AGENTS.md`, `.cursorrules`, `.cursor/rules`, `.vscode/settings.json`, `.vscode/extensions.json`; the errors "Parsing error: ESLint was configured to run on", "next lint is deprecated and will be removed in Next.js 16", "It looks like you're trying to use tailwindcss directly as a PostCSS plugin". Not here: `projectService`, `strictTypeChecked`, and `tsc --noEmit` (`references/typescript-config-and-enforcement.md`), `eslint-plugin-react-hooks` and `reactCompiler` (`references/state-and-effects.md`), the boundaries block (`references/directory-and-module-boundaries.md`), the hook that calls a script (`references/dependencies-and-git-workflow.md`), the Tailwind theme (`references/design-tokens-and-theming.md`), the test layout and the coverage threshold (domain 20), the CI workflow (domain 22). | `references/lint-format-and-scripts.md` |
| What enters the repository and how a change leaves it — pnpm, npm, yarn, bun, `pnpm install --frozen-lockfile`, `pnpm-lock.yaml`, a lockfile conflict, a hand-edited lockfile, `packageManager`, Corepack, `corepack enable`, Volta, `.nvmrc`, `engines.node`, `.npmrc`, `overrides`; `minimumReleaseAge`, `minimumReleaseAgeExclude`, a cooldown, `onlyBuiltDependencies`, `allowBuilds`, `ignore-scripts`, a lifecycle script, `pnpm audit`, a new dependency and its justification; Renovate, `renovate.json`, Dependabot, `.github/dependabot.yml`, `automerge`, `matchUpdateTypes`, `fetch-metadata`; lefthook, `lefthook.yml`, Husky, `.husky/`, `lint-staged`, `stage_fixed`, a `prepare` script, a hook that never fires; commitlint, `commitlint.config.js`, a Conventional Commit, a commit-msg hook; `.gitignore`, a committed `.env.local`, gitleaks, `.gitleaks.toml`, a secret in the history; `CODEOWNERS`, `.gitattributes`, `linguist-generated`. Not here: the body of a script that a hook calls (`references/lint-format-and-scripts.md`), the folder that a new file goes into (`references/directory-and-module-boundaries.md`), the `NEXT_PUBLIC_` prefix (`references/app-router-structure.md`), whether a package is dangerous (`references/secret-boundary-and-supply-chain.md`), the CI workflow and the deploy (domain 22), the server-side secret storage (sibling skill `secure-code-auditor`). | `references/dependencies-and-git-workflow.md` |
| The schema that the types come from — drf-spectacular, `SPECTACULAR_SETTINGS`, `COMPONENT_SPLIT_REQUEST`, `COMPONENT_SPLIT_PATCH`, `OAS_VERSION`, `@extend_schema`, `@extend_schema_field`, `@extend_schema_serializer`, `OpenApiExample`, `OpenApiParameter`, `ENUM_NAME_OVERRIDES`, `ENUM_ADD_EXPLICIT_BLANK_NULL_CHOICE`, `schema.yml`, `openapi.json`, `/api/schema/`, `drf-yasg` and Swagger 2.0, django-ninja and OpenAPI 3.1; `openapi-typescript`, `openapi-fetch`, Orval, `@hey-api/openapi-ts`, `@kubb/*`, `swagger-typescript-api`, `openapi-generator`, `operationId`, the `paths` type, `components["schemas"]`; `api:generate`, a client generated from a live URL, a stale schema in production, `oasdiff`, `openapi-diff`, a breaking change on an enum, an enum emitted as a TypeScript `enum`; a `SerializerMethodField` typed `any`, `XRequest` against `X`, a response field that became optional, upstream issue #810; camelCase against snake_case, `djangorestframework-camel-case`, `camelize_serializer_fields`, `humps`, `ts-case-convert`. Not here: the types that a DRF construct produces and the parse over them (`references/boundary-validation-and-api-types.md`), the output folder and its `.gitignore` entry (`references/directory-and-module-boundaries.md`), the `package.json` script surface (`references/lint-format-and-scripts.md`), the client that sends a request (`references/api-client-and-request-safety.md`), the serializer, the viewset, and the deprecation of a field (sibling skill `django-api-contract`). | `references/openapi-schema-and-codegen.md` |
| The request itself, and the failure that comes back — `apiClient`, `createClient`, `endpoints.ts`, a base URL, a `fetch("/api/...")` literal in a component, `DJANGO_URL` against `NEXT_PUBLIC_API_BASE_URL`, an ECONNREFUSED from a server fetch inside Docker; a trailing slash, `APPEND_SLASH`, a 301 that drops a POST body, "you called this URL via POST"; `AbortController`, `AbortSignal.timeout`, `AbortSignal.any`, a request that never ends, a `DOMException` against a `TypeError`; a retry, an exponential backoff, `Idempotency-Key`, `Retry-After`, a 429 loop, a throttle, a duplicate row from a retried POST; `normalizeApiError`, `ApiError`, `fieldErrors`, `retryable`, `ErrorDetail`, `non_field_errors`, `detail`, `[object Object]` in a toast; a 204 with no body, "Unexpected end of JSON input", a 500 that returns HTML; the `next` and `previous` URLs, a computed page offset, `CursorPagination` with no `count`; `FormData`, a multipart boundary, an empty `request.FILES`. Not here: which module holds the call and the order inside a Server Action (`references/data-access-and-mutations.md`), the `Paginated<T>` type and the parse (`references/boundary-validation-and-api-types.md`), the schema and the generator (`references/openapi-schema-and-codegen.md`), CORS and the CSRF header (`references/cross-origin-and-bff-proxy.md`), `queryKey` and `staleTime` (`references/server-state-and-query-cache.md`), the single-flight token refresh (`references/session-and-token-lifecycle.md`), the upload progress bar (`references/file-upload-and-transport.md`). | `references/api-client-and-request-safety.md` |
| The origin boundary that the browser enforces — `django-cors-headers`, `CORS_ALLOWED_ORIGINS`, `CORS_ALLOW_CREDENTIALS`, `CORS_ALLOW_HEADERS`, a preflight, an `OPTIONS` request, "No 'Access-Control-Allow-Origin' header is present", a wildcard beside `credentials: "include"`, two `Access-Control-Allow-Origin` values, a request that never reaches the Django log; `X-CSRFToken`, the `csrftoken` cookie, `ensure_csrf_cookie`, `CSRF_HEADER_NAME`, `CSRF_TRUSTED_ORIGINS` and its scheme, "CSRF Failed: CSRF token missing", `SESSION_COOKIE_SAMESITE`, `httpOnly`, `Secure`, a cookie that the browser drops on a cross-site call; a BFF, a rewrite in front of Django, a Route Handler proxy, a fixed upstream host, SSRF, `169.254.169.254`, a `?target=` parameter, a body size cap on a proxy. Not here: `proxy.ts`, its permitted work, and CVE-2025-29927 (`references/app-router-structure.md`), the choice between a Server Component fetch, a Route Handler, and a browser call (`references/data-access-and-mutations.md`), the timeout and the error shape (`references/api-client-and-request-safety.md`), the session strategy and the cookie prefixes (`references/session-and-token-lifecycle.md`), the redirect after a 401 (`references/route-protection-and-permissions.md`), the CSP and the response headers (`references/security-headers-and-csp.md`), the DRF permission class and the server settings (sibling skill `secure-code-auditor`). | `references/cross-origin-and-bff-proxy.md` |
| The cache that holds server state — `@tanstack/react-query`, `queryOptions`, `infiniteQueryOptions`, `useQuery`, `useSuspenseQuery`, `useInfiniteQuery`, `useMutation`, `QueryClient`, `QueryClientProvider`, `staleTime`, `gcTime`, `select`, `enabled`, `initialData`, `placeholderData`, `keepPreviousData`, `isPending` against `isFetching` against `isLoading`; a key factory, a static key over a filtered list, a list that flickers, an inline `queryFn` in a component, a `QueryClient` at module scope, one user who sees another user's rows; `prefetchQuery`, `dehydrate`, `HydrationBoundary`, a refetch straight after the hydration; `invalidateQueries`, `refetchQueries`, `setQueryData`, `cancelQueries`, `onMutate`, `onSettled`, an optimistic value with no rollback, a saved record that the list does not show; `initialPageParam`, `getNextPageParam`, `getPreviousPageParam`, DRF `PageNumberPagination`, `LimitOffsetPagination`, and `CursorPagination` behind an infinite query; `refetchInterval`, a poll with no stop condition; the loading, empty, error, and ready states of a view; `cacheTime`, `keepPreviousData: true`, `useErrorBoundary`, and `isInitialLoading` from version 4; `@tanstack/eslint-plugin-query`; the errors "No QueryClient set, use QueryClientProvider to set one" and "Missing queryFn". Not here: the taxonomy, the URL, and the store (`references/client-and-url-state.md`), the page that prefetches (`references/data-access-and-mutations.md`), the typed client and `ApiError` (`references/api-client-and-request-safety.md`), the server cache and `updateTag` (`references/caching-and-revalidation.md`), the token refresh after a 401 (`references/session-and-token-lifecycle.md`), the push transport (`references/push-transport-and-connection.md`), the event that writes into this cache (`references/live-events-and-cache-merge.md`), the field error map (`references/form-submission-and-server-errors.md`), the column model (`references/data-table-and-server-driven-state.md`). | `references/server-state-and-query-cache.md` |
| Where a value lives when the backend does not own it — the state taxonomy, one owner for one value, a filter or a sort or a page or a tab or a search term held in `useState`, a copied link that loses the view, a back button that changes nothing; `useSearchParams`, `nuqs`, `useQueryState`, `useQueryStates`, `parseAsString`, `parseAsInteger`, `.withDefault()`, `throttleMs`, `createLoader`, `queryTypes` from version 1, the error "Missing Suspense boundary with useSearchParams"; Zustand, `create`, `createStore`, `useStore`, `useShallow`, `persist`, `getState()`, a store at module scope that leaks between requests, a store that holds a list from the backend, a theme that flashes on the first paint; Jotai, Valtio, Redux Toolkit, RTK Query, `swr`. Not here: every value that the backend owns (`references/server-state-and-query-cache.md`), the derived value and the effect (`references/state-and-effects.md`), the hydration error itself (`references/server-and-client-components.md`), the server `searchParams` prop (`references/app-router-structure.md`), the parse over a URL value (`references/boundary-validation-and-api-types.md`), the field value before a submit (`references/form-schema-and-field-binding.md`), the sort model of a table (`references/data-table-and-server-driven-state.md`), the locale segment (domain 19). | `references/client-and-url-state.md` |
| The credential and how long it lives — login, logout, sign-in, register, a session, `SessionAuthentication`, an access token, a refresh token, a bearer token, `Authorization`, `AUTH_HEADER_TYPES`; `httpOnly`, `Secure`, `SameSite=Lax` against `Strict` against `None`, `Max-Age`, `Domain`, `Path`, the `__Host-` and `__Secure-` prefixes, the 4 KB cookie ceiling, a `Set-Cookie` that a proxy rewrote, the console message `Cookie "myCookie" rejected because it has the "SameSite=None" attribute but is missing the "secure" attribute`; `localStorage.setItem('access', token)`, a token or a permission list in web storage; a silent refresh, a single-flight refresh, concurrent 401s, a refresh storm, `ROTATE_REFRESH_TOKENS`, `BLACKLIST_AFTER_ROTATION`, token rotation, reuse detection, a blacklist, `SIGNING_KEY`, `LEEWAY`, clock skew, an `exp` decoded on the client, a session that ends mid-visit, a session that a 500 ends; `queryClient.clear()`, `BroadcastChannel`, a cross-tab logout, a back navigation that shows the previous user; `djangorestframework-simplejwt`, `TokenObtainPairView`, `TokenRefreshView`, `/api/token/refresh/`, `dj-rest-auth`, `django-allauth` headless, `_allauth`, `X-Session-Token`, `iron-session`, `sealData`, Auth.js, NextAuth, `knox`, DRF `TokenAuthentication`, PKCE, the OAuth implicit flow, RFC 6749 and RFC 7636. Not here: the gate on a route, a Server Action, or a control (`references/route-protection-and-permissions.md`), the `X-CSRFToken` exchange and the CORS preflight (`references/cross-origin-and-bff-proxy.md`), `normalizeApiError` and the retry rule (`references/api-client-and-request-safety.md`), the cache that a logout clears (`references/server-state-and-query-cache.md`), the carrier on a socket handshake (`references/push-transport-and-connection.md`), the login form fields (`references/form-schema-and-field-binding.md`), the CSP (`references/security-headers-and-csp.md`), the DRF settings and the password hash (sibling skill `secure-code-auditor`). | `references/session-and-token-lifecycle.md` |
| Who may proceed, and what the interface shows them — `verifySession`, `getCurrentUser`, a data access layer, React `cache()` for one session read, a protected route, a page gate against a layout gate, a route group `(auth)` and `(app)`; an auth check in `proxy.ts` alone, CVE-2025-29927, CVE-2026-64642, a forged `x-middleware-subrequest` header; a Server Action that takes a `userId`, a `role`, a `tenantId`, or an `isAdmin` argument, `curl` against a Server Action with no cookie, CVE-2026-64643 and a Server Function endpoint that the client chunks disclose, "authenticate within the boundary"; `redirect('/login?next=')`, `unauthorized()`, `forbidden()`, `unauthorized.tsx`, `forbidden.tsx`, `authInterrupts`, an experimental auth interrupt, a 307 loop between `/login` and `/dashboard`, an open redirect through a `?next=` value that names another host, or `?next=//evil.example`; RBAC, a role, a group, a permission list, an object permission, a feature flag, `<Can>`, `usePermission`, a hidden button that is the whole gate, a blank page in place of a 403 explanation; a multi-tenant application, an active organization, cross-tenant cache bleed; a cached page that renders the name of a user, protected data in an RSC payload or a prefetch. Not here: the credential, the cookie attributes, and the refresh (`references/session-and-token-lifecycle.md`), the permitted work of `proxy.ts` and the route files themselves (`references/app-router-structure.md`), the rest of the data access module and the Server Action order (`references/data-access-and-mutations.md`), the `"use cache"` rules (`references/caching-and-revalidation.md`), the key factory (`references/server-state-and-query-cache.md`), the focus on a 403 message (`references/keyboard-focus-and-live-regions.md`), the Server Action as a public endpoint (`references/exposed-endpoints-and-destinations.md`), the DRF permission class and the object-level check (sibling skill `secure-code-auditor`). | `references/route-protection-and-permissions.md` |
| The connection that pushes data, and how long it lives — live, realtime, a notification feed, a chat, a progress feed, push, subscribe; `WebSocket`, `new WebSocket()`, `EventSource`, `ws://`, `wss://`, `Sec-WebSocket-Protocol`, a subprotocol that carries a token, a token in a socket URL, a single-use ticket, an `Origin` check on a handshake; `text/event-stream`, `Last-Event-ID`, the `data:` and `retry:` frame fields, `ReadableStream`, `TextDecoderStream`, NDJSON, one response read in parts, an `AbortController` over a stream; `StreamingHttpResponse`, an async iterator under ASGI, Django Channels, `ProtocolTypeRouter`, `URLRouter`, `AsyncJsonWebsocketConsumer`, a channel layer, `group_send`; a reconnect, a reconnect storm, an exponential backoff, jitter, a heartbeat, a ping and a pong, a zombie connection, a socket that dies every 60 seconds or every 100 seconds, a `readyState` that stays `OPEN` with nothing on it; close code 1000, 1001, 1006, 1008, 1011, 1012, 1013, 4001, 4003, a 1006 that logs the user out, the error "closed before the connection is established"; `document.hidden`, `visibilitychange`, a connection in a background tab; a socket in a component body, a socket for each list row, the six-connection limit of HTTP/1.1, `BroadcastChannel` for one connection over many tabs; `partysocket`, `reconnecting-websocket`, `@microsoft/fetch-event-source`, `socket.io`, WebTransport, `experimental_streamedQuery`; `proxy_http_version 1.1`, `Upgrade`, `Connection: upgrade`, `proxy_read_timeout`, `proxy_buffering off`, `X-Accel-Buffering`, a Cloudflare idle close, events that stream in development and batch in production. Not here: the frame that arrives and the cache that it writes into (`references/live-events-and-cache-merge.md`), `refetchInterval` and the poll (`references/server-state-and-query-cache.md`), where the credential lives and the refresh over it (`references/session-and-token-lifecycle.md`), the deadline and the `ApiError` of an ordinary request (`references/api-client-and-request-safety.md`), the effect rules and `useSyncExternalStore` (`references/state-and-effects.md`), the same-origin rewrite (`references/cross-origin-and-bff-proxy.md`), the politeness of a live announcement (`references/keyboard-focus-and-live-regions.md`), the upload progress bar (`references/file-upload-and-transport.md`), the Nginx file itself (domain 22), the consumer and the queue (sibling skill `django-async-jobs`). | `references/push-transport-and-connection.md` |
| The message that arrives, and what it changes — an event envelope, a `type` discriminator, an `origin` field, a `payload` from a serializer; `onmessage`, `JSON.parse(event.data)`, a malformed frame, a frame that is not JSON, an event type that the build cannot name, a feed that stops with the connection still open, a handler that threw; `z.discriminatedUnion` over an event, `safeParse` on a frame, a dropped-frame counter, `assertNever` in an event switch; `setQueryData` from a pushed event, `invalidateQueries` after a push, `removeQueries`, an invalidation that never refetches, an `isInvalidated` mark that a later write cleared, pushed rows held in `useState`; a ticker, a cursor feed, a render thrash, a coalesce into one animation frame, the React Profiler commit count; a duplicate row after your own edit, the echo of an optimistic write, `X-Origin-Id`; a resync after a reconnect, a gap in the feed, a `Last-Event-ID` replay; a renamed event type that stops the screen in silence, a channel layer that is down and a view that renders empty. Not here: the transport, the handshake, the reconnect, the heartbeat, and the close code (`references/push-transport-and-connection.md`), the key factory, the invalidation of a mutation, and the optimistic rollback (`references/server-state-and-query-cache.md`), the parse doctrine and the DRF envelopes (`references/boundary-validation-and-api-types.md`), `assertNever` itself (`references/type-modeling-and-narrowing.md`), the generated payload types (`references/openapi-schema-and-codegen.md`), the live row in a table (`references/data-table-and-server-driven-state.md`), the metric behind the counter (domain 21), the event as a versioned published surface (sibling skill `django-api-contract`). | `references/live-events-and-cache-merge.md` |
| The value layer of the interface — a design token, a semantic color, `--primary`, `--background`, `--ring`, `--chart-1`, a raw hex in a component, `text-[13px]`, a magic spacing number, a named z-index scale, `z-[9999]`; Tailwind v4, `globals.css`, `@import "tailwindcss"`, `@theme`, `@theme inline`, `@custom-variant`, `@utility`, `@variant`, `@source`, `@plugin`, `@config`, a `tailwind.config.ts` in a v4 project, a `@tailwind` directive; `oklch(`, `color-mix(`, HSL variables in a v4 setup, the OKLCH support floor; dark mode, `.dark`, the removed `darkMode: 'class'` key, a toggle that changes no pixel, `filter: invert()`, `color-scheme`, elevation in a dark theme; a flash of the wrong theme, `suppressHydrationWarning` on `<html>`, `next-themes`, `ThemeProvider`, `@wrksz/themes`, a stale theme after a navigation; `tw-animate-css` against `tailwindcss-animate`, `bg-linear-*` against `bg-gradient-*`, `bg-opacity-*`, `shadow-xs`, the 1px ring default, the `currentColor` border default, stacked variants left to right. Not here: `cn()` and the variant API (`references/component-styles-and-variants.md`), the container query and the type scale (`references/layout-and-typography.md`), where a preference is stored (`references/client-and-url-state.md`), the class order and the Prettier plugin (`references/lint-format-and-scripts.md`), the contrast ratio of a pair (`references/visual-and-motor-criteria.md`), the animation that reads a motion token (`references/motion-primitives-and-reduced-motion.md`). | `references/design-tokens-and-theming.md` |
| The classes on a part, and the API that selects them — `cn()`, `clsx`, `tailwind-merge`, `twMerge`, a template string of classes, a caller `className` that loses, `!important` against a primitive; `tailwind-variants`, `tv(`, `VariantProps`, `class-variance-authority`, `cva(`, a variant map, `defaultVariants`, a slot; `components.json`, the shadcn CLI, `data-slot`, a `div` with the appearance of a button, a hand-built control that a primitive already covers; `outline-none`, `ring-0`, `focus-visible:ring-ring`, a focus ring that somebody removed; a class name built from a run-time value, `` translate-y-${offset} ``, a `style` attribute that carries a color or a space; a CSS Module, `@layer`, `vanilla-extract`, Emotion, `styled-components`, runtime CSS-in-JS, `sonner`, `lucide-react`, Storybook, `@axe-core/react`. Not here: the tokens behind every class (`references/design-tokens-and-theming.md`), the decomposition and the `render` or `asChild` prop (`references/component-composition.md`), the layout classes (`references/layout-and-typography.md`), the accessible name (`references/semantics-and-accessible-names.md`), the authoritative focus criteria (`references/keyboard-focus-and-live-regions.md` and `references/visual-and-motor-criteria.md`), the field control and its error state (`references/form-schema-and-field-binding.md`). | `references/component-styles-and-variants.md` |
| The space that an element occupies, and the text inside it — `@container`, `@sm:`, `@md:`, `@max-*`, `container-type`, `@tailwindcss/container-queries`, a card that breaks in a sidebar, a viewport breakpoint in a reusable component; `ms-`, `me-`, `ps-`, `pe-`, `text-start`, `border-s`, a physical `ml-` or `text-left`, a layout that mirrors wrongly under `dir="rtl"`; `vh`, `dvh`, `svh`, `lvh`, `h-screen`, a section that the iOS toolbar clips, `safe-area-inset`, `viewport-fit=cover`, `scrollbar-gutter`; `aspect-ratio`, `aspect-video`, a reserved box, a skeleton whose geometry differs from the content, content that moves when an image arrives; `clamp()`, a fluid type scale, `text-wrap: balance`, `text-wrap: pretty`, the measure of a paragraph, `h-[40px]` on a text container, text that clips at 200 percent zoom; `next/font`, `next/font/google`, `next/font/local`, `subsets`, `display: 'swap'`, `preload`, `adjustFontFallback`, `variable`, `axes`, `declarations`, `size-adjust`, `ascent-override`, a variable font, a shift after a new weight; one signature idea, a screenshot critique, a design that could belong to any product. Not here: the tokens and the theme (`references/design-tokens-and-theming.md`), `cn()` and the variant API (`references/component-styles-and-variants.md`), the Suspense boundary itself (`references/suspense-and-actions.md`), `next/image` and the bytes behind it (`references/image-and-video-delivery.md`), the reflow and the zoom criteria (`references/visual-and-motor-criteria.md`), the LCP element and the layout shift budget (`references/paint-and-interaction-cost.md`), the locale route and the non-Latin subset (domain 19). | `references/layout-and-typography.md` |
| The element, the role, and the name that a screen reader reads — accessibility, a11y, WCAG, ARIA, an accessible name, a screen reader, NVDA, VoiceOver, TalkBack; a `div` with an `onClick`, `role="button"` beside a `keydown` handler, an `<a>` with no `href`, a native element, `<label>`, `<fieldset>`, `<legend>`, `<details>`, `<summary>`, `<dialog>`, `<output>`, `<progress>`, `<meter>`, `<caption>`, `scope`, `<figcaption>`; the five rules of ARIA, `aria-label`, `aria-labelledby`, `aria-describedby`, `aria-details`, `title` as the only name, an icon-only button with no name; `aria-expanded`, `aria-controls`, `aria-current`, `aria-selected`, `aria-checked`, `aria-pressed`, `aria-invalid`, `aria-required`, `aria-busy`, `disabled` against `aria-disabled`; `aria-hidden` on a focusable element, `inert`, `role="presentation"`, `role="none"`, `sr-only`; a landmark, `<main>`, `<nav>`, `<header>`, `<footer>`, `<aside>`, a skipped heading level, more than one `<h1>`; `alt=""`, an alt text that repeats a file name, a decorative icon, `role="img"` on an SVG; `lang` on `<html>`, `lang` on a passage, `dir`; a placeholder used as a label, `htmlFor`, `autoComplete`, `one-time-code`, a field that blocks paste, criterion 3.3.8, criterion 3.3.7, criterion 1.3.5, a session that ends with no warning. Not here: the tab order, the focus, and the live region (`references/keyboard-focus-and-live-regions.md`), the contrast ratio and the target size (`references/visual-and-motor-criteria.md`), the axe rule and the lint plugin (`references/wcag-conformance-and-verification.md`), the shape of the component and its parts (`references/component-composition.md`), the classes on a part (`references/component-styles-and-variants.md`), the resolver and the field array (`references/form-schema-and-field-binding.md`), the text alternative of a chart (`references/charts-and-visual-encoding.md`), the words in a label (`references/interface-copy-and-voice.md`), the locale route (domain 19). | `references/semantics-and-accessible-names.md` |
| The path that a keyboard takes, and the change that a screen reader hears — a tab order, `tabIndex`, a positive `tabindex`, `tabIndex={-1}`, a keyboard trap, a focus ring that a sticky header hides, criterion 2.4.11, `scroll-padding-top`; an APG pattern, a menu, a menubar, a listbox, a combobox, a tab list, an accordion, a dialog, an alert dialog, a tree, a grid, a slider, a toolbar, a disclosure, a carousel, a roving `tabindex`, `aria-activedescendant`; a modal, a focus trap, `showModal()`, `inert` on the background, focus lost to `<body>` after a close, an initial focus target; a route change that announces nothing, `usePathname`, a route announcer, a skip link, `#main-content`, a bypass block; a single-character shortcut, criterion 2.1.4, a shortcut help dialog; a tooltip that only a hover produces, criterion 1.4.13, dismissible, hoverable, persistent; `aria-live`, `role="status"`, `role="alert"`, `role="log"`, `aria-atomic`, `aria-relevant`, criterion 4.1.3, a live region that mounts with its message, a toast that announces nothing, a progress value that announces on every update, one announcer for the application; an error summary, focus on the first error after a submit, a toast as the only report of a validation error. Not here: the role, the name, and `inert` itself (`references/semantics-and-accessible-names.md`), the contrast of the indicator and the target size (`references/visual-and-motor-criteria.md`), the axe assertion (`references/wcag-conformance-and-verification.md`), the ring token and `outline-none` (`references/component-styles-and-variants.md`), the effect rules and the cleanup (`references/state-and-effects.md`), the Action state that holds the field errors (`references/suspense-and-actions.md`), the pushed event itself (`references/live-events-and-cache-merge.md`), the resolver and the multi-step flow (`references/form-schema-and-field-binding.md` and `references/multi-step-forms-and-unsaved-work.md`), the keyboard path through a data grid (`references/data-table-and-server-driven-state.md`), the view transition (`references/view-transitions-and-animation-libraries.md`). | `references/keyboard-focus-and-live-regions.md` |
| The measurable properties of a surface, and the input that operates it — a contrast ratio, 4.5:1, 3:1, a color pair, an opacity modifier on a text color, contrast in a dark theme, contrast on a gradient, criterion 1.4.1, a status that is a colored dot, a link with no underline in body text; a target size, 24 by 24 CSS pixels, 44 by 44, criterion 2.5.8, an icon button with no box; reflow, 320 CSS pixels, 400 percent zoom, criterion 1.4.10, a two-directional scroll, text spacing, criterion 1.4.12, a fixed height on a text container, `userScalable`, `maximumScale`, `user-scalable=no`, the `viewport` export, orientation; `prefers-reduced-motion`, `prefers-reduced-transparency`, `prefers-contrast`, `forced-colors`, `motion-reduce:`, `contrast-more:`, `forced-colors:`, a panel that disappears in the high-contrast mode; a drag with no second path, `draggable`, criterion 2.5.7, a pointer gesture, a motion actuation, a slider with no arrow keys; consistent navigation, consistent help, criterion 3.2.6, a context change that the user did not ask for, an action with no undo. Not here: the role, the name, and the alt text (`references/semantics-and-accessible-names.md`), the focus indicator itself and the announcement (`references/keyboard-focus-and-live-regions.md`), the axe run that measures a ratio (`references/wcag-conformance-and-verification.md`), the token pairs and the dark theme (`references/design-tokens-and-theming.md`), the container query and the fluid type scale (`references/layout-and-typography.md`), the chart series (`references/charts-and-visual-encoding.md`), the video player and its captions (`references/image-and-video-delivery.md`), the animation itself (`references/motion-primitives-and-reduced-motion.md`), the LCP element and the layout shift budget (`references/paint-and-interaction-cost.md`). | `references/visual-and-motor-criteria.md` |
| The conformance target, and the evidence for it — WCAG 2.2, Level AA, POUR, a success criterion, WCAG 2.1, WCAG 3.0, APCA, a conformance claim, an accessibility audit; the ADA, Section 508, the European Accessibility Act, EN 301 549, AODA, an accessibility statement, a VPAT, an Accessibility Conformance Report; axe, `axe-core`, `eslint-plugin-jsx-a11y`, `vitest-axe`, `jest-axe`, `toHaveNoViolations`, `@axe-core/playwright`, `AxeBuilder`, `withTags`, `wcag22aa`, Lighthouse, Pa11y, IBM Equal Access, the Storybook accessibility addon, `@axe-core/react` on React 19; the 30 to 40 percent that automation finds, the manual pass, an unplugged mouse, a screen-reader pair, Narrator, the forced-colors mode; an accessibility gate in CI, a baseline of known violations, a keyboard walkthrough in a pull request, accessibility criteria in a definition of done. Not here: the role, the name, and the field label (`references/semantics-and-accessible-names.md`), the tab path and the announcement (`references/keyboard-focus-and-live-regions.md`), the ratio and the target size (`references/visual-and-motor-criteria.md`), the flat config array and the `package.json` scripts (`references/lint-format-and-scripts.md`), the test runner and the Playwright project config (domain 20), the workflow file (domain 22). | `references/wcag-conformance-and-verification.md` |
| The schema that one form stands on, and the control that binds to it — a form, `<form>`, `useForm`, React Hook Form, `register`, `Controller`, `useController`, `useFieldArray`, `useWatch`, `getValues`, `watch()`, `formState`, `defaultValues`, `trigger`, `shouldUnregister`; `zodResolver`, `standardSchemaResolver`, `@hookform/resolvers`, a resolver that disagrees with the generic, `useForm<z.infer<...>>` over a schema that carries a `.default()`, `z.input`, `z.output`, `.refine()` with no `path`, `z.discriminatedUnion` over a branch of a form, `valueAsNumber`, `z.coerce.number()`; `mode: "onChange"`, `onTouched`, `onBlur`, `reValidateMode`, a message at the first keystroke, a re-render on each keystroke, "A component is changing an uncontrolled input to be controlled" on a field, an index key on a field array row; an OTP or one-time-code field, `input-otp`, a phone field, `libphonenumber-js`, `react-phone-number-input`, E.164, a date picker, `react-day-picker`, `@internationalized/date`, a password strength meter, `@zxcvbn-ts/core`, `zxcvbn`, an input mask, `react-imask`, `cleave.js`; `next/form` and a search or filter form, `@tanstack/react-form`, `@conform-to/react`, `valibot`, `vest`, `formik`, the React Hook Form 8 pre-release and its `keyName` removal. Not here: the submit and the server rejection (`references/form-submission-and-server-errors.md`), the step and the exit guard (`references/multi-step-forms-and-unsaved-work.md`), the Zod 4 API surface and the DRF envelopes (`references/boundary-validation-and-api-types.md`), `useActionState` and `useFormStatus` (`references/suspense-and-actions.md`), the `<label>`, `aria-describedby`, and `aria-invalid` (`references/semantics-and-accessible-names.md`), the classes on a field control (`references/component-styles-and-variants.md`), one `useState` for each field (`references/state-and-effects.md`), the file picker and the upload (`references/file-upload-and-transport.md`), the words in a label or a message (`references/interface-copy-and-voice.md`). | `references/form-schema-and-field-binding.md` |
| The submit, and the failure that comes back to the form — `handleSubmit`, `setError`, `shouldFocus`, `shouldFocusError`, `formState.errors`, `isSubmitting`, `applyServerErrors`, a double submit, two POST requests from one click, a submit button that is inert, `disabled={!isValid}`, `isValid` that validates every field; a DRF 400 field dictionary, `non_field_errors`, a nested serializer error such as `address.city`, a list serializer error such as `items.1.sku`, `attr`, `drf-standardized-errors`, an error `code` against a translated message; a toast as the only report of a validation failure, a server error that reaches no field, a message that names no field, an error region with `role="alert"`; 409, 422, 429, `Retry-After`, `Idempotency-Key` on a retried submit, a 5xx that clears the form, `reset()` in a `catch` or a `finally`, values lost on a failed submit, a reset with the values the client sent; an Action state that returns no values, `z.flattenError`, `formErrors` and `fieldErrors`, `safeParse` over `FormData`; a password, a token, or a personal value in a console log after a failed submit. Not here: the schema, the resolver, and the bind of a control (`references/form-schema-and-field-binding.md`), the step and the exit guard (`references/multi-step-forms-and-unsaved-work.md`), `normalizeApiError`, the retry rule, and the deadline (`references/api-client-and-request-safety.md`), the envelope shapes themselves (`references/boundary-validation-and-api-types.md`), the Action hooks and the expected-error rule (`references/suspense-and-actions.md`), the order inside a Server Action (`references/data-access-and-mutations.md`), the key that a success invalidates (`references/server-state-and-query-cache.md`), the error summary, the focus, and the live region (`references/keyboard-focus-and-live-regions.md`), the words in the message (`references/error-and-empty-state-copy.md`), the MSW handler and the test (domain 20). | `references/form-submission-and-server-errors.md` |
| A form over more than one screen, and the work that a navigation destroys — a wizard, a stepper, a step index, a step in `useState`, a reload that returns the user to the first step, a back button in the middle of a flow, a schema for each step, `trigger` over the fields of one step, a step change that announces nothing; unsaved changes, a dirty form, `isDirty`, `beforeunload`, `onNavigate` on `<Link>`, `event.preventDefault()` on a navigation, the Navigation API `navigate` event and `intercept()`, a traverse that no API cancels, `router.push` that nothing intercepts, `next-navigation-guard`, a draft, an autosave, a guard that stays after a successful submit; a value that the flow requests twice, criterion 3.3.7, a password or a payment value in a query string or in a stored draft. Not here: the schema, the resolver, and the `shouldUnregister` default (`references/form-schema-and-field-binding.md`), the submit and the server error map (`references/form-submission-and-server-errors.md`), the `nuqs` parsers and the store (`references/client-and-url-state.md`), the `<Link>` component and the route files (`references/app-router-structure.md`), the context and the effect cleanup (`references/state-and-effects.md`), the focus that a step change moves (`references/keyboard-focus-and-live-regions.md`), the criterion that forbids a second request for one answer (`references/semantics-and-accessible-names.md`), the words in a warning (`references/interface-copy-and-voice.md`), the test that reloads mid-flow (domain 20). | `references/multi-step-forms-and-unsaved-work.md` |
| The dense row surface, and the server that drives it — a data table, a data grid, a dashboard grid, a row, a column, a cell, a bulk action, an inline edit; `useReactTable`, `createColumnHelper`, `ColumnDef`, `accessorKey`, `accessorFn`, `getCoreRowModel`, `getSortedRowModel`, `getFacetedRowModel`, `flexRender`, `tableFeatures()`, `table.FlexRender`, TanStack Table v8 against v9, `react-table` v7, column pinning, column resizing, column visibility; `manualPagination`, `manualSorting`, `manualFiltering`, `rowCount`, a `pageCount` of `-1`, a dead "last page" control, `getRowId`, an index as a row id, `enableRowSelection`, a selection that moves after a sort, "select all matching" against "select all on this page"; `?page=`, `?page_size=`, `?limit=`, `?offset=`, `?cursor=`, `?ordering=`, `?search=`, `PageNumberPagination`, `LimitOffsetPagination`, `CursorPagination` with no `count`, `OrderingFilter`, `SearchFilter` and its `^` and `=` and `@` and `$` prefixes, `django-filter`, `lookup_expr`; a filter or a sort that a link does not reproduce, `keepPreviousData`, a search that fires on each keystroke, a row height that jumps between two pages; `useVirtualizer`, `estimateSize`, `measureElement`, `overscan`, `content-visibility`, `contain-intrinsic-size`, find-in-page that misses a row, `display: grid` on a `<table>` that drops the native roles; `aria-sort`, `aria-rowcount`, `aria-rowindex`, `role="grid"`, `role="treegrid"`, `<caption>`, `scope`, a sticky header over the focused row, a table that scrolls sideways on a phone; a 409 on an inline edit. Not here: the chart itself (`references/charts-and-visual-encoding.md`), the `Intl` formatter in a cell and the file that an export produces (`references/cell-formatting-and-export.md`), the query key, the cache times, and the infinite query (`references/server-state-and-query-cache.md`), the `nuqs` parsers and the write rate (`references/client-and-url-state.md`), the APG grid keys and the polite announcement (`references/keyboard-focus-and-live-regions.md`), the accessible name of a control (`references/semantics-and-accessible-names.md`), the INP of a sort (`references/paint-and-interaction-cost.md`), the row payload budget (`references/performance-budgets-and-measurement.md`), the serializer and the filter field on the server (sibling skill `django-api-contract`). | `references/data-table-and-server-driven-state.md` |
| The chart, and what it encodes — a chart, a graph, a plot, a bar, a line, an area, a pie, a donut, a treemap, a scatter, a series, a legend, an axis, a dashboard tile; Recharts, `ResponsiveContainer`, `@visx`, `@visx/xychart`, ECharts, `echarts-for-react`, `AriaComponent`, `aria.decal`, Chart.js, `react-chartjs-2`, `@nivo`, `@observablehq/plot`; a chart library imported into a Server Component, a `ResponsiveContainer` that measures 0 by 0 on the server, a canvas library with no server DOM, `next/dynamic` with `ssr: false`, a skeleton with no aspect ratio, a chart that shifts the layout when it arrives; a truncated baseline, a pie of more than five slices, a status carried by hue alone, a legend that maps a name to a color alone, criterion 1.4.1, criterion 1.4.11, a grayscale screenshot; `<figure>`, `<figcaption>`, `role="img"` on a chart, a chart with no text alternative, a "download data" control, criterion 1.1.1; `--chart-1`, a hex color inside a chart component; more points than pixels, LTTB, a downsample. Not here: the table under the chart (`references/data-table-and-server-driven-state.md`), the `Intl` label on an axis and the download of the values (`references/cell-formatting-and-export.md`), the tokens themselves and the dark theme (`references/design-tokens-and-theming.md`), the contrast ratio and the criteria (`references/visual-and-motor-criteria.md`), the boundary and the hydration failure (`references/server-and-client-components.md`), the shape of a fallback (`references/suspense-and-actions.md`), the bundle bytes of a chart library (`references/client-bundle-and-third-party-scripts.md`), the aggregate query behind a series (sibling skill `django-performance-optimizer`). | `references/charts-and-visual-encoding.md` |
| The value a user reads, and the file a user takes away — `Intl.NumberFormat`, `Intl.DateTimeFormat`, `notation: "compact"`, `signDisplay`, `roundingMode`, `roundingIncrement`, `trailingZeroDisplay`, `useGrouping`, `timeZone`, `toLocaleString()` with no locale, a hydration mismatch on a date cell, a formatter built inside a render, a stored formatted string, `tabular-nums`; an export, "export CSV", a report download, an export that holds the current page, an export that ignores the filter; `papaparse`, a CSV, the formula prefixes `=` and `+` and `-` and `@` with a tab and a carriage return, CSV injection, a leading apostrophe, a UTF-8 byte order mark, `\uFEFF`, a `Blob` of `text/csv`; SheetJS, `xlsx`, the npm package frozen at 0.18.5, CVE-2023-30533, CVE-2024-22363, the `cdn.sheetjs.com` tarball at 0.20.3; a backend export job, a task id, a poll, a download link, an expiry. Not here: the table that holds the cells and the filter that an export repeats (`references/data-table-and-server-driven-state.md`), the chart (`references/charts-and-visual-encoding.md`), the request, its timeout, and its abort signal (`references/api-client-and-request-safety.md`), the poll and its stop condition (`references/server-state-and-query-cache.md`), the announcement when a file is ready (`references/keyboard-focus-and-live-regions.md`), the object URL (`references/image-and-video-delivery.md`), the `Content-Disposition` header (`references/served-content-and-downloads.md`), the locale that the application chooses (domain 19), the worker behind a large export (sibling skill `django-async-jobs`), the escape inside a file that the server builds (sibling skill `secure-code-auditor`). | `references/cell-formatting-and-export.md` |
| The file that a user picks, and the path that it takes to storage — `<input type="file">`, `accept`, `capture`, `multiple`, a file picker, a drop zone, `dragenter`, `dragover`, `dragleave`, `drop`, `dataTransfer`, `react-dropzone`, a drag with no keyboard path; `File`, `Blob`, `FormData`, `file.type`, an extension check, a disguised file, `image.png` that holds HTML, a magic byte, a file signature, `magic-bytes.js`, `filetypemime`; EXIF, GPS in a photograph, `browser-image-compression`, `preserveExif`, a canvas re-encode, a sideways photograph, HEIC, HEIF, `heic2any`, libheif, `createImageBitmap`, `canvas.toBlob`, `react-image-crop`, `cropperjs`, an avatar upload, a 12-megapixel file for a 48-pixel box; `serverActions.bodySizeLimit`, a 413, "Body exceeded 1 MB limit", a file that passes through Node twice, CVE-2026-64646; a presigned URL, a presigned POST, `createPresignedPost`, `content-length-range`, `starts-with`, `$Content-Type`, `EntityTooLarge`, an opaque 403 from storage, a confirm call, an orphaned object, a pending scan, a `processing` state; a resumable upload, a chunked upload, `tus`, `tus-js-client`, Uppy, `@uppy/tus`, `@uppy/aws-s3`, an S3 multipart upload, a list-parts call, Golden Retriever, a resume that restarts at 0 percent; `XMLHttpRequest`, `upload.onprogress`, `lengthComputable`, upload progress, a bar with no percentage, a cancel control, a retry, `duplex: "half"`, a request stream. Not here: the request that carries the file and the multipart header (`references/api-client-and-request-safety.md`), the CORS preflight and the `X-CSRFToken` header (`references/cross-origin-and-bff-proxy.md`), the choice between a Server Action and a Route Handler (`references/data-access-and-mutations.md`), the schema over a file field (`references/form-schema-and-field-binding.md`), the submit and the server error map (`references/form-submission-and-server-errors.md`), the preview and its object URL (`references/image-and-video-delivery.md`), the origin that serves the stored file back (`references/served-content-and-downloads.md`), criterion 2.5.7 and the second path beside a drag (`references/visual-and-motor-criteria.md`), the Content Security Policy (`references/security-headers-and-csp.md`), the reverse-proxy body limit and the bucket (domain 22), the server-side sniff, the virus scan, and the name of a stored file (sibling skill `secure-code-auditor`), the shape of the presigned response (sibling skill `django-api-contract`), the scan worker and the transcode worker (sibling skill `django-async-jobs`). | `references/file-upload-and-transport.md` |
| The bytes of a picture and of a moving picture, from the source to the screen — `next/image`, `<Image>`, a raw `<img>`, `width` and `height`, `fill`, `sizes`, a phone that downloads a desktop variant, a picture with no reserved box, `placeholder="blur"`, `blurDataURL`, `quality`, `unoptimized`, AVIF, WebP; `preload`, the deprecated `priority` prop, `fetchPriority="high"`, `loading="lazy"` on the hero, an element that the first document never names; `remotePatterns`, `hostname: '**'`, `images.domains`, `localPatterns`, `loaderFile`, `sharp`, `minimumCacheTTL`, `imageSizes`, `qualities`, `maximumRedirects`, `dangerouslyAllowSVG`, `imgOptSkipMetadata`, an optimizer that runs out of memory, `next/legacy/image`, CVE-2025-59471, CVE-2026-64644; `URL.createObjectURL`, `URL.revokeObjectURL`, a blob URL, a preview list whose memory climbs; `<video>`, `<audio>`, `poster`, `playsInline`, `controls`, `muted`, autoplay that never starts, `<track kind="captions">`, WebVTT, a transcript, HLS, DASH, `hls.js`, `shaka-player`, `mux-player`, `video.js`, Media Source Extensions, CEA-708; a facade, `lite-youtube-embed`, `youtube-nocookie.com`, an embed that loads a player before anybody presses play. Not here: the CSS that reserves the ratio and the skeleton geometry (`references/layout-and-typography.md`), the `next.config.ts` keys and the Next 16 defaults (`references/app-router-structure.md`), the `"use client"` directive over a player island (`references/server-and-client-components.md`), the cleanup that an effect returns (`references/state-and-effects.md`), the `preload` function of `react-dom` (`references/suspense-and-actions.md`), the alternative text and the name of a play control (`references/semantics-and-accessible-names.md`), the keyboard contract of a custom player (`references/keyboard-focus-and-live-regions.md`), the contrast and the target size of a control over a picture (`references/visual-and-motor-criteria.md`), the picked file before the preview (`references/file-upload-and-transport.md`), the header over a served file (`references/served-content-and-downloads.md`), the largest paint (`references/paint-and-interaction-cost.md`), the bundle budget (`references/client-bundle-and-third-party-scripts.md`), the Open Graph image (domain 18), the transcode job (sibling skill `django-async-jobs`). | `references/image-and-video-delivery.md` |
| The bytes that leave the application for a user — user content on the origin of the application, a stored cross-site script through an avatar or a logo, a user SVG, an HTML file renamed to an image, `Content-Disposition: attachment`, `X-Content-Type-Options: nosniff`, a MIME sniff, a separate media subdomain, a raster conversion of an SVG; a download, a download control that does nothing, the `download` attribute ignored on another origin, a file that opens in a tab instead, a short-lived signed URL, an expired link with no state, a streaming Route Handler, `Response(upstream.body)`, `arrayBuffer()` on a large file, a Node process that runs out of memory on a download; `showSaveFilePicker`, the File System Access API, `createWritable`, a picker that Safari and Firefox do not carry, a `blob:` URL, a `data:` URL, an anchor built in code; a CSV, a PDF, or a ZIP that the browser builds, a frozen tab during an export, a Web Worker for a large build. Not here: what goes inside an export file, the formula escape, and the byte order mark (`references/cell-formatting-and-export.md`), the object URL and its release (`references/image-and-video-delivery.md`), the upload that stored the file (`references/file-upload-and-transport.md`), the session gate inside a Route Handler (`references/route-protection-and-permissions.md`), the cookie attributes and the domain of a cookie (`references/session-and-token-lifecycle.md`), the Route Handler as a shape (`references/data-access-and-mutations.md`), the request, its timeout, and its abort signal (`references/api-client-and-request-safety.md`), the announcement when a file is ready (`references/keyboard-focus-and-live-regions.md`), the Content Security Policy and the response headers of the application (`references/security-headers-and-csp.md`), the cost of a build on the main thread (`references/paint-and-interaction-cost.md`), the reverse proxy and the storage bucket (domain 22), the headers that Django sends and the check over a stored file (sibling skill `secure-code-auditor`). | `references/served-content-and-downloads.md` |
| The animation that the code starts, and the preference that limits it — an animation, a transition, a fade, a slide, a spring, an easing function, a stagger, a choreography, a micro-interaction, a shimmer, a ripple, a hover that moves, decoration with no purpose; a hand-typed `200ms` or a `cubic-bezier(` curve in a component where a `--duration-*` or an `--ease-*` token exists; `transition`, `transition-behavior: allow-discrete`, `@starting-style`, `@keyframes`, `animation`, `animation-fill-mode`, `will-change`, `@property`, `interpolate-size`, `calc-size()`, `grid-template-rows: 0fr`, an accordion that snaps open, a `height: auto` that does not animate, a dialog that vanishes with no exit, an exit that stopped after a `transition: all` edit; jank, a dropped frame, layout thrash, `transition: height`, `top` and `left` animated per frame, a 4× CPU throttle, a Performance recording, paint flashing; `prefers-reduced-motion`, `motion-reduce:`, a global `animation: none`, a flash of motion before the JavaScript runs; criterion 2.2.2, criterion 2.3.1, criterion 2.3.3, an autoplaying carousel, a marquee, a background video with no pause, content that flashes more than three times a second; a spinner for a 40 ms response, `transitionend`, a focus that lands on a moving element. Not here: where the `--duration-*`, `--ease-*`, and `--distance-*` tokens live, `@theme { --animate-* }`, and `tw-animate-css` (`references/design-tokens-and-theming.md`), the global preference block and the conformance verdict (`references/visual-and-motor-criteria.md`), which element takes the focus and the live region that reports a change (`references/keyboard-focus-and-live-regions.md`), the fallback and its shape (`references/suspense-and-actions.md`), Motion and the view transition (`references/view-transitions-and-animation-libraries.md`), the drag and the scroll (`references/gesture-and-scroll-interaction.md`), the INP threshold (`references/performance-budgets-and-measurement.md`), the long task and the yield (`references/paint-and-interaction-cost.md`). | `references/motion-primitives-and-reduced-motion.md` |
| The continuity between two states, and the library that buys it — a shared element, a hero animation, a list that morphs into a detail view, an exit animation that never plays, an element that vanishes on unmount, a layout animation, an interruptible morph; `document.startViewTransition`, `view-transition-name`, `viewTransitionName`, `::view-transition-old`, `::view-transition-new`, `@view-transition`, `navigation: auto`, `<ViewTransition>`, `unstable_ViewTransition`, `addTransitionType`, `experimental.viewTransition`, a transition that the browser skips, two elements that share one name; `motion`, `motion/react`, `motion/react-client`, `framer-motion`, `AnimatePresence`, `layout`, `layoutId`, `useAnimate`, `useInView`, `whileInView`, `MotionConfig`, `useReducedMotion`, `LazyMotion`, `domAnimation`, `domMax`, the `m` component, 34 kB for a fade; `gsap`, `ScrollTrigger`, `SplitText`, a Club GreenSock licence comment, `@formkit/auto-animate`, `useAutoAnimate`, `@react-spring/web`, `lottie-react`, `@dotlottie/react-player`, `@rive-app/react-canvas`, `react-transition-group`, `next-view-transitions`; a Motion import in a Server Component file, a non-animatable value warning, text that stretches during a layout animation, "layout animations are blocked during horizontal window resize". Not here: the CSS transition, the tokens, and the reduced variant (`references/motion-primitives-and-reduced-motion.md`), the drag and the scroll timeline (`references/gesture-and-scroll-interaction.md`), the router, the prefetch, and `next.config.ts` (`references/app-router-structure.md`), the `"use client"` rule itself (`references/server-and-client-components.md`), `<Activity>` and the hook rules (`references/suspense-and-actions.md`), the virtualiser under a long list (`references/data-table-and-server-driven-state.md`), the bundle budget over a library (`references/client-bundle-and-third-party-scripts.md`). | `references/view-transitions-and-animation-libraries.md` |
| The motion that a finger, a pointer, or a scroll drives — a drag, a drag handle, a sortable list, a reorder, a swipe, a pull-to-refresh, a long-press, a bottom sheet, a drawer, a scroll reveal, a parallax, a scroll progress bar, a cursor effect, a magnetic button, scroll-jacking, a page that fights the scrollbar, motion sickness; `dnd-kit`, `@dnd-kit/core`, `@dnd-kit/sortable`, `DndContext`, `PointerSensor`, `KeyboardSensor`, `useSensors`, `activationConstraint`, `draggable`, `dataTransfer`, the native HTML drag and drop API, `vaul`; `animation-timeline`, `scroll()`, `view()`, `ScrollTimeline`, `@supports (animation-timeline: scroll())`, `animation-fill-mode: both`, an element that snaps back at the top of the page, a scroll animation that does nothing in one browser; `scroll-snap-type`, `scroll-behavior`, `overscroll-behavior`, `touch-action`, an `addEventListener("scroll")` that writes a style, an `onWheel` that moves the page; `useScroll`, `useMotionValue`, `useSpring`, `prefers-reduced-motion: no-preference`, `@media (hover: hover)`, `pointer: coarse`. Not here: criterion 2.5.7 and the second path beside a drag (`references/visual-and-motor-criteria.md`), the focus inside a sheet and the announcement of a reorder (`references/keyboard-focus-and-live-regions.md`), the CSS transition and the tokens (`references/motion-primitives-and-reduced-motion.md`), the Motion entry points and the bundle cost (`references/view-transitions-and-animation-libraries.md`), the row model and the virtualiser (`references/data-table-and-server-driven-state.md`), the mutation that saves a new order (`references/data-access-and-mutations.md`), the long task and its budget (`references/paint-and-interaction-cost.md`). | `references/gesture-and-scroll-interaction.md` |
| The words that a person reads on a control — microcopy, a button label, a field label, a heading, a nav label, an onboarding step, a voice and tone guide, a glossary, a terminology decision; `Submit`, `OK`, `Yes`, `No`, `Click here`, `Read more`, `Learn more`, a label that names a mechanism rather than an outcome, one act with two verbs across two screens; a destructive act, a confirmation dialog, `Are you sure`, `role="alertdialog"`, a confirm control that repeats no verb, an act with no undo; the accessible name that must hold the visible label, criterion 2.5.3, Label in Name, an `aria-label` that paraphrases the text on the control, a speech reader, Voice Control, Dragon; a placeholder that carries an instruction, a hint before the submit, `title` on a control, a rule that only the error teaches, criterion 3.3.3; sentence case, title case, a trailing period on a label, a label that overflows its control in translation; criterion 2.4.4, link text out of context; criterion 1.3.3, criterion 1.4.1, "the green button on the right"; a dark pattern, confirmshaming, a pre-checked consent, a false deadline, a disguised advertisement, a hidden cost. Not here: the name computation and the label wiring (`references/semantics-and-accessible-names.md`), the three conditions of content on hover and the announcer (`references/keyboard-focus-and-live-regions.md`), the criterion verdict and the color pair (`references/visual-and-motor-criteria.md`), the message after a failure and the empty view (`references/error-and-empty-state-copy.md`), the key and the count (`references/message-catalog-and-plurals.md`), the primitive behind a dialog (`references/component-composition.md`), the box that holds a label (`references/layout-and-typography.md`), the lawful basis of a consent string (domain 23). | `references/interface-copy-and-voice.md` |
| What a person reads when a request fails, and when a view holds nothing — an error message, a mapped message, a recovery control, a message that names no next step, an offline message; `error.tsx`, `global-error.tsx`, `not-found.tsx`, `notFound()`, `error.message` rendered in production, `error.digest`, a `<details>` disclosure over a digest, a boundary with no `reset()` control; a provider that `global-error.tsx` cannot reach, a root boundary that throws inside itself, literal last-resort copy, `<html lang>` inside a root boundary; a map from an `ApiError` `code` onto a key, a renamed code that falls to the generic message, `attr`, `detail`, a raw serializer string in a view; an empty state, "No data", "No results", a list that never held a row, a filter that excluded every row, a request that failed, a degraded state under a dropped connection; a 401, a 403 that renders a blank page, a 404 that names a record the account may not read, a 409, a 429 and `Retry-After`; a toast, a snackbar, a notification as the only report of a failure, a toast that repeats an inline message. Not here: `ApiError` and the normalizer (`references/api-client-and-request-safety.md`), the envelope shapes (`references/boundary-validation-and-api-types.md`), the map onto a form control (`references/form-submission-and-server-errors.md`), the four states of a data view (`references/server-state-and-query-cache.md`), the route files themselves (`references/app-router-structure.md`), the gate that produces the refusal (`references/route-protection-and-permissions.md`), the announcer and the politeness level (`references/keyboard-focus-and-live-regions.md`), the words on the control inside a message (`references/interface-copy-and-voice.md`), the rule against a leak of exception text (`references/exposed-endpoints-and-destinations.md`), the lookup from a digest into a log (domain 21). | `references/error-and-empty-state-copy.md` |
| The string as data — a message key, a catalog key, `t('...')`, `useTranslations`, `getTranslations`, a hardcoded string in the markup, a sentence built from two keys and a template string; ICU MessageFormat, `plural`, `select`, `selectordinal`, `=0` against the `zero` category, the `#` token, the CLDR categories `zero`, `one`, `two`, `few`, `many`, and `other`, `count + " items"`, "1 items"; `Intl.ListFormat`, a `join(", ")` with a hand-written "and", `conjunction`, `disjunction`, `unit`; bidi isolation, U+2068, U+2069, FSI, PDI, UAX #9, a right-to-left name inside a left-to-right sentence; a pseudo-locale, a 40 percent expansion, a control that clips its translated label; `next-intl`, `@formatjs/cli` extract, `eslint-plugin-formatjs`, `eslint-plugin-i18next`, `pseudo-localization`, `vale`, `cspell`, `textlint`, `i18next`, `@lingui/core`, `write-good`, `alex`, `Intl.MessageFormat` as a proposal. Not here: the words themselves (`references/interface-copy-and-voice.md` and `references/error-and-empty-state-copy.md`), `Intl.NumberFormat` and `Intl.DateTimeFormat` (`references/cell-formatting-and-export.md`), the box that a label overflows (`references/layout-and-typography.md`), the `lang` and the `dir` attributes (`references/semantics-and-accessible-names.md`), the lint config array (`references/lint-format-and-scripts.md`), the file that holds the catalog and the locale route (domain 19), the snapshot test over a locale (domain 20). | `references/message-catalog-and-plurals.md` |
| Speed as a measured property, and the gate that holds it — performance, slow, a complaint about speed, Core Web Vitals, Web Vitals, `LCP`, `INP`, `CLS`, `TTFB`, `FCP`, `TBT`, First Input Delay, p75, CrUX, RUM, a field percentile, a Lighthouse score; `next build`, the route table, First Load JS, a performance budget, a byte budget, a budget that warns rather than fails, `size-limit`, `@lhci/cli`, `lighthouserc.json`, `numberOfRuns`, `resource-summary:script:size`, `@next/bundle-analyzer`, `ANALYZE=true`, the Turbopack bundle analyzer, `.next/diagnostics/analyze`, a treemap of the chunks; `web-vitals`, `web-vitals/attribution`, `useReportWebVitals`, `sendBeacon`, `onLCP`, `onINP`, `onCLS`, `longAnimationFrameEntries`, `longestScript`; a number taken from `next dev`, a before-and-after with no build stated, a 4× CPU throttle, Slow 4G, the coverage panel, a heap snapshot, the React DevTools profiler, a production profiling build; an awaited DRF call in a Server Component, `Promise.all` over independent calls, an over-wide serializer, a payload that grew with no type change, `openapi-fetch` against a heavy client; the PWA category in an old Lighthouse config, `next lint` in a CI performance step. Not here: the bytes that ship and the third-party script (`references/client-bundle-and-third-party-scripts.md`), the largest paint, the layout shift, and the long task (`references/paint-and-interaction-cost.md`), the `analyze` script itself (`references/lint-format-and-scripts.md`), the render mode of a route and the build report symbols (`references/app-router-structure.md`), the transport that carries a field report (domain 21), the compression, the CDN, and the CI workflow file (domain 22), the slow endpoint and the query count (sibling skill `django-performance-optimizer`). | `references/performance-budgets-and-measurement.md` |
| The JavaScript that reaches the browser, and the moment at which it runs — bundle size, First Load JS that jumped, code splitting, tree-shaking, `sideEffects`, unused JavaScript, a chunk that one branch needs, hydration cost; `next/dynamic`, `React.lazy`, `ssr: false`, a `loading` skeleton, an editor or a chart or a map or a date picker imported at the top of a shared layout; a barrel import, `import _ from "lodash"`, `lodash-es`, `moment`, `date-fns`, `Intl.DateTimeFormat` in place of a date package, `optimizePackageImports`, an icon package; `next/script`, `strategy`, `beforeInteractive`, `afterInteractive`, `lazyOnload`, `worker`, Partytown, `@next/third-parties`, a tag manager, an analytics tag, a session recorder, a chat widget, a facade in front of an embed; `preconnect`, `prefetchDNS`, `preinit`, a resource hint, `modulepreload`, domain sharding; `<Link prefetch>`, `prefetch={true}`, a prefetch storm, Partial Prefetching, Instant Navigations, `partialPrefetching`. Not here: the budget and the gate over these bytes (`references/performance-budgets-and-measurement.md`), the largest paint and the long task (`references/paint-and-interaction-cost.md`), the `"use client"` directive itself (`references/server-and-client-components.md`), `optimizePackageImports` as a boundary rule (`references/directory-and-module-boundaries.md`), the chart island and `ssr: false` for a canvas library (`references/charts-and-visual-encoding.md`), the facade in front of a video player (`references/image-and-video-delivery.md`), the `Intl` formatter and its time zone (`references/cell-formatting-and-export.md`), `cacheComponents` and the rest of `next.config.ts` (`references/app-router-structure.md`), the Content Security Policy over a vendor script (`references/security-headers-and-csp.md`), the consent gate over a tag (domain 23). | `references/client-bundle-and-third-party-scripts.md` |
| The element that paints, the layout that holds still, and the answer to a tap — the largest paint, an LCP element, a hero that arrives late, resource load delay, render delay, an image inside a deferred import; a layout shift, CLS that only production shows, content that moves after load, a banner that pushes the page down, a skeleton of the wrong height, a heading that reflows when the font arrives; a slow click, a button that does not press, a long task, 50 ms, `scheduler.yield`, `requestIdleCallback`, `setTimeout` as a yield, main-thread blocking, input delay, processing duration, presentation delay; `startTransition`, `useTransition`, `useDeferredValue`, a search field that drops keystrokes, one stale frame; `reactCompiler`, a hand-written `useMemo` with no measurement, a render that no change explains; a memory leak, a heap that grows over a session, two heap snapshots, a timer or an observer with no cleanup, an unbounded client cache. Not here: the thresholds, the profile, and the field report (`references/performance-budgets-and-measurement.md`), the bytes and the third-party script (`references/client-bundle-and-third-party-scripts.md`), the props of `<Image>`, `preload`, and `URL.revokeObjectURL` (`references/image-and-video-delivery.md`), the reserved ratio box and the `next/font` options (`references/layout-and-typography.md`), the virtualiser threshold and `content-visibility` (`references/data-table-and-server-driven-state.md`), the compiler config and the effect cleanup (`references/state-and-effects.md`), `useTransition` as an Action hook (`references/suspense-and-actions.md`), the animated property and `will-change` (`references/motion-primitives-and-reduced-motion.md`), `gcTime` and the bound on the cache (`references/server-state-and-query-cache.md`). | `references/paint-and-interaction-cost.md` |
| Data that becomes code in the browser — XSS, cross-site scripting, stored XSS, an injection, a sink, a script that runs from a comment field, a payload saved in a profile field; `dangerouslySetInnerHTML`, `__html`, raw HTML from a content management system, rich text, `innerHTML`, `eval`, `new Function`, `document.write`, `srcdoc`, a `javascript:` URL, an `href` built from a user value, a `postMessage` payload; sanitise, a sanitiser, DOMPurify, `isomorphic-dompurify`, `dompurify`, a protocol allowlist, `ALLOW_UNKNOWN_PROTOCOLS`; `react-markdown`, `rehype-raw`, `rehype-sanitize`, a Markdown comment that renders HTML; Trusted Types, `require-trusted-types-for`, a trusted type policy, a report-only run of one. Not here: the policy that stops the script a sanitiser missed (`references/security-headers-and-csp.md`), the endpoint that stores the value and the destination it reaches (`references/exposed-endpoints-and-destinations.md`), the parse over any value from outside the program (`references/boundary-validation-and-api-types.md`), the formula escape in an export file (`references/cell-formatting-and-export.md`), the separate origin and `Content-Disposition` for a user file (`references/served-content-and-downloads.md`), `dangerouslyAllowSVG` and `remotePatterns` (`references/image-and-video-delivery.md`), the JSON-LD block (domain 18), server-side injection and template injection (sibling skill `secure-code-auditor`). | `references/untrusted-markup-and-injection.md` |
| The rules that the browser enforces on a response — `Content-Security-Policy`, CSP, a policy that broke the application, a nonce, `x-nonce`, `strict-dynamic`, `unsafe-inline`, `unsafe-eval`, `default-src`, `script-src`, `style-src`, `connect-src`, `img-src`, `font-src`, `object-src`, `base-uri`, `form-action`, `frame-ancestors`, `upgrade-insecure-requests`, `Content-Security-Policy-Report-Only`, a violation report, a CSP evaluator, a header grader; `Strict-Transport-Security`, HSTS, `X-Content-Type-Options`, `nosniff`, `Referrer-Policy`, `Permissions-Policy`, `X-Frame-Options`, clickjacking, a hidden frame over a control, `poweredByHeader`, `x-powered-by`, `headers()` in `next.config.ts`, a header that arrives twice, a header set by both the application and the proxy, a nonce on a static route. Not here: `proxy.ts` itself and the rest of `next.config.ts` (`references/app-router-structure.md`), the CORS headers and the preflight (`references/cross-origin-and-bff-proxy.md`), the `wss://` handshake and its `Origin` check (`references/push-transport-and-connection.md`), the sink that the policy is the second defence for (`references/untrusted-markup-and-injection.md`), the strategy and the cost of a vendor script (`references/client-bundle-and-third-party-scripts.md`), the cost of a dynamic render (`references/performance-budgets-and-measurement.md`), the reverse proxy and the TLS termination (domain 22), the consent gate over a tag (domain 23). | `references/security-headers-and-csp.md` |
| What the network can reach, and where the server code goes next — a Server Action reachable by `curl`, an action id, `Next-Action`, "Failed to find Server Action", `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`, `serverActions.allowedOrigins`, an `Origin` and `Host` comparison, `X-Forwarded-Host`, a sandboxed frame that passed the check, CVE-2026-27978; a Route Handler with one export for each method, 405, 413, a body with no cap; a payload that holds a field nobody renders, a Django traceback in the interface, an object id changed in a request, broken object level authorization; SSRF, server-side request forgery, `169.254.169.254`, a metadata address, an internal range, a link preview, an image proxy, a webhook fetch, a host allowlist, `redirect: 'error'`, `rewrites()`, `redirects()`, `/_next/image?url=`, CVE-2026-64645, CVE-2026-64649; an open redirect, `?next=`, `returnTo`, a callback URL, a phishing redirect; a per-user response in a cache, `Cache-Control: public` on private content, cache confusion, CVE-2026-64647. Not here: the session gate inside an action and the redirect helper (`references/route-protection-and-permissions.md`), the fixed order inside an action (`references/data-access-and-mutations.md`), the proxy Route Handler with a constant upstream (`references/cross-origin-and-bff-proxy.md`), the parse over the input (`references/boundary-validation-and-api-types.md`), `remotePatterns` as configuration (`references/image-and-video-delivery.md`), `bodySizeLimit` and the upload threshold (`references/file-upload-and-transport.md`), the cache decision of a route (`references/caching-and-revalidation.md`), the words of a refusal (`references/error-and-empty-state-copy.md`), the DRF permission class and the rate limit (sibling skill `secure-code-auditor`). | `references/exposed-endpoints-and-destinations.md` |
| What must not cross to the browser, and what the browser runs that nobody here wrote — a secret in the bundle, an API key in a chunk, a leaked key alert, `NEXT_PUBLIC_`, a public variable that holds a credential, a rotation after a leak; `server-only`, `client-only`, a serialized prop that carries a whole record, an RSC payload that holds a private value, `experimental_taintObjectReference`, `experimental_taintUniqueValue`, `experimental.taint`; a CVE, an advisory, a security release, a patched version, `npm ls react`, `next --version`, CVE-2025-29927, `x-middleware-subrequest`, CVE-2025-55182, React2Shell, a React floor, a bundled React Server Components runtime; `pnpm audit`, `npm audit`, an audit that fails CI, severity, reachability, a dangerous package, a supply chain attack, OWASP Top 10:2025, `gitleaks`, `trufflehog`, a secret in a commit, a tracked `.env` file; SRI, `integrity`, `crossorigin`, a self-hosted vendor file, a tag manager as a code channel. Not here: `minimumReleaseAge`, the lifecycle script allowlist, the update bot, and the `gitleaks` hook (`references/dependencies-and-git-workflow.md`), the `server-only` guard as a boundary rule (`references/server-and-client-components.md`), the React floor and the compiler (`references/state-and-effects.md`), the strategy and the measured cost of a vendor script (`references/client-bundle-and-third-party-scripts.md`), the policy that admits that script (`references/security-headers-and-csp.md`), the store that must hold no token (`references/session-and-token-lifecycle.md`), the Docker build secret (domain 22), the consent gate (domain 23), server-side secret storage and password hashing (sibling skill `secure-code-auditor`). | `references/secret-boundary-and-supply-chain.md` |

The operating doctrine is integrated but has no row, because it is always in
effect and lives in this file rather than in `references/`. Each release
integrates one domain and adds its rows. Write each row as trigger vocabulary,
never as a summary. A vague row is a domain that never loads.

A domain of more than one file splits on one line. The leading phrase of a row
states what the file owns, and its `Not here` clause states what it does not.

Eighty-four seams cross the domains, and this table settles each one.

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
| 03 ↔ 09 | 03 owns the shape of a component, its parts, and its props. 09 owns the classes on those parts, and the tokens behind the classes. |
| 04 ↔ 09 | 04 owns the class order, the Prettier plugin, and the folder that the stylesheet sits in. 09 owns the theme inside that stylesheet. |
| 06 ↔ 09 | 06 owns the store that holds a theme preference. 09 owns the class that the preference selects, and the first paint that carries it. |
| 01 ↔ 10 | 01 owns the route file and the client navigation. 10 owns the focus that a navigation must move, or the message that announces it. |
| 03 ↔ 10 | 03 owns the shape of a component and its parts. 10 owns the role, the name, and the keyboard contract of those parts. |
| 04 ↔ 10 | 04 owns the lint config array and the scripts. 10 owns the `jsx-a11y` rules inside that array, and the axe assertions. |
| 06 ↔ 10 | 06 owns the four states of a data view. 10 owns the announcement of a change between two of them. |
| 07 ↔ 10 | 07 owns the gate that refuses a request. 10 owns the focus and the message that the refusal produces. |
| 08 ↔ 10 | 08 owns the event and the status of a connection. 10 owns the politeness of the announcement over them. |
| 09 ↔ 10 | 09 owns the token, the class, and the ring. 10 owns the ratio that a pair must meet, and the criteria for the indicator. |
| 03 ↔ 11 | 03 owns the shape of a field component, and where the state of a component lives. 11 owns the value of a field, and the bind that carries it. |
| 05 ↔ 11 | 05 owns the envelope, the normalizer, and the dotted path that it produces. 11 owns the map from that path onto a control. |
| 06 ↔ 11 | 06 owns the cache entry and the URL parser. 11 owns the submit that invalidates the entry, and the step that the parser carries. |
| 09 ↔ 11 | 09 owns the classes on a field control. 11 owns the error state that selects them. |
| 10 ↔ 11 | 10 owns the label, the description, the summary, and the focus. 11 owns the state that supplies each one. |
| 03 ↔ 12 | 03 owns the shape of the table component, its parts, and its list `key`. 12 owns the column model, the row model, and the virtualiser. |
| 05 ↔ 12 | 05 owns the pagination envelope, the filter parameter, and the generated type over them. 12 owns the map from the table state onto that parameter. |
| 06 ↔ 12 | 06 owns the key, the cache entry, and the four states of a data view. 12 owns the column model over that entry, and the view that a link must reproduce. |
| 08 ↔ 12 | 08 owns the pushed row and the write into the cache. 12 owns the order that the row must not disturb. |
| 09 ↔ 12 | 09 owns the tokens, and the classes on a cell. 12 owns the second channel that a series carries beside the color. |
| 10 ↔ 12 | 10 owns the APG grid keys, the announcement, and the contrast ratio. 12 owns the markup of a table, and the text alternative of a chart. |
| 05 ↔ 13 | 05 owns the request that carries the file, the multipart header, and the shape of the presigned response. 13 owns the size at which that request stops being correct, and the progress over it. |
| 07 ↔ 13 | 07 owns the session, the cookie, and the domain of that cookie. 13 owns the separate origin for user content, which the cookie must never reach. |
| 08 ↔ 13 | 08 owns the connection that the server pushes over. 13 owns the progress of an upload, which needs no connection. |
| 09 ↔ 13 | 09 owns the CSS box that a picture sits in, and the tokens on a control. 13 owns the props that fill the box, and the source set behind them. |
| 10 ↔ 13 | 10 owns the alternative text, the name of a control, and the keyboard contract. 13 owns the element that carries them, and the captions track. |
| 11 ↔ 13 | 11 owns the schema over a file field, and the submit around it. 13 owns the picker, the transport, and the progress. |
| 12 ↔ 13 | 12 owns what goes inside an export file. 13 owns how that file reaches the disk of a user. |
| 01 ↔ 14 | 01 owns the router, the prefetch, and the navigation itself. 14 owns the transition that wraps that navigation. |
| 03 ↔ 14 | 03 owns the shape of a component and the hook rules inside it. 14 owns the animation that the component starts, and its purpose class. |
| 06 ↔ 14 | 06 owns the optimistic value and its reconciliation with the server. 14 owns the visual feedback that reaches the reader first. |
| 09 ↔ 14 | 09 owns the duration, the easing, and the distance tokens, and the scale behind them. 14 owns the animation that reads them, and the reduced-motion override over them. |
| 10 ↔ 14 | 10 owns the preference, the global block, and the conformance verdict. 14 owns the reduced variant above that block, and the moment at which a focus moves. |
| 12 ↔ 14 | 12 owns the virtualiser and the row model under a long list. 14 owns the animation that must not wrap them. |
| 13 ↔ 14 | 13 owns the picture and the player. 14 owns the transition between two of them. |
| 01 ↔ 15 | 01 owns `error.tsx`, `global-error.tsx`, and `not-found.tsx` as route files. 15 owns the copy inside each one. |
| 05 ↔ 15 | 05 owns the envelope, the `code`, and the normalizer over them. 15 owns the message that each code produces. |
| 06 ↔ 15 | 06 owns the four states of a data view. 15 owns the three cases behind the empty one, and the words in each. |
| 07 ↔ 15 | 07 owns the gate that refuses a request. 15 owns the explanation that the refusal renders. |
| 08 ↔ 15 | 08 owns the connection and its status. 15 owns the degraded message over that status. |
| 09 ↔ 15 | 09 owns the box and the classes that hold a label. 15 owns the words inside it, and the length that a translation adds. |
| 10 ↔ 15 | 10 owns the name computation, the announcer, and the criterion verdict. 15 owns the words in a name and in an announcement. |
| 11 ↔ 15 | 11 owns the state of a field, and the map from a server error onto a control. 15 owns the label, the hint, and the message that each state supplies. |
| 12 ↔ 15 | 12 owns the column model, the selection, and the export. 15 owns the header word, and the bulk-action confirmation. |
| 13 ↔ 15 | 13 owns the picker, the transport, and the progress. 15 owns the words of a refusal, and of a progress label. |
| 14 ↔ 15 | 14 owns the animation that reports a state change. 15 owns the static word beside it. |
| 01 ↔ 16 | 01 owns the render mode of a route, the cache decision, and the prefetch configuration. 16 owns the first-byte budget over them, and the request count that the reader pays. |
| 03 ↔ 16 | 03 owns the render rules and the compiler configuration. 16 owns the frame that a render must meet, and the yield that a long task needs. |
| 04 ↔ 16 | 04 owns the `analyze` script and the rest of the script surface. 16 owns the budget that the analyzer output is read against. |
| 05 ↔ 16 | 05 owns the request, the client, and the pagination envelope. 16 owns the payload cost of that envelope, and the round trips that one route pays. |
| 06 ↔ 16 | 06 owns the key, the cache entry, and the prefetch. 16 owns the request count and the payload that the cache decision produces. |
| 08 ↔ 16 | 08 owns the event and the write into the cache. 16 owns the render cost of a feed that writes many times a second. |
| 09 ↔ 16 | 09 owns the token, the font setup, and the reserved box. 16 owns the layout shift that a wrong box produces, and the byte budget over the font. |
| 10 ↔ 16 | 10 owns the criterion, and it holds a veto. 16 owns the metric, and it holds none. A performance number never trades a criterion down. |
| 12 ↔ 16 | 12 owns the row count at which a virtualiser earns its cost, and the containment beside it. 16 owns the INP of a sort, and the payload of one page. |
| 13 ↔ 16 | 13 owns the props of `<Image>`, the source set, and the facade. 16 owns which element receives the early fetch, and the budget over the bytes. |
| 14 ↔ 16 | 14 owns the animated property and the reduced variant. 16 owns the frame budget and the interaction threshold that they must meet. |
| 01 ↔ 17 | 01 owns `proxy.ts`, `next.config.ts`, and the render mode of a route. 17 owns the header set that those two files emit, and the destination rule over a rewrite. |
| 02 ↔ 17 | 02 owns the parse over every value from outside the program. 17 owns the endpoint that must not act before that parse returns. |
| 03 ↔ 17 | 03 owns the component and the render. 17 owns the one prop that turns the React escape off, and the sanitiser in front of it. |
| 04 ↔ 17 | 04 owns the install policy, the cooldown, and the hook. 17 owns the judgment of a package, and of an advisory against it. |
| 05 ↔ 17 | 05 owns the CORS exchange, the CSRF token, and the proxy whose upstream is a constant. 17 owns the `connect-src` entry for that origin, and every destination that varies. |
| 07 ↔ 17 | 07 owns the gate inside an action, and the redirect helper. 17 owns the exposure that makes the gate mandatory, and the two tests that the helper runs. |
| 08 ↔ 17 | 08 owns the connection and the close code. 17 owns the `connect-src` entry that admits the `wss://` origin. |
| 09 ↔ 17 | 09 owns the token, the class, and the primitive. 17 owns the supply chain of the package behind them. |
| 10 ↔ 17 | Both hold a veto. The conflict rule ranks security above accessibility, and neither trades down to satisfy the other. A task that fails either one is a failed task. |
| 11 ↔ 17 | 11 owns the submit, the resolver, and the field error. 17 owns the endpoint that the submit reaches, and what that endpoint returns. |
| 13 ↔ 17 | 13 owns the separate origin, the disposition header, the SVG flag, and the host list. 17 owns the threat behind each one, and the policy that names the origin. |
| 15 ↔ 17 | 15 owns the words that a failure produces. 17 owns the rule that no exception text reaches those words. |
| 16 ↔ 17 | 16 owns the byte and the metric. 17 owns the policy over the script that produced them, and it takes a hash rather than spend a static render on a nonce. |

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

The router table above is the integrated material at 1.16.0, and it is the
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
- A redirect target from a request reaches `redirect()` with no leading-slash
  test and no parsed-origin test.
- `verifySession()` runs inside a `"use cache"` scope.
- A route that renders per-user data is reported as static.
- A protected RSC payload holds a private value with the cookies cleared.
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

**`accessibility-wcag` — integrated, blocking, and with a veto.** The task
fails when any one of these holds:

- A control is a generic element with a click handler, rather than a native
  element.
- An interactive element carries no accessible name, or `title` is the only
  name that it carries.
- An ARIA role or state is present that no APG pattern requires, or
  `aria-hidden` sits on a focusable element.
- The page carries no `<main>`, more than one `<h1>`, or a heading level that
  it skips.
- An image carries no decision — no empty `alt`, no informative `alt`, and no
  functional name.
- `<html>` carries no `lang`.
- A field takes its name from a placeholder, or holds an error that no
  `aria-describedby` reaches.
- A positive `tabIndex` is present, or focus enters a control and cannot leave
  it.
- A control has no visible focus indicator, or a sticky element hides the
  focused control.
- A modal does not set the initial focus, does not trap the focus, or does not
  return it on close.
- A route change moves no focus and announces nothing.
- An asynchronous result, an error, or a pushed event reaches the user with no
  live region.
- A text pair falls under 4.5:1, or a border, an icon, or a focus indicator
  that carries meaning falls under 3:1.
- A pointer target is under 24 by 24 CSS pixels, or the surface scrolls in two
  directions at 320 CSS pixels of width.
- The `viewport` export blocks the zoom, the project holds no
  `prefers-reduced-motion` block, or a drag has no single-pointer alternative.
- The lint gate omits `eslint-plugin-jsx-a11y`, or a new component carries no
  axe assertion.
- A route carries no axe check, or the five manual steps did not run.

**`design-system-and-styling` — integrated, not blocking.** Report each of
these:

- A hex color, a `px` font size, or a spacing number in a feature file, rather
  than in the theme.
- A color token that `:root`, `.dark`, and `@theme inline` do not all carry.
- A raw z-index number, rather than a name from one scale.
- A conditional class list that `cn()` does not merge, or an `!important` that
  defeats a primitive.
- A new primitive whose variants are not one typed variant map, or
  `class-variance-authority` in new code.
- A UI element hand-built from a `div` where a primitive already exists.
- An `outline-none` with no visible replacement in the same class list.
- A dark theme from `filter: invert()`, or a `dark:` utility with no
  `@custom-variant dark` behind it.
- A theme class that reaches `<html>` after the first paint.
- A reusable component whose only responsive behavior is a viewport
  breakpoint.
- A physical direction utility in new component code, with no stated reason.
- A `vh` unit on a full-height section, or an image or a skeleton with no
  reserved box.
- A font that `next/font` does not self-host, or a fixed `px` height on a text
  container.

**`forms-and-validation` — integrated, not blocking.** Report each of these:

- A `useForm` call with no resolver, or a second model of the same rules beside
  the schema.
- A schema that carries a `.default()`, a `.transform()`, or a `.catch()`, and
  a form generic that is not the `z.input` and `z.output` pair.
- A field with no `defaultValues` entry, or a cross-field rule with no `path`.
- A `Controller` around a native input, or a native input with no `register`.
- A validation mode of `onChange` from the mount.
- A `watch()` at the root of a form, or a field array row keyed by its index.
- A server field error that reaches a toast rather than its control, or a
  `non_field_errors` value that reaches no form-level region.
- A submit button bound to `isValid` rather than to `isSubmitting`, or bound to
  neither.
- A `reset()` in a `catch` or a `finally`, or a 5xx on a submit that discards
  the values.
- A 429 that ignores `Retry-After`, or a retried POST with no idempotency key.
- A failed submit that moves focus nowhere, or that reports through a toast
  alone.
- A log or a console call that carries a field value.
- A step index that only memory holds, or a flow whose values a reload
  destroys.
- A form with a dirty state and no guard on a `<Link>` click and a reload.
- A password, a payment value, or a personal identifier in a query string or in
  a stored draft.

**`data-tables-and-visualization` — integrated, not blocking.** Report each of
these:

- A table that fetches the whole set, and pages it in the browser.
- `manualPagination`, `manualSorting`, or `manualFiltering` set alone rather
  than as the three together.
- A manual table with neither `rowCount` nor `pageCount`.
- A `getRowId` that returns the index of the array, or a `useReactTable` call
  with no `getRowId`.
- A page, a sort, a filter, or a search term that the address bar does not
  carry.
- A column definition that each render creates again, or an `Intl` formatter
  built inside a cell.
- A paginated query with no `placeholderData: keepPreviousData`.
- A `role="grid"` with no arrow-key model, or a sortable header with no
  `aria-sort` and no `<button>` inside the `<th>`.
- A virtualiser under about 200 rows, or a virtualised body with no
  `aria-rowcount` and no `aria-rowindex`.
- A table that CSS lays out as a grid, and that declares no role again.
- A bulk action over the whole set that sends a list of identifiers rather than
  the filter.
- A chart library imported into a Server Component, or a canvas chart with no
  `ssr: false`.
- A chart with no text alternative, or a series that only a color separates.
- A number or a date formatted with no locale, or a date with no `timeZone`.
- An export that covers the current page rather than the current filter.
- A client-built file that escapes no leading `=`, `+`, `-`, `@`, tab, or
  carriage return, or that carries no UTF-8 byte order mark.
- An `xlsx` dependency taken from npm.

**`media-and-file-handling` — integrated, not blocking.** Report each of these:

- A drop zone that holds no real `<input type="file">`, or an upload that a
  keyboard cannot complete.
- A type decision taken from `file.type` or from the extension alone.
- A client-side check with no comment that states that the server re-validates.
- A file above the project threshold sent through a Server Action or a Route
  Handler, or a threshold that the repository never states.
- An upload with no percentage for each file, no cancel control, no retry
  control, or an error message that names no cause.
- A progress report taken from `fetch` rather than from `XMLHttpRequest`.
- A presigned POST that sends the file before the policy fields, or a rejection
  from storage that reaches the user as a status code.
- A direct upload with no confirm call, or a file that the interface presents as
  usable before the confirm returns 2xx.
- A user photograph stored with its EXIF block, with no stated need for it.
- An image with neither `next/image` nor a written reason, or one with no
  `width` and `height` and no `fill` over a reserved ratio.
- A responsive image or a `fill` image with no `sizes`.
- The element that paints the largest area carrying `loading="lazy"`, or
  carrying neither `preload` nor `fetchPriority="high"`.
- A `priority` prop on an `<Image>` under Next 16.
- A `remotePatterns` entry with a `**` hostname, or `dangerouslyAllowSVG` set.
- A `URL.createObjectURL` call with no matching `URL.revokeObjectURL` call.
- A `<video>` with no captions track and no transcript, or a player control that
  a keyboard cannot reach.
- A third-party player embedded with no facade in front of it.
- A user-supplied file served from the origin of the application, or served with
  no `Content-Disposition: attachment` and no `X-Content-Type-Options: nosniff`.
- A private download with no signed URL and no session check inside the handler,
  or a handler that reads the whole file into memory.
- A `showSaveFilePicker` call with no feature detection.

**`motion-and-interaction` — integrated, not blocking.** Report each of these:

- An animation in the diff whose purpose class nobody can name.
- A `width`, `height`, `top`, `left`, `margin`, `box-shadow`, or `filter`
  animated on an interaction path, with no written reason.
- A millisecond value or a `cubic-bezier()` curve typed into a feature file.
- No reduced variant at all, or one built from `animation: none` rather than
  from the tokens.
- A reduced-motion gate that only JavaScript reads, so the first paint still
  moves.
- An animated state change whose only signal is the movement.
- A `will-change` in a base class, or one that no code removes.
- A dialog or a popover whose exit never runs, with no `@starting-style` and no
  `transition-behavior: allow-discrete`.
- A reveal that animates `height` to `auto`, rather than `grid-template-rows` or
  `interpolate-size`.
- Motion that runs past 5 seconds with no pause, stop, or hide control, or
  content that flashes more than three times in one second.
- A focus that moves before an entrance ends, or that lands on an element the
  transition then moves off the screen.
- A spinner on a response under about 100 milliseconds.
- A `view-transition-name` that two elements share in one snapshot, or a
  `startViewTransition` call with no feature detection.
- A `<ViewTransition>` or an `addTransitionType` call with no statement that the
  API is experimental, or an import name that nobody read from the installed
  React build.
- A `motion/react` import in a Server Component file, rather than
  `motion/react-client` or a file that carries `"use client"`.
- An `AnimatePresence` or a `layout` prop around a virtualised list.
- The full Motion component in the first render, with no `LazyMotion` and no
  stated reason.
- A `framer-motion` import, a `react-transition-group` entrance in new code, or
  a Lottie file behind a spinner.
- A sortable list on the native drag and drop API, a sensor list with no
  keyboard sensor, or a drag with no single-pointer path beside it.
- A scroll listener that writes a style, or a handler that moves the page by its
  own amount.
- A scroll-driven animation with no `@supports` guard, or with no
  `animation-fill-mode`.
- A parallax layer that `prefers-reduced-motion: no-preference` does not gate.
- A non-trivial animation that no Performance recording covers at a 4× CPU
  throttle.

**`ux-writing-and-content-design` — integrated, not blocking.** Report each of
these:

- A control whose label is `Submit`, `OK`, `Yes`, `No`, or `Click here`, or one
  whose label names a mechanism rather than an outcome.
- One act that carries a different verb on the control, the confirmation, the
  result, and the state on the screen.
- A destructive confirmation that states no consequence and no scope, or a
  confirm control that repeats no verb.
- An `aria-label` that does not hold the visible text of its control.
- A format constraint that only a rejected submit teaches.
- A hint in a `title` attribute, or a placeholder that carries a label or an
  instruction.
- A label in title case, or a label with a trailing period.
- Link text that names no destination.
- An instruction that names a color, a shape, or a position rather than a
  control.
- A pre-checked consent control, a confirmshaming decline, or a false deadline.
- `error.message` or `ApiError.message` rendered as the message of a view.
- An error boundary with no recovery control, or one that never reaches the
  digest.
- A provider hook inside `global-error.tsx`, or a root boundary with no
  `<html lang>`.
- An error code with no entry in the message map, or a map with no test against
  the generated enum.
- One empty state for the three cases, or an empty case with no action beside
  it.
- A dropped connection reported as an empty list.
- A refused route that states no permission, or a message that names a record
  the account may not read.
- A toast as the only report of a failure, or a toast that repeats a message
  the view already holds.
- A user-facing string in the markup, outside `global-error.tsx`.
- A sentence built from two keys and a template string.
- A count joined to a noun, or a plural message that omits a category a target
  locale needs.
- A list joined by hand, rather than by `Intl.ListFormat`.
- A user value interpolated into a sentence with no isolate around it.
- A project with no pseudo-locale, or a primary surface that no snapshot covers
  in it.

**`performance-and-web-vitals` — integrated, not blocking.** Report each of
these:

- A performance claim with no number before it and no number after it, or a
  number taken from `next dev`.
- A number with no build, no CPU profile, and no network profile stated beside
  it.
- A route class with no First Load JS budget, or a budget that no gate enforces.
- A Lighthouse configuration that runs fewer than five times, or that asserts on
  the PWA category.
- An increase in First Load JS that the pull request does not explain.
- A project that reports no field metric, or a verdict taken from a laboratory
  score while the field percentile is poor.
- A transform with no browser API and no interaction that runs on the client.
- A heavy client widget imported at the top of a shared layout, or a dynamic
  import with no skeleton of the size of the final component.
- `moment`, a default `lodash` import, or a date or a number formatted by a
  package where `Intl` answers.
- A third-party script with no recorded cost, no named owner, or no stated
  strategy.
- `beforeInteractive` on an analytics, a marketing, or a chat script, or
  `strategy="worker"` anywhere in the project.
- A blanket prefetch prop on a link-dense page.
- A route that names no element as the largest paint, or a largest-paint element
  inside a deferred import.
- An element that mounts above on-screen content with no space reserved from the
  first paint.
- An interaction handler that runs past 50 ms at a 4× CPU throttle, or a
  `scheduler.yield()` call with no fallback beside it.
- The value of an input updated inside a transition.
- Independent server calls in one route awaited one after another.
- A heap that grows across one navigation cycle, with no snapshot taken.

**`frontend-security` — integrated, blocking, and with a veto.** The task fails
when any one of these holds:

- A `dangerouslySetInnerHTML` takes a value that no sanitiser returned, or one
  that no comment names a sanitiser and a configuration for.
- A `rehype-raw` plugin runs with no `rehype-sanitize` after it.
- An `href`, a `src`, or a `srcdoc` takes a value from outside the program with
  no parse over it.
- `eval`, `new Function`, `document.write`, or `innerHTML` is present in the
  code that this repository owns.
- Production carries no `Content-Security-Policy`.
- The production `script-src` carries `'unsafe-inline'` or `'unsafe-eval'` with
  no written exception and no end date.
- A prerendered route carries a nonce, or a route becomes dynamic only to
  obtain one.
- `connect-src` omits the Django origin, or a `wss://` endpoint that the
  application opens.
- `frame-ancestors` is absent, `poweredByHeader` is not `false`, or more than
  one layer emits a security header.
- A Route Handler exports a catch-all method, or accepts a body with no size
  cap.
- A payload that reaches the browser holds exception text, or a record that the
  interface does not render.
- A `fetch`, a `rewrites()` destination, a `redirects()` destination, or an
  image host takes its target from request input with no allowlist.
- An outbound request to an allowlisted host follows a redirect to a host that
  the allowlist refuses.
- A per-user response carries a public cache directive at the CDN or at the
  reverse proxy.
- A credential, a token, or a private address sits in a `NEXT_PUBLIC_`
  variable, or crosses the boundary inside a serialized prop.
- A module that reads a secret does not import `server-only`.
- The installed Next.js is below the current security release, or the installed
  React is below 19.2.6.
- An audit report is dismissed with no written judgment of its severity and its
  reachability.
- A vendor script loads from another origin with no `integrity` attribute and
  no written reason.

**Conflict rule.** security > accessibility > correctness > performance >
developer convenience. No level trades down to satisfy a level above it. When
a requirement appears to force such a trade, say so and stop rather than take
it silently.
