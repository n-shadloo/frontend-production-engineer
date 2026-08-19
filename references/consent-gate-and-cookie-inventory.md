# Consent gate and cookie inventory

Next.js 16.3, React 19.2, Zod 4.4, GDPR, the ePrivacy Directive, the Global
Privacy Control specification. This file owns the permission that a reader gives
before the application stores anything that the product does not need. The
subjects are the essential category, and every category beside it. They also
include the gate that holds a script out of the document, and the record that
keeps the answer. The last subjects are the signal in a request header, the tool
choice that removes the question, and the inventory behind the policy page.

The event, the analytics module, and the collector are
`references/event-taxonomy-and-tracking-plan.md`. The export, the deletion, and
the policy page as a rendered surface are
`references/data-rights-and-privacy-surfaces.md`. The words on the two controls,
and the five patterns that take a decision from the reader, are
`references/interface-copy-and-voice.md`.

## Principle

Consent is permission to act, and it exists before the act. A permission that
arrives after the script ran is a record of a thing that already happened.

The rule covers access to storage, and not the send of a value. A script that
reads the device before the reader answers has already broken the rule, whatever
it sends afterward.

A choice is free only where refusal costs the reader nothing. A refusal that
takes more presses than an agreement is not a free choice.

An answer that the application forgets is an answer that the application asks
again. The record holds what the reader chose, when they chose it, and which
policy they read.

The tool decides the size of the question. A tool that stores nothing on the
device asks the reader for nothing.

An inventory that nobody verifies becomes a policy page that states a false
thing. The page is a claim, and the browser holds the evidence.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### Essential and non-essential decide everything

Article 5(3) of the ePrivacy Directive covers every access to the device of a
reader. It covers a cookie, `localStorage`, `sessionStorage`, IndexedDB, and a
cache that the application writes. Access to any of them needs consent, unless
the reader asked for the thing that the access delivers.

| The category | What belongs in it | Does it need consent |
| --- | --- | --- |
| Essential | The session cookie, the CSRF cookie, the consent record, a cart, a language choice, a fraud control | No. The reader asked for the service that needs it. |
| Analytics | A product analytics cookie, a session replay, a heatmap, an experiment assignment | Yes |
| Marketing | An advertising pixel, a conversion tag, a remarketing identifier | Yes |
| Preference | A theme, a layout, a dismissed notice, where the product works without it | Yes, unless the reader set the value themselves |

`references/session-and-token-lifecycle.md` owns the session cookie and its
attributes. That cookie is essential, and it never sits behind the gate.

NEVER place an analytics cookie in the essential category. The category is the
purpose of the value, and never the wish of the team.

### The gate holds the script out of the document

```tsx
// src/app/layout.tsx
// Correct: the server reads the record, and the tag reaches the document only
// where the reader granted the category.
import { cookies } from "next/headers";
import Script from "next/script";
import { ConsentBanner } from "@/components/consent/banner";
import { CONSENT_COOKIE, readConsent } from "@/lib/consent";

export default async function RootLayout({ children }: LayoutProps<"/">) {
  const consent = readConsent((await cookies()).get(CONSENT_COOKIE)?.value);

  return (
    <html lang="en">
      <body>
        {children}
        {consent?.categories.analytics ? (
          <Script src="/ingest/static/collector.js" strategy="lazyOnload" />
        ) : null}
        <ConsentBanner decided={consent !== null} />
      </body>
    </html>
  );
}
```

```tsx
// Wrong: the script loads, and a flag decides whether the events send.
// Failure: the vendor script runs before the reader answers. It writes its
// cookies, it reads the storage, and it opens a connection to the vendor origin.
// Article 5(3) covers that access, so the consent that arrives afterward
// records a thing that already happened.
<Script src="/ingest/static/collector.js" strategy="lazyOnload" />;

if (consent.analytics) {
  track("page_viewed", { path });
}
```

The gate is the render, and not the send. A script that never reaches the
document writes nothing, reads nothing, and opens no connection.

Read the record on the server, so the first document already holds the correct
markup. A record that the browser reads after hydration produces a banner that
appears on every visit for a moment, and the appearance moves the content.

A `cookies()` read makes the scope that holds it dynamic.
`references/caching-and-revalidation.md` owns that rule, and it states that the
read sits outside a `"use cache"` scope and behind `<Suspense>`.

