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
effect. At 1.5.0 the integrated material is that doctrine, the App Router
foundation, the type system, the React component tree, the project structure,
the DRF contract, and the client cache and state in `references/`. `SKILL.md`
is the authoritative list of what is loadable.

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
