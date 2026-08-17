# Paint and interaction cost

Next.js 16.3, React 19.2.6 or later with React Compiler 1.0, TypeScript 5.9, the
`scheduler.yield()` platform API. This file owns what the browser paints and how
fast it answers a tap. The subjects are the element that paints the largest area,
the four parts of that measurement, and the layout that holds still. They also
include the interaction that answers within 200 ms, the long task that must
yield, and the update that waits behind an urgent one. The last subjects are the
render that the compiler removes and the memory that a session grows.

The thresholds, the budget, and the measurement loop are
`references/performance-budgets-and-measurement.md`. The bytes that ship and the
third-party script are `references/client-bundle-and-third-party-scripts.md`.

## Principle

Every route has one element that paints the largest area. A team that cannot
name it cannot make it arrive sooner.

A slow paint has four possible causes, and only one of them is the size of the
resource. Work on the wrong one returns nothing.

Layout stability is a property of the markup, and not a defect to patch. Space
that the first paint reserves cannot shift when the content arrives.

A reader judges an interaction by the frame that follows it. Work that holds the
main thread holds that frame.

Not every update is urgent. The value in an input is urgent. The list that the
value filters is not.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The largest paint has one element, and the route names it

| The part | What it covers | Who owns the fix |
| --- | --- | --- |
| The first byte | The time until the first byte of the document arrives | `references/caching-and-revalidation.md`, and the sibling skill `django-performance-optimizer` |
| The resource load delay | The gap between the first byte and the start of the request for the element | This file. The document must name the resource early. |
| The resource load duration | The download of the resource itself | `references/image-and-video-delivery.md` for the format and the variant |
| The render delay | The gap between the arrival of the resource and the paint | This file, and `references/client-bundle-and-third-party-scripts.md` |

```tsx
// Wrong: nobody named the element that paints the largest area.
// Failure: `next/image` is lazy by default. The browser discovers the hero
// after the layout runs, so the resource load delay grows by a whole round
// trip. No change to the image format moves that part of the measurement.
<Image src={hero} alt="" width={1200} height={600} />
```

```tsx
// Correct: the route names the element, and the document fetches it early.
<Image src={hero} alt="" width={1200} height={600} preload sizes="100vw" />
```

Name the element for every route, and record the name. On a marketing route it
is usually the hero image. On an application route it is usually a heading or a
first block of text.

NEVER put `loading="lazy"` on that element, and never load it through a
deferred import. Both move its discovery out of the first document.

Where the element is text in a font that the page loads, the paint waits for the
font. Where it is text in a system font, the paint is the first byte plus the
render, and no image work helps.

`references/image-and-video-delivery.md` owns the props of `<Image>`, the
`preload` prop against `fetchPriority`, and the source set behind them. This
file owns the decision about which element receives that treatment.

Read the four parts before you change anything. A team that optimises the
download while the first byte dominates spends a release and moves no number.

### The layout is stable by construction

```tsx
// Wrong: the notice mounts above the list once the request resolves.
// Failure: every row moves down by the height of the notice. A reader who was
// about to press a control presses the one that took its place.
<>
  {notice ? <Banner>{notice}</Banner> : null}
  <OrderList orders={orders} />
</>
```

```tsx
// Correct: the box exists from the first paint, and it holds its own height.
<>
  <div className="min-h-12">{notice ? <Banner>{notice}</Banner> : null}</div>
  <OrderList orders={orders} />
</>
```

Three sources produce almost every shift. An image or an embed with no reserved
box. A font that swaps to metrics that do not match the fallback. An element
that mounts above content that is already on screen.

Reserve the space in the markup. `references/layout-and-typography.md` owns the
ratio box, the skeleton geometry, and the `next/font` options that generate the
metric overrides. This file states that a shift is a failure of the budget, and
that a patch after the fact is not a fix.

CAUTION: a skeleton is a promise about geometry. A skeleton whose height differs
from the content that replaces it produces the shift that it was added to
prevent.

### An interaction answers within 200 ms

