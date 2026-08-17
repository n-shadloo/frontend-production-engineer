# Client bundle and third-party scripts

Next.js 16.3, React 19.2.6 or later, `@next/third-parties`, TypeScript 5.9. This
file owns the JavaScript that reaches the browser, and the moment at which it
runs. The subjects are the boundary that decides what ships, the widget that
arrives on demand, and the import shape. They also include the third-party
script with its load strategy, the facade in front of an embed, and the origin
hint. The last subject is the route prefetch.

The thresholds, the budget, and the measurement loop are
`references/performance-budgets-and-measurement.md`. The largest paint, the
layout shift, and the answer to a tap are
`references/paint-and-interaction-cost.md`.

## Principle

Every byte of JavaScript is downloaded, parsed, compiled, and executed. The last
three costs land on the main thread of a phone.

The render boundary is the largest lever. A component that ships nothing costs
nothing, and no later optimisation matches that.

A byte that one branch of one route needs still costs every reader of that
route, unless a separate request carries it.

A third-party script runs on the same main thread as the application. Nobody on
the team wrote it, nobody on the team reviews it, and it changes without a
release.

The fastest library is the one that the platform already ships.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The boundary is the first lever

```tsx
// Wrong: the page is a client component, so it can hold one button.
// Failure: the page, the chart library, and every import below them ship to the
// browser. First Load JS rises by the whole subtree, and the server data fetch
// is no longer available to the page.
"use client";
import { HugeChart } from "huge-chart-lib";

export default function DashboardPage({ data }: { data: Series[] }) {
  return (
    <div>
      <HugeChart data={data} />
      <button onClick={() => print()}>Print</button>
    </div>
  );
}
```

```tsx
// Correct: the page stays a Server Component, and only the island is client.
// app/dashboard/page.tsx
import { ChartIsland } from "@/features/dashboard/chart-island";

export default async function DashboardPage() {
  const data = await getSeries(); // server fetch, no client cost
  return <ChartIsland data={data} />;
}
```

Prove that a component needs the client before you split it. A transform with no
event handler, no state, and no browser API runs on the server. Markdown, syntax
highlight, and a chart that renders to SVG are all such transforms.

`references/server-and-client-components.md` owns the directive itself and the
rule that it sits on the leaf. This file states the cost that the directive
carries.

### The heavy widget arrives on demand

| The condition | The decision | What forces the change |
| --- | --- | --- |
| The component runs on the client, weighs about 30 kB or more, and sits below the fold or behind an interaction | `next/dynamic` with a `loading` skeleton of the final size | The decision flips where the component holds the element that paints the largest area. A deferred import then delays that paint. |
| The component runs on the client, weighs about 30 kB or more, and sits above the fold | `next/dynamic` with the server render kept, so the chunk splits and the markup still arrives | `ssr: false` above the fold removes the markup from the first document, and the largest paint waits for the JavaScript. |
| The component holds no state, no event handler, and no browser API | No split. Keep it on the server. | The component gains one event handler, and the first row then applies. |

```tsx
// Wrong: the editor is imported at the top of a shared layout.
// Failure: every route under that layout downloads the editor, including the
// routes that never render it.
import { RichTextEditor } from "rich-text-editor";
```

```tsx
// Correct: the editor loads when the branch that needs it renders.
"use client"; // it holds the editor state

import dynamic from "next/dynamic";

const RichTextEditor = dynamic(
  () => import("rich-text-editor").then((m) => m.RichTextEditor),
  { loading: () => <div className="h-64 w-full animate-pulse rounded-md bg-muted" /> },
);
```

CAUTION: give the skeleton the size of the component that replaces it. A
skeleton of the wrong size trades a byte cost for a layout shift, which
`references/paint-and-interaction-cost.md` fails.

`references/charts-and-visual-encoding.md` owns the chart island and the
`ssr: false` rule for a library that paints on a canvas.

### The import shape decides the bytes

```ts
// Wrong: the whole library graph enters the bundle.
// Failure: the default import defeats tree-shaking, so a project that needs one
// function ships every function. `moment` adds 290.4 kB before compression, and
// 72.1 kB after it, for a date that the platform can format.
import _ from "lodash";
import moment from "moment";
```

