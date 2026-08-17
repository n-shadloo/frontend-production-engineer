# Motion primitives and reduced motion

CSS transitions and CSS keyframes, Tailwind CSS v4.3, `tw-animate-css` 1.4.0,
Next.js 16.3, React 19.2.6 or later. This file owns the animation that the
application starts, and the preference that limits it. The subjects are the
purpose that each animation states, the properties that one frame can afford,
and the tokens behind the duration. They also include the entrance and the exit
of a platform element, the panel that reveals its height, and the `will-change`
hint. The last subjects are the reduced variant, the static signal beside every
movement, and the focus that follows an entrance.

The view transition, Motion, and the other libraries are
`references/view-transitions-and-animation-libraries.md`. The drag, the sheet,
and the scroll are `references/gesture-and-scroll-interaction.md`.

## Principle

An animation costs the reader time and attention. An animation that nobody can
explain costs both and returns nothing.

The browser paints a frame about every 16 milliseconds. A property that changes
the size or the position of a box costs two steps inside that budget. The
browser lays out the page again, and then it paints again. Two properties do
not cost those steps, because the compositor holds them.

An element that leaves the document takes its styles with it. A style cannot
animate a box that is no longer there, so an exit needs a mechanism that an
entrance does not.

A preference is a request from a person. Some readers get sick from movement,
and some read a moving surface as a broken product. The answer is a different
animation, and never a product with no feedback.

Movement is one channel. A reader who does not see the screen receives nothing
from it.

Focus is a position in the document. An animation moves the document under that
position, and the reader loses the place.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### Every animation states its purpose

Four classes cover the work that an animation can do.

| The class | What it does | An example |
| --- | --- | --- |
| Orientation | It tells the reader where a new surface came from | A panel that slides in from the edge it belongs to |
| Feedback | It confirms that the product received an input | A button that presses down under the pointer |
| Continuity | It carries one element between two states | A thumbnail that becomes a header picture |
| Attention | It moves the eye to a change the reader did not make | A row that lights up when a new value arrives |

Name the class in the pull request. An animation with no class is decoration,
and this skill deletes decoration.

### `transform` and `opacity` on the hot path

```css
/* Wrong: the transition animates three layout properties.
   Failure: the browser lays out the page and paints it again on every frame.
   The interaction drops frames on a mid-range phone, and INP goes up. */
.card {
  transition: width 300ms ease, height 300ms ease, top 300ms ease;
}
```

```css
/* Correct: the compositor holds both properties, so no frame needs a layout. */
.card {
  transition:
    transform var(--duration-base) var(--ease-out),
    opacity var(--duration-base) var(--ease-out);
}
```

Animate `transform` and `opacity` alone on any path that an input starts.
`width`, `height`, `top`, `left`, `margin`, `box-shadow`, and `filter` each need
a written reason in the diff.

### The duration and the easing function come from tokens

`references/design-tokens-and-theming.md` owns the `--duration-*`, `--ease-*`,
and `--distance-*` tokens, and the scale behind them. This file owns what reads
them. NEVER type a millisecond value or a `cubic-bezier()` curve into a feature
file.

The convention behind the scale is three bands. A micro-interaction takes about
100 to 200 milliseconds. A change inside the page takes about 200 to 300. A
surface that covers the screen takes about 300 to 500. An entrance takes an
ease-out curve, and an exit takes an ease-in curve.

These bands are conventions of Material Design and of the Apple Human Interface
Guidelines. They are not a standard, and the design system fixes the exact
values.

A transition above 300 milliseconds on a hover reads as a broken product.

### CSS first, and the point at which a library wins

| The effect | The mechanism | What forces the change |
| --- | --- | --- |
| A hover, a focus, a press, or one state change | CSS `transition` | Nothing, while the element stays in the document |
| A loop with a fixed timeline, such as a spinner | CSS `@keyframes` and `animation` | A need to interrupt the timeline, or to scrub it |
| The entrance and the exit of a dialog or a popover | CSS `@starting-style` and `transition-behavior: allow-discrete` | An exit that must be interruptible, or a spring curve |
| The exit of an element that React unmounts | Motion `AnimatePresence` | — |
| A panel that reveals its height | `grid-template-rows`, or `interpolate-size` | The browser support that the product states |
| Continuity between two route states | A view transition | Several morphs at once, or an interruption |
| A physics curve, an inertia, or a long timeline | Motion, or GSAP | — |

