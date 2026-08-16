# frontend-production-engineer

A Claude Agent Skill for frontend engineering. It plans, writes, and reviews
production Next.js and TypeScript against a Django / Django REST Framework
backend. The deep specialty is the pinned stack — Next.js 16, React 19,
TypeScript 5, Tailwind v4, and a DRF contract consumed through a generated
OpenAPI client — over a portable principle layer that survives the next major
version. It covers the browser-facing application only; the Django backend
belongs to the author's other skills.

## Why this exists

A frontend is easy to get to the point where it renders and hard to get to the
point where it ships. The work that separates the two — the render boundary a
component actually sits on, types that come from the backend schema instead of
a guess, a focus order a keyboard user can follow, a bundle that does not
regress INP, a build that survives a self-hosted deploy — is well understood,
but it is spread across framework docs, specs, and release notes that move
every few months. An agent working without it writes code that looks right and
fails in production.

This skill packages that knowledge so an agent applies it the same way every
time. It is one skill, not twenty-four: `SKILL.md` is the spine and router,
and every domain lives in `references/`, loaded only when the task calls for
it. Each reference file has two layers — a framework-agnostic statement of the
concern, then the pinned-stack implementation with runnable code and a binary
review checklist.

Two things are non-negotiable in it. **Never invent an API:** the installed
versions are verified from `package.json` before any code is written, and an
unconfirmed function, prop, or option is reported as unconfirmed rather than
guessed. **The conflict rule:** security > accessibility > correctness >
performance > developer convenience, and no level trades down to satisfy a
level above it.

## What it covers

Domains are integrated one per release. The roster below is the full plan,
each entry marked with its current status; the router table in `SKILL.md` is
the authoritative list of what is loadable today.

Tier 0 is the operating discipline, in effect on every task.

- `24 agent-operating-doctrine` — how the agent plans, verifies the installed
  versions, refuses to guess at an API, keeps the diff minimal, runs the
  commands, and decides the work is done. Integrated.

Tier 1 is the foundations: where files go and how they are typed.

- `01 nextjs-app-router-architecture` — routing, rendering, caching, the
  server/client boundary, `proxy.ts`. Integrated.
- `02 typescript-type-system-discipline` — type safety, generics, `tsconfig`,
  the no-`any` doctrine. Pending.
- `03 react-component-architecture` — composition, hooks, the React 19 APIs,
  render correctness. Pending.
- `04 project-structure-and-tooling` — folders, ESLint and Prettier, Git,
  monorepos, dependencies. Pending.

Tier 2 is the backend contract and the state built on it.

- `05 django-drf-api-contract` — OpenAPI codegen, DRF error and pagination
  shapes, CSRF and CORS. Pending.
- `06 data-fetching-and-state` — TanStack Query, cache design, client state,
  URL state. Pending.
- `07 authentication-and-authorization` — session and token strategy, route
  protection, RBAC in the UI. Pending.
- `08 realtime-and-streaming` — WebSockets and Channels, server-sent events,
  optimistic concurrency. Pending.

Tier 3 is interface craft.

- `09 design-system-and-styling` — tokens, Tailwind v4, shadcn/ui, theming,
  layout. Pending.
- `10 accessibility-wcag` — WCAG 2.2 AA, the ARIA authoring practices,
  keyboard, focus, screen readers. Pending.
- `11 forms-and-validation` — React Hook Form with Zod, React 19 Actions,
  server-error mapping, multi-step flows. Pending.
- `12 data-tables-and-visualization` — TanStack Table, server-driven tables,
  charts. Pending.
- `13 media-and-file-handling` — uploads, images, video, downloads, progress.
  Pending.
- `14 motion-and-interaction` — Motion and View Transitions, gestures, the
  performance cost of animation. Pending.
- `15 ux-writing-and-content-design` — microcopy, error and empty states,
  voice, information architecture. Pending.

Tier 4 is the non-functional guarantees, applied as a review pass before done.

- `16 performance-and-web-vitals` — INP, LCP, CLS, bundles, budgets,
  measurement. Pending.
- `17 frontend-security` — XSS, CSP, response headers, server action safety,
  supply chain. Pending.
- `18 seo-and-metadata` — the Metadata API, JSON-LD, sitemaps, canonical URLs,
  Open Graph. Pending.
- `19 internationalization-and-rtl` — next-intl, locale routing, RTL and
  Persian, the `Intl` APIs. Pending.
- `20 testing-and-quality` — Vitest, React Testing Library, MSW, Playwright,
  contract tests. Pending.
- `21 observability-and-resilience` — error boundaries, Sentry, real-user
  monitoring, graceful degradation. Pending.
