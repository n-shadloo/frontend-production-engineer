# Component styles and variants

Tailwind CSS v4.3, `tailwind-variants` 3.x, `tailwind-merge` v3, `clsx` 2.x,
shadcn/ui on Base UI. This file owns the classes on a part, and the API that
selects them. The subjects are the merge of two class lists, the variant API of
a primitive, and the style hook that a primitive exposes. They also include the
focus ring, and the point at which a utility loses to plain CSS.

The tokens behind every class are `references/design-tokens-and-theming.md`.
The shape of the component and the parts that compose it are
`references/component-composition.md`. The layout classes and the type scale
are `references/layout-and-typography.md`.

## Principle

Two class lists that state the same property do not merge by themselves. The
last rule in the stylesheet wins, and the order of the stylesheet is not the
order of the call site.

A caller that passes a class expects that class to win. A component that cannot
grant that gets an override with `!important` instead.

A variant is a named choice, and not a string that a caller builds. The names
are the API of the component, and a reader sees every one of them in one place.

A primitive is the unit of style. A second element with the same appearance and
none of the behavior is a defect that looks correct.

A control with no visible focus is unusable with a keyboard. The indicator is
part of the design, and never a default that somebody removes.

A utility system has a limit. Past that limit, plain CSS is the clearer answer,
and not a longer class string.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### `cn()` merges every conditional class

```tsx
// Wrong: a template string builds the class list.
// Failure: a caller passes className="bg-black". The element carries both
// bg-primary and bg-black, and the winner is the rule that comes last in the
// generated stylesheet. The caller cannot predict it, and it can change when
// an unrelated class enters the project.
<button className={`px-4 py-2 ${danger ? "bg-red-600" : "bg-primary"} ${className}`} />
```

```tsx
// Correct: cn() runs clsx and then tailwind-merge, so the last class of the
// same property wins, and the caller wins over the component.
<button className={cn("px-4 py-2 bg-primary", danger && "bg-red-600", className)} />
```

```ts
// src/lib/utils.ts holds the one helper that the whole project imports.
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]): string {
  return twMerge(clsx(inputs));
}
```

`clsx` resolves the conditions, and `tailwind-merge` resolves the conflicts.
Neither one does the work of the other. `tailwind-merge` v3 reads the Tailwind
v4 class grammar, and v2.6.0 is the last line for Tailwind v3.

NEVER build a class list with a template string. NEVER defeat a primitive with
`!important`. An override that needs `!important` is a missing variant, or a
`cn()` call that the component never made.

### The variant API of a primitive

```tsx
// Correct: src/components/ui/button.tsx states every choice in one place.
import { tv, type VariantProps } from "tailwind-variants";

export const button = tv({
  base: "inline-flex items-center justify-center rounded-md text-sm font-medium outline-none focus-visible:ring-2 focus-visible:ring-ring disabled:opacity-50",
  variants: {
    variant: {
      default: "bg-primary text-primary-foreground hover:bg-primary/90",
      destructive: "bg-destructive text-white hover:bg-destructive/90",
      outline: "border bg-background hover:bg-accent",
    },
    size: { sm: "h-8 px-3", md: "h-9 px-4", lg: "h-10 px-6" },
  },
  defaultVariants: { variant: "default", size: "md" },
});

export type ButtonVariants = VariantProps<typeof button>;
```

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| A new primitive needs variants | `tailwind-variants`, with `tv()` and `VariantProps`. It resolves the class conflicts of its own slots since 3.3.0. | Never, for new code. | One dependency, and a second place beside `cn()` that resolves a conflict. |
| The project already runs `class-variance-authority` across many primitives | Keep CVA for those files, and state the reason in the pull request. | The project starts a migration of every primitive at once. | The project holds two variant APIs, and a reader must know both. |
| One element needs one conditional class | `cn()` alone. No variant API. | A second variant appears, so the choice becomes an API. | Nothing. |

