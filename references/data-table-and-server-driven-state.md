# Data table and server-driven state

`@tanstack/react-table` 9.1.2 and 8.21.3, `@tanstack/react-virtual` 3.14.9,
TanStack Query 5.101 or later, nuqs 2.9, Next.js 16.3, React 19.2.6 or later,
against a Django and DRF backend. This file owns a dense row surface. The
subjects are the column model, the row model, and the identity of a row. They
also include the server that drives the page, the sort, and the filter, and the
markup that a screen reader reads. The last subjects are the scroll container
for a long list, the selection across pages, and the edit inside a cell.

The chart is `references/charts-and-visual-encoding.md`. The value inside a cell
and the file that leaves the browser are
`references/cell-formatting-and-export.md`. The query key, the cache entry, and
the four states of a data view are `references/server-state-and-query-cache.md`.

## Principle

A table that sorts one page lies. The user reads a sort as a sort of the whole
set, and the interface must mean what the user reads.

A row has an identity, and a position in an array is not one. A sort changes
every position, and it changes no identity.

A view that a link cannot reproduce is not a view. The address bar is the only
place that survives a reload, a share, and the back button.

A table is a table. The browser and the screen reader already hold that
contract. A role that replaces it takes on the whole contract, and every key
inside it.

The DOM node is the cost. Below the count at which that cost appears, the
virtualiser takes more than it returns.

An export is a promise about a set. A selection is a promise about a set. Both
promises name the set that the user filtered.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### Read the installed major before you write a column

TanStack Table ships two lines. Version 9.1.2 became stable on 4 August 2026,
and it is the `latest` tag. Version 8.21.3 is the line that most deployed
applications run today. The shadcn/ui Data Table targets version 9.

Read the installed major from `package.json` first. NEVER mix the two APIs in
one file.

| The line | The row models | The render helper |
| --- | --- | --- |
| 8.21.3 | Import each row model, and pass it as an option | `flexRender(...)` |
| 9.1.2 | Declare the features through `tableFeatures()`, and the row models follow the features | `<table.FlexRender/>` |

Every option in this file exists on both lines. That set is `data`, `columns`,
`getRowId`, `manualPagination`, `manualSorting`, `manualFiltering`, `rowCount`,
`pageCount`, `state`, and each `on*Change` handler. Only the row-model
declaration differs, and each example below marks that line.

The shadcn/ui base primitive moved from Radix to Base UI on 3 July 2026. The
Data Table is a composition over a plain HTML `<table>`, so that move changes no
table primitive. `references/component-composition.md` owns the primitive
libraries.

### The column definitions are stable

```tsx
// Wrong: a new array and new cell closures on every render.
// Failure: the whole table renders again on each keystroke in an unrelated
// field. The virtualiser measures every row a second time, and a checkbox in a
// selected row flickers. The date cell also has no locale and no time zone, so
// the server and the browser print two different strings.
function OrdersTable({ data }: { data: Order[] }) {
  const columns = [
    { accessorKey: "reference", header: "Reference" },
    {
      accessorKey: "createdAt",
      header: "Created",
      cell: (c) => new Date(c.getValue<string>()).toLocaleString(),
    },
  ];
  const table = useReactTable({ data, columns, getCoreRowModel: getCoreRowModel() });
}
```

```tsx
// Correct: the definitions sit at module scope, so they are stable by
// construction. The formatter is built once, and not once for each cell.
import { createColumnHelper } from "@tanstack/react-table";

const created = new Intl.DateTimeFormat("en-GB", {
  dateStyle: "medium",
  timeZone: "UTC",
});
const column = createColumnHelper<Order>();

export const orderColumns = [
  column.accessor("reference", { header: "Reference" }),
  column.accessor("createdAt", {
    header: "Created",
    cell: (context) => created.format(new Date(context.getValue())),
  }),
];
```

React Compiler 1.0 memoises a value that a component body creates. A column
array declared in a render is therefore memoised where the compiler is on.
Module scope is stable whether the compiler is on or off, so module scope is
the default.
`references/state-and-effects.md` owns the compiler and the bailout.

`references/cell-formatting-and-export.md` owns the formatter and the reason
that a bare `toLocaleString()` breaks hydration.

### `getRowId` returns the domain identifier