| The part | What it covers | The usual cause of a long one |
| --- | --- | --- |
| The input delay | The wait before the handler runs | A long task that is already on the main thread, often a third-party script |
| The processing duration | The handler itself, and the render that it starts | Synchronous work in the handler, and a render of a large subtree |
| The presentation delay | The wait until the next frame paints | A large layout, or a paint of many nodes |

```tsx
// Wrong: the handler does the whole transform before it returns.
// Failure: the main thread is held for more than 50 ms under a 4× CPU throttle.
// The button does not press, no spinner appears, and a second tap queues behind
// the first one.
function onGenerate() {
  const report = buildReport(rows);
  setReport(report);
}
```

```ts
// Correct: lib/yield-to-main.ts holds one yield helper, with a fallback.
type SchedulerWithYield = { yield: () => Promise<void> };

function hasYield(value: unknown): value is SchedulerWithYield {
  return typeof value === "object" && value !== null && "yield" in value;
}

export async function yieldToMain(): Promise<void> {
  const candidate = (globalThis as { scheduler?: unknown }).scheduler;
  if (hasYield(candidate)) {
    await candidate.yield();
    return;
  }
  await new Promise<void>((resolve) => {
    setTimeout(resolve, 0);
  });
}
```

```tsx
// Correct: the work runs in chunks, and the result lands as a non-urgent update.
async function onGenerate() {
  setPending(true);
  for (const chunk of chunksOf(rows, 200)) {
    accumulate(chunk);
    await yieldToMain();
  }
  startTransition(() => {
    setReport(collect());
  });
  setPending(false);
}
```

Hold no interaction handler on the main thread for more than 50 ms. Measure the
handler under a 4× CPU throttle, because an unthrottled desktop hides the cost.

CAUTION: `scheduler.yield()` is not available in every browser. Always ship the
fallback beside it, as the helper above does.

`references/gesture-and-scroll-interaction.md` owns the scroll handler, and
`references/motion-primitives-and-reduced-motion.md` owns the property that one
frame can afford. Both costs land in this measurement.

### The urgent update stays urgent

| The condition | The decision | What forces the change |
| --- | --- | --- |
| An expensive view derives from a value, and this component owns the setter | `startTransition` around the expensive update alone | The setter moves to a parent or to a library, so this component no longer controls it. |
| An expensive view derives from a value, and this component does not own the setter | `useDeferredValue` over the value | The component gains the setter, and `startTransition` then states the intent more directly. |
| The value is the text in an input | Neither. The input value stays an urgent update. | Nothing. A deferred input value drops keystrokes and reads as a broken field. |

The cost of both is one stale frame in the derived view. The reader sees the
previous list for one frame while the new one renders.

`references/suspense-and-actions.md` owns `useTransition`, `useActionState`, and
the pending state that a submit shows. `references/state-and-effects.md` owns
where the value lives.

### The render that the compiler removes

React Compiler 1.0 became stable on 7 October 2025. With `reactCompiler: true`
the build inserts the memoization that a hand-written `useMemo`, `useCallback`,
or `memo` used to carry. Meta reported up to 12 percent faster initial loads and
more than 2.5 times faster interactions on one product after it adopted the
compiler.

In a project that runs the compiler, a hand-written memo needs a measurement
beside it. Without one it is code that the build already wrote, and it hides the
bailouts that matter.

`references/state-and-effects.md` owns the compiler configuration, the
`"use memo"` and `"use no memo"` directives, and the bailout rules.

### The offscreen DOM

A long list costs layout time and paint time for every node, on screen or not.
`references/data-table-and-server-driven-state.md` owns that decision. It states
the row count at which a virtualiser earns its cost, and the case for
`content-visibility: auto` with `contain-intrinsic-size` instead. This file adds
no second threshold.

### The memory that a session grows

A single-page application keeps its heap for the whole visit. A leak that a
reload would hide becomes a slow tab after twenty minutes.

Four causes account for most of it. A listener, a timer, or an observer that no
cleanup removes. An object URL that no code revokes. A client cache with no
bound. A closure that holds a large response after the view is gone.

Compare two heap snapshots, taken before and after the same navigation cycle.
A retained object that the cycle should have released names its own cause.

