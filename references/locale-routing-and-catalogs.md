# Locale routing and catalogs

`next-intl` 4.13.6, Next.js 16.3, React 19.2.6 or later, TypeScript 5.9,
against a Django and DRF backend. This file owns the locale as a property of
the request. The subjects are the segment that carries the locale, the proxy
that resolves it, and the config that loads a catalog. They also include the
provider boundary, the static render over a locale route, and the switcher that
keeps the current view. The last subjects are the typed key set, and the split
between a string that this application owns and a string that Django owns.

The key itself, the ICU body behind it, and the plural rule are
`references/message-catalog-and-plurals.md`. The formatter that renders a date,
a number, or a name is `references/locale-formatting-and-calendars.md`. The
direction of the document is `references/bidirectional-layout-and-scripts.md`.

## Principle

A locale is a property of a request. The setting of a browser is one input to
that property, and it is never the answer by itself.

A URL that does not carry the locale answers one address with two documents. A
reader cannot share it, a cache cannot key it, and a crawler indexes one of the
two.

A catalog that loads every locale at once sends each reader the languages that
the reader cannot read.

A key that no build checks reaches production as an empty label, or as the key
itself.

A product with one language pays a small discipline today. A product that adds
the second language later pays an edit in every component that holds a string.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark.

### The locale sits in the path

```ts
// Wrong: a cookie alone carries the locale.
// Failure: one URL answers with two documents. A reader who shares the link
// sends the language of the reader, and not the language of the page. A CDN
// serves the first response to the next reader. A crawler indexes one
// document, and the second language never enters the index.
const locale = cookies().get("locale")?.value ?? "en";
```

```ts
// Correct: src/i18n/routing.ts declares the set, and the path carries it.
import { defineRouting } from "next-intl/routing";

export const routing = defineRouting({
  locales: ["en", "fa"],
  defaultLocale: "en",
  localePrefix: "always",
});
```

```ts
// Correct: src/proxy.ts resolves the locale before the router runs.
import createMiddleware from "next-intl/middleware";
import { routing } from "@/i18n/routing";

export const proxy = createMiddleware(routing);

export const config = {
  matcher: ["/((?!api|_next|.*\\..*).*)"],
};
```

Next 16 renamed `middleware.ts` to `proxy.ts`, and the export is now `proxy`.
A project that upgrades from Next 15 with the file name unchanged loses the
locale resolution in silence. Every request then falls to the default locale.
`references/app-router-structure.md` owns that file and the rename.

Take `localePrefix: "always"`, so each locale holds its own prefix. The
`"as-needed"` option leaves the default locale with a bare path, which gives
one route two URLs. Take it only where an existing product already ships those
URLs.

Put every route under `app/[locale]/`. The segment is a route parameter, so
`await params` reads it.

### The request config returns the locale and the messages

```ts
// Wrong: the config returns the requested value with no test.
// Failure: a request for /de-DE loads no catalog, and the render throws
// MODULE_NOT_FOUND on the server. A crafted segment reads a path outside the
// messages folder.
export default getRequestConfig(async ({ requestLocale }) => {
  const locale = await requestLocale;
  return { locale, messages: (await import(`./messages/${locale}.json`)).default };
});
```

```ts
// Correct: src/i18n/request.ts proves the value against the declared set.
import { getRequestConfig } from "next-intl/server";
import { hasLocale } from "next-intl";
import { routing } from "@/i18n/routing";

export default getRequestConfig(async ({ requestLocale }) => {
  const requested = await requestLocale;
  const locale = hasLocale(routing.locales, requested)
    ? requested
    : routing.defaultLocale;

  return {
    locale,
    messages: (await import(`../../messages/${locale}.json`)).default,
    timeZone: "Europe/London",
  };
});
```

The `locale` field is mandatory in `next-intl` 4. A config that omits it fails
the render.

State `timeZone` in the config. A formatter with no zone renders the zone of
the machine, so the server and the browser disagree.
`references/locale-formatting-and-calendars.md` owns that rule.

`hasLocale` is a type guard. It narrows `string` to the union of the declared
locales, so the compiler rejects a locale that the routing file does not
declare.

### The provider takes the client subtree, and not the tree

```tsx
// Wrong: the root layout sends every message to the browser.
// Failure: the whole catalog reaches the client bundle for every route. A
// catalog of 900 keys costs the reader about 60 kB of JSON on the first
// paint, and the reader uses the keys of one screen.
const messages = await getMessages();
return (
  <NextIntlClientProvider messages={messages}>{children}</NextIntlClientProvider>
);
```

