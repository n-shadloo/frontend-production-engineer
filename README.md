# frontend-production-engineer

A Claude Agent Skill for frontend engineering. It plans, writes, and reviews
production Next.js and TypeScript against a Django / Django REST Framework
backend. The deep specialty is the pinned stack: Next.js 16, React 19,
TypeScript 5, Tailwind v4, and a DRF contract that a generated OpenAPI client
consumes. A portable principle layer sits under that stack, and it survives the
next major version. It covers the browser-facing application only; the Django
backend belongs to the author's other skills.

## Why this exists

A frontend is easy to get to the point where it renders and hard to get to the
point where it ships. Five things separate the two. They are the render
boundary a component actually sits on, and types that come from the backend
schema rather than a guess. The others are a focus order a keyboard user can
follow, and a bundle that does not regress INP. The last is a build that
survives a self-hosted deploy.

The work is well understood. Framework docs, specs, and release notes hold it,
and those sources move every few months. An agent that works without it writes
code that looks right and fails in production.

This skill packages that knowledge so an agent applies it the same way every
time. It is one skill, not twenty-four. `SKILL.md` is the spine and the router.
Every domain lives in `references/`, and it loads only when the task calls for
it. Each reference file has two layers — a framework-agnostic statement of the
concern, then the pinned-stack implementation with runnable code and a binary
review checklist.

Two things are non-negotiable in it. **Never invent an API.** The skill
verifies the installed versions from `package.json` before it writes any code.
It reports an unconfirmed function, prop, or option as unconfirmed, and it
never guesses. **The conflict rule:** security > accessibility > correctness >
performance > developer convenience, and no level trades down to satisfy a
level above it.

## What it covers

Domains are integrated one per release. The roster below is the full plan, and
each entry carries its current status. The router table in `SKILL.md` is the
authoritative list of what is loadable today.

Tier 0 is the operating discipline, in effect on every task.

- `24 agent-operating-doctrine` — how the agent plans, verifies the installed
  versions, and refuses to guess at an API. It also keeps the diff minimal,
  runs the commands, and decides the work is done. Integrated.

Tier 1 is the foundations: where files go and how they are typed.

- `01 nextjs-app-router-architecture` — routing, rendering, caching, the
  server/client boundary, `proxy.ts`. Integrated.
- `02 typescript-type-system-discipline` — type safety, generics, `tsconfig`,
  the no-`any` doctrine. Integrated.
- `03 react-component-architecture` — composition, hooks, the React 19 APIs,
  render correctness. Integrated.
- `04 project-structure-and-tooling` — folders, ESLint and Prettier, Git,
  monorepos, dependencies. Integrated.

Tier 2 is the backend contract and the state built on it.

- `05 django-drf-api-contract` — OpenAPI codegen, DRF error and pagination
  shapes, CSRF and CORS. Integrated.
- `06 data-fetching-and-state` — TanStack Query, cache design, client state,
  URL state. Integrated.
- `07 authentication-and-authorization` — session and token strategy, route
  protection, RBAC in the UI. Integrated.
- `08 realtime-and-streaming` — WebSockets and Channels, server-sent events,
  optimistic concurrency. Integrated.

Tier 3 is interface craft.

- `09 design-system-and-styling` — tokens, Tailwind v4, shadcn/ui, theming,
  layout. Integrated.
- `10 accessibility-wcag` — WCAG 2.2 AA, the ARIA authoring practices,
  keyboard, focus, screen readers. Integrated.
- `11 forms-and-validation` — React Hook Form with Zod, React 19 Actions,
  server-error mapping, multi-step flows. Integrated.
- `12 data-tables-and-visualization` — TanStack Table, server-driven tables,
  charts. Integrated.
- `13 media-and-file-handling` — uploads, images, video, downloads, progress.
  Integrated.
- `14 motion-and-interaction` — Motion and View Transitions, gestures, the
  performance cost of animation. Integrated.
- `15 ux-writing-and-content-design` — microcopy, error and empty states,
  voice, information architecture. Integrated.

Tier 4 is the non-functional guarantees, applied as a review pass before done.

- `16 performance-and-web-vitals` — INP, LCP, CLS, bundles, budgets,
  measurement. Integrated.
