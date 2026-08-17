# Gesture and scroll interaction

`@dnd-kit/core` with `@dnd-kit/sortable`, `vaul`, Motion 12.43.x, CSS
scroll-driven animations, Tailwind CSS v4.3, Next.js 16.3, React 19.2.6 or
later. This file owns the motion that the reader drives with a finger, a
pointer, or a scroll. The subjects are the drag with its sensors, the sheet that
a drag dismisses, and the scroll that stays under the control of the reader.
They also include the animation that a scroll position drives, and the fallback
under a browser that does not run one. The last subjects are the parallax gate
and the value that a pointer moves.

The CSS transition, the tokens, and the reduced variant are
`references/motion-primitives-and-reduced-motion.md`. The view transition and
the animation libraries are
`references/view-transitions-and-animation-libraries.md`.

## Principle

A drag asks for two working hands, a steady grip, and fine control. A product
that offers no other path excludes the readers who have none of the three.

The scroll belongs to the reader. A page that takes it over removes the one
control that every reader already knows.

A scroll handler runs on the main thread, and it runs for every scroll event. A
handler that reads the size of a box inside itself makes the browser lay out the
page in the middle of a scroll.

The browser can bind an animation to a scroll position with no listener at all.
That animation runs on the compositor, and it costs the main thread nothing.

A large background movement that a scroll drives makes some readers sick. The
browser already reports which readers those are.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The drag needs sensors, and a second path beside them

```tsx
// Wrong: the pointer sensor is the only sensor.
// Failure: a keyboard reaches the handle and then does nothing with it, so no
// keyboard path through the list exists. A reader with a tremor cannot hold the
// drag either, and criterion 2.5.7 fails.
const sensors = useSensors(useSensor(PointerSensor));
```

```tsx
// Correct: a keyboard sensor beside the pointer sensor, and a threshold that
// separates a drag from a click.
"use client"; // it holds the drag state

import {
  DndContext,
  KeyboardSensor,
  PointerSensor,
  useSensor,
  useSensors,
} from "@dnd-kit/core";

export function SortableRows({ children }: { children: React.ReactNode }) {
  const sensors = useSensors(
    useSensor(PointerSensor, { activationConstraint: { distance: 8 } }),
    useSensor(KeyboardSensor),
  );

  return <DndContext sensors={sensors}>{children}</DndContext>;
}
```

Take `@dnd-kit/core` with `@dnd-kit/sortable` for every sortable list and every
drag target. It replaces the native HTML drag and drop API, which gives no
keyboard path and no control over the drag image.

CAUTION: a keyboard sensor is not the second path. Criterion 2.5.7 asks for a
path that one pointer completes with no drag, such as a pair of move controls or
a menu. `references/visual-and-motor-criteria.md` holds the control set that
answers it, and that domain holds a veto.

### The scroll belongs to the reader

NEVER take over the scroll of the document. A handler on `wheel` or on `scroll`
can move the page by its own amount. That handler breaks the scrollbar, the
keyboard, the trackpad, and the reading position. It also makes some readers
sick.

State the scroll behaviour in CSS rather than in a handler. `scroll-snap-type`
holds a sectioned page at its section edges. `overscroll-behavior` states what
happens at the end of a scroll container. `touch-action` states which gestures
the element handles itself. Each of the three runs with no listener, and a
handler that reproduces one of them is the wrong mechanism.

### The animation that a scroll position drives

```ts
// Wrong: a listener drives the progress bar.
// Failure: the handler runs on the main thread for every scroll event, and it
// reads a layout value inside that handler. The scroll stutters on a phone.
window.addEventListener("scroll", () => {
  const max = document.body.scrollHeight - window.innerHeight;
  bar.style.width = `${(window.scrollY / max) * 100}%`;
});
```

```css
/* Correct: the compositor drives the bar from the scroll timeline. */
@supports (animation-timeline: scroll()) {
  .progress {
    transform-origin: left;
    animation: grow linear both;
    animation-timeline: scroll(root block);
  }
}
@keyframes grow {
  from { transform: scaleX(0); }
  to { transform: scaleX(1); }
}
```

`scroll()` binds the animation to the scroll position of a container. `view()`
binds it to the position of the element inside the viewport, which is the
mechanism behind a reveal.

Set `animation-fill-mode: both`, which the `both` keyword in the shorthand
above carries. Without it the animation drops back to the start state at a
scroll position of zero, and the element snaps.

