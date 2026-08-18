# Test strategy and component tests

Vitest, React Testing Library, `@testing-library/user-event` v14,
`@testing-library/jest-dom`, `@faker-js/faker`, Next.js 16.3, React 19.2.6 or
later, TypeScript 5.9. This file owns the decision of what gets a test, and at
which level. It also owns the component test, which carries most of the value.
The subjects are the five levels with a rule for each, the code that never gets
a test, and coverage as a diagnostic. They also include the Vitest project for
a Next application, the query order of React Testing Library, and the one
provider helper. The last subjects are the determinism that a suite needs, the
factory that produces the data, and the boundary that no runner renders.

The handler that answers a request is
`references/network-mocks-and-contract-tests.md`. The journey in a real browser
is `references/end-to-end-journeys-and-flake-control.md`. The order of the
gates, the coverage report, and the flaky-test policy are
`references/merge-gates-and-quality-signals.md`.

## Principle

A test exists so that a person can change the code and know what broke. It has
no other purpose. A test that reports work rather than defects costs more than
it returns.

A refactor changes the implementation and keeps the behavior. A test that
fails on such a refactor read the implementation. The test was wrong, and not
the refactor.

A user finds a control by what it is and by what it is called. A test that
finds the same control another way asserts on a tree that no user meets.

Coverage measures the lines that ran. It does not measure the behavior that a
test proved. A high number over weak assertions is a comfortable number.

A test that can fail for a reason outside the change is not a gate. It is a
delay with a red icon.

An agent cannot verify its own work by inspection. The suite is the only
mechanism that reports a defect the agent did not predict.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The five levels, and the rule for each

| The level | What it covers | The rule | The tool |
| --- | --- | --- | --- |
| Unit | A pure function: a schema, a formatter, an error normalizer, a permission rule, a query-key factory, a reducer | Take it where the logic has a branch and no DOM | Vitest |
| Component | The visible behavior of one feature, with the network mocked | Take it for every feature. This level returns the most defects for the cost. | Vitest, React Testing Library, MSW |
| Contract | The generated client and the handlers against the committed schema | Take it once for the client, and once for each handler set | Vitest and the schema |
| End to end | A critical journey in a real browser | Take it for the sign-in, for the primary conversion, and for the create, read, update, and delete of the core record | Playwright |
| Visual | A primitive of the design system, and a small set of key layouts | Take it where a pixel carries the meaning | Playwright screenshots |

The shape is a trophy, and not a pyramid. A React unit test over a component
proves that the component renders, which nobody doubts. The component level sits
above it because it exercises the same path that a user takes: the markup, the
event, the request, and the state that follows.

The end-to-end level is the most expensive and the least stable. Keep it to the
journeys that lose money or lock a user out. Nobody runs a journey suite that
grows past a few minutes, and a suite that nobody runs is not a gate.

### What never gets a test

Five kinds of code get no test of their own.

- An implementation detail: a private function, an internal state name, a class
  list, a call count on a router.
- A third-party package. The package has its own suite.
- A generated file. `src/api/generated/` is an output of the schema, and
  `references/openapi-schema-and-codegen.md` owns the gate over it.
- A styling value that no rule depends on. A spacing token is not behavior.
- A constant.

```tsx
// Wrong: the assertion reads the class list.
// Failure: a designer renames the utility from `text-red-600` to
// `text-destructive`, the message still renders in red for the user, and the
// suite goes red for nobody.
expect(screen.getByTestId("error")).toHaveClass("text-red-600");
```

```tsx
// Correct: the assertion reads what the user reads.
expect(
  await screen.findByText("We could not save the order."),
).toBeInTheDocument();
```

### Coverage is a diagnostic, and never a target

A global number of 100 forces a test over a constant and over a generated file.
It buys the number and no confidence. Read coverage to find a branch that no
test reaches. Then decide whether that branch deserves one.

