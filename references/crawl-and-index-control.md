# Crawl and index control

Next.js 16.3 with `proxy.ts` on the Node runtime, React 19.2.6 or later,
TypeScript 5.9, against a Django and DRF backend. This file owns whether a
crawler reaches a route, and whether it may keep the route in an index. The
subjects are the indexability decision that each route carries, and `robots.ts`
with the difference between a refused crawl and a refused index. They also
include the environment gate that keeps a preview deployment out of the
results, and the sitemap that the DRF data produces. The last subjects are the
status code that a missing record returns, and the redirect that a rename ships
with it.

What one route declares about itself is
`references/route-metadata-and-social-cards.md`. The machine-readable claim
about its content is `references/structured-data-and-rich-results.md`.

## Principle

Indexability is a decision, and never a default. Each public route carries one,
and somebody wrote it down.

A rule that a crawler may not fetch is a rule that the crawler never reads. A
refused crawl and a refused index are two different controls, and one of them
blocks the other.

A sitemap is a hint. The status code is the fact. Where the two disagree, the
status code wins.

A 200 response for a record that does not exist is a false statement to the
crawler and to the reader.

A URL that changes with no redirect beside it loses every link that pointed at
it. The redirect belongs in the same change as the rename.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The indexability decision, one for each route

| The class | The route declares | The sitemap |
| --- | --- | --- |
| Public, and the product wants it found | No `robots` field, so the default applies | The route is listed |
| Public, and the product does not want it found, such as a thank-you page or a filtered view | `robots: { index: false, follow: true }` | The route is absent |
| Behind a session, such as a dashboard | `robots: { index: false, follow: false }` | The route is absent |

Write the decision for every route, including the ones that need no field. A
route with no written decision is a route where nobody chose.

Spend no effort on a share card or on a description for a route in the third
class. `references/route-protection-and-permissions.md` owns the gate that
gives the route its session, and that domain holds a veto.

### A refused crawl hides the refused index

```ts
// Wrong: the route sets noindex, and robots.txt also refuses the path.
// Failure: the crawler never fetches the route, so it never reads the
// noindex. One external link is then enough to list the bare URL, with no
// title and no description under it, and nothing on the route can remove it.
import type { MetadataRoute } from "next";

export default function robots(): MetadataRoute.Robots {
  return { rules: [{ userAgent: "*", disallow: ["/reports/"] }] };
}
```

```ts
// Correct: src/app/robots.ts. The crawl is allowed, and the route itself
// refuses the index.
import type { MetadataRoute } from "next";
import { env } from "@/env";

export default function robots(): MetadataRoute.Robots {
  const origin = env.NEXT_PUBLIC_SITE_ORIGIN;
  return {
    rules: [
      {
        userAgent: "*",
        allow: "/",
        disallow: ["/dashboard/", "/api/", "/search"],
      },
    ],
    sitemap: `${origin}/sitemap.xml`,
    host: origin,
  };
}
```

A `disallow` rule refuses the fetch. A `noindex` value refuses the index. Use
`disallow` for a path that wastes the crawl and holds nothing worth indexing,
such as an internal search route. Use `noindex` for a route that must not
appear in the results.

NEVER use both on one path. The rule that the route carries is the rule that
the crawler cannot read.

The `sitemap` field takes an absolute URL, so it reads the origin from the
parsed environment module.

### The environment gate

```ts
// Wrong: the preview deployment serves the production rules.
// Failure: the staging origin is indexed. It then competes with production
// for the same content, and it exposes work that nobody has released.
import type { MetadataRoute } from "next";
import { env } from "@/env";

export default function robots(): MetadataRoute.Robots {
  const origin = env.NEXT_PUBLIC_SITE_ORIGIN;
  return {
    rules: [{ userAgent: "*", allow: "/" }],
    sitemap: `${origin}/sitemap.xml`,
  };
}
```