`references/client-bundle-and-third-party-scripts.md` owns the strategy on the
tag, its measured cost, and its named owner. This file owns the condition in
front of it.

### The record carries a version, a time, and a scope

```ts
// src/lib/consent.ts
// Correct: the record states which policy the reader read, and when.
import { z } from "zod";

export const CONSENT_COOKIE = "app_consent";
export const POLICY_VERSION = "2026-08-19";

export const ConsentSchema = z.object({
  version: z.string(),
  decidedAt: z.iso.datetime(),
  categories: z.object({
    essential: z.literal(true),
    analytics: z.boolean(),
    marketing: z.boolean(),
  }),
});

export type Consent = z.infer<typeof ConsentSchema>;

export function readConsent(raw: string | undefined): Consent | null {
  if (!raw) return null;
  try {
    const parsed = ConsentSchema.safeParse(JSON.parse(raw));
    if (!parsed.success) return null;
    return parsed.data.version === POLICY_VERSION ? parsed.data : null;
  } catch {
    return null;
  }
}
```

```ts
// Wrong: a boolean in a store, and nothing on the device.
// Failure: the store is empty on the next visit, so the banner returns for a
// reader who answered it last week. The record also proves nothing to a
// regulator, because it holds no time and no policy version.
const [accepted, setAccepted] = useState(false);
```

Four fields make the record useful. The version names the policy text that the
reader read. The time states when they read it. The category map states the
scope of the answer. The `essential` field is `true` by construction, so a
future reader of the code cannot make it optional.

`readConsent` returns `null` where the version differs, so a new policy asks the
question again. Move `POLICY_VERSION` when the purposes change, and never when
the words change.

Store the record in a first-party cookie, and not in `localStorage`. The server
must read it to render the correct first document, and the server cannot read
`localStorage`.

`references/boundary-validation-and-api-types.md` owns the parse over a value
that enters the program. This file owns the shape that the parse proves.

### Reject costs the same as accept

| The rule | The failure that it prevents |
| --- | --- |
| The reject control and the accept control sit on one screen, at one level | A reject that hides in a second panel is not a free choice |
| Every non-essential control ships unchecked | A checked control records an agreement that nobody made |
| A persistent control reopens the choice from every page | A choice that the reader cannot change is not one that they can withdraw |
| The banner never covers the whole page with no way out | A reader who answers nothing must still reach the content |

```tsx
// Correct: two controls, one level, and no default agreement.
<form action={saveConsent}>
  <fieldset>
    <legend>Choose what this site may store</legend>
    <label>
      <input type="checkbox" name="analytics" value="on" />
      Product analytics
    </label>
    <label>
      <input type="checkbox" name="marketing" value="on" />
      Advertising
    </label>
  </fieldset>
  <button type="submit" name="decision" value="all">Accept all</button>
  <button type="submit" name="decision" value="none">Reject all</button>
  <button type="submit" name="decision" value="selected">Save my choice</button>
</form>
```

```tsx
// Wrong: one control agrees, and the refusal is a link into a second panel.
// Failure: the reader presses the large control because the other path costs
// three more presses. The record then holds an agreement that the reader did
// not freely give, and every category ships checked under it.
<input type="checkbox" name="analytics" defaultChecked />
<button>Accept all</button>
<a href="/cookie-settings">Manage preferences</a>
```

`references/interface-copy-and-voice.md` owns the words on each control and the
five patterns that take a decision from the reader. That file already states
that every consent control ships unchecked. This file owns the two controls at
one level, and the persistent control that reopens the choice.

`references/keyboard-focus-and-live-regions.md` owns the focus that enters the
banner, the key that closes it, and the return of focus afterward. That domain
holds a veto. A banner that traps a keyboard is a failed task.

### The signal arrives in the request

The Global Privacy Control travels as the `Sec-GPC` request header, with the
value `1`. Some jurisdictions treat it as a valid instruction, which tells the
product not to sell or share personal data. Read it, and let it decide the
default.

```ts
// proxy.ts
// Correct: the signal decides the default, and the banner records the rest.
import { NextResponse, type NextRequest } from "next/server";

export function proxy(request: NextRequest) {
  const response = NextResponse.next();
  if (request.headers.get("sec-gpc") === "1") {
    response.headers.set("x-privacy-signal", "opt-out");
  }
  return response;
}
```

