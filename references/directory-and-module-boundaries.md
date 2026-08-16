# Directory and module boundaries

Next.js 16.3 baseline, TypeScript 5.9, pnpm 10.x, Node 20.9 or later. This file
owns the place where a file goes, and the rule for which module may import it.
The subjects are the directory layout, the dependency direction, and the lint
rule that holds the direction. They also include the path alias, the barrel
file, the generated client folder, and the monorepo decision.

The lint config, the formatter, and the script surface are
`references/lint-format-and-scripts.md`. The package manager and the git
workflow are `references/dependencies-and-git-workflow.md`. The `server-only`
guard that a module carries is `references/server-and-client-components.md`.

## Principle

A directory tree is a dependency graph with names. Decide the direction of the
edges once, at the start of the project. A direction that no lint rule holds is
a convention, and a convention decays.

A folder that accepts every file states nothing. The name of a folder must
predict what is inside it.

A shared layer earns its place on the second consumer. A file that moves to a
shared folder in anticipation is a guess about the future.

A public API is a file, never a habit. One file for each feature states what
the rest of the application may import.

A generated folder is an output. No file inside it is a source file.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### `src/` holds the application, and the root holds the config

Put the application under `src/`. Keep `package.json`, `next.config.ts`,
`tsconfig.json`, `public/`, and every `.env` file at the repository root. The
router then reads `src/app/`.

The split gives each file one address. A reader who opens the root sees the
tools. A reader who opens `src/` sees the product.

### The layout

```text
// Wrong: one folder holds every component in the application.
src/components/  Button.tsx  UserAvatar.tsx  CheckoutForm.tsx  OrderRow.tsx ...
// Failure: the folder reaches 120 flat files. No file states an owner, and no
// rule states a boundary. A reader cannot tell a design-system primitive from
// feature UI. Imports run in both directions until a cycle appears, and every
// session adds one more file to the pile.
```

```text
// Correct: routing, features, and shared layers, with one direction.
src/
  app/                   routing and composition only
    (marketing)/         a route group
    (app)/
    (auth)/
  features/
    checkout/
      components/  hooks/  api/  schemas/  types/  utils/
      index.ts           the only path another feature may import
  components/
    ui/                  unbranded primitives, and the shadcn/ui output
    common/              branded, and used by two features or more
  lib/                   shared code that renders nothing
  server/                server-exclusive modules
  config/  styles/  types/
  api/generated/         the output of api:generate, and not in git
```

`app/` composes. It holds the route files and the tree that they render, and
it holds no business logic. `src/features/<feature>/` holds the logic.

A private folder such as `app/dashboard/_components/` is for small route-local
composition. Move that code to `src/features/` when the logic grows, or when a
second route needs it. `references/app-router-structure.md` owns the meaning of
each folder token. The tokens are `(group)`, `_folder`, `[param]`, and `@slot`.

### Colocate until a second consumer exists

| Condition | Where the file goes | It reverses when | The cost |
| --- | --- | --- | --- |
| One feature uses it | Inside that feature | A second feature needs it, which the next two rows cover. | A later move to a shared layer changes every import of the file. |
| A second feature needs it, and it renders | `src/components/common` | The second consumer is removed, so one feature is left. | The file now serves two callers, so a change to it must suit both. |
| A second feature needs it, and it renders nothing | `src/lib` | The module starts to render, which the row above covers. | The same. The module is shared, and no feature owns it. |
| It is an unbranded primitive | `src/components/ui` | The part takes a brand decision, so it moves to `common`. | The generator that writes this folder overwrites a hand edit. |
| It reads a secret, a session, or the database | `src/server` or `src/lib/dal` | Never. A client import of the module must fail the build. | No client component can import it, so the value must arrive as a prop. |
| It is a route file | `src/app/`, under the segment that serves the URL | Never. The path in this folder is the URL. | A move of the file is a change of the URL. |

NEVER move a file to a shared folder before the second consumer exists. A
shared folder that holds a file with one consumer is a folder with no rule.

### The direction runs one way, and a lint rule holds it

Three layers, and three edges:

- `app` may import a feature, and may import a shared layer.
- A feature may import a shared layer.
- A shared layer may import a shared layer only.

A shared layer NEVER imports a feature. A feature NEVER imports the internal
path of another feature.

```ts
// eslint.config.ts — the boundaries block. The rest of the array is
// references/lint-format-and-scripts.md.
import boundaries from "eslint-plugin-boundaries";
import { defineConfig } from "eslint/config";

export default defineConfig([
  {
    files: ["**/*.{ts,tsx}"],
    plugins: { boundaries },
    settings: {
      "boundaries/elements": [
        { type: "app", pattern: "src/app/*" },
        { type: "feature", pattern: "src/features/*" },
        { type: "shared", pattern: "src/{components,lib,server,config,types}/*" },
      ],
    },
    rules: {
      "boundaries/element-types": [
        2,
        {
          default: "disallow",
          rules: [
            { from: "app", allow: ["feature", "shared"] },
            { from: "feature", allow: ["shared"] },
            { from: "shared", allow: ["shared"] },
          ],
        },
      ],
    },
  },
]);
```

