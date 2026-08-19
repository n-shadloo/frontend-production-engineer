# Version proof and unconfirmed code

Next.js 16.3, React 19.2.6 or later, TypeScript 5.9, pnpm 10.x. This file owns
the proof that the code can run. The subjects are the installed version, read
before any framework code, and the docs that the framework ships beside itself.
They also include the name that the installed version must carry, and the check
that finds it. The last subjects are the gate that a suppression turned green,
and the claim that states what was read.

The plan, the scope of a diff, and the summary that closes the work are
`references/task-plan-and-scope-control.md`. The instruction files, and the
reason that a bundled index outperforms a retrieval step, are
`references/instruction-files-and-skill-discovery.md`. The order of the pipeline
stages and the report of each command are
`references/merge-gates-and-quality-signals.md`. The suppression policy over
TypeScript is `references/typescript-config-and-enforcement.md`.

## Principle

A version is a fact in the repository. It is never a memory.

Two adjacent major versions of one framework share their names and not their
behavior. Code that mixes them compiles, and it fails at the first request.

A name that the installed version does not carry is not a small mistake. The
code cannot run at all, and the compiler reports it as a missing export rather
than as a wrong idea.

The cost of the check is one command. The cost of the guess is a build that
fails, or worse, a build that passes and a route that breaks.

A gate reports the state of the code. A suppression changes the report and not
the code, so the gate then reports the suppression.

A reader cannot tell a verified claim from a confident one. State which of the
two each claim is, and the reader stops having to guess.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### Read the installed version before you write framework code

Read three things, in this order. The declared range in `package.json`, the
resolved version in the lockfile, and the version that the package itself
reports.

```bash
# The declared range, which is a range and not a version.
rg -n '"next"|"react"|"typescript"' package.json

# The version that the install resolved. This is the fact.
node -p "require('next/package.json').version"
node -p "require('react/package.json').version"
```

The declared range is not the answer. A range of `^19.2.0` is satisfied by
`19.2.3`, which sits below the security floor of the line.
`references/secret-boundary-and-supply-chain.md` owns the range and the floor,
and it holds a veto.

Read the major and the minor. A minor release moves a default, adds a config
key, and deprecates a name, so the major alone does not settle an idiom.

| What you are about to write | Read this first | The domain that owns the idiom |
| --- | --- | --- |
| A route file, a `params` read, a cache scope, or `proxy.ts` | The installed Next.js minor | `references/app-router-structure.md` and `references/caching-and-revalidation.md` |
| A hook, a Server Action, or a React 19 API | The installed React minor | `references/component-composition.md` and `references/state-and-effects.md` |
| A compiler flag, a satisfies clause, or a type-level test | The installed TypeScript minor | `references/typescript-config-and-enforcement.md` |
| A query, a mutation, or a cache write | The installed TanStack Query minor | `references/server-state-and-query-cache.md` |
| A schema, a resolver, or a parse | The installed Zod major | `references/boundary-validation-and-api-types.md` |

Never mix the idioms of two major versions in one file. A file that holds both
teaches the next reader that both are correct.

### The docs that ship with the framework outrank recall

Next.js 16.2 and later write the documentation into the install. Read the area
that the change touches before you write the code.

```bash
# Confirm that the bundled docs are present, and list the areas.
ls node_modules/next/dist/docs/

# Read the area that the change touches.
rg -n 'cacheLife|cacheTag' node_modules/next/dist/docs/
```

These docs match the installed version by construction. Training data does not,
and the gap between them is where an invented name comes from.

Read the bundled docs first, then the types, and then this skill. Where the
bundled docs and this file disagree on a name, the bundled docs win, because
they ship with the version that the repository runs.

### Never write a name that the installed version does not carry

```ts
// Wrong: the export is recalled and never read.
// Failure: the build stops at "refreshTag is not exported from next/cache".
// The name is plausible, it appeared in a proposal, and it never shipped.
import { refreshTag } from "next/cache";

export async function publishPost(id: string) {
  await savePost(id);
  refreshTag("posts");
}
```

```ts
// Correct: the export is read from the installed types, and then written.
// Confirmed in node_modules/next/cache.d.ts and in the bundled docs.
import { revalidateTag } from "next/cache";

export async function publishPost(id: string) {
  await savePost(id);
  revalidateTag("posts");
}
```

Five kinds of name need the check, and each one fails the build or the run:

- An exported function or a hook.
- A prop, or a key inside an options object.
- A key in `next.config.ts`, `tsconfig.json`, or a plugin config.
- A flag on a command.
- An error string that a test asserts on.

