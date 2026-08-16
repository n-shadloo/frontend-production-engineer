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
/ Django REST Framework backend. Scope is the browser-facing application:
routing and rendering, the React component tree, the typed contract with the
backend, and the non-functional guarantees a shipped frontend owes its users.
The Django backend itself is out of scope and defers to the author's
`secure-code-auditor` and `django-performance-optimizer` skills. Domains are
integrated one per release across four tiers — foundations, backend contract
and state, interface craft, and non-functional guarantees — over a Tier 0
operating doctrine that is always in effect. At 1.2.0 the integrated material
is that doctrine plus the App Router foundation, the type system, and the React
component tree in `references/`; `SKILL.md` is the authoritative list of what
is loadable.

## Two modes
- Review-time: audit existing frontend code, produce findings ordered by
  severity (location, the concrete failure a user would experience, fix), and
  state what was not reviewed. Read-only by default.
- Write-time: plan first, verify the installed versions, apply the defaults
  from the integrated domains while generating code, and note the choices a
  reviewer would want to see.
Ambiguous cases take write-time guardrails, with a review offered afterward.

## How to use the content
1. Read `SKILL.md` for the router, the standing rules, the mode logic, and the
   definition of done.
2. Open the `references/*.md` file(s) for the concern in front of you (the
   table in `SKILL.md` maps concern → file).
3. Verify the installed versions from `package.json`, `next.config.ts`, and
   `tsconfig.json` before generating code. Never mix Next 15 and Next 16
   idioms in one file, and never claim an API exists without confirming it.
4. Run the definition-of-done checks — typecheck, lint, tests, production
   build — before reporting work complete. Code that renders is not done.

Conflict rule: security > accessibility > correctness > performance >
developer convenience. No level trades down to satisfy a level above it.

## Tool-specific entry points
- Claude Code: `SKILL.md` (native Agent Skill).
- OpenAI Codex CLI: reads this `AGENTS.md`.
- Cursor: `.cursor/rules/frontend-production-engineer.mdc`.
- Gemini CLI: `GEMINI.md`.
All of them defer to `SKILL.md` and `references/`.