```ts
// Correct: src/app/robots.ts refuses everything outside production.
import type { MetadataRoute } from "next";
import { env } from "@/env";

export default function robots(): MetadataRoute.Robots {
  const origin = env.NEXT_PUBLIC_SITE_ORIGIN;
  if (env.NEXT_PUBLIC_SITE_ENV !== "production") {
    return { rules: [{ userAgent: "*", disallow: "/" }] };
  }
  return {
    rules: [{ userAgent: "*", allow: "/", disallow: ["/dashboard/", "/api/"] }],
    sitemap: `${origin}/sitemap.xml`,
    host: origin,
  };
}
```

The gate reads an environment value. It never reads the host of the request,
and it is never a rule that somebody remembers to apply.

Send `X-Robots-Tag: noindex` from the same deployment. A robots file refuses
the crawl, and the header refuses the index for a URL that a crawler already
holds. Two controls are correct here, because the deployment must never be
indexed under any condition.

`references/security-headers-and-csp.md` owns the header set, and it states
that one layer emits it. Add this header to that layer.
`references/app-router-structure.md` owns `proxy.ts` and states that a response
header is permitted work there.

### The sitemap comes from the data

```ts
// Wrong: src/app/sitemap.ts holds a list that a person wrote.
// Failure: the list is correct on the day of the commit. Every record added
// after it is absent, and no check in the repository compares the list
// against the data.
import type { MetadataRoute } from "next";
import { env } from "@/env";

export default function sitemap(): MetadataRoute.Sitemap {
  const origin = env.NEXT_PUBLIC_SITE_ORIGIN;
  return [
    { url: `${origin}/`, lastModified: new Date() },
    { url: `${origin}/products/first-product`, lastModified: new Date() },
  ];
}
```

```ts
// Correct: src/app/sitemap.ts reads the published records.
import type { MetadataRoute } from "next";
import { env } from "@/env";
import { listPublishedProducts } from "@/lib/dal/products";

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const origin = env.NEXT_PUBLIC_SITE_ORIGIN;
  const products = await listPublishedProducts();
  return products.map((product) => ({
    url: `${origin}/products/${product.slug}`,
    lastModified: new Date(product.updatedAt),
  }));
}
```

The second sample also fixes `lastModified`. A call to `new Date()` reports the
moment of the request for every URL. The field then carries no information, and the
crawler learns to ignore it. The route can also no longer be cached, because a
route that reads the clock is dynamic.
`references/caching-and-revalidation.md` owns that rule.

`lastModified` is the one sitemap field that the search engine uses. It ignores
`changeFrequency` and `priority`, so spend no effort on either.

List only URLs that a signed-out reader can open, and only ones that answer
200. A sitemap that lists a route behind a session advertises that route.

### Above 50,000 URLs the sitemap splits

A single sitemap file holds 50,000 URLs or 50 MB, whichever comes first. Above
either limit, `generateSitemaps` produces numbered children and an index over
them.

```ts
// Wrong: the signature of Next 15.
// Failure: Next 16 resolves the id from a promise. The build throws
// "id.includes is not a function", and the route produces no sitemap at all.
import type { MetadataRoute } from "next";
import { buildEntries } from "@/lib/seo/entries";

export default async function sitemap({
  id,
}: {
  id: string;
}): Promise<MetadataRoute.Sitemap> {
  return buildEntries(Number(id) * 50_000);
}
```

```ts
// Correct: src/app/products/sitemap.ts on Next 16.
import type { MetadataRoute } from "next";
import { env } from "@/env";
import { countPublishedProducts, listPublishedProducts } from "@/lib/dal/products";

const PAGE_SIZE = 50_000;

export async function generateSitemaps() {
  const total = await countPublishedProducts();
  const pages = Math.max(1, Math.ceil(total / PAGE_SIZE));
  return Array.from({ length: pages }, (_, index) => ({ id: index }));
}

export default async function sitemap(props: {
  id: Promise<string>;
}): Promise<MetadataRoute.Sitemap> {
  const origin = env.NEXT_PUBLIC_SITE_ORIGIN;
  const page = Number(await props.id);
  const products = await listPublishedProducts({ page, pageSize: PAGE_SIZE });
  return products.map((product) => ({
    url: `${origin}/products/${product.slug}`,
    lastModified: new Date(product.updatedAt),
  }));
}
```

