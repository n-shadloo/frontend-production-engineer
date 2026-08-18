# Route metadata and social cards

Next.js 16.3, React 19.2.6 or later, TypeScript 5.9, against a Django and DRF
backend. This file owns what one route declares about itself. The subjects are
the two metadata exports, `metadataBase` with the absolute URL that it
resolves, and the title template. They also include the canonical link, the
reciprocal `hreflang` set, and the Open Graph card that a share preview
renders. The last subjects are the card image that a route builds for one
record, and the metadata that streams. A user agent that runs no JavaScript
never receives that stream.

The machine-readable claim about the content of a page is
`references/structured-data-and-rich-results.md`. Whether a crawler may reach
the route, and whether it may index it, is
`references/crawl-and-index-control.md`.

## Principle

A page that declares nothing about itself is described by a machine that
guesses. The guess reaches a search result, a share card, and a browser tab.

Two pages with one title are one page to a search engine. The title is the
strongest signal that a route controls.

A relative URL means nothing outside the document that holds it. Another origin
reads a share card, so every URL in the metadata is absolute.

Metadata that only the browser builds reaches no user agent that runs no
JavaScript. A social crawler is one such agent.

One system owns one tag. Two systems that each emit a `<title>` emit two of
them, and the crawler takes the first.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The two metadata exports

| The condition | The export | The forcing function |
| --- | --- | --- |
| No field depends on a param or on a fetch | `export const metadata` | The value is static. The build reads it once and puts it in the prerendered shell. |
| A field depends on `params`, on `searchParams`, or on the backend | `export async function generateMetadata` | The value cannot exist before the request. |
| Both are present in one file | Neither. Next.js refuses the pair. | One route file exports one form. |

Take the static export where every field is static. It is shorter to read, and
the build can analyze it.

`generateMetadata` runs on the server alone. NEVER put it in a file that
carries `"use client"`. `references/server-and-client-components.md` owns the
directive.

A file-based convention beats the export. Where `opengraph-image.tsx` and an
`openGraph.images` field both exist for one route, the file wins.

### `metadataBase` is set once, in the root layout

```tsx
// Wrong: src/app/layout.tsx names a relative image and no base.
// Failure: Next 16.3 fails the build for a relative URL in a metadata field
// with no metadataBase above it. Where an older release accepted it, the URL
// resolved against the development origin, and every share card was blank.
import type { Metadata } from "next";

export const metadata: Metadata = {
  openGraph: { images: "/og.png" },
};
```

```tsx
// Correct: one absolute base. Every child route then writes a path.
import type { Metadata } from "next";
import { env } from "@/env";

export const metadata: Metadata = {
  metadataBase: new URL(env.NEXT_PUBLIC_SITE_ORIGIN),
  title: { default: "Acme", template: "%s | Acme" },
  description: "The one sentence that a search result shows.",
  openGraph: {
    siteName: "Acme",
    type: "website",
    locale: "en_US",
    images: "/og.png",
  },
};
```

The origin comes from a parsed environment module, and never from a literal.
The production origin, the staging origin, and the development origin differ.
`references/boundary-validation-and-api-types.md` owns the parse over
`process.env`.

Set the base one time. A second base in a child layout gives one asset two
absolute URLs, and the two disagree.

### `generateMetadata` reads what the page reads

```tsx
// Wrong: src/app/products/[id]/page.tsx builds its own call for the metadata.
// Failure: one render sends two requests to Django for one record. The load
// on the endpoint doubles, and the two answers can disagree.
import { fetchProductWithFreshClient } from "@/lib/products-client";

export async function generateMetadata(props: PageProps<'/products/[id]'>) {
  const { id } = await props.params;
  const product = await fetchProductWithFreshClient(id);
  return { title: product.title };
}
```

```tsx
// Correct: src/lib/dal/products.ts. One memoised read serves both.
import "server-only";
import { cache } from "react";
import { ProductSchema } from "@/lib/schemas/product";

export const getProduct = cache(async (id: string) => {
  const response = await fetch(`${process.env.DJANGO_URL}/api/products/${id}/`);
  if (!response.ok) return null;
  return ProductSchema.parse(await response.json());
});
```

