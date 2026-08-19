# Structured data and rich results

`schema-dts` 2.0.0, Next.js 16.3, React 19.2.6 or later, TypeScript 5.9. This
file owns the claim that a page makes about its content in machine-readable
form. The subjects are the JSON-LD block, the escape that the grammar of its
host element needs, and the type that the compiler checks. They also include
the rule that the claim matches what the reader sees, and the types that the
search engine has retired.

The title, the canonical link, and the card that a share preview renders are
`references/route-metadata-and-social-cards.md`. Whether a crawler may index
the page at all is `references/crawl-and-index-control.md`.

## Principle

Structured data is a claim to a machine about a page that the machine cannot
read for itself. A claim that the page does not support is a false statement
with a penalty behind it.

A JSON document inside a script element is not a JSON document to the parser
that reads it first. The grammar of the host element decides the escape, and
the JSON serializer knows nothing about that grammar.

A vocabulary is not a rank. It is eligibility for a display surface, and the
search engine can withdraw that surface at any time.

Values that a backend stores reach this block unchanged. Treat every one of
them as untrusted output, because a script element is the strongest sink in the
document.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The block, and the escape that its grammar needs

```tsx
// Wrong: the serialized JSON goes into the element with no escape.
// Failure: a field value that holds the characters of a closing script tag
// ends the element early. Everything after it becomes markup, so a value that
// a person typed into a Django admin form runs as script in every visit.
<script
  type="application/ld+json"
  dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
/>
```

```tsx
// Correct: src/components/seo/product-json-ld.tsx
import type { Product, WithContext } from "schema-dts";
import { SCHEMA_CONTEXT } from "@/lib/seo/context";

export function ProductJsonLd({ product }: { product: ProductRecord }) {
  const jsonLd: WithContext<Product> = {
    "@context": SCHEMA_CONTEXT,
    "@type": "Product",
    name: product.title,
    description: product.summary,
    image: product.cardImage,
  };
  return (
    <script
      type="application/ld+json"
      dangerouslySetInnerHTML={{
        __html: JSON.stringify(jsonLd).replace(/</g, "\\u003c"),
      }}
    />
  );
}
```

`SCHEMA_CONTEXT` holds the schema.org vocabulary address on the secure scheme.
`WithContext<T>` needs that exact literal, so declare it one time with
`as const` and import it.

`JSON.stringify` produces valid JSON. It does not produce a safe script body,
and it sanitises nothing. Replace every `<` with the escape `\u003c`. The JSON
parser inside the browser turns the escape back into `<`, so the value that the
consumer reads is unchanged.

Escape at the point of render, and never on write. A second write path and
every older record reach the document uncleaned.
`references/untrusted-markup-and-injection.md` owns that rule, and it owns
every other sink that `dangerouslySetInnerHTML` opens. That domain holds a
veto.

A policy that names `object-src 'none'` and `base-uri 'self'` does not stop
this failure. A JSON-LD block is inline content that the page itself wrote, so
a nonce admits it. `references/security-headers-and-csp.md` owns the policy.

### The compiler checks the type

```tsx
// Wrong: a hand-written object with no type over it.
// Failure: "@type": "Prodct" and a misspelled field both compile. The block
// reaches production, the search engine ignores it, and no test fails.
const jsonLd = {
  "@context": SCHEMA_CONTEXT,
  "@type": "Prodct",
  productName: product.title,
};
```

```tsx
// Correct: WithContext<T> names the type, and the compiler rejects a field
// that the vocabulary does not carry.
const jsonLd: WithContext<Product> = {
  "@context": SCHEMA_CONTEXT,
  "@type": "Product",
  name: product.title,
};
```

`schema-dts` supplies a TypeScript type for each schema.org type. It costs no
run-time bytes, because the types disappear at the build. Google publishes the
package, and Google states that it is not a supported Google product.

`references/type-modeling-and-narrowing.md` owns the rule that a value carries
a type rather than a shape. This block is one more value under it.

### The claim matches the page

