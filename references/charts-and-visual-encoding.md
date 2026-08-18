# Charts and visual encoding

recharts 3.10.1, Next.js 16.3, React 19.2.6 or later, Tailwind CSS v4.3. This
file owns the chart. The subjects are the mark that answers the question, the
library that draws it, and the boundary that it renders on. They also include
the second channel beside the color, and the text that stands for the picture.

The table under the chart is `references/data-table-and-server-driven-state.md`.
The number on an axis is `references/cell-formatting-and-export.md`. The color
tokens are `references/design-tokens-and-theming.md`.

## Principle

A chart is an argument about data. The mark carries the argument, so the mark
follows the question and never the fashion.

A baseline that does not start at zero misstates a ratio. The reader compares
the length of the bars, and the length is then a lie.

Color is one channel, and part of the audience does not receive it. A chart that
carries meaning in color alone carries no meaning for those readers.

A picture is not text. A chart with no text alternative holds no content for a
screen reader, and no content in a search index.

A canvas has no server DOM. A library that paints on a canvas cannot render
before the browser exists.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The question decides the mark

| The question | The mark | What forces the change |
| --- | --- | --- |
| Which category is larger? | Bars, with the baseline at zero | A truncated baseline misstates the ratio between two bars. |
| How did the value change over time? | A line, or an area for one series | — |
| What are the parts of a whole, and there are five or fewer? | A stacked bar, or one pie or donut | A pie of more than five slices is unreadable. |
| What are the parts of a whole, and there are more than five? | A bar chart, or a treemap. NEVER a pie | The reader cannot compare the angles. Where the product needs a pie, group the small parts into one "Other" slice. |
| How do two variables relate? | A scatter plot | — |

### The chart is a Client Component

```tsx
// Wrong: the chart library sits in a file with no directive.
// Failure: ResponsiveContainer measures a box of 0 by 0 on the server, so the
// first paint holds an empty chart and React reports a hydration mismatch. A
// library that paints on a canvas fails earlier, because the server has no
// canvas.
import { LineChart } from "recharts";

export default async function DashboardPage() {
  const points = await getWeeklyUsers();
  return <LineChart data={points} />;
}
```

```tsx
// Correct: the chart is an island, and the page reserves its box.
// chart.tsx
"use client"; // reason: the chart measures its container in the browser
import { Line, LineChart, ResponsiveContainer, XAxis, YAxis } from "recharts";

export function UsersTrend({ points }: { points: Point[] }) {
  return (
    <ResponsiveContainer width="100%" aspect={16 / 9}>
      <LineChart data={points}>
        <XAxis dataKey="week" />
        <YAxis />
        <Line dataKey="users" />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

```tsx
// Correct: the server page loads the island, and the skeleton holds the same
// aspect ratio, so nothing moves when the chart arrives.
import dynamic from "next/dynamic";

const UsersTrend = dynamic(() => import("./chart").then((m) => m.UsersTrend), {
  loading: () => <div className="aspect-video w-full animate-pulse rounded-lg bg-muted" />,
});
```

A library that paints on an SVG, such as recharts or `@visx`, can render on the
server. A library that paints on a canvas, such as ECharts or Chart.js, cannot.
Give a canvas library `ssr: false`, and give every chart a skeleton of the final
aspect ratio.

The `ssr: false` option is valid only inside a Client Component. Put that
`next/dynamic` call in the client island, and never in a Server Component. The
call above holds no `ssr: false`, so the server page can make it.

`references/server-and-client-components.md` owns the boundary and the
hydration failure. `references/suspense-and-actions.md` owns the shape of a
fallback.

### Every chart carries a text alternative

```tsx
// Wrong: the chart is the whole content.
// Failure: a screen reader reads an empty region, or it reads the axis tick
// labels as loose numbers with no relation. The reader receives no trend and no
// total, and the picture holds the only copy of the data.
<div className="h-64">
  <UsersTrend points={points} />
</div>
```

```tsx
// Correct: the figure names the chart, the image role states the trend, and a
// table holds the values.
<figure>
  <figcaption>Weekly active users, last 12 weeks</figcaption>
  <div role="img" aria-label="A line that rises from 1,200 users to 4,800 users over 12 weeks">
    <UsersTrend points={points} />
  </div>
  <table className="sr-only">
    <caption>Weekly active users</caption>
    {/* one row for each week */}
  </table>