Set a threshold for each directory that holds logic. Leave the directories that
hold markup out of the threshold.

```ts
// Wrong: vitest.config.ts asks for one number over the whole tree.
// Failure: the number is met by tests that render a component and assert
// nothing, while the error normalizer keeps an unreached branch.
export default defineConfig({
  test: { coverage: { thresholds: { lines: 100 } } },
});
```

```ts
// Correct: the logic directories carry the threshold, and the rest do not.
export default defineConfig({
  test: {
    coverage: {
      provider: "v8",
      reporter: ["text", "lcov"],
      include: ["src/**"],
      exclude: ["src/api/generated/**", "src/**/*.stories.tsx"],
      thresholds: {
        "src/lib/**": { statements: 90, branches: 85 },
        "src/features/**/api/**": { statements: 90, branches: 85 },
      },
    },
  },
});
```

Confirm the glob form of `coverage.thresholds` in the installed Vitest. This
file states the shape of the option, and the installed package is the authority
on its API.

### The Vitest project for a Next application

```ts
// Correct: vitest.config.ts. The zone is set before any test module loads.
process.env.TZ = "UTC";

import react from "@vitejs/plugin-react";
import tsconfigPaths from "vite-tsconfig-paths";
import { defineConfig } from "vitest/config";

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  test: {
    environment: "jsdom",
    globals: false,
    restoreMocks: true,
    setupFiles: ["./src/test/setup.ts"],
    include: ["src/**/*.test.{ts,tsx}"],
    exclude: ["e2e/**"],
  },
});
```

`tsconfigPaths` gives the runner the `paths` aliases of `tsconfig.json`. Without
it, every `@/` import fails to resolve, and the failure reads as a missing
module. `references/typescript-config-and-enforcement.md` owns those aliases.

`exclude` keeps the Playwright specs out of the Vitest run. A `test()` from
`@playwright/test` inside a Vitest run throws, and the message names neither
runner.

### Where a test file sits

A unit test and a component test sit beside the module that they prove, as
`<module>.test.ts` or `<module>.test.tsx`. A reader who opens the folder sees
the test, and a module that moves takes its test with it.

A journey sits under `e2e/` at the root of the repository, because it proves the
served application and not one module. The shared setup, the factories, and the
handlers sit under `src/test/`, which no feature owns.

`references/directory-and-module-boundaries.md` owns the feature folder itself,
and the boundary rules over it. This file owns only the place of a test inside
that structure.

Two environments are cheaper than one. Pure logic needs no DOM, and a `node`
environment starts faster. Vitest carries a multi-environment option under
`projects` in the current major, and under `vitest.workspace.ts` in an earlier
one. Read the installed major before you write either.

### `globals: false` needs an explicit cleanup

```ts
// Wrong: src/test/setup.ts with no cleanup, under globals: false.
// Failure: React Testing Library registers its automatic cleanup only where
// `afterEach` is a global. The previous component stays in the document, the
// next `getByRole` matches two elements, and the second test fails with
// "Found multiple elements" in a file that nobody changed.
import "@testing-library/jest-dom/vitest";
```

```ts
// Correct: src/test/setup.ts unmounts after each test, and registers the
// matchers once for the project.
import { cleanup } from "@testing-library/react";
import { afterEach } from "vitest";
import "@testing-library/jest-dom/vitest";

afterEach(() => {
  cleanup();
});
```

`references/wcag-conformance-and-verification.md` owns the axe matchers that
this same setup file registers. That domain holds a veto.

### The query order

| Priority | The query | When it applies |
| --- | --- | --- |
| 1 | `getByRole`, with the `name` option | Every interactive element, and every heading, table, and region |
| 2 | `getByLabelText` | A form control, where the role query needs the label anyway |
| 3 | `getByPlaceholderText` | Only where no label exists, which is itself a defect |
| 4 | `getByText` | A passage of content that is not a control |
| 5 | `getByDisplayValue` | The current value of a filled control |
| 6 | `getByTestId` | An element with no role, no name, and no text. State the reason above the line. |