```tsx
// Correct: src/app/products/[id]/page.tsx. The metadata and the page share
// one call, and a missing record takes the same branch in both.
import type { Metadata } from "next";
import { notFound } from "next/navigation";
import { getProduct } from "@/lib/dal/products";

export async function generateMetadata(
  props: PageProps<'/products/[id]'>,
): Promise<Metadata> {
  const { id } = await props.params;
  const product = await getProduct(id);
  if (!product) return { title: "Product not found", robots: { index: false } };
  return {
    title: product.title,
    description: product.summary,
    alternates: { canonical: `/products/${id}` },
    openGraph: { title: product.title, images: [product.cardImage] },
  };
}

export default async function Page(props: PageProps<'/products/[id]'>) {
  const { id } = await props.params;
  const product = await getProduct(id);
  if (!product) notFound();
  return <h1>{product.title}</h1>;
}
```

Next.js memoises `fetch` across `generateMetadata`, `generateStaticParams`, a
layout, and a page in one render. A function that calls something other than
`fetch` needs React `cache` around it, or the request runs twice.

`references/data-access-and-mutations.md` owns the data access layer.
`generateMetadata` calls that layer. It never opens a second path to Django.

### The title template, and the leaf that overrides it

| The key | What it does | Use it for |
| --- | --- | --- |
| `title.default` | The title of a route that states none | The root layout alone |
| `title.template` | A pattern that a child title fills, such as `%s \| Acme` | The root layout, and a section layout that needs its own suffix |
| `title.absolute` | A title that ignores every template above it | A landing page whose title must carry no suffix |

A leaf route that sets no title takes the `default` of its parent. Two leaf
routes that both take it share one title, which makes them one page to a search
engine. Give every public route its own title.

The words in a title are `references/interface-copy-and-voice.md`. This file
owns the presence of the field, and the rule that no two public routes share
its value.

### One canonical, absolute and self-referencing

```tsx
// Wrong: the canonical of every filtered view points at the base route.
// Failure: the base route is the canonical of a view whose content differs
// from it, so the search engine drops the view and shows the base instead.
alternates: { canonical: "/products" },
```

```tsx
// Correct: the canonical names the route that renders this content.
alternates: { canonical: `/products/${id}` },
```

A canonical link states which URL holds this content. Write one for each route,
and point it at the route itself.

Three rules bind the target. It resolves to a 200 response. It carries no
`noindex`. It is not a redirect. A canonical that breaks one of them is a
signal that the search engine drops.

### Every locale variant links back

```tsx
// Correct: the reciprocal set, the self-reference, and the default.
import type { Metadata } from "next";

export const metadata: Metadata = {
  alternates: {
    canonical: "/fa/about",
    languages: {
      "en-US": "/en/about",
      "fa-IR": "/fa/about",
      "x-default": "/about",
    },
  },
};
```

Each locale variant lists every other variant, and lists itself. A set where A
names B and B does not name A is asymmetric, and the search engine ignores the
whole cluster.

`x-default` names the URL for a reader whose locale matches no variant.

Every variant in the set must be indexable. One `noindex` variant breaks the
cluster for all of them.

The locale segment, and the router that produces it, are
`references/locale-routing-and-catalogs.md`. The `lang` attribute is
`references/semantics-and-accessible-names.md`, and the `dir` attribute on the
document is `references/bidirectional-layout-and-scripts.md`. This file owns
the link set alone.

### One asset serves every card

| The field | The value to ship |
| --- | --- |
| `openGraph.images` | One asset at 1200 × 630 pixels, in PNG or JPEG, under 5 MB |
| `openGraph.title` and `openGraph.description` | The title and the description of the route, which may be shorter than the search result versions |
| `openGraph.type` and `openGraph.siteName` | `website` or `article`, and the product name, from the root layout |
| `twitter.card` | `summary_large_image` |
| `twitter.images` | Absent. The platform reads `openGraph.images` where the field is absent. |
| The alternative text of the card | `openGraph.images[].alt`, because a card reaches a screen reader on some platforms |

X states a ratio of 2 to 1 for the large card, and the Open Graph convention is
1.91 to 1. One asset at 1200 × 630 satisfies both. X reads the Open Graph value
where no `twitter:` field exists, and it crops the difference. Below 300 × 157
X drops to the small card, and it reports nothing.

Two assets for one card is the failure that this rule prevents. The second
asset goes stale, and no test reads either one.

