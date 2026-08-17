# Design tokens and theming

Tailwind CSS v4.3, Next.js 16.3, React 19.2.4 or later, shadcn/ui on Base UI.
This file owns the value layer of the interface. The subjects are the token,
the CSS block that publishes it, and the color space that holds it. They also
include the dark theme, the class that selects it, and the first paint that
must carry it.

The classes on a part, the variant API, and `cn()` are
`references/component-styles-and-variants.md`. The layout, the type scale, and
the font are `references/layout-and-typography.md`. Where a stored preference
lives is `references/client-and-url-state.md`.

## Principle

A value that a feature file states is a decision that nobody can change in one
place. A token is that one place.

A design system publishes names. Feature code reads the name, and it never
reads the value behind the name.

A dark theme is a second design. It is never a filter over the first one, and
it is never a mechanical inversion of it.

A theme that arrives after the first paint is a defect that the user sees. The
correct theme is on the document before the browser paints.

A color space that follows human perception makes an equal step look equal. A
lightness ramp in such a space needs no correction by hand.

A token that no theme publishes produces no style. The definition and the
publication are two acts, and both are necessary.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### Tokens or nothing

A feature file holds no raw value. NEVER write a hex color, a `px` font size,
or a spacing number that the theme does not hold. Add the value to the theme
first, and then consume the utility that the theme generates.

```tsx
// Wrong: three raw values in a feature component.
// Failure: the brand color changes, and a search for "#4f46e5" misses the two
// files that wrote it as "#4F46E5". The dark theme has no value for any of
// the three, so the card keeps its light appearance under the dark class.
<article className="bg-[#4f46e5] p-[7px] text-[13px]">{title}</article>
```

```tsx
// Correct: every value is a token, and the dark theme redefines each one.
<article className="bg-primary p-2 text-sm">{title}</article>
```

The exception is a value that the run time computes, such as a distance from a
measurement. Put that one value in a `style` attribute, and take every color
and every space from the theme. `references/component-styles-and-variants.md`
holds that rule and its detector.

### One color token needs three places

Define a semantic token for the light theme. Redefine it for the dark theme.
Publish it to the utility layer. Three places, and every one of them is
necessary.

```css
/* Wrong: the token is defined and never published.
   Failure: bg-warning generates no CSS. The class is in the DOM, no rule
   matches it, and the element keeps the background of its parent. */
:root {
  --warning: oklch(0.84 0.16 84);
}
```

```css
/* Correct: define it, redefine it for dark, and publish it. */
:root {
  --warning: oklch(0.84 0.16 84);
}
.dark {
  --warning: oklch(0.41 0.11 46);
}
@theme inline {
  --color-warning: var(--warning);
}
```

`@theme inline` publishes the name to the utility layer, and it keeps the
reference to the custom property. The theme value is therefore the current
value of `--warning`, which the `.dark` block overrides. A plain `@theme` block
copies the value at build time, so the dark override never reaches the utility.

### The color space is OKLCH

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| A new color value enters the theme | Write it as `oklch()` in `:root` and in `.dark`, and publish it through `@theme inline`. | Never. A mixed theme of `oklch()` and `#rrggbb` values has two lightness models, and no ramp is even. | Each value needs a conversion once, and a reviewer must read a notation that is new to the team. |
| The project supports a browser below the OKLCH floor | State a fallback declaration before the `oklch()` line. | The support target rises above the floor. | One more declaration for each token, and two values to keep in agreement. |
| A value must derive from another token | `color-mix()` over the token, or the slash opacity syntax such as `bg-primary/90`. | The derived value needs its own name in the design, so it becomes a token. | The derived value has no name, so no reviewer can find every use of it. |

OKLCH parses from Safari 15.4, Chrome 111, and Firefox 113. The Tailwind v4
engine targets Safari 16.4 and Chrome 111, which is above that floor, so every
browser that runs v4 parses the notation. shadcn/ui emits OKLCH values by
default since its Tailwind v4 support landed.

### The config is CSS, and not JavaScript

Tailwind v4 has no `tailwind.config.js` by default. One `@import "tailwindcss"`
line replaces the three `@tailwind` directives, and the CSS file holds the
config.

| The directive | What it does |
| --- | --- |
| `@theme` | Defines a theme value, and generates the utilities for it. It copies the value at build time. |
| `@theme inline` | The same, and it keeps a reference to a custom property. Use it for every token that a theme class overrides. |
| `@custom-variant` | Defines a variant such as `dark`, from a selector. |
| `@variant` | Applies an existing variant inside a CSS rule. |
| `@utility` | Defines one named utility, with the declarations that it carries. |
| `@source` | Names a path that the class scanner must read, beyond the automatic set. |
| `@plugin` | Loads a plugin that is written for the JavaScript API. |
| `@config` | Loads a legacy `tailwind.config.js`. It is a bridge for a migration, and not a destination. |

