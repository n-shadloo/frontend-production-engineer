# Merge gates and quality signals

Vitest, Playwright, ESLint 9 with the flat config, TypeScript 5.9,
`@lhci/cli`, `size-limit`, Storybook 9, Next.js 16.3. This file owns the set of
gates that a change passes before it merges, and the signals that sit beside
them. The subjects are the order of the gates, the rule that a gate fails
rather than reports, and the coverage of the change. They also include the
skipped test, the quarantine, and the flaky-test rate as a number. The last
subjects are the acceptance criteria of a feature, the output that a completion
claim carries, and the signals that are not gates.

The level that a test belongs to, and the coverage threshold itself, are
`references/test-strategy-and-component-tests.md`. The handler and the contract
test are `references/network-mocks-and-contract-tests.md`. The journey and the
flaky-test diagnosis are
`references/end-to-end-journeys-and-flake-control.md`.

## Principle

A definition of done is a list of commands, or it is an opinion. A command has
an exit code. An opinion does not.

A gate that prints a warning is not a gate. It is a message that a person
learns to scroll past.

Order the gates by cost. The cheapest check that can fail must fail first, so
the first red arrives in under a minute rather than after twenty.

A skipped test is a decision to ship the behavior untested. The decision needs
an owner and a date, or it is a deletion with a delay.

A signal is not a gate. A signal tells a person something. A gate stops a
merge. A project that confuses the two either blocks on noise or ships on
hope.

An agent cannot report a result that it did not read. The output of the run is
the evidence, and a sentence about the run is not.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The gate order

| The stage | The command | What it proves | Cost | It fails on |
| --- | --- | --- | --- | --- |
| Lint | `pnpm lint` | The static rules, including the accessibility rules and the type-aware rules | Seconds | Any report, at `--max-warnings=0` |
| Typecheck | `pnpm typecheck` | Every type in the repository, including the generated client | Seconds to a minute | A non-zero exit |
| Unit and component | `pnpm test` | The behavior of the logic and of each feature, with the network mocked | A minute or two | A failed assertion, or a threshold that the coverage misses |
| Build | `pnpm build` | The production compile, the route table, and the render mode of each route | A minute or more | A non-zero exit |
| End to end | `pnpm test:e2e` | The journeys against the served build, and the route accessibility scan | Minutes | A failed assertion |
| Budget | `npx size-limit`, then `npx lhci autorun` | The bytes and the laboratory metrics of the built routes | Minutes | A number above the budget |

The first two stages can run together, because neither needs the other. The
end-to-end stage needs the build, so it never starts before the build succeeds.
The budget stage reads the built output, so it follows the build as well.

`references/lint-format-and-scripts.md` owns the name of each script and the
`check` command that runs the first four in order. This file owns which stage
runs when, and what each one is allowed to fail on.
`references/performance-budgets-and-measurement.md` owns the budget stage and
the Lighthouse configuration inside it.
`references/wcag-conformance-and-verification.md` owns the accessibility lanes
that sit inside the lint stage, the component stage, and the end-to-end stage.
That domain holds a veto.

A pipeline that costs more time than the team accepts is a pipeline that the
team avoids. Where the total passes that point, cut the end-to-end list
before you cut the component list. The component stage returns more defects for
each minute it costs.

### A gate fails, and it never reports

```yaml
# Wrong: the test step is allowed to fail.
# Failure: the step is red inside the log and green on the merge button.
# Nobody opens the log of a passing run, so the suite rots for a month and
# then everybody agrees that "the tests are broken".
- run: pnpm test
  continue-on-error: true
```

```yaml
# Correct: the step fails the job, and the job is a required check.
- run: pnpm test
```

Three settings undo a gate in silence, and each one is a defect.

- `continue-on-error` on a check step.
- A test command that ends in `|| true`.
- A required check that the branch protection does not list, so the merge
  button ignores it.

`references/dependencies-and-git-workflow.md` owns the pre-push hook that runs
the typecheck and the tests before a push. Domain 22
`build-deploy-and-runtime-ops` owns the workflow file, the runners, and the
cache. It is not integrated yet.

### The change carries the coverage

Read the coverage of the files that the change touched, and not the number over
the repository. A repository number moves by a fraction on any single change,
so it reports nothing about that change.

`references/test-strategy-and-component-tests.md` owns the threshold for each
directory. This file owns the rule that the report is read on the pull request,
and that a fall in a touched directory is a finding.

A branch report is the useful one. A statement report says that the line ran. A
branch report says which side of the condition ran, and the untested side is
where the defect is.

### A skipped test carries an issue and a date

```ts
// Wrong: the test is disabled with no record.
// Failure: nobody knows what the test asserted, whether the behavior still
// exists, or who turned it off. The line survives three refactors, and the
// behavior it guarded breaks in the second one.
it.skip("rolls the optimistic row back where the write fails", async () => {});
```