The last three rows are
`references/view-transitions-and-animation-libraries.md`.

Take a JavaScript library for four reasons only. The reasons are an
interruption, a physics curve, a measurement of the layout, and the exit of an
element that the framework unmounts. A library that arrives for a hover state
is a failure.

### The entrance and the exit of a platform element

```css
/* Wrong: the element disappears with no exit.
   Failure: display flips from block to none in one frame, so the opacity
   transition never runs. The dialog vanishes. */
dialog { opacity: 0; transition: opacity 200ms; }
dialog[open] { opacity: 1; }
```

```css
/* Correct: the start style is declared, and the discrete properties animate. */
dialog {
  opacity: 0;
  transition:
    opacity var(--duration-fast),
    overlay var(--duration-fast) allow-discrete,
    display var(--duration-fast) allow-discrete;
}
dialog[open] { opacity: 1; }
@starting-style {
  dialog[open] { opacity: 0; }
}
```

`@starting-style` gives the browser the style to start from.
`transition-behavior: allow-discrete` lets `display` and `overlay` change in
steps across the duration. Together they replace a JavaScript state machine for
a dialog, for a popover, and for any element with the `hidden` attribute.

CAUTION: a `transition: all` shorthand, or any `transition` shorthand that
follows the one that carries `allow-discrete`, resets the behavior to `normal`.
The exit then stops, and the rule that broke it sits several lines away. Declare
`allow-discrete` inside the last `transition` shorthand on the selector.

### The panel that reveals its height

```css
/* Wrong: height animates from 0 to auto.
   Failure: auto is not a length, so the browser has nothing to interpolate
   between. The panel snaps open with no transition at all. */
.panel { height: 0; transition: height var(--duration-base); }
.panel[data-open] { height: auto; }
```

```css
/* Correct: a grid row carries the reveal, and the content sits in a wrapper. */
.panel {
  display: grid;
  grid-template-rows: 0fr;
  transition: grid-template-rows var(--duration-base) var(--ease-out);
}
.panel[data-open] { grid-template-rows: 1fr; }
.panel > * { overflow: hidden; }
```

The grid row works in every browser that the product supports today, and it
costs one wrapper element. `interpolate-size: allow-keywords` on `:root` is the
newer answer, and it makes `height: 0` to `height: auto` animate directly. Read
the support that the product states before you take it.

Both mechanisms animate a layout property. They are the documented case that
the rule above sends to a written reason. The reveal runs once for each press,
and it is not a path that an input drives frame by frame.

### `will-change` is a hint with a cost

```css
/* Wrong: the hint sits on every card, for the life of the document.
   Failure: the browser promotes each card to its own layer, so the GPU memory
   grows with the list. No frame gets faster. */
.card { will-change: transform, opacity; }
```

```css
/* Correct: the hint arrives just before the animation, and it leaves after. */
.card:hover { will-change: transform; }
```

Set `will-change` on the element that is about to animate, and remove it when
the animation ends. NEVER set it on a whole list, and never in a base class.

### The reduced variant is a design

```css
/* Correct: the token override, in the file that declares the tokens. */
@media (prefers-reduced-motion: reduce) {
  :root {
    --duration-fast: 0.01ms;
    --duration-base: 0.01ms;
    --distance-slide: 0px;
  }
}
```

`references/visual-and-motor-criteria.md` owns the global block that covers
every animation the project did not think about. That block is the floor. This
override is the design above it, and it works because every duration in the
product reads a token.

The preference says "reduce", and it never says "none". Replace a movement with
an opacity change, or with the final state. A dialog still appears, a toast
still arrives, and a value still updates.

NEVER write `* { animation: none }` or `* { transition: none }`. An animation
with no duration fires no `transitionend` event and no `animationend` event. The
focus step below then never runs, and the reader loses the focus for good. The
global block sets a duration of 0.01 milliseconds for that reason, and the event
still fires.

CAUTION: a check that runs in JavaScript runs after the first paint. Gate the
movement in CSS, so the reader who asked for less motion sees none of it.

Where one component needs a different answer from the global rule, Tailwind v4
supplies the `motion-reduce:` variant for that one element.

### Motion is never the only signal

Every animated state change carries a static equivalent. The static equivalent
is a word, an icon, a color, or an ARIA state.

A row that only fades to report a save reports nothing to a reader on a screen
reader. It also reports nothing to a reader who set the preference.
`references/keyboard-focus-and-live-regions.md` owns the live region that
carries the message.

