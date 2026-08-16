# TypeScript config and enforcement

TypeScript 5.9 baseline, Next.js 16.3, React 19.2, typescript-eslint v8, Node
20.9 or later. This file owns the compiler configuration and the checks that
prove it. The subjects are `tsconfig.json`, the generated type files, the
type-aware lint config, and the gates that run in CI. The vocabulary that
models a value is
`references/type-modeling-and-narrowing.md`. The proof that an external value
has the shape it claims is `references/boundary-validation-and-api-types.md`.
The route files that the compiler checks are
`references/app-router-structure.md`.

## Principle

A compiler flag is a class of bug that cannot reach production. Turn each one
on once, at the start of the project. A flag added later is a backlog.

A type system checks the claims inside the program. It checks no claim about
the world outside the program. Know which of the two you have.

A check that only a person runs is not a check. Put it in CI, and fail the
build on it.

A build is not a typecheck. A bundler can emit correct output from code that
does not compile.

A suppression that carries no reason is permanent. A suppression that expires
when the problem goes away is temporary.

## Pinned-stack depth

### `tsconfig.json`

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "module": "esnext",
    "moduleResolution": "bundler",
    "jsx": "preserve",
    "noEmit": true,
    "allowJs": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "incremental": true,
    "skipLibCheck": true,

    "strict": true,
    "noUncheckedIndexedAccess": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,

    "plugins": [{ "name": "next" }],
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

| Key | What it prevents |
| --- | --- |
| `strict` | The whole family: an implicit `any`, an unchecked `null`, an unsound `this`. Never turn one member off. |
| `noUncheckedIndexedAccess` | `arr[i]` and `record[key]` return `T`. With the flag they return `T \| undefined`, which is the truth. |
| `isolatedModules` | A file that only one compiler can read. Required for a single-file transpiler. |
| `verbatimModuleSyntax` | A type-only import that survives to run time, or a side-effect import that the bundler removes. It reports `TS1484`. |
| `noImplicitOverride` | A method that stops overriding a base method after a rename, and silently becomes a new method. |
| `noPropertyAccessFromIndexSignature` | A typo on an index signature that reads as a valid property. |
| `moduleResolution: "bundler"` | A resolution mode that disagrees with Turbopack. |
| `jsx: "preserve"` and `noEmit: true` | A second, competing build. Next.js owns the emit. |
| `plugins: [{ "name": "next" }]` | The editor loses the Next.js type service. |
| `skipLibCheck` | A slow build caused by errors inside `node_modules` types. |

Two flags depend on the project. Turn `exactOptionalPropertyTypes` on when the
backend distinguishes an absent field from a `null` field, because the flag
keeps `?` and `| undefined` apart. It has a known rough edge with a compound
assignment such as `??=` on an optional property. Where you turn it on, assign
with an explicit `if (x === undefined)` test instead. Set `moduleDetection` to
`"force"` when a file with no import or export is read as a script.

### The generated type files are artifacts

`next-env.d.ts`, the `PageProps` and `LayoutProps` and `RouteContext` helpers,
and the files under `.next/types/` are generated. Run `npx next typegen` to
generate them. `next dev` and `next build` generate them as well.

NEVER edit a generated type file by hand. The next command overwrites the
edit. Keep `next-env.d.ts` out of version control. A missing helper type is a
sign that the generation step did not run, never a sign that you must write
the type yourself.

The route files that consume these helpers, and the rule that request data is
awaited, are `references/app-router-structure.md`.

### `next build` is not the typecheck

```jsonc
// Wrong: next.config.ts turns the type error into a warning nobody reads.
// Failure: the build is green, the editor is red, and the error reaches
// production. The flag also hides every future error on the same code.
{
  "typescript": { "ignoreBuildErrors": true }
}
```

```jsonc
// Correct: package.json runs the typecheck as its own command.
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "test:types": "vitest --typecheck --run"
  }
}
```

Run `tsc --noEmit` in CI as a gate of its own, beside the build. It exits 0 and
prints nothing when the domain passes. A non-zero exit is a failed task.

Next.js 16.3 runs the project-local `tsc` CLI for its own check, through
`experimental.useTypeScriptCli`, which defaults to true. That check is still
not your gate. Keep the standalone command.

### The lint gate needs type information

```ts
// eslint.config.ts
import tseslint from "typescript-eslint";

export default tseslint.config(
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
    rules: {
      "@typescript-eslint/ban-ts-comment": [
        "error",
        { "ts-expect-error": "allow-with-description", minimumDescriptionLength: 10 },
      ],
    },
  },
  {
    files: ["**/*.js"],
    extends: [tseslint.configs.disableTypeChecked],
  },
);
```

Take the type-checked presets, never the plain `strict` and `stylistic`
presets. A rule without type information cannot see a promise or a union.
`projectService: true` is the stable form in v8; the v7 name was
`EXPERIMENTAL_useProjectService`.

| Rule | The failure it catches |
| --- | --- |
| `no-explicit-any` | An `any` written by hand. |
| `no-unsafe-assignment`, `no-unsafe-return`, `no-unsafe-argument`, `no-unsafe-member-access` | An `any` that arrived from `res.json()` or from a package with weak types. |
| `no-floating-promises` | A missing `await`. The rejection disappears with the promise. |
| `no-misused-promises` | `onClick={handleAsync}`. The handler wants a `void` return, so the rejection is dropped. Wrap it: `onClick={() => { void handleAsync(); }}`. |
| `switch-exhaustiveness-check` | A union `switch` that a new variant walks past. |
| `ban-ts-comment` | A suppression with no reason. |