```tsx
// Wrong: the row id defaults to the index of the array.
// Failure: the user selects row 0 and sorts the column. The selection state is
// still { "0": true }, so the checkbox now marks a different order. A bulk
// action then acts on records that the user never chose.
useReactTable({ data, columns, enableRowSelection: true });
```

```tsx
// Correct: the identity comes from the record.
useReactTable({
  data,
  columns,
  enableRowSelection: true,
  getRowId: (row) => row.id,
});
```

The same identifier keys the optimistic write of an inline edit, and it keys the
merge of a pushed row. `references/live-events-and-cache-merge.md` owns that
merge.

### The server drives the page, the sort, and the filter

| The condition | The decision | What forces the change |
| --- | --- | --- |
| Fewer than about 200 rows, and the set does not change between requests | A plain semantic `<table>`, with the sort and the filter in the browser | A round trip costs more than it saves at this scale. The decision flips when the row count or the filter cardinality passes a few hundred, or when the data changes between two requests. |
| More than a few hundred rows, or live data | `manualPagination`, `manualSorting`, and `manualFiltering` together, over the DRF query parameters | A sort of one page states an order that the set does not hold. The cost is one request for each change of state. |

Set the three `manual*` options together, or set none of them. NEVER set one
alone.

```tsx
// Wrong: the page comes from the server, and the sort does not.
// Failure: the user sorts by total on page 1 of 20. The browser reorders the 25
// rows that it holds, and the header shows a sort of the whole set. Page 2 then
// arrives in the order of the server, and the two pages contradict each other.
useReactTable({
  data: page.results,
  columns,
  getCoreRowModel: getCoreRowModel(), // 8.x; 9.x declares this through tableFeatures()
  manualPagination: true,
  state: { pagination, sorting },
});
```

```tsx
// Correct: every operation happens where the whole set is.
const table = useReactTable({
  data: page.results,
  columns: orderColumns,
  getCoreRowModel: getCoreRowModel(), // 8.x; 9.x declares this through tableFeatures()
  manualPagination: true,
  manualSorting: true,
  manualFiltering: true,
  rowCount: page.count, // the envelope of PageNumberPagination or LimitOffsetPagination
  state: { pagination, sorting, columnFilters },
  onPaginationChange: setPagination,
  onSortingChange: setSorting,
  onColumnFiltersChange: setColumnFilters,
  getRowId: (row) => row.id,
});
```

A manual table cannot derive the page count from the length of `data`. Supply
`rowCount`, which is the preferred option because the table derives `pageCount`
from it. Supply `pageCount` where the endpoint gives no total. With neither one,
the page count resolves to `-1` and the control that jumps to the last page does
nothing.

### The table state maps onto the DRF query parameters

| The table state | The DRF mechanism | The query parameter |
| --- | --- | --- |
| The page number | `PageNumberPagination` | `?page=3&page_size=25`, and `page_size` only where `page_size_query_param` is set |
| The offset window | `LimitOffsetPagination` | `?limit=25&offset=50` |
| The cursor | `CursorPagination` | `?cursor=<opaque>`, which the client never reads and never builds |
| The sort | `OrderingFilter` | `?ordering=-created_at`, where `-` means descending and a comma separates two fields |
| The search term | `SearchFilter` | `?search=foo`, unless the backend overrides `SEARCH_PARAM` |
| A column filter | `django-filter` `FilterSet` | `?status=active&price__gte=10` |

The prefix on an entry in `search_fields` decides what the search box finds. It
is `^` for a match at the start, `=` for an exact match, `@` for a full-text
match, and `$` for a regular expression. A field with no prefix matches anywhere
inside the value. Read the prefix before you write the placeholder text of the
search box, because the box makes a promise about the match.

The sibling skill `django-api-contract` owns which prefix and which lookup each
field takes, and `references/openapi-schema-and-codegen.md` owns the generated
types that carry them.

### The pagination class decides which control the table may offer

| The class | The envelope | What the interface may offer |
| --- | --- | --- |
| `PageNumberPagination` | `{count, next, previous, results}` | Numbered pages, a total, and a jump to the last page |
| `LimitOffsetPagination` | `{count, next, previous, results}` | The same set of controls |
| `CursorPagination` | `{next, previous, results}`, with no `count` | Next and previous alone, or an infinite scroll. No total, and no jump to a page |

Set `rowCount` from `count`. Under `CursorPagination` there is no `count`, so
`rowCount` stays undefined and the total is unknowable. Derive
`hasNext = next !== null` instead, and render no numbered control.

