# Performance budgets and measurement

`web-vitals` 6.1.0, `@lhci/cli` 0.15.x with Lighthouse 12.6.1, `size-limit`,
`@next/bundle-analyzer`, Next.js 16.3, React 19.2.6 or later. This file owns
speed as a measured property. The subjects are the three metrics with their
thresholds, the build that a measurement is valid against, and the budget for
each route class. They also include the gate that fails the build, the field
report that gives the verdict, and the profile that finds the cost. The last
subject is the backend seam at the first byte.

The bytes that ship and the third-party script are
`references/client-bundle-and-third-party-scripts.md`. The largest paint, the
layout shift, and the answer to a tap are
`references/paint-and-interaction-cost.md`.

## Principle

Speed is a number at a percentile, over the devices that the readers hold.
Without that number it is an opinion.

A laboratory tool reports one run on one machine. It diagnoses a cause. It never
gives the verdict.

A budget that prints a warning is not a budget. A number that no gate enforces
returns to its old value in two releases.

A measurement of a development build measures the development build. The
development server adds work that production never runs.

A change with no number before it and no number after it is not an improvement.
It is a hope.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The three metrics, and the threshold for each

| The metric | What it measures | Good | Poor above |
| --- | --- | --- | --- |
| LCP | The time until the largest element paints | 2.5 s or less | 4.0 s |
| INP | The time from an interaction to the next paint | 200 ms or less | 500 ms |
| CLS | The largest layout shift in one session window | 0.1 or less | 0.25 |

Read each threshold at the 75th percentile of the visits. Read mobile and
desktop apart. A median that passes hides a quarter of the readers.

INP replaced First Input Delay on 12 March 2024. A budget or a document that
still names FID is stale. FID measured the delay of the first interaction alone.
INP measures every interaction of the visit.

### A measurement is valid against the production build

```bash
# Wrong: the number comes from the development server.
# Failure: `next dev` disables the prefetch, it compiles a route on the first
# request, and it ships development React. The number is 2 to 5 times the
# production number, and a change that moves it moves nothing for a reader.
next dev
```

```bash
# Correct: build, serve the build, and measure that process.
next build            # read the printed route table
next start            # measure against this process alone
```

NEVER report a performance number from `next dev`. State the build command, the
device profile, and the network profile beside every number. A number with no
profile beside it cannot be compared with the next one.

### The route table is the budget

`next build` prints First Load JS for each route. That column is the JavaScript
that a reader downloads before the route is interactive. Read it on every pull
request, and investigate an increase before you merge it.

Set the budget from the repository, and never from an external table. Run
`next build` once on the current main branch. Record First Load JS for each
route class. Set the budget at that number, less the headroom that the team
agrees on.

Anchor the result against the real world. The 2025 Web Almanac measured the July
2025 crawl. The median page shipped 646 kB of JavaScript on mobile, and 708 kB
on desktop. Of that, 251 kB on mobile went unused. Only 48 percent of mobile
origins passed all three metrics. A budget that sits well under the mobile
median is what separates a passing product from the failing majority.

Group the routes into classes before you set a number. A marketing route, an
application shell, and a dense data route each carry a different floor. One
budget over every route either passes everything or blocks the dense route
forever.

### The budget fails the build

```jsonc
// Wrong: the gate reports and continues.
// Failure: the report scrolls past in the CI log. First Load JS rises by 8 kB
// in each of six releases, and nobody reads the line that says so.
{ "assert": { "assertions": { "total-blocking-time": ["warn", { "maxNumericValue": 200 }] } } }
```

```jsonc
// Correct: lighthouserc.json, where the assertion fails the run.
{
  "ci": {
    "collect": { "numberOfRuns": 5, "startServerCommand": "next start" },
    "assert": {
      "assertions": {
        "largest-contentful-paint": ["error", { "maxNumericValue": 2500 }],
        "cumulative-layout-shift":  ["error", { "maxNumericValue": 0.1 }],
        "total-blocking-time":      ["error", { "maxNumericValue": 200 }],
        "resource-summary:script:size": ["error", { "maxNumericValue": 300000 }]
      }
    }
  }
}
```

