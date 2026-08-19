# Event taxonomy and tracking plan

Next.js 16.3, React 19.2, TypeScript 5.9, `next/script`. This file owns the
measurement that a product takes of itself. The subjects are the question that
an event answers, the name and the properties of that event, and the one module
that sends it. They also include the identity on an event, the page view over a
soft navigation, and the event that the server sends. The last subjects are the
sample rate, the origin that the collector answers on, and the application under
a blocked script.

The consent that must arrive before any of this runs is
`references/consent-gate-and-cookie-inventory.md`. The export, the deletion, and
the policy page are `references/data-rights-and-privacy-surfaces.md`. The
strategy of a `next/script` tag, its measured cost, and the facade in front of an
embed are `references/client-bundle-and-third-party-scripts.md`.

## Principle

A measurement that changes no decision is a cost with no return. Name the
decision first. Name the event second.

An event dictionary is a contract between the code and the people who read the
charts. A name that two features spell differently produces two charts, and
neither one is correct.

A feature that calls a vendor SDK is a feature that the next vendor rewrites.
One module holds the vendor. Every feature calls that module.

An event property is a permanent record in a store that somebody else operates.
That store has its own retention, its own access list, and its own jurisdiction.
Send the minimum that answers the question.

A measurement tool is not a dependency of the product. The application must work
when the tool is absent, blocked, or down.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The decision comes before the event

| The question the team must answer | What to measure | The number that answers nothing |
| --- | --- | --- |
| Does a new reader reach the first value of the product | The activation event, and the step before it | The count of new accounts |
| Do readers come back | The retention cohort over a stated window | The total count of accounts |
| Where does a flow lose people | The ordered funnel of that flow | The page view of the last screen |
| Does a change improve the flow | The variant, and the outcome event beside it | The page view of the changed screen |

Write the question in the pull request that adds the event. An event with no
question behind it never reaches a chart, and it costs a permanent record.

NEVER add an event that no person has agreed to act on.

### One module holds the vendor

```ts
// src/lib/analytics.ts
// Correct: one module holds the vendor, and every feature imports track().
import type { TrackedEvent } from "@/lib/analytics-events";

type Collector = {
  capture: (name: string, properties: Record<string, unknown>) => void;
  identify: (id: string, properties: Record<string, unknown>) => void;
  reset: () => void;
};

let collector: Collector | null = null;

export function registerCollector(next: Collector): void {
  collector = next;
}

export function track<K extends TrackedEvent["name"]>(
  name: K,
  properties: Extract<TrackedEvent, { name: K }>["properties"],
): void {
  if (process.env.NODE_ENV !== "production") {
    console.info("[analytics]", name, properties);
    return;
  }
  collector?.capture(name, properties);
}
```

```tsx
// Wrong: the feature imports the vendor SDK.
// Failure: the vendor name appears in forty files, so a vendor change edits all
// forty. A component test that renders this button sends a real event, and the
// counts of the test suite reach the production charts.
import posthog from "posthog-js";

<button onClick={() => posthog.capture("Checkout Button Clicked")}>Pay</button>;
```

Three rules bind the module. It exports `track`, `identifyUser`, and
`resetIdentity`, and nothing else. It sends no event outside production, so a
test and a development session emit no traffic. It holds a `null` collector
until a registered vendor replaces it.

The `console.info` line is the event inspector. A developer reads the name and
the properties of every event in the browser console, before the event reaches a
chart.

`references/directory-and-module-boundaries.md` owns the folder that holds the
module. `references/test-strategy-and-component-tests.md` owns the test that
proves the no-op.

### The event dictionary is a type

```ts
// src/lib/analytics-events.ts
// Correct: the union is the dictionary. An event outside it fails the typecheck.
export type TrackedEvent =
  | { name: "signup_completed"; properties: { plan: "free" | "team"; source: string } }
  | { name: "invoice_downloaded"; properties: { invoiceId: string; format: "pdf" | "csv" } }
  | { name: "checkout_step_viewed"; properties: { step: 1 | 2 | 3 } };
```