```tsx
// Correct: app/[locale]/layout.tsx sends the namespaces that the client needs.
import { NextIntlClientProvider, hasLocale } from "next-intl";
import { setRequestLocale, getMessages } from "next-intl/server";
import { notFound } from "next/navigation";
import { routing } from "@/i18n/routing";

export function generateStaticParams() {
  return routing.locales.map((locale) => ({ locale }));
}

export default async function LocaleLayout(props: LayoutProps<'/[locale]'>) {
  const { locale } = await props.params;
  if (!hasLocale(routing.locales, locale)) notFound();
  setRequestLocale(locale);

  const messages = await getMessages();
  return (
    <html lang={locale} dir={getLangDir(locale)}>
      <body>
        <NextIntlClientProvider messages={{ nav: messages.nav }}>
          {props.children}
        </NextIntlClientProvider>
      </body>
    </html>
  );
}
```

A Server Component reads `getTranslations`, and it sends no message to the
browser. A Client Component reads `useTranslations`, and it needs the provider
above it. Prefer the server hook. Move the provider down to the subtree that
holds the interactive parts.

`references/server-and-client-components.md` owns the boundary itself, and
`references/client-bundle-and-third-party-scripts.md` owns the byte budget over
this decision.

`references/semantics-and-accessible-names.md` owns the `lang` attribute, and
that domain holds a veto. This file owns the value that the resolved locale
supplies to it. `references/bidirectional-layout-and-scripts.md` owns `dir` and
the `getLangDir` helper behind it.

### `setRequestLocale` keeps a locale route static

```tsx
// Wrong: the page reads the locale and renders nothing else that is dynamic.
// Failure: the build report marks every locale route as dynamic. Each request
// renders on the server, and the first byte carries the render every time.
export default async function Page(props: PageProps<'/[locale]'>) {
  const t = await getTranslations("home");
  return <h1>{t("title")}</h1>;
}
```

```tsx
// Correct: the page declares the locale before it reads a message.
import { setRequestLocale, getTranslations } from "next-intl/server";

export default async function Page(props: PageProps<'/[locale]'>) {
  const { locale } = await props.params;
  setRequestLocale(locale);

  const t = await getTranslations("home");
  return <h1>{t("title")}</h1>;
}
```

Call `setRequestLocale` in every layout and in every page of the locale tree.
The call is what lets the renderer resolve a message with no request.

Pair it with `generateStaticParams` over `routing.locales`. The build then
emits one static document for each locale.

Read the build report and confirm the mode. A route that reads a cookie or a
session is dynamic for that reason, and no locale call changes it.
`references/caching-and-revalidation.md` owns the lifetime of the response, and
`references/app-router-structure.md` owns the render mode.

### The negotiation reads a header and a cookie

The proxy performs the negotiation for a route under `[locale]`. Where a Route
Handler or a server module must resolve a locale by itself, take the matcher of
the standard and never a hand-written split.

```ts
// Wrong: the code takes the first tag of the header.
// Failure: "Accept-Language: fr;q=0.2, en;q=0.9" resolves to fr, which the
// reader ranked lowest. A tag of "en-GB" matches no entry in a locale set
// that holds "en", so the reader falls to the default.
const locale = request.headers.get("accept-language")?.split(",")[0] ?? "en";
```

```ts
// Correct: the matcher of the standard reads the quality values.
import { match } from "@formatjs/intl-localematcher";
import Negotiator from "negotiator";
import { routing } from "@/i18n/routing";

export function resolveLocale(headers: Record<string, string>) {
  const requested = new Negotiator({ headers }).languages();
  return match(requested, [...routing.locales], routing.defaultLocale);
}
```

An explicit choice outranks the header. Write that choice to a cookie, and read
the cookie before the header. A reader who selects a language keeps it on the
next visit.

The rule that no redirect reads a network address is
`references/crawl-and-index-control.md`, which also owns the one URL for each
locale.

### The switcher keeps the path and the query

```tsx
// Wrong: the switcher navigates to the root of the other locale.
// Failure: a reader on /en/orders/8821?status=open presses the switcher and
// lands on /fa. The record, the filter, and the scroll position are gone, and
// the reader restarts the task.
<a href={`/${next}`}>{next}</a>
```

