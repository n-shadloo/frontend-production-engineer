# End-to-end journeys and flake control

Playwright, Next.js 16.3, React 19.2.6 or later, TypeScript 5.9, a Django and
DRF backend. This file owns the journey that a real browser runs against a real
build, and the stability of the suite that holds it. The subjects are the journeys
that earn the run time, the Playwright project for this stack, and the locator
that a journey uses. They also include the assertion that
waits, the sign-in that happens once, and the data that each test owns. The
last subjects are the interception for a failure path, the visual comparison,
and the flaky test.

The unit level and the component level are
`references/test-strategy-and-component-tests.md`. The handler that a component
test uses, and the contract test, are
`references/network-mocks-and-contract-tests.md`. The order of the gates and
the quarantine policy are `references/merge-gates-and-quality-signals.md`.

## Principle

An end-to-end test is the only test that proves the whole path: the build, the
route, the request, the backend, and the browser. Nothing else reports that the
product works.

It is also the slowest test and the least stable one. Its cost is paid on every
branch, by every person who waits for it. Nobody runs a suite that grows
without a limit, and a suite that nobody runs is not a gate.

A journey is a sequence that a person performs to reach an outcome. A page is
not a journey, and a route scan is not a journey.

A wait for a duration is a guess about a machine. The guess is too long on a
fast laptop and too short on a loaded runner. Both numbers are wrong.

A test that passes on the second attempt did not pass. The retry converted a
report into silence.

Two tests that share one record share one result. Isolation is a property of
the data, and not of the code.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The journeys, and nothing else

| The journey | Why it is on the list |
| --- | --- |
| Sign in, and sign out | A failure locks every user out of the product |
| The primary conversion | A failure stops the revenue or the core act of the product |
| Create, read, update, and delete the core record | A failure breaks the work that the product exists for |
| One recovery path | A user must be able to leave a failure state without a reload |

Four journeys are a full list for most products. Add one where a new flow
carries money or identity. Remove one where a component test now covers the
same behavior with the network mocked.

A route scan is not a journey. `references/wcag-conformance-and-verification.md`
requires an `@axe-core/playwright` check on every route, and that domain holds
a veto. That check runs as its own spec, in its own project, and it does not
count against the journey list. The two suites share the browser and the
config, and they answer different questions.

### The Playwright project for this stack

```ts
// Wrong: playwright.config.ts starts the development server.
// Failure: `next dev` compiles a route on the first request, ships development
// React, and disables the prefetch. A route that fails the production build
// passes here, and every timing in the run belongs to a build that no user
// loads.
webServer: { command: "pnpm dev", url: process.env.E2E_BASE_URL },
```

```ts
// Correct: playwright.config.ts serves the production build, and keeps the
// evidence of a failure.
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./e2e",
  fullyParallel: true,
  forbidOnly: Boolean(process.env.CI),
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 4 : undefined,
  reporter: process.env.CI ? [["html"], ["github"]] : [["list"]],
  use: {
    baseURL: process.env.E2E_BASE_URL,
    trace: "on-first-retry",
    video: "retain-on-failure",
    screenshot: "only-on-failure",
    reducedMotion: "reduce",
  },
  webServer: {
    command: "pnpm build && pnpm start",
    url: process.env.E2E_BASE_URL,
    reuseExistingServer: !process.env.CI,
    timeout: 180_000,
  },
  projects: [
    { name: "setup", testMatch: /global\.setup\.ts/ },
    {
      name: "chromium",
      use: { ...devices["Desktop Chrome"], storageState: "e2e/.auth/agent.json" },
      dependencies: ["setup"],
    },
    {
      name: "mobile",
      use: { ...devices["Pixel 7"], storageState: "e2e/.auth/agent.json" },
      dependencies: ["setup"],
    },
  ],
});
```

`reducedMotion: "reduce"` removes the largest single source of timing noise in
this stack. An entrance that runs for 300 ms moves the target of a click. A
click that lands during a transition reports an element that is not stable.
`references/motion-primitives-and-reduced-motion.md` owns the reduced variant
that the application must already carry, so the run exercises a real code path
rather than a test-only one.

`forbidOnly` fails the run where a `test.only` reached the branch. Without it,
one focused test hides the whole suite and the pipeline stays green.