```tsx
// Wrong: the banner asks, and the header goes unread.
// Failure: the reader set the signal in the browser, and the application asks
// them again. In a jurisdiction that recognizes the signal, an agreement that
// the banner then collects does not override it.
<ConsentBanner decided={false} />
```

Three rules hold. A `Sec-GPC` of `1` sets every non-essential category to
`false` by default. The reader may still turn a category on themselves, and that
press is the record. Do Not Track carries no such legal weight, and the research
names it as a signal to read, not to rely on.

CPRA also expects a "Do Not Sell or Share My Personal Information" control where
the product sells or shares personal data.
`references/data-rights-and-privacy-surfaces.md` owns the page that holds it.

`references/app-router-structure.md` owns `proxy.ts` and the work that it may
perform. `references/route-protection-and-permissions.md` states that `proxy.ts`
never authorizes. A privacy signal is a default, and never a gate.

### The tool decides the size of the question

| The tool | What it stores on the device | The question that it forces |
| --- | --- | --- |
| A cookieless, self-hosted counter | Nothing | None in many jurisdictions. No banner, and no third-party origin. |
| A self-hosted product analytics tool that sets a cookie | A first-party cookie | A banner, and one category |
| A hosted product analytics tool | A cookie, on a vendor origin | A banner, a category, a data controller, and a transfer question |
| An advertising tag, or a tag manager that loads one | A cookie, and an identifier that follows the reader | A banner, a category, and the advertising framework of that market |

Take the first row unless a stated product requirement rules it out. The choice
removes the banner, the vendor origin, the transfer question, and the loss that
a content blocker produces.

`references/event-taxonomy-and-tracking-plan.md` holds the vendor table and the
first-party path for the script and the collector. This file owns the consent
consequence of each choice.

WARNING: a tag manager is a channel through which somebody adds a tag after the
review, and that tag can carry its own storage.
`references/secret-boundary-and-supply-chain.md` owns the review over that
channel, and it holds a veto. The gate in this file covers the container, and it
cannot cover what somebody publishes into it.

### A session replay needs consent, and a mask beside it

A session replay records what a reader typed, read, and pressed. It is analytics,
never essential, and it never runs before the reader grants the category.

Two rules hold beside the gate. The recorder masks every text node and blocks
every media element. The project names each selector that holds a personal value,
and the recorder blocks that selector as well.

`references/error-capture-and-reporting.md` owns the mask over the Sentry
replay, and it states `maskAllText` and `blockAllMedia`. This file owns the
consent that must arrive before a replay starts at all.

### The banner reserves its space

A banner that appears after the first paint moves the content under it. The
metric records that move, and the reader loses their place.

Two rules keep the metric flat. The server decides whether the banner renders,
so no swap follows hydration. The banner sits in a container of a fixed height,
or it overlays the content and reserves nothing.

`references/paint-and-interaction-cost.md` owns the layout shift and the number
that it must meet. `references/performance-budgets-and-measurement.md` owns the
budget. This file owns the render decision that keeps the shift at zero.

### The inventory is a file, and an audit proves it

```ts
// src/lib/cookie-inventory.ts
// Correct: one list, and the policy page renders it.
export const COOKIE_INVENTORY = [
  { name: "__Host-session", purpose: "Keeps you signed in", days: 7, party: "first", category: "essential" },
  { name: "csrftoken", purpose: "Protects a form against forgery", days: 365, party: "first", category: "essential" },
  { name: "app_consent", purpose: "Keeps your privacy choice", days: 180, party: "first", category: "essential" },
  { name: "_analytics_id", purpose: "Counts visits to a page", days: 400, party: "first", category: "analytics" },
] as const;
```

```ts
// Wrong: the policy page holds the list as prose.
// Failure: a vendor change adds a cookie, and the page still names the old set.
// The page is a public claim, and the browser holds the evidence against it.
<p>We use a session cookie and Google Analytics.</p>
```

Audit the inventory against the browser, and never against the code alone. Load
the application with consent refused, and read `document.cookie` and the storage
panel. Every value that appears must sit in the essential category of the list.
Accept the categories, reload, and read both again. Every value that appears now
must sit in the list with the category that granted it.

