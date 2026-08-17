# App Router structure

Next.js 16.3 baseline, React 19.2.4 or later, Node 20.9 or later. This file
owns the route tree. It rules on which file the App Router loads for a URL,
and on how a route reads request data. It also rules on `proxy.ts`, which runs
before the route, and on the `next.config.ts` keys that decide route behavior.

The server and client split inside a route is
`references/server-and-client-components.md`. Where a route gets its data is
`references/data-access-and-mutations.md`. How long that data lives is
`references/caching-and-revalidation.md`.

## Principle

A file-system router makes the directory tree the public contract. A file
move is a URL change. Read the tree, and you know the URLs.

A request-scoped value arrives with the request, not with the module. A
framework that streams its response must therefore make that value
asynchronous. Await it.

The layer in front of the route rewrites, redirects, and sets headers. It is
not a security boundary. It runs before the router resolves the route, it has
no access to the data layer, and a header that reaches it can be forged.

Every route declares four facts: the render mode, the data source, the cache
strategy, and the invalidation trigger. A route that cannot state all four is
not designed yet.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The route files

| File | The router uses it for | The rule |
| --- | --- | --- |
| `layout.tsx` | The shell around a segment and its children | A layout persists across a navigation and does not re-render. |
| `page.tsx` | The unique UI of a route | The route is public only when this file exists. |
| `template.tsx` | A shell beside `layout.tsx` | Prefer `layout.tsx`. |
| `loading.tsx` | The Suspense fallback of the segment | Required for a dynamic segment. |
| `error.tsx` | The error boundary of the segment | Required for a fallible segment. It exports `reset()`. |
| `global-error.tsx` | The error UI for the root layout | — |
| `not-found.tsx` | The UI for `notFound()` | — |
| `forbidden.tsx` | The UI for `forbidden()` | The call is EXPERIMENTAL in 16.3, and it needs `authInterrupts: true`. |
| `unauthorized.tsx` | The UI for `unauthorized()` | The call is EXPERIMENTAL in 16.3, and it needs `authInterrupts: true`. |
| `default.tsx` | The fallback UI of a parallel route slot | Next 16 fails the build when a slot has none. |
| `route.ts` | An HTTP endpoint at that path | Use it for a public API, a webhook, a file upload, or a stream. |
| `proxy.ts` | The code that runs before the route | It rewrites, redirects, and sets headers. It never authorizes. |
| `instrumentation.ts` | The server start hook | The content is domain 21 `observability-and-resilience`. |
| `instrumentation-client.ts` | The browser start hook | The content is domain 21 `observability-and-resilience`. |

`middleware.ts` is not on this list. Next 16 renamed it to `proxy.ts`, so
`middleware.ts` is alive only in legacy code.

### The folder tokens

| Token | Effect on the URL | Use it for |
| --- | --- | --- |
| `(group)` | None | A shared layout, or a folder split that the URL must not show |
| `_folder` | The folder is private and routes nothing | Colocated code inside `app/` |
| `[param]` | One segment | An id or a slug |
| `[...slug]` | One or more segments | A path of unknown depth |
| `[[...slug]]` | Zero or more segments | The same, and the parent path too |
| `@slot` | None | A parallel route |
| `(.)` `(..)` `(...)` | None | An intercepting route |

An intercepting prefix counts route segments, not folders. A `@slot` folder is
not a segment. Count the segments before you choose the prefix.

### A parallel route slot needs a default file

```tsx
// Wrong: the slot has a page and no default file.
// app/@modal/page.tsx exists. app/@modal/default.tsx does not.
// Failure: next build fails. A hard reload of the intercepted URL also
// leaves the slot empty, because the router has no fallback to render.
export default function Modal() {
  return <dialog open>...</dialog>;
}
```

```tsx
// Correct: app/@modal/default.tsx
export default function Default() {
  return null;
}
```

Give every `@slot` a `default.tsx`. Return `null`, or call `notFound()`.

### Request data is asynchronous

`params`, `searchParams`, `cookies()`, `headers()`, and `draftMode()` are
promises in Next 16. Await each one.

```tsx
// Wrong: params is read as a plain object.
// Failure: the build stops with "Cannot access Request information
// synchronously". Under cacheComponents it is a hard error, not a warning.
export default function Page({ params }: { params: { slug: string } }) {
  return <h1>{params.slug}</h1>;
}
```

```tsx
// Correct: params is a promise, and the type comes from the generated helper.
export default async function Page(props: PageProps<'/blog/[slug]'>) {
  const { slug } = await props.params;
  return <h1>{slug}</h1>;
}
```

