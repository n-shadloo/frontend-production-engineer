# AGENTS.md

This repository is a frontend engineering skill. Its canonical instructions
live in `SKILL.md`, which routes to the domain files under `references/`. Any
agent working in this repo should load `SKILL.md` first and then read only the
`references/*.md` file(s) relevant to the task.

Primary integration: **Claude** (Anthropic Agent Skills). The files below let
other agents use the same content; they are pointers, not copies. If anything
here disagrees with `SKILL.md`, `SKILL.md` wins. The current version is
recorded in `SKILL.md` frontmatter (`metadata.version`).

## What this skill does
Plans, writes, and reviews production Next.js and TypeScript against a Django
/ Django REST Framework backend. Scope is the browser-facing application. That
scope is routing and rendering, the React component tree, and the typed
contract with the backend. It also holds the non-functional guarantees a
shipped frontend owes its users. The Django backend itself is out of scope, and
it defers to the author's `secure-code-auditor` and
`django-performance-optimizer` skills.

Domains are integrated one per release across four tiers. The tiers are
foundations, backend contract and state, interface craft, and non-functional
guarantees. A Tier 0 operating doctrine sits under them, and it is always in
effect.

At 1.22.0 the integrated material in `references/` is that doctrine, the
App Router foundation, the type system, and the React component tree. It also
holds the project structure, the DRF contract, and the client cache and state.
It holds the session with the gates over it, and the push transport with the
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

The next part is the proof. It holds the level that each test belongs to,
and the component test that carries most of the value. It also holds the answer
that a mock gives to a request, and the journey that a real browser runs. The
last of it is the gate that a change passes before a merge.

Another part is what happens when something breaks. It holds the report that
a failure sends, and the personal value that the report must never carry. It
also holds the identifier that joins one screen to one Django log line, and the
trace that crosses to the backend. The last of it is the application under an
outage. That part is the gate over a dead backend, the offline state, and the
probe that answers for the chain.

Another part is the release. It holds the artifact that a build produces, and
the image that carries it. It also holds the process that serves it, and the
front door in front of that process. The last of it is the pipeline, the deploy
that a probe proves, and the way back.

The newest part is the measurement, and the permission that comes before it. It
holds the event, its name, and the one module that sends it. It also holds the
consent that a script waits for, the record that keeps the answer, and the
inventory that proves the cookie policy true. The last of it is the export, the
deletion, and the retention window that the interface states. `SKILL.md` is the
authoritative list of what is loadable.

## Two modes
- Review-time: audit existing frontend code, produce findings ordered by
  severity (location, the concrete failure a user would experience, fix), and
  state what was not reviewed. Read-only by default.
- Write-time: plan first, verify the installed versions, and apply the defaults
  from the integrated domains as you write. Note the choices a reviewer would
  want to see.
Ambiguous cases take write-time guardrails, with a review offered afterward.

## How to use the content
1. Read `SKILL.md` for the router, the standing rules, the mode logic, and the
   definition of done.
2. Open the `references/*.md` file(s) for the concern in front of you (the
   table in `SKILL.md` maps concern → file).
3. Verify the installed versions from `package.json`, `next.config.ts`, and
   `tsconfig.json` before you generate code. Never mix Next 15 and Next 16
   idioms in one file. Never claim that an API exists unless you confirm it.
4. Run the definition-of-done checks before you report the work complete. The
   checks are the typecheck, the lint, the tests, and the production build.
   Code that renders is not done.

Conflict rule: security > accessibility > correctness > performance >
developer convenience. No level trades down to satisfy a level above it.

## Tool-specific entry points
- Claude Code: `SKILL.md` (native Agent Skill).
- OpenAI Codex CLI: reads this `AGENTS.md`.
- Cursor: `.cursor/rules/frontend-production-engineer.mdc`.
- Gemini CLI: `GEMINI.md`.
All of them defer to `SKILL.md` and `references/`.