### The card image that a route builds for one record

```tsx
// Wrong: src/app/products/[id]/opengraph-image.tsx.
// Failure: Satori supports flexbox alone. A container with two children and
// no display value throws at request time, so the route serves no image and
// every share card falls back to the site-wide asset.
import { ImageResponse } from "next/og";

export default async function Image() {
  return new ImageResponse(
    <div>
      <h1>Acme</h1>
      <p>A product</p>
    </div>,
  );
}
```

```tsx
// Correct: src/app/products/[id]/opengraph-image.tsx.
import { ImageResponse } from "next/og";
import { getProduct } from "@/lib/dal/products";

export const size = { width: 1200, height: 630 };
export const contentType = "image/png";
export const alt = "The product name over the brand color";

export default async function Image(props: { params: Promise<{ id: string }> }) {
  const { id } = await props.params;
  const product = await getProduct(id);
  return new ImageResponse(
    (
      <div
        style={{
          display: "flex",
          width: "100%",
          height: "100%",
          alignItems: "center",
          justifyContent: "center",
          background: "#0b1120",
          color: "#f8fafc",
          fontSize: 64,
        }}
      >
        {product?.title ?? "Acme"}
      </div>
    ),
    size,
  );
}
```

Take the file convention where the image changes for each record. Take a static
`opengraph-image.png` where it never changes, because a static file costs no
render.

Three limits bind `ImageResponse`. It renders flexbox alone. The whole route,
with every font and image that it embeds, stays under 500 kB. The image is not
the largest paint of any page, because no reader ever loads it in the document.
`references/paint-and-interaction-cost.md` owns the largest paint, and this
route is outside it.

`references/app-router-structure.md` owns the position of a file convention in
the route tree. This file owns what the file returns.

### The viewport export is not the metadata export

`viewport` and `generateViewport` are separate exports. `themeColor`,
`colorScheme`, `width`, `initialScale`, and `userScalable` belong to them, and
never to `metadata`.

`references/visual-and-motor-criteria.md` owns the values in that export,
because the zoom keys carry a WCAG criterion, and that domain holds a veto.
This file owns only the rule that the two exports stay apart.

### React hoists a tag, and it removes no duplicate

```tsx
// Wrong: a component renders its own title beside the metadata export.
// Failure: React 19 hoists the tag into <head> and deduplicates nothing. The
// document holds two <title> elements. The crawler reads the first, which is
// the one that the component did not intend to be read.
export function ProductHeader({ name }: { name: string }) {
  return (
    <>
      <title>{name}</title>
      <h1>{name}</h1>
    </>
  );
}
```

```tsx
// Correct: the Metadata API owns every document-level tag, and the component
// renders only what the reader sees.
export function ProductHeader({ name }: { name: string }) {
  return <h1>{name}</h1>;
}
```

React 19 renders `<title>`, `<meta>`, and `<link>` from inside a component and
lifts each one into `<head>`. `references/suspense-and-actions.md` records that
the API exists. Use it for a tag that no route owns, such as a `<link>` that a
third-party widget needs. NEVER use it for the title, the description, the
canonical, or a card field.

### Metadata that streams, and the bot that reads no stream

Next.js streams the metadata to a browser, so the page shell paints before the
metadata resolves. For a user agent in the `htmlLimitedBots` list it blocks the
response instead, and it sends a complete `<head>`.

```ts
// Wrong: next.config.ts widens the list to every user agent.
// Failure: every response now waits for the metadata, so the first byte rises
// for every reader, and a partially prerendered route can lose its cached
// shell.
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  htmlLimitedBots: /.*/,
};

export default nextConfig;
```

```ts
// Correct: next.config.ts states no override. The default list covers the
// crawlers that a product normally needs.
import type { NextConfig } from "next";

const nextConfig: NextConfig = {};

export default nextConfig;
```

Write an override only where one named crawler is absent from the default list
and its card is broken. Read the default list in the documentation of the
installed version first. Confirm there whether the value that you write extends
that list or replaces it, because the two behaviors need different values.

CAUTION: an override that matches every agent has cost a partially prerendered
route its shared cache in a reported Next.js defect. Read the release notes of
the installed version before you combine an override with Partial Prerendering.
`references/caching-and-revalidation.md` owns the prerender.

