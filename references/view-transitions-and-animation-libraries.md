# View transitions and animation libraries

Motion 12.43.x, GSAP 3.13, `@formkit/auto-animate` 0.10.0, the View Transition
API, Next.js 16.3, React 19.2.6 or later. This file owns the continuity between
two states, and the JavaScript that buys it. The subjects are the same-document
view transition, the name that carries one element across a snapshot, and the
React component that is still experimental. They also include the exit of an
element that React unmounts, the layout animation and its projection cost, and
Motion inside a Server Component tree. The last subjects are the library index
and the bundle that each entry costs.

The CSS primitives, the tokens, and the reduced variant are
`references/motion-primitives-and-reduced-motion.md`. The drag, the sheet, and
the scroll are `references/gesture-and-scroll-interaction.md`.

## Principle

A reader who moves from a list to a detail view has to find the same object
twice. A transition that carries the object across does that work for them.

The browser can take a picture of the old state and a picture of the new state,
and animate between the two. It needs one thing from the markup: a name that
appears exactly once in each picture.

A library that animates a layout does the arithmetic that the browser will not.
That arithmetic runs on the main thread, and it competes with everything else on
it.

A package that ships 34 kilobytes to fade one element is a bad trade. The same
package with the right import ships under 5.

An experimental API is a real API with no promise behind it. It is recommendable
only where the text states that it is experimental.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The same-document view transition

```tsx
// Wrong: two rows carry the same name in one snapshot.
// Failure: the browser cannot decide which box to morph, so it skips the whole
// transition and logs a warning. Nothing animates, and nothing states why.
<article style={{ viewTransitionName: "thumbnail" }}>…</article>
```

```tsx
// Correct: the name carries the identity of the record.
"use client"; // it starts a transition on a click

import { useRouter } from "next/navigation";

export function ListingLink({ id, children }: { id: string; children: React.ReactNode }) {
  const router = useRouter();

  function open() {
    if (!document.startViewTransition) {
      router.push(`/listings/${id}`);
      return;
    }
    document.startViewTransition(() => router.push(`/listings/${id}`));
  }

  return (
    <button type="button" onClick={open} style={{ viewTransitionName: `listing-${id}` }}>
      {children}
    </button>
  );
}
```

`document.startViewTransition` reached every current browser in October 2025,
and it is the stable path today. Give each `view-transition-name` a value that
is unique inside one snapshot, and derive it from the identity of the record.

Feature-detect the API. A browser without it takes the navigation with no
animation, which is the correct result.

CAUTION: a running view transition blocks pointer input until it ends. Keep the
duration short, and never wrap a transition around a slow navigation.

### The React component is experimental

`<ViewTransition>` and `addTransitionType` are in the Canary and Experimental
channels of React. React 19.2 shipped `<Activity>`, and it did not ship
`<ViewTransition>`. Recommend the component only with that condition stated.

```tsx
// Correct, with the condition stated: the component is experimental, so the
// installed React build decides between the prefixed name and the unprefixed
// one. This sample takes the prefixed name, and the verification block below
// reads the exports of the build.
import { unstable_ViewTransition as ViewTransition } from "react";

export function Thumbnail({ id, children }: { id: string; children: React.ReactNode }) {
  return <ViewTransition name={`listing-${id}`}>{children}</ViewTransition>;
}
```

Read the type exports of the installed React build, and take the name that the
build carries. NEVER assume which of the two names resolves.

`experimental.viewTransition: true` in `next.config.ts` turns on the deeper
integration, which covers the automatic transition types. Next.js 16.3 also
changed the timing of a navigation with Cache Components and with partial
prefetching. The composition of that timing with a transition moves fast. Read
the view transitions guide of the installed Next.js version before you wire one
onto a navigation. `references/app-router-structure.md` owns the router, the
prefetch, and `next.config.ts` itself.

### The cross-document transition is not available

The `@view-transition { navigation: auto }` rule animates between two documents.
It has limited availability, and it is not Baseline. Never rely on it for a
production multi-page transition. Take it as progressive enhancement, where a
browser with no support gives a plain navigation.

### The exit of an element that React unmounts

React removes a node from the document, and CSS then has no box to animate.
Two answers exist, and each one fits a different case.

Take `@starting-style` with `transition-behavior: allow-discrete` where the
platform owns the element, and where the exit needs no interruption.
`references/motion-primitives-and-reduced-motion.md` holds that pattern.

Take Motion `AnimatePresence` where React owns the mount, and where the exit
must be interruptible or must carry a spring curve. `AnimatePresence` keeps the
node in the document until the exit ends.

NEVER wrap `AnimatePresence` around a virtualised list. It mounts and measures
the whole tree, and the main thread stalls.
`references/data-table-and-server-driven-state.md` owns the virtualiser.

### Motion in a Server Component tree

```tsx
// Wrong: a Server Component file imports the client entry point.
// Failure: the module needs the browser APIs and the React client runtime, so
// the render fails. The error names the missing "use client" directive.
import { motion } from "motion/react";
```