```ts
// Wrong: the name and the properties are free strings.
// Failure: one feature sends "Checkout Step Viewed" and another sends
// "checkoutStepViewed". The funnel splits into two charts, and neither half is
// the number that the team reads. Nothing fails the build.
track("Checkout Step Viewed", { step: "two" });
```

Name every event as the object and then the action, in one case for the whole
project. `invoice_downloaded` is the object `invoice` and the action
`downloaded`. Keep that case for every event, in every feature.

The union is the tracking plan, and the file is the document. A reviewer reads
one file to learn every event that the product sends.

| The kind of property | What it describes | Where it belongs |
| --- | --- | --- |
| An event property | The one action, at the moment of the action | The `properties` of the event |
| A user property | The person, and it holds until it changes | The `identifyUser` call |
| A group property | The organization or the workspace of the person | The group call of the vendor |

NEVER repeat a user property on every event. The value is the same on each one,
it costs bytes on each one, and two events then disagree after a change.

`references/type-modeling-and-narrowing.md` owns the discriminated union and the
`Extract` helper. This file owns the shape that the union holds.

### The identity, and the reset that ends it

```ts
// src/lib/analytics.ts, continued
// Correct: the stable backend id, and no direct identifier beside it.
export function identifyUser(userId: string, plan: "free" | "team"): void {
  collector?.identify(userId, { plan });
}

export function resetIdentity(): void {
  collector?.reset();
}
```

```ts
// Wrong: the email is the identifier, and no call ends the session.
// Failure: the vendor store now holds a direct identifier for every reader, so
// a deletion request covers that store as well. The next person on a shared
// device inherits the identity, and their events join the previous account.
posthog.identify(user.email, { name: user.fullName });
```

Four rules hold. The browser carries an anonymous id before a login. The login
calls `identifyUser` with the stable id that Django issues. The sign-up merges
the anonymous id into the new account, so the events before the sign-up stay in
the funnel. The logout calls `resetIdentity`.

`references/session-and-token-lifecycle.md` owns the login and the logout. This
file owns the two calls that each one makes.

### The page view fires once for one navigation

```tsx
// src/components/analytics/page-view.tsx
"use client"; // reason: the hooks read the live pathname and the query string

import { usePathname, useSearchParams } from "next/navigation";
import { useEffect } from "react";
import { safePath, track } from "@/lib/analytics";

export function PageView() {
  const pathname = usePathname();
  const searchParams = useSearchParams();

  useEffect(() => {
    track("page_viewed", {
      path: safePath(pathname),
      tab: searchParams.get("tab"),
    });
  }, [pathname, searchParams]);

  return null;
}
```

```tsx
// Wrong: the vendor captures the page view, and the component sends one too.
// Failure: every soft navigation records two page views. Each funnel step
// counts twice, so the reported conversion rate of the funnel is half the real
// one, and no code change explains the number.
posthog.init(key, { capture_pageview: true });
```

Turn the automatic capture of the vendor off, and own the event. A Single Page
Application changes the route with no document load, so the automatic capture and
the manual call both fire.

WARNING: `useSearchParams` needs a `<Suspense>` boundary above it, or the build
stops. Wrap `<PageView />` in the root layout.
`references/client-and-url-state.md` owns that rule and the error message.

```tsx
// src/app/layout.tsx
// Correct: the consumer of useSearchParams sits under a boundary.
<Suspense fallback={null}>
  <PageView />
</Suspense>
```

NEVER send the whole query string. A password reset key, an invitation token, and
a signed URL all live in a query string. Read the parameters that the tracking
plan names, and drop the rest.

### The property carries no personal value

| The value | Why it never leaves | What to send instead |
| --- | --- | --- |
| An email, a name, or a telephone number | It is a direct identifier in a store that somebody else operates | The stable backend id, on the `identifyUser` call alone |
| A full URL that holds a token, a reset key, or a record id | The query string reaches the vendor whole, and the vendor keeps it | The pathname with the identifiers replaced, and a named parameter |
| Free text that a reader typed | Nobody can state what a reader typed into it | The count of characters, or nothing |
| The network address of the reader | It identifies a household, and it crosses a border with the event | A vendor setting that discards it, or a self-hosted collector |