```tsx
// Wrong: the test reaches for a test id.
// Failure: the button has no accessible name, so a screen reader announces
// "button" and nothing else. The test id hides the defect and the suite is
// green. The accessibility gate catches it later, or a user does.
await user.click(screen.getByTestId("save-button"));
```

```tsx
// Correct: the query is the same one that assistive technology uses.
await user.click(screen.getByRole("button", { name: "Save order" }));
```

A `getByRole` query that cannot find a control is a report about the markup.
Fix the element, the role, or the accessible name.
`references/semantics-and-accessible-names.md` owns all three.

### `user-event` over `fireEvent`

```tsx
// Wrong: the event is dispatched directly.
// Failure: `fireEvent.change` sets the value in one step. It fires no
// pointer event, no focus event, and no key event. A control that opens on
// focus stays closed, a mask that reads each key never runs, and the test
// passes over a path that no user can take.
fireEvent.change(screen.getByLabelText("Quantity"), { target: { value: "3" } });
```

```tsx
// Correct: the interaction is the sequence that a person performs.
const user = userEvent.setup();
await user.type(screen.getByLabelText("Quantity"), "3");
```

Call `userEvent.setup()` once for each test, before the render. Every `user`
method returns a promise, and each one must be awaited. An unawaited call is
the most common cause of an `act` warning in this stack.

An `act` warning names a state update that happened outside the act boundary.
Read it as a real report. The usual cause is an awaited call that nobody
awaited, or an assertion that runs before a pending request settles.

### The four states of a data view get four tests

`references/server-state-and-query-cache.md` states the four states that every
data view owns. A component test covers each one, because each one is a
different tree.

```tsx
// Wrong: the test asserts that the spinner appears.
// Failure: the request fails, the spinner never leaves, and this assertion
// still passes. The test proves that the component mounts, and nothing else.
render(<OrderList />);
expect(screen.getByRole("status")).toBeInTheDocument();
```

```tsx
// Correct: each state carries its own test, and each one asserts on the
// content that the user reads.
it("renders the rows once the request settles", async () => {
  render(<OrderList />);
  expect(await screen.findByRole("row", { name: /ORD-1001/ })).toBeVisible();
});

it("renders the empty state where the account has no order", async () => {
  server.use(emptyOrdersHandler);
  render(<OrderList />);
  expect(await screen.findByText("No orders yet")).toBeVisible();
});

it("renders the error state and keeps a retry control", async () => {
  server.use(serverErrorHandler);
  render(<OrderList />);
  expect(await screen.findByRole("alert")).toHaveTextContent(
    "We could not load your orders.",
  );
  expect(screen.getByRole("button", { name: "Try again" })).toBeEnabled();
});
```

`findBy*` waits for the element and retries the query. Use it for anything that
arrives after a request. Use `waitFor` only where the assertion is not a query,
such as a call count or a URL. Never pass a timeout to either one to make a
test pass.

### One provider helper, and one `QueryClient` for each test

```tsx
// Wrong: the test file builds a client at module scope.
// Failure: one cache serves every test in the file. A row that test one wrote
// is still there in test three, and the order of the file decides the result.
// The file passes alone and fails in the suite.
const queryClient = new QueryClient();
```

```tsx
// Correct: src/test/render.tsx builds a client for each render, and turns the
// retry off so a failure path resolves in one attempt.
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { render, type RenderOptions } from "@testing-library/react";
import type { ReactElement, ReactNode } from "react";

export function renderWithProviders(
  ui: ReactElement,
  options?: Omit<RenderOptions, "wrapper">,
) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false, staleTime: 0 } },
  });

  function Providers({ children }: { children: ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>
        <ThemeProvider>{children}</ThemeProvider>
      </QueryClientProvider>
    );
  }

  return { queryClient, ...render(ui, { wrapper: Providers, ...options }) };
}
```