A project on `@config` is current but in decline. Move the values into `@theme`
and delete the JavaScript file.

### The scales beyond color

A z-index number in feature code is a stacking fight that nobody can resolve.
Name the layers once, in the order that they stack, and consume the names.

```css
/* Correct: one named scale, and no raw number in a feature file. */
:root {
  --z-dropdown: 30;
  --z-sticky: 40;
  --z-overlay: 50;
  --z-modal: 60;
  --z-toast: 70;
}
@utility z-dropdown {
  z-index: var(--z-dropdown);
}
@utility z-modal {
  z-index: var(--z-modal);
}
```

Spacing, radius, and shadow follow the same rule as color. Each one is a token
in the theme, and no feature file states a number for it. The motion tokens are
the durations, the easings, and the distances that an animation reads. This
file owns those three values as tokens. Domain 14 `motion-and-interaction` owns
the animation that consumes them, and it is not integrated yet.

### The dark theme is designed

Tailwind v4 removed the `darkMode: 'class'` config key. A project that toggles
a class MUST declare the variant in CSS, or every `dark:` utility is dead.

```css
/* Correct: globals.css, in the order that the file needs. */
@import "tailwindcss";
@import "tw-animate-css";
@custom-variant dark (&:is(.dark *));
```

The upgrade tool sometimes writes this line in a form that matches nothing. The
symptom is a toggle that changes the class on `<html>` and changes no pixel.
Read the generated line, and confirm that a `dark:` utility takes effect.

NEVER build a dark theme with `filter: invert()`. It destroys the brand color,
it inverts every photograph, and it produces contrast pairs that nobody chose.

Express elevation in the dark theme with lightness and with a border. The
shadow that separates two surfaces in the light theme separates nothing against
a dark background. A dark token set that is the arithmetic inverse of the light
set is the same failure with more steps.

Declare `color-scheme` beside the theme class. The browser then paints the
scrollbar, the caret, and the default form controls to match the theme.

```css
:root {
  color-scheme: light;
}
.dark {
  color-scheme: dark;
}
```

Domain 10 `accessibility-wcag` owns the contrast ratio of every pair that this
file defines, and it holds a veto. It is not integrated yet. Verify each new
pair against that domain when it lands.

### The first paint carries the theme

The theme class must be on `<html>` before the browser paints. Anything later
is a flash of the wrong theme, and the user reads it as a fault.

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| The product needs no toggle | Write `dark:` utilities, and set no class. Tailwind v4 follows `prefers-color-scheme` by default. | The product adds a toggle. | The user cannot choose a theme that differs from the one that the operating system sets. |
| The server must render the chosen theme in the markup | Hold the preference in a cookie, read the cookie on the server, and render the class on `<html>`. | The route must stay static, which the next row covers. | The cookie travels on every request, and the route that reads it becomes dynamic. |
| The route must stay static, and only the class must be correct | An inline script sets the class before the first paint, and `<html>` carries `suppressHydrationWarning`. | The server must render a value that depends on the theme. | One blocking script in the document, and an attribute mismatch that React must ignore. |

`references/client-and-url-state.md` owns the choice of the store behind the
preference, and it states the cookie row. This file owns what must be true on
the document before the paint.

`references/server-and-client-components.md` states that
`suppressHydrationWarning` works one level deep, and that it belongs on a
single element rather than on a subtree. The `<html>` element is that single
element here. The attribute on `<html>` differs between the two renders, and
nothing inside the subtree differs. The one-level behavior is therefore the
behavior that this case needs.

```tsx
// Wrong: the provider sets the class after the hydration.
// Failure: the server sends the light theme, the browser paints it, and the
// effect then adds the dark class. A user who chose dark sees a white page for
// one frame on every navigation that reloads the document.
"use client"; // reason: the provider reads storage and writes a class
import { useEffect, useState, type PropsWithChildren } from "react";

export function ThemeProvider({ children }: PropsWithChildren) {
  const [theme, setTheme] = useState("light");
  useEffect(() => setTheme(localStorage.getItem("theme") ?? "light"), []);
  useEffect(() => document.documentElement.classList.toggle("dark", theme === "dark"), [theme]);
  return <>{children}</>;
}
```

```tsx
// Correct: src/app/layout.tsx reads the cookie, and the server renders the
// class. No script runs before the paint, and no frame shows the wrong theme.
import { cookies } from "next/headers";

export default async function RootLayout({ children }: LayoutProps<"/">) {
  const theme = (await cookies()).get("theme")?.value === "dark" ? "dark" : "";
  return (
    <html lang="en" className={theme}>
      <body>{children}</body>
    </html>
  );
}
```

The cookie read makes the route dynamic.
`references/caching-and-revalidation.md` owns that consequence, and
`references/app-router-structure.md` owns the awaited `cookies()` call.

