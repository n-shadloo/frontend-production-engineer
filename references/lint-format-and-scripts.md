# Lint, format, and scripts

ESLint 9 flat config, Prettier 3, `eslint-config-next` for the Next 16 line,
typescript-eslint v8, pnpm 10.x. This file owns the checks that run over the
code and the commands that run them. The subjects are the `package.json`
scripts, the flat config array, the formatter choice, and the rule for a lint
disable. They also include the file that an agent reads before it works in the
repository.

The layout that the boundary rules describe is
`references/directory-and-module-boundaries.md`. The package manager and the
git hooks that call these commands are
`references/dependencies-and-git-workflow.md`. The type-aware rules inside the
config array are `references/typescript-config-and-enforcement.md`.

## Principle

One job has one command. A contributor who must remember a flag runs the wrong
command, and an agent invents one.

A warning is a failure with a delay on it. A gate that accepts a warning
accumulates warnings until the output carries no signal.

The format of the code is not a review subject. A tool decides it, the same way
every time, and the review discusses the behavior.

A suppression is a claim that the rule is wrong here. State the claim, or fix
the code.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### One command for one job

```jsonc
// package.json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint . --max-warnings=0",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write .",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:e2e": "playwright test",
    "api:generate": "openapi-typescript schema.yml -o src/api/generated/schema.d.ts",
    "analyze": "ANALYZE=true next build",
    "check": "pnpm lint && pnpm typecheck && pnpm test && pnpm build"
  }
}
```

`check` is the command that ends a task. Run it before you report the work
complete. It runs the four gates in the order that fails fastest and cheapest
first.

This file owns the presence of `test`, `test:watch`, and `test:e2e`. Domain 20
`testing-and-quality` owns the test layout, the fixtures, and the coverage
threshold.

### Zero warnings

```jsonc
// Wrong: package.json accepts a warning.
// Failure: warnings accumulate, because nothing forces anyone to clear one.
// The list grows past the screen, a real report joins the noise, and the
// command exits 0 while the codebase gets worse.
{ "scripts": { "lint": "eslint ." } }
```

```jsonc
// Correct: a warning exits non-zero.
{
  "scripts": {
    "lint": "eslint . --max-warnings=0",
    "lint:fix": "eslint . --fix"
  }
}
```

Every invocation carries `--max-warnings=0`. That includes the script, the CI
step, and the pre-commit hook. A rule that reports at the `warn` level is
either worth an error or worth deletion.

### The flat config array

```ts
// eslint.config.ts — the whole array, in order.
import coreWebVitals from "eslint-config-next/core-web-vitals";
import nextTypescript from "eslint-config-next/typescript";
import { defineConfig } from "eslint/config";

export default defineConfig([
  { ignores: [".next/", "node_modules/", "src/api/generated/", "next-env.d.ts"] },
  ...coreWebVitals,
  ...nextTypescript,
  // The boundaries block: references/directory-and-module-boundaries.md
  // The type-aware block: references/typescript-config-and-enforcement.md
  // The React rules block: references/state-and-effects.md
]);
```

`eslint-config-next` ships two flat exports. `core-web-vitals` carries the
Next.js rules. `typescript` carries the TypeScript rules. Take both. A project
that takes only the first loses every TypeScript report, and the loss is
silent.

Ignore the generated folder. A generated file must never fail a gate that no
person can fix.

| Symptom | Cause | What to do |
| --- | --- | --- |
| `Parsing error: ESLint was configured to run on ... however that TSConfig does not include this file` | A type-aware rule met a file outside the `tsconfig.json` include, such as a root config file | Put that file in a glob override with the type-aware rules off. `references/typescript-config-and-enforcement.md` holds the override |
| A TypeScript rule reports nothing | The array holds `core-web-vitals` and not `typescript` | Add the second export. Plant an unused variable and confirm the report |
| `It looks like you're trying to use tailwindcss directly as a PostCSS plugin` | Tailwind v4 moved the plugin | Use `@tailwindcss/postcss`. The CSS entry is `references/design-tokens-and-theming.md` |

### Next 16 removed `next lint`, so CI runs the lint step

`next lint` is alive only in legacy code.

