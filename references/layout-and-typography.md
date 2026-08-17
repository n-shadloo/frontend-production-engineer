# Layout and typography

Tailwind CSS v4.3, Next.js 16.3, React 19.2.6 or later. This file owns the
space that an element occupies, and the text inside it. The subjects are the
container query, the logical property, and the unit that survives a mobile
browser. They also include the box that a loading value reserves, the type
scale, the font, and the direction that a surface takes as a whole.

The tokens behind every value are `references/design-tokens-and-theming.md`.
The classes on a part and the variant API are
`references/component-styles-and-variants.md`. The boundary that renders while
a value is absent is `references/suspense-and-actions.md`.

## Principle

A component does not know the width of the window. It knows the width of the
space that its parent gives it, and that is the measurement it must answer to.

A layout that names a physical side assumes one writing direction. A layout
that names a logical side works in both.

The visible height of a mobile browser changes while the user scrolls. A unit
that ignores that change clips the content or moves it.

Space that arrives with the content moves everything below it. Reserve the box
before the bytes arrive, and nothing moves.

Text has a comfortable measure, and it is not the width of the screen. A line
that is too long is hard to read, however correct the font is.

A surface with one clear idea is memorable. A surface with five competing ideas
is noise, and the reader remembers none of them.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The container query answers for a component

```tsx
// Wrong: a reusable card answers to the viewport.
// Failure: the viewport is 1440px, so md: applies. The card sits in a 280px
// sidebar, and it renders the two-column layout in that width. The columns are
// 130px each, and every label wraps to four lines.
<article className="grid grid-cols-1 md:grid-cols-2 gap-4">…</article>
```

```tsx
// Correct: the parent declares a container, and the card answers to it.
<div className="@container">
  <article className="grid grid-cols-1 @md:grid-cols-2 gap-4">…</article>
</div>
```

The rule is categorical, and no pixel threshold decides it. A reusable
component answers to its container. A page-level layout answers to the
viewport, because the page is the container.

Container queries are in the Tailwind v4 core. A project that still installs
`@tailwindcss/container-queries` carries a plugin that does nothing. Remove it,
and keep the `@container` syntax.

### The logical property costs nothing, and it saves the RTL work

```tsx
// Wrong: the physical side is in the class.
// Failure: the surface renders under dir="rtl", and the icon stays on the
// left of the text. The whole row reads backwards, and every screen needs a
// second pass before the product ships in Persian or in Arabic.
<span className="ml-2 border-l pl-3 text-left">{label}</span>
```

```tsx
// Correct: the logical side follows the writing direction.
<span className="ms-2 border-s ps-3 text-start">{label}</span>
```

Author every new component with `ms-`, `me-`, `ps-`, `pe-`, `start`, and `end`.
A physical property is still correct for a concern that is physical, such as
the offset of a drop shadow. State the reason in a comment where you write one.

Domain 19 `internationalization-and-rtl` owns the locale route, the `dir`
attribute, and the font subset for a non-Latin script. It is not integrated
yet. This file owns only the authoring rule that makes that work cheap.

### The unit that survives a mobile browser

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| A section must fill the visible height | `dvh`, which follows the toolbar as it hides and appears. | The content must not move while the toolbar moves, which the next row covers. | The height changes during a scroll, so a centred element moves with it. |
| A section must fill a height that never changes | `svh`, which is the smallest visible height. | The design needs the extra space that the hidden toolbar frees. | Some space stays unused when the toolbar hides. |
| The layout must reach under the notch or the home indicator | `env(safe-area-inset-*)` in the padding, with `viewport-fit=cover`. | The surface has no full-bleed region. | One more value in the padding, and a device to test it on. |
| A page grows a scrollbar on some routes and not on others | `scrollbar-gutter: stable`. | The design has no centred content. | A reserved strip on every route, including the routes that do not scroll. |

NEVER use `vh` for a full-height section. It ignores the dynamic toolbar of iOS
and Android. The browser then clips the section, or the section jumps while the
user scrolls.

### Reserve the box before the bytes arrive

```tsx
// Wrong: the image has no reserved box.
// Failure: the browser lays out the page with a height of zero for the image.
// The bytes arrive, the image takes its natural height, and every element
// below it moves down. The user is reading, and the text moves under the
// cursor.
<img src={src} className="w-full" />
```

```tsx
// Correct: the ratio reserves the box before the request finishes.
<div className="aspect-video w-full overflow-hidden rounded-lg">
  <img src={src} className="size-full object-cover" />
</div>
```

A skeleton follows the same rule. Give it the geometry of the content that
replaces it. `references/suspense-and-actions.md` owns the boundary and the
rule that a fallback holds the same box. This file owns the CSS that holds it.

`references/image-and-video-delivery.md` owns the request for the image itself,
and domain 16 `performance-and-web-vitals` owns the layout shift budget. That
domain is not integrated yet.

