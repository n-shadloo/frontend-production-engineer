# Dependencies and git workflow

pnpm 10.x, Node 20.9 or later, lefthook, commitlint, gitleaks. This file owns
what enters the repository and how a change leaves it. The subjects are the
package manager, the lockfile, the supply-chain policy, and the update bot.
They also include the git hooks, the commit message, `.gitignore`, and the
secret scan.

The scripts that these hooks call are
`references/lint-format-and-scripts.md`. The folder that a new file goes into
is `references/directory-and-module-boundaries.md`.

## Principle

A dependency is a permanent liability with a temporary benefit. Count the cost
before the install, because the removal costs more.

A lockfile is the build. Two machines that resolve different versions run
different programs, and only one of them was tested.

Time is a supply-chain control. A malicious release is found in days, so an
install that waits misses it.

A hook that a fresh clone does not install is not a hook. Prove it from the
clone, never from your own machine.

A secret that reached the history stays in the history. The fix is a rotation.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### pnpm, at the version that the Node floor allows

Use pnpm. Choose npm only where a tool needs it, and choose yarn only for a
repository that already has it.

pnpm 11 needs Node 22 or later. This stack pins Node 20.9 or later, so pin
pnpm to the 10.x line. Move to pnpm 11 only when the Node floor moves to 22.

Three files state a version, and all three must agree.

| File or field | What it pins |
| --- | --- |
| `packageManager` in `package.json` | The exact pnpm version, which Corepack reads |
| `engines.node` in `package.json` | The Node range that an install accepts |
| `.nvmrc` | The Node version that a contributor and CI select |

Corepack ships with Node, and it is off by default. Run `corepack enable` once
on each machine. Volta pins the same versions where a team prefers it.

### The lockfile is the build

Run `pnpm install --frozen-lockfile` in CI. The install fails when the lockfile
disagrees with `package.json`, which is the report you want.

NEVER hand-edit `pnpm-lock.yaml`. Change `package.json`, run the install
locally, and commit the lockfile that the install produced.

### A new dependency needs one sentence

State three facts in the pull request: what the package replaces, its install
size, and its maintenance status. A package with no answer to all three does
not go in.

A library added for one component is the common failure. It carries a bundle
cost and a supply-chain surface for one widget. Copy the forty lines, or build
the part on a primitive that `src/components/ui` already holds.

### The install waits

```yaml
# Wrong: pnpm-workspace.yaml carries no cooldown key.
# Failure: the install takes a version the moment it publishes. A compromised
# release reaches the lockfile inside the window between the publish and the
# takedown. That window has been about two hours in a real npm incident, and
# no install inside it sees a warning.
onlyBuiltDependencies:
  - "sharp"
```

```yaml
# Correct: pnpm-workspace.yaml holds every install back.
minimumReleaseAge: 10080          # 7 days, in minutes
minimumReleaseAgeExclude:
  - "react@*"                     # a vetted emergency security patch only
```

pnpm 11 defaults `minimumReleaseAge` to 1440, which is one day. The 10.x line
has no such default, so set the value explicitly. Use
`minimumReleaseAgeExclude` for a security patch that cannot wait, and for
nothing else.

A lifecycle script in a dependency runs with your privileges at install time.
Keep the allowlist of packages that may run one explicit. The key is
`onlyBuiltDependencies` on the 10.x line, and pnpm 11 replaces it with
`allowBuilds`.

This file owns the policy. `references/secret-boundary-and-supply-chain.md`
owns the threat model and the judgment of whether a package is dangerous.

### The update bot waits as well

| Condition | Bot | It reverses when | The cost |
| --- | --- | --- | --- |
| The project wants grouped updates, presets, and one cooldown setting | Renovate | The team cannot install a third-party GitHub application. | A configuration file to maintain, and one more application with write access to the repository. |
| The project prefers a GitHub-native tool with no extra app | Dependabot | The project needs grouped updates or a preset that Dependabot does not hold. | One pull request for each update, so a large dependency set produces a large queue. |

Renovate calls the setting `minimumReleaseAge`. The earlier name was
`stabilityDays`, which is alive only in legacy code. Its best-practice preset sets `"14 days"` where a rule
automerges. Dependabot now waits three days by default on a version update,
with no configuration, and a security update still opens at once.

Automerge a patch and a minor update only. NEVER automerge a major update. In
Renovate, set `matchUpdateTypes: ["patch", "minor"]` and
`matchCurrentVersion: "!/^0/"`, because a package below 1.0 breaks on a minor
bump. In Dependabot, read `update-type` with `dependabot/fetch-metadata` and
gate `gh pr merge --auto` on the same two types.

### The hooks run the gates that are cheap

Use lefthook. It is one Go binary, it runs jobs in parallel, and it re-stages a
file that the formatter changed under `stage_fixed: true`. Choose Husky only
where the team wants the most-adopted Node tool and needs no parallel run. The
Husky `prepare` script re-runs on every install.

| Hook | What it runs | Why there |
| --- | --- | --- |
| pre-commit | `prettier --write` and `eslint --max-warnings=0` over the staged files, through `lint-staged` or the lefthook glob | It is fast, and it keeps the format out of review |
| pre-commit | `gitleaks git --pre-commit --redact --staged --verbose` | A secret must never reach a commit |
| pre-push | `pnpm typecheck` and `pnpm test` | Slower gates, run once for each push |
| commit-msg | `commitlint --edit` | The message is checked while it is still editable |