Where a card is blank on one platform and correct on another, read the raw
response before you read the browser tools. A tool that shows the hydrated
document hides this failure completely.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those facts on 17 August 2026. A cell that holds no
date is a package with a current registry entry on that date. This file does
not state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | The Next.js Metadata API | Every document-level tag of every route. It ships with the framework. | Ships with Next 16.3 | 2026-08-03 | Vercel, active | None |
| Recommend | `next/og`, the `ImageResponse` class | The card image that a route builds for one record. It ships with the framework. | Ships with Next 16.3 | 2026-08-03 | Vercel, active | None |
| Conditional | The React 19 hoisting of `<title>`, `<meta>`, and `<link>` | Only a tag that no route owns, such as one that a third-party widget needs. It deduplicates nothing. | React 19.2.6 or later | Current | Meta, active | None |
| Reject | `next/head` | The Pages Router API. It does nothing in the App Router. | — | — | — | — |
| Reject | `react-helmet` and every head manager | The Metadata API and the React hoisting supersede it. Two systems for one tag emit the tag twice. | — | — | — | — |
| Reject | A second card asset beside the 1200 × 630 one | One asset satisfies every platform. The second goes stale, and no test reads it. | — | — | — | — |
| Reject | The `keywords` metadata field | The search engines stopped reading it long ago. It costs effort and returns nothing. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| Two public routes carry one title | A leaf route sets no title and takes the parent `default` | Read the title of each route in the raw response | Give the leaf its own `title`, under a `template` in the layout |
| The build fails on a metadata field | A relative URL with no `metadataBase` above it | Read the build error, which names the field | Set `metadataBase` once in the root layout |
| The card is blank on one platform and correct on another | The metadata streamed, and that platform is not in `htmlLimitedBots` | Read the raw response, never the hydrated document | Confirm the user agent against the default list, and add the one token that it omits |
| The endpoint carries twice the load of the render count | `generateMetadata` opens its own path to Django | Count the requests for one route render | Call the memoised read that the page calls |
| The document holds two `<title>` elements | A component renders a title beside the Metadata API | Read the raw response for a second `<title>` | Delete the component tag |
| The card image is missing for every record | The `ImageResponse` container has no `display` value | Request the image route directly, and read the error | Give every container `display: "flex"` |
| A locale cluster is ignored | One variant is `noindex`, or one link is missing | Read the link set of each variant | Emit the reciprocal set, the self-reference, and `x-default` |
| The card shows the old title after an edit | The platform cached the card | Compare the raw response against the card | Wait for the platform cache, or change the image path |

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next 14 moved `viewport` and `themeColor` out of `metadata` | `rg -n 'themeColor' -g '*.tsx' src/app` reports a hit inside a `metadata` object | Run `npx @next/codemod@latest metadata-to-viewport-export .` |
| Next 15 and 16 made `params` and `searchParams` promises | `rg -n 'params\.' -g '*.tsx' src/app` reports a read with no `await` | Await the promise. `references/app-router-structure.md` owns the rename |
| Next 16 broadened the default `htmlLimitedBots` list | A card broke after an upgrade from 15 | Confirm the user agent against the default list before you override it |
| Next 16 renamed `middleware.ts` to `proxy.ts` | A `middleware.ts` file is present, so a metadata header never runs | Run `npx @next/codemod@latest rename-middleware-to-proxy .` |
| React 19 hoists `<title>` and `<meta>` from a component | A component that renders a document-level tag | Move the value into the Metadata API |

## Verification