Set `numberOfRuns` to 5 or more. One Lighthouse run on a shared CI machine
varies enough to fail a passing branch.

Add a byte gate beside the Lighthouse gate. `size-limit` reads the built chunks
and exits non-zero above the limit, which is the cheaper and the more stable of
the two.

```jsonc
// Correct: package.json, where the byte gate names the chunks and the limit.
{
  "size-limit": [{ "path": ".next/static/chunks/*.js", "limit": "170 kB" }]
}
```

CAUTION: a waiver is a line in the pull request, and never a raised limit with
no note. Raise a limit only where the release states what the bytes buy.

### The field gives the verdict

```text
// Wrong: the team reads the Lighthouse score alone.
// Failure: the laboratory runs a fast machine on a fast link. The p75 reader
// runs a mid-range phone on a slow link. The score is green while the field
// INP sits in the poor band, and the work goes to the wrong metric.
```

```tsx
// Correct: app/web-vitals.tsx reports the field metric for every visit.
"use client"; // it reads a browser measurement API

import { useReportWebVitals } from "next/web-vitals";

export function WebVitals() {
  useReportWebVitals((metric) => {
    navigator.sendBeacon(
      "/api/vitals",
      JSON.stringify({
        name: metric.name,
        value: metric.value,
        rating: metric.rating,
        id: metric.id,
      }),
    );
  });
  return null;
}
```

Mount that component once, in the root layout. `useReportWebVitals` reports LCP,
INP, CLS, FCP, and TTFB. It also reports the Next.js route-change metrics, which
no external tool measures.

Import from `web-vitals/attribution` where the report must name a cause. The
attribution build carries the element that produced the largest paint, the
target of the slowest interaction, and the longest script in the frame.

Where the field says a metric is good and the laboratory says it is poor, trust
the field. Where the two disagree the other way, trust the field again. The
laboratory then tells you which sub-part to open.

`references/error-and-empty-state-copy.md` states that a failed request needs a
message. A beacon that fails needs none. Never let a measurement path change what
the reader sees.

### The profile that finds the cost

Set a 4× CPU throttle and a Slow 4G profile before you profile anything. An
unthrottled desktop hides every cost that this domain exists to find.

| The question | The instrument | What it answers |
| --- | --- | --- |
| Which element is the largest paint, and which sub-part dominates it | The LCP breakdown in the browser performance panel | Whether the cost is the first byte, the discovery of the resource, the download, or the render |
| Which interaction is slow, and where the time goes | The `web-vitals` attribution build, over the field | The target, the processing time, and the longest script in the frame |
| Which bytes arrive and stay unused | The coverage panel, over the production build | The library that one branch of one route needs |
| Which component renders, and how often | The React DevTools profiler, on the production profiling build | The render cost that a state change produces |
| Where the memory goes over a session | Two heap snapshots, compared | The listener, the timer, or the object URL that no cleanup released |

Take one measurement, make one change, and take the measurement again. Two
changes between two numbers prove nothing about either.

### The backend seam

This domain meets Django and DRF at the payload and at the round trip. The
contract itself is `references/openapi-schema-and-codegen.md`, and the server
query belongs to the sibling skill `django-performance-optimizer`.

| The seam point | What it costs the browser | The decision |
| --- | --- | --- |
| A Server Component that awaits a DRF call | Every awaited call adds its latency to the first byte. Sequential awaits add up. | Run independent calls through `Promise.all`. Ask the backend for one aggregate endpoint where a view needs three resources or more. |
| A serializer that returns every field | A large JSON body becomes a large RSC payload, a larger stream, and more work at hydration. | Request the fields that the view renders. Project a large object to the three fields that a client component needs before you pass it as a prop. |
| A page size that the viewport does not need | 200 rows fetched for 20 rendered inflates the payload and the interaction cost. | Match the page size to the viewport. `references/data-table-and-server-driven-state.md` owns the server-driven table. |
| The generated client | The types cost no run-time bytes. The client that sends the request does. | `openapi-fetch` is 1 kB to 2 kB. A general-purpose HTTP client is an order of magnitude more. |
| A slow first byte from Django | Where the largest element is text in a system font, LCP is the first byte plus the render. No image work helps. | Prove it from the LCP sub-parts, then hand the measured server timing to `django-performance-optimizer`. |