The helper wraps the providers that the application wraps: the query client,
the theme, and the locale. A test that builds its own subset renders a tree
that production never renders.
`references/locale-routing-and-catalogs.md` owns the provider for the messages,
and `references/design-tokens-and-theming.md` owns the theme.

A mutation that fails must roll back. Assert the row after the rejection, and
not the call.

```tsx
// Correct: the optimistic row is gone after the failure, and the old value is
// back on the screen.
it("rolls the optimistic row back where the write fails", async () => {
  server.use(patchOrderFails);
  const user = userEvent.setup();
  renderWithProviders(<OrderRow order={order} />);

  await user.click(screen.getByRole("button", { name: "Mark as shipped" }));
  expect(await screen.findByRole("alert")).toBeVisible();
  expect(screen.getByRole("cell", { name: "Pending" })).toBeVisible();
});
```

### A hook is proved through the component that uses it

`renderHook` belongs to a hook that other features import. A hook that one
component uses is proved by the test over that component.

```ts
// Correct: a shared hook gets its own test, through the same providers.
const { result } = renderHook(() => useOrderFilters(), {
  wrapper: Providers,
});
```

### The suite is deterministic

Four inputs make a suite give two answers for one commit. They are the clock,
the zone, the random source, and the state that one test leaves for the next.

```ts
// Wrong: the test builds a date from the clock.
// Failure: the assertion reads "2 days ago" until a Monday, and "4 days ago"
// after it. The suite goes red on a day that nobody changed the code.
const created = new Date();
```

```ts
// Correct: the clock is frozen for the test, and released after it.
beforeEach(() => {
  vi.useFakeTimers();
  vi.setSystemTime(new Date("2026-03-01T09:00:00Z"));
});

afterEach(() => {
  vi.useRealTimers();
});
```

`vi.useFakeTimers()` stops the timers that `user-event` needs. Pass the advance
function to `userEvent.setup({ advanceTimers: vi.advanceTimersByTime })` where
a test uses both.

The zone is set once, in `vitest.config.ts`, and never in a shell command that
one machine runs and another does not. A machine in Tehran and a runner in UTC
must produce the same output.
`references/locale-formatting-and-calendars.md` owns the rendered zone itself.

Seed the random source. `faker.seed(1)` in the setup file makes every generated
name repeat, and a failure message names a value that a person can search for.

### The factory produces the data

```ts
// Wrong: one object serves every test.
// Failure: a test sets `fixture.status = "shipped"` for its own case. Every
// later test in the run reads "shipped", and the file passes only in one
// order.
export const order = { id: "ORD-1001", status: "pending", total: "12.00" };
```

```ts
// Correct: a builder returns a new record, and the caller states the part
// that its case needs.
import { faker } from "@faker-js/faker";
import type { components } from "@/api/generated/schema";

type Order = components["schemas"]["Order"];

export function buildOrder(overrides: Partial<Order> = {}): Order {
  return {
    id: faker.string.uuid(),
    reference: `ORD-${faker.number.int({ min: 1000, max: 9999 })}`,
    status: "pending",
    total: "12.00",
    currency: "EUR",
    created_at: "2026-03-01T09:00:00Z",
    ...overrides,
  };
}
```

The builder returns the generated type, so a renamed serializer field fails the
typecheck inside the test data as well as inside the application.
`references/boundary-validation-and-api-types.md` owns that type.

### Mock the network, and never a module of this repository

```ts
// Wrong: the API client is replaced.
// Failure: the test proves that the component calls a function that no longer
// exists in production. The parse, the error normalizer, the timeout, and the
// header are all skipped, so every defect in them ships green.
vi.mock("@/lib/api/client");
```

```ts
// Correct: the request leaves the client and a handler answers it.
server.use(
  http.get("*/api/orders/", () => HttpResponse.json(paginated([buildOrder()]))),
);
```