Read the next page from the `next` address, and never compute an offset.
`references/server-state-and-query-cache.md` holds that rule and the infinite
query behind it. `references/api-client-and-request-safety.md` holds the same
rule for a plain request.

A renamed parameter, a renamed envelope key, or a dropped `count` breaks the
pagination control in silence. The contract test that catches it is domain 20
`testing-and-quality`.

### The view lives in the URL

```tsx
// Wrong: the page, the sort, and the filters live in component state.
// Failure: the user filters, sorts, and sends the address to a colleague. The
// colleague opens an unfiltered first page. The back button also returns the
// same view rather than the previous one.
const [pagination, setPagination] = useState({ pageIndex: 0, pageSize: 25 });
```

```tsx
// Correct: the address carries the view, so a link and the back button work.
"use client"; // reason: the control reads and writes the address bar
import { parseAsInteger, parseAsString, useQueryStates } from "nuqs";

const [view, setView] = useQueryStates({
  page: parseAsInteger.withDefault(1),
  ordering: parseAsString.withDefault("-created_at"),
  search: parseAsString.withDefault(""),
});
```

`references/client-and-url-state.md` owns the parsers and the write rate. Give
the search box `limitUrlUpdates: debounce(300)`, so one word produces one
history entry and one request rather than one for each letter. TanStack Query
cancels the superseded request, because the key changes with the term.

Derive the query key from the same values, so the address and the cache never
disagree. `references/server-state-and-query-cache.md` owns the key factory.

Set `placeholderData: keepPreviousData` on the query. The previous rows stay on
the screen while the next page arrives, so the table keeps its height and the
layout does not shift.

### The markup is a semantic `<table>`

```tsx
// Wrong: a role that the keyboard model does not support.
// Failure: role="grid" tells a screen reader that arrow keys move a cursor
// between cells. No arrow key handler exists, so the user hears a promise that
// the table does not keep. This is worse than no role at all.
<table role="grid">
```

```tsx
// Correct: the native table, with one sortable header.
<table>
  <caption>Orders, newest first</caption>
  <thead>
    <tr>
      <th scope="col" aria-sort={sort === "reference" ? "ascending" : "none"}>
        <button type="button" onClick={() => setSort("reference")}>
          Reference
        </button>
      </th>
    </tr>
  </thead>
  <tbody>{/* rows */}</tbody>
</table>
```

Use `role="grid"` or `role="treegrid"` only where the product needs a
spreadsheet cursor and an edit in place. Then implement every key of the ARIA
APG grid pattern, which is the arrow keys, `Home`, `End`, `Page Up`, and
`Page Down`. `references/keyboard-focus-and-live-regions.md` owns the APG rule,
and that domain holds a veto.

Carry `aria-sort` on one header at a time. Put the control inside the `<th>` as
a `<button>`, so a keyboard reaches it and a screen reader names it.
`references/semantics-and-accessible-names.md` owns the accessible name.

Announce the result count after a sort or a filter, through the polite region
that `references/keyboard-focus-and-live-regions.md` owns. A table that changes
under a screen reader with no announcement reports nothing.

A sticky header that covers the focused row fails criterion 2.4.11. Set
`scroll-padding-top` on the scroll container.
`references/keyboard-focus-and-live-regions.md` holds that technique.

A table that scrolls sideways on a phone with no alternative is not usable
there. Render one card for each row at the narrow width. Where the table must
stay a table, give the scroll container a name and a `tabindex` of `0`, so a
keyboard reaches it.

### The long list

| The condition | The decision | What forces the change |
| --- | --- | --- |
| More than about 200 rows are visible in one scroll container | `useVirtualizer` from `@tanstack/react-virtual` | The DOM node count drives the layout cost and the paint cost. |
| Fewer than about 200 rows | A plain `<table>` | Below this count the virtualiser adds absolute positions and breaks find-in-page for no gain. |
| A long list of simple blocks, such as cards, and not one scroll container with a sticky header | `content-visibility: auto` with `contain-intrinsic-size` | It keeps the DOM, the accessibility tree, and find-in-page. It reached Baseline on 15 September 2025. The decision flips to the virtualiser where the rows need one scroll container, or where there are thousands of them. |