```ts
// Correct: the skip names the reason, the issue, and the date it expires.
// Flaky under the parallel run. See ENG-4312. Re-enable by 2026-10-01.
it.skip("rolls the optimistic row back where the write fails", async () => {});
```

A skip that passes its date is deleted, with the test. That deletion is
honest. A skip that stays is a claim of coverage that the suite does not have.

The same rule covers `test.fixme` in Playwright, and a whole spec that a
condition excludes.

### The quarantine, and the flaky-test rate

A test that passes only on a retry is quarantined, not celebrated. The
quarantine list holds the test, the owner, the issue, and the date. A list with
no date does not shrink.

Track one number: the count of runs in which some test passed only after a
retry, over the count of runs. Read it weekly. A rise names a suite that loses
its value, and it gives that report before the team ignores a red run.

`references/end-to-end-journeys-and-flake-control.md` owns the causes and the
fix for each one. This file owns the policy and the number.

Never raise the retry count to lower the rate. That changes the measurement and
not the suite.

### The changed surface on a branch, and the whole suite on the main line

A large end-to-end list can run against the surface that a branch changed, and
the whole list on the main line after the merge. The gain in time is real, and
the risk is one thing: a change to a shared primitive changes every surface.

Select by the import graph, and never by the name of a folder. Where the
selection cannot be proved, run everything. A selection that misses a
dependency produces a green branch and a red main line, which costs more than
the minutes it saved.

The component stage is not selected. It is fast enough to run whole, and it is
the stage that returns the most defects.

### The acceptance criteria come before the code

A feature states its criteria as a list of tests, before any code is written.
The list is the definition of done for that feature, and a reviewer reads it
first.

```text
Acceptance criteria for the orders table
1. A signed-in agent sees the first page of their own orders.
2. An empty account renders the empty state, with a create control.
3. A 500 renders the error state, with a working retry control.
4. A 400 on the filter renders the message beside the filter control.
5. Page 2 follows the `next` URL from the response, and survives a reload.
6. A sort by total sends the ordering to the server, and the URL carries it.
7. axe reports no violation in the ready, empty, and error states.
```

Each line becomes a test at the level that
`references/test-strategy-and-component-tests.md` states. A criterion that no
test can express is a criterion that nobody can verify.

A bug follows the same shape with one line: the test that reproduces the
report.

### The completion claim carries the output

```text
Wrong: the report is a sentence.
Failure: the sentence is a prediction, and the reader cannot tell a run from a
guess. A suite that never ran, a suite that ran on a stale build, and a suite
that ran green all produce the same sentence.

  All tests pass and the build is clean.
```

```text
Correct: the report is the result of each command that ran.

  pnpm lint         clean, at --max-warnings=0
  pnpm typecheck    clean
  pnpm test         148 passed, 0 skipped, 4.2 s
  pnpm build        succeeded
  pnpm test:e2e     12 passed, 0 flaky, 1 m 51 s
```

`SKILL.md` states this as a standing rule: run the commands, and never assert
their result. This file states what the output must show. A skipped count above
zero and a flaky count above zero are both findings, and both belong in the
report rather than in a footnote.

### The lint rules for a test file

The static rules catch the mistakes that this domain repeats. Scope each plugin
to the glob that it applies to, so a rule about `screen` does not run over the
application code.

| The plugin | The glob | What it catches |
| --- | --- | --- |
| `eslint-plugin-testing-library` | `src/**/*.test.tsx` | An `await` on a `getBy*` query, a `container.querySelector`, a missing `await` on a `findBy*` |
| `@vitest/eslint-plugin` | `src/**/*.test.{ts,tsx}` | A focused test, a disabled test with no comment, an assertion outside a test |
| `eslint-plugin-playwright` | `e2e/**/*.ts` | `waitForTimeout`, a focused test, a conditional inside a test, a raw selector |
| `eslint-plugin-jsx-a11y` | The application globs | The static accessibility defects that `references/wcag-conformance-and-verification.md` owns |

`references/lint-format-and-scripts.md` owns the flat config array that holds
these entries, and the `--max-warnings=0` flag over the whole run. This file
owns which plugins a test suite needs.

### The signals that are not gates