Stay with one `sitemap.ts` below the limit. The split adds a route, a count
call, and an index file, and it earns nothing under 50,000 URLs.

### The backend seam

This domain meets Django and DRF at the enumeration. The schema that types the
records is `references/openapi-schema-and-codegen.md`, and the query behind a
slow endpoint belongs to the backend.

| The seam point | The client decision | The reason |
| --- | --- | --- |
| A sitemap over tens of thousands of rows | Ask the backend for `CursorPagination`, and never for a page or an offset | An offset scan slows as the table grows. A concurrent insert also makes one row appear twice, or not at all, part way through the walk. Cursor pagination needs one unchanging ordering field, and that field needs an index. |
| A sitemap over a few hundred rows | `PageNumberPagination` is enough | The cursor earns nothing on a small, stable set. |
| The value of `lastModified` | Map it from a server timestamp field, such as `updated_at` | It is the one field that the search engine reads. |
| A serializer field that the backend renames | Regenerate the types, then run the typecheck | With fresh types the build fails, which is the correct outcome. With stale types the field resolves to `undefined`, and nothing reports it. |
| Which rows the enumeration may return | Only rows that a signed-out reader can open | `secure-code-auditor` owns the server-side permission check. This file owns the rule that the sitemap advertises no private URL. |

### The lifetime of the sitemap response

A sitemap route is a route handler, and it caches by default until it reads
something at request time. A call to Django is data that the route did not
cache, so the route falls back to a render for each request.

Two forms fix it, and the installed configuration decides which one.

| The configuration | The form | The rule |
| --- | --- | --- |
| Cache Components on | `"use cache"` on a helper that holds the call, with a `cacheLife` and a `cacheTag` | The directive cannot sit in the body of a route handler. Put it on the helper that the handler calls. |
| Cache Components off | `export const revalidate = 3600` | The build must be able to analyze the value, so it takes a literal. `60 * 60` is not a literal, and the build rejects it. |

Revalidate the sitemap when a record publishes, through the same tag that the
mutation already invalidates.
`references/caching-and-revalidation.md` owns all four revalidation APIs, and
`references/data-access-and-mutations.md` owns the order inside a mutation.

### A missing record answers 404

```tsx
// Wrong: the existence check sits below a Suspense boundary.
// Failure: the response has already begun to stream with a 200 status when
// notFound() runs. The reader sees the not-found view, and the crawler
// records a 200 for a page with no content. That is a soft 404, and the
// crawler may index it or spend its budget on it.
import { Suspense } from "react";
import { ProductDetail, ProductSkeleton } from "@/features/products/detail";

export default async function Page(props: PageProps<'/products/[id]'>) {
  return (
    <Suspense fallback={<ProductSkeleton />}>
      <ProductDetail params={props.params} />
    </Suspense>
  );
}
```

```tsx
// Correct: the check runs before anything streams.
import { Suspense } from "react";
import { notFound } from "next/navigation";
import { getProduct } from "@/lib/dal/products";
import { ProductReviews, ReviewsSkeleton } from "@/features/products/reviews";

export default async function Page(props: PageProps<'/products/[id]'>) {
  const { id } = await props.params;
  const product = await getProduct(id);
  if (!product) notFound();
  return (
    <>
      <h1>{product.title}</h1>
      <Suspense fallback={<ReviewsSkeleton />}>
        <ProductReviews id={id} />
      </Suspense>
    </>
  );
}
```

Resolve the record in the segment above the boundary, or refuse the request in
`proxy.ts`. A boundary below the check is correct, because it defers only the
parts that the reader can wait for.

CAUTION: a non-200 response may skip the render queue of a search engine
altogether. Handle the status and the canonical link on the server. Client code
that fixes either one after hydration never runs for that reader.

`references/app-router-structure.md` owns `not-found.tsx` as a route file, and
`references/error-and-empty-state-copy.md` owns the words in it.

### A rename ships its redirect