`PageProps` and `LayoutProps` are generated. Run `npx next typegen` to
generate them. Run `npx @next/codemod@latest next-async-request-api .` on a
Next 15 codebase. The codemod leaves an `@next-codemod-error` comment, or an
`UnsafeUnwrapped` type, where a human must decide. Both markers fail the
build. Remove each one before you report the work complete.

A route that reads `cookies()`, `headers()`, or the `searchParams` prop is
dynamic. Do not try to make it static. Put the dynamic read behind
`<Suspense>` instead, and keep the shell static.

### `proxy.ts` never authorizes

```ts
// Wrong: proxy.ts decides the authorization.
// Failure: the check runs before the router, it has no data layer, and a
// forged request header can skip it. CVE-2025-29927 is this failure class:
// a spoofed x-middleware-subrequest header skipped the file completely on
// every self-hosted version up to 15.2.2.
import { NextResponse, type NextRequest } from "next/server";

export async function proxy(request: NextRequest) {
  const user = await verifySession(request.cookies.get("sessionid")?.value);
  if (!user?.isStaff) {
    return NextResponse.redirect(new URL("/login", request.url));
  }
  return NextResponse.next();
}
```

```ts
// Correct: proxy.ts redirects on the absence of a cookie, as a UX
// optimization only.
import { NextResponse, type NextRequest } from "next/server";

export function proxy(request: NextRequest) {
  if (request.nextUrl.pathname.startsWith("/dashboard")) {
    if (!request.cookies.has("sessionid")) {
      return NextResponse.redirect(new URL("/login", request.url));
    }
  }
  return NextResponse.next();
}
```

```tsx
// Correct: the real check runs in the layout, on the server, with the data.
// app/dashboard/layout.tsx
import { redirect } from "next/navigation";
import { getSession } from "@/lib/dal/session";

export default async function DashboardLayout(props: LayoutProps<'/dashboard'>) {
  const session = await getSession();
  if (!session?.isStaff) redirect("/login");
  return <>{props.children}</>;
}
```

Permitted in `proxy.ts`: a rewrite, a redirect, a response header, a locale
decision, and a test for the presence of a cookie. Forbidden in `proxy.ts`: a
database call, a session verification, and any decision that a forged header
must not control. Verify the session in the layout, the page, the Server
Action, or the data access layer.

Rename `middleware.ts` to `proxy.ts`, and export a function named `proxy`. Run
`npx @next/codemod@latest middleware-to-proxy .`. Rename
`skipMiddlewareUrlNormalize` to `skipProxyUrlNormalize`. The old file is
deprecated, not removed: it still runs on the Edge runtime and it logs a
warning. Keep it only when the Edge runtime is a stated requirement.

### `next.config.ts`

```ts
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  typedRoutes: true,
  cacheComponents: true,
  images: {
    remotePatterns: [
      { protocol: "https", hostname: "media.example.com", pathname: "/**" },
    ],
  },
};

export default nextConfig;
```

| Removed or renamed in Next 16 | Replacement |
| --- | --- |
| `serverRuntimeConfig`, `publicRuntimeConfig` | Environment variables |
| `experimental.ppr`, the `experimental_ppr` export | Cache Components, where Partial Prerendering is automatic |
| `experimental.dynamicIO` | `cacheComponents` |
| `experimental.typedRoutes` | Top-level `typedRoutes` |
| `experimental.turbopack` | Top-level `turbopack` |
| `images.domains` | `images.remotePatterns` |
| `devIndicators` sub-options | — |
| `unstable_rootParams()` | — |
| AMP, `useAmp`, `amp: true` | — |
| `next/legacy/image` | `next/image` |
| `next lint` | ESLint or Biome, run directly. Codemod: `next-lint-to-eslint-cli` |

Every key in the left column is alive only in legacy code.

`next build` no longer lints. Add the lint command to the build script, or to
CI, or the check disappears.

Next 16 also changed the `next/image` defaults. `minimumCacheTTL` moved from
60 seconds to 14400 seconds. `imageSizes` no longer includes `16`. `qualities`
moved from the full range to `[75]`.

`dangerouslyAllowLocalIP` is blocked. `maximumRedirects` defaults to 3. A local
`src` that carries a query string needs an `images.localPatterns` entry. Read
each default before you assume the Next 15 behavior.

### Environment variables

A variable without the `NEXT_PUBLIC_` prefix stays on the server. A variable
with the prefix is inlined into the browser bundle at build time. NEVER put a
secret, an API key, or a Django admin URL in a `NEXT_PUBLIC_` variable. A
single Docker image that must read a public variable at run time needs
`next-runtime-env`; every other case bakes the value at build time.

### Version and patch discipline