| The signal | What it tells you | When it earns its cost | Who owns it |
| --- | --- | --- | --- |
| Storybook 9, with the interaction test and the accessibility addon | Each state of a primitive, in isolation, while the author builds it | Where a design system serves more than one application or more than one team. On a small team it is a second application to maintain, and the component suite already covers the states. | This file, for the decision. `references/component-styles-and-variants.md` for the primitive. |
| A visual difference service | A pixel change in a primitive, reviewed by a person | Where a design system has a review process that a person performs | `references/end-to-end-journeys-and-flake-control.md` for the run |
| Mutation testing | Whether the assertions would notice a changed operator | Over pure logic that carries money, permission, or a tax rule. Never over a component tree. | This file |
| Type coverage | The share of the tree that carries a real type | Where a team converts a legacy area, as a number that must go down | `references/typescript-config-and-enforcement.md` |
| Lighthouse CI and `size-limit` | The metric and the byte count of the built routes | Always, as the budget stage above | `references/performance-budgets-and-measurement.md` |
| A smoke run after a deploy | That the released build serves its critical routes | Always, once a deploy pipeline exists | Domain 22 `build-deploy-and-runtime-ops`. Not integrated yet. |
| A load run | The behavior of the system under a stated scenario | Before a launch, or after a change to the request pattern | The sibling skill `django-performance-optimizer`. The frontend states the scenario. |

Storybook deserves the honest answer. It is a real cost: a second build, a
second dependency tree, and a second place where a component can render
correctly while the application renders it wrongly. Take it where the audience
is outside the team that writes the code. Refuse it where the only reader is
the author, because the component test already renders every state and asserts
on them.

The Storybook accessibility addon covers a primitive while it is built. It
replaces no lane of
`references/wcag-conformance-and-verification.md`, and that domain holds a
veto.

### The libraries

The table gives each package its rule and its maintenance status. The dossier
for this domain carries no registry facts, so no cell states a version number
or a release date that this repository cannot confirm. Read the installed
version from `package.json` before you write code.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `eslint-plugin-testing-library` | The static rules over the component suite. | Current | Current | Testing Library, active | None |
| Recommend | `eslint-plugin-playwright` | The static rules over the journey suite. It reports `waitForTimeout`. | Current | Current | Playwright community, active | None |
| Recommend | `@vitest/eslint-plugin` | The static rules over the Vitest suite. It reports a focused test. | Current | Current | Vitest team, active | None |
| Conditional | `storybook` | Only where the audience for the primitives is outside the team that writes them. | Current | Current | Storybook team, active | None |
| Conditional | `@stryker-mutator/core` | Only over pure logic that carries money, permission, or a rate. The run is slow, so scope it to one folder. | Current | Current | Stryker team, active | None |
| Conditional | `type-coverage` | Only during a conversion, as a number with a direction. | Current | Current | Community, active | None |
| Reject | `continue-on-error` on a check step | It reports a failure and merges the change. | — | — | — | — |
| Reject | A coverage target over the whole repository | It buys the number and no confidence. | — | — | — | — |
| Reject | A raised retry count to lower the flaky rate | It changes the measurement, and not the suite. | — | — | — | — |
| Reject | `ladle` beside Storybook | Two component workshops in one repository is one too many. Pick one, or pick none. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| A broken suite merges | `continue-on-error`, or a check that branch protection does not require | Read the required checks beside the workflow | Remove the flag, and require the check |
| The first failure arrives after twenty minutes | The cheap stages run last, or run after the journeys | Read the stage order | Run the lint and the typecheck first |
| The branch is green and the main line is red | The selected end-to-end set missed a dependency | Compare the two runs | Select by the import graph, or run everything |
| Coverage is high and the change is untested | The number covers the repository | Read the report for the touched files | Read the branch report of the change |
| A skipped test outlives the release | The skip carries no date | Grep the suite for a skip | Delete it, or restore it |
| The team ignores a red run | The flaky rate rose with no owner | Read the rate over four weeks | Quarantine each entry with an owner and a date |
| A `test.only` hides the whole suite | No lint rule and no `forbidOnly` | Read the run count | Add the lint plugin, and set `forbidOnly` |
| Storybook renders correctly and the application does not | The story wraps a provider set that the application does not use | Compare the wrapper against the application tree | Take the one provider helper, or drop the story |
| An agent reports a green run that never happened | The claim carries no output | Ask for the command output | Paste the result of each command |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next 16 removed `next lint` | `rg -n 'next lint' package.json .github/` reports a hit | Call `eslint` directly, at `--max-warnings=0` |
| ESLint 9 takes the flat config only | An `.eslintrc` file in the repository | Move the entries into `eslint.config.ts`, which `references/lint-format-and-scripts.md` owns |
| The Vitest ESLint plugin moved to the `@vitest` scope | `rg -n 'eslint-plugin-vitest' package.json` reports a hit | Install `@vitest/eslint-plugin`, and rename the entry |
| Storybook 9 consolidated its addons into the core | A separate addon package for the interaction test | Read the installed major, and remove the package that the core now carries |
| Lighthouse removed the PWA category | An assertion on that category in the budget stage | Delete the assertion, which `references/performance-budgets-and-measurement.md` states |
| Playwright browsers are pinned to the library version | A cache key that omits the Playwright version | Key the browser cache by that version |