### The limits that a criterion sets

| The limit | What it forbids | What the interface adds |
| --- | --- | --- |
| Criterion 2.2.2 | Motion that starts by itself, runs past 5 seconds, and sits beside other content | A pause control, a stop control, or a hide control |
| Criterion 2.3.1 | Content that flashes more than three times in one second | A slower change, or no flash |
| Criterion 2.3.3 | A large movement that an interaction triggers, with no way out | A `prefers-reduced-motion` gate over that movement |

A carousel that advances by itself, a marquee, a background video, and an
animated banner each meet the first row.
`references/wcag-conformance-and-verification.md` and
`references/visual-and-motor-criteria.md` own the conformance verdict, and that
domain holds a veto. This file owns the implementation.

### Focus follows the entrance

```tsx
// Wrong: the focus moves in the same tick that opens the dialog.
// Failure: the dialog is still at its start position, so the browser scrolls
// to a box that then moves. A screen reader announces a node in mid-flight.
function open(node: HTMLDialogElement) {
  node.showModal();
  node.querySelector<HTMLElement>("[data-autofocus]")?.focus();
}
```

```tsx
// Correct: the entrance completes first, and the focus lands on a still box.
function open(node: HTMLDialogElement) {
  node.addEventListener(
    "transitionend",
    () => node.querySelector<HTMLElement>("[data-autofocus]")?.focus(),
    { once: true },
  );
  node.showModal();
}
```

`references/keyboard-focus-and-live-regions.md` owns which element takes the
focus, and the trap around a modal. This file owns the moment.

NEVER leave the focus on an element that a transition then moves off the screen.
Where a transition replaces the focused node, move the focus before the
transition starts.

### The response that needs no animation

A response under about 100 milliseconds needs no spinner. A spinner that appears
and disappears inside 100 milliseconds reads as a flicker, and a flicker reads
as a fault.

Delay the indicator by 300 to 500 milliseconds, or leave it out. A response past
about one second takes a skeleton whose shape matches the content.
`references/suspense-and-actions.md` owns the fallback and its shape.

### Measure before the change ships

Record the interaction in the Performance panel of the browser. The recording
shows no long task, and no layout bar and no paint bar on each frame. Turn on
paint flashing and the frame rate counter in the Rendering panel. Confirm that
the work stays on the compositor.

Repeat the recording at a 4× CPU throttle. This throttle is an engineering proxy
for a mid-range phone, and it is not a published threshold.
`references/performance-budgets-and-measurement.md` owns the device profile,
the field measurement, and the INP threshold. That domain holds no veto, so a
number above the threshold is a review finding.

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| The interaction drops frames | A layout property animates on every frame | Layout and paint bars on each frame in the Performance recording | Move to `transform` and `opacity` |
| The element disappears with no exit | `display` flips in one frame | The entrance runs, and the exit does not | Add `@starting-style` and `allow-discrete` |
| The exit stopped after an unrelated edit | A later `transition` shorthand reset the behavior | Read every `transition` line on the selector | Move `allow-discrete` into the last shorthand |
| A reader who set the preference still sees movement | The check runs in JavaScript, after the first paint | Set the preference, and reload | Gate the movement in CSS |
| The panel snaps open | `height` animates to `auto` | No transition on the reveal | Take `grid-template-rows`, or `interpolate-size` |
| The GPU memory grows with the list | `will-change` sits in a base class | Read the memory panel with the list open | Scope the hint to the animated element |
| The focus lands nowhere after a dialog opens | The focus moved before the entrance ended | Open the dialog with the keyboard alone | Sequence the focus to `transitionend` |
| A save reports nothing to a screen reader | The only signal is a movement | Complete the flow with a screen reader | Add a live region and a static state |

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| `@starting-style` and `transition-behavior: allow-discrete` reached every current browser | A JavaScript state machine that keeps a dialog mounted for its exit | Delete the state machine, and declare the two CSS rules |
| `interpolate-size: allow-keywords` and `calc-size()` animate an intrinsic size | A hook that measures `scrollHeight` and writes a pixel height | Take the CSS, or take `grid-template-rows` |
| Tailwind v4 replaced `tailwindcss-animate` with `tw-animate-css` | `rg 'tailwindcss-animate' package.json src/` reports a hit | `references/design-tokens-and-theming.md` owns the migration |

## Verification