### The locator is the role and the name

```ts
// Wrong: the locator reads the markup.
// Failure: a designer wraps the control in one more element, or a utility
// class is renamed. The selector matches nothing, the failure names a timeout,
// and nobody can tell a broken product from a moved div.
await page.click(".order-table > tbody > tr:nth-child(2) button.primary");
```

```ts
// Correct: the locator is what a user sees, and what a screen reader
// announces.
await page.getByRole("row", { name: /ORD-1001/ })
  .getByRole("button", { name: "Mark as shipped" })
  .click();
```

Take `getByRole` with a `name` first, then `getByLabel` for a control, then
`getByText` for content. Take `getByTestId` only where the element carries no
role, no name, and no stable text, and state the reason above the line.
`references/semantics-and-accessible-names.md` owns the role and the name that
these locators depend on.

Never take a CSS descendant chain or an XPath expression. Both read the shape
of the markup, which the product is free to change.

### The assertion waits, and the test never does

```ts
// Wrong: the test sleeps.
// Failure: 2000 ms is 1900 ms of waste on a fast machine, and 500 ms short on
// a loaded runner. The suite is both slow and flaky, and the number teaches
// the next person to add a larger one.
await page.click("text=Save");
await page.waitForTimeout(2000);
expect(await page.textContent(".toast")).toBe("Order saved");
```

```ts
// Correct: the web-first assertion retries until the deadline.
await page.getByRole("button", { name: "Save order" }).click();
await expect(page.getByRole("status")).toHaveText("Order saved");
```

`expect(locator)` re-queries and re-checks until it passes or the timeout ends.
`expect(value)` does neither, so an assertion over a value read once is a race.
Read the value inside the locator assertion, and never into a variable first.

`waitForTimeout` has no correct use in this suite. Neither does a `sleep`
helper, an arbitrary `setTimeout`, or a raised default timeout that hides a
real delay.

### The sign-in happens once

```ts
// Correct: e2e/global.setup.ts signs in through the interface once, and writes
// the state that every project reuses.
import { expect, test as setup } from "@playwright/test";

const stateFile = "e2e/.auth/agent.json";

setup("authenticate the agent", async ({ page }) => {
  await page.goto("/sign-in");
  await page.getByLabel("Email").fill(process.env.E2E_USER_EMAIL ?? "");
  await page.getByLabel("Password").fill(process.env.E2E_USER_PASSWORD ?? "");
  await page.getByRole("button", { name: "Sign in" }).click();
  await expect(page.getByRole("heading", { name: "Orders" })).toBeVisible();
  await page.context().storageState({ path: stateFile });
});
```

The state file holds a live session cookie. Treat it as a credential. Add
`e2e/.auth/` to `.gitignore`, and never upload it as a pipeline artifact.
`references/secret-boundary-and-supply-chain.md` owns that rule, and it holds a
veto. `references/dependencies-and-git-workflow.md` owns the ignore list.

The refresh token sits in an `httpOnly` cookie, which
`references/session-and-token-lifecycle.md` requires. The stored state
therefore carries a real cookie and no token in web storage. A setup that
writes a token into `localStorage` describes an application that this
repository refuses.

One journey must not reuse the state: the sign-in journey itself. Give it an
empty `storageState`, so it proves the path that the setup takes for granted.

A test that changes the account, such as a password change or a role change,
takes its own worker-scoped user. A shared account and a parallel run give one
test the state of another.

### Each test owns its data

```ts
// Wrong: the test names a record that a seed created once.
// Failure: two workers edit ORD-1001 at the same time and one assertion reads
// the other's value. A second run on the same database finds the record
// already shipped, and the test fails on a product that works.
test("marks order ORD-1001 as shipped", async ({ page }) => {
  await page.goto("/orders/ORD-1001");
});
```

```ts
// Correct: a fixture creates the record for this test, with a unique value.
import { test as base, expect } from "@playwright/test";

export const test = base.extend<{ orderRef: string }>({
  orderRef: async ({ request }, use, testInfo) => {
    const reference = `E2E-${testInfo.workerIndex}-${testInfo.testId}`;
    const created = await request.post("/api/orders/", {
      data: { reference, quantity: 1 },
    });
    expect(created.ok()).toBe(true);
    await use(reference);
    await request.delete(`/api/orders/${reference}/`);
  },
});
```