```tsx
// Correct: src/i18n/navigation.ts produces the locale-aware primitives.
import { createNavigation } from "next-intl/navigation";
import { routing } from "@/i18n/routing";

export const { Link, redirect, usePathname, useRouter, getPathname } =
  createNavigation(routing);
```

```tsx
// Correct: the switcher replaces the locale and keeps everything else.
"use client"; // it reads the current path and the current query

import { useSearchParams } from "next/navigation";
import { usePathname, useRouter } from "@/i18n/navigation";

export function LocaleSwitcher({ locales }: { locales: readonly string[] }) {
  const pathname = usePathname();
  const params = useSearchParams();
  const router = useRouter();
  const query = Object.fromEntries(params.entries());

  return (
    <>
      {locales.map((locale) => (
        <button
          key={locale}
          type="button"
          lang={locale}
          onClick={() => router.replace({ pathname, query }, { locale })}
        >
          {new Intl.DisplayNames([locale], { type: "language" }).of(locale)}
        </button>
      ))}
    </>
  );
}
```

The `usePathname` of `next-intl` returns the path with no locale prefix, so the
value goes straight back into a navigation for another locale.

Write the name of each language in that language. A reader who cannot read the
current interface must still find the entry. The `lang` attribute on each
control tells a screen reader which voice to use.

Never switch the locale inside a form with unsaved values. Warn first.
`references/multi-step-forms-and-unsaved-work.md` owns that guard.

`references/client-and-url-state.md` owns the search parameter itself, and it
requires a `<Suspense>` boundary above a `useSearchParams` consumer.

### The build fails on a missing key

```ts
// Correct: src/global.d.ts binds the message shape to the default catalog.
import type messages from "../messages/en.json";

declare module "next-intl" {
  interface AppConfig {
    Messages: typeof messages;
    Locale: (typeof import("@/i18n/routing").routing.locales)[number];
  }
}
```

The compiler then rejects `t("titel")` and reports the key that no catalog
holds. Run `tsc --noEmit` in CI, so a renamed key cannot reach production.
`references/typescript-config-and-enforcement.md` owns that gate.

A translated catalog lags the default catalog. Declare a fallback chain, and
never render a raw key. A key on the screen is a defect that the reader sees.
Add a visible marker in development, so you find it before a reader does.

### Django owns the content string

| The string | The owner | How it reaches the reader |
| --- | --- | --- |
| A label, a button, a heading, a validation message, an empty state | This application | A catalog key, in `messages/<locale>.json` |
| A record field that a person authored, such as a product name or an article body | Django | A serializer field, in the response |
| An enumeration that both sides render, such as an order status | This application | The API returns the stable code, and the catalog holds the word for it |
| A message that only the server can produce, such as a mail body | Django | The server renders it, and the client never formats it |

```ts
// Wrong: the interface renders the string that the server sent.
// Failure: the status word arrives in the language of the Django default. A
// reader in Persian reads "Shipped" inside a Persian sentence, and no
// translator can reach the word.
<span>{order.status_display}</span>
```

```ts
// Correct: the API returns the code, and the catalog holds the word.
<span>{t(`status.${order.status}`)}</span>
// status.shipped = "Shipped"
```

Send `Accept-Language` on a request whose response holds content that a person
authored. The header carries the resolved locale, and never the raw header of
the browser. `references/api-client-and-request-safety.md` owns the client that
sets it.

A serializer that returns a display string for an enumeration is a contract
decision, and the sibling skill `django-api-contract` owns it. State the need
for the stable code, and never write the serializer here.

### One language today

A product with one locale still holds every rule above, at almost no cost. Four
of them are the whole discipline:

1. Every user-facing string sits behind a catalog key.
2. Every date and every number passes through an `Intl` formatter with a stated
   locale and a stated time zone.
3. Every component uses a logical CSS property, and no physical one.
4. Every count takes an ICU plural.

A project that holds these four adds the second locale in a week. A project
that holds none of them edits every component, and it finds the layout defects
one screen at a time.