Read the installed version before you generate code. Next 16 requires Node
20.9 or later and TypeScript 5.1 or later. Node 18 is unsupported.

Next.js publishes security releases every month. The release of 2026-07-20
fixed nine CVEs in 16.2.11, the Active LTS line, and in 15.5.21, the
Maintenance LTS line. Four were HIGH, and two of those were server-side
request forgery, one of them through `rewrites`. Upgrade to a patched version.
Treat every `rewrites` destination as untrusted until domain 17
`frontend-security` rules on it.

## Verification

```bash
# 1. Read the installed version before you write code.
node -p "require('next/package.json').version"

# 2. Build, then read the route report and the legend that it prints.
#    Every route symbol must match its declared render mode.
pnpm build
pnpm build --debug

# 3. Find a synchronous read of request data. This must print nothing.
#    The look-behind needs the PCRE2 engine, which -P selects.
rg -P '(?<!await )\b(params|searchParams)\.[a-zA-Z]' -g '*.tsx' -g '*.ts' .
rg '@next-codemod-error|UnsafeUnwrapped' .

# 4. Find a legacy middleware file. This must print nothing.
test -f middleware.ts && echo "FAIL: rename to proxy.ts"
test -f src/middleware.ts && echo "FAIL: rename to proxy.ts"

# 5. Find a parallel route slot with no default file.
fd -t d '@' app | while read s; do
  test -f "$s/default.tsx" || echo "MISSING default: $s"
done

# 6. Confirm that proxy.ts holds no data call and no session check.
rg -n 'fetch\(|verifySession|getSession|DJANGO_URL' proxy.ts src/proxy.ts

# 7. Find a segment with no loading.tsx. Each hit needs a <Suspense> in the
#    page instead.
fd -t d . app | while read d; do
  test -f "$d/page.tsx" || continue
  test -f "$d/loading.tsx" || echo "no loading.tsx: $d"
done
```

## Review checklist

- [ ] Does every route state its render mode, its data source, its cache
      strategy, and its invalidation trigger?
- [ ] Are `params`, `searchParams`, `cookies()`, `headers()`, and
      `draftMode()` always awaited?
- [ ] Is the code free of `@next-codemod-error` and `UnsafeUnwrapped`?
- [ ] Does every `@slot` folder hold a `default.tsx`?
- [ ] Does an intercepting prefix count route segments and skip `@slot`
      folders?
- [ ] Is `middleware.ts` absent, unless the Edge runtime is a requirement?
- [ ] Does `proxy.ts` hold only rewrites, redirects, headers, locale logic,
      and cookie-presence tests?
- [ ] Does a layout, a page, a Server Action, or the data access layer verify
      the session?
- [ ] Does every dynamic segment hold a `loading.tsx`, or a `<Suspense>` in
      the page?
- [ ] Does every fallible segment hold an `error.tsx`?
- [ ] Is `next.config.ts` free of the keys that Next 16 removed?
- [ ] Is every `NEXT_PUBLIC_` variable free of secrets?
- [ ] Does the build report agree with the declared render mode of each route?

## Handoffs

- The types of a fetched value, and the generics that carry it → domain 02
  `typescript-type-system-discipline`, in
  `references/type-modeling-and-narrowing.md`. The rule that a generated route
  type is never edited by hand is
  `references/typescript-config-and-enforcement.md`.
- The DRF schema and the generated client →
  `references/openapi-schema-and-codegen.md`. The error envelope and the client
  that carries a request are
  `references/api-client-and-request-safety.md`. CORS, the CSRF token, and the
  proxy Route Handler are `references/cross-origin-and-bff-proxy.md`. The
  server side of that contract belongs to the sibling skill
  `django-api-contract`.
- The four enforcement layers above `proxy.ts`, the page gate, the Server
  Action gate, and the role checks in the UI →
  `references/route-protection-and-permissions.md`. That file records
  CVE-2026-64642, which is the second bypass of this file. The session, the
  cookie attributes, and the refresh are
  `references/session-and-token-lifecycle.md`. The server-side permission
  classes belong to the sibling skill `secure-code-auditor`.
- The values of the security headers, and the CSP that `next.config.ts`
  emits → domain 17 `frontend-security`. Not integrated yet.
- The content of the metadata files, such as `opengraph-image.tsx` → domain 18
  `seo-and-metadata`. Not integrated yet. This file owns only the existence of
  the file at the route.
- Locale routing with `next-intl`, whose proxy code now lives in `proxy.ts` →
  domain 19 `internationalization-and-rtl`. Not integrated yet.
- Docker, `output: 'standalone'`, and the reverse proxy in front of Node →
  domain 22 `build-deploy-and-runtime-ops`. Not integrated yet.
