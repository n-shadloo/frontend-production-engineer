# Served content and downloads

Next.js 16.3, React 19.2.6 or later, the File System Access API, against a
Django and DRF backend. This file owns the bytes that leave the application for
a user. The subjects are the origin that serves a stored file, and the headers
that make it safe. They also include the reason that a user-supplied file never
shares the origin of the application. The last subjects are the download
trigger, the private file behind a session, and the file that the browser builds
by itself.

The path that puts the file into storage is
`references/file-upload-and-transport.md`. The picture and the player that
render a stored file are `references/image-and-video-delivery.md`. What goes
inside an export file is `references/cell-formatting-and-export.md`.

## Principle

A file that a user supplied is a document that another user opens. A browser
runs a document, and it runs it with the rights of the origin that served it.

An origin is the boundary of a session. Content on the origin of the
application reads the cookies of the application.

A browser guesses the type of a response where the server does not state one.
The guess is what an attacker aims at.

A private file needs the same gate as the page that links to it. A URL that
anyone can replay is not a gate.

A download of 500 MB that a process holds in memory is an outage with a delay
on it.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### User content leaves from another origin

```text
# Wrong: the avatar comes back from the origin of the application.
# Failure: a user uploads an SVG that holds a script, and the browser runs that
# script on app.example.com. The script then reads the session cookie and the
# document of the application. This is stored cross-site scripting.
GET app.example.com/media/avatars/42.svg
    Content-Type: image/svg+xml
```

```text
# Correct: another origin, a stated disposition, and no guess about the type.
GET user-content.example.com/media/avatars/42.svg
    Content-Type: image/svg+xml
    Content-Disposition: attachment
    X-Content-Type-Options: nosniff
```

Three rules hold together, and each one alone is not enough.

1. Serve user content from another origin or another subdomain. The cookie of
   the application must not reach it, so the domain of the cookie stays narrow.
   `references/session-and-token-lifecycle.md` owns the cookie attributes.
2. Send `Content-Disposition: attachment`, so the browser saves the file and
   does not render it. An image that the application shows in an `<img>` element
   still works, because that element reads the bytes and does not navigate.
3. Send `X-Content-Type-Options: nosniff`, so the browser takes the stated type
   and guesses nothing.

An SVG and an HTML file are the two formats that carry a script. Convert a
user-supplied SVG to a raster image on the server where the product allows it.
Where it does not, the three rules above are the whole defence.

The origins that the application may load media from belong in the Content
Security Policy, which is domain 17 `frontend-security`. That domain owns the
policy, and this file owns the origin that the policy must name. The sibling
skill `secure-code-auditor` owns the server-side check over the stored file, and
the response headers that Django sends.

### The download of a private file

| The condition | The decision | What forces the change |
| --- | --- | --- |
| The file sits in object storage, and the user may read it | The backend mints a short-lived signed URL, and the browser follows it | No byte passes through Node. The interface owns the expiry, so a stale link needs a new one rather than an error page. The decision flips where the storage is not reachable from the browser. |
| The file sits behind the application, or the session must be checked at the moment of the request | A Route Handler that streams the response with the session attached | The gate stays in the application. The handler must stream. The decision flips at the size where a stream still costs a Node connection for the whole transfer. |
| The bytes are already in the browser | A `Blob`, an object URL, and an `<a download>` element | It needs no request. `references/image-and-video-delivery.md` owns the release of that object URL. |

```ts
// Wrong: the handler reads the whole file into memory first.
// Failure: a 500 MB export takes 500 MB of the Node heap for each concurrent
// download. Four users at once end the process.
const buffer = await upstream.arrayBuffer();
return new Response(buffer);
```

```ts
// Correct: the body passes through as a stream, and the session is checked
// first.
export async function GET(request: Request, context: RouteContext<"/api/files/[id]">) {
  const session = await verifySession();
  if (!session) return new Response(null, { status: 401 });

  const { id } = await context.params;
  const upstream = await fetchFile(id, session);

  return new Response(upstream.body, {
    headers: {
      "Content-Type": "application/pdf",
      "Content-Disposition": `attachment; filename="invoice-${id}.pdf"`,
      "X-Content-Type-Options": "nosniff",
    },
  });
}
```

`references/route-protection-and-permissions.md` owns the rule that the handler
verifies the session inside its own body. A signed URL is a credential, so state
its lifetime and treat an expiry as a state of the interface rather than as an
error.

### The download trigger

```tsx
// Wrong: the download attribute on a cross-origin link. storedFileUrl points at
// the media subdomain.
// Failure: a browser ignores `download` when the href is on another origin, so
// the PDF opens in a tab and the file name is the key of the object.
<a href={storedFileUrl} download="invoice.pdf">
  Download
</a>
```

```tsx
// Correct: the server states the disposition and the name, and the link points
// at a same-origin route that streams it.
<a href={`/api/files/${id}`}>Download the invoice</a>
```

The `download` attribute works for a same-origin URL, for a `blob:` URL, and for
a `data:` URL. It does nothing for another origin. Where the file must come from
another origin, `Content-Disposition: attachment` from that origin does the same
work.

`showSaveFilePicker` of the File System Access API lets the user choose the
folder and the name. Only Chromium browsers carry it, and a cross-origin frame
cannot call it. Feature-detect it, and fall back to the `<a download>` path.

```ts
// Correct: the picker where the browser has it, and the anchor everywhere else.
export async function saveFile(blob: Blob, suggestedName: string): Promise<void> {
  if ("showSaveFilePicker" in window) {
    const handle = await window.showSaveFilePicker({ suggestedName });
    const stream = await handle.createWritable();
    await stream.write(blob);
    await stream.close();
    return;
  }

  const url = URL.createObjectURL(blob);
  const anchor = document.createElement("a");
  anchor.href = url;
  anchor.download = suggestedName;
  anchor.click();
  URL.revokeObjectURL(url);
}
```