A field name that the backend renames fails the typecheck, which is a
correctness gate. An endpoint that silently returns more data fails no type. Only
the route table and the field payload watch catch it.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those facts on 17 August 2026. A cell that holds no
date is a package with a current registry entry on that date. This file does not
state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `web-vitals`, the attribution build | The field measurement of LCP, INP, and CLS. It is the source of the verdict. | 6.1.0 | Current | Google, active | None |
| Recommend | `@lhci/cli` | The laboratory gate in CI, over a production build, with 5 runs or more. | 0.15.x, on Lighthouse 12.6.1 | Current | Google, active | None |
| Recommend | `size-limit` | The byte gate over the built chunks. It is cheaper and steadier than a Lighthouse run. | Current | Current | Evil Martians, active | None |
| Recommend | `@next/bundle-analyzer` | The treemap of the built chunks, behind the `analyze` script. | Ships with Next 16.3 | 2026-08-03 | Vercel, active | None |
| Conditional | The Turbopack bundle analyzer | Next 16.1 and later. It writes the report to disk, so two builds compare. It is experimental. | Ships with Next 16.3 | 2026-08-03 | Vercel, active | None |
| Conditional | An external field service, such as WebPageTest | Only where the team needs a filmstrip, a connection-level first byte, or a percentile from a named region. | — | — | — | — |
| Conditional | The React DevTools profiler | Only for a render cost. It needs the production profiling build. | — | — | — | — |
| Reject | A performance number from `next dev` | The development server adds work that production never runs. | — | — | — | — |
| Reject | The Lighthouse PWA category | Lighthouse 12.0.0 removed it. An old configuration that asserts on it errors. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| The laboratory LCP is good and the field LCP is poor | The laboratory runs a fast machine on a fast link | Compare the field percentile against the Lighthouse run | Throttle to 4× CPU and Slow 4G, and optimise the sub-part that dominates the field |
| The largest paint does not move after the image work | The first byte or the render delay dominates, and not the image | Read the four LCP sub-parts | Where it is the first byte, hand the timing to `django-performance-optimizer` |
| Every branch fails the Lighthouse gate at random | One run on a shared CI machine | Run the same commit twice | Set `numberOfRuns` to 5 or more, and assert on the median |
| First Load JS rose and nobody noticed | The route table is not read on a pull request | Compare the route table of two builds | Add the byte gate, and read the table in the review |
| The budget passes and the readers complain | The budget was copied from an article | Compare the budget against the route table of this repository | Set the budget from `next build` on main, less the agreed headroom |
| The score is green and the field INP is poor | The work went to the laboratory number | Read the field percentile beside the score | Move the work to the interaction that the attribution build names |
| An old Lighthouse configuration errors in CI | The configuration asserts on the PWA category | Read the CI failure | Delete the assertion |

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| INP replaced FID on 12 March 2024 | `rg -in 'first input delay' docs/` reports a hit | Budget INP, and measure every interaction of the visit |
| Lighthouse 12.0.0 removed the PWA category, and PageSpeed Insights followed on 10 May 2024 | `rg -n 'pwa' lighthouserc.json` reports a hit | Delete the assertion |
| `web-vitals` 6 carries the long animation frame in the attribution build | An import from `web-vitals` in code that needs a cause | Import from `web-vitals/attribution` |
| Next 16.1 added the Turbopack bundle analyzer | The project runs the webpack analyzer alone | Read the report at `.next/diagnostics/analyze`, and compare two builds |
| Next 16 removed `next lint` | A CI step that expects `next lint` to run a check | Call `next build`, `size-limit`, and `@lhci/cli` directly |
| Next 16.3 native Node streams, which the release notes measure at up to 22 percent more requests under load | A Next version below 16.3 in `package.json` | Upgrade, and take a new first-byte baseline before you compare |

## Verification

