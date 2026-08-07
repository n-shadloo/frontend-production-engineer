---
name: frontend-production-engineer
description: >-
  Production-grade frontend engineering for Next.js and TypeScript against a
  Django and Django REST Framework backend. Use whenever frontend work is
  planned, written, or reviewed in a Next.js repository and the task touches
  App Router routes, React components, next.config.ts, tsconfig.json,
  package.json, or a DRF endpoint being consumed, and the agent has to
  establish the installed versions, plan before editing, keep the diff
  minimal, and hold the result to a stated definition of done rather than
  guessing at an API, even if "frontend" is never used. Review-time audits
  existing code and reports what fails; write-time applies the defaults while
  generating code. Next.js 16 and React 19 are the pinned baseline; the router
  below lists the reference material currently integrated and grows one domain
  per release.
license: MIT
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
metadata:
  author: n-shadloo
  version: 1.0.0
---

# frontend-production-engineer

A frontend engineering skill. It plans, writes, and reviews production Next.js
and TypeScript against a Django / Django REST Framework backend, holding the
work to a stated definition of done instead of stopping at code that renders.
Scope is the browser-facing application: routing and rendering, the React
component tree, the typed contract with the backend, and the non-functional
guarantees a shipped frontend owes its users. It does not cover the Django
backend itself — server-side code, the ORM, the database, backend security,
and backend performance belong to the author's `secure-code-auditor` and
`django-performance-optimizer` skills, and this skill defers to them rather
than restating their material. Its canonical content is reused by other agents
(Codex, Cursor, Gemini CLI) via `AGENTS.md`, with Claude as the primary
integration.

## How the reference material is organized

Domain depth lives in `references/`, never in this file. Every reference file
has two layers:

1. **Principle** — the concern, why it matters, and the correct approach,
   stated framework-agnostically so it survives the next major version.
2. **Pinned-stack depth** — the concrete Next.js, React, TypeScript, and
   Django/DRF-contract implementation: runnable code, `// Wrong:` and
   `// Correct:` pairs naming the failure each wrong version produces, and a
   binary review checklist. This is where the depth lives.

Load only the file(s) relevant to the concern in front of you.

| Concern | Reference file |
|---|---|

The table is empty at 1.0.0. The operating doctrine is integrated but has no
row, because it is always in effect and lives in this file rather than in
`references/`. Each release integrates one domain and adds its row, written as
the trigger vocabulary that should route to it — the concrete nouns and verbs
a task might contain, plus the near-misses that belong to a neighbouring
domain — never as a summary of the file. A vague row is a domain that never
loads.

Cross-references between reference files are deliberate, and the seams are
recorded here as the domains land: which file owns a concern, and which files
defer to it rather than repeating it. No domain is integrated yet, so there
are no seams to state.

## Mode selection

Both modes are bound by the same standing rules. Plan before editing, and
state the success criteria the work will be checked against. Verify the
installed versions from `package.json`, `next.config.ts`, and `tsconfig.json`
before generating code — never mix Next 15 and Next 16 idioms in one file, and
never write against a version the repository does not have. Never invent an
API: if you cannot confirm that a function, prop, or option exists in the
installed version, say so instead of guessing. Keep the diff minimal, so every
changed line traces to the request. Run the commands rather than asserting
their result.

**Review-time.** Trigger when the user asks to review, audit, or "check"
existing frontend code; pastes a component and asks whether it is correct; or
has just finished a feature and wants it looked at. Behavior:

- Treat the codebase as **read-only**. Do not edit, refactor, or "fix in
  place" unless the user explicitly asks you to apply fixes afterward.
- Establish the installed versions first. A finding written against the wrong
  major version is noise.
- Investigate before flagging. Confirm the render path, the boundary the
  component actually sits on, and whether the code is reachable at runtime.
  Do not pattern-match a keyword into a finding.
- Report findings ordered by severity, each with a location, the concrete
  failure a user would experience, and a fix. End with what you did *not*
  review.

**Write-time.** Trigger when you are generating or modifying frontend code for
a feature. Behavior:

- Apply the defaults from the relevant reference file(s) as you write.
- Take types from the backend schema, never from imagination. A response shape
  you guessed is a bug with a delay on it.
- Note the choices a reviewer would want to see. If a requirement forces a
  pattern this skill would otherwise reject, say so and describe the residual
  risk rather than hiding it.
- Run the definition-of-done checks before reporting the work complete.

**If it is ambiguous,** apply write-time guardrails while coding and offer to
run a review afterward.

## Definition of done

Resolve domains in this order on any feature task: the standing rules above,
always; then the foundations, for where files go and how they are typed; then
the backend contract, before any code that touches Django; then whichever
domain files match the feature; then the non-functional domains, as a review
pass before done. At 1.0.0 only the standing rules are integrated, so the
order is a statement of intent that fills in as the router does.

Work is not done because it renders. It is done when all of the following
hold, each established by running the command rather than by inspection:

- The installed versions were verified before any code was written, and no
  file mixes idioms from two major versions.
- TypeScript compiles clean, with no suppression added to make it do so.
- Lint passes.
- The tests covering the change pass.
- The production build succeeds.
- The diff contains nothing the request did not ask for.

**Blocking domains.** Once integrated, `nextjs-app-router-architecture`,
`typescript-type-system-discipline`, `django-drf-api-contract`,
`authentication-and-authorization`, `accessibility-wcag`, `frontend-security`,
and `testing-and-quality` each hold a veto over completion. A task that fails
one of them is not complete, however finished the feature looks.
`frontend-security` and `accessibility-wcag` are absolute: a keyboard trap, an
interactive control with no accessible name, an unescaped injection sink, or a
secret that reaches the client is a failed task, not a follow-up ticket.

**Conflict rule.** security > accessibility > correctness > performance >
developer convenience. No level trades down to satisfy a level above it. When
a requirement appears to force such a trade, say so and stop rather than take
it silently.