```bash
# 1. Find a transition over a layout property. Each hit needs a written reason.
rg -n 'transition:[^;]*\b(width|height|top|left|right|bottom|margin)\b' \
  -g '*.css' -g '*.tsx' src/

# 2. Find a hand-typed duration or curve in a feature file. This prints nothing.
rg -n 'transition:[^;]*[0-9]+m?s|cubic-bezier\(' -g '*.tsx' src/

# 3. Find a global kill of every animation. This prints nothing.
rg -n 'animation:\s*none|transition:\s*none' -g '*.css' src/

# 4. Read the reduced-motion rules. One override sits over the tokens, beside
#    the global block.
rg -n -A6 'prefers-reduced-motion' src/

# 5. Find a will-change in a base class. Read every hit.
rg -n 'will-change' -g '*.css' -g '*.tsx' src/

# 6. Read the selectors around each allow-discrete hit. No later transition
#    shorthand follows the one that carries it.
rg -n -A4 'allow-discrete' -g '*.css' src/

# 7. Find a focus call beside an open call. Read every hit, and confirm the
#    sequence.
rg -n -B3 '\.focus\(\)' -g '*.tsx' src/

# 8. Record the interaction in the Performance panel. No long task appears, and
#    no frame carries a layout bar or a paint bar.

# 9. Repeat the recording at a 4× CPU throttle. The interaction still answers
#    at once.

# 10. Set the reduced-motion preference in the Rendering panel, then complete
#     one flow. Every state change still reaches the reader.

# 11. Trigger one animation ten times in a row. The animations do not stack,
#     and the element ends in the correct state.
```

## Review checklist

- [ ] Does every animation in the diff state a purpose class?
- [ ] Do the hot-path animations use `transform` and `opacity` alone?
- [ ] Does every animation over a layout property carry a written reason?
- [ ] Do the duration and the easing function come from tokens?
- [ ] Do the entrance and the exit of a platform element use `@starting-style`
      and `transition-behavior: allow-discrete`?
- [ ] Does a reveal of a height use `grid-template-rows` or `interpolate-size`?
- [ ] Is `will-change` scoped to the element that is about to animate, and
      removed after it?
- [ ] Does a reduced variant exist, defined once over the tokens?
- [ ] Are `animation: none` and `transition: none` absent from the project?
- [ ] Does the reduced-motion gate sit in CSS, rather than in JavaScript?
- [ ] Does every animated state change carry a static equivalent?
- [ ] Does motion that runs past 5 seconds carry a pause, a stop, or a hide
      control?
- [ ] Does nothing flash more than three times in one second?
- [ ] Does the focus move after an entrance completes, and never during it?
- [ ] Is a spinner absent from a response under about 100 milliseconds?
- [ ] Does a Performance recording at a 4× CPU throttle show no long task?

## Handoffs

- The `--duration-*`, `--ease-*`, and `--distance-*` tokens, the scale behind
  them, and `tw-animate-css` → `references/design-tokens-and-theming.md`.
- The classes on a part, and the variant that a state selects →
  `references/component-styles-and-variants.md`.
- The reserved box, the container query, and the skeleton geometry →
  `references/layout-and-typography.md`.
- The global `prefers-reduced-motion` block, the four user preferences, and the
  criterion verdict → `references/visual-and-motor-criteria.md`. That domain
  holds a veto.
- The element that takes the focus, the trap around a modal, and the live region
  that carries a message → `references/keyboard-focus-and-live-regions.md`. That
  domain holds a veto.
- The conformance claim, the axe lane, and the manual steps →
  `references/wcag-conformance-and-verification.md`. That domain holds a veto.
- The fallback, its shape, and the React 19 Action behind a pending state →
  `references/suspense-and-actions.md`.
- The cleanup that an effect returns, and the rule against a value in a
  dependency array → `references/state-and-effects.md`.
- The view transition, Motion, GSAP, and the bundle that each one costs →
  `references/view-transitions-and-animation-libraries.md`.
- The drag, the sheet, the scroll, and the animation that a scroll position
  drives → `references/gesture-and-scroll-interaction.md`.
- The `"use client"` directive over an animated island →
  `references/server-and-client-components.md`.
- The optimistic value that a mutation writes back →
  `references/server-state-and-query-cache.md`.
- The words of a label that reports a state change →
  `references/interface-copy-and-voice.md`.
- The INP threshold and the device profile →
  `references/performance-budgets-and-measurement.md`. The long task and the
  yield → `references/paint-and-interaction-cost.md`.