```ts
// Correct: the platform first, then a named import from an ESM package.
const dateFormat = new Intl.DateTimeFormat("en-GB", { dateStyle: "medium" });
import debounce from "lodash-es/debounce";
```

Reach for `Intl.DateTimeFormat` and `Intl.NumberFormat` before any date or
number package. They cost no bytes. A package earns its place only where the
work is calendar arithmetic that `Intl` cannot express.

List an external package that ships a barrel in `optimizePackageImports`, which
`references/directory-and-module-boundaries.md` owns. Confirm the result in the
analyzer, because Next.js already carries a list of packages that it optimises
without the key.

`references/cell-formatting-and-export.md` owns the formatter itself, its locale,
and its time zone.

### A third-party script needs a cost, an owner, and a strategy

| The script | The strategy | Why |
| --- | --- | --- |
| Analytics, a marketing tag, a session recorder, a chat widget | `strategy="lazyOnload"` | It runs after the browser is idle, so it competes with no interaction of the first seconds. |
| A consent manager, or a tag manager that other scripts depend on | `strategy="afterInteractive"` | It runs soon after hydration, which is the earliest point that a non-critical script deserves. |
| A polyfill that the page needs before hydration | `strategy="beforeInteractive"` | It blocks. It is correct only where the application cannot hydrate without it. |
| A video player, a map, or a chat panel behind a button | A facade | No third-party code runs until the reader asks for it. |

```tsx
// Wrong: a chat widget blocks hydration.
// Failure: `beforeInteractive` runs the vendor script before React hydrates.
// The main thread is busy with code that no reader has asked for, so the first
// taps queue behind it and the interaction metric rises.
<Script src={chatWidgetSrc} strategy="beforeInteractive" />
```

```tsx
// Correct: the vendor script waits for an idle main thread.
import Script from "next/script";
import { chatWidgetSrc } from "@/config/third-party";

<Script src={chatWidgetSrc} strategy="lazyOnload" />;
```

Measure the script before it merges. Load the route with the script and without
it, on the same build and the same profile. Record both numbers in the pull
request, with the name of the person who owns the script.

A move from `afterInteractive` to `lazyOnload` lowers the interaction cost of an
analytics script. No primary source states the size of that change, so measure it
in this repository rather than quote a figure.

Take the component from `@next/third-parties` for a vendor that it covers. It
already carries the correct strategy and the correct attributes.

`references/image-and-video-delivery.md` owns the facade in front of a video
player. Domain 23 `analytics-privacy-and-consent` owns the lawful basis, the
consent gate, and the event schema. This file owns the cost and the moment.

### The origin hint

React 19 makes `preconnect`, `prefetchDNS`, and `preinit` stable, and
`references/suspense-and-actions.md` records that the functions exist. This file
states which one to call.

Call `preconnect` for an origin that the first seconds of the page need, and
that the first document does not name. Two or three such origins are the
ceiling. Each open connection costs the browser a socket and the device some
power.

Call `prefetchDNS` instead for an origin that a later interaction may need. It
resolves the name and stops there.

NEVER add a hint for an origin that the page already names in the first
document. The browser connects to that origin anyway, and the hint is dead
markup.

### The route prefetch

```tsx
// Wrong: every link in a list of 200 forces a full prefetch.
// Failure: the browser sends 200 requests that the reader did not ask for. On a
// mobile data plan those bytes are the reader's, and the requests compete with
// the data of the route that is on screen.
{rows.map((row) => (
  <Link key={row.id} href={`/orders/${row.id}`} prefetch>
    {row.reference}
  </Link>
))}
```

```tsx
// Correct: the default prefetch, over the shell of each route.
{rows.map((row) => (
  <Link key={row.id} href={`/orders/${row.id}`}>
    {row.reference}
  </Link>
))}
```

Next.js 16.3 prefetches one shell for each route rather than the whole route for
each link. Enable it with `cacheComponents` and `partialPrefetching` in
`next.config.ts`, which `references/app-router-structure.md` owns. A link-dense
page then costs a small number of requests instead of one for each link.

The first click into a route that no prefetch covered renders the shell, and the
content streams into it. That is the trade, and it is the correct one on a phone.

### The bytes that arrive and never run

Read the coverage panel over the production build. It reports the share of each
chunk that the route never executed. The 2025 Web Almanac measured 251 kB of
unused JavaScript on the median mobile page.