```bash
# Read the exported names of a module, from the installed types.
rg -n 'export (declare )?(function|const|type)' node_modules/next/cache.d.ts

# Confirm a config key against the installed types.
rg -n 'cacheComponents|reactCompiler' node_modules/next/dist/**/*.d.ts

# Confirm a flag against the installed command.
npx next build --help
```

Where the check cannot confirm the name, do not write it. Say that the name is
unconfirmed, and name the alternative that the types do carry. An unconfirmed
name in a code sample is a defect, and never a suggestion.

### A gate that a suppression turned green is not a passed gate

```ts
// Wrong: the suppression makes the typecheck exit 0.
// Failure: the property name is wrong, so the call throws at run time on
// undefined. The gate now reports the suppression, and not the code.
// @ts-ignore
user.emial.toLowerCase();
```

```ts
// Correct: the cause is fixed, and the gate reports the code.
user.email.toLowerCase();
```

Four suppressions turn a gate green, and each one is a defect where this change
introduced it to pass that gate:

| The suppression | The gate that it silences | The domain that owns the policy |
| --- | --- | --- |
| `@ts-ignore`, or `@ts-expect-error` with no description | `pnpm typecheck` | `references/typescript-config-and-enforcement.md` |
| An `eslint-disable` comment with no reason | `pnpm lint` | `references/lint-format-and-scripts.md` |
| `it.skip`, `test.fixme`, or a spec that a condition excludes | `pnpm test` | `references/merge-gates-and-quality-signals.md` |
| `it.only`, which hides every other test | `pnpm test` | `references/merge-gates-and-quality-signals.md` |

Those files own whether a suppression may exist at all. This file owns the
narrower rule. A suppression that this change added to pass a gate is a failed
task, whatever the policy allows in general.

```bash
# Find a suppression that this change introduced. This prints nothing.
git diff | rg -n '^\+.*(@ts-ignore|@ts-expect-error|eslint-disable|\.only\(|\.skip\(|test\.fixme)'
```

A pre-existing suppression is a finding and never a deletion.
`references/task-plan-and-scope-control.md` owns that rule.

### A claim states what was read

```text
// Wrong: the claim carries no source.
// Failure: the reader cannot separate a read fact from a confident memory, so
// either every claim needs re-checking or none of them does.

  Cache Components need a cacheLife on every "use cache" scope.
```

```text
// Correct: each claim names what produced it.

  Confirmed: "use cache" takes cacheTag and cacheLife. Read in
  node_modules/next/dist/docs/ and in next/cache.d.ts, at next 16.3.1.

  Not confirmed: whether a nested scope inherits the outer cacheLife. I did
  not find it in the bundled docs. Declare it on the inner scope as well.
```

Use two marks, and never a third. A claim is confirmed against a file that you
read, or it is not confirmed. A confirmed claim names the file.

Three claims need the mark most, because a wrong one is expensive:

- A claim about what a version does or does not carry.
- A claim about a security property, such as a cookie attribute or a header.
- A claim that a command passed.

`references/merge-gates-and-quality-signals.md` owns the report that each
command produces. A pasted output is the strongest form of this mark, because
the reader reads the result rather than the claim.

### Version discipline

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next 16 renamed `middleware.ts` to `proxy.ts`, on the Node runtime | `ls middleware.ts` succeeds | `references/app-router-structure.md` owns the move and the permitted work |
| Next 16 made `params` and `searchParams` promises | `rg -n 'params: \{' -g '*.tsx' app/` reports a hit | `references/app-router-structure.md` owns the awaited read |
| Next 16 removed `next lint` | `rg -n 'next lint' package.json .github/` reports a hit | `references/lint-format-and-scripts.md` owns the direct `eslint` call |
| Next 16.2 bundles the docs into the install | `ls node_modules/next/dist/docs/` fails on an older minor | Upgrade, or read the docs of the installed version from another source |
| The React 19.2 line moved its security floor to 19.2.6 | `node -p "require('react/package.json').version"` reports a lower version | `references/secret-boundary-and-supply-chain.md` owns the floor. That domain holds a veto |
| CVE-2026-64642 of July 2026 is a second bypass of `proxy.ts` | `node -p "require('next/package.json').version"` reports a version below the patched minor | `references/route-protection-and-permissions.md` owns the layers that do not depend on that file |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| The build stops on a name that is not exported | The name came from recall | `rg` for the name in the installed `.d.ts` returns nothing | Read the exported names, and write one of them |
| A route works in development and fails in the production build | Two major-version idioms in one file | Compare the file against the installed minor | Read the version, and take the one idiom that it carries |
| A prop is accepted and does nothing | The prop belongs to a later minor | The prop is absent from the installed types | Remove it, and name the version that adds it |
| The typecheck exits 0 and the page throws | A suppression hid the error | `git diff` shows an added `@ts-ignore` | Remove the suppression, and fix the cause |
| The whole suite reports two passing tests | An `it.only` reached the branch | Compare the test count against the last run | Remove `.only`, and add the lint rule |
| Every claim in a review needs re-checking | No claim named its source | Read the answer for a named file beside each fact | Mark each claim confirmed or not confirmed |
| A team upgrades and the agent writes the old idiom | The version was read once and then assumed | The answer names no version | Read the version at the start of each task |