```tsx
// Wrong: the virtualiser over 30 rows.
// Failure: find-in-page misses the rows that are off screen, the row heights
// need a measurement pass, and the code carries absolute positions. The 30 rows
// cost nothing in the first place.
const rows = useVirtualizer({ count: 30, estimateSize: () => 48, getScrollElement });
```

```tsx
// Correct: the virtualiser over a large set, with the absolute row index
// declared for the screen reader.
<table aria-rowcount={page.count}>
  <tbody>
    {virtualRows.map((virtualRow) => (
      <tr key={rows[virtualRow.index].id} aria-rowindex={offset + virtualRow.index + 1}>
        {/* cells */}
      </tr>
    ))}
  </tbody>
</table>
```

Set `aria-rowcount` on the table to the total of the whole set, and
`aria-rowindex` on each row to its absolute position. Without them a screen
reader reads "row 20 of 20" over a set of ten thousand.

Dynamic row heights need `display: grid` on the table and `display: flex` on the
row. That CSS removes the native table roles, and the accessibility tree then
shows generic groups. Declare `role="table"`, `role="row"`,
`role="columnheader"`, and `role="cell"` again on those elements, and prove the
result with an axe assertion.
`references/wcag-conformance-and-verification.md` owns the assertion.

The virtualiser keeps the off-screen rows out of the DOM, so find-in-page cannot
reach them. That is a real cost, and no code removes it. Give the user a search
field that queries the server instead, and state the trade in the pull request.

### One selection, two meanings

A checkbox in the header selects the rows that are loaded. A set that spans
pages needs a second, separate control that selects every record that the
current filter matches.

```tsx
// Wrong: one checkbox, and a bulk action over the ids that the client holds.
// Failure: the filter matches 4,300 records and the page holds 25. The user
// selects all and archives, and 25 records change. The interface reports
// success, and 4,275 records are untouched.
await archiveOrders({ ids: table.getSelectedRowModel().rows.map((r) => r.id) });
```

```tsx
// Correct: the whole-set action sends the filter, and never a list of ids.
if (scope === "all-matching") {
  await archiveOrders({ filter: view }); // the server resolves the predicate
} else {
  await archiveOrders({ ids: selectedIds });
}
```

The server must accept a filter predicate for the whole-set action. Where it
accepts only a list of identifiers, offer no whole-set control. The bulk
endpoint itself belongs to the sibling skill `django-performance-optimizer` for
its query cost, and to `secure-code-auditor` for its permission check.

State the count and the scope in the confirmation. "Archive 4,300 orders that
match this filter" and "Archive 25 orders on this page" are two different
sentences.

### The edit inside a cell

Key the optimistic write by the same value that `getRowId` returns. Hold the
pending state and the error state per row, so one failure marks one row rather
than the whole table. `Escape` returns the cell to the committed value.

A 409 means that another writer changed the record first. Offer a reload or an
overwrite, and never resolve it in silence.
`references/server-state-and-query-cache.md` owns the snapshot and the rollback,
and `references/form-submission-and-server-errors.md` owns the map from a DRF
400 onto a control.

### A row that arrives while the user reads