- `17 frontend-security` — XSS, CSP, response headers, server action safety,
  supply chain. Integrated.
- `18 seo-and-metadata` — the Metadata API, JSON-LD, sitemaps, canonical URLs,
  Open Graph. Integrated.
- `19 internationalization-and-rtl` — next-intl, locale routing, RTL and
  Persian, the `Intl` APIs. Integrated.
- `20 testing-and-quality` — Vitest, React Testing Library, MSW, Playwright,
  contract tests. Integrated.
- `21 observability-and-resilience` — error boundaries, Sentry, real-user
  monitoring, graceful degradation. Integrated.
- `22 build-deploy-and-runtime-ops` — Docker, standalone output, Nginx, CI/CD,
  self-hosting. Pending.
- `23 analytics-privacy-and-consent` — events, consent gating, GDPR,
  third-party scripts. Pending.

Six further domains follow the same pipeline once the twenty-four are in. They
are commerce and checkout, the admin shell, chat UI, maps, PDF and print
output, and email templates.

Seven of these are blocking. `nextjs-app-router-architecture`,
`typescript-type-system-discipline`, `django-drf-api-contract`,
`authentication-and-authorization`, `accessibility-wcag`, `frontend-security`,
and `testing-and-quality` each hold a veto over completion. All seven are
integrated. The last two are absolute. A task that fails accessibility or
security is a failed task, and never a follow-up ticket.

The stack baseline is pinned, and it was verified in August 2026. The framework
is Next.js 16.3, with Turbopack by default and `proxy.ts` in place of
`middleware.ts` on the Node runtime. That release also makes `params` and
`searchParams` async only, adds Cache Components, and removes `next lint`. The
runtime is Node.js 20.9 or later. The UI layer is React 19.2.6 or later, which
is the security floor of the 19.2 line, with React Compiler 1.0. The language is TypeScript 5.9, with `strict` and
`noUncheckedIndexedAccess`.

Tailwind CSS v4.3 supplies the styling, with the CSS-first `@theme` config, and
shadcn/ui sits on Base UI. TanStack Query 5.101 or later holds the server
state, nuqs 2.9 holds the URL state, and Zustand 5 holds the client store. Zod
4.4 validates the values. React Hook Form 7.85 binds the forms, with
`@hookform/resolvers` 5.9 between them. TanStack Table 9.1 models the data
tables, over `@tanstack/react-virtual` 3.14 for a long list, and recharts 3.10
draws the charts. Vitest, React Testing Library, MSW, and Playwright run the
tests.

`web-vitals` 6.1 reports the field metrics. `@lhci/cli` 0.15 and `size-limit`
gate the budget in CI. `@sentry/nextjs` 10.70 captures the errors, on a floor
of 10.27 for the advisory GHSA-6465-jgvq-jhgp. `pino` writes the structured
log, `@vercel/otel` registers the tracer, and `react-error-boundary` covers a
widget inside a segment.

The backend is Django
and DRF 3.17, and it publishes OpenAPI 3.0.3 through drf-spectacular 0.30.
`openapi-typescript` 7.13 generates the types, `openapi-fetch` 0.17 sends the
request, and `oasdiff` gates a breaking change in the schema.
`djangorestframework-simplejwt` 5.5.1 issues and rotates the tokens, and
`django-allauth` in headless mode covers registration, social login, and MFA.
Django Channels 4.3 carries a WebSocket, and `partysocket` 1.3 wraps it in the
browser.

pnpm manages the packages, on the 10.x line while the Node floor is 20.9.
ESLint 9 with the flat config and Prettier 3 run the checks. lefthook holds the
git hooks, and commitlint reads the messages.

The baseline is what the reference material is written against. It is not an
assumption about your repository. The skill verifies the installed version from
`package.json` before it generates any code. It never mixes Next 15 and Next 16
idioms in one file.

## Install

The repository is a plain-Markdown Agent Skill. The canonical instructions
live in the root `SKILL.md`, which routes to the files under `references/`.
Claude reads the skill directly. Cursor and OpenAI Codex CLI reuse the same
canonical content through their native discovery mechanisms, while Gemini CLI
reads a `GEMINI.md` context file. Nothing needs to be built; there are no
dependencies beyond `git`.

### Claude

One project:

```bash
git clone https://github.com/n-shadloo/frontend-production-engineer.git \
  .claude/skills/frontend-production-engineer
```

All your projects:

```bash
git clone https://github.com/n-shadloo/frontend-production-engineer.git \
  ~/.claude/skills/frontend-production-engineer
```

For claude.ai or the API, upload the folder as a custom skill in Settings.

### Codex CLI

Codex CLI discovers Agent Skills from the `.agents/skills/` directory and uses
the bundled pointer skill to load the canonical `SKILL.md`.

One project:

```bash
git clone https://github.com/n-shadloo/frontend-production-engineer.git \
  .agents/skills/frontend-production-engineer
```

All your projects:

```bash
git clone https://github.com/n-shadloo/frontend-production-engineer.git \
  ~/.agents/skills/frontend-production-engineer
```

`AGENTS.md` provides project-wide context, while the pointer skill forwards to
the canonical instructions in the repository root.

### Cursor

Cursor natively supports Agent Skills, so the same repository works:

```bash
git clone https://github.com/n-shadloo/frontend-production-engineer.git \
  .cursor/skills/frontend-production-engineer
```

The included `.cursor/rules/frontend-production-engineer.mdc` file is optional
reinforcement that points back to the canonical `SKILL.md`.

### Gemini CLI

Gemini CLI doesn't read Agent Skills directly; it reads `GEMINI.md`.

- **Per project:** copy `GEMINI.md` into the repository root.
- **All projects:** copy it to `~/.gemini/GEMINI.md`.

`GEMINI.md` points Gemini to the canonical `SKILL.md` and `references/`, and it
does not duplicate the content.

The only requirement is `git` and a Git repository to run in.

## Use

Two modes, chosen from context.

Review existing frontend code — ask for a review, or point it at a component:

```
Review this dashboard route before we ship it.
```

It establishes the installed versions first. Then it reports the findings,
ordered by severity. Each finding carries a location, the concrete failure a
user would experience, and a fix. It ends with a statement of what it did not
review.

Write new code. The skill plans first, states the success criteria, and
verifies the installed versions. It then applies the defaults from the
integrated domains, and it runs the checks:

```
Add a paginated orders table backed by the DRF orders endpoint.
```

It reports the choices a reviewer would want to see. It never calls the work
done until the typecheck, the lint, the tests, and the production build have
run.

## Example output

At 1.20.0 the integrated material is the operating doctrine, the App Router
foundation, the type system, and the React component tree. It also holds the
project structure, the backend contract, and the client cache and state. It
holds the session with the gates over it, and the push transport with the
events on it. It holds the design system and accessibility — the tokens and the
layout, the element and its name, the keyboard, and the measurable criteria.

It holds forms, the dense data surface, media, and motion. That part is the
schema and the bind, and the table with the server that drives it. It also
holds the file that leaves for a user, and the animation over all of them.

It holds the copy — the words on a control, the message that a failure
produces, and the string as data behind a key.

It holds speed as a measured property — the thresholds, the budget, and the
gate over them. It also holds the bytes of JavaScript, with the third-party
script beside them. The last part is the largest paint, the layout shift,
and the answer to a tap.

It holds the browser-side threat model, and that part holds a veto. It is the
sink that turns data into code, and the sanitiser in front of it. It also holds
the policy and the header set that the response carries. The endpoint that the
network reaches, and the destination that the server reaches back, are in it
too. The last of
it is what must never cross to the browser, and the code that arrives there
that nobody on the team wrote.

One part is what a machine reads. It is the title, the canonical link,
and the card that a share preview renders. It also holds the claim that a page
makes in machine-readable form, and the escape that the grammar of that block
needs.

The last part is the crawl and the index. It holds the robots rules, and the
sitemap that the DRF data produces. It also holds the status code that a
missing record returns, and the redirect that a rename ships.

Another part is the second language. It holds the route that carries a
locale, and the file that holds the catalog behind it. It also holds the date,
the number, and the calendar that each locale writes in its own form. The last
of it is the direction of a surface, and the script that a font must carry.

The newest part is what happens when something breaks. It holds the report that
a failure sends, and the personal value that the report must never carry. It
also holds the identifier that joins one screen to one Django log line, and the
trace that crosses to the backend. The last of it is the application under an
outage — the gate over a dead backend, the offline state, and the probe that
answers for the chain.

