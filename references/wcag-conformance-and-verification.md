# WCAG conformance and verification

WCAG 2.2 Level AA, WAI-ARIA 1.2, the ARIA Authoring Practices Guide,
`eslint-plugin-jsx-a11y`, `vitest-axe`, and `@axe-core/playwright`. This file
owns the conformance target of the product, and the evidence that the product
meets it. The subjects are the criteria of WCAG 2.2, the four verification
lanes, and the limit of every automated tool. They also include the manual
pass, the gate in the pipeline, the output contract of a component, and the
legal obligation behind the whole domain.

The element, the role, and the name are
`references/semantics-and-accessible-names.md`. The tab path, the focus, and
the announcement are `references/keyboard-focus-and-live-regions.md`. The
contrast, the target size, and the reflow are
`references/visual-and-motor-criteria.md`.

## Principle

Conformance is a claim about the page that the user receives. It is not a claim
about the intention of the author, and it is not a claim about the design file.

An automated tool reads the markup. It cannot read the sense of an alternative
text, follow a keyboard path, or hear an announcement. It is a floor, and never
a proof.

A check that runs after the merge is a report. A check that runs before the
merge is a gate. Only a gate changes what ships.

A failure that a screenshot cannot show needs a person who completes the flow.
That person needs a keyboard, a screen reader, and ten minutes.

The obligation is legal in several markets. It is a requirement of the product,
and not a preference of the team.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The target is WCAG 2.2 Level AA

Every criterion at Level A and at Level AA applies. Read the level and the
exact wording of a criterion from the WCAG 2.2 Recommendation. This file does
not restate them, and NEVER state a level from memory.

The four principles give a diagnostic frame. Perceivable and Robust are mostly
`references/semantics-and-accessible-names.md` and
`references/visual-and-motor-criteria.md`. Operable is mostly
`references/keyboard-focus-and-live-regions.md` and
`references/visual-and-motor-criteria.md`. Understandable crosses all three.

WCAG 2.2 added these criteria. Each one is new work for an application
interface.

| The criterion | Where this skill states it |
| --- | --- |
| Focus Not Obscured (Minimum) 2.4.11 | `references/keyboard-focus-and-live-regions.md` |
| Focus Not Obscured (Enhanced) | `references/keyboard-focus-and-live-regions.md` |
| Focus Appearance | `references/keyboard-focus-and-live-regions.md`, with the ratio in `references/visual-and-motor-criteria.md` |
| Dragging Movements 2.5.7 | `references/visual-and-motor-criteria.md` |
| Target Size (Minimum) 2.5.8 | `references/visual-and-motor-criteria.md` |
| Consistent Help 3.2.6 | `references/visual-and-motor-criteria.md` |
| Redundant Entry 3.3.7 | `references/semantics-and-accessible-names.md` |
| Accessible Authentication 3.3.8 | `references/semantics-and-accessible-names.md` |

Seven criteria of WCAG 2.1 fail most often in a single-page application. A
server-rendered page passes them by accident, and a client navigation breaks
them.

| The criterion | Where this skill states it |
| --- | --- |
| Reflow 1.4.10 | `references/visual-and-motor-criteria.md` |
| Text Spacing 1.4.12 | `references/visual-and-motor-criteria.md` |
| Content on Hover or Focus 1.4.13 | `references/keyboard-focus-and-live-regions.md` |
| Status Messages 4.1.3 | `references/keyboard-focus-and-live-regions.md` |
| Orientation | `references/visual-and-motor-criteria.md` |
| Pointer Gestures | `references/visual-and-motor-criteria.md` |
| Motion Actuation | `references/visual-and-motor-criteria.md` |

WCAG 3.0 is a draft, and APCA is the contrast method inside it. Neither one is
a conformance target today. Measure against WCAG 2.2, and never claim
conformance from an APCA result.

### The obligation

| The market | The instrument |
| --- | --- |
| The United States | The Americans with Disabilities Act, and Section 508 for federal procurement |
| The European Union | The European Accessibility Act, in force since June 2025, and EN 301 549 for public procurement |
| Ontario, Canada | The Accessibility for Ontarians with Disabilities Act |