WARNING: a pipeline that called `next lint` still exits 0 after the upgrade,
and it lints nothing. The build stays green and unlinted code ships. This is
the failure mode of the whole domain, because nothing reports it.

Do three things, in order. Search CI and `package.json` for `next lint`.
Replace each hit with `eslint . --max-warnings=0`. Plant an error and confirm
that the step now fails.

The codemod is `pnpm dlx @next/codemod@canary next-lint-to-eslint-cli .`.
Remove the `eslint` key from `next.config.ts` as well; Next 16 no longer reads
it. `references/app-router-structure.md` owns that file.

### Prettier is the default, and Biome has a condition

| Condition | Choice | It reverses when | The cost |
| --- | --- | --- | --- |
| The project needs `eslint-plugin-react-hooks`, `jsx-a11y`, `eslint-config-next`, or type-aware rules | ESLint 9 and Prettier. This stack needs all four | Biome ships every one of the four rule sets. | The lint run reads type information, so it is the slowest gate in the set. |
| None of those are needed, and the CI lint time is a measured bottleneck | Biome, as one binary for the format and the lint | Any one of the four rule sets becomes a requirement. | The React Compiler rules and the type-aware rules report nothing. |
| The lint time is the problem, and the rules above are still needed | Biome for the format, and a reduced ESLint for the rules that Biome lacks | The measurement states that one tool is inside the budget. | Two tools decide the format and the rules, so a contributor must configure both. |

Biome covers a large part of the ESLint and typescript-eslint rule set, and it
is close to Prettier on output. Its type-aware coverage is partial, and it
carries no `eslint-plugin-react-hooks`. A Next.js application on the React
Compiler needs that plugin, so Biome loses by default here.

Measure before you switch. A lint time that nobody has timed is not a
bottleneck.

### The class order is a format concern

Order the Tailwind classes with `prettier-plugin-tailwindcss`. Tailwind v4 has
no `tailwind.config.js`, so the plugin needs the path to the CSS entry that
holds `@theme`.

```jsonc
// .prettierrc
{
  "plugins": ["prettier-plugin-tailwindcss"],
  "tailwindStylesheet": "./src/styles/globals.css"
}
```

Add no Tailwind lint plugin for class validation. The `eslint-plugin-tailwindcss`
package has partial v4 support, so it is current but in decline. Where a project already has it, confirm that it
resolves the v4 CSS entry, or replace it with `eslint-plugin-better-tailwindcss`.
`references/design-tokens-and-theming.md` owns the tokens and the theme.

`.editorconfig` carries the indent and the line ending for editors that do not
run Prettier. Prettier stays the authority on everything it formats.

### A disable names a rule and states a reason

```tsx
// Wrong: the disable carries no reason.
// Failure: a later reader cannot tell whether the rule was wrong or the code
// was. The comment outlives the fix that made it unnecessary, and the rule
// erodes one file at a time.
// eslint-disable-next-line boundaries/element-types
import { InvoiceRow } from "@/features/billing/components/invoice-row";
```

```tsx
// Correct: two hyphens separate the rule from the reason, and the reason
// states when the line goes away.
// eslint-disable-next-line boundaries/element-types -- FE-412 lifts this part
// to src/components/common in the next change
import { InvoiceRow } from "@/features/billing/components/invoice-row";
```

Each disable names one rule and covers one line. NEVER write a file-level
`/* eslint-disable */`. It turns every rule off for that file, and no report
tells you which rule it hid.

### Generate the route types before the typecheck

A CI job that runs the lint and the typecheck without a build has no generated
route types. `next-env.d.ts` and the files under `.next/types/` stay out of
version control, so `tsc` reports a type that it cannot find.

Run `next typegen` before `pnpm typecheck` in that job. The command is cheap,
and it makes the job independent of the build.
`references/typescript-config-and-enforcement.md` owns the generated type
files.

### The repository states its own commands

A repository under work carries an `AGENTS.md` at the root. Keep it short.
State three things: the exact commands, the paths that nobody edits by hand,
and the definition of done.

The paths that nobody edits by hand are `src/api/generated/`, the lockfile, and
`next-env.d.ts`. `AGENTS.md` is plain Markdown with no schema. The file closest
to the edited file wins, and a direct instruction in the chat wins over both.
`references/instruction-files-and-skill-discovery.md` owns the size cap on the
joined text, and the region that `next dev` writes into this file.