Next 16 removed `next lint`. Run ESLint or Biome directly, from the `lint`
script and from CI. The codemod is `next-lint-to-eslint-cli`.

### A suppression states its reason and expires

```ts
// Wrong: the blanket suppression.
// Failure: @ts-ignore silences this error and every future error on the next
// line, forever. It stays after the cause is fixed, because nothing reports
// an unused @ts-ignore.
// @ts-ignore
legacy.render(node);
```

```ts
// Correct: the suppression names the reason and reports itself when it is
// no longer needed.
// @ts-expect-error legacy.render is untyped until the vendor ships v4 types
legacy.render(node);
```

`@ts-ignore` is rejected. `@ts-expect-error` with a description is the only
permitted suppression, and a count of them is a number that must go down.

### Type-level tests

Vitest runs `tsc` over `*.test-d.ts` files under the `--typecheck` flag. Use
`expectTypeOf` and `assertType`. A `@ts-expect-error` line that stops erroring
fails the run.

```ts
// lib/api/paginated.test-d.ts
import { expectTypeOf, test } from "vitest";
import type { Paginated } from "./paginated";

test("results is an array, never the bare item", () => {
  expectTypeOf<Paginated<{ id: string }>>().toHaveProperty("results");
  expectTypeOf<Paginated<{ id: string }>["results"]>().toEqualTypeOf<{ id: string }[]>();
});
```

Add type-level tests where a shared utility or a generated client is exported
to other code. Skip them for application components. The test runner itself is
domain 20 `testing-and-quality`.

### When the compiler is slow

| Symptom | Cause | What to do |
| --- | --- | --- |
| `TS2589`, "type instantiation is excessively deep" | A recursive utility type, or a very large union | Simplify the type. Add an explicit annotation to stop the inference. Bound the recursion. |
| A slow editor, a slow `tsc` | Deep conditional types, a large union, a lib check | Keep `skipLibCheck: true`. Use a `Prettify` helper sparingly. Split the project with project references. |

Measure before you change anything. Run `tsc --noEmit --extendedDiagnostics`
for the instantiation counts, and `tsc --generateTrace` for the detail.

### Version discipline

TypeScript 5.9 is the pin. Zod 4 requires TypeScript 5.5 or later, so 5.5 is
the hard floor for this stack. Next.js 16 requires 5.1 or later, which is
lower than both.

TypeScript 6.0 is stable and 7.0 is a release candidate. 7.0 is the native
port, and it does not yet provide the JavaScript compiler API. NEVER rebase
this material onto a 6.0 or a 7.0 idiom while the project pins 5.9. Read the
installed version first.

## Verification

```bash
# 1. Read the installed version before you write code.
node -p "require('typescript/package.json').version"

# 2. The typecheck gate. It exits 0 and prints nothing.
pnpm exec tsc --noEmit

# 3. The lint gate, with type information.
pnpm exec eslint .

# 4. Find a blanket suppression. This must print nothing.
rg -n '@ts-ignore' src/

# 5. Find a suppression with no description. This must print nothing.
rg -n '@ts-expect-error\s*$' src/

# 6. Confirm that the build does not ignore type errors. This must print
#    nothing.
rg -n 'ignoreBuildErrors' next.config.ts

# 7. Confirm that no generated type file was edited by hand.
git log --oneline -- next-env.d.ts

# 8. Measure the compiler when the editor is slow.
pnpm exec tsc --noEmit --extendedDiagnostics
```

## Review checklist

- [ ] Does `tsconfig.json` set `strict`, `noUncheckedIndexedAccess`,
      `isolatedModules`, and `verbatimModuleSyntax`?
- [ ] Does it set `moduleResolution: "bundler"`, `jsx: "preserve"`,
      `noEmit: true`, and `plugins: [{ "name": "next" }]`?
- [ ] Is every member of the `strict` family on, with none turned off
      individually?
- [ ] Does a decision on `exactOptionalPropertyTypes` exist, and is it
      recorded?
- [ ] Is `next-env.d.ts` unedited, and out of version control?
- [ ] Is `ignoreBuildErrors` absent from `next.config.ts`?
- [ ] Does CI run `tsc --noEmit` as a gate separate from the build?
- [ ] Does the lint config use `strictTypeChecked` and `stylisticTypeChecked`
      with `projectService: true`?
- [ ] Does the lint config turn type-aware rules off for `*.js` files?
- [ ] Does a lint command run in CI, now that `next lint` is removed?
- [ ] Is `@ts-ignore` absent from the codebase?
- [ ] Does every `@ts-expect-error` carry a description?
- [ ] Does a shared utility or a generated client have type-level tests?

## Handoffs

- The vocabulary that the flags enforce — a union, a brand, a cast, a
  narrow → `references/type-modeling-and-narrowing.md`.
- The schema that proves a value from outside the program →
  `references/boundary-validation-and-api-types.md`.
- The route files, `next typegen` as a build step, and the awaited request
  data → `references/app-router-structure.md`.
- The folder layout, the Prettier config, the package manager, and the
  monorepo project references → domain 04 `project-structure-and-tooling`.
  Not integrated yet.
- The Vitest setup, the test file layout, and the contract test against the
  schema → domain 20 `testing-and-quality`. Not integrated yet.
- The CI pipeline that runs these gates, and the Docker build → domain 22
  `build-deploy-and-runtime-ops`. Not integrated yet.