The fixture sends the request through `request`, which carries the stored
session state. It creates before the test and removes after it, so a failed run
leaves no record that the next run reads.

The unique value comes from the worker index and the test id. A timestamp is
not unique under a parallel run on one machine.

Where the backend offers a transactional reset or a seed command, it owns both.
This file owns only the call that the frontend makes, and the rule that no
frontend test depends on a record that another test wrote.

### The failure path uses interception

A journey must also prove that a failure renders. A backend that is down proves
the wrong thing, and it takes the whole suite with it.

```ts
// Correct: one route fails for one test, and the rest of the product runs.
test("shows the error state and keeps a retry control", async ({ page }) => {
  await page.route("**/api/orders/**", (route) =>
    route.fulfill({ status: 500, json: { detail: "Server error" } }),
  );
  await page.goto("/orders");
  await expect(page.getByRole("alert")).toContainText("We could not load your orders.");
  await expect(page.getByRole("button", { name: "Try again" })).toBeEnabled();
});
```

`page.route` also covers the offline path, the slow response, and the 403. It
answers the question that a seeded backend cannot answer without a special
state. `references/error-and-empty-state-copy.md` owns the words that these
assertions read.

A dropped socket needs the same treatment. Playwright carries
`routeWebSocket` for that path, and
`references/push-transport-and-connection.md` states the behavior to assert: a
degraded state, and never an empty list.

### The visual comparison

```ts
// Wrong: the comparison covers a page with a relative time on it.
// Failure: "2 minutes ago" becomes "3 minutes ago", the pixels differ, and the
// suite reports a design regression that nobody made. After the third such
// report, the team updates every snapshot with no review.
await expect(page).toHaveScreenshot();
```

```ts
// Correct: the comparison covers one component, and it masks the region that
// moves.
await expect(page.getByTestId("order-card")).toHaveScreenshot("order-card.png", {
  mask: [page.getByTestId("relative-time")],
  maxDiffPixelRatio: 0.01,
});
```

Keep this lane to the primitives of the design system and a small set of key
layouts. A screenshot of a whole application page fails on every content
change, and it reports nothing about the design.

One font paints different pixels on an operating system and inside a container.
Produce and compare the screenshots inside the same container image that CI
runs, or every local update rewrites every file.
`references/design-tokens-and-theming.md` owns the theme that each comparison
must cover in both modes.

`references/charts-and-visual-encoding.md` owns the chart itself. A chart is a
reasonable target for this lane, because its output is a picture and no text
assertion reaches it.

### The flaky test

A flaky test is a broken test. It is fixed or it is deleted. It is never
retried into green.

`retries: 2` in the configuration exists to produce a trace, and not to produce
a pass. Read the run report: a test that passed on a retry is reported as
flaky, and that report is the work item.

| The cause | How it looks | The fix |
| --- | --- | --- |
| A fixed delay | The failure moves between machines and between hours | Delete the delay, and assert on the locator |
| A value read once | The assertion names the value that the first read returned | Assert on the locator, and never on a variable |
| Shared data | The test passes alone and fails in the suite | Create the record inside the test, with a unique value |
| An order dependence | A reversed order changes the result | Remove the state that one test leaves for another |
| An animation | The click lands and nothing happens | Set `reducedMotion: "reduce"`, and keep the reduced variant real |
| A real network call | The failure names a timeout and no assertion | Intercept the route, or seed the state |
| A parallel write to one account | Two workers report two different roles | Give the test its own worker-scoped user |
| A stale build | The failure names an element that the branch removed | Build inside `webServer`, and never reuse a served build in CI |

Diagnose with the trace. The trace viewer holds the DOM at each step, the
network log, and the console output. It answers what the element was at the
moment of the failure, which no screenshot answers.

A test that cannot be fixed today is quarantined, and the quarantine carries an
issue link and a date. `references/merge-gates-and-quality-signals.md` owns
that policy, and the rule that no test is skipped without one.

Never add a `sleep` to make a run green. It converts a report into a delay, and
the defect ships.

### The libraries