```bash
# 1. Read the installed version before you write code.
node -p "require('next/package.json').version"

# 2. Find every route file that exports neither metadata form. Read every hit.
rg --files-without-match 'export const metadata|generateMetadata' \
  -g 'src/app/**/page.tsx'

# 3. Confirm that the root layout sets the base. This prints the line.
rg -n 'metadataBase' src/app/layout.tsx

# 4. Find a second base under a child route. This prints nothing.
rg -n 'metadataBase' -g '!src/app/layout.tsx' -g 'src/app/**/*.tsx'

# 5. Find a component that renders a document-level tag. This prints nothing.
rg -n '<title>|<meta |<link rel="canonical"' -g '*.tsx' src/components src/features

# 6. Find a keywords field. This prints nothing.
rg -n 'keywords:' -g '*.tsx' src/app

# 7. Find a themeColor inside a metadata object. This prints nothing.
rg -n -B6 'themeColor' -g '*.tsx' src/app | rg 'const metadata'

# 8. Find a generateMetadata in a client file. This prints nothing.
rg -l 'generateMetadata' -g '*.tsx' src/app | xargs rg -l '"use client"'

# 9. Find an override of the bot list. Read every hit.
rg -n 'htmlLimitedBots' next.config.ts

# 10. Read the raw response of a route. The head holds the title, the
#    description, the canonical, and the card fields.
curl -s "$APP_ORIGIN/products/123" \
  | rg -o '<title>[^<]*</title>|rel="canonical"|property="og:[a-z:]+"'

# 11. Request the card image of that route. It answers 200 with an image type.
curl -sI "$APP_ORIGIN/products/123/opengraph-image" \
  | rg -i 'HTTP/|content-type'

# 12. Collect the title of every public route from the raw responses. Sort
#     the list, and read it for a duplicate. No two public routes share one.

# 13. Paste the route into the composer of each platform that the product
#     shares to. The card renders the intended title, description, and image.
```

## Review checklist

- [ ] Does every public route export `metadata` or `generateMetadata`?
- [ ] Does every public route carry its own title?
- [ ] Is `metadataBase` set once, in the root layout?
- [ ] Does the base come from a parsed environment value, and not from a
      literal?
- [ ] Does `generateMetadata` call the same memoised read as the page?
- [ ] Does each route carry one canonical, which points at itself?
- [ ] Does the canonical target answer 200, carry no `noindex`, and perform no
      redirect?
- [ ] Does every locale variant list every other variant, itself, and
      `x-default`?
- [ ] Is every locale variant in the set indexable?
- [ ] Does one asset at 1200 × 630 serve the Open Graph and the `twitter`
      fields?
- [ ] Does the card asset carry alternative text?
- [ ] Does every `ImageResponse` container declare `display: "flex"`?
- [ ] Are `themeColor` and the zoom keys in the `viewport` export?
- [ ] Is the `keywords` field absent from every route?
- [ ] Is `<title>`, `<meta>`, and `<link rel="canonical">` absent from every
      component?
- [ ] Does `htmlLimitedBots` keep the default list, plus at most a named token?
- [ ] Was the raw response read, rather than the hydrated document?

## Handoffs

- The claim that a page makes in machine-readable form, and the escape that its
  grammar needs → `references/structured-data-and-rich-results.md`.
- Whether a crawler may reach the route, the sitemap that lists it, and the
  status code that a missing record returns →
  `references/crawl-and-index-control.md`.
- The route files, the file conventions in the route tree, `next.config.ts`,
  and the rename to `proxy.ts` → `references/app-router-structure.md`.
- The `"use client"` directive, and the rule that `generateMetadata` never sits
  behind it → `references/server-and-client-components.md`.
- The data access layer that `generateMetadata` calls, and the memoisation
  over it → `references/data-access-and-mutations.md`. The prerender and the cached
  shell are `references/caching-and-revalidation.md`.
- The parse over `process.env`, and the module that holds it →
  `references/boundary-validation-and-api-types.md`.
- The React hoisting of a document-level tag, and the resource preload beside it
  → `references/suspense-and-actions.md`.
- The values in the `viewport` export, the zoom criterion, and the contrast of a
  theme color → `references/visual-and-motor-criteria.md`. That domain holds a
  veto.
- The `lang` and the `dir` attributes on the document, and the alternative text
  of an image → `references/semantics-and-accessible-names.md`.
- The words in a title and in a description →
  `references/interface-copy-and-voice.md`.
- The props of `<Image>`, the source set, and the element that paints the
  largest area → `references/image-and-video-delivery.md` and
  `references/paint-and-interaction-cost.md`. A card image is in neither,
  because no reader loads it in the document.
- The locale segment, the router that produces it, and the file that holds the
  catalog → `references/locale-routing-and-catalogs.md`. The `dir` attribute on
  the document → `references/bidirectional-layout-and-scripts.md`.
- The serializer field that `generateMetadata` consumes, and the schema that
  types it → `references/openapi-schema-and-codegen.md`. The server side of
  that contract is the sibling skill `django-api-contract`.