`references/network-mocks-and-contract-tests.md` owns the handler and the
server. `vi.mock` stays for a browser API that jsdom does not implement. It
also stays for a module that reaches a real clock outside `fetch`. State the
reason above the call.

Never mock the module under test. A test that replaces the thing it asserts on
proves that the replacement works.

### The snapshot is small, and it is inline

```ts
// Wrong: the whole tree becomes the assertion.
// Failure: the file grows past a thousand lines, every change updates it, and
// no reviewer reads the diff. The snapshot records the markup and asserts no
// behavior.
expect(container).toMatchSnapshot();
```

```ts
// Correct: the snapshot covers one pure output, and the value is in the file.
expect(formatOrderTotal({ total: "1234.5", currency: "EUR" }))
  .toMatchInlineSnapshot(`"€1,234.50"`);
```

### The boundary that no runner renders

A Server Component, a Server Action, and a Route Handler run in the framework
and not in jsdom. No component runner renders one today. Two moves cover them,
and a mock of `next/headers` covers neither.

```tsx
// Wrong: the test calls the page function and replaces the request APIs.
// Failure: the mock returns a plain object where the framework returns an
// awaited store. The test proves the shape of the mock. A real request
// reaches a different code path, and the page throws in production.
vi.mock("next/headers", () => ({ cookies: () => ({ get: () => "abc" }) }));
const tree = await OrdersPage({ params: Promise.resolve({ id: "1" }) });
```

```ts
// Correct: the decision leaves the boundary as a pure function, and it gets a
// unit test.
export function selectVisibleOrders(
  orders: readonly Order[],
  role: Role,
): readonly Order[] {
  return role === "agent" ? orders : orders.filter((o) => o.status !== "draft");
}
```

The rendered result of the boundary is proved end to end, against a running
build. `references/end-to-end-journeys-and-flake-control.md` owns that run.
`references/app-router-structure.md` owns the boundary itself, and
`references/data-access-and-mutations.md` owns the order of the work inside a
Server Action.

### A bug fix starts with a failing test

Write the test that reproduces the report. Run it, and watch it fail for the
reason in the report. Then fix the code, and watch the same test pass. A test
written after the fix proves that the code does what it does.

The same rule gives an agent its only proof. A test that has never failed
proves nothing about the assertion inside it. Break the behavior on purpose
once, confirm the red, and restore it.

### The libraries