The table gives each package its rule and its maintenance status. This skill
cannot confirm a version number or a release date for them, so no cell states
one. Read the installed version from `package.json` before you write code.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `@playwright/test` | The runner, the fixtures, the projects, and the trace. One tool for the journey, the visual comparison, and the route scan. | Current | Current | Microsoft, active | None |
| Recommend | The Playwright trace viewer | The first tool to open on a failure. It holds the DOM, the network, and the console at each step. | Ships with Playwright | Current | Microsoft, active | None |
| Recommend | `@axe-core/playwright` | The route scan that `references/wcag-conformance-and-verification.md` requires. That domain holds a veto. | Current | Current | Deque, active | None |
| Conditional | Playwright component testing | Only where a component needs a real layout or a real pointer. React Testing Library covers the rest here, and two component runners is one too many. It is experimental. | Ships with Playwright | Current | Microsoft, active | None |
| Conditional | A container image for the visual lane | Only where the project runs a visual comparison. The image must match the CI image exactly. | — | — | — | — |
| Conditional | `k6` | Only for a load scenario that the frontend defines. The run and the server verdict belong to the backend. | Current | Current | Grafana Labs, active | None |
| Reject | `cypress` in a new project here | It runs one browser process per spec and carries its own assertion library. Playwright already covers the journey, the visual lane, and the route scan. | — | — | — | — |
| Reject | `selenium-webdriver` | It has no auto-wait, so every test needs an explicit poll. | — | — | — | — |
| Reject | `page.waitForTimeout` | There is no correct use of it in this suite. | — | — | — | — |
| Reject | A journey suite that runs only on the main branch | A gate that runs after the merge is a report, and not a gate. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| Every locator times out on CI and passes locally | `webServer` reused a stale served build | Read the first failure and the server log | Build inside `webServer`, and set `reuseExistingServer` to false in CI |
| A route that the build breaks passes the suite | The suite runs `next dev` | Read the `webServer` command | Serve the production build |
| The suite is green and one test never ran | A `test.only` reached the branch | Read the run count | Set `forbidOnly` under CI |
| A click lands and nothing happens | An entrance animation moved the target | Open the trace at the click step | Set `reducedMotion: "reduce"` |
| The failure names a timeout and no assertion | The locator is a CSS chain over changed markup | Read the locator in the trace | Query by role and accessible name |
| Two workers report two different roles | One account serves the parallel run | Run with one worker | Give the test a worker-scoped user |
| A rerun on the same database fails | The test edits a seeded record | Run the same spec twice | Create the record in a fixture, and remove it after |
| Every visual file changes on a laptop | The images were produced outside the CI image | Compare two runs of one commit | Produce and compare inside the CI image |
| A test passes on the second attempt every week | A real defect under a race | Read the flaky report of the run | Fix it, or quarantine it with an issue link and a date |
| The auth state file appears in a pull request | `e2e/.auth/` is not ignored | Read the diff | Add the folder to `.gitignore`, and rotate the account |

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Web-first assertions replaced the read-then-assert form | `rg -n 'await page\.(textContent\|innerText)\(' e2e/` reports a hit | Assert on the locator, which retries |
| `page.getBy*` replaced the string selector form | `rg -n 'page\.(click\|fill)\("' e2e/` reports a hit | Take the role locator, and then the action on it |
| `storageState` with a setup project replaced `globalSetup` for auth | A `globalSetup` key beside a project list | Take the setup project, and a `dependencies` entry |
| Next 16 serves with `next start` after `next build` | A `webServer` command that runs `next dev` | Build first, then serve the build |
| Next 16 renamed `middleware.ts` to `proxy.ts` | A journey that asserts on a redirect from the old file | Read `references/app-router-structure.md`, and assert on the served result |
| Playwright browsers are downloaded, and they age with the library | `npx playwright install --with-deps` absent from the pipeline | Install the browsers for the installed version, and cache them by that version |

## Verification