```bash
# 1. Read the installed versions before you write code.
rg -n '"next"|"web-vitals"|"@lhci/cli"|"size-limit"|"@next/bundle-analyzer"' package.json

# 2. Build, then read the route table. Compare First Load JS against the budget
#    for each route class.
next build

# 3. Serve the build, and measure that process. Never measure `next dev`.
next start

# 4. Run the byte gate. It exits non-zero above the limit.
npx size-limit

# 5. Run the laboratory gate over the production build.
npx lhci autorun

# 6. Find a Lighthouse configuration that still asserts on the PWA category.
#    This prints nothing.
rg -n 'pwa' lighthouserc.json

# 7. Find a project with a Lighthouse configuration and no run count. This
#    prints nothing.
rg --files-with-matches 'lighthouserc' . \
  | xargs rg --files-without-match 'numberOfRuns'

# 8. Find a stale FID budget. This prints nothing.
rg -in 'first input delay' docs/ .github/

# 9. Confirm that the field report is mounted once. Read every hit.
rg -n 'useReportWebVitals' -g '*.tsx' src/

# 10. Throttle the CPU to 4× and the network to Slow 4G. Load the primary
#     route. The largest paint lands under 2.5 s, and the primary interaction
#     answers under 200 ms.

# 11. Read the field percentile after the release. p75 LCP is 2.5 s or less,
#     p75 INP is 200 ms or less, and p75 CLS is 0.1 or less.
```

## Review checklist

- [ ] Does every performance number in the review name the build, the CPU
      profile, and the network profile that produced it?
- [ ] Is every number from `next build` and `next start`, rather than from
      `next dev`?
- [ ] Does the change carry a number before it and a number after it?
- [ ] Does each route class carry a First Load JS budget, taken from this
      repository?
- [ ] Does a gate fail the build above the budget, rather than print a warning?
- [ ] Does the Lighthouse configuration run 5 times or more?
- [ ] Does the project report LCP, INP, and CLS from the field?
- [ ] Does the code import from `web-vitals/attribution` where it needs a cause?
- [ ] Does the verdict come from the field percentile, rather than from the
      laboratory score?
- [ ] Was the route table read on this pull request, with every increase
      explained?
- [ ] Do independent server calls in one route run through `Promise.all`?
- [ ] Was the first byte proved as the cause before any server work was
      requested?

## Handoffs

- The bytes of JavaScript, the import shape, the third-party script, and the
  prefetch → `references/client-bundle-and-third-party-scripts.md`.
- The largest paint element, the layout shift, the long task, and the yield →
  `references/paint-and-interaction-cost.md`.
- The `analyze` script, the `check` script, and the rest of the `package.json`
  script surface → `references/lint-format-and-scripts.md`.
- The route render mode, the build report symbols, and `next.config.ts` →
  `references/app-router-structure.md`.
- The cache decision behind a first byte, and `"use cache"` →
  `references/caching-and-revalidation.md`.
- The query key, the cache times, and the four states of a data view →
  `references/server-state-and-query-cache.md`.
- The generated client, the schema, and the pagination envelope →
  `references/openapi-schema-and-codegen.md` and
  `references/api-client-and-request-safety.md`.
- The server-driven table, the page size, and the virtualiser threshold →
  `references/data-table-and-server-driven-state.md`.
- The install size of a new dependency, and its maintenance status →
  `references/dependencies-and-git-workflow.md`.
- The transport that carries a field report, and the trace →
  `references/correlation-and-telemetry.md`. The error tracker is
  `references/error-capture-and-reporting.md`.
- The CI workflow file that runs the budget stage →
  `references/release-pipeline-and-rollback.md`. The reverse proxy and the
  compression in front of Node →
  `references/runtime-process-and-reverse-proxy.md`.
- The test runner → `references/test-strategy-and-component-tests.md`. The
  place of the budget stage in the gate order →
  `references/merge-gates-and-quality-signals.md`.
- The slow endpoint, the query count, and the server cache → sibling skill
  `django-performance-optimizer`.
- The serializer fields and the pagination envelope as a published surface →
  sibling skill `django-api-contract`.