```ts
// src/lib/analytics.ts, continued
// Correct: the module replaces a path segment that holds an identifier.
const ID_SEGMENT = /^\d+$|^[0-9a-f]{8,}$/i;

export function safePath(pathname: string): string {
  return pathname
    .split("/")
    .map((segment) => (ID_SEGMENT.test(segment) ? ":id" : segment))
    .join("/");
}
```

```ts
// Wrong: the raw location goes into the property.
// Failure: the invitation token in the query string reaches the vendor store.
// Anybody with access to that store can accept the invitation, and the token
// stays in the record after the invitation expires.
track("page_viewed", { url: window.location.href });
```

Put the scrub inside the module, and never inside a feature. A rule that each
caller must remember is a rule that one caller forgets.

`references/error-capture-and-reporting.md` owns the same scrub on the path that
an error report takes. The two paths are separate, and each one needs its own
gate.

### The server sends the event that must arrive

| The event | Where it belongs | Why |
| --- | --- | --- |
| A sign-up, a payment, a subscription change, a plan change | The server | The number decides revenue, and a blocked browser must not change it |
| A click, a hover, a scroll, a step of a flow | The browser | The action exists only in the browser, and a loss of some of them is acceptable |
| A page view | The browser | The route change happens in the browser |

```ts
// src/lib/analytics-server.ts
// Correct: the server sends the event, and a failure never fails the mutation.
import "server-only";
import { after } from "next/server";
import { env } from "@/env";
import { logger } from "@/lib/logger";

export function trackServer(
  name: string,
  distinctId: string,
  properties: Record<string, unknown>,
): void {
  after(async () => {
    try {
      await fetch(env.ANALYTICS_COLLECTOR_URL, {
        method: "POST",
        headers: { "content-type": "application/json" },
        body: JSON.stringify({ event: name, distinct_id: distinctId, properties }),
        signal: AbortSignal.timeout(3000),
      });
    } catch (cause) {
      logger.warn({ cause, event: name }, "the analytics collector refused an event");
    }
  });
}
```

```ts
// Wrong: the Server Action waits for the collector.
// Failure: the collector answers in 900 ms, so every subscription takes 900 ms
// longer. A collector that stops answering holds the action open until the
// request times out, and the reader sees a failed subscription that succeeded.
export async function subscribe(formData: FormData) {
  const result = await createSubscription(formData);
  await fetch(collectorUrl, { method: "POST", body: JSON.stringify(result) });
  return result;
}
```

`after()` runs the callback when the response is sent, so the reader waits for
nothing. `references/build-output-and-container-image.md` states that the process
runs those callbacks before it exits, which is why the drain period matters.

The `catch` writes a line and stops. `references/correlation-and-telemetry.md`
owns the logger and the fields that a line may carry.

### The script and the collector answer on your own origin

A vendor script on a vendor origin costs three things. The browser opens a second
connection. The policy needs a `script-src` entry and a `connect-src` entry for
that origin. A content blocker recognizes the origin and drops the request.

```ts
// next.config.ts
// Correct: one rewrite puts the vendor script and its collector on this origin.
async rewrites() {
  return [
    { source: "/ingest/static/:path*", destination: `${env.ANALYTICS_ASSET_HOST}/static/:path*` },
    { source: "/ingest/:path*", destination: `${env.ANALYTICS_HOST}/:path*` },
  ];
}
```

```tsx
// Wrong: the vendor origin appears in the markup.
// Failure: the policy needs two more origins, and the list of hosts that may run
// code on this application grows. A blocker that holds the vendor origin in its
// list drops the script, and the property loses those readers.
<Script src={`${VENDOR_ORIGIN}/static/array.js`} strategy="lazyOnload" />
```

WARNING: the destination of a rewrite is a constant, and never a value that a
request supplies. `references/app-router-structure.md` states that rule and names
the advisory behind it.

`references/security-headers-and-csp.md` owns the `script-src` and `connect-src`
entries. `references/runtime-process-and-reverse-proxy.md` owns the layer in
front of Node, which may hold the same rewrite. The judgment of the vendor
package, and the `integrity` attribute on a versioned file, are
`references/secret-boundary-and-supply-chain.md`. That domain holds a veto.