Run that audit whenever a vendor changes, and whenever a release adds a script.

`references/data-rights-and-privacy-surfaces.md` owns the page that renders this
list.

### The libraries

The research names each consent product, and it compares a self-built layer
against them. It states no version, no release date, and no advisory record for
any of them. Those three columns hold `Not stated`. Read the registry entry and
the advisory database of a package before the project installs it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | A hand-written consent layer | The first choice where the product has two or three categories. It costs one cookie, one schema, one form, and one gate, and it adds no third-party script. | — | — | — | — |
| Recommend | `zod` | The parse over the record that the cookie holds. `references/boundary-validation-and-api-types.md` owns the module. | 4.4 | Current | Active | None |
| Conditional | Klaro | Only where the team wants an open-source manager that it can host itself. | Not stated | Not stated | Not stated | Not stated |
| Conditional | Cookiebot, Osano, or CookieYes | Only where a legal requirement names a certified platform, or where the site carries many third-party tags. Each one adds a script and a vendor. | Not stated | Not stated | Not stated | Not stated |
| Conditional | An IAB TCF implementation | Only where the product carries advertising that the framework covers. It is a large surface, and it needs the legal owner in the review. | Not stated | Not stated | Not stated | Not stated |
| Reject | A consent value in `localStorage` alone | The server cannot read it, so the first document is wrong and the banner appears again. | — | — | — | — |
| Reject | A banner that loads the tags and then hides the choice | The storage access already happened. The gate is the render, and not the send. | — | — | — | — |
| Reject | A pre-checked category | It records an agreement that nobody made. `references/interface-copy-and-voice.md` states the rule. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| A vendor cookie appears before the reader answers | The script renders, and a flag gates the send alone | Open a private window, refuse nothing, and read `document.cookie` | Render the tag behind the category |
| A request reaches a vendor origin with consent refused | A tag sits outside the gate, or a tag manager loaded one | Refuse every category, then filter the network panel by the vendor origin | Move the tag inside the gate, and review the container |
| The banner returns on every visit | The record lives in memory or in a store with no cookie | Answer the banner, close the tab, and return | Write the record to a first-party cookie |
| The banner returns after a release | `POLICY_VERSION` moved for a change of words | Compare the version in the code against the previous release | Move the version for a purpose change alone |
| The content jumps once on the first visit | The browser decides whether the banner renders | Load the route on a throttled profile | Read the record on the server |
| A keyboard cannot leave the banner | The dialog traps focus and offers no close | Press Tab and then Escape with no pointer | `references/keyboard-focus-and-live-regions.md` owns the pattern |
| The policy page names a cookie that the application never sets | The list is prose, and nobody audited it | Read `document.cookie` against the page | Render the page from the inventory, and audit both states |
| A reader with the signal set still sees the question | Nothing reads `sec-gpc` | Send one request with the header, and read the default | Read the header, and set the categories to `false` |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next.js 16 makes `cookies()` async | `rg -n 'cookies\(\)\.get' src/` reports a call with no `await` | `await cookies()`, which `references/app-router-structure.md` owns |
| Next.js 16 renames `middleware.ts` to `proxy.ts` | A `middleware.ts` file that reads `sec-gpc` | Rename the file and the export |
| Cache Components make everything dynamic until `"use cache"` | A layout that reads the consent cookie inside a cached scope | Move the read outside the scope, behind `<Suspense>` |
| Zod 4 moves the date-time check to `z.iso.datetime()` | `rg -n 'z\.string\(\)\.datetime\(' src/` reports a hit | Take `z.iso.datetime()` |

## Verification

