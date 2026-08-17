# Visual and motor criteria

Tailwind CSS v4.3, Next.js 16.3, React 19.2.6 or later, WCAG 2.2 Level AA. This
file owns the measurable properties of a rendered surface, and the input that
operates it. The subjects are the contrast ratio, the second channel beside
color, the size of a target, and the reflow at 400 percent zoom. They also
include the four user preferences that a browser reports, and the alternative
to every drag. The last subject is the consistency that a reader with a
cognitive disability depends on.

The element, the role, and the name are
`references/semantics-and-accessible-names.md`. The tab path, the focus, and
the announcement are `references/keyboard-focus-and-live-regions.md`. The
tooling that proves this file is
`references/wcag-conformance-and-verification.md`.

## Principle

Contrast is a measurement. An opinion about a color is not a measurement, and
a screenshot on a bright monitor proves nothing.

Color is a second channel. It is never the only one, because a part of the
users receives none of it.

A target that a finger misses is a control that does not exist. The size of the
hit area is part of the control, and not part of the decoration.

A page must work at 400 percent zoom, which is a viewport of 320 CSS pixels.
That is the same width as a small phone, and the same layout answers both.

A stated user preference is an instruction. The browser reports it, and the
application obeys it.

A drag needs two working hands and fine control. Every drag therefore needs a
second path that one pointer or one key can take.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The ratios

| The content | The minimum ratio |
| --- | --- |
| Body text | 4.5:1 against its background |
| Large text | 3:1 against its background |
| A user interface component, such as a border that shows the bounds of a field | 3:1 against the colors next to it |
| A graphical object that carries information, such as an icon in a status | 3:1 against the colors next to it |
| A focus indicator | 3:1 against the colors next to it |

Measure the ratio. Never accept it by eye.
`references/design-tokens-and-theming.md` owns the token pairs that produce
these colors, and this file owns the ratio that each pair must meet.

Four cases fail after the light theme passes.

1. The dark theme, where a pair that inverts is a different pair.
2. A disabled state, where the reduced opacity produces a new color.
3. Text over a gradient or over a photograph, where the background is a range
   and not one value.
4. A color that `color-mix()` produces at run time, where no static pair
   exists to check.

```tsx
// Wrong: the muted text takes a lighter token to look calm.
// Failure: the pair falls under 4.5:1. The text is unreadable in daylight, on
// a low-quality panel, and for a reader with reduced contrast sensitivity.
<p className="text-muted-foreground/60">Order placed 3 days ago</p>
```

```tsx
// Correct: the token pair is one that the project measured.
<p className="text-muted-foreground">Order placed 3 days ago</p>
```

An opacity modifier on a text color creates a new pair that nobody measured.
Define a token for the color that the design needs, and measure that token.

Run the contrast check in a browser. A component test in a simulated DOM
computes no painted color, so it can report nothing about a ratio.
`references/wcag-conformance-and-verification.md` owns the two test lanes.

### Color is never the only channel

```tsx
// Wrong: the status is a colored dot.
// Failure: a reader who does not receive the color difference sees three
// identical dots. A screen reader reports nothing at all.
<span className="size-2 rounded-full bg-destructive" />
```

```tsx
// Correct: the status carries a shape, a word, and a color.
import { CircleAlert } from "lucide-react";

<span className="inline-flex items-center gap-1 text-destructive">
  <CircleAlert aria-hidden="true" className="size-4" />
  Failed
</span>
```

Four places break this rule most often. A status indicator, a validation state
on a field, a series in a chart, and a link inside a paragraph of text. A link
in body text needs an underline, or another difference that is not the color.

`references/charts-and-visual-encoding.md` owns the second channel of a chart
series.

### The size of a target

A pointer target is at least 24 by 24 CSS pixels. A primary touch action is 44
by 44 CSS pixels. Spacing around a smaller target can satisfy the criterion,
and the size is the simpler answer.

```tsx
// Wrong: the icon is the whole control.
// Failure: the hit area is 16 CSS pixels square. It falls under criterion
// 2.5.8, and a finger or a tremor misses it.
<button type="button" onClick={close}>
  <X aria-hidden="true" className="size-4" />
</button>
```

```tsx
// Correct: the control carries the target, and the icon stays small.
<button
  type="button"
  onClick={close}
  className="inline-flex size-11 items-center justify-center"
>
  <X aria-hidden="true" className="size-4" />
  <span className="sr-only">Close</span>
</button>
```

A row of small controls costs more than one large control. Count the targets in
a table row, in a toolbar, and in a list item.

### The reflow and the zoom

The page must work at 320 CSS pixels of width with no scroll in two directions.
A user at 400 percent zoom on a 1280-pixel display sees exactly that viewport.

A horizontal scroll is acceptable only for content that needs its two
dimensions, such as a wide data table or a map.

The page must also work when the user overrides the text spacing. These four
values must clip no content.

- A line height of 1.5 times the font size.
- A space between paragraphs of 2 times the font size.
- A letter spacing of 0.12 times the font size.
- A word spacing of 0.16 times the font size.