The next part is the proof. It holds the level that each test belongs to,
and the component test that carries most of the value. It also holds the answer
that a mock gives to a request, and the journey that a real browser runs. The
last of it is the gate that a change passes before a merge.

The worked example below is the shape of a task, and the facts that gate
it:

```
Plan
- Goal: orders table, server-paginated, at /orders.
- Success: typecheck + lint + tests + build clean; page size and cursor
  survive a reload; empty and error states render.
- Steps: (1) generate types from the DRF schema, (2) query hook,
  (3) table, (4) tests for empty / error / page-2.

Versions verified (package.json)
- next 16.3.1, react 19.2.7, typescript 5.9.2, @tanstack/react-query 5.104.0
- Async params apply; proxy.ts, not middleware.ts.

Unconfirmed
- No generated client in the repo. Step 1 runs api:generate against the
  committed schema.yml. No response shape is assumed.

Route facts (/orders)
- Render mode: dynamic. Data source: the DAL, server-side.
- Cache strategy: none, the rows are per-user. Invalidation: updateTag.
- "use client" on the sort control only, with the reason above it.

Type facts
- Shape from the generated components["schemas"]["PaginatedOrderList"].
- Totals parsed at the boundary; no cast on response.json().
- OrderId branded. Row state is a union, not three optional booleans.

Component facts
- State lives in the sort control; the table stays a Server Component.
- Rows keyed by order id, never by index.
- Section <Suspense> around the table, with a row-shaped skeleton.

Structure facts
- Route at src/app/(app)/orders; the table in src/features/orders.
- Types from src/api/generated, which is gitignored and regenerated.
- No new dependency; the sort control uses the existing ui primitive.

Contract facts
- Schema from the committed schema.yml; COMPONENT_SPLIT_REQUEST is True.
- One openapi-fetch client; /api/orders/ keeps the trailing slash.
- Timeout of 10 s, combined with the caller signal on every request.
- Page 2 follows the next URL; no offset is computed.
- A 400 becomes ApiError.fieldErrors; the GET retries, a POST never does.

State facts
- Server state in the Query cache; one ordersListOptions in features/orders.
- Key from the factory, carrying the search term and the page.
- Prefetch on the server, one HydrationBoundary, and no refetch on mount.
- q and page live in the URL through nuqs; the cache key derives from them.

Auth facts
- verifySession() in the DAL, memoised with cache(); the page calls it.
- proxy.ts redirects on the cookie alone; /orders verifies on the server.
- Route stays dynamic; no order row enters a "use cache" scope.
- The export action re-authenticates itself and takes no userId argument.

Live facts
- No connection opens. Nobody watches one order change second by second.
- A 30 s refetchInterval with a stop condition covers the totals.
- The pull request carries that reason in one line.

Accessibility facts
- The sort control is a button; its icon is decorative, and a span names it.
- The table carries a caption, and every th carries a scope.
- Focus moves to the h1 after a navigation; the row count announces polite.
- Contrast measured in both themes; the status column carries an icon and a word.
- axe clean in the component test, and on /orders in Playwright.
- Keyboard walkthrough recorded; NVDA with Firefox ran the primary flow.

Test facts
- Criteria written before the code: rows, empty, error, page 2, sort, axe.
- Component tests for the four states; MSW answers, and never a stubbed client.
- One Playwright journey; the route axe scan is its own spec.
- Clock frozen and TZ=UTC, so the created_at column reads the same in CI.
- No waitForTimeout anywhere; every assertion waits on the locator.

Table facts
- manualPagination, manualSorting, and manualFiltering are all set.
- rowCount comes from the DRF count; a cursor endpoint would drop page numbers.
- getRowId returns the order id, so a selection survives a sort.
- Columns at module scope; the currency formatter is built once, with a locale.
- 25 rows to a page, so no virtualiser. The export sends the filter, not the page.

Done
- pnpm lint              clean, at --max-warnings=0
- pnpm typecheck         clean
- pnpm test              14 passed
- pnpm build             succeeded
- Diff: 4 files, all traceable to the request.
```

## Notes