`class-variance-authority` is audit-only. Its last release is 26 November 2024,
and it carries no known advisory. It works, and it is not a defect in an
existing project. NEVER add it to new code.

A new primitive states three things beyond its variants. It states whether it
is controlled or uncontrolled, it takes `ref` as a plain prop, and it accepts a
`className` that `cn()` merges last.
`references/component-composition.md` owns the first two, and it owns the
`render` prop that renders the primitive as another element.

### The primitive is the unit of style

```tsx
// Wrong: a div with the appearance of a button.
// Failure: the element has no button semantics, no keyboard activation, no
// disabled state, and no focus ring. It looks correct in the screenshot, and a
// keyboard user cannot reach it.
<div className="rounded-md bg-primary px-4 py-2 text-sm" onClick={submit}>
  Save
</div>
```

```tsx
// Correct: the primitive carries the behavior, and the variant names the
// appearance.
import { Button } from "@/components/ui/button";

<Button variant="default" size="md" onClick={submit}>
  Save
</Button>;
```

shadcn/ui copies its components into the repository, and `components.json`
records the choices that the CLI made. Those choices are the base library, the
style, the alias paths, and the CSS entry. Read that file before you add a
component, so the CLI writes into the folders that the project already uses.

Every shadcn primitive carries a `data-slot` attribute on each of its parts.
That attribute is the style hook for a parent that must change one part.

```tsx
// Correct: the parent styles one part of a primitive, and it needs no
// !important and no wrapper element.
<Dialog>
  <DialogContent className="[&_[data-slot=dialog-title]]:text-lg">…</DialogContent>
</Dialog>
```

Prefer a variant over a selector like this one. Use the selector where the
change belongs to one call site, and where a new variant would serve no second
caller.

### The focus ring is a token, and it never disappears

```tsx
// Wrong: the outline is removed, and nothing replaces it.
// Failure: a keyboard user cannot tell which control has focus. Every
// keyboard path through the surface becomes guesswork, and the task fails
// domain 10, which holds a veto.
<button className="outline-none" />
```

```tsx
// Correct: the default outline goes, and a token ring replaces it.
<button className="outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2" />
```

Write `outline-none` only beside a visible replacement in the same class list.
Confirm the indicator at 200 percent zoom before you ship the control.
Domain 10 `accessibility-wcag` owns the authoritative criterion for the size,
the contrast, and the zoom behavior of that indicator. It is not integrated
yet, and it holds a veto.

The Tailwind v4 ring default is 1px and `currentColor`. A project that upgraded
from v3 and kept `ring` alone now draws a thin ring in the text color. Set
`--color-ring` in the theme, and state the width.
`references/design-tokens-and-theming.md` owns that token.

### The run-time value, and the utility that cannot exist

Tailwind generates classes from the source text. A class that a run-time value
builds does not exist in the stylesheet.

```tsx
// Wrong: the class name is built at run time.
// Failure: the scanner never sees "translate-y-37", so no rule is generated.
// The element does not move, and no error appears.
<div className={`translate-y-${offset}`} />
```

```tsx
// Correct: the computed number goes in a style attribute, and every token
// value stays in a class.
<div className="bg-card text-card-foreground" style={{ transform: `translateY(${offset}px)` }} />
```

The `style` attribute is correct for a measured number alone. NEVER put a
color, a space, or a radius there. Those values have names, and the names are
in the theme.

### When a utility loses

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| The style is ordinary component style | Utilities, with the tokens of the theme. | Never. | Nothing. |
| A large keyframe set, or an override of third-party markup that the project does not own | A CSS Module, or a plain `@layer` block in the stylesheet. | The markup becomes markup that the project owns. | A second styling system, and a file that the class scanner does not read. |
| A whole design system with build-time contracts and typed style objects | `vanilla-extract`. State the reason, because it adds a build step. | The project has one application and one team. | A build step, and a second model of the tokens. |
| Any of the above, on a runtime CSS-in-JS library | Reject it. | Never on this stack. | The library forces a client boundary on a Server Component, and it costs run-time work on every render. |