</figure>
```

The `aria-label` states what the chart shows, and never that a chart is present.
"A line chart" is not a description. The direction, the range, and the period
are the description.

The alternative is one of three things. It is a data table beside the chart, a
visually hidden data table, or a control that downloads the values.
`references/cell-formatting-and-export.md` owns that download.

`references/semantics-and-accessible-names.md` owns `role="img"`, the accessible
name, and the `sr-only` class. That domain holds a veto.

### Color is never the only channel

Give every series a second channel. That channel is a shape on a line, or a
pattern inside an area. It is also a direct label at the end of a series, or a
distinct dash. A legend that maps a name to a color alone fails criterion 1.4.1,
which is a Level A criterion.

Take the series colors from the theme tokens `--chart-1` and the tokens after
it, and never from a hex value in the component.
`references/design-tokens-and-theming.md` owns the tokens, and it states that
every color token needs `:root`, `.dark`, and `@theme inline`.

A line, a border, and a legend swatch are non-text content, so each one needs a
contrast of at least 3:1 against its background. That is criterion 1.4.11.
`references/visual-and-motor-criteria.md` owns both criteria, and that domain
holds a veto.

Test it with a grayscale screenshot. Apply `filter: grayscale(1)`, and read the
chart. Every series must stay separable.

### The library follows the render and the data volume

| The condition | The library | What forces the change |
| --- | --- | --- |
| A standard chart in a React product | recharts | It is SVG, it renders on the server, and its API is React-first. |
| A bespoke chart that needs full control of the marks | `@visx/*` | You compose the primitives yourself, and you accept that cost. |
| Tens of thousands of points, or deep feature needs | ECharts, with `echarts-for-react` | It paints on a canvas, so it needs `ssr: false`. |
| Many points, and a small bundle matters | Chart.js v4, with `react-chartjs-2` | Version 4 is tree-shakable. It also paints on a canvas. |

ECharts turns its accessibility features off by default. Turn them on:

```ts
// Correct: the aria component is not in the default build, so import it.
import * as echarts from "echarts/core";
import { AriaComponent } from "echarts/components";

echarts.use([AriaComponent]);

const option = {
  aria: { enabled: true, decal: { show: true } }, // decal is the second channel
};
```

`decal` adds a pattern to each series, which satisfies the second-channel rule.
It does not make the data points reachable by a keyboard. No `aria` option in
ECharts does that today. The data table alternative is therefore the accessible
answer for an ECharts chart, and not an option beside it.

### More points than pixels

A chart that receives more points than the container has pixels draws each mark
on top of the last one. It also costs main-thread time for no gain. Reduce the
set before it reaches the library. Aggregate on the server where the backend can
group by a period, because that also cuts the payload. Where the client must
reduce the set, the common algorithm is Largest Triangle Three Buckets. It keeps
the visible peaks that a plain sample drops.

The aggregate query behind a grouped series belongs to the sibling skill
`django-performance-optimizer`.

### The numbers on the axis

An axis label is a formatted number, so it takes the same rule as a cell. State
the locale, and state the time zone for a date.
`references/cell-formatting-and-export.md` owns the formatter, and it states
what a bare `toLocaleString()` costs.

A compact axis label takes `notation: "compact"`, so 4800 reads as "4.8K".

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry supplied those
facts on 16 August 2026. A cell that holds no date is a package with a current
registry entry on that date. This file does not state an exact release date for
it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `recharts` 3.10.1 | The default chart library on this stack. SVG, so it renders on the server. The 3.x line supports React 19. | 3.10.1 | 2026 | Active. About 54,869,498 downloads each week | None |
| Conditional | `@visx/xychart` 4.0.0 | Only for a bespoke chart that needs control of every mark. You compose the primitives. | 4.0.0 | Current | Active | None |
| Conditional | `echarts` with `echarts-for-react` | Only for a large data volume or a feature that recharts lacks. Canvas, so `ssr: false`. It needs `AriaComponent` and `aria.decal`. | Current | Current | Active | None |
| Conditional | `chart.js` 4.5.1 with `react-chartjs-2` | Only where canvas performance over many points matters. Version 4 is tree-shakable. | 4.5.1 | Current | Active. About 1,600,000 downloads each week for the React wrapper | None |
| Conditional | `@nivo/*` | Only in a design-forward dashboard where the bundle cost is acceptable. | Current | Current | Active | None |
| Conditional | `@observablehq/plot` | Only for an exploratory chart, where a grammar of graphics fits the work. | Current | Current | Active | None |
| Reject | A chart library imported into a Server Component | The container measures 0 by 0 on the server, and a canvas library has no server DOM. | — | — | — | — |
| Reject | A chart with no text alternative | It holds no content for a screen reader, and it fails criterion 1.1.1. | — | — | — | — |

### Version discipline

Read the installed version before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| recharts supports React 19 on the 3.x line | A `recharts` 2.x entry in `package.json` beside React 19 | Move to 3.x, and read the release notes for the renamed props |
| A canvas library cannot render on the server | A canvas chart import with no `ssr: false` | Load it through `next/dynamic` with `ssr: false`, and a skeleton of the final aspect ratio |
| ECharts ships `aria` off, and `AriaComponent` outside the default build | An `aria` option with no `echarts.use([AriaComponent])` | Import the component, then set `aria.enabled` and `aria.decal.show` |

## Verification

```bash
# 1. Read the installed version before you write code.
node -p "require('recharts/package.json').version"

# 2. Find a chart import in a file with no client directive. This prints
#    nothing.
rg -l 'from "recharts"|echarts|chart\.js|@visx' -g '*.tsx' src/ | \
  xargs rg --files-without-match '"use client"'

# 3. Find a canvas chart with no ssr:false. Read every hit.
rg -n 'dynamic\(' -g '*.tsx' src/ | rg -i 'chart|echarts'

# 4. Find a chart with no text alternative. Read every hit.
rg -ln 'ResponsiveContainer|ReactECharts|<Chart' -g '*.tsx' src/ | \
  xargs rg --files-without-match 'aria-label|<figure'

# 5. Find a hex color inside a chart component. This prints nothing.
rg -n '#[0-9a-fA-F]{3,8}\b' -g '*chart*.tsx' src/

# 6. Find an ECharts option with no aria component. This prints nothing.
rg -ln 'aria:\s*\{' -g '*.ts*' src/ | \
  xargs rg --files-without-match 'AriaComponent'

# 7. Apply filter: grayscale(1) to the page, and read every chart. Each series
#    stays separable.

# 8. Read each chart with a screen reader. Confirm the caption, the described
#    trend, and the reachable data values.

# 9. Load the route on a slow connection. Confirm that the skeleton holds the
#    final box, and that nothing moves when the chart arrives.
```

## Review checklist

- [ ] Does the mark answer the question that the chart asks?
- [ ] Does every bar chart start its baseline at zero?
- [ ] Does every pie or donut hold five parts or fewer?
- [ ] Does the chart file carry `"use client"`, with the reason above it?
- [ ] Does a canvas library load through `next/dynamic` with `ssr: false`,
      inside a Client Component?
- [ ] Does the skeleton hold the aspect ratio of the final chart?
- [ ] Does the chart sit in a `<figure>` with a `<figcaption>`, or carry
      `role="img"` with an `aria-label`?
- [ ] Does the `aria-label` state the trend, the range, and the period, rather
      than the word "chart"?
- [ ] Does a data table or a download control hold the values?
- [ ] Does every series carry a second channel beside the color?
- [ ] Do the series colors come from the chart tokens rather than from a hex
      value?
- [ ] Does every line, border, and legend swatch reach 3:1 against its
      background?
- [ ] Does the chart stay readable in a grayscale screenshot?
- [ ] Does an ECharts chart import `AriaComponent` and set `aria.decal`?
- [ ] Is the point count reduced before the data reaches the library?
- [ ] Does every axis label state its locale, and its time zone for a date?

## Handoffs

- The table under the chart, its server-driven state, and its markup →
  `references/data-table-and-server-driven-state.md`.
- The `Intl` formatter on an axis, and the download of the values →
  `references/cell-formatting-and-export.md`.
- The `--chart-*` tokens, the dark theme, and the color space →
  `references/design-tokens-and-theming.md`.
- The contrast of a mark and of a legend swatch, and criterion 1.4.1 →
  `references/visual-and-motor-criteria.md`. That domain holds a veto.
- `role="img"`, the accessible name, the `<figcaption>`, and the `sr-only`
  class → `references/semantics-and-accessible-names.md`. That domain holds a
  veto.
- The `"use client"` directive, the island, and the hydration failure →
  `references/server-and-client-components.md`.
- `next/dynamic`, the route file, and the segment that loads the island →
  `references/app-router-structure.md`.
- The `<Suspense>` boundary and the shape of a fallback →
  `references/suspense-and-actions.md`.
- The query that holds the series, and its four states →
  `references/server-state-and-query-cache.md`.
- The classes on the chart container →
  `references/component-styles-and-variants.md`.
- The words in a caption and in a legend →
  `references/interface-copy-and-voice.md`. The copy of an empty chart →
  `references/error-and-empty-state-copy.md`.
- The bundle bytes of a chart library →
  `references/client-bundle-and-third-party-scripts.md`. The main-thread cost
  of a paint → `references/paint-and-interaction-cost.md`.
- The direction of an axis under `dir="rtl"`, and the rule that a time axis
  never mirrors → `references/bidirectional-layout-and-scripts.md`. The locale
  and the calendar behind a label →
  `references/locale-formatting-and-calendars.md`.
- The visual regression test over a chart → domain 20 `testing-and-quality`.
  Not integrated yet.
- The aggregate query behind a grouped series, and its index → the sibling skill
  `django-performance-optimizer`.