```bash
# 1. Read the installed version before you write code.
rg -n '"@playwright/test"|"@axe-core/playwright"' package.json

# 2. Install the browsers for the installed version.
npx playwright install --with-deps

# 3. Confirm that the served target is the production build.
rg -n 'webServer' -A4 playwright.config.ts

# 4. Find a fixed delay. This prints nothing.
rg -n 'waitForTimeout|setTimeout\(|sleep\(' e2e/

# 5. Find a CSS or an XPath locator. Read every hit.
rg -n 'page\.(click|fill|locator)\("[.#/]' e2e/

# 6. Find a read-then-assert assertion. Read every hit.
rg -n 'await page\.(textContent|innerText|inputValue)\(' e2e/

# 7. Find a spec that names a seeded record. Read every hit.
rg -n 'ORD-[0-9]{4}' e2e/

# 8. Confirm that the auth state folder is ignored.
rg -n 'e2e/.auth' .gitignore

# 9. Find a focused test. This prints nothing.
rg -n 'test\.only|describe\.only' e2e/

# 10. Find a skipped test with no issue link. Read every hit.
rg -n 'test\.skip|test\.fixme' e2e/

# 11. Run the journey suite against the production build.
pnpm test:e2e

# 12. Run it three times. The three results are identical.
pnpm test:e2e && pnpm test:e2e && pnpm test:e2e

# 13. Read the flaky report of the last run, and open the trace of each entry.
npx playwright show-report
```

## Review checklist

- [ ] Does the journey list hold only journeys, and does each one carry a
      stated reason?
- [ ] Does `webServer` build and serve the production build, rather than run
      the development server?
- [ ] Is `forbidOnly` set under CI?
- [ ] Is `reducedMotion` set to `reduce` in the shared `use` block?
- [ ] Does every locator take a role and an accessible name, or a stated
      reason for a test id?
- [ ] Is every assertion a web-first assertion on a locator?
- [ ] Is `waitForTimeout` absent from the whole suite?
- [ ] Does the sign-in run once in a setup project, and does every other
      project reuse the stored state?
- [ ] Does the sign-in journey itself run with an empty stored state?
- [ ] Is the stored state folder ignored by git, and absent from the pipeline
      artifacts?
- [ ] Does each test create the record that it needs, with a value that is
      unique per worker?
- [ ] Does each test remove what it created?
- [ ] Does a failure path use route interception, rather than a backend that is
      down?
- [ ] Does the visual lane cover primitives and key layouts only, with the
      moving regions masked?
- [ ] Are the visual images produced in the same container image that CI runs?
- [ ] Does the route accessibility scan run as its own spec, outside the
      journey list?
- [ ] Is `trace` set to at least `on-first-retry`?
- [ ] Did a test that passed on a retry get an issue, rather than a shrug?
- [ ] Does every skipped test carry an issue link and a date?

## Handoffs

- The unit level, the component level, the provider helper, and the
  determinism rules → `references/test-strategy-and-component-tests.md`.
- The MSW handler, the failure shapes, and the contract test →
  `references/network-mocks-and-contract-tests.md`.
- The gate order, the quarantine policy, and the flaky-test rate →
  `references/merge-gates-and-quality-signals.md`.
- The `@axe-core/playwright` route scan, the four verification lanes, and the
  manual pass → `references/wcag-conformance-and-verification.md`. That domain
  holds a veto.
- The role and the accessible name that a locator depends on →
  `references/semantics-and-accessible-names.md`.
- The reduced variant that the application must already carry →
  `references/motion-primitives-and-reduced-motion.md`.
- The words that an error assertion reads →
  `references/error-and-empty-state-copy.md`.
- The theme that a visual comparison must cover in both modes →
  `references/design-tokens-and-theming.md`.
- The chart that the visual lane covers →
  `references/charts-and-visual-encoding.md`.
- The session cookie, the refresh, and the sign-out →
  `references/session-and-token-lifecycle.md`.
- The dropped connection and the degraded state →
  `references/push-transport-and-connection.md`.
- The `.gitignore` entry and the pre-push hook →
  `references/dependencies-and-git-workflow.md`.
- The stored state that must never leave the machine →
  `references/secret-boundary-and-supply-chain.md`. That domain holds a veto.
- The production build that the run serves →
  `references/app-router-structure.md`.
- The seed command, the transactional reset, and the test-only endpoint on the
  server → the backend.
- The Django process, the health check, and the container that CI brings up →
  the backend.
- The load run and the server verdict under it → the backend. This file owns
  only the scenario that the frontend states.