Declare the routing file and the `[locale]` segment on the first day. The
segment costs one folder, and it saves the URL migration later.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry supplied
those facts on 18 August 2026. A cell that holds no date is a package with a
current registry entry on that date. This file does not state an exact release
date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `next-intl` 4.13.6 | The default on this stack. It supports the App Router, the Server Components, and ICU with no second runtime. | 4.13.6 | 10 Aug 2026 | Active, on a weekly cadence | None |
| Recommend | `@formatjs/intl-localematcher` | The locale match of the standard, for a path that the proxy does not cover. | Current | Current | Active | None |
| Recommend | `negotiator` | It parses the quality values of `Accept-Language`. Take it with the matcher. | Current | Current | Active | None |
| Conditional | Paraglide with `inlang` | Only where the message bytes are a measured problem. It compiles each message to a function, so the bundler removes what no route uses. The translator workflow needs its own editor. | Current | Current | Active | None |
| Conditional | `@lingui/core` 6.6.0 | Only where the project already holds Lingui catalogs. It compiles the catalog, and it carries a macro build step. | 6.6.0 | Current | Active | None |
| Audit only | `i18next` with `react-i18next` | Alive in a project that standardised on it. Its Server Component support needs an extra adapter. This stack takes `next-intl`. | 26.3.6, and 17.0.11 | Current | Active | None |
| Audit only | `react-intl` with FormatJS | Alive where the team already runs the FormatJS extraction tools. `@formatjs/cli` works beside `next-intl`, so the tooling does not force the runtime. | Current | Current | Active | None |
| Audit only | `next-translate` | Alive in a Pages Router project. It has no Server Component story on this baseline. | Current | Current | Active | None |
| Reject | `next-i18next` | It is the Pages Router binding of `i18next`, and the App Router is out of its scope. | Current | Current | Active | — |
| Reject | A locale in a cookie with no segment in the path | One URL answers with two documents, and no cache and no crawler can tell them apart. | — | — | — | — |
| Reject | A second ICU runtime beside `next-intl` | Two parsers read one message format, and the bundle pays for both. | — | — | — | — |

`references/message-catalog-and-plurals.md` owns `@formatjs/cli` and
`eslint-plugin-formatjs`. Do not add a second row for either one.

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| Every request falls to the default locale after a Next 16 upgrade | The file is still `middleware.ts` | Read the file name at the root of `src/` | Rename it to `proxy.ts`, and export `proxy` |
| The render throws `MODULE_NOT_FOUND` for a catalog | The config imports the requested value with no test | Request a locale that the routing file does not declare | Narrow with `hasLocale` before the import |
| The first paint carries the whole catalog | The root layout passes every message to the provider | Read the JSON payload of the first document | Pass the namespaces that the client subtree reads |
| Every locale route is dynamic in the build report | No `setRequestLocale` call runs before the first message | Read the build report | Call it in each layout and each page of the locale tree |
| The switcher loses the record and the filter | The control navigates to the root of the other locale | Switch the locale on a detail route that carries a query | Replace the locale, and keep the path and the query |
| A key renders on the screen | The catalog of that locale lacks the key | Render every route in the second locale | Declare the fallback chain, and fail the build on a missing key |
| A status word arrives in one language | The serializer returns the display string | Read the response beside the rendered word | Return the stable code, and hold the word in the catalog |
| A reader ranked English first and reads French | The code took the first tag of the header | Send a header with quality values | Take the matcher of the standard |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next 16 renamed `middleware.ts` to `proxy.ts`, and the export is `proxy` | A `middleware.ts` at the root of `src/`, with `createMiddleware` inside it | Rename the file, and export the result as `proxy` |
| `next-intl` 4 requires the `locale` field from `getRequestConfig` | A config that returns `messages` alone | Return the narrowed locale beside the messages |
| `next-intl` 4 replaced `createSharedPathnamesNavigation` and `createLocalizedPathnamesNavigation` with `createNavigation` | Either older factory in `src/i18n/` | Take `createNavigation(routing)`, which reads the routing object |
| `next-intl` 4 declares the message shape through `AppConfig` | A `global.d.ts` that augments `IntlMessages` | Declare `Messages` and `Locale` on the `AppConfig` interface |

## Verification