The table gives each package its rule and its maintenance status. The dossier
for this domain carries no registry facts, so no cell states a version number
or a release date that this repository cannot confirm. Read the installed
version from `package.json` before you write code.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `vitest` | The runner for the unit level and the component level. It reads the Vite config, so the aliases and the transform match the application. | Current | Current | Vitest team, active | None |
| Recommend | `@testing-library/react` | The render and the queries. The query order above is the contract. | Current | Current | Testing Library, active | None |
| Recommend | `@testing-library/user-event` | Version 14 or later. Every method returns a promise. | Current | Current | Testing Library, active | None |
| Recommend | `@testing-library/jest-dom` | The matchers, through the `/vitest` entry point. | Current | Current | Testing Library, active | None |
| Recommend | `@faker-js/faker` | The generated value inside a builder, under a fixed seed. | Current | Current | faker-js, active | None |
| Recommend | `vite-tsconfig-paths` | The `paths` aliases of `tsconfig.json`, inside the runner. | Current | Current | Community, active | None |
| Recommend | `@vitejs/plugin-react` | The JSX transform for the component level. | Current | Current | Vite team, active | None |
| Conditional | `happy-dom` | Only where the jsdom start cost dominates the run. It implements less of the platform, so a passing test can hide a real defect. | Current | Current | Community, active | None |
| Conditional | Vitest browser mode | Only where a test needs a real layout or a real pointer. Playwright already covers that ground here. | Ships with Vitest | Current | Vitest team, active | None |
| Conditional | `@stryker-mutator/core` | Only over pure logic that carries money or permission. `references/merge-gates-and-quality-signals.md` states when it earns the run time. | Current | Current | Stryker team, active | None |
| Reject | `jest` in a new Next 16 project | It needs a second transform config beside Vite, and it starts slower on the same suite. It is alive only in legacy code. | — | — | — | — |
| Reject | `enzyme` | It reads the internal state of a component, which the doctrine of this file refuses. | — | — | — | — |
| Reject | `react-test-renderer` | It is deprecated, and it renders no DOM. | — | — | — | — |
| Reject | A DOM snapshot over a page | It records markup and asserts no behavior. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| "Found multiple elements with the role" | No cleanup runs under `globals: false` | Run the file alone, and then in the suite | Call `cleanup()` in an `afterEach` inside the setup file |
| An `act` warning on a passing test | An unawaited `user-event` call, or an assertion before the request settles | Read the stack of the warning | Await every `user` call, and query with `findBy*` |
| The file passes alone and fails in the suite | A shared client, a shared fixture object, or a frozen clock that nobody released | Run with a reversed order | Build the client and the record inside the test |
| A relative date assertion goes red on one day | The test builds a date from the clock | Run the file with the machine clock moved | Freeze the time with `vi.setSystemTime` |
| A currency or a date reads one way locally and another in CI | The zone comes from the machine | Compare the two outputs | Set `process.env.TZ` in `vitest.config.ts` |
| Every `@/` import fails to resolve | The runner does not read the `tsconfig.json` aliases | Read the module-not-found path | Add `vite-tsconfig-paths` to the plugins |
| A Playwright spec throws inside the Vitest run | `include` covers the `e2e` folder | Read the failing path | Exclude `e2e/**` in `vitest.config.ts` |
| Coverage is high and defects still ship | The tests render and assert little | Delete one assertion and re-run | Assert on the content that the user reads, and read the branch report |
| A refactor breaks twenty tests and no behavior | The tests read the implementation | Read what each failed assertion names | Query by role and name, and assert on the rendered result |

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| `user-event` 14 made every method asynchronous | `rg -n 'userEvent\.(click\|type)\(' src/ \| rg -v 'await'` reports a hit | Call `userEvent.setup()` once, and await every method |
| `fireEvent` is alive only in legacy code | `rg -n 'fireEvent\.' src/` reports a hit | Replace it with the `user-event` sequence for the same act |
| `@testing-library/jest-dom` publishes a Vitest entry point | An import of the bare package inside a Vitest setup file | Import `@testing-library/jest-dom/vitest` |
| `react-test-renderer` is deprecated | `rg -n 'react-test-renderer' package.json` reports a hit | Render through `@testing-library/react` |
| Vitest moved the multi-environment option to `projects` | A `vitest.workspace.ts` beside a current Vitest major | Read the installed major, and take the option that it documents |
| Next 16 removed `next lint` | A test script that expects `next lint` to run a check | Call `eslint` and `vitest` directly |

## Verification

```bash
# 1. Read the installed versions before you write code.
rg -n '"vitest"|"@testing-library/react"|"@testing-library/user-event"|"@faker-js/faker"' package.json

# 2. Run the unit and component levels.
pnpm test

# 3. Read the coverage branch report, and find the logic with no test.
pnpm vitest run --coverage

# 4. Find a class-list assertion. Read every hit.
rg -n 'toHaveClass' src/

# 5. Find a test id in a test file. Each hit needs a stated reason above it.
rg -n 'getByTestId|findByTestId|data-testid' src/

# 6. Find a mock of a module that this repository owns. Read every hit.
rg -n "vi\.mock\(['\"]@/" src/

# 7. Find an unawaited user-event call. This prints nothing.
rg -n 'userEvent\.[a-z]+\(|user\.[a-z]+\(' -g '*.test.tsx' src/ | rg -v 'await|setup\('

# 8. Find a QueryClient at the module scope of a test file. This prints nothing.
rg -n --multiline '^const [a-zA-Z]+ = new QueryClient' -g '*.test.tsx' src/

# 9. Find a date built from the clock inside a test. Read every hit.
rg -n 'new Date\(\)|Date\.now\(\)' -g '*.test.ts' -g '*.test.tsx' src/

# 10. Confirm that the setup file unmounts after each test.
rg -n 'cleanup' src/test/setup.ts

# 11. Confirm that the runner sets the zone.
rg -n 'process\.env\.TZ' vitest.config.ts

# 12. Find a whole-tree snapshot. This prints nothing.
rg -n 'toMatchSnapshot\(\)' src/

# 13. Break one behavior on purpose, and confirm that a test fails for it.
pnpm test

# 14. Run the suite three times. The three results are identical.
pnpm test && pnpm test && pnpm test
```