```tsx
// Correct: the file becomes a Client Component.
"use client"; // it animates on mount

import { motion } from "motion/react";

export function Fade({ children }: { children: React.ReactNode }) {
  return <motion.div initial={{ opacity: 0 }} animate={{ opacity: 1 }}>{children}</motion.div>;
}
```

```tsx
// Correct: the file stays a Server Component, and the re-export carries the
// directive for it.
import * as motion from "motion/react-client";

export function Fade({ children }: { children: React.ReactNode }) {
  return <motion.div initial={{ opacity: 0 }} animate={{ opacity: 1 }}>{children}</motion.div>;
}
```

`references/server-and-client-components.md` owns the boundary rule and the
one-line reason above each directive. This file owns the two entry points that
Motion publishes.

### Motion obeys the preference

```tsx
"use client"; // it reads a media query and animates

import { motion, useReducedMotion } from "motion/react";

export function Sidebar({ isOpen }: { isOpen: boolean }) {
  const reduce = useReducedMotion();
  const closedX = reduce ? 0 : "-100%"; // it fades alone when the reader asked

  return <motion.aside animate={{ opacity: isOpen ? 1 : 0, x: isOpen ? 0 : closedX }} />;
}
```

Wrap the application in `<MotionConfig reducedMotion="user">` for the default,
and take `useReducedMotion` where one component needs a different variant. The
reduced variant is still a variant, and it still communicates the state change.
`references/motion-primitives-and-reduced-motion.md` owns that rule, and
`references/visual-and-motor-criteria.md` owns the preference itself.

### The layout animation and its cost

Motion `layout` and `layoutId` measure the old box and the new box, and they
animate a `transform` between the two. The measurement runs on the main thread
for each element, on each frame.

Scope `layout` to a small subtree. A broad `layout` prop over a large tree is a
per-frame arithmetic cost with no ceiling.

A `scale` transform stretches the children of the box. Add `layout` to a child
so it counter-scales, or take `layout="position"` where only the position
changes. Set `borderRadius` and `boxShadow` through `style` so Motion corrects
them each frame. `layout` does not work on `border`, on `display: inline`, or on
an SVG element.

Take `layoutId` over a view transition where several morphs must run at once,
where the reader can interrupt one, or where a gesture drives it. A view
transition blocks pointer input, and Motion does not.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those facts on 17 August 2026. A cell that holds no
date is a package with a current registry entry on that date. This file does not
state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `motion` | The default library, for an exit, a spring curve, a layout animation, and a gesture. Import from `motion/react`, or from `motion/react-client` in a Server Component file. | 12.43.x stable, 13.1.0 published | Current | Active, and a key ecosystem project | None |
| Recommend | `@formkit/auto-animate` | One line for the add, the remove, and the reorder of a list. Wrong where the transition needs control. | 0.10.0 | Current | Active | None |
| Recommend | `gsap` | A long orchestrated timeline, an inertia, and ScrollTrigger. Free for commercial use since 30 April 2025. | 3.13 | May 2025 | Active, and owned by Webflow | None |
| Conditional | `@rive-app/react-canvas` | Only where the product needs a bespoke interactive vector animation. Always with a static poster under the reduced-motion preference. | Current | Current | Active | None |
| Conditional | `@dotlottie/react-player` | Only for a Lottie illustration that a designer delivers. Audit the file size and the CPU cost first. | Current | Current | Active | None |
| Conditional | `@react-spring/web` | Only for continuous physics-driven motion, and only where Motion is not already in the bundle. | Current | Current | Active | None |
| Conditional | `three` with `@react-three/fiber` | Only where the effect needs WebGL. Load it on demand, and gate it behind the reduced-motion and the reduced-data preference. | Current | Current | Active | None |
| Audit-only | `framer-motion` | The old package name. Move to `motion`, and import from `motion/react`. The React API did not break at v12. | — | — | Superseded | — |
| Audit-only | `react-transition-group` | A legacy entrance and exit. Take `AnimatePresence`, or the CSS pattern. | — | — | Superseded | — |
| Audit-only | `next-view-transitions` | A third-party wrapper. The platform API and the React component now cover most cases. Re-evaluate it. | — | — | Superseded | — |
| Reject | A Lottie file for a spinner or a simple loader | It costs CPU and bundle bytes for what one CSS rule does. | — | — | — | — |

Motion publishes two entry points with very different weights. The full
`motion` component is about 34 kilobytes, and no bundler splits it smaller. The
`m` component with `LazyMotion` is under 4.6 kilobytes for the first render. A
feature pack then adds its own bytes. `domAnimation` adds about 15, and
`domMax` adds about 25. The `useAnimate` mini API is 2.3 kilobytes.

`references/performance-budgets-and-measurement.md` owns the budget that
these numbers meet.

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Framer Motion became Motion in mid-2025 | `rg 'from "framer-motion"' src/` reports a hit | Install `motion`, and import from `motion/react` |
| GSAP 3.13 made every plugin free on 30 April 2025 | A comment that gates ScrollTrigger or SplitText behind a licence | Delete the gate. SplitText is rewritten, about half the size, and readable by a screen reader |
| React `<ViewTransition>` and `addTransitionType` stay in the Canary channel | `rg 'ViewTransition' src/` reports an import from `react` on a stable build | Read the type exports of the installed React, and take the name it carries |
| `document.startViewTransition` reached every current browser in October 2025 | A third-party wrapper around a same-document transition | Take the platform API, and re-evaluate the wrapper |
| The cross-document `@view-transition` rule has limited availability | A multi-page transition with no fallback | Treat the rule as progressive enhancement |