Two documents are deliverables of a conformant product. The accessibility
statement tells a user what the product supports and how to report a problem.
The conformance report records the result against each criterion, in the VPAT
format or as an Accessibility Conformance Report.

This skill states that the obligation exists. It gives no legal advice, and a
lawyer decides what a market requires of a specific product.

### What an automated tool finds, and what it does not

An automated tool finds about 30 to 40 percent of the real failures. The
remaining majority needs a person.

A tool cannot judge whether an alternative text describes the image. It cannot
judge whether the heading order matches the structure of the content. It cannot
follow a keyboard path through a flow, hear an announcement, or decide that a
label is wrong rather than absent.

Treat a clean axe result as the start of the review, and never as the end of
it.

### The four lanes

| The lane | The tool | What it catches | What it cannot catch |
| --- | --- | --- | --- |
| The editor and the lint gate | `eslint-plugin-jsx-a11y` | A static defect in the JSX: a missing `alt`, a handler on a generic element, an invalid role, a positive `tabIndex` | Anything that depends on the rendered tree, the painted color, or the interaction |
| The component test | `vitest-axe`, or `jest-axe` on a Jest project | A rule violation in the rendered markup of one component, in a simulated DOM | A contrast ratio, a focus order across a page, an announcement |
| The route test | `@axe-core/playwright` | A rule violation on a whole route in a real browser, including the contrast | The keyboard path, the screen-reader output, and the sense of a name |
| The manual pass | A keyboard, a screen reader, the zoom, and the forced-colors mode | Everything above, and the majority of the failures | Nothing that this table covers |

Three tools produce a per-page report for a stakeholder. They are Lighthouse,
Pa11y, and IBM Equal Access. They add a report, and they replace no lane above.
The Storybook accessibility addon covers each state of a primitive while the
author builds it. It says nothing about the application that composes the
primitives.

NEVER add `@axe-core/react`. It does not support React 18 or later, so it
cannot run on React 19.2. The four lanes above are the replacement.

### The lint gate

`eslint-plugin-jsx-a11y` is the cheapest lane, and it runs on every save.
`references/lint-format-and-scripts.md` owns the flat config array and the
`--max-warnings=0` rule that makes a warning fail the pipeline. This file owns
the reason that the plugin is in that array.

The plugin reads the JSX alone. It cannot follow a component that renders
another component, so a defect inside a primitive passes it. The component
tests cover that gap.

### The component test

```ts
// Correct: src/test/setup.ts registers the matchers once for the project.
import { expect } from "vitest";
import * as matchers from "vitest-axe/matchers";

expect.extend(matchers);
```

```tsx
// Wrong: the test asserts on the default state alone.
// Failure: the empty state renders no heading, and the error state renders a
// message that no live region carries. Neither state is in the test, so the
// lane reports a pass and both defects ship.
import { render } from "@testing-library/react";
import { expect, it } from "vitest";
import { axe } from "vitest-axe";
import { OrderTable } from "./order-table";

it("has no axe violations", async () => {
  const { container } = render(<OrderTable rows={rows} />);
  expect(await axe(container)).toHaveNoViolations();
});
```

```tsx
// Correct: each state that a user can meet carries its own assertion.
import { render } from "@testing-library/react";
import { expect, it } from "vitest";
import { axe } from "vitest-axe";
import { OrderTable } from "./order-table";

it.each([
  ["ready", <OrderTable rows={rows} />],
  ["empty", <OrderTable rows={[]} />],
  ["error", <OrderTable rows={[]} error="The request failed" />],
])("has no axe violations in the %s state", async (_state, ui) => {
  const { container } = render(ui);
  expect(await axe(container)).toHaveNoViolations();
});
```

Confirm the matcher import path in the installed version before you write the
setup file. Do not assume the path from another project.

A component test runs in a simulated DOM. It renders no pixels, so it reports
nothing about a contrast ratio. Send the contrast to the route lane.

### The route test