Install the hooks from the repository, never by hand. Run `lefthook install`,
or call it from a `prepare` script. A contributor who clones and commits must
get the hooks with no extra step.

`gitleaks` reads `.gitleaks.toml` from the target path. The `detect` and
`protect` commands are deprecated in favor of `gitleaks git`, so those two
commands are alive only in legacy code. The project is
feature-complete and takes security patches only, which is a stable state and
not an abandoned one.

### The commit message is machine-readable

```text
// Wrong: the message names no type and no scope.
broke everything
// Failure: commitlint rejects it. Where no hook runs, the history carries no
// type at all, so no release note and no version bump can be derived from it.
```

```text
// Correct: a Conventional Commit.
fix(orders): correct the total on a partial refund
```

Use `@commitlint/config-conventional` through the commit-msg hook. The types
are the conventional set, and the scope names the feature folder that changed.

### `.gitignore` and the secret

```text
# Wrong: .gitignore covers the build output and not the environment file.
# Failure: git tracks .env.local on the next add, and the secret enters the
# history. A later delete removes it from the tree and not from the history,
# so the only fix is to rotate the secret.
.next/
out/
```

```text
# Correct: .gitignore, with the entries this stack needs.
.next/
out/
.env*.local
coverage/
playwright-report/
.turbo/
next-env.d.ts
src/api/generated/
```

`references/directory-and-module-boundaries.md` states why
`src/api/generated/` is on this list, and what replaces the diff check when it
is.

### Ownership and the generated diff

`CODEOWNERS` names a reviewer for each path. Put it at `.github/CODEOWNERS`,
and give the generated paths and the config files an owner.

`.gitattributes` marks a generated path `linguist-generated` and `-diff`, so a
review folds that file and does not print every line. Apply it only to a path
that the project commits.

## Verification

```bash
# 1. Read the three version pins. All three must agree.
node -p "[require('./package.json').packageManager, require('./package.json').engines.node].join(' ')"
rg -n '.' .nvmrc

# 2. Confirm the install cooldown.
rg -n 'minimumReleaseAge' pnpm-workspace.yaml .npmrc

# 3. Prove a frozen install and a green tree. WARNING: run this in a fresh
#    clone, because git clean deletes every untracked file.
git clean -xdf && pnpm install --frozen-lockfile && pnpm check

# 4. Find a tracked environment file. This must print nothing.
git ls-files | rg '\.env'

# 5. Prove that commitlint runs on a fresh clone.
git commit --allow-empty -m "broke everything"
# Expect a non-zero exit and a commitlint report.

# 6. Prove that the pre-commit hook runs.
git commit --allow-empty -m "chore: probe hooks"
# Expect the hook output. No output means the hooks are not installed.

# 7. Scan the history for a secret.
gitleaks git --redact --verbose

# 8. Confirm that a committed generated path is marked.
rg -n 'linguist-generated' .gitattributes
```

## Review checklist

- [ ] Do `packageManager`, `engines.node`, and `.nvmrc` state versions that
      agree?
- [ ] Is pnpm on the 10.x line while the Node floor is 20.9?
- [ ] Does CI install with `--frozen-lockfile`, and is the lockfile committed?
- [ ] Is `pnpm-lock.yaml` free of a hand edit?
- [ ] Does `minimumReleaseAge` have an explicit value?
- [ ] Is the install-script allowlist explicit?
- [ ] Does each new dependency carry a stated replacement, size, and
      maintenance status?
- [ ] Does the update bot wait, and does it automerge a patch and a minor only?
- [ ] Do the pre-commit, pre-push, and commit-msg hooks exist, and does a fresh
      clone install them?
- [ ] Does the pre-commit hook run a secret scan?
- [ ] Does commitlint reject a message with no type?
- [ ] Does `.gitignore` cover `.next`, `out`, `.env*.local`, `coverage`,
      `playwright-report`, `.turbo`, `next-env.d.ts`, and the generated client?
- [ ] Is no `.env` file tracked by git?
- [ ] Does `CODEOWNERS` exist, and does `.gitattributes` mark a committed
      generated path?

## Handoffs

- The `package.json` scripts that the hooks call, the lint flags, and the
  formatter → `references/lint-format-and-scripts.md`.
- The folder that a new file goes into, the generated client path, and the
  monorepo decision → `references/directory-and-module-boundaries.md`.
- The environment variable that a bundle inlines, and the `NEXT_PUBLIC_`
  prefix → `references/app-router-structure.md`.
- The parse that proves an environment variable at boot →
  `references/boundary-validation-and-api-types.md`.
- The threat model of a dependency, the judgment on a malicious package, and
  the audit of a lockfile advisory →
  `references/secret-boundary-and-supply-chain.md`. That domain holds a veto.
- The test job that a pre-push hook runs →
  `references/merge-gates-and-quality-signals.md`.
- The CI workflow, the cache, the release pipeline, and the deploy →
  `references/release-pipeline-and-rollback.md`.
- The server-side secret storage and the rotation procedure →
  `secure-code-auditor`.