This stack rejects Emotion and `styled-components`. They compute styles at run
time, and that computation cannot happen in a Server Component.
`references/server-and-client-components.md` owns the boundary that they force.

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| A caller's class does not win | A template string built the class list | Read the element in the inspector, and find both classes | Merge with `cn()` |
| A class appears twice, with two values for one property | `clsx` ran, and `tailwind-merge` did not | The same element carries `p-2` and `p-4` | Wrap the `clsx` call in `twMerge` |
| An override needs `!important` | The component never merged the caller's class, or the variant is missing | `!important` in a feature file | Add the variant, or add the `cn()` call |
| A class that a run-time value builds does nothing | The scanner never saw the full class name | The class is in the DOM, and no rule matches it | Put the computed number in `style`, and keep the token in a class |
| A copied primitive drifted from the rest | The element was hand-built rather than composed | Two buttons with different heights on one screen | Replace it with the primitive |
| A focus ring is thin and gray after the v4 upgrade | The ring default changed to 1px and `currentColor` | Tab through the surface | Set `--color-ring`, and state the width |
| A class outside the scanned paths produces nothing | The file lives outside the automatic source set | A class works in one folder and not in another | Add the path with `@source` |

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those four facts on 16 August 2026. A cell that
holds no date is a package with a current registry entry on that date. This
file does not state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `clsx` 2.x | The conditional class list, inside `cn()`. | 2.x | Current | Active | None |
| Recommend | `tailwind-merge` v3 | The conflict resolution, inside `cn()`. v3 is required for Tailwind v4, and v2.6.0 serves Tailwind v3. | 3.x | Current | Active | None |
| Recommend | `tailwind-variants` 3.x | The variant API for a new primitive. It resolves slot conflicts since 3.3.0. | 3.x | Current | Active | None |
| Recommend | `lucide-react` | The default icon set. Size an icon with `size-*`, and never with a width and a height class. | Current | Current | Active | None |
| Recommend | `sonner` | The toast. shadcn/ui deprecated its own toast component in favour of it. | Current | Current | Active | None |
| Conditional | CSS Modules | A large keyframe set, or an override of third-party markup. No dependency, because the framework supports it. | — | — | — | — |
| Conditional | `vanilla-extract` | A typed style contract across a design system. It adds a build step, so state the reason. | Current | Current | Active | None |
| Conditional | Storybook 9 | A shared primitive library that needs documented states, and interaction and visual tests. The overhead does not pay for a small application. | 9.x | 2025 | Active | None |
| Audit-only | `class-variance-authority` 0.7.1 | It works, and it has no advisory. Keep it in the files that hold it, and never add it to new code. | 0.7.1 | 26 November 2024 | No release for over 12 months | None |
| Reject | Emotion, `styled-components` | Run-time cost, and a forced client boundary on a Server Component. | — | — | — | — |
| Reject | `@axe-core/react` | It does not support React 18 or later, so it cannot run on React 19.2. Domain 10 owns the replacement. | — | — | — | — |

### Version discipline

Read the installed versions before you write code.

`tailwind-merge` v3 is the floor for Tailwind v4. A project on v2.6.0 with
Tailwind v4 merges the wrong classes in silence, because the class grammar
changed.

`tailwind-variants` resolves the class conflicts of its own slots since 3.3.0.
Below that version the project needs `tailwind-merge` for the slot output as
well.

`class-variance-authority` never merges conflicts. A CVA primitive still needs
`cn()` around its output and the caller's class.

React 19.2.4 is the security floor, for the reason that
`references/state-and-effects.md` states.

## Verification