A single `.cursorrules` file is deprecated, and it is alive only in legacy
code. Use `.cursor/rules/*.mdc`, or
prefer `AGENTS.md` where more than one agent reads the repository. Commit
`.vscode/settings.json` and `.vscode/extensions.json` so every contributor gets
the same editor setup.

## Verification

```bash
# 1. Read the script surface. Every job must have a name.
node -p "Object.keys(require('./package.json').scripts).join(' ')"

# 2. Confirm that the lint script refuses a warning.
rg -n '"lint":.*--max-warnings=0' package.json

# 3. Find the removed Next command. This must print nothing.
rg -n 'next lint' package.json .github/

# 4. Find the removed config key. This must print nothing.
rg -n 'eslint' next.config.ts

# 5. Prove that the lint gate fails. Add an unused import, then run:
pnpm lint

# 6. Find a disable with no reason. This must print nothing.
rg -n 'eslint-disable' src/ | rg -v ' -- '

# 7. Find a file-level disable. This must print nothing.
rg -n '/\* eslint-disable \*/' src/

# 8. Confirm that the tree is formatted.
pnpm exec prettier --check .

# 9. Run the gate that ends a task.
pnpm check
```

## Review checklist

- [ ] Does `package.json` carry `dev`, `build`, `start`, `lint`, `lint:fix`,
      `format`, `typecheck`, `test`, `test:watch`, `test:e2e`, `api:generate`,
      `analyze`, and `check`?
- [ ] Does `check` run the lint, the typecheck, the tests, and the build?
- [ ] Does every lint invocation carry `--max-warnings=0`?
- [ ] Does the flat config take both `eslint-config-next/core-web-vitals` and
      `eslint-config-next/typescript`?
- [ ] Does the `ignores` entry cover `.next/`, `node_modules/`,
      `src/api/generated/`, and `next-env.d.ts`?
- [ ] Is `next lint` absent from `package.json` and from CI, and does an
      explicit `eslint` step run instead?
- [ ] Is the `eslint` key absent from `next.config.ts`?
- [ ] Does a planted error fail the CI lint step?
- [ ] Is the formatter choice recorded, and does Prettier stay the default
      where the React Compiler rules are needed?
- [ ] Does `.prettierrc` set `tailwindStylesheet` for Tailwind v4?
- [ ] Does every `eslint-disable` name one rule and carry a reason after two
      hyphens?
- [ ] Is a file-level `/* eslint-disable */` absent?
- [ ] Does a CI job without a build run `next typegen` before the typecheck?
- [ ] Does `AGENTS.md` exist, and does it state the commands, the untouched
      paths, and the definition of done?

## Handoffs

- The folder layout, the boundaries block in the config array, and the
  `api:generate` output path →
  `references/directory-and-module-boundaries.md`.
- The package manager, the hooks that run these commands on commit, and the
  dependency policy → `references/dependencies-and-git-workflow.md`.
- `tsconfig.json`, `projectService`, the type-aware presets, and the generated
  type files → `references/typescript-config-and-enforcement.md`.
- `eslint-plugin-react-hooks`, its flat preset, and the React Compiler keys →
  `references/state-and-effects.md`.
- The `next.config.ts` keys other than `eslint` →
  `references/app-router-structure.md`.
- The Tailwind theme, the tokens, and the CSS entry that `@theme` lives in →
  `references/design-tokens-and-theming.md`.
- The `jsx-a11y` rule set and the accessibility gate →
  `references/wcag-conformance-and-verification.md`. That domain holds a veto.
- The test layout, the fixtures, and the coverage threshold →
  `references/test-strategy-and-component-tests.md`. The lint plugins for a test
  file, and the order of the gates →
  `references/merge-gates-and-quality-signals.md`.
- The CI workflow that calls these scripts, the runners, and the cache →
  `references/release-pipeline-and-rollback.md`.
- The precedence of `AGENTS.md`, its 32 KiB cap, the managed
  `nextjs-agent-rules` region, and the audit of an installed skill →
  `references/instruction-files-and-skill-discovery.md`.