```bash
# 1. Confirm that the record has a schema and a version. Expect both.
rg -n 'ConsentSchema|POLICY_VERSION' src/lib/consent.ts

# 2. Find a consent value that lives only in the browser. Each hit is a defect.
rg -n 'localStorage\.(set|get)Item\([^)]*consent' src/

# 3. Confirm that every vendor tag sits behind the gate. Read every hit.
rg -n -B4 '<Script' src/app src/components

# 4. Find a tag outside the gate. This prints nothing.
rg -l '<Script' -g '*.tsx' src/ \
  | xargs rg --files-without-match 'categories\.'

# 5. Confirm that a control ships unchecked. This prints nothing.
rg -n 'defaultChecked|checked=\{true\}' -g '*.tsx' src/components/consent

# 6. Confirm that the signal is read. Expect the header.
rg -n 'sec-gpc' proxy.ts src/

# 7. Read the inventory, and count the entries.
rg -c 'name:' src/lib/cookie-inventory.ts

# 8. Refuse every category, load the main route, and list the cookies. Every
#    name must sit in the essential category of the inventory.
#    In the browser console: document.cookie

# 9. Refuse every category, then filter the network panel by each vendor
#    origin. The count must be zero.

# 10. Accept every category, reload, and list the cookies and the storage keys.
#     Every name must sit in the inventory with the category that granted it.

# 11. Answer the banner, close the tab, and return. The banner must not appear.

# 12. Send one request with the signal, and read the rendered default.
curl -s -H 'Sec-GPC: 1' localhost:3000/ | rg -o 'name="analytics"[^>]*'
```

## Review checklist

- [ ] Does every non-essential script render only where the reader granted its
      category?
- [ ] Does the essential category hold the session, the CSRF token, and the
      consent record, and nothing else?
- [ ] Does the server read the record, so the first document is correct?
- [ ] Does the record hold a policy version, a time, and a category map?
- [ ] Does a first-party cookie hold the record?
- [ ] Does a version change ask the question again?
- [ ] Do the accept control and the reject control sit on one screen, at one
      level?
- [ ] Does every non-essential control ship unchecked?
- [ ] Does a persistent control reopen the choice from every page?
- [ ] Does the application read `Sec-GPC`, and does the value `1` set every
      non-essential category to `false`?
- [ ] Does a session replay wait for the analytics category, and does it mask
      every text node?
- [ ] Does the banner render with no layout shift?
- [ ] Does one file hold the cookie inventory, and does the policy page render
      it?
- [ ] Does an audit of the browser, in both states, agree with that file?
- [ ] Is a tag manager under a review process that names who may publish?

## Handoffs

- The event, the analytics module, the vendor table, the first-party path for the
  script and the collector, and the identity →
  `references/event-taxonomy-and-tracking-plan.md`.
- The policy page, the cookie policy page, the export, the deletion, and the
  "Do Not Sell or Share" control →
  `references/data-rights-and-privacy-surfaces.md`.
- The words on the two controls, and the five patterns that take a decision from
  the reader → `references/interface-copy-and-voice.md`.
- The focus that enters the banner, the key that closes it, and the return of
  focus → `references/keyboard-focus-and-live-regions.md`. That domain holds a
  veto. The contrast of the two controls, and their target size →
  `references/visual-and-motor-criteria.md`.
- The `next/script` strategy, the measured cost, and the named owner →
  `references/client-bundle-and-third-party-scripts.md`. The layout shift and its
  number → `references/paint-and-interaction-cost.md`.
- The review over a tag manager, the judgment of a vendor package, and the
  `integrity` attribute → `references/secret-boundary-and-supply-chain.md`. The
  `script-src` and `connect-src` entries →
  `references/security-headers-and-csp.md`. That domain holds a veto.
- The mask over a session replay, and the scrub before an error report leaves →
  `references/error-capture-and-reporting.md`.
- The session cookie, the CSRF cookie, and the attributes of each one →
  `references/session-and-token-lifecycle.md`.
- The parse over a cookie value, and the schema module →
  `references/boundary-validation-and-api-types.md`.
- `proxy.ts` as a route file, and the rule that it never authorizes →
  `references/app-router-structure.md` and
  `references/route-protection-and-permissions.md`.
- The dynamic scope that a `cookies()` read produces, and the `<Suspense>`
  boundary around it → `references/caching-and-revalidation.md`.
- The form, the resolver, and the Server Action that saves the answer →
  `references/form-schema-and-field-binding.md` and
  `references/suspense-and-actions.md`.
- The browser test that refuses every category and counts the vendor requests →
  `references/end-to-end-journeys-and-flake-control.md`.
- The consent record on the server, its retention, and the endpoint that
  receives it → the backend and `secure-code-auditor`. This file owns the record
  that the browser holds and the gate that reads it.
