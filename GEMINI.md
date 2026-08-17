# Gemini CLI context: frontend-production-engineer

This repo's frontend instructions live in `SKILL.md` and `references/`. Load
`SKILL.md` first, then the relevant `references/*.md` file. See `AGENTS.md`
for the full description. Do not duplicate the content here — read the source
files.

Coverage is production Next.js and TypeScript against a Django / Django REST
Framework backend. The subjects are routing and rendering, the React component
tree, and the typed contract with the backend. They also include the
non-functional guarantees a shipped frontend owes its users. The Django backend
itself is out of scope.

Domains are integrated one per release across four tiers. Tier 1 is the
foundations: App Router architecture, the type system, React component
architecture, and project structure and tooling. Tier 2 is the backend contract
and the state built on it: DRF contract and OpenAPI codegen, data fetching,
auth, and realtime. Tier 3 is interface craft: design system, accessibility,
forms, tables, media, motion, and UX writing. Tier 4 is the non-functional
guarantees: Web Vitals, frontend security, SEO, internationalization and RTL,
testing, observability, build and deploy, and analytics and consent. A Tier 0
operating doctrine sits under all four, and it is always in effect.

At 1.13.0 the integrated material in `references/` is that doctrine, the App
Router foundation, the type system, and the React component tree. It also holds
the project structure, the DRF contract, and the client cache and state. It
holds the session with the gates over it, and the push transport with the
events on it. It holds the design system and accessibility — the tokens and the
layout, the element and its name, the keyboard, and the measurable criteria.
It holds forms, the dense data surface, and media — the schema and the bind,
the table with the server that drives it, the chart, and the file that leaves
for a user.

The newest part is motion. It holds the animation with the properties that one
frame can afford, the transition between two views, and the drag and the scroll
that a reader drives. `SKILL.md` is the authoritative list of what is loadable.

Two standing rules govern everything. Verify the installed versions from
`package.json` before you generate code. Never mix Next 15 and Next 16 idioms
in one file, and report an unconfirmed API as unconfirmed. Apply the conflict
rule: security > accessibility > correctness > performance > developer
convenience. No level trades down to satisfy a level above it.

Primary integration is Claude; this file exists so Gemini CLI uses the same
single source of truth. Modes (review-time / write-time), the standing rules,
and the definition of done are in `SKILL.md`. The version is recorded in
`SKILL.md` frontmatter (`metadata.version`).
