# Cell formatting and export

The `Intl` API of ECMA-402, `papaparse`, SheetJS `xlsx` 0.20.3, Next.js 16.3,
React 19.2.4 or later, against a Django and DRF backend. This file owns the
value that a user reads and the file that a user takes away. The subjects are
the number and the date inside a cell or on an axis, and the locale and the time
zone that they need. They also include the set that an export covers, and the
text that a spreadsheet reads as a formula. The last subject is the point at
which the file belongs to the server.

The table around the cell is `references/data-table-and-server-driven-state.md`.
The chart is `references/charts-and-visual-encoding.md`.

## Principle

A formatted value with no stated locale is a value that the server and the
browser disagree about. The two machines hold two settings, and the user sees
the disagreement.

A time with no stated zone is a different time on each machine that reads it.

An export is a promise about a set. The set is the one that the user filtered,
and it is neither the page on the screen nor the whole table.

A spreadsheet reads some text as a formula. Text that a user typed must never
become one.

A browser holds one dataset in memory. Past that size the file belongs to the
server, and the interface owns the wait rather than the work.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### `Intl` with an explicit locale and an explicit time zone

```tsx
// Wrong: the platform default decides the format.
// Failure: the server runs in UTC with an "en-US" default, and the browser runs
// in Europe/London with "en-GB". The server sends "8/17/2026, 2:30:00 PM" and
// the browser renders "17/08/2026, 15:30:00". React reports a hydration
// mismatch, and one of the two times is wrong for the user.
<td>{new Date(order.createdAt).toLocaleString()}</td>
<td>{order.total.toLocaleString()}</td>
```

```tsx
// Correct: the locale and the zone are stated, and the formatter is built once.
const money = new Intl.NumberFormat("en-GB", {
  style: "currency",
  currency: "GBP",
});
const timestamp = new Intl.DateTimeFormat("en-GB", {
  dateStyle: "medium",
  timeStyle: "short",
  timeZone: "Europe/London",
});

// inside the cell
<td>{timestamp.format(new Date(order.createdAt))}</td>
<td className="text-right tabular-nums">{money.format(order.total)}</td>
```

Build the formatter once at module scope. An `Intl` constructor is expensive,
and a table of 25 rows by 8 columns builds it 200 times for each render
otherwise.

Never store a formatted string. Hold the value, and format it at the moment of
the render. A stored string cannot change its locale, and it cannot be sorted.

The `Intl.NumberFormat` V3 options are available on the baseline runtimes. Take
`notation: "compact"` for a short axis label, and `signDisplay` for a delta
column. Take `roundingMode` and `roundingIncrement` for a money column, and
`trailingZeroDisplay` for a price. The `useGrouping` option takes a string.

Right-align a numeric column, and give it `tabular-nums`, so the digits form a
column that the eye can compare.

The locale itself, the message catalog, and the direction of the document are
domain 19 `internationalization-and-rtl`. Until that domain lands, take the
locale from one constant in the application, and never from the browser. The
server and the browser then always agree.

### The export covers the current query

```ts
// Wrong: the export takes the rows that the table holds.
// Failure: the user filters 4,300 orders down to 812, sorts them, and presses
// Export. The file holds the 25 rows of the current page in that order. The user
// finds this out in the spreadsheet, and not in the application.
const rows = table.getRowModel().rows.map((row) => row.original);
```

```ts
// Correct: the export sends the same filter and the same sort as the view.
const rows = await fetchAllMatching({ ...view, pageSize: 1000 }); // it follows next
```

| The condition | The decision | What forces the change |
| --- | --- | --- |
| About 10,000 rows or fewer, and the client can fetch them | Build the file in the browser, with `papaparse` for a CSV | It avoids a round trip and a job queue. The decision flips when the row count or the column count passes what the browser can hold, or when the export must carry a value that the client never loads. |
| A large set, every matching record, or a set that spans many pages | A job on the backend. The client posts the filter, receives a task id, polls the status, and then downloads the file | The browser cannot hold the whole set. The worker itself belongs to the sibling skill `django-async-jobs`. |