| The condition | The status | The mechanism |
| --- | --- | --- |
| The route moved, and the old address will not return | 308 | `permanent: true` in `redirects()`, or `permanentRedirect()` |
| The route moved for now, and the old address returns | 307 | `permanent: false` in `redirects()`, or `redirect()` |
| The record is gone, and nothing replaces it | 404, or 410 at the reverse proxy | `notFound()` |
| The route was renamed with no redirect | None of the above | Not an option. The redirect belongs in the same change. |

A 308 passes the accumulated signal of the old URL to the new one, and the
browser caches it. A 307 states that the move is temporary, so the search
engine keeps the old URL and consolidates nothing.

Collapse a chain to one hop. Three hops or more spend the crawl budget and lose
signal at each step. After a second migration, point the oldest address at the
newest one directly.

`references/exposed-endpoints-and-destinations.md` owns the rule that no
redirect target comes from request input, and it owns the two tests that a
target passes. That domain holds a veto. This file owns only the status code
and the moment at which the entry ships.

### The filter that must not multiply, and the locale that must not redirect

A filtered list route can produce more URLs than the catalog holds. Name the
combinations worth indexing, and give every other combination a `noindex` value
or a canonical link back to the base route. An uncontrolled set spends the
crawl budget on views that no person searches for.

```ts
// Wrong: proxy.ts sends a reader to a locale that the address decides.
// Failure: a crawler fetches from one country, so it only ever reaches one
// locale. Every other locale stays unindexed, whatever the link set says.
if (request.headers.get("x-client-country") === "IR") {
  return NextResponse.redirect(new URL("/fa", request.url));
}
```

```ts
// Correct: src/proxy.ts reads no address. Each locale keeps its own URL, and
// a banner in the client offers the alternative.
import { NextResponse } from "next/server";

export function proxy() {
  return NextResponse.next();
}
```

Serve one URL for each locale, and let the reader choose. Offer a banner in the
client that suggests another locale, and never a redirect that removes the
choice. The locale router itself is
`references/locale-routing-and-catalogs.md`.

### The files under `.well-known`

Put a file that a standard defines, and that no reader opens in the interface,
under `public/.well-known/`. RFC 9116 defines `security.txt` there, with a
`Contact` field and an `Expires` field, and it requires the media type
`text/plain; charset=utf-8`.

A route handler is the other form, and it takes it where the content changes.
`references/served-content-and-downloads.md` owns the header over a served
file. No package is needed for either form.

### The crawler that trains a model

Control access for a model crawler with a user agent rule in `robots.ts`, the
same way as for any other crawler. The tokens differ for each vendor, and the
list moves. Read the crawler documentation of each vendor before you write a
rule, and never take the token list from memory.

Separate the two purposes before you write the rule. One token fetches a page
because a person asked for it. Another collects text to train a model. A
product can refuse the second and allow the first.