Unused bytes have three usual causes. A client component pulls a library that
one branch needs. A shared layout imports a widget that one route renders. A
package ships a barrel that the build could not unroll.

Fix the cause, and never the symptom. Move the branch behind `next/dynamic`, move
the logic to the server, or change the import shape.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those facts on 17 August 2026. A cell that holds no
date is a package with a current registry entry on that date. This file does not
state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `next/dynamic` | Every heavy client widget, with a `loading` skeleton of the final size. | Built in | 2026-08-03 | Vercel, active | None |
| Recommend | `next/script` | Every third-party script, with a stated strategy. | Built in | 2026-08-03 | Vercel, active | None |
| Recommend | `@next/third-parties` | A vendor that the package covers, in place of a hand-written `next/script` call. | Current | Current | Vercel, active | None |
| Recommend | `Intl.DateTimeFormat` and `Intl.NumberFormat` | Every date and every number that a reader sees. They cost no bytes. | Web platform | — | Web platform | None |
| Conditional | `lodash-es`, with a named import | Only where the platform has no equivalent. Never the default import. | Current | Current | Active | None |
| Audit-only | `lodash`, the CommonJS package | The barrel defeats tree-shaking. Move to `lodash-es` with named imports, or to the platform. | — | — | — | — |
| Reject | `moment` | 290.4 kB before compression, 72.1 kB after it. It is mutable and it does not tree-shake. Take `Intl`. | — | — | — | — |
| Reject | `next/script` with `strategy="worker"`, and Partytown behind it | The Next.js documentation marks the worker strategy experimental, and states that it does not work with the App Router. Take `lazyOnload` or a facade. | — | — | — | — |
| Reject | `strategy="beforeInteractive"` for analytics, marketing, or chat | It blocks hydration for code that no reader asked for. | — | — | — | — |
| Reject | Domain sharding | HTTP/2 and HTTP/3 reuse one connection. A second host costs a handshake and returns nothing. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| First Load JS jumped after a dependency landed | A barrel import, a package that does not tree-shake, or a new client boundary | Compare the route table of two builds, then open the analyzer | Use a named import, list the package in `optimizePackageImports`, or move the work to the server |
| A library ships on every route | The import sits at the top of a shared layout | Read the import chain in the analyzer | Move the import into the leaf that renders it, behind `next/dynamic` |
| A whole page ships to the browser | The directive sits on the page | `rg -n "'use client'" -g '*/page.tsx' src/` | Push the directive down to the smallest island |
| The coverage panel reports a high unused share | One branch of one route needs the library | The coverage panel over the production build | Split the branch behind `next/dynamic` |
| The first taps of a visit queue | A third-party script runs before or during hydration | The field attribution report names the longest script | Move the script to `lazyOnload`, or put a facade in front of it |
| The network panel floods with prefetch requests | A link-dense page forces a prefetch for each link | Load the page and count the requests | Delete the per-link prop, and enable the partial prefetch |
| A widget appears and the content below it moves | The `loading` skeleton is not the size of the widget | Reload on a throttled profile | Give the skeleton the height and the ratio of the final component |
| The bundle is small and the page is still slow | The cost is the render or the first byte, and not the bytes | Read the LCP sub-parts | `references/paint-and-interaction-cost.md` and `references/performance-budgets-and-measurement.md` |

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next.js 16.3 prefetches one shell for each route | `rg -n 'prefetch' -g '*.tsx' src/` reports the prop on many links | Set `cacheComponents` and `partialPrefetching`, and delete the per-link prop |
| The `worker` strategy never worked in the App Router | `rg -n 'strategy="worker"' -g '*.tsx' src/` reports a hit | Take `lazyOnload`, or put a facade in front of the embed |
| React 19 made `preconnect`, `prefetchDNS`, and `preinit` stable | A hand-written `<link rel="preconnect">` in a component | Call the function from `react-dom` |
| `@next/third-parties` covers the common vendors | A hand-written `next/script` call for a vendor that the package covers | Take the component, which carries the strategy already |

## Verification