`references/state-and-effects.md` owns the cleanup that an effect returns.
`references/image-and-video-delivery.md` owns `URL.revokeObjectURL`.
`references/server-state-and-query-cache.md` owns `gcTime` and the bound on the
query cache.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those facts on 17 August 2026. A cell that holds no
date is a package with a current registry entry on that date. This file does not
state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `scheduler.yield()`, with a `setTimeout` fallback | Every loop in an interaction handler that can run past 50 ms. | Web platform | — | Chrome and Edge 129 and later, Firefox 143 and later | None |
| Recommend | `startTransition` and `useDeferredValue` | Every expensive update that the reader does not wait on. | react 19.2.x | Current | Meta, active | None |
| Recommend | The React Compiler, behind `reactCompiler: true` | Every project on React 19. It writes the memoization that a hand-written hook used to carry. | 1.0 | 2025-10-07 | Meta, active | None |
| Conditional | The React DevTools profiler | Only for a render cost, and only on the production profiling build. | — | — | — | — |
| Conditional | The Rust port of the compiler, behind `turbopackRustReactCompiler` | Only where the build time is the problem. It is experimental. | Ships with Next 16.3 | 2026-08-03 | Vercel, active | None |
| Reject | A hand-written `useMemo`, `useCallback`, or `memo` with no measurement, in a project that runs the compiler | The build already wrote it. The hand-written form hides the bailout that matters. | — | — | — | — |
| Reject | `setTimeout(fn, 0)` as the only yield | A timeout puts the continuation behind every task that is already queued. Prefer the scheduler, and keep the timeout as the fallback. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| The largest paint does not move after the image work | The first byte or the render delay dominates | Read the four parts of the measurement | Work on the part that dominates, and hand a slow first byte to the server |
| The hero arrives late on a slow link | The element is lazy, and the document never named it | Read the resource load delay | Set `preload` on that element, and delete any lazy attribute |
| The layout shift number rises only in production | The font swaps to metrics that do not match the fallback | Reload on a throttled profile, and read the field value | `references/layout-and-typography.md` holds the metric overrides |
| The content moves down a second after load | An element mounts above content that is on screen | Reload on a throttled profile | Reserve the box from the first paint |
| The interaction is fine in development and poor in production | The development build hides the production bundle and the third-party script | The field attribution report | Break the long task, yield, and defer the non-urgent update |
| The button does not press | The handler holds the main thread past 50 ms | Profile the handler under a 4× CPU throttle | Chunk the work, and yield between the chunks |
| The input drops keystrokes | The input value update sits inside a transition | Type quickly into the field | Keep the value update urgent, and defer the derived view alone |
| The tab slows after twenty minutes | A listener, a timer, an object URL, or an unbounded cache | Compare two heap snapshots across one navigation cycle | Return a cleanup, revoke the URL, and bound the cache |
| A render cost that no change explains | A hand-written memo hides a compiler bailout | The React DevTools profiler on the production profiling build | Delete the memo that carries no measurement |

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| React Compiler 1.0 became stable on 7 October 2025 | `rg -n 'useMemo\(|useCallback\(|memo\(' -g '*.tsx' src/` reports hits in a project with `reactCompiler: true` | Delete each memo that carries no measurement, and let the build insert it |
| `scheduler.yield()` is not available in every browser | A call with no fallback beside it | Route every call through one helper that carries the `setTimeout` fallback |
| Next 16 deprecated the `priority` prop of `<Image>` | `rg -n '\bpriority\b' -g '*.tsx' src/` reports a hit | Take `preload`. `references/image-and-video-delivery.md` holds the rule |
| Next 16.3 keeps a route shell in the browser for an instant navigation | A Next version below 16.3 in `package.json` | Upgrade. The release notes state no memory figure for the client cache, so read the memory track on the target device before you enable it across every route |

## Verification