- `22 build-deploy-and-runtime-ops` — Docker, standalone output, Nginx, CI/CD,
  self-hosting. Pending.
- `23 analytics-privacy-and-consent` — events, consent gating, GDPR,
  third-party scripts. Pending.

Six further domains — commerce and checkout, the admin shell, chat UI, maps,
PDF and print output, and email templates — follow the same pipeline once the
twenty-four are in.

Seven of these are blocking. `nextjs-app-router-architecture`,
`typescript-type-system-discipline`, `django-drf-api-contract`,
`authentication-and-authorization`, `accessibility-wcag`, `frontend-security`,
and `testing-and-quality` each hold a veto over completion once integrated,
and the last two are absolute: a task that fails accessibility or security is
a failed task, not a follow-up ticket.

Stack baseline is pinned: Next.js 16.3 (Turbopack by default, `middleware.ts`
replaced by `proxy.ts` on the Node runtime, async-only `params` and
`searchParams`, Cache Components, `next lint` removed), Node.js 20.9 or later,
React 19.2.x, React Compiler 1.0, TypeScript 5.x with `strict` and
`noUncheckedIndexedAccess`, Tailwind CSS v4 with CSS-first `@theme` config,
shadcn/ui on Base UI, TanStack Query 5.101 or later, Zod 4, React Hook Form,
Vitest with React Testing Library, MSW, and Playwright, against a Django and
DRF backend that publishes OpenAPI 3.0.3 through drf-spectacular (verified
August 2026). The baseline is what the reference material is written against,
not an assumption about your repository: the installed version is verified
from `package.json` before any code is generated, and Next 15 and Next 16
idioms are never mixed in one file.

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

`GEMINI.md` points Gemini to the canonical `SKILL.md` and `references/`
instead of duplicating the content.

The only requirement is `git` and a Git repository to run in.

## Use

Two modes, chosen from context.

Review existing frontend code — ask for a review, or point it at a component:

```
Review this dashboard route before we ship it.
```

You get the installed versions established first, then findings ordered by
severity, each with a location, the concrete failure a user would experience,
and a fix — followed by a statement of what was not reviewed.

Write new code — it plans first, states the success criteria, verifies the
installed versions, applies the defaults from the integrated domains, and runs
the checks:

```
Add a paginated orders table backed by the DRF orders endpoint.
```

It reports the choices a reviewer would want to see, and it does not call the
work done until typecheck, lint, tests, and the production build have actually
been run.

## Example output

At 1.0.0 the integrated material is the operating doctrine and the App Router
foundation. The worked example is the shape of a task and the route facts that
gate it:

```
Plan
- Goal: orders table, server-paginated, at /orders.
- Success: typecheck + lint + tests + build clean; page size and cursor
  survive a reload; empty and error states render.
- Steps: (1) generate types from the DRF schema, (2) query hook,
  (3) table, (4) tests for empty / error / page-2.

Versions verified (package.json)
- next 16.3.1, react 19.2.0, typescript 5.9.2, @tanstack/react-query 5.104.0
- Async params apply; proxy.ts, not middleware.ts.

Unconfirmed
- No generated client in the repo. The response shape is taken from the
  drf-spectacular schema at /api/schema/, not assumed.

Route facts (/orders)
- Render mode: dynamic. Data source: the DAL, server-side.
- Cache strategy: none, the rows are per-user. Invalidation: updateTag.
- "use client" on the sort control only, with the reason above it.

Done
- tsc --noEmit           clean
- eslint .               clean
- vitest run             14 passed
- next build             succeeded
- Diff: 4 files, all traceable to the request.
```

## Notes

The skill version is recorded in `SKILL.md` frontmatter (`metadata.version`);
releases are tagged in git. Versioning tracks integrated domains rather than
brief numbers: **1.0.0** is the scaffold plus the first domain, released
together; each domain after the first is one minor bump, so the minor number
is the count of integrated domains minus one; **1.23.0** is all twenty-four
in; patch releases are corrections to already-integrated material. Domains may
be integrated in any order — count the router rows, not the version string.

The reference material is a strong, current baseline, not a guarantee. The
stack moves; verify the installed version before trusting a pinned-stack
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
│   ├── app-router-structure.md
│   ├── caching-and-revalidation.md
│   ├── data-access-and-mutations.md
│   └── server-and-client-components.md
├── README.md
├── LICENSE
└── .gitignore
```

`references/` holds the first domain at 1.0.0 and fills one domain at a time.
`scripts/` and `assets/` are not present; they are added when a domain ships
something executable or a template worth copying.

## License

MIT. See `LICENSE`.