The job path needs four states in the interface, and each one needs a design.
They are queued, in progress with a percentage where the server reports one,
ready with a download link, and failed with a reason. State the expiry of the
link, because a link that dies with no warning reads as a defect.

### The client-side file escapes the formula prefixes

```ts
// Wrong: the values join straight into the file.
// Failure: a user typed an order note that starts with "=HYPERLINK(". Excel and
// LibreOffice Calc read that cell as a formula and run it, so the file sends the
// neighboring cell to another host when the user opens it. This is CSV
// injection.
const line = values.join(",");
```

```ts
// Correct: a value that starts with a formula trigger takes a leading
// apostrophe, which the spreadsheet reads as text.
const FORMULA_PREFIX = /^[=+\-@\t\r]/;

export function escapeCell(value: unknown): string {
  const text = String(value ?? "");
  return FORMULA_PREFIX.test(text) ? `'${text}` : text;
}
```

The triggers are `=`, `+`, `-`, `@`, the tab character `0x09`, and the carriage
return `0x0D`. OWASP names this attack CSV Injection.

Prefix the file with `\uFEFF`, which is the UTF-8 byte order mark. Without it
Excel reads the bytes in the local code page, and every accented character and
every non-Latin character is wrong.

```ts
// Correct: the mark goes in front of the content, and the type states UTF-8.
const blob = new Blob([`\uFEFF${csv}`], { type: "text/csv;charset=utf-8" });
```

A file that the server builds is the server's responsibility for the same
escape, and the sibling skill `secure-code-auditor` owns that check. This file
owns the escape for a file that the browser builds.

`references/api-client-and-request-safety.md` owns the request that fetches the
rows, its timeout, and its abort signal.

### XLSX needs a decision before it needs code

SheetJS no longer publishes to npm. The npm package `xlsx` is frozen at 0.18.5,
and that version carries two open advisories. They are CVE-2023-30533, a
prototype pollution defect with a CVSS score of 7.8, which the vendor fixed in
0.19.3. The second is CVE-2024-22363, a regular expression denial of service
with a score of 7.5, which the vendor fixed in 0.20.2.

Take one of two paths. Install the vendor tarball from `cdn.sheetjs.com`, which
ships 0.20.3. Or generate the workbook on the backend, which also removes the
library from the bundle. NEVER take `xlsx` from npm.

Ask first whether the product needs XLSX at all. A CSV opens in every
spreadsheet, it carries no library, and it holds no formula.

### The download itself

The browser download, the object URL and its release, and the
`Content-Disposition` header are domain 13 `media-and-file-handling`. This file
owns what goes into the file. That domain owns how the file reaches the disk.

Give the download control a state while the file is built, so a large export
does not read as a dead button. Announce the finish through the polite region
that `references/keyboard-focus-and-live-regions.md` owns.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those facts on 16 August 2026. A cell that holds no
date is a package with a current registry entry on that date. This file does not
state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `Intl.NumberFormat` and `Intl.DateTimeFormat` | A platform API, and no package. Reach for it before any formatting library. | Platform | — | Platform | None |
| Recommend | `papaparse` | The CSV writer and reader in the browser. | Current | Current | Active | None |
| Conditional | SheetJS `xlsx` 0.20.3 | Only where the product needs XLSX, and only from the `cdn.sheetjs.com` tarball. | 0.20.3 | Current | Active, outside npm | None |
| Audit-only | `xlsx` 0.18.5 from npm | Frozen, and behind two fixed advisories. Move to the vendor tarball, or move the work to the backend. | 0.18.5 | Frozen | No further npm release | CVE-2023-30533 and CVE-2024-22363 |
| Reject | A stored formatted string in place of a value | It cannot change its locale, and it cannot be sorted or summed. | — | — | — | — |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| The `Intl.NumberFormat` V3 options reached the baseline | A hand-written compact formatter, or a rounding helper | Take `notation`, `signDisplay`, `roundingMode`, `roundingIncrement`, and `trailingZeroDisplay` |
| SheetJS left npm, and npm froze `xlsx` at 0.18.5 | An `"xlsx": "^0.18.5"` entry in `package.json` | Install the `cdn.sheetjs.com` tarball at 0.20.3, or generate the workbook on the backend |

## Verification

```bash
# 1. Read the installed versions before you write code.
rg -n '"xlsx"|"papaparse"' package.json