```tsx
// Wrong: a fixed height holds a line of text.
// Failure: the text overflows the box at the user's spacing, and at 200
// percent zoom. The end of the sentence is invisible, and no scrollbar
// appears.
<div className="h-10 overflow-hidden">{title}</div>
```

```tsx
// Correct: the box grows with its content.
<div className="min-h-10 py-2">{title}</div>
```

`references/layout-and-typography.md` owns the container query, the fluid type
scale, and the fixed height that this rule forbids. This file owns the
criterion behind them.

Never block the zoom.

```ts
// Wrong: src/app/layout.tsx blocks the pinch zoom.
// Failure: a user who needs magnification cannot get it on a touch screen.
import type { Viewport } from "next";

export const viewport: Viewport = { width: "device-width", initialScale: 1, userScalable: false };
```

```ts
// Correct: the zoom stays available.
import type { Viewport } from "next";

export const viewport: Viewport = { width: "device-width", initialScale: 1, userScalable: true };
```

A `maximumScale` value of 1 blocks the zoom in the same way. Leave the key
absent.

The page must also work in both orientations. Never lock a surface to one
orientation, unless the orientation is essential to the task.

### The four user preferences

| The preference | What the browser reports | What the application must do |
| --- | --- | --- |
| `prefers-reduced-motion: reduce` | The user asked the system for less motion | Remove the transform animation, the parallax, and the autoplay. Keep an opacity change, or no change. |
| `prefers-reduced-transparency: reduce` | The user asked for fewer translucent surfaces | Replace the translucent panel with an opaque one. |
| `prefers-contrast: more` | The user asked for stronger contrast | Raise the contrast of the borders and of the muted text. |
| `forced-colors: active` | The operating system replaced the palette | Let the system colors win. Keep the borders, because a background color no longer separates the surfaces. |

```css
/* Correct: globals.css obeys the motion preference for the whole product. */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

This block is the one place in the project that carries `!important`.
`references/component-styles-and-variants.md` forbids it in a feature file, and
the reason is that a feature file must never defeat a primitive. This block
defeats an animation on purpose, and it defeats it everywhere.

Tailwind v4 supplies the `motion-reduce:`, `contrast-more:`, and
`forced-colors:` variants for a single element. Take the variant where one
component needs a different answer from the global rule.

Forced colors mode removes every background image and every background color
that the project set. A surface that separates itself by background alone
disappears. Give each card, each panel, and each menu a border.

`references/motion-primitives-and-reduced-motion.md` owns the animation itself,
and `references/design-tokens-and-theming.md` owns the duration tokens behind
it. This file owns the preference that both must obey, and the global block
above. That block is the floor, and the reduced variant is the design over it.

### Every drag has a second path

```tsx
// Wrong: the reorder happens by drag alone.
// Failure: a user with a tremor, a user on a switch device, and a keyboard
// user cannot reorder the list at all. Criterion 2.5.7 fails.
<li draggable onDragStart={…} onDrop={…}>{item.name}</li>
```

```tsx
// Correct: the drag stays, and two buttons do the same work.
<li draggable onDragStart={…} onDrop={…}>
  {item.name}
  <button type="button" onClick={() => move(item.id, -1)}>
    <span className="sr-only">Move {item.name} up</span>
    <ChevronUp aria-hidden="true" />
  </button>
  <button type="button" onClick={() => move(item.id, 1)}>
    <span className="sr-only">Move {item.name} down</span>
    <ChevronDown aria-hidden="true" />
  </button>
</li>
```

The same rule covers three more inputs. A gesture with more than one finger, or
a gesture that follows a path, needs a single-pointer alternative. A feature
that a device movement triggers needs a control that does the same thing, and a
switch that turns the movement off. A slider that only a drag operates needs
arrow keys.

### The consistency that a reader depends on

Put the navigation in the same place on every page, and in the same order. A
reader who learned one page has learned all of them.

Put the help mechanism in the same relative place on every page that carries
it. Criterion 3.2.6 covers the contact link, the help link, and the chat
launcher.

Never change the context when the user did not ask for it. Focus on a field
never submits a form, and a change of a value never navigates. Give the user a
control that performs the action.

Give the user a way back. An action that destroys data needs a confirmation, an
undo, or both.

Domain 15 `ux-writing-and-content-design` owns the plain language of the
content itself. It is not integrated yet.

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| Text that a reader cannot read in daylight | A pair under 4.5:1, often a muted token with an opacity modifier | The axe contrast rule in the browser lane | Define and measure a token for the color |
| A pair that passes in light and fails in dark | Only the light theme was measured | Run the check with the dark class on `<html>` | Measure both themes |
| Three identical dots in a status column | Color is the only channel | Render the page in grayscale | Add an icon and a word |
| A control that a finger misses | The icon is the whole control | Measure the hit area in the inspector | Give the control a 24 or a 44 pixel box |
| Two-directional scroll at 400 percent zoom | A fixed width or a fixed height in the layout | Set the viewport to 320 pixels wide | Remove the fixed dimension |
| Text that clips at the user's spacing | A fixed height on a text container | Apply the text-spacing overrides | Replace the height with a minimum height |
| A pinch zoom that does nothing on a phone | `userScalable: false`, or a `maximumScale` of 1 | Read the `viewport` export | Remove the key |
| An animation that plays for a user who asked for none | No global reduced-motion block | Set the preference, and reload | Add the global block |
| Panels that merge into one surface | The surfaces separate by background alone | Turn on the forced-colors mode | Give each surface a border |
| A list that only a drag reorders | No second path | Reorder the list with the keyboard alone | Add the move controls |

### Version discipline

Read the installed versions before you write code.

Tailwind CSS v4 supplies `motion-reduce:`, `contrast-more:`,
`contrast-less:`, and `forced-colors:` as variants. A project on v3 wrote some
of them differently, and
`references/design-tokens-and-theming.md` owns the list of renames.

The `viewport` export of Next.js 16 replaces the viewport keys of the old
`metadata` export. A project that still writes a viewport `<meta>` tag by hand
has two sources for one value.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

## Verification

```bash
# 1. Find an opacity modifier on a text color. Read every hit.
rg -n 'text-[a-z-]+/[0-9]{1,2}\b' -g '*.tsx' src/