```bash
# 1. Read the installed versions before you write code.
cat package.json | rg '"next"|"next-intl"|"react"'

# 2. Confirm that the proxy file carries the Next 16 name. This prints the
#    file, and any middleware.ts hit is the stale name.
rg --files src/ | rg 'proxy\.ts|middleware\.ts'

# 3. Find a locale read from a cookie with no segment behind it. Read every
#    hit.
rg -n 'cookies\(\)[^;]*locale|get\("locale"\)' -g '*.ts' -g '*.tsx' src/

# 4. Confirm that the request config narrows the value before the import.
rg -n -A6 'getRequestConfig' src/i18n/request.ts | rg 'hasLocale'

# 5. Find a layout or a page under the locale tree that calls no
#    setRequestLocale. Read every hit.
rg --files-without-match 'setRequestLocale' \
  -g 'src/app/\[locale\]/**/{layout,page}.tsx' src/

# 6. Find a provider that passes every message. Read every hit.
rg -n -B2 -A2 'NextIntlClientProvider' -g '*.tsx' src/ | rg 'getMessages\(\)'

# 7. Find a switcher that navigates to a bare locale path. Read every hit.
rg -n 'href=\{?[`"]/\$\{?locale' -g '*.tsx' src/

# 8. Find a rendered display string from the server. Read every hit.
rg -n '_display|Display\}' -g '*.tsx' src/

# 9. Confirm that the compiler holds the catalog shape.
pnpm exec tsc --noEmit

# 10. Request each locale, and read the status and the language of the
#     document.
curl -sSI "$APP_ORIGIN/en" | rg 'HTTP/|content-language'
curl -sSI "$APP_ORIGIN/fa" | rg 'HTTP/|content-language'

# 11. Read the build report, and confirm the mode of each locale route.
pnpm build

# 12. Switch the locale on a detail route that carries a query. The record and
#     the filter survive the switch.
```

## Review checklist

- [ ] Does the path carry the locale for every route that a reader shares?
- [ ] Does one routing file declare the locale set and the default?
- [ ] Is the proxy file named `proxy.ts`, with a `proxy` export?
- [ ] Does the request config narrow the requested locale before it imports a
      catalog?
- [ ] Does the request config return a `locale` field and a `timeZone` field?
- [ ] Does every layout and every page in the locale tree call
      `setRequestLocale`?
- [ ] Does `generateStaticParams` enumerate the declared locales?
- [ ] Does a Server Component read `getTranslations` rather than the client
      hook?
- [ ] Does the provider receive only the namespaces that the client subtree
      reads?
- [ ] Does the switcher keep the current path and the current query?
- [ ] Does the switcher write each language name in that language?
- [ ] Does an explicit locale choice persist in a cookie, and outrank the
      header?
- [ ] Does a hand-written negotiation read the quality values of the header?
- [ ] Does the compiler hold the message shape, so a wrong key fails the build?
- [ ] Does a fallback chain cover a key that a translated catalog lacks?
- [ ] Does the API return a stable code for every enumeration that the
      interface renders?

## Handoffs

- The catalog key, the ICU body, the plural rule, the list format, and the
  pseudo-locale → `references/message-catalog-and-plurals.md`.
- The words inside a message, and the voice →
  `references/interface-copy-and-voice.md`. The message after a failure →
  `references/error-and-empty-state-copy.md`.
- The date, the number, the calendar, and the digits →
  `references/locale-formatting-and-calendars.md`.
- The direction of the document, the mirrored layout, and the non-Latin font →
  `references/bidirectional-layout-and-scripts.md`.
- The route files, the folder tokens, and `proxy.ts` as a file →
  `references/app-router-structure.md`. The render mode and the cache scope →
  `references/caching-and-revalidation.md`.
- The `"use client"` boundary and the provider that needs it →
  `references/server-and-client-components.md`. The bytes that the boundary
  sends → `references/client-bundle-and-third-party-scripts.md`.
- The search parameter, and the `<Suspense>` boundary above a
  `useSearchParams` consumer → `references/client-and-url-state.md`.
- The `lang` attribute of the document and of a passage →
  `references/semantics-and-accessible-names.md`. That domain holds a veto.
- The `hreflang` link set, `x-default`, and the canonical link →
  `references/route-metadata-and-social-cards.md`. One URL for each locale, and
  the rule that no redirect reads a network address →
  `references/crawl-and-index-control.md`.
- The compiler gate that a wrong key fails →
  `references/typescript-config-and-enforcement.md`. The lint config array →
  `references/lint-format-and-scripts.md`.
- The request that carries `Accept-Language`, and its timeout →
  `references/api-client-and-request-safety.md`.
- The warning before a navigation discards unsaved values →
  `references/multi-step-forms-and-unsaved-work.md`.
- The test that renders each locale →
  `references/test-strategy-and-component-tests.md`.
- The serializer that returns a stable code for an enumeration, and the
  translated model field behind it → the sibling skill `django-api-contract`.