The research reports that a content blocker removes between about 20 and 40
percent of client-side events. It names no source for that range. Measure the
loss in this property, and compare the client count of one event against the
server count of the same event.

NEVER build a feature that a reader needs on a script that a blocker can remove.

### A blocked script never breaks the application

```tsx
// Wrong: the handler reads the global that the vendor script defines.
// Failure: a blocker drops the script, `window.posthog` is undefined, and the
// handler throws on the first click. The error boundary replaces the page, so a
// blocked measurement tool becomes a blank screen.
<button onClick={() => window.posthog.capture("invoice_downloaded")}>Download</button>
```

```tsx
// Correct: the module holds the vendor, and a null collector is a no-op.
<button onClick={() => track("invoice_downloaded", { invoiceId, format: "pdf" })}>
  Download
</button>
```

Three rules keep the application alive. No module holds a top-level `await` on a
vendor. No feature reads a global that a script defines. The module answers with
no error when the collector is `null`.

`references/degradation-and-health-checks.md` owns the application under a dead
backend. This file owns the application under a dead measurement tool. The two
failures need separate rules, because a dead backend is visible and a dead
collector is silent.

### The server resolves the variant

```tsx
// Wrong: the browser resolves the variant after hydration.
// Failure: every reader sees the control for a moment, and the variant then
// replaces it. The swap moves the content below it, so the layout shift of the
// route rises and the reader loses the line that they were reading.
const variant = useVariant("checkout_layout");
```

```tsx
// Correct: the server resolves the variant, and the route renders one of them.
import { cookies } from "next/headers";
import { resolveVariant } from "@/lib/experiments";

export default async function Page() {
  const variant = await resolveVariant("checkout_layout", await cookies());
  return variant === "b" ? <CheckoutB /> : <CheckoutA />;
}
```

Resolve the variant in a Server Component, or in `proxy.ts` where the assignment
must reach every route. The reader then receives one variant in the first
document, and no swap follows.

Carry the variant as a property on the outcome event, and never as a separate
event. An experiment with no outcome event beside it measures the assignment
alone.

A route that reads a cookie is a dynamic route.
`references/caching-and-revalidation.md` owns that consequence. The layout shift
that a client-side swap produces is
`references/paint-and-interaction-cost.md`.

### The sample rate is a property

```ts
// src/lib/analytics.ts, continued
// Correct: the rate travels with the event, so a chart multiplies it back.
export function trackSampled<K extends TrackedEvent["name"]>(
  name: K,
  properties: Extract<TrackedEvent, { name: K }>["properties"],
  rate: number,
): void {
  if (Math.random() >= rate) return;
  track(name, { ...properties, sample_rate: rate });
}
```

Sample a high-frequency event, such as a scroll or a pointer move. The full rate
costs money and adds no information.

NEVER sample a sign-up, a payment, or a subscription change. A rate on a revenue
number makes that number an estimate.

### The libraries

