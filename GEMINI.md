# Gemini CLI context: frontend-production-engineer

This repo's frontend instructions live in `SKILL.md` and `references/`. Load
`SKILL.md` first, then the relevant `references/*.md` file. See `AGENTS.md`
for the full description. Do not duplicate the content here — read the source
files.

Coverage is production Next.js and TypeScript against a Django / Django REST
Framework backend: routing and rendering, the React component tree, the typed
contract with the backend, and the non-functional guarantees a shipped
frontend owes its users. Domains are integrated one per release across four
tiers — foundations (App Router architecture, the type system, React component
architecture, project structure and tooling), the backend contract and the
state built on it (DRF contract and OpenAPI codegen, data fetching, auth,
realtime), interface craft (design system, accessibility, forms, tables,
media, motion, UX writing), and the non-functional guarantees (Web Vitals,
frontend security, SEO, internationalization and RTL, testing, observability,
build and deploy, analytics and consent) — over a Tier 0 operating doctrine
that is always in effect. At 1.2.0 the integrated material is that doctrine
plus the App Router foundation, the type system, and the React component tree
in `references/`; `SKILL.md` is the authoritative list of what is loadable. The
Django backend itself is out of scope.

Two standing rules govern everything. Verify the installed versions from
`package.json` before generating code, never mix Next 15 and Next 16 idioms in
one file, and report an unconfirmed API as unconfirmed rather than guessing.
And apply the conflict rule: security > accessibility > correctness >
performance > developer convenience, with no level trading down to satisfy a
level above it.

Primary integration is Claude; this file exists so Gemini CLI uses the same
single source of truth. Modes (review-time / write-time), the standing rules,
and the definition of done are in `SKILL.md`. The version is recorded in
`SKILL.md` frontmatter (`metadata.version`).