## Review checklist

- [ ] Does the change carry a test at the level that the table above states?
- [ ] Does every component test assert on the content that a user reads, rather
      than on a class list, a call count, or internal state?
- [ ] Does every query take the highest priority that the element allows?
- [ ] Does every test id carry a stated reason above it?
- [ ] Does the test suite use `user-event`, with every call awaited?
- [ ] Does the feature carry a test for the ready state, the empty state, the
      error state, and the loading path?
- [ ] Does a failed mutation carry an assertion that the rolled-back value is
      on the screen?
- [ ] Does each render build its own `QueryClient`, through the one provider
      helper?
- [ ] Is every mock a network handler, rather than a module of this repository?
- [ ] Does each `vi.mock` that remains carry a stated reason?
- [ ] Does the test data come from a builder, rather than from a shared object?
- [ ] Is the random source seeded?
- [ ] Is the clock frozen wherever a test reads a relative date?
- [ ] Does `vitest.config.ts` set the time zone for the process?
- [ ] Is every snapshot inline, and over one pure output?
- [ ] Is a Server Component, a Server Action, or a Route Handler proved by
      extracted logic and an end-to-end run?
- [ ] Is a mock of `next/headers` absent from the suite?
- [ ] Did the bug fix start with a test that failed for the reported reason?
- [ ] Does the coverage threshold cover the logic directories, rather than the
      whole tree?
- [ ] Did the suite run three times with identical results?

## Handoffs

- The handler that answers a request, the DRF failure shapes, and the contract
  test → `references/network-mocks-and-contract-tests.md`.
- The journey in a real browser, the stored sign-in state, the visual
  comparison, and the flaky-test policy →
  `references/end-to-end-journeys-and-flake-control.md`.
- The order of the gates, the coverage report in CI, and the rule that no test
  is skipped without an issue →
  `references/merge-gates-and-quality-signals.md`.
- The `test`, `test:watch`, and `test:e2e` scripts, and the `check` command →
  `references/lint-format-and-scripts.md`.
- `expectTypeOf`, `assertType`, and the `*.test-d.ts` file →
  `references/typescript-config-and-enforcement.md`.
- The axe matcher, the four verification lanes, and the manual pass →
  `references/wcag-conformance-and-verification.md`. That domain holds a veto.
- The role, the accessible name, and the label that a query depends on →
  `references/semantics-and-accessible-names.md`.
- The four states of a data view, the query key, and the optimistic write →
  `references/server-state-and-query-cache.md`.
- The generated type that a builder returns, and the parse at the boundary →
  `references/boundary-validation-and-api-types.md`.
- The Server Component, the Server Action, and the route file →
  `references/app-router-structure.md` and
  `references/data-access-and-mutations.md`.
- The locale and the zone that a rendered value takes →
  `references/locale-formatting-and-calendars.md`.
- The feature folder, the module boundary around it, and the generated client
  path → `references/directory-and-module-boundaries.md`.
- The Django test suite, the factory on the server, and the pytest fixture →
  the sibling skill `django-test-auditor`. This file changes nothing on the
  server.