# 2. Find a format call with no locale. This prints nothing.
rg -n 'toLocaleString\(\)|toLocaleDateString\(\)|toLocaleTimeString\(\)' src/

# 3. Find a date format with no time zone. Read every hit.
rg -n 'Intl\.DateTimeFormat' src/ | rg -L 'timeZone'

# 4. Find an Intl constructor inside a render. Read every hit.
rg -n 'new Intl\.' -g '*.tsx' src/

# 5. Find an export that builds from the row model of the table. Read every
#    hit.
rg -n 'getRowModel\(\)|getFilteredRowModel\(\)' -g '*.ts*' src/ | rg -i 'export|csv|download'

# 6. Find a CSV writer with no escape. Each hit needs the formula guard.
rg -n 'unparse\(|join\(","\)' -g '*.ts*' src/

# 7. Find the frozen npm package. This prints nothing.
rg -n '"xlsx":\s*"\^?0\.18' package.json

# 8. Build an export, then read the file. An unescaped trigger prints a
#    failure.
grep -aP '^[=+\-@]' export.csv && echo "FAIL: an unescaped formula prefix" || echo "OK"

# 9. Confirm the byte order mark on the same file.
head -c 3 export.csv | xxd | grep -q "efbb bf" && echo "the BOM is present"

# 10. Filter and sort the table, then export. Open the file, and confirm that
#     the row count matches the filtered count and that the order matches.

# 11. Render a table of dates on the server and in the browser. Read the
#     console for a hydration error.
```

## Review checklist

- [ ] Does every number and every date pass through `Intl` with an explicit
      locale?
- [ ] Does every date format state a `timeZone`?
- [ ] Is each formatter built once, outside the render?
- [ ] Does the code hold the value rather than a formatted string?
- [ ] Is every numeric column right-aligned, with tabular figures?
- [ ] Does the export cover the current filter and the current sort, rather than
      the current page?
- [ ] Does the export take the backend job path above what the browser can hold?
- [ ] Does the job path render a queued, an in-progress, a ready, and a failed
      state?
- [ ] Does the interface state the expiry of a download link?
- [ ] Does every client-built cell escape a leading `=`, `+`, `-`, `@`, tab, or
      carriage return?
- [ ] Does every client-built CSV carry the UTF-8 byte order mark?
- [ ] Is `xlsx` absent from `package.json`, or present only as the vendor
      tarball?
- [ ] Does the download control hold a state while the file is built, and does
      it announce the finish?

## Handoffs

- The table that holds the cells, and the filter that the export repeats →
  `references/data-table-and-server-driven-state.md`.
- The axis label that takes the same formatter →
  `references/charts-and-visual-encoding.md`.
- The request that fetches the rows, its timeout, and its abort signal →
  `references/api-client-and-request-safety.md`.
- The query that holds the job status, and the stop condition on the poll →
  `references/server-state-and-query-cache.md`.
- The announcement when a file is ready →
  `references/keyboard-focus-and-live-regions.md`. That domain holds a veto.
- The name of the download control, and the name of a status region →
  `references/semantics-and-accessible-names.md`. That domain holds a veto.
- The lockfile entry, the cooldown, and the justification for a new dependency →
  `references/dependencies-and-git-workflow.md`.
- The object URL, its release, and the `Content-Disposition` header → domain 13
  `media-and-file-handling`. Not integrated yet.
- The locale that the application chooses, the currency for each locale, and the
  direction of the document → domain 19 `internationalization-and-rtl`. Not
  integrated yet.
- The words in a download message and in an expiry warning → domain 15
  `ux-writing-and-content-design`. Not integrated yet.
- The test that opens an export and reads its rows → domain 20
  `testing-and-quality`. Not integrated yet.
- The worker that builds a large export, its retries, and its idempotency → the
  sibling skill `django-async-jobs`.
- The permission check on an export endpoint, and the escape inside a file that
  the server builds → the sibling skill `secure-code-auditor`.