A pushed row must not reorder the rows under the cursor. Add the row at the edge
of the current order, or hold it behind a "3 new orders" control that the user
presses. `references/live-events-and-cache-merge.md` owns the write into the
cache, and `references/push-transport-and-connection.md` owns the connection.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry supplied those
facts on 16 August 2026. A cell that holds no date is a package with a current
registry entry on that date. This file does not state an exact release date for
it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `@tanstack/react-table` 9.1.2 | The line for new work. Features are opt-in through `tableFeatures()`, so the bundle carries what the table declares. | 9.1.2 | 4 August 2026 | Active. 3,150 dependent projects | None |
| Recommend | `@tanstack/react-table` 8.21.3 | The line that most deployed applications run. Keep it, and migrate on its own change. | 8.21.3 | 14 April 2025 | Active. About 14,002,470 downloads each week | None |
| Recommend | `@tanstack/react-virtual` 3.14.9 | The scroll container over a large set. No platform feature replaces it. | 3.14.9 | 2026 | Active. About 17,737,477 downloads each week | None |
| Recommend | `content-visibility` | A platform property, and no package. Cheaper than the virtualiser for a list of simple blocks. | Baseline | 15 September 2025 | Platform | None |
| Conditional | AG Grid Enterprise | Only where the product needs a pivot, a grouped aggregate, or parity with a spreadsheet. TanStack Table has no pivot. It carries a per-developer licence and a large bundle. | Current | Current | Active | None |
| Conditional | MUI X Data Grid Pro or Premium | Only in a Material product. Pro adds column pinning and the virtualiser, and Premium adds the row group, the aggregate, and the Excel export. Both are commercial. | Current | Current | Active | None |
| Audit-only | `react-table` v7 | The predecessor of TanStack Table, and unmaintained. Migrate it. | 7.x | — | Unmaintained | None |
| Reject | A fetch of the whole set, and a page in the browser | It sends every row over the network, it holds every row in memory, and it makes the first interaction slow. | — | — | — | — |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| TanStack Table 9.1.2 became stable on 4 August 2026 | An unconditional import of a whole row model, such as `getSortedRowModel` | Declare the features through `tableFeatures()`, and render through `<table.FlexRender/>` |
| The shadcn/ui base moved from Radix to Base UI on 3 July 2026 | The `base` key in `components.json` | Nothing for the table. The Data Table is a plain `<table>` composition, and it holds no Radix primitive |
| React Compiler 1.0 memoises a value from a component body | A hand-written `useMemo` over a column array, with no measurement beside it | The `useMemo` is optional, and it is not forbidden. Module scope needs neither |
| `content-visibility` reached Baseline on 15 September 2025 | An `@supports` gate around it | Remove the gate for an evergreen target. The find-in-page defect in Safari is still open |
| nuqs replaced `throttleMs` with `limitUrlUpdates` | `throttleMs` in any nuqs call | Write `limitUrlUpdates: throttle(ms)` or `limitUrlUpdates: debounce(ms)` |

## Verification

```bash
# 1. Read the installed versions before you write code.
node -p "require('@tanstack/react-table/package.json').version"
node -p "require('@tanstack/react-virtual/package.json').version"

# 2. Find a table that sets one manual option and not the other two.
rg -ln 'manualPagination' -g '*.tsx' src/ | \
  xargs rg --files-without-match 'manualSorting'

# 3. Find a manual table with no total. Each hit needs rowCount or pageCount.
rg -ln 'manualPagination' -g '*.tsx' src/ | \
  xargs rg --files-without-match 'rowCount|pageCount'

# 4. Find a table with no getRowId. Read every hit.
rg -ln 'useReactTable' -g '*.tsx' src/ | \
  xargs rg --files-without-match 'getRowId'

# 5. Find a page, a sort, or a filter in component state. Read every hit.
rg -n 'useState' -g '*.tsx' src/ | rg -i 'page|sort|filter|search'

# 6. Find a role that the keyboard model may not support. Read every hit.
rg -n 'role="grid"|role="treegrid"' -g '*.tsx' src/

# 7. Find a virtualiser over a small set. Read every hit.
rg -n 'useVirtualizer' -g '*.tsx' src/

# 8. Find a virtualised body with no absolute row index. This prints nothing.
rg -ln 'useVirtualizer' -g '*.tsx' src/ | \
  xargs rg --files-without-match 'aria-rowcount'

# 9. Type the table. A column definition that disagrees with the row fails here.
npx tsc --noEmit

# 10. Sort a multi-page table. Confirm that page 2 continues the order of
#     page 1, and that the header shows one aria-sort value.

# 11. Select a row, then sort. Confirm that the checkbox stays on the same
#     record.

# 12. Copy the address after a filter and a sort. Open it in a new tab, and
#     read the same view. Then press the back button, and read the previous
#     view.

# 13. Run the keyboard alone. Reach every control, sort with Enter, select
#     with Space, and run the bulk action.

# 14. Read the table with a screen reader. Confirm the row count, the column
#     names, and the announcement of the result count after a sort.
```

```ts
// Playwright: one sort produces one request, and it reaches the address bar.
test("a sort updates the address and fires one request", async ({ page }) => {
  const requests: string[] = [];
  page.on("request", (r) => r.url().includes("/api/orders") && requests.push(r.url()));
  await page.goto("/orders");
  await page.getByRole("columnheader", { name: "Reference" }).getByRole("button").click();
  await expect(page).toHaveURL(/[?&]ordering=reference/);
  expect(requests.filter((u) => u.includes("ordering=reference"))).toHaveLength(1);
});
```

## Review checklist

- [ ] Does the code match the installed major of TanStack Table, with no idiom
      from the other line?