A dedicated text file for model crawlers is not read by the major search
engines. Two large surveys found that almost no crawler ever fetches one.
Write no rule that depends on it, and make no visibility claim from it. Serve
the content in the server response instead, because that is what every crawler
reads.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those facts on 17 August 2026. A cell that holds no
date is a package with a current registry entry on that date. This file does
not state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `sitemap.ts` and `robots.ts`, the file conventions | The sitemap and the robots file, generated from the data. They ship with the framework. | Ships with Next 16.3 | 2026-08-03 | Vercel, active | None |
| Recommend | `generateSitemaps` | Only above 50,000 URLs. It produces the numbered children and the index. | Ships with Next 16.3 | 2026-08-03 | Vercel, active | None |
| Conditional | `next-sitemap` | Only a project that must write the files at build time, or a setup that predates the file conventions. | Community | — | Community, active | Audit at install |
| Reject | A sitemap written as a list in the repository | It is correct on the day of the commit alone. | — | — | — | — |
| Reject | A dedicated text file for model crawlers, as a visibility strategy | The major search engines state that they do not read one. | — | — | — | — |
| Reject | A locale redirect that reads the network address | It traps a crawler in one locale, so every other locale stays unindexed. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| A URL is listed with no title and no description | The path is refused in `robots.txt`, and an external link points at it | Read `robots.txt`, then inspect the URL in the search console | Allow the crawl, and let the route carry `noindex` |
| The `noindex` value never takes effect | The same path is refused in `robots.txt` | The inspection reports that the crawl is not allowed | Remove the `disallow` entry for that path |
| The preview deployment is in the results | No environment gate on the robots file | Read `robots.txt` on the staging origin | Gate on an environment value, and send `X-Robots-Tag: noindex` |
| A missing record answers 200 | `notFound()` runs after the stream begins | Read the status line for a slug that does not exist | Resolve the record above the boundary |
| The sitemap is stale between releases | It is a list that a person wrote | Compare the entry count against the record count | Generate it from the data |
| Every sitemap entry reports the same instant | `lastModified` takes `new Date()` | Read the sitemap | Map the field from the record timestamp |
| The build throws on the sitemap id | The signature predates Next 16 | Read the build error | Await the promise that `props.id` holds |
| A rename lost its traffic | The old address answers 404 | Request the old address and read the status | Add the 308 entry, and ship it with the rename |
| The crawl budget goes to filtered views | The filter set produces unbounded URLs | Read the crawl statistics | Index a named set, and refuse the rest |
| One locale is never indexed | A redirect reads the network address | Request the route with a crawler user agent from another country | Serve one URL for each locale |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next 16.0.0 made the `generateSitemaps` id a promise | `rg -n 'id }: { id: string }' -g '*.ts' src/app` reports a hit | Take `props: { id: Promise<string> }`, and await it |
| Next 16 renamed `middleware.ts` to `proxy.ts` | A `middleware.ts` file holds a redirect or a robots header | Run `npx @next/codemod@latest rename-middleware-to-proxy .` |
| Next 16 route handlers run at request time under Cache Components | A sitemap route with no `"use cache"` helper and no `revalidate` | Add one of the two forms, and read `references/caching-and-revalidation.md` |
| Google no longer supports `rel="prev"` and `rel="next"` | A paginated list that relies on the pair | Give each page a crawlable `<a href>` link to the next one |
| Google clarified in December 2025 that a non-200 page may skip the render | Client code that sets a canonical link or a status after hydration | Handle both on the server |

## Verification

```bash
# 1. Read the installed version before you write code.
node -p "require('next/package.json').version"

# 2. Confirm that both file conventions exist. Each prints the path.
rg --files src/app | rg 'robots\.ts|sitemap\.ts'

# 3. Find a path that both a disallow rule and a noindex value cover. Read the
#    robots rules and the route fields side by side.
rg -n -A12 'MetadataRoute.Robots' src/app/robots.ts
rg -n 'index: false' -g '*.tsx' src/app

# 4. Confirm that the robots file reads an environment value. This prints the
#    line.
rg -n 'SITE_ENV|NODE_ENV' src/app/robots.ts

# 5. Find a sitemap that reads the clock. This prints nothing.
rg -n 'lastModified:\s*new Date\(\)' -g '*.ts' src/app

# 6. Find the pre-16 sitemap signature. This prints nothing.
rg -n 'id }: { id: string }' -g '*.ts' src/app

# 7. Find a sitemap route with neither cache form. Read every hit.
rg -l 'MetadataRoute.Sitemap' -g '*.ts' src/app \
  | xargs rg --files-without-match 'use cache|revalidate'

# 8. Find a locale redirect that reads the network address. This prints
#    nothing.
rg -n 'country|geo|x-vercel-ip' proxy.ts src/proxy.ts

# 9. Read the robots file and the sitemap of the deployed application.
curl -s "$APP_ORIGIN/robots.txt"
curl -s "$APP_ORIGIN/sitemap.xml" | head -50

# 10. Confirm that staging refuses everything, and that it sends the header.
curl -s  "$STAGING_ORIGIN/robots.txt" | rg 'Disallow: /'
curl -sI "$STAGING_ORIGIN/" | rg -i 'x-robots-tag'

# 11. Confirm that a missing record answers 404, and never 200.
curl -sI "$APP_ORIGIN/products/does-not-exist" | head -1

# 12. Confirm that a renamed route answers 308, in one hop.
curl -sI "$APP_ORIGIN/old-slug" | rg -i 'HTTP/|location'
curl -sIL "$APP_ORIGIN/old-slug" | rg -c 'HTTP/'

# 13. Confirm the media type of the security file.
curl -sI "$APP_ORIGIN/.well-known/security.txt" | rg -i 'content-type'

# 14. Compare the sitemap entry count against the published record count in
#     Django. The two agree.
curl -s "$APP_ORIGIN/sitemap.xml" | rg -c '<loc>'

# 15. Read the coverage report of the search console after the release. It
#     holds no new error.
```