Wrap the rule in `@supports (animation-timeline: scroll())`. A browser with no
support then shows the final state and no movement, which is a correct page.
Treat the whole feature as progressive enhancement, and never as a requirement.

### Parallax runs only where the reader allows it

```css
/* Correct: the gate is the media query, and the default is no movement. */
@media (prefers-reduced-motion: no-preference) {
  @supports (animation-timeline: view()) {
    .layer-back {
      animation: drift linear both;
      animation-timeline: view();
    }
  }
}
```

A parallax layer, a large scroll reveal, and any wide background movement each
need this gate. A movement that an interaction triggers, and that the reader
cannot stop, fails criterion 2.3.3.
`references/motion-primitives-and-reduced-motion.md` states the criterion, and
`references/wcag-conformance-and-verification.md` owns the verdict.

Write the query as `no-preference`, and never as `reduce`. The `no-preference`
form makes no movement the default, so a browser that reports nothing gives the
safe answer.

### The value that a pointer or a scroll moves

Take Motion `useMotionValue`, `useScroll`, and `useSpring` for a value that
follows a pointer or a scroll position with a spring curve. Motion holds these
values outside the React state, so a move costs no render.

This mechanism fits a continuous value alone. A one-shot state change takes a
CSS transition, which
`references/motion-primitives-and-reduced-motion.md` holds.
`references/view-transitions-and-animation-libraries.md` owns the Motion entry
points, the bundle cost, and the reduced-motion configuration.

### The sheet that a drag dismisses

Take `vaul` for a bottom sheet and for a drawer that a drag dismisses. It builds
on a dialog, so the focus trap and the dismiss contract come with it.

The sheet still needs a control that a pointer presses to close it, and a
keyboard path to that control. A drag is the fast path, and it is never the only
path. `references/keyboard-focus-and-live-regions.md` owns the focus inside the
sheet, and that domain holds a veto.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those facts on 17 August 2026. A cell that holds no
date is a package with a current registry entry on that date. This file does not
state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `@dnd-kit/core` with `@dnd-kit/sortable` | Every sortable list and every drag target. It carries a keyboard sensor and an accessible announcement. | Current | Current | Active, with an accessibility focus | None |
| Recommend | `vaul` | The bottom sheet and the drawer that a drag dismisses. | Current | Current | Active | None |
| Conditional | `motion` for `useScroll` and `useSpring` | Only for a continuous value that a pointer or a scroll drives. `references/view-transitions-and-animation-libraries.md` holds its version facts. | 12.43.x stable | Current | Active | None |
| Reject | The native HTML drag and drop API for a sortable list | It gives no keyboard path, and no control over the drag image. | — | — | — | — |
| Reject | A parallax library with no reduced-motion path | It triggers a vestibular response, and criterion 2.3.3 fails with it. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| A list that only a drag reorders | No pointer path beside the drag | Reorder the list with one pointer and no drag | Add the move controls that domain 10 states |
| A handle that the keyboard reaches and cannot use | No keyboard sensor | Tab to the handle, and press the arrow keys | Add `KeyboardSensor` to the sensor list |
| A drag that fires on a click | No activation constraint | Click a row without moving the pointer | Set a distance constraint on the pointer sensor |
| The scroll stutters on a phone | A scroll listener reads a layout value | Long task bars in the Performance recording | Take a scroll timeline, and delete the listener |
| A scroll animation does nothing in one browser | No scroll timeline support, and no fallback | Load the page in a browser with no support | Wrap the rule in `@supports` |
| The element snaps back at the top of the page | No `animation-fill-mode` | Scroll down, then scroll back to the top | Add `both` to the animation shorthand |
| A reader reports motion sickness on the landing page | A parallax layer with no gate | Set the reduced-motion preference, and reload | Gate the layer behind `no-preference` |
| The page fights the scrollbar and the trackpad | A `wheel` handler moves the page itself | Scroll with a trackpad and with the keyboard | Delete the handler, and take `scroll-snap-type` |

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| CSS scroll-driven animations replaced the scroll listener | `rg "addEventListener\(\"scroll\"" src/` reports a hit that drives a style | Take `animation-timeline: scroll()` or `view()`, behind `@supports` |
| Framer Motion became Motion in mid-2025 | `rg 'from "framer-motion"' src/` reports a hit on `useScroll` | Install `motion`, and import from `motion/react` |