```tsx
// Wrong: the block states a rating that the page never renders.
// Failure: the markup and the document disagree. The search engine can apply
// a manual action against the whole site, and the rich result disappears for
// every page on it.
const jsonLd: WithContext<Product> = {
  "@context": SCHEMA_CONTEXT,
  "@type": "Product",
  name: product.title,
  aggregateRating: { "@type": "AggregateRating", ratingValue: "4.9", reviewCount: "120" },
};
```

```tsx
// Correct: the block reads the same record that the markup renders, and it
// states the rating only where the page shows it.
const jsonLd: WithContext<Product> = {
  "@context": SCHEMA_CONTEXT,
  "@type": "Product",
  name: product.title,
  ...(product.rating
    ? {
        aggregateRating: {
          "@type": "AggregateRating",
          ratingValue: String(product.rating.value),
          reviewCount: String(product.rating.count),
        },
      }
    : {}),
};
```

One record feeds the block and the markup. A block that reads a second source
drifts from the page on the first backend change.

### Where the block goes

| The claim | The route that holds it | The rule |
| --- | --- | --- |
| `Organization` and `WebSite` | The root layout | One block for the whole site. A second one gives the site two identities. |
| `BreadcrumbList` | The layout of the section | The list matches the breadcrumb that the reader sees. |
| An entity, such as `Product` or `Article` | The route that renders that entity | One block for one entity. |

The block renders on the server. NEVER build it inside an effect, and never
inside a client component that a crawler reaches after hydration. A crawler
that runs no JavaScript sees nothing that an effect wrote.

One page states one graph. Where a page needs two related types, give the block
a `@graph` array. Give each node an `@id`, so the two nodes reference each other
rather than stand apart.

### The types worth writing, and the types that are gone

| The type | The status | The rule |
| --- | --- | --- |
| `Organization`, `WebSite` | Current | One block in the root layout. |
| `BreadcrumbList` | Current | Where the reader sees a breadcrumb. |
| `Product`, `Article`, `Event`, `Recipe`, `JobPosting`, `LocalBusiness` | Current | Only on a route that renders that entity. |
| `FAQPage` | Google ended the rich result on 7 May 2026 | The markup stays valid schema.org, and it costs nothing to keep. Write no new one for the search surface. |
| `HowTo` | Google removed the rich result in 2023, from every surface | Write none. |
| `Book Actions`, `Course Info`, `Claim Review`, `Estimated Salary`, `Learning Video`, `Special Announcement`, `Vehicle Listing` | Google retired all seven in June 2025 | Write none for the search surface. |

A retired type is not an error. It is effort with no return. Delete a retired
block only where the deletion is the subject of the change.
`references/dependencies-and-git-workflow.md` owns the rule against a change
that the request did not ask for.

Read the current list before you write a new type. The list moves, and this
table states the state on 17 August 2026.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those facts on 17 August 2026. A cell that holds no
date is a package with a current registry entry on that date. This file does
not state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `schema-dts` | The TypeScript type for each schema.org type. It costs no run-time bytes. | 2.0.0 | 2026-04 | Google open source, active. Google states that it is not a supported Google product. | None |
| Conditional | `serialize-javascript` | Only where a block carries deeply nested values from several sources, and one escape call is hard to audit. | Audit at install | — | Yahoo, active | Audit at install |
| Reject | `JSON.stringify` with no escape after it | It produces valid JSON and an unsafe script body. | — | — | — | — |
| Reject | A JSON-LD block built inside an effect | A crawler that runs no JavaScript sees nothing. | — | — | — | — |
| Reject | A new `HowTo` or `FAQPage` block for the search surface | Google ended both rich results. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| A stored value runs as script | `JSON.stringify` with no escape after it | Read the raw response for a block whose element ends early | Replace every `<` with `\u003c` |
| The block is absent from the raw response | It renders inside a client component or an effect | Compare the raw response against the hydrated document | Render the block on the server |
| The rich result never appears | The type name or a field name is misspelled | Run the block through the structured data test | Type the object with `WithContext<T>` |
| The rich result disappears for the whole site | The markup claims what the page does not render | Read the search console messages | Delete every claim that the page does not support |
| Two blocks describe one entity | A layout and a page each render one | Count the blocks in the raw response | Keep one, or join them with `@graph` and an `@id` |
| The block is valid and no surface shows it | The type is retired | Compare the type against the current list | Delete the block, or accept that it earns nothing |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Google ended the `FAQPage` rich result on 7 May 2026 | `rg -n 'FAQPage' -g '*.tsx' src/` reports a hit | Keep the markup, and plan no new one |
| Google removed the `HowTo` rich result in 2023 | `rg -n 'HowTo' -g '*.tsx' src/` reports a hit | Write none for new work |
| Google retired seven further types in June 2025 | A block that names one of the seven | Write none for new work |
| `schema-dts` 2.0.0 | A hand-written object with no type over it | Take `WithContext<T>` from the package |