The research names each vendor. It compares them on privacy posture, cookie
requirement, self-hosting, bundle weight, data ownership, and legal exposure in
the European Union. It states no version, no release date, and no advisory record
for any of them. Those three columns hold `Not stated`. Read the registry entry
and the advisory database of a package before the project installs it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | Plausible, Umami, or Fathom | The first choice for a small product. Each one is cookieless and self-hostable, so the count needs no banner in many jurisdictions and a blocker recognizes no vendor origin. | Not stated | Not stated | Not stated | Not stated |
| Recommend | A hand-written `analytics.ts` module | Every project. It costs one file, and it makes every row below replaceable. | — | — | — | — |
| Conditional | PostHog | Only where the product needs product analytics, flags, and replay in one tool. It sets a cookie, so it needs consent. It is self-hostable. | Not stated | Not stated | Not stated | Not stated |
| Conditional | Matomo | Only where the team needs a mature self-hosted tool with a cookieless mode. Verify which mode the deployment runs. | Not stated | Not stated | Not stated | Not stated |
| Conditional | Mixpanel, Amplitude, or Vercel Analytics | Only where a stated product requirement names the tool. Each one is a hosted third party, so it adds an origin, a data controller, and a transfer question. | Not stated | Not stated | Not stated | Not stated |
| Conditional | GrowthBook, Statsig, or Unleash | Only where the experiment needs a tool that the analytics vendor does not carry. Resolve the variant on the server. | Not stated | Not stated | Not stated | Not stated |
| Audit-only | Google Analytics 4 | It sets cookies, it needs consent, it carries the largest legal exposure in the European Union, and blockers remove it first. Replace it where no requirement names it. | Not stated | Not stated | Not stated | Not stated |
| Reject | Google Tag Manager with no review process | It is a channel through which somebody adds code after the review. `references/secret-boundary-and-supply-chain.md` holds the veto. | — | — | — | — |
| Reject | A vendor SDK imported in a feature file | The vendor then appears in every feature, and no test can silence it. | — | — | — | — |
| Reject | A heatmap, a survey, or an in-app message tool with no measured cost | Each one ships a script that runs on every route. `references/client-bundle-and-third-party-scripts.md` owns the budget. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| The funnel reports half the real conversion rate | The vendor captures the page view, and the application sends one too | Navigate once, and count the events in the vendor inspector | Turn the automatic capture off, and keep the manual call |
| Two charts carry one action | Two features spell the event name differently | Read the event list of the vendor, and sort it | Move both names into the union, and correct the loser |
| A vendor store holds an email address | A feature passed the email as a property or as the identifier | Open one event, and read its properties | Send the backend id, and ask the vendor to delete the field |
| The events of a test run reach the production charts | The module sends outside production | Run the suite, and read the vendor inspector | Return early when `NODE_ENV` is not production |
| The counts fall by a third with no product change | A blocker recognizes the vendor origin | Compare the browser count of an event against the server count | Proxy the script and the collector through this origin |
| A click does nothing, and the page turns blank | The handler reads a global that a blocker removed | Block the vendor origin, and press the control | Call `track` from the module, and never a global |
| A subscription takes a second longer than the endpoint | The Server Action awaits the collector | Read the server timing of the action | Move the send into `after()` |
| The layout moves once on every first visit | The browser resolves the variant after hydration | Load the route on a throttled profile | Resolve the variant on the server |
| A revenue number changes each time somebody reads it | A sample rate sits on a revenue event | Read the rate of the event in the tracking plan | Remove the rate from that event |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next.js 16 makes `params` and `searchParams` async | A page that reads `searchParams` with no `await` before it tracks a view | `references/app-router-structure.md` owns the await. Read the value in the client component instead. |
| `useSearchParams` needs a `<Suspense>` boundary | `rg -n 'useSearchParams' -g '*.tsx' src/` reports a consumer with no boundary above it | Wrap the consumer. `references/client-and-url-state.md` owns the rule. |
| `after()` runs work when the response is sent | An `await` on a collector inside a Server Action | Move the send into `after()` |
| React 19 makes `preconnect` and `prefetchDNS` stable | A hand-written `<link rel="preconnect">` for a vendor origin | Call the function, or delete the hint once the collector answers on this origin |

## Verification

```bash
# 1. Confirm that one module holds the vendor. Expect one file.
rg -l 'posthog-js|@amplitude|mixpanel-browser|@vercel/analytics' src/

# 2. Find a feature that imports a vendor SDK. Each hit is a defect.
rg -n 'posthog-js|mixpanel-browser|@amplitude' -g '!src/lib/analytics*' src/

# 3. Find a global read of a vendor. This prints nothing.
rg -n 'window\.(posthog|gtag|dataLayer|_paq|amplitude)' src/

# 4. Confirm that every event name sits in the union.
rg -o -N "track\(\"[a-z_]+\"" src/ | sort -u
rg -n 'name: "' src/lib/analytics-events.ts

# 5. Find an event name outside the project case. Each hit is a defect.
rg -n 'track\("[^"]*[A-Z ]' src/

# 6. Find a personal value in a property. Read every hit.
rg -n 'track\([^)]*(email|name|phone|token|password)' src/

# 7. Find a raw location in a property. This prints nothing.
rg -n 'track\([^)]*(location\.href|location\.search|toString\(\))' src/

# 8. Confirm the no-op outside production. Expect the early return.
rg -n -A3 'NODE_ENV' src/lib/analytics.ts

# 9. Find a Server Action that awaits the collector. Read every hit.
rg -n -B4 'ANALYTICS_COLLECTOR_URL' src/ | rg 'await fetch'

# 10. Confirm that the collector answers on this origin. Expect the rewrite.
rg -n -A6 'rewrites' next.config.ts

# 11. Run the suite, then read the vendor inspector. Expect no event.
pnpm test

# 12. Block the vendor origin in the browser, then walk the main flow. Every
#     control still answers, and no boundary renders.
```