- [ ] Are the column definitions at module scope, or otherwise stable?
- [ ] Does `getRowId` return the domain identifier rather than the index?
- [ ] Are `manualPagination`, `manualSorting`, and `manualFiltering` all set, or
      all unset?
- [ ] Does a manual table supply `rowCount`, or `pageCount`?
- [ ] Does the interface offer a numbered page control only where the envelope
      carries `count`?
- [ ] Does the next page come from the `next` address rather than a computed
      offset?
- [ ] Do the page, the sort, the filters, and the search term live in the
      address bar?
- [ ] Does the search box limit its write rate, and does the superseded request
      stop?
- [ ] Is `placeholderData: keepPreviousData` set on the paginated query?
- [ ] Is the markup a semantic `<table>`, with `role="grid"` only beside a full
      keyboard model?
- [ ] Does one header at a time carry `aria-sort`, with a `<button>` inside the
      `<th>`?
- [ ] Does the result count announce through a polite region after a sort or a
      filter?
- [ ] Is the virtualiser present only above about 200 visible rows?
- [ ] Does a virtualised table carry `aria-rowcount` and an absolute
      `aria-rowindex`?
- [ ] Does a table that CSS lays out as a grid declare its roles again, with an
      axe assertion over them?
- [ ] Does a bulk action over the whole set send the filter rather than the
      loaded identifiers?
- [ ] Does the confirmation state the count and the scope of the action?
- [ ] Does an inline edit hold its pending state and its error state per row,
      and does it answer a 409?
- [ ] Does a pushed row leave the order under the cursor unchanged?

## Handoffs

- The chart, its library, and its text alternative →
  `references/charts-and-visual-encoding.md`.
- The number and the date inside a cell, and the file that an export produces →
  `references/cell-formatting-and-export.md`.
- The query key, the cache times, the infinite query over DRF pagination, the
  optimistic rollback, and the four states of a data view →
  `references/server-state-and-query-cache.md`.
- The `nuqs` parsers, the write rate, and the rule that the address owns a
  shareable value → `references/client-and-url-state.md`.
- The request, the timeout, the `ApiError`, and the read of the `next` address →
  `references/api-client-and-request-safety.md`.
- The generated types behind a filter parameter and a pagination envelope →
  `references/openapi-schema-and-codegen.md` and
  `references/boundary-validation-and-api-types.md`.
- The APG grid pattern, the announcement of a result count, and
  `scroll-padding-top` under a sticky header →
  `references/keyboard-focus-and-live-regions.md`. That domain holds a veto.
- The `<caption>`, the `scope` attribute, and the accessible name of a header
  control → `references/semantics-and-accessible-names.md`. That domain holds a
  veto.
- The axe assertion that proves the roles of a table →
  `references/wcag-conformance-and-verification.md`. That domain holds a veto.
- The target size of a row control, and the two-directional scroll at 320 CSS
  pixels → `references/visual-and-motor-criteria.md`. That domain holds a veto.
- The classes on a cell, a header, and a row →
  `references/component-styles-and-variants.md`, over the tokens in
  `references/design-tokens-and-theming.md`.
- The shape of the table component and its parts, and the list `key` →
  `references/component-composition.md`.
- The pushed row that a live table receives →
  `references/live-events-and-cache-merge.md`, over the connection in
  `references/push-transport-and-connection.md`.
- The map from a DRF 400 onto a control inside an editable cell →
  `references/form-submission-and-server-errors.md`.
- The words in a column header and in a bulk-action confirmation →
  `references/interface-copy-and-voice.md`. The three cases behind an empty
  table → `references/error-and-empty-state-copy.md`.
- The INP of a sort and of a filter →
  `references/paint-and-interaction-cost.md`. The budget over the row payload
  → `references/performance-budgets-and-measurement.md`.
- The direction of a table under `dir="rtl"`, and the locale that a sort assumes
  → domain 19 `internationalization-and-rtl`. Not integrated yet.
- The contract test over the pagination envelope, and the test that runs a sort
  → domain 20 `testing-and-quality`. Not integrated yet.
- The serializer, the filter field, the pagination class, and any breaking change
  in them → the sibling skill `django-api-contract`.
- The query cost of a filtered list, and the index behind an `ordering` value →
  the sibling skill `django-performance-optimizer`.
- The permission check on a list endpoint and on a bulk endpoint → the sibling
  skill `secure-code-auditor`.