## Review checklist

- [ ] Does every route carry a written indexability decision?
- [ ] Is every route behind a session marked `index: false, follow: false`?
- [ ] Is every route behind a session absent from the sitemap?
- [ ] Does no path carry both a `disallow` rule and a `noindex` value?
- [ ] Does `robots.ts` refuse everything outside production, on an environment
      value?
- [ ] Does a non-production deployment send `X-Robots-Tag: noindex`?
- [ ] Does `robots.ts` name the sitemap, with an absolute URL?
- [ ] Is the sitemap generated from the data, rather than written as a list?
- [ ] Does `lastModified` come from a record timestamp, and never from
      `new Date()`?
- [ ] Does the enumeration use cursor pagination above a few thousand rows?
- [ ] Does the sitemap split with `generateSitemaps` above 50,000 URLs?
- [ ] Does the sitemap route declare `"use cache"` on a helper, or a literal
      `revalidate`?
- [ ] Does a missing record answer 404, with the check above every boundary?
- [ ] Does every renamed route answer 308, in one hop?
- [ ] Does the filter set index a named list, and refuse the rest?
- [ ] Is every locale served at its own URL, with no redirect on the network
      address?
- [ ] Does `.well-known/security.txt` answer with `text/plain`?

## Handoffs

- The title, the canonical link, the locale link set, and the share card →
  `references/route-metadata-and-social-cards.md`.
- The JSON-LD block, and the escape that its grammar needs →
  `references/structured-data-and-rich-results.md`.
- `not-found.tsx` as a route file, the file conventions in the route tree,
  `next.config.ts`, and `proxy.ts` →
  `references/app-router-structure.md`. The words inside the view are
  `references/error-and-empty-state-copy.md`.
- The four revalidation APIs, `"use cache"`, `cacheLife`, `cacheTag`, and the
  literal `revalidate` → `references/caching-and-revalidation.md`. The order
  inside a mutation is `references/data-access-and-mutations.md`.
- The rule that no redirect target comes from request input, and the two tests
  over it → `references/exposed-endpoints-and-destinations.md`. That domain
  holds a veto.
- The header set, the layer that emits it, and the policy above it →
  `references/security-headers-and-csp.md`.
- The header over a served file, and the media type on it →
  `references/served-content-and-downloads.md`.
- The gate that puts a route behind a session →
  `references/route-protection-and-permissions.md`. That domain holds a veto.
- The parse over `process.env`, and the module that holds it →
  `references/boundary-validation-and-api-types.md`. The generated types over
  the records are `references/openapi-schema-and-codegen.md`.
- The cursor page that a list request takes, and the client over it →
  `references/api-client-and-request-safety.md`.
- The crawl budget as a cost, and the first byte that a sitemap render pays →
  `references/performance-budgets-and-measurement.md`.
- The locale router, the locale segment, and the file that holds the catalog →
  `references/locale-routing-and-catalogs.md`. The key inside that file →
  `references/message-catalog-and-plurals.md`.
- The reverse proxy that serves a static file, and the 410 status at that layer
  → `references/runtime-process-and-reverse-proxy.md`. The environment value
  that a pipeline supplies to a non-production deployment →
  `references/release-pipeline-and-rollback.md`.
- The slow list endpoint behind an enumeration, the index on the ordering
  field, the serializer field, and the schema → the backend. The permission
  check over the enumerated rows → `secure-code-auditor`.