```ts
// Wrong: the run gates on a tag that the installed version does not know.
// Failure: axe matches no rule, the violation list is empty, and the
// assertion passes. The route reports a clean result, and nothing ran.
const results = await new AxeBuilder({ page }).withTags(["wcag23aa"]).analyze();
expect(results.violations).toEqual([]);
```

```ts
// Correct: e2e/a11y.spec.ts checks each route in a real browser, and it
// asserts that the run matched rules before it asserts the result.
import AxeBuilder from "@axe-core/playwright";
import { expect, test } from "@playwright/test";

const routes = ["/", "/orders", "/orders/new", "/settings"];

for (const route of routes) {
  test(`${route} has no axe violations`, async ({ page }) => {
    await page.goto(route);
    const results = await new AxeBuilder({ page })
      .withTags(["wcag2a", "wcag2aa", "wcag21a", "wcag21aa", "wcag22aa"])
      .analyze();
    expect(results.passes.length + results.violations.length).toBeGreaterThan(0);
    expect(results.violations).toEqual([]);
  });
}
```

Confirm the tag names and the result shape in the installed version of axe-core
before you gate on them. A tag that the version does not know runs no rules and
reports a pass.

Run this lane in the light theme and in the dark theme. The contrast rule
produces a different result in each one.

`references/test-strategy-and-component-tests.md` owns the runner, the
fixtures, and the harness of the component lane.
`references/end-to-end-journeys-and-flake-control.md` owns them for the browser
lane. This file owns the assertions inside both.

### The manual pass

Complete these five steps on every feature, before you call the work done.

1. Unplug the mouse. Complete the whole flow with the keyboard alone, and
   watch the focus indicator at every stop.
2. Run one screen reader through the primary flow. Confirm the name, the role,
   and the state of every control, and confirm every announcement.
3. Set the zoom to 400 percent, or the viewport to 320 pixels wide. Confirm
   that no surface scrolls in two directions.
4. Turn on the dark theme and the forced-colors mode. Confirm that every
   surface keeps its bounds.
5. Set the reduced-motion preference, and repeat the flow. Confirm that no
   transform animation plays.

| The platform | The pair to run |
| --- | --- |
| Windows | NVDA with Firefox, or NVDA with Chrome |
| macOS | VoiceOver with Safari |
| iOS | VoiceOver with Safari |
| Android | TalkBack with Chrome |
| Windows, a second reader | Narrator |

One pair is the minimum for a feature. Choose the pair that the users of the
product have. The readers differ in what they announce, so a result on one pair
is evidence and not a proof.

Record the keyboard walkthrough in the pull request. A reviewer who sees the
steps can repeat them.

### The gate in the pipeline

| The stage | The check | The failure condition |
| --- | --- | --- |
| Lint | `eslint-plugin-jsx-a11y`, at `--max-warnings=0` | Any report |
| Unit and component tests | `vitest-axe` on every new component | Any violation |
| End-to-end tests | `@axe-core/playwright` on every route | Any violation |
| Review | The manual pass of five steps | Any step that fails |

A project with a backlog of failures takes a baseline. Record the current
violations, fail the pipeline on a new one, and shrink the baseline on a stated
schedule. A baseline that never shrinks is a permanent failure with a
comfortable name.

`references/lint-format-and-scripts.md` owns the scripts that these stages
call. `references/release-pipeline-and-rollback.md` owns the workflow file.

### The output contract

Report these facts for each component that the work adds or changes.

- The APG pattern that it implements, or the statement that it implements
  none.
- Its keyboard map.
- The source of its accessible name.
- Its live-region behavior, where it reports an asynchronous result.
- Its axe result.

Report one more fact for each feature. It is the completed manual pass, with
the five steps and the screen-reader pair.