The rule reports `Importing elements of type 'feature' is not allowed` on a
cross-feature import. The report arrives in the editor and in CI, which is the
point. A boundary that only a review catches is not a boundary.

### The path alias replaces the deep relative import

```ts
// Wrong: the relative path counts the folders between two files.
// Failure: the import breaks when either file moves, and the reader cannot
// tell which module it names. The depth is also a signal that the alias
// exists and that this file does not use it.
import { formatMoney } from "../../../../lib/money";
```

```ts
// Correct: the alias names the module from the root of the source tree.
import { formatMoney } from "@/lib/money";
```

`tsconfig.json` declares the alias as `"paths": { "@/*": ["./src/*"] }`.
`references/typescript-config-and-enforcement.md` owns that file.

### A feature exposes one barrel, and the application has none

```ts
// Wrong: one feature reaches into the internals of another.
// Failure: the two features couple. A rename inside checkout breaks the
// importer, the boundaries rule reports "Importing elements of type
// 'feature' is not allowed", and each later refactor cascades.
import { CartLineItem } from "@/features/checkout/components/cart-line-item";
```

```ts
// Correct: import the declared public API, or lift the part to a shared layer.
// src/features/checkout/index.ts re-exports CheckoutSummary and nothing else.
import { CheckoutSummary } from "@/features/checkout";
```

```ts
// Wrong: src/components/index.ts re-exports the whole tree.
// Failure: an import of one Button pulls every module in the barrel into the
// module graph. The development compile slows down, and the bundle grows
// where the tree shake fails.
export * from "./ui/button";
export * from "./ui/calendar";
export * from "./common/user-avatar"; // and 80 more lines
```

```ts
// Correct: import the module directly.
import { Button } from "@/components/ui/button";
```

The rule has three parts. One shallow barrel for each feature is the public
API. A file inside a feature imports its own modules by direct path, never
through the barrel. No other barrel exists.

An external package that ships a barrel is a different case. List that package
in `optimizePackageImports` in `next.config.ts`, and the build unrolls the
barrel. That key targets a package in `node_modules`, never a folder of your
own.

### A cycle needs a graph

The boundaries rule reads one import at a time. It cannot see a cycle across
three files, and it cannot see a file that nothing imports.

| Condition | Tool | It reverses when | The cost |
| --- | --- | --- | --- |
| Import direction and the public API, one Next application | `eslint-plugin-boundaries`, in the config above | The repository holds two or more applications, so the graph crosses packages. | One more plugin in the lint array, and a settings block that each new folder must join. |
| A path-level rule beside the element rule | `no-restricted-paths`, from `eslint-plugin-import` or `import-x` | The element rule already covers the path, so the second rule repeats it. | Two rules state one boundary, and a later reader must read both. |
| A cycle, an orphan, or a graph of the whole tree | `dependency-cruiser`, run in CI | The tree is small enough that the lint rule alone reports every cycle. | One more CI step, one more config file, and a run over the whole tree. |
| A TypeScript monorepo that needs the `index.ts` public API rule, and little config | `sheriff`, which has no dependencies | The project needs a rule that `sheriff` does not hold. | Less control over each rule than the plugin above gives. |

Run `dependency-cruiser` as a gate, not as a report. It exits non-zero on a
planted violation, and that exit is the check.

### The generated client is an output folder

The frontend consumes a DRF contract through a generated TypeScript client.
This file owns three facts about it. `src/api/generated/` is the path. The
script name is `api:generate`. The folder stays out of version control.

`references/openapi-schema-and-codegen.md` owns the generator choice and the
drf-spectacular config. `references/data-access-and-mutations.md` owns the
module that calls the backend.
`references/boundary-validation-and-api-types.md` owns the parse.

```jsonc
// package.json — the script name is fixed here; the generator is
// references/openapi-schema-and-codegen.md.
{
  "scripts": {
    "api:generate": "openapi-typescript schema.yml -o src/api/generated/schema.d.ts"
  }
}
```

Django CI publishes `schema.yml` as a build artifact. The frontend pipeline
downloads that artifact and runs `pnpm api:generate`. No shared package and no
monorepo is needed for the exchange.

CAUTION: a gitignored folder produces no `git diff`, so a diff check on it
always passes. Prove the freshness with the compiler instead. Run
`pnpm api:generate`, then run `pnpm typecheck`. A renamed serializer field
that the application reads becomes a compile error.

Where a reviewer must see every contract change in the pull request, commit
the folder instead. Then the gate is
`pnpm api:generate && git diff --exit-code src/api/generated`, and
`.gitattributes` marks the path `linguist-generated` so the diff stays folded.
Take one of the two, and state which one in `AGENTS.md`.

NEVER edit a file under `src/api/generated/`. The next run of `api:generate`
overwrites the edit, and the fix disappears with no report.