# 2. Find a fixed height on a text container. Read every hit.
rg -n 'className="[^"]*\bh-\[?[0-9]' -g '*.tsx' src/

# 3. Confirm that the zoom is not blocked. This must print nothing.
rg -n 'userScalable:\s*false|maximumScale:\s*1|user-scalable=no' src/

# 4. Confirm the global reduced-motion block.
rg -n 'prefers-reduced-motion' src/

# 5. Find an icon button with no explicit box. Read every hit.
rg -n '<button[^>]*>\s*<[A-Z]' -g '*.tsx' src/

# 6. Find a drag handler with no second path. Read every hit.
rg -n 'draggable|onDragStart|onPointerMove' -g '*.tsx' src/

# 7. Measure the contrast of every rendered surface in a browser, in the
#    light theme and in the dark theme.

# 8. Set the viewport to 320 pixels wide, or the zoom to 400 percent. No
#    surface scrolls in two directions, and no content is lost.

# 9. Apply the text-spacing overrides, and read every surface. No container
#    clips its text.

# 10. Turn on the forced-colors mode of the operating system. Every surface
#     keeps its bounds, and every control stays visible.

# 11. Set the reduced-motion preference, and complete one flow. No transform
#     animation plays.

# 12. Render one surface in grayscale. Every state still reads.
```

## Review checklist

- [ ] Does every text pair meet 4.5:1, or 3:1 for large text?
- [ ] Does every border, icon, and focus indicator that carries meaning meet
      3:1?
- [ ] Were the ratios measured in the dark theme as well as the light theme?
- [ ] Is every opacity modifier on a text color replaced by a measured token?
- [ ] Does every status, validation state, and chart series carry a second
      channel beside color?
- [ ] Is every pointer target at least 24 by 24 CSS pixels, and every primary
      touch action 44 by 44?
- [ ] Does the surface work at 320 CSS pixels with no two-directional scroll?
- [ ] Does the surface survive the text-spacing overrides with no clipped
      content?
- [ ] Is the zoom unblocked in the `viewport` export?
- [ ] Does the project carry one global `prefers-reduced-motion` block?
- [ ] Does every surface keep its bounds under `forced-colors: active`?
- [ ] Does every drag, path gesture, and multi-finger gesture carry a
      single-pointer alternative?
- [ ] Does every device-movement feature carry a control and a switch?
- [ ] Do the navigation and the help mechanism sit in the same place on
      every page?
- [ ] Is every destructive action reversible, or confirmed?

## Handoffs

- The role, the accessible name, and the alternative text →
  `references/semantics-and-accessible-names.md`.
- The focus indicator, the tab path, and the announcement →
  `references/keyboard-focus-and-live-regions.md`.
- The axe rules, the two test lanes, and the manual pass →
  `references/wcag-conformance-and-verification.md`.
- The token pairs, the dark theme, and `color-mix()` →
  `references/design-tokens-and-theming.md`.
- The container query, the fluid type scale, and the reserved box →
  `references/layout-and-typography.md`.
- `cn()`, the variant API, and the rule against `!important` in a feature file
  → `references/component-styles-and-variants.md`.
- The `viewport` export and the route files →
  `references/app-router-structure.md`.
- The second channel of a chart series → `references/charts-and-visual-encoding.md`.
- The wide data table, and its alternative on a phone →
  `references/data-table-and-server-driven-state.md`.
- The video player, its captions, and its controls →
  `references/image-and-video-delivery.md`.
- The animation itself, its purpose class, and the reduced variant over the
  tokens → `references/motion-primitives-and-reduced-motion.md`.
- The drag library, its sensors, and the scroll that a gesture drives →
  `references/gesture-and-scroll-interaction.md`.
- The plain language of the content → domain 15 `ux-writing-and-content-design`.
  Not integrated yet.
- The layout cost of a zoom, and the bytes of a large surface → domain 16
  `performance-and-web-vitals`. Not integrated yet.