Write the accessibility criteria into the definition of done of the feature,
before the work starts. A criterion that arrives after the code is a change
request.

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| A clean tool result on a page that a keyboard cannot use | The lanes ran, and the manual pass did not | Complete the flow with no pointer | Run the five steps |
| A violation that the component test missed | The defect lives in the composition, and not in one component | The route lane in a real browser | Add the route to the end-to-end spec |
| A contrast failure that no test reports | The check ran in a simulated DOM | Compare the two lanes | Move the contrast check to the route lane |
| A pass on a route that has no rules | An axe tag that the installed version does not know | Print the rule count of the run | Confirm the tag names |
| A dark theme that fails after the light theme passed | The lane ran in one theme | Run the route lane in both themes | Add the second run |
| A pipeline that reports and never fails | The lint gate accepts warnings | Read the lint script | Add `--max-warnings=0` |
| A baseline that grows every month | Nobody owns the schedule that shrinks it | Compare two baselines | Set the schedule, and fail on a new violation |

### Version discipline

Read the installed versions before you write code.

`@axe-core/react` cannot run on React 19.2, and
`references/component-styles-and-variants.md` states the same. Take the four
lanes above.

`vitest-axe` and `jest-axe` are two packages for two runners. A project takes
the one that matches its runner, and never both.

Confirm the axe tag names, the matcher import path, and the `AxeBuilder`
signature in the installed versions. This file states the shape of each call,
and the installed package is the authority on its API.

## Verification

```bash
# 1. Read the installed versions before you write code.
node -p "require('axe-core/package.json').version"
node -p "require('eslint-plugin-jsx-a11y/package.json').version"

# 2. Confirm that the lint config holds the accessibility plugin.
rg -n 'jsx-a11y' eslint.config.*

# 3. Confirm that the lint script fails on a warning.
rg -n '"lint"' package.json

# 4. Confirm the matcher setup of the component lane.
rg -n 'vitest-axe|jest-axe' src/ vitest.config.* package.json

# 5. Confirm that every route is in the end-to-end accessibility spec.
rg -n 'AxeBuilder' e2e/
rg --files -g 'page.tsx' src/app/

# 6. Run the three automated lanes.
pnpm lint
pnpm test
pnpm test:e2e

# 7. Complete the five manual steps on the feature, and record the result in
#    the pull request.
```

## Review checklist

- [ ] Is the conformance target stated as WCAG 2.2 Level AA?
- [ ] Does the lint config hold `eslint-plugin-jsx-a11y`, at
      `--max-warnings=0`?
- [ ] Does every new component carry an axe assertion in its test?
- [ ] Does the component lane cover the error state, the empty state, and the
      loading state?
- [ ] Does every route carry an `@axe-core/playwright` check?
- [ ] Does the route lane run in the light theme and in the dark theme?
- [ ] Are the axe tag names confirmed against the installed version?
- [ ] Is `@axe-core/react` absent from the project?
- [ ] Did the five manual steps run on this feature?
- [ ] Is the screen-reader pair named in the pull request?
- [ ] Does each component report its pattern, its keyboard map, its name
      source, its live-region behavior, and its axe result?
- [ ] Does the feature carry accessibility criteria in its definition of done?
- [ ] Does the pipeline fail on a new violation, rather than report it?
- [ ] Does a baseline of known failures carry a schedule that shrinks it?

## Handoffs

- The element, the role, the name, and the field label →
  `references/semantics-and-accessible-names.md`.
- The tab path, the focus, the modal, and the announcement →
  `references/keyboard-focus-and-live-regions.md`.
- The contrast, the target size, the reflow, and the user preferences →
  `references/visual-and-motor-criteria.md`.
- The flat config array, the `--max-warnings=0` rule, and the `package.json`
  scripts → `references/lint-format-and-scripts.md`.
- The React 19.2 floor that rejects `@axe-core/react` →
  `references/component-styles-and-variants.md` and
  `references/state-and-effects.md`.
- The dark theme that the route lane must also run →
  `references/design-tokens-and-theming.md`.
- The test runner and the fixtures →
  `references/test-strategy-and-component-tests.md`. The MSW handlers →
  `references/network-mocks-and-contract-tests.md`. The Playwright project
  config → `references/end-to-end-journeys-and-flake-control.md`.
- The workflow file that runs these stages →
  `references/release-pipeline-and-rollback.md`.
- The supply chain of a test dependency →
  `references/secret-boundary-and-supply-chain.md`. That domain holds a veto.
- The words of the accessibility statement →
  `references/interface-copy-and-voice.md`.