### The theme provider on this stack

`next-themes` is the default recommendation, and its status is a live concern.
Prefer the options in this order.

1. No toggle, and no JavaScript. Write `dark:` utilities, and let
   `prefers-color-scheme` decide.
2. The cookie, read on the server, with the class rendered in the layout above.
3. `next-themes`, where the product needs its storage, its system-preference
   listener, and its multi-theme support.
4. A single-writer manager over `useSyncExternalStore`, where the two open
   defects below actually appear in the product.

Two defects are open against `next-themes` on this stack. The React 19 development overlay
reports a script tag inside a client component, which is a false positive at
server render time. A hidden provider under Cache Components or `<Activity>`
can write a stale theme back after a navigation.
`references/state-and-effects.md` owns `useSyncExternalStore`.

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| A white flash on load, with dark saved | The class reaches `<html>` after the hydration | Reload a route with the dark theme saved | Render the class on the server, or set it from a blocking script |
| A custom utility such as `bg-warning` produces no style | The token is in `:root` and no `@theme inline` publishes it | The class is in the DOM, and no CSS rule matches it | Add `--color-warning: var(--warning)` to `@theme inline` |
| The toggle changes the class and changes no pixel | The `@custom-variant dark` line is absent, or the upgrade tool wrote it wrong | Toggle the theme, and read the computed style | Declare `@custom-variant dark (&:is(.dark *));` |
| Every border changed color after the v4 upgrade | The default border color is now `currentColor` | A visual difference on any bare `border` class | State the border color token on the element |
| Every shadow looks heavier after the v4 upgrade | The shadow scale renamed one step down | A visual difference across the whole surface | Remap the scale, and read `shadow-xs` for the old `shadow-sm` |
| The focus ring is thinner and gray after the v4 upgrade | The ring default is 1px and `currentColor` | Tab through any control | Set `--color-ring`, and use `ring-3` for the old width |
| A dark hover state stopped working after the v4 upgrade | Stacked variants now apply from left to right | Compare `hover:dark:` against `dark:hover:` | Reorder the variants |
| The theme returns to a stale value after a navigation | A hidden provider writes the previous theme back | Toggle the theme, navigate away, and return | Key the provider, or move to a single-writer manager |

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those four facts on 16 August 2026. A cell that
holds no date is a package with a current registry entry on that date. This
file does not state an exact release date for it.

`references/component-composition.md` holds the primitive libraries, which are
shadcn/ui, Base UI, and Radix Primitives. This table holds the style layer
only.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `tailwindcss` v4.3 | The styling engine, with the config in CSS. | 4.3 | About May 2026 | Active | None |
| Recommend | `@tailwindcss/postcss` v4 | The PostCSS entry point. Next.js takes this one, and not the Vite plugin. | 4.x | Current | Active | None |
| Recommend | `tw-animate-css` 1.4.0 | The animation utilities on v4. A 2.0.0 line with breaking changes is announced. | 1.4.0 | Current | Active | None |
| Conditional | `@tailwindcss/typography` | Rendered prose from Markdown. shadcn/ui also ships a typeset registry item for this. | Current | Current | Active | None |
| Conditional | `@wrksz/themes` | Only where the two open `next-themes` defects appear in the product. It has a smaller maintainer base. | Current | Current | Active | None |
| Audit-only | `next-themes` 0.4.6 | Keep it where it works. Read the two open defects above before you add it to a Next 16 and React 19 project. | 0.4.6 | 11 March 2025 | Two maintainers, and no release for over a year | None |
| Audit-only | `tailwindcss-animate` | Superseded on v4. Replace it with `tw-animate-css`. Alive only in legacy code. | Current | Current | Superseded | None |
| Reject | `filter: invert()` as a dark theme | It destroys the color intent, the photographs, and the contrast. | — | — | — | — |

### Version discipline

Read the installed versions before you write code. Tailwind v4 changed enough
that a v3 idiom compiles to nothing rather than to an error.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| `@tailwind` directives became one `@import "tailwindcss"` | `@tailwind ` in any CSS file | Replace the three lines with the one import |
| The JavaScript config is gone by default | A `tailwind.config.ts` file | Move the values into `@theme`; `@config` bridges the gap |
| `darkMode: 'class'` was removed | A toggle that changes no pixel | Declare `@custom-variant dark (&:is(.dark *));` |
| `bg-gradient-*` became `bg-linear-*` | `bg-gradient-to-` in any file | Rename; the upgrade tool handles most of them |
| The default border color became `currentColor` | A bare `border` class | State the border color token |
| The ring default became 1px and `currentColor` | A thin gray focus ring | Set `--color-ring`, and use `ring-3` for the old width |
| `shadow-sm` became `shadow-xs`, and `shadow` became `shadow-sm` | Shadows that look heavier | Remap the scale across the theme |
| `bg-opacity-*` became the slash syntax | `bg-opacity-` in any file | Write `bg-black/50` |
| Stacked variants apply from left to right | `hover:dark:` in any file | Reorder, and read every dark hover state |
| Container queries moved into the core | `@tailwindcss/container-queries` in `package.json` | Remove the plugin, and keep the `@container` syntax |
| shadcn/ui moved from HSL to OKLCH, and it adds `data-slot` | HSL variables in a v4 project | Add the components again through the CLI |
| Tailwind v4.3 added scrollbar utilities and `container-size` | — | Adopt where useful |