```bash
# 1. Read the installed versions before you write code.
rg -n '"next"|"@next/third-parties"|"lodash"|"moment"' package.json

# 2. Build, and read First Load JS for each route.
next build

# 3. Open the treemap of the built chunks, and read the import chain of the
#    largest entry.
ANALYZE=true next build

# 4. Find a directive on a page or a layout. Each hit needs a reason above it.
rg -n "^\s*[\"']use client[\"']" -g 'page.tsx' -g 'layout.tsx' src/

# 5. Find a package that does not tree-shake. This prints nothing.
rg -n "from ['\"]moment['\"]|^import _ from ['\"]lodash['\"]" -g '*.ts*' src/

# 6. Find a third-party script. Each hit needs a strategy, a recorded cost, and
#    an owner.
rg -n '<Script' -g '*.tsx' src/

# 7. Find a blocking strategy on a non-critical script. This prints nothing.
rg -n 'strategy="beforeInteractive"|strategy="worker"' -g '*.tsx' src/

# 8. Find a heavy widget imported at the top of a layout. Read every hit.
rg -n '^import .*(editor|chart|map|calendar)' -g 'layout.tsx' src/

# 9. Find a dynamic import with no skeleton.
rg --files-with-matches 'next/dynamic' -g '*.tsx' src/ \
  | xargs rg --files-without-match 'loading:'

# 10. Find a forced prefetch on a link. Each hit needs a written reason.
rg -n 'prefetch' -g '*.tsx' src/

# 11. Open the coverage panel over `next start`, and load the primary route.
#     The unused share of the main chunk stays under about 40 percent.

# 12. Load the link-dense route, and count the prefetch requests in the network
#     panel. The count is a small number, and not one for each link.
```

## Review checklist

- [ ] Does every `"use client"` sit on the smallest island that needs it?
- [ ] Does every transform with no browser API and no interaction run on the
      server?
- [ ] Does every heavy client widget load through `next/dynamic`?
- [ ] Does every dynamic import carry a skeleton of the final size?
- [ ] Is `ssr: false` absent from a component that holds the element which paints
      the largest area?
- [ ] Is `moment` absent, and is every `lodash` import a named ESM import?
- [ ] Does every date and every number use `Intl`, rather than a package?
- [ ] Does every external barrel package appear in `optimizePackageImports`?
- [ ] Does every third-party script carry a recorded cost, a named owner, and a
      stated strategy?
- [ ] Is `beforeInteractive` absent from every analytics, marketing, and chat
      script?
- [ ] Is `strategy="worker"` absent from the project?
- [ ] Does every heavy embed sit behind a facade?
- [ ] Is a blanket prefetch prop absent from a link-dense page?
- [ ] Does the coverage panel report an unused share under about 40 percent on
      the main route?

## Handoffs

- The budget, the gate, the field report, and the measurement loop →
  `references/performance-budgets-and-measurement.md`.
- The largest paint, the layout shift, the long task, and the yield →
  `references/paint-and-interaction-cost.md`.
- The `"use client"` directive, the leaf rule, and the serialised prop →
  `references/server-and-client-components.md`.
- `optimizePackageImports`, the barrel rule, and the folder that holds a module
  → `references/directory-and-module-boundaries.md`.
- The chart island, and `ssr: false` for a canvas library →
  `references/charts-and-visual-encoding.md`.
- The facade in front of a video player, and the embed host →
  `references/image-and-video-delivery.md`.
- The `Intl` formatter, its locale, and its time zone →
  `references/cell-formatting-and-export.md`.
- `cacheComponents`, `partialPrefetching`, and the rest of `next.config.ts` →
  `references/app-router-structure.md`.
- The React resource functions as APIs →
  `references/suspense-and-actions.md`.
- The Motion entry points, and the cost of an animation library →
  `references/view-transitions-and-animation-libraries.md`.
- The install size of a new dependency, and its maintenance status →
  `references/dependencies-and-git-workflow.md`.
- The supply-chain surface of a third-party script, its `integrity`
  attribute, and the self-hosting decision →
  `references/secret-boundary-and-supply-chain.md`. The Content Security Policy
  that admits it is `references/security-headers-and-csp.md`. That domain holds
  a veto.
- The consent gate over a script, the lawful basis, and the event schema →
  domain 23 `analytics-privacy-and-consent`. Not integrated yet.
- The compression, the cache headers, and the CDN in front of `_next/static` →
  domain 22 `build-deploy-and-runtime-ops`. Not integrated yet.