### The type scale

Define the scale once, in the theme, and consume the utilities. A fluid step
uses `clamp()`, which takes a minimum, a preferred value, and a maximum.

```css
/* Correct: one fluid step in the theme. The minimum is the value at the
   narrow end, and the maximum is the value at the wide end. */
@theme {
  --text-display: clamp(2rem, 1.2rem + 3vw, 3.5rem);
}
```

Set the minimum of a fluid step at a size that stays readable at 200 percent
zoom. A `clamp()` whose minimum is too small removes the benefit of the zoom,
because the viewport width falls as the zoom rises.

| The property | The rule |
| --- | --- |
| `text-wrap: balance` | A heading of a few lines. Chromium applies it up to six lines, Firefox up to ten, and Safari sets no line limit. |
| `text-wrap: pretty` | A paragraph, to remove the single short last line. Confirm the support target before you gate a design on it. |
| The measure | Hold a paragraph near 45 to 75 characters with a `max-w-*` token, and never at the full width of a wide screen. |

NEVER set a fixed `px` height on a container that holds text. The text grows
with the zoom, with a longer translation, and with a different script, and the
container then clips it. Set a minimum height and a padding, and let the
content decide the rest.

### The font

`next/font` self-hosts the font files at build time. The browser makes no
request to a third-party host, and the module generates the fallback metrics
that hold the layout while the font loads.

```ts
// Correct: src/app/fonts.ts declares one variable font, and the layout
// consumes the CSS variable that it generates.
import { Inter } from "next/font/google";

export const sans = Inter({
  subsets: ["latin"],
  display: "swap",
  variable: "--font-sans",
});
```

```tsx
// The class goes on <html>, and the theme reads the variable.
<html lang="en" className={sans.variable}>
```

| The option | The default | The rule |
| --- | --- | --- |
| `subsets` | None | State the subsets that the product renders. Every extra subset is bytes that no reader needs. |
| `display` | `swap` | Keep `swap`. The text is readable while the font loads. |
| `preload` | `true` | Keep it for the font of the first paint. Turn it off for a font that only a later route uses. |
| `adjustFontFallback` | `true` for a Google font, and `'Arial'` for a local font | Keep it. It generates the metric overrides that stop the shift when the font arrives. |
| `variable` | None | State it, so the font reaches the theme as a custom property. |
| `axes` | None | State the axes of a variable font that the design uses. |
| `declarations` | None | Add a metric override by hand, where a static family needs one for each weight. |

Prefer one variable font over several static weights. The module computes the
fallback metrics from the first weight alone. A family of four static weights
can therefore shift the layout when a heading in a second weight arrives.

Domain 16 `performance-and-web-vitals` owns the byte budget and the layout
shift budget over this setup. Domain 19 `internationalization-and-rtl` owns the
subset for a non-Latin script. Neither is integrated yet.

### The surface as a whole

These are review heuristics, and no specification sets them. They are decidable
against the design brief for the surface, and not against a lint rule.

Give a screen one signature idea. Cut every decoration that does not serve the
content. A screen that carries five competing ideas has none.

Build the surface, take a screenshot, and read the screenshot against the
brief. A defect of proportion, of rhythm, or of contrast is visible in an image
and invisible in the markup.

| The pattern | Why it appears | The cost | The replacement |
| --- | --- | --- | --- |
| A design that could belong to any product | The default output of a generator, with no direction from the brief | The product is forgettable, and it looks like every other product | Take the direction from the subject, and keep one signature element |
| A new dependency for one element | The habit of a search before a build | Bundle bytes, and one more supply chain to audit | Build it from the primitive that the project already holds |
| Decoration that competes with the content | Every element was designed, and none was cut | The reader cannot tell what matters | Remove one element, and read the screen again |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| A card is correct in a grid and broken in a sidebar | The component answers to the viewport | Put the component in a narrow parent | `@container` on the parent, and `@`-prefixed variants on the children |
| Every row reads backwards under `dir="rtl"` | Physical direction utilities | Set `dir="rtl"` on `<html>` | Author with the logical utilities |
| A full-height section is clipped, or it jumps while the user scrolls | `100vh` ignores the dynamic toolbar | Open the route on iOS Safari | `dvh` or `svh`, by the rule in the table above |
| The content moves down when an image arrives | No reserved box | Load the route on a slow connection | An `aspect-ratio` box around the image |
| The layout shift number rose after a release that added a weight | The fallback metrics come from the first weight | Compare the shift between two releases | One variable font, or a metric override for each weight |
| A heading breaks to one short last line | No `text-wrap` value | Read the heading at several widths | `text-wrap: balance` on the heading |
| A container clips its text at 200 percent zoom | A fixed `px` height | Zoom the browser to 200 percent | A minimum height and a padding |
| The centred content moves between two routes | One route has a scrollbar, and the other does not | Navigate between the two routes | `scrollbar-gutter: stable` |