## Verification

```bash
# 1. Read the resolved versions. These are the facts, not the ranges.
node -p "require('next/package.json').version"
node -p "require('react/package.json').version"
node -p "require('typescript/package.json').version"

# 2. Confirm that the bundled docs are present, and read the touched area.
ls node_modules/next/dist/docs/

# 3. Confirm every framework name that this change writes.
rg -n 'export (declare )?(function|const|type)' node_modules/next/cache.d.ts

# 4. Find a Next 15 idiom in the route tree. This must print nothing.
rg -n 'middleware\.ts' package.json next.config.ts
rg -P '(?<!await )\b(params|searchParams)\.[a-zA-Z]' -g '*.tsx' app/

# 5. Find a suppression that this change introduced. This prints nothing.
git diff | rg -n '^\+.*(@ts-ignore|eslint-disable|\.only\(|\.skip\()'

# 6. Find a source file that names no version and asserts a version fact.
rg --files-without-match 'next 16|React 19' -g '*.md' docs/

# 7. Run the gates, and read the output of each one.
pnpm lint && pnpm typecheck && pnpm test && pnpm build

# 8. Read the route table of the build against the declared render modes.
#    references/caching-and-revalidation.md states the symbols.
```

## Review checklist

- [ ] Was the resolved version of Next.js, React, and TypeScript read before
      any framework code was written?
- [ ] Does the answer state each version that it read?
- [ ] Were the bundled docs under `node_modules/next/dist/docs/` read for the
      area that the change touches?
- [ ] Does every framework name in the change exist in the installed types?
- [ ] Is every config key confirmed against the installed types rather than
      recalled?
- [ ] Does no file mix the idioms of two major versions?
- [ ] Is every unconfirmed name reported as unconfirmed rather than written?
- [ ] Did this change add no suppression to make a gate pass?
- [ ] Is every pre-existing suppression reported rather than deleted?
- [ ] Does each claim carry either a named source or a statement that it is not
      confirmed?
- [ ] Does a claim that a command passed carry the output of that command?
- [ ] Is the installed React version at or above the floor that
      `references/secret-boundary-and-supply-chain.md` states?

## Handoffs

- The plan, the scope of a diff, the pre-existing orphan, and the closing
  summary → `references/task-plan-and-scope-control.md`.
- `AGENTS.md`, the managed Next.js block, the bundled index, and the audit of a
  third-party skill → `references/instruction-files-and-skill-discovery.md`.
- The awaited request data, `proxy.ts`, and the render mode of a route →
  `references/app-router-structure.md`.
- The cache scope, `cacheTag`, `cacheLife`, and the build report symbols →
  `references/caching-and-revalidation.md`.
- The suppression policy, `@ts-expect-error` with a description, and the
  type-level test → `references/typescript-config-and-enforcement.md`.
- The lint command, the `eslint-disable` reason, and the removed `next lint` →
  `references/lint-format-and-scripts.md`.
- The declared range, the resolved lockfile version, and the security floor →
  `references/secret-boundary-and-supply-chain.md`. That domain holds a veto.
- The pipeline order, the skipped test, and the report of each command →
  `references/merge-gates-and-quality-signals.md`.
- The React 19 APIs and the advisory that set the floor of the line →
  `references/state-and-effects.md`.
- The generated client, and the drift gate over the schema →
  `references/openapi-schema-and-codegen.md`.
- The layers that authorize a route, and the bypasses of `proxy.ts` →
  `references/route-protection-and-permissions.md`.
- The installed Django and DRF versions, and what the server carries → the
  backend. This file owns the frontend read only.