Give the control a pending state while the file is built or fetched, and
announce the finish through the polite region that
`references/keyboard-focus-and-live-regions.md` owns. A control that does
nothing for eight seconds reads as a broken control.

### The file that the browser builds

A CSV, a PDF, or a ZIP that the browser builds costs no round trip, and it
blocks the main thread while it runs. Keep the work in the browser below the
size that the tab can hold. Above it, ask the backend for the file, which
`references/cell-formatting-and-export.md` states as the job path with its four
states.

Move any build that takes more than a moment into a Web Worker. A frozen tab
during an export is an interaction cost, and domain 16
`performance-and-web-vitals` owns the budget for it.

`references/cell-formatting-and-export.md` owns what goes inside the file, which
is the escape over a formula prefix and the byte order mark. This file owns how
the file reaches the disk.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those facts on 16 August 2026. A cell that holds no
date is a package with a current registry entry on that date. This file does not
state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | The File System Access API | A platform API, and no package. Feature-detect it, and fall back to `<a download>`. | Platform | — | Platform | None |
| Recommend | `Blob` with `URL.createObjectURL` | A platform API, and no package. Release the URL after the click. | Platform | — | Platform | None |
| Reject | A handler that reads a file into memory before it answers | It costs the process the size of the file for each concurrent download. | — | — | — | — |
| Reject | A `download` attribute as the only mechanism for a cross-origin file | A browser ignores it there. The header from that origin does the work. | — | — | — | — |

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| The File System Access API stays a Chromium feature, and Safari and Firefox carry no picker method | A `showSaveFilePicker` call with no feature detection | Detect the method, and fall back to the `<a download>` path |

## Verification

```bash
# 1. Find a handler that buffers a whole file. This prints nothing.
rg -n 'arrayBuffer\(\)|\.buffer\b' -g 'src/app/**/route.ts' src/

# 2. Find a download attribute on a cross-origin href. Read every hit.
rg -n -B2 'download=' -g '*.tsx' src/

# 3. Find a picker call with no feature detection.
rg --files-with-matches 'showSaveFilePicker' -g '*.ts*' src/ \
  | xargs rg --files-without-match "'showSaveFilePicker' in window"

# 4. Find a media URL on the origin of the application. Read every hit, and
#    confirm that no user-supplied file is served from it.
rg -n 'NEXT_PUBLIC_MEDIA|/media/' -g '*.ts*' src/

# 5. Fetch a stored user file in a fresh session, and read its headers. The
#    origin differs from the application, Content-Disposition is attachment,
#    and X-Content-Type-Options is nosniff.
curl -sSI "$MEDIA_ORIGIN/media/avatars/42.svg"

# 6. Upload an SVG that holds a script, then open its served URL. The browser
#    downloads the file, and it runs no script.

# 7. Start a download of a large file, and watch the memory of the Node
#    process. It stays flat.

# 8. Let a signed link expire, then press the download control. The interface
#    states that the link expired, and it offers a new one.
```

## Review checklist

- [ ] Does every user-supplied file come from an origin other than the
      application?
- [ ] Does every served user file carry `Content-Disposition: attachment`?
- [ ] Does every served user file carry `X-Content-Type-Options: nosniff`?
- [ ] Is a user-supplied SVG converted to a raster image, or served under all
      three rules above?
- [ ] Does a private download use a short-lived signed URL, or a Route Handler
      that verifies the session in its own body?
- [ ] Does every streaming handler pass the body through, and never read it into
      memory?
- [ ] Does the interface hold a state for an expired link?
- [ ] Is the `download` attribute used only for a same-origin, `blob:`, or
      `data:` URL?
- [ ] Is `showSaveFilePicker` feature-detected, with an `<a download>` fallback?
- [ ] Is every object URL that a download creates released after the click?
- [ ] Does the download control hold a pending state, and does it announce the
      finish?
- [ ] Does a file that the browser builds run in a Worker above a stated size?

## Handoffs

- What goes inside an export file, the escape over a formula prefix, and the
  byte order mark → `references/cell-formatting-and-export.md`.
- The object URL of a preview, and its release →
  `references/image-and-video-delivery.md`.
- The upload that put the file in storage →
  `references/file-upload-and-transport.md`.
- The session gate inside a Route Handler, and the 403 that it produces →
  `references/route-protection-and-permissions.md`.
- The cookie attributes, the domain of the cookie, and the lifetime of a
  credential → `references/session-and-token-lifecycle.md`.
- The Route Handler as a shape, and its place in the data access layer →
  `references/data-access-and-mutations.md`.
- The request that fetches a file, its timeout, and its abort signal →
  `references/api-client-and-request-safety.md`.
- The name of the download control, and the announcement when a file is ready →
  `references/semantics-and-accessible-names.md` and
  `references/keyboard-focus-and-live-regions.md`. Both domains hold a veto.
- The Content Security Policy, the response headers of the application, and the
  full threat model over user content → domain 17 `frontend-security`. Not
  integrated yet.
- The interaction cost of a build on the main thread → domain 16
  `performance-and-web-vitals`. Not integrated yet.
- The words in a download message and in an expiry warning →
  `references/error-and-empty-state-copy.md`.
- The reverse proxy, the storage bucket, and the header that it sends → domain
  22 `build-deploy-and-runtime-ops`. Not integrated yet.
- The test that follows a download and reads the file → domain 20
  `testing-and-quality`. Not integrated yet.
- The server-side check over a stored file, the name that it is stored under,
  and the headers that Django sends → the sibling skill `secure-code-auditor`.
- The worker that builds a large file, and its retries → the sibling skill
  `django-async-jobs`.