## Verification

```bash
# 1. Read the installed versions before you write code.
rg -n '"vitest"|"@playwright/test"|"eslint"|"storybook"' package.json

# 2. Run the gates in order, and read each exit code.
pnpm lint && pnpm typecheck && pnpm test && pnpm build && pnpm test:e2e

# 3. Find a step that is allowed to fail. This prints nothing.
rg -n 'continue-on-error|\|\| true' .github/ package.json

# 4. Find a skipped test with no issue reference. Read every hit.
rg -n 'it\.skip|test\.skip|test\.fixme|describe\.skip' src/ e2e/

# 5. Find a focused test. This prints nothing.
rg -n '\.only\(' src/ e2e/

# 6. Confirm that the test lint plugins are installed.
rg -n 'eslint-plugin-testing-library|eslint-plugin-playwright|@vitest/eslint-plugin' package.json

# 7. Confirm that each plugin is scoped to its own glob.
rg -n 'testing-library|playwright|vitest' -A3 eslint.config.ts

# 8. Read the branch coverage of the files that this change touched.
pnpm vitest run --coverage --changed origin/main

# 9. Find a test file with no assertion. Read every hit.
rg --files-with-matches 'it\(|test\(' -g '*.test.ts' -g '*.test.tsx' src/ \
  | xargs rg --files-without-match 'expect\('

# 10. Read the flaky entries of the last journey run.
npx playwright show-report

# 11. Confirm that the required checks match the stages.
gh api "repos/:owner/:repo/branches/main/protection" --jq '.required_status_checks.contexts'

# 12. Paste the output of each command into the completion report.
```

## Review checklist

- [ ] Does the pipeline run the lint and the typecheck before the tests?
- [ ] Does the end-to-end stage run only after the build succeeds?
- [ ] Is `continue-on-error` absent from every check step?
- [ ] Does branch protection require each stage that the pipeline runs?
- [ ] Does the review read the branch coverage of the touched files, rather
      than the repository number?
- [ ] Does every skipped test carry a reason, an issue, and a date?
- [ ] Is every skip that passed its date deleted?
- [ ] Does the project quarantine a flaky test with an owner and a date, rather
      than retry it into green?
- [ ] Is the flaky-test rate recorded, and read on a stated period?
- [ ] Is the retry count unchanged since the last release, or changed for a
      stated reason?
- [ ] Does a selected end-to-end run select by the import graph?
- [ ] Does the whole journey list run on the main line?
- [ ] Did the feature state its acceptance criteria as a test list before the
      code was written?
- [ ] Does each criterion map to a test at a stated level?
- [ ] Does the completion report hold the output of each command, rather than a
      sentence about it?
- [ ] Are the skipped count and the flaky count in that report, and are both
      zero?
- [ ] Does the flat config carry a test lint plugin for each suite, scoped to
      its glob?
- [ ] Does every signal in the project have a stated reason to exist, and does
      no signal block a merge?

## Handoffs

- The level of each test, the coverage threshold for a directory, and the
  determinism rules → `references/test-strategy-and-component-tests.md`.
- The handler, the failure shapes, and the contract test →
  `references/network-mocks-and-contract-tests.md`.
- The journey list, the flaky-test causes, and the trace →
  `references/end-to-end-journeys-and-flake-control.md`.
- The script names, the `check` command, the flat config array, and
  `--max-warnings=0` → `references/lint-format-and-scripts.md`.
- The pre-push hook, the commit message check, and the lockfile gate →
  `references/dependencies-and-git-workflow.md`.
- `tsc --noEmit`, the type-aware lint rules, and the type-level test →
  `references/typescript-config-and-enforcement.md`.
- The four accessibility lanes, the manual pass, and the baseline that must
  shrink → `references/wcag-conformance-and-verification.md`. That domain holds
  a veto.
- The budget stage, the Lighthouse configuration, and the byte gate →
  `references/performance-budgets-and-measurement.md`.
- The drift gate and `oasdiff` → `references/openapi-schema-and-codegen.md`.
- The primitive that a story renders →
  `references/component-styles-and-variants.md`.
- The audit of a new test dependency →
  `references/secret-boundary-and-supply-chain.md`. That domain holds a veto.
- The workflow file, the runners, the cache, the artifact upload, and the smoke
  run after a deploy → domain 22 `build-deploy-and-runtime-ops`. Not integrated
  yet.
- The error report and the alarm that a released build raises → domain 21
  `observability-and-resilience`. Not integrated yet.
- The Django suite, its runner, and its fixtures → the sibling skill
  `django-test-auditor`. This file owns the frontend gates only.
- The load scenario and the server verdict under it → the sibling skill
  `django-performance-optimizer`.