## Verification

```bash
# 1. Read the installed version before you write code.
node -p "require('schema-dts/package.json').version"

# 2. Find every JSON-LD block. Read every hit.
rg -n -B4 'application/ld\+json' -g '*.tsx' src/

# 3. Find a block that serializes with no escape after it. This prints
#    nothing.
rg -l 'application/ld\+json' -g '*.tsx' src/ \
  | xargs rg -n 'JSON.stringify' | rg -v 'replace\(/</g'

# 4. Find a block in a client file. This prints nothing.
rg -l 'application/ld\+json' -g '*.tsx' src/ | xargs rg -l '"use client"'

# 5. Find a block that carries no schema-dts type. Read every hit.
rg -l 'application/ld\+json' -g '*.tsx' src/ \
  | xargs rg --files-without-match 'WithContext'

# 6. Find a retired type. Read every hit.
rg -n 'HowTo|FAQPage|ClaimReview|SpecialAnnouncement|VehicleListing' -g '*.tsx' src/

# 7. Confirm that the block is in the raw response, and that it appears one
#    time for one entity.
curl -s "$APP_ORIGIN/products/123" | rg -c 'application/ld\+json'

# 8. Read the block out of the raw response. It parses as JSON, and the
#    element ends where the block ends.
curl -s "$APP_ORIGIN/products/123" | rg -o 'application/ld\+json.{0,400}'

# 9. Save a record whose text field holds the characters of a closing script
#    tag. Load the route, and confirm that the element still ends where the
#    block ends.

# 10. Run each block through the structured data test of the search engine. It
#     reports no error.

# 11. Compare each claim in the block against the rendered page. Every value
#     that the block states is visible to the reader.
```

## Review checklist

- [ ] Does every JSON-LD block replace `<` with `\u003c` before it renders?
- [ ] Does every block carry a `schema-dts` type, rather than a bare object?
- [ ] Does every block render on the server?
- [ ] Does the block read the same record that the markup renders?
- [ ] Is every claim in the block visible to the reader on that page?
- [ ] Does one entity carry one block?
- [ ] Do `Organization` and `WebSite` appear once, in the root layout?
- [ ] Does a `BreadcrumbList` match the breadcrumb that the page shows?
- [ ] Is every type in the block still supported by the search engine?
- [ ] Did the structured data test pass for each type?
- [ ] Does a stored value that holds script characters leave the element
      intact?

## Handoffs

- Every other sink that `dangerouslySetInnerHTML` opens, the sanitiser in front
  of it, and the rule that the escape happens at render →
  `references/untrusted-markup-and-injection.md`. That domain holds a veto.
- The policy that admits an inline block, the nonce, and `object-src` →
  `references/security-headers-and-csp.md`.
- The title, the canonical link, the locale set, and the share card →
  `references/route-metadata-and-social-cards.md`.
- Whether the page is indexable at all, and the sitemap that lists it →
  `references/crawl-and-index-control.md`.
- The type over a value, and the rule against a hand-copied shape →
  `references/type-modeling-and-narrowing.md`. The parse over the record itself
  is `references/boundary-validation-and-api-types.md`.
- The `"use client"` directive, and the reason that a crawler sees nothing
  behind it → `references/server-and-client-components.md`.
- The breadcrumb component, its markup, and its accessible name →
  `references/semantics-and-accessible-names.md`.
- The rule against a change that the request did not ask for →
  `references/dependencies-and-git-workflow.md`.
- The serializer that supplies the record, and the field that a rename breaks →
  `references/openapi-schema-and-codegen.md`. The server side of that contract
  is the backend.