```bash
# 1. Read the installed versions before you write code.
node -p "require('tailwind-merge/package.json').version"
node -p "require('tailwind-variants/package.json').version"

# 2. Find a class list built from a template string. Read every hit.
rg -n 'className=\{`[^`]*\$\{' src/

# 3. Confirm that one cn() helper exists, and that the project imports it.
rg -n 'export function cn' src/lib/
rg -c 'from "@/lib/utils"' -g '*.tsx' src/ | head

# 4. Find an !important in a feature file. This must print nothing.
rg -n '!important|![a-z-]+-\[' src/

# 5. Find outline-none with no replacement ring. Read every hit.
rg -n 'outline-none' src/ | rg -v 'ring'

# 6. Find a class name that a run-time value builds. This must print nothing.
rg -n 'className=\{[^}]*\$\{[^}]*\}[a-z-]' src/

# 7. Find a color or a space in a style attribute. This must print nothing.
rg -n 'style=\{\{[^}]*(#[0-9a-fA-F]{3,8}|padding|margin|color)' src/

# 8. Find class-variance-authority in new code. Read every hit.
rg -n 'class-variance-authority|\bcva\(' src/

# 9. Find a hand-built control that a primitive already covers. Read every hit.
rg -n '<div[^>]*onClick' -g '*.tsx' src/

# 10. Confirm components.json before you add a component with the CLI.
cat components.json

# 11. The typecheck proves the variant names against VariantProps.
pnpm typecheck
```

## Review checklist

- [ ] Does every conditional or merged class list pass through `cn()`?
- [ ] Does the project hold exactly one `cn()` helper?
- [ ] Does every primitive accept a `className` that `cn()` merges last?
- [ ] Are the variants of a primitive defined in one variant API, with the
      names typed?
- [ ] Is `class-variance-authority` absent from new code?
- [ ] Is `!important` absent from every feature file?
- [ ] Is every new UI element composed from a primitive, rather than hand-built
      from a `div`?
- [ ] Does every interactive element carry a visible focus indicator from a
      token?
- [ ] Does every `outline-none` sit beside a visible replacement in the same
      class list?
- [ ] Does a `style` attribute carry a computed number alone, and never a
      color, a space, or a radius?
- [ ] Is every class name a literal that the scanner can read?
- [ ] Does each drop to a CSS Module or to `@layer` state its reason?
- [ ] Is every run-time CSS-in-JS library absent from the project?

## Handoffs

- The tokens, the theme, the dark variant, and the `--color-ring` value →
  `references/design-tokens-and-theming.md`.
- The decomposition, the slots, the controlled and uncontrolled choice, and
  `ref` as a prop → `references/component-composition.md`. That file also owns
  the prop that renders a primitive as another element.
- The container query, the logical property, and the type scale →
  `references/layout-and-typography.md`.
- The client boundary that a run-time styling library forces →
  `references/server-and-client-components.md`.
- The `components/ui` folder, and the module that may import it →
  `references/directory-and-module-boundaries.md`.
- The class order, `prettier-plugin-tailwindcss`, and the Tailwind lint plugins
  → `references/lint-format-and-scripts.md`.
- The `satisfies` operator behind a typed variant map →
  `references/type-modeling-and-narrowing.md`.
- The accessible name, the role, the keyboard path, and the authoritative focus
  indicator criterion → domain 10 `accessibility-wcag`. Not integrated yet.
  That domain holds a veto.
- The field control, the error state on it, and the resolver behind it → domain
  11 `forms-and-validation`. Not integrated yet.
- The cell, the column, and the row of a table → domain 12
  `data-tables-and-visualization`. Not integrated yet.
- The transition and the animation that a variant change triggers → domain 14
  `motion-and-interaction`. Not integrated yet.
- The words inside a control, and the message beside it → domain 15
  `ux-writing-and-content-design`. Not integrated yet.
- The CSS bytes and the icon bytes that a surface adds → domain 16
  `performance-and-web-vitals`. Not integrated yet.
- The supply chain of a style dependency, and its advisories → domain 17
  `frontend-security`. Not integrated yet. That domain holds a veto.