## Verification

```bash
# 1. Read the installed versions before you write code.
node -p "require('tailwindcss/package.json').version"
rg -n '@import "tailwindcss"' src/styles/globals.css

# 2. Find a raw hex color outside the theme file. This must print nothing.
rg -n --glob '!**/globals.css' '#[0-9a-fA-F]{3,8}\b' src/

# 3. Find a px font size and an arbitrary spacing value. Read every hit.
rg -n 'text-\[[0-9]+px\]|p-\[[0-9]+px\]|m-\[[0-9]+px\]' src/

# 4. Find a raw z-index. This must print nothing.
rg -n 'z-\[[0-9]+\]|z-index:\s*[0-9]' src/

# 5. Find a token that no @theme inline block publishes. Read every hit.
rg -n '^\s*--[a-z-]+:' src/styles/globals.css

# 6. Confirm the dark variant, where the project toggles a class.
rg -n '@custom-variant dark' src/styles/globals.css

# 7. Find an inverted dark theme. This must print nothing.
rg -n 'filter:\s*invert|\binvert\b' src/

# 8. Find the stale v3 idioms. Each one must print nothing.
rg -n '@tailwind ' src/
rg -n 'bg-gradient-to-' src/
rg -n 'bg-opacity-|text-opacity-' src/
rg -n 'tailwindcss-animate' package.json src/

# 9. Confirm color-scheme, so the browser paints the native parts.
rg -n 'color-scheme' src/styles/globals.css

# 10. Confirm the first paint. Choose the dark theme, reload the route, and
#     read the first frame. No white frame appears.

# 11. Confirm the dark theme on every new surface. Toggle it, and read the
#     surface. No element loses its contrast, and no element disappears.
```

## Review checklist

- [ ] Is every color, space, radius, and shadow value in the theme rather than
      in a feature file?
- [ ] Does every color token appear in `:root`, in `.dark`, and in
      `@theme inline`?
- [ ] Is every color value written as `oklch()`, with no stray hex or HSL in a
      v4 theme?
- [ ] Does the project hold one named z-index scale, with no raw number in a
      feature file?
- [ ] Is `@custom-variant dark` declared, where the project toggles a class?
- [ ] Is the dark theme a designed set, with elevation from lightness and from
      a border?
- [ ] Is `filter: invert()` absent from the repository?
- [ ] Does `color-scheme` follow the theme, so the browser paints the native
      parts correctly?
- [ ] Is the theme class on `<html>` before the first paint?
- [ ] Does `suppressHydrationWarning` sit on `<html>` alone, where an inline
      script sets the class?
- [ ] Are the v3 idioms absent — `@tailwind`, `bg-gradient-*`, `bg-opacity-*`,
      and `tailwindcss-animate`?
- [ ] Does the project state a reason for each `@config` bridge that remains?

## Handoffs

- `cn()`, the variant API, the primitive contract, and the focus ring token →
  `references/component-styles-and-variants.md`.
- The container query, the logical property, the type scale, and the font →
  `references/layout-and-typography.md`.
- The store behind a stored preference, and the cookie against `persist` →
  `references/client-and-url-state.md`.
- `suppressHydrationWarning`, and the rest of the hydration failures →
  `references/server-and-client-components.md`.
- The awaited `cookies()` call, and the route that a cookie read makes dynamic
  → `references/app-router-structure.md` and
  `references/caching-and-revalidation.md`.
- `useSyncExternalStore`, and the effect rules around a subscription →
  `references/state-and-effects.md`.
- The `styles/` folder, and the file that holds the theme →
  `references/directory-and-module-boundaries.md`.
- `prettier-plugin-tailwindcss`, `tailwindStylesheet`, and the class order →
  `references/lint-format-and-scripts.md`.
- The contrast ratio of every token pair, and the forced-colors mode → domain
  10 `accessibility-wcag`. Not integrated yet. That domain holds a veto.
- The animation that consumes the motion tokens → domain 14
  `motion-and-interaction`. Not integrated yet.
- The chart series tokens in use, and the color of a data series → domain 12
  `data-tables-and-visualization`. Not integrated yet.
- The CSS bytes that a theme costs, and the budget over them → domain 16
  `performance-and-web-vitals`. Not integrated yet.