`SKILL.md` frontmatter records the skill version in `metadata.version`, and git
holds the release tags. The version tracks the integrated domains, and never a
brief number. **1.0.0** is the scaffold and the first domain, released
together. Each domain after the first adds one to the minor number. The
doctrine of domain 24 has no router entry, because `SKILL.md` holds it. The
minor number is therefore the count of router domains minus one, and
**1.22.0** is all twenty-four domains.

A patch release corrects material that is already integrated. Domains land in
any order, so read the router rather than the version string.

The reference material is a strong, current baseline, not a guarantee. The
stack moves; verify the installed version before you trust a pinned-stack
example, and treat the principle layer as the part with the longer half-life.

## Layout

```text
frontend-production-engineer/
├── SKILL.md                                   # canonical skill and router
├── AGENTS.md                                  # always-on project context
├── GEMINI.md                                  # Gemini CLI context
├── .cursor/
│   └── rules/
│       └── frontend-production-engineer.mdc   # Cursor reinforcement rule
├── references/                                # domain depth, one release each
│   ├── api-client-and-request-safety.md
│   ├── app-router-structure.md
│   ├── bidirectional-layout-and-scripts.md
│   ├── boundary-validation-and-api-types.md
│   ├── caching-and-revalidation.md
│   ├── cell-formatting-and-export.md
│   ├── charts-and-visual-encoding.md
│   ├── client-and-url-state.md
│   ├── client-bundle-and-third-party-scripts.md
│   ├── component-composition.md
│   ├── component-styles-and-variants.md
│   ├── correlation-and-telemetry.md
│   ├── crawl-and-index-control.md
│   ├── cross-origin-and-bff-proxy.md
│   ├── data-access-and-mutations.md
│   ├── data-table-and-server-driven-state.md
│   ├── degradation-and-health-checks.md
│   ├── dependencies-and-git-workflow.md
│   ├── design-tokens-and-theming.md
│   ├── directory-and-module-boundaries.md
│   ├── end-to-end-journeys-and-flake-control.md
│   ├── error-and-empty-state-copy.md
│   ├── error-capture-and-reporting.md
│   ├── exposed-endpoints-and-destinations.md
│   ├── file-upload-and-transport.md
│   ├── form-schema-and-field-binding.md
│   ├── form-submission-and-server-errors.md
│   ├── gesture-and-scroll-interaction.md
│   ├── image-and-video-delivery.md
│   ├── interface-copy-and-voice.md
│   ├── keyboard-focus-and-live-regions.md
│   ├── layout-and-typography.md
│   ├── lint-format-and-scripts.md
│   ├── live-events-and-cache-merge.md
│   ├── locale-formatting-and-calendars.md
│   ├── locale-routing-and-catalogs.md
│   ├── merge-gates-and-quality-signals.md
│   ├── message-catalog-and-plurals.md
│   ├── motion-primitives-and-reduced-motion.md
│   ├── multi-step-forms-and-unsaved-work.md
│   ├── network-mocks-and-contract-tests.md
│   ├── openapi-schema-and-codegen.md
│   ├── paint-and-interaction-cost.md
│   ├── performance-budgets-and-measurement.md
│   ├── push-transport-and-connection.md
│   ├── route-metadata-and-social-cards.md
│   ├── route-protection-and-permissions.md
│   ├── secret-boundary-and-supply-chain.md
│   ├── security-headers-and-csp.md
│   ├── semantics-and-accessible-names.md
│   ├── served-content-and-downloads.md
│   ├── server-and-client-components.md
│   ├── server-state-and-query-cache.md
│   ├── session-and-token-lifecycle.md
│   ├── state-and-effects.md
│   ├── structured-data-and-rich-results.md
│   ├── suspense-and-actions.md
│   ├── test-strategy-and-component-tests.md
│   ├── type-modeling-and-narrowing.md
│   ├── typescript-config-and-enforcement.md
│   ├── untrusted-markup-and-injection.md
│   ├── view-transitions-and-animation-libraries.md
│   ├── visual-and-motor-criteria.md
│   └── wcag-conformance-and-verification.md
├── README.md
├── LICENSE
└── .gitignore
```

`references/` holds twenty-one domains at 1.20.0 and fills one domain at a
time.
`scripts/` and `assets/` are not present; they are added when a domain ships
something executable or a template worth copying.

## License

MIT. See `LICENSE`.