## Verification

```bash
# 1. Read the installed versions before you write code.
rg -n '"@dnd-kit/core"|"@dnd-kit/sortable"|"vaul"|"motion"' package.json

# 2. Find the native drag and drop API on a sortable list. Read every hit.
rg -n 'draggable|onDragStart|dataTransfer' -g '*.tsx' src/

# 3. Find a sensor list with no keyboard sensor.
rg --files-with-matches 'useSensors' -g '*.tsx' src/ \
  | xargs rg --files-without-match 'KeyboardSensor'

# 4. Find a scroll or wheel listener. Each hit needs a written reason, and no
#    hit moves the page itself.
rg -n 'addEventListener\("(scroll|wheel)"|onWheel' -g '*.ts*' src/

# 5. Find a scroll timeline with no support query. This prints nothing.
rg --files-with-matches 'animation-timeline' -g '*.css' src/ \
  | xargs rg --files-without-match '@supports'

# 6. Find a scroll timeline with no fill mode. Read every hit.
rg -n -B4 'animation-timeline' -g '*.css' src/

# 7. Find a parallax layer with no preference gate. Read every hit.
rg -n -B6 'parallax' -g '*.css' -g '*.tsx' src/

# 8. Reorder one list with the keyboard alone. Then reorder it with one pointer
#    and no drag. Both paths complete.

# 9. Load the page in a browser with no scroll timeline support. The content is
#    complete, and it holds its final state.

# 10. Set the reduced-motion preference, and scroll the landing page. No large
#     background movement plays.

# 11. Scroll one page with a trackpad, with a scrollbar, and with the keyboard.
#     Each input moves the page by the amount the reader asked for.
```

## Review checklist

- [ ] Does every sortable list use `@dnd-kit`, rather than the native drag and
      drop API?
- [ ] Does the sensor list carry a keyboard sensor beside the pointer sensor?
- [ ] Does the pointer sensor carry an activation constraint?
- [ ] Does every drag carry a single-pointer path that needs no drag?
- [ ] Is a handler that moves the page itself absent from the project?
- [ ] Does the scroll behaviour come from `scroll-snap-type`,
      `overscroll-behavior`, and `touch-action`, rather than from a listener?
- [ ] Does every scroll-driven animation sit behind `@supports`?
- [ ] Does every scroll-driven animation set `animation-fill-mode: both`?
- [ ] Does the page hold its final state where the browser runs no scroll
      timeline?
- [ ] Is every parallax layer gated behind
      `prefers-reduced-motion: no-preference`?
- [ ] Does a continuous pointer value use `useMotionValue` or `useScroll`,
      rather than the React state?
- [ ] Does a sheet that a drag dismisses also carry a control that a pointer
      presses?

## Handoffs

- The CSS transition, the tokens, the reduced variant, and the criteria that
  limit motion → `references/motion-primitives-and-reduced-motion.md`.
- The Motion entry points, the bundle cost, and the view transition →
  `references/view-transitions-and-animation-libraries.md`.
- Criterion 2.5.7, the second path beside a drag, the target size, and the four
  user preferences → `references/visual-and-motor-criteria.md`. That domain
  holds a veto.
- The focus inside a sheet, the trap around it, and the announcement of a
  reorder → `references/keyboard-focus-and-live-regions.md`. That domain holds a
  veto.
- The role and the accessible name of a drag handle →
  `references/semantics-and-accessible-names.md`. That domain holds a veto.
- The conformance verdict over criterion 2.3.3 →
  `references/wcag-conformance-and-verification.md`. That domain holds a veto.
- The row model, the virtualiser, and the order that a pushed row must not
  disturb → `references/data-table-and-server-driven-state.md`.
- The mutation that saves a new order, and the optimistic write behind it →
  `references/server-state-and-query-cache.md` and
  `references/data-access-and-mutations.md`.
- The `"use client"` directive over a drag surface →
  `references/server-and-client-components.md`.
- The tokens behind a sheet and a handle →
  `references/design-tokens-and-theming.md`.
- A new dependency, its size, and its maintenance status →
  `references/dependencies-and-git-workflow.md`.
- The long task, the INP threshold, and the budget over a scroll effect →
  domain 16 `performance-and-web-vitals`. Not integrated yet.
- The words on a move control and on a drag announcement → domain 15
  `ux-writing-and-content-design`. Not integrated yet.