```bash
# 1. Read the installed versions before you write code.
rg -n '"next"|"react"|"babel-plugin-react-compiler"' package.json

# 2. Confirm that the compiler is on before you judge a hand-written memo.
rg -n 'reactCompiler' next.config.ts

# 3. Find a lazy attribute. No hit is the element that paints the largest area.
rg -n 'loading="lazy"' -g '*.tsx' src/

# 4. Find the deprecated prop on an image. This prints nothing.
rg -n '\bpriority\b' -g '*.tsx' src/

# 5. Find a yield call with no fallback beside it. This prints nothing.
rg --files-with-matches 'scheduler' -g '*.ts*' src/ \
  | xargs rg --files-without-match 'setTimeout'

# 6. Find an input value update inside a transition. Read every hit.
rg -n -A4 'startTransition' -g '*.tsx' src/

# 7. Find a hand-written memo. In a project that runs the compiler, each hit
#    needs a measurement beside it.
rg -n 'useMemo\(|useCallback\(|React\.memo\(' -g '*.tsx' src/

# 8. Find an effect that adds a listener and returns no cleanup.
rg --files-with-matches 'addEventListener' -g '*.tsx' src/ \
  | xargs rg --files-without-match 'removeEventListener'

# 9. Load the primary route at a 4× CPU throttle and Slow 4G. Read the four
#    parts of the largest paint, and record which one dominates.

# 10. Press the primary control under the same throttle. The next frame paints
#     within 200 ms, and no task in the handler runs past 50 ms.

# 11. Reload the route on the same profile. No element moves after the first
#     paint.

# 12. Take a heap snapshot, run one navigation cycle, and take a second
#     snapshot. No object from the first view is retained.
```

## Review checklist

- [ ] Does every route name the element that paints the largest area?
- [ ] Does that element carry `preload`, and is `loading="lazy"` absent from it?
- [ ] Is that element outside every deferred import?
- [ ] Were the four parts of the measurement read before any work started?
- [ ] Does every image, embed, and skeleton hold a reserved box?
- [ ] Does every skeleton hold the height of the content that replaces it?
- [ ] Is every element that mounts above on-screen content given space from the
      first paint?
- [ ] Does every interaction handler stay under 50 ms at a 4× CPU throttle?
- [ ] Does every long loop in a handler yield through the shared helper?
- [ ] Does that helper carry the `setTimeout` fallback?
- [ ] Does every expensive derived update sit in a transition or behind
      `useDeferredValue`?
- [ ] Does the input value itself stay an urgent update?
- [ ] Does every hand-written memo carry a measurement, in a project that runs
      the compiler?
- [ ] Does every listener, timer, and observer have a cleanup?

## Handoffs

- The thresholds, the budget, the CI gate, and the field report →
  `references/performance-budgets-and-measurement.md`.
- The bytes of JavaScript, the third-party script, and the prefetch →
  `references/client-bundle-and-third-party-scripts.md`.
- The props of `<Image>`, `preload` against `fetchPriority`, the source set, and
  `URL.revokeObjectURL` → `references/image-and-video-delivery.md`.
- The reserved ratio box, the skeleton geometry, and the `next/font` metric
  overrides → `references/layout-and-typography.md`.
- The row count at which a virtualiser earns its cost, and
  `content-visibility: auto` → `references/data-table-and-server-driven-state.md`.
- The compiler configuration, the bailout, the effect cleanup, and where a value
  lives → `references/state-and-effects.md`.
- `useTransition`, the pending state of a submit, and the shape of a fallback →
  `references/suspense-and-actions.md`.
- The property that one frame can afford, and the `will-change` hint →
  `references/motion-primitives-and-reduced-motion.md`.
- The scroll handler, and the animation that a scroll position drives →
  `references/gesture-and-scroll-interaction.md`.
- `gcTime`, and the bound on the client cache →
  `references/server-state-and-query-cache.md`.
- The cache decision behind a slow first byte →
  `references/caching-and-revalidation.md`.
- The reduced-motion preference, the contrast ratio, and the conformance verdict
  → `references/visual-and-motor-criteria.md` and
  `references/wcag-conformance-and-verification.md`. That domain holds a veto.
- The error tracker, the trace, and the transport of a field report → domain 21
  `observability-and-resilience`. Not integrated yet.
- The slow endpoint behind a first byte → sibling skill
  `django-performance-optimizer`.