### Version discipline

Read the installed versions before you write code.

Container queries are in the Tailwind v4 core. Tailwind v4 supersedes the v3
plugin, and that plugin is alive only in legacy code.

Tailwind v4.3 added first-party scrollbar utilities, a `container-size` value,
and more logical utilities. They are optional, and nothing in this file
requires them.

`next/font` has no change that is specific to Next.js 16. The options in the
table above are the current API under 16.3. The one rename in its history is
`@next/font` to `next/font`, and it happened in the version 13 line. The old
package name is alive only in legacy code.

## Verification

```bash
# 1. Read the installed versions before you write code.
node -p "require('next/package.json').version"
node -p "require('tailwindcss/package.json').version"

# 2. Find a reusable component whose only responsive behavior is a viewport
#    breakpoint. Read every hit.
rg -n 'sm:|md:|lg:' -g '*.tsx' src/components src/features | rg -v '@'

# 3. Find a physical direction utility in new component code. Read every hit.
rg -n '\b(ml|mr|pl|pr)-[0-9]|text-(left|right)|border-[lr]\b' -g '*.tsx' src/

# 4. Find vh on a full-height section. This must print nothing.
rg -n 'h-screen|100vh|h-\[[0-9]+vh\]' src/

# 5. Find an image with no reserved box. Read every hit.
rg -n '<img' -g '*.tsx' src/ | rg -v 'aspect-|size-full'

# 6. Find a fixed px height on a text container. Read every hit.
rg -n 'h-\[[0-9]+px\]' src/

# 7. Confirm that the container query plugin is gone from a v4 project.
rg -n '@tailwindcss/container-queries' package.json

# 8. Confirm one font declaration, and read its options.
rg -n 'next/font' src/

# 9. Confirm the layout at both ends. Set the viewport to 320px and then to
#    2560px. No element overflows, and no grid breaks.

# 10. Confirm the direction. Set dir="rtl" on <html>, and read the surface.
#     Every row mirrors correctly.

# 11. Confirm the zoom. Set the browser to 200 percent, and read the surface.
#     No container clips its text, and every control keeps a visible focus
#     indicator.

# 12. Take a screenshot of the surface, and read it against the brief before
#     you ship it.
```

## Review checklist

- [ ] Does every reusable component answer to a container, rather than to the
      viewport?
- [ ] Is a viewport breakpoint used at the page level alone?
- [ ] Does every new component use the logical direction utilities?
- [ ] Does each physical property that remains state its reason?
- [ ] Does every full-height section use `dvh` or `svh`, and never `vh`?
- [ ] Does every image and every embed carry a reserved box?
- [ ] Does every skeleton hold the geometry of the content that replaces it?
- [ ] Is the type scale in the theme, with no `px` size in a feature file?
- [ ] Does every fluid step keep a minimum that stays readable at 200 percent
      zoom?
- [ ] Is every text container free of a fixed `px` height?
- [ ] Does the project self-host its fonts with `next/font`, with `display` at
      `swap`?
- [ ] Is one variable font preferred over several static weights?
- [ ] Does the surface hold one signature idea, with the decoration that serves
      nothing removed?
- [ ] Was the surface read as a screenshot against the brief before it shipped?

## Handoffs

- The tokens, the theme, the dark variant, and the scales →
  `references/design-tokens-and-theming.md`.
- `cn()`, the variant API, the primitive, and the focus ring →
  `references/component-styles-and-variants.md`.
- The Suspense boundary, and the fallback that holds the same box →
  `references/suspense-and-actions.md`.
- The decomposition of the component that holds the layout →
  `references/component-composition.md`.
- The route file that renders the page shell →
  `references/app-router-structure.md`.
- The reflow criterion and the zoom criterion →
  `references/visual-and-motor-criteria.md`. The keyboard path and the
  authoritative focus indicator rule →
  `references/keyboard-focus-and-live-regions.md`. That domain holds a veto.
- The step of a multi-step form →
  `references/multi-step-forms-and-unsaved-work.md`. The error state that sits
  beside a field is `references/form-submission-and-server-errors.md`.
- The column widths, the sticky header, and the virtualiser of a long list →
  `references/data-table-and-server-driven-state.md`.
- `next/image`, the responsive source set, and the request for the bytes →
  `references/image-and-video-delivery.md`.
- The transition between two layouts, and the reduced variant behind it →
  `references/motion-primitives-and-reduced-motion.md`.
- The words in a heading, and the length that the design assumes → domain 15
  `ux-writing-and-content-design`. Not integrated yet.
- The layout shift budget, the font byte budget, and the LCP element → domain
  16 `performance-and-web-vitals`. Not integrated yet.
- The locale route, the `dir` attribute, and the subset for a non-Latin script
  → domain 19 `internationalization-and-rtl`. Not integrated yet.