### One Next application and one Django repository is not a monorepo

| Condition | Decision | It reverses when | The cost |
| --- | --- | --- | --- |
| One Next application, and one Django repository | No monorepo. Django publishes `schema.yml`, and the frontend runs `api:generate` | A second JS or TS application appears, which the next row covers. | A shared change crosses two repositories, so it needs two pull requests. |
| Two or more JS or TS applications, with shared UI, config, or client packages | pnpm workspaces and Turborepo | One application is left, or the tasks need an affected graph across languages. | A workspace file, a `turbo.json`, a second install mode, and a slower first install. |
| Enterprise scale that needs generators, an affected graph, and more than one language | Nx | The graph fits Turborepo, so the extra tooling returns nothing. | A large configuration surface, and a tool that each contributor must learn. |

A monorepo tool for a single application is a build system with no work to do.
It costs a `turbo.json`, a workspace file, and a second install mode, and it
returns nothing.

Where a workspace does exist, consume an internal package from source. List it
in `transpilePackages` in `next.config.ts`, and the application compiles it
with no separate build step. Compile the package to `dist/` only when you
publish it, or when another runtime consumes it.

### A structure change is its own task

NEVER restructure the repository inside a feature task. A moved folder makes
the diff unreviewable, and it hides the change that the request asked for.

Record the structure problem, finish the feature, and open the move as its own
change.

## Verification

```bash
# 1. Confirm the layout. Each path must exist.
ls -d src/app src/features src/components/ui

# 2. Find a deep relative import. This must print nothing.
rg -n '\.\./\.\./\.\./' -g '*.ts' -g '*.tsx' src/

# 3. Find a cross-feature import. Read each hit; a feature may import its own
#    modules by direct path.
rg -n "from ['\"]@/features/[^/'\"]+/" src/features/

# 4. Prove that the boundary rule fails. Add a cross-feature import, then run:
pnpm lint
# Expect a non-zero exit and "Importing elements of type 'feature' is not allowed".

# 5. Find a barrel outside a feature. This must print nothing.
rg -n '^export \* from' src/components src/lib src/server

# 6. Find a cycle and an orphan.
pnpm dlx depcruise --config .dependency-cruiser.js src

# 7. Confirm that the generated client is not in version control.
git ls-files src/api/generated

# 8. Prove that the generated client matches the schema.
pnpm api:generate && pnpm typecheck

# 9. Find a hand-edited generated file.
git log --oneline -- src/api/generated
```

## Review checklist

- [ ] Does the application live under `src/`, with the config and `public/` at
      the repository root?
- [ ] Does `app/` hold routing and composition only, with the logic in
      `src/features/`?
- [ ] Is every file in a shared folder used by two consumers or more?
- [ ] Does the boundaries config declare `app`, `feature`, and `shared`, with
      `default: "disallow"`?
- [ ] Does `pnpm lint` fail on a planted cross-feature import?
- [ ] Is every import through the `@/*` alias, with no path of three parent
      steps or more?
- [ ] Does each feature expose one shallow `index.ts`, and does no other
      barrel exist?
- [ ] Does an external barrel package appear in `optimizePackageImports`?
- [ ] Does CI run `dependency-cruiser` for a cycle and an orphan?
- [ ] Is the generated client at `src/api/generated/`, produced by
      `api:generate`, and out of version control?
- [ ] Does CI prove the client against the schema, by `api:generate` and then
      the typecheck?
- [ ] Is the monorepo decision recorded, and does one application still use no
      workspace tool?
- [ ] Is the diff free of a file move that the request did not ask for?

## Handoffs

- The lint config array, the formatter, and the `package.json` scripts →
  `references/lint-format-and-scripts.md`.
- The package manager, the lockfile, the git hooks, and `.gitignore` →
  `references/dependencies-and-git-workflow.md`.
- The `server-only` and `client-only` guards that a module in `src/server`
  carries → `references/server-and-client-components.md`.
- `tsconfig.json`, the `paths` value, and the generated type files →
  `references/typescript-config-and-enforcement.md`.
- The module that calls Django, and the Server Action that mutates →
  `references/data-access-and-mutations.md`.
- The parse that proves a generated type at run time →
  `references/boundary-validation-and-api-types.md`.
- The folder tokens `(group)`, `_folder`, and `@slot`, and the route files →
  `references/app-router-structure.md`.
- The decomposition threshold that starts a new file →
  `references/component-composition.md`.
- The drf-spectacular config, the generator choice, the schema artifact, and
  the drift gate → `references/openapi-schema-and-codegen.md`. The server side
  belongs to the sibling skill `django-api-contract`.
- The tokens that `src/components/ui` renders → domain 09
  `design-system-and-styling`. Not integrated yet.
- The test file layout under a feature → domain 20 `testing-and-quality`. Not
  integrated yet.
- The CI pipeline that downloads the schema artifact, and the Docker build →
  domain 22 `build-deploy-and-runtime-ops`. Not integrated yet.