## Review checklist

- [ ] Does every event carry a written question that a person agreed to act on?
- [ ] Does one module hold the vendor, and does every feature import `track`?
- [ ] Does the union hold every event name and its property shape?
- [ ] Does an event name outside the union fail `tsc --noEmit`?
- [ ] Does every event name follow the object and action order, in one case?
- [ ] Does the module send nothing outside production?
- [ ] Does `identifyUser` take the stable backend id, and never an email?
- [ ] Does the logout call `resetIdentity`?
- [ ] Does one navigation produce one page view, with the automatic capture off?
- [ ] Does the page view send a scrubbed path and a named parameter, and never
      the whole query string?
- [ ] Does the scrub sit inside the module rather than inside each caller?
- [ ] Does every revenue event leave from the server?
- [ ] Does the server send run inside `after()`?
- [ ] Does the vendor script and its collector answer on this origin?
- [ ] Does the application work with the vendor origin blocked?
- [ ] Is a top-level `await` on a vendor absent, and is a global read absent?
- [ ] Does the server resolve every experiment variant?
- [ ] Is a sample rate absent from every sign-up, payment, and plan change?

## Handoffs

- The consent that a script waits for, the consent record, the `Sec-GPC`
  signal, and the cookie inventory →
  `references/consent-gate-and-cookie-inventory.md`.
- The export, the deletion, the retention statement, and the policy page →
  `references/data-rights-and-privacy-surfaces.md`.
- The `next/script` strategy, the measured cost of a script, the named owner, and
  the facade in front of an embed →
  `references/client-bundle-and-third-party-scripts.md`.
- The layout shift that a late swap produces, and the long task that a vendor
  script adds → `references/paint-and-interaction-cost.md`. The budget over both
  → `references/performance-budgets-and-measurement.md`.
- The `script-src` and `connect-src` entries that admit a vendor origin →
  `references/security-headers-and-csp.md`. The judgment of a vendor package, the
  `integrity` attribute, and the tag manager as a channel →
  `references/secret-boundary-and-supply-chain.md`. That domain holds a veto.
- The scrub on the path that an error report takes, and the mask over a session
  replay → `references/error-capture-and-reporting.md`. The logger and the fields
  that one line may carry → `references/correlation-and-telemetry.md`.
- The login, the logout, and the stable id that Django issues →
  `references/session-and-token-lifecycle.md`.
- The `<Suspense>` boundary over a `useSearchParams` consumer, and the parse over
  a query value → `references/client-and-url-state.md`.
- The discriminated union, the `Extract` helper, and the parse over an
  environment value → `references/type-modeling-and-narrowing.md` and
  `references/boundary-validation-and-api-types.md`.
- `next.config.ts` as a file, the rewrite rule, `proxy.ts`, and the dynamic route
  that a cookie read produces → `references/app-router-structure.md` and
  `references/caching-and-revalidation.md`.
- The drain period that lets an `after()` callback finish, and the layer in front
  of Node → `references/build-output-and-container-image.md` and
  `references/runtime-process-and-reverse-proxy.md`.
- The test that proves the module sends nothing, and the browser test that blocks
  the vendor origin → `references/test-strategy-and-component-tests.md` and
  `references/end-to-end-journeys-and-flake-control.md`.
- The application under a dead backend, and the offline state →
  `references/degradation-and-health-checks.md`.
- The Django endpoint that receives a server event, and the retention of that
  record on the server → the sibling skills `django-api-contract` and
  `secure-code-auditor`. This file owns the event that the frontend sends.