## Verification

```bash
# 1. Read the installed versions before you write code.
rg -n '"motion"|"framer-motion"|"gsap"|"@formkit/auto-animate"|"react"' package.json

# 2. Find the old package name. This prints nothing.
rg -n 'from "framer-motion"' -g '*.ts*' src/

# 3. Find a Motion import in a file with no client directive. Each hit uses
#    motion/react-client, or the file carries the directive.
rg --files-with-matches 'from "motion/react"' -g '*.tsx' src/ \
  | xargs rg --files-without-match '"use client"'

# 4. Find a view-transition-name. Each value interpolates an identifier, and no
#    value is a bare literal that two elements can share.
rg -n 'viewTransitionName|view-transition-name|ViewTransition name=' -g '*.ts*' -g '*.css' src/

# 5. Find a ViewTransition import. Confirm the exported name against the
#    installed React build.
rg -n -B2 'ViewTransition' -g '*.tsx' src/
node -p "Object.keys(require('react')).filter((k) => k.includes('ViewTransition'))"

# 6. Find a startViewTransition call with no feature detection. Read every hit.
rg -n -B3 'startViewTransition' -g '*.ts*' src/

# 7. Find AnimatePresence or a layout prop over a virtualised list. Read every
#    hit.
rg -n 'AnimatePresence|<motion\.[a-z]+ layout' -g '*.tsx' src/

# 8. Find the reduced-motion path in Motion. At least one hit is present where
#    Motion is installed.
rg -n 'MotionConfig|useReducedMotion' -g '*.tsx' src/

# 9. Read the bundle report for the route that imports Motion. The first render
#    carries the m component and a feature pack, or it states why it carries
#    the full component.

# 10. Navigate from the list to the detail view, and press the browser back
#     button in the middle of the transition. The application ends in the
#     correct state.

# 11. Set the reduced-motion preference, and repeat the navigation. The
#     transition still communicates the change, with no large movement.
```

## Review checklist

- [ ] Is every `view-transition-name` unique inside one snapshot?
- [ ] Does a `startViewTransition` call sit behind a feature detection?
- [ ] Does every use of `<ViewTransition>` or `addTransitionType` state that
      the API is experimental?
- [ ] Was the exported React name read from the installed build, rather than
      assumed?
- [ ] Does a cross-document `@view-transition` rule carry a fallback?
- [ ] Does the exit of an unmounted element use `AnimatePresence`, or the CSS
      pattern?
- [ ] Is `AnimatePresence` absent from every virtualised list?
- [ ] Does every Motion import sit behind `"use client"`, or come from
      `motion/react-client`?
- [ ] Does the application set `MotionConfig reducedMotion="user"`, or handle
      the preference for each component?
- [ ] Is the `layout` prop scoped to a small subtree?
- [ ] Do the children of a `layout` box counter-scale, or does the box take
      `layout="position"`?
- [ ] Does the bundle carry the `m` component with `LazyMotion`, or state why it
      carries the full component?
- [ ] Is `framer-motion`, `react-transition-group`, or `next-view-transitions`
      absent from new code?
- [ ] Is a Lottie file absent from every spinner and loader?

## Handoffs

- The CSS transition, the tokens, the reduced variant, and the focus that
  follows an entrance →
  `references/motion-primitives-and-reduced-motion.md`.
- The drag, the sheet, and the animation that a scroll position drives →
  `references/gesture-and-scroll-interaction.md`.
- The router, the prefetch, `next.config.ts`, and the navigation that a
  transition wraps → `references/app-router-structure.md`.
- The `"use client"` directive and the one-line reason above it →
  `references/server-and-client-components.md`.
- The shape of a component, the hook rules, and `<Activity>` →
  `references/component-composition.md` and
  `references/suspense-and-actions.md`.
- The virtualiser and the row model behind a long list →
  `references/data-table-and-server-driven-state.md`.
- The optimistic value, the mutation, and the write back into the cache →
  `references/server-state-and-query-cache.md`.
- The focus that a navigation must move, and the announcement of a route change
  → `references/keyboard-focus-and-live-regions.md`. That domain holds a veto.
- The reduced-motion preference and the conformance verdict over it →
  `references/visual-and-motor-criteria.md`. That domain holds a veto.
- A new dependency, its size, and its maintenance status →
  `references/dependencies-and-git-workflow.md`.
- The bundle budget over an animation library →
  `references/performance-budgets-and-measurement.md`. The long task and the
  INP threshold → `references/paint-and-interaction-cost.md`.
- The supply chain of an animation dependency, and its advisories →
  `references/secret-boundary-and-supply-chain.md`. That domain holds a veto.
