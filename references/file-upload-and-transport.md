# File upload and transport

Next.js 16.3, React 19.2.6 or later, `magic-bytes.js` 1.13.1,
`browser-image-compression`, Uppy with `@uppy/tus` and `@uppy/aws-s3`, the tus
protocol 1.0.0, against a Django and DRF backend. This file owns the path that
a file takes from the disk of a user into storage. The subjects are the file
input, the drop zone, the check that the browser makes, and the step that
rewrites the bytes before they leave. They also include the transport decision,
the progress report, the cancel control, the retry, and the confirm call that
makes the object real.

The image and the video after a request stores them are
`references/image-and-video-delivery.md`. The file that the application serves
back, and the file that reaches the disk of a user, are
`references/served-content-and-downloads.md`.

## Principle

A file that a user picks is untrusted input of an unknown size. The browser
reports the type from the name of the file, and an attacker controls that name.

Every check that the browser makes is for the user. It gives a fast answer, and
it saves a failed request. The server decides.

A file is many times larger than a text field. The transport that carries a
text field does not carry a video.

An upload takes time that a user can see. Time that a user can see needs a
number, a stop control, and a second attempt.

An upload has two ends. The bytes reach storage first, and the record of them
reaches the database second. The file does not exist until the second one
returns.

A photograph carries more than a picture. It carries the camera, the moment,
and often the position of the person who took it.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The drop zone holds a real file input

```tsx
// Wrong: the drop zone is a div with two handlers.
// Failure: a keyboard user and a switch-device user cannot upload at all, which
// fails criterion 2.5.7. The highlight also flickers, because dragenter and
// dragleave both fire again for each child element under the pointer.
<div onDrop={onDrop} onDragOver={(e) => e.preventDefault()}>Drop files</div>
```

```tsx
// Correct: a label wraps a real input, and a counter holds the highlight
// steady across the child elements.
"use client"; // it holds drag state and a file input

import { useState } from "react";

import { cn } from "@/lib/utils";

function DropZone({ onFiles }: { onFiles: (files: FileList) => void }) {
  const [depth, setDepth] = useState(0);

  return (
    <label
      className={cn(
        "block rounded-lg border border-dashed p-6",
        "focus-within:ring-2 focus-within:ring-ring",
        depth > 0 && "ring-2 ring-ring",
      )}
      onDragEnter={() => setDepth((d) => d + 1)}
      onDragLeave={() => setDepth((d) => d - 1)}
      onDragOver={(e) => e.preventDefault()}
      onDrop={(e) => {
        e.preventDefault();
        setDepth(0);
        onFiles(e.dataTransfer.files);
      }}
    >
      <span>Drag files here, or browse</span>
      <input
        type="file"
        multiple
        accept="image/png,image/jpeg"
        className="sr-only"
        onChange={(e) => e.target.files && onFiles(e.target.files)}
      />
    </label>
  );
}
```

The input carries the name of the control, so keep the `<span>` inside the
`<label>`. The input is `sr-only` and not `hidden`, because a hidden input takes
no focus. The visible ring comes from `focus-within`, so a keyboard user sees
where the focus sits.

The `accept` attribute filters the file dialog. It is a convenience and not a
check, because a user can select any file. The `capture` attribute asks a phone
for the camera rather than the gallery.

`references/visual-and-motor-criteria.md` owns criterion 2.5.7 and the rule that
every drag needs a second path. This file owns the input under the drop zone.

### The browser checks for the user, and the server decides

```ts
// Wrong: the accepted set is the type that the browser reported.
// Failure: file.type comes from the extension map of the operating system.
// A page named evil.html and renamed to logo.png arrives as image/png, passes,
// and may be served back as HTML.
if (file.type === "image/png") accept(file);
```

```ts
// Correct: read the signature of the file for the message, and state that the
// server decides.
import { filetypemime } from "magic-bytes.js";

const head = new Uint8Array(await file.slice(0, 64).arrayBuffer());
const mimes = filetypemime(Array.from(head));

if (!mimes.includes("image/png")) {
  reject("That file is not a PNG.");
}
// The server re-validates the type, the size, and the content of every upload.
// This check exists to give the user a fast and specific message.
```

Every client-side check carries that comment. A check with no comment reads as
the gate, and the next reader deletes the server check as a duplicate.

Check the size and the count in the same place. The schema that states the type,
the size, and the count belongs to `references/form-schema-and-field-binding.md`,
and that file owns the rule. This file owns the check that runs on the bytes.

The sibling skill `secure-code-auditor` owns the server-side sniff and the virus
scan. It also owns the name that a stored file takes, and the rate limit over
the endpoint. Never write those steps here.

### The bytes change before they leave

A 12-megapixel photograph for a 48-pixel avatar wastes the bandwidth of the
user, the storage of the deployment, and the time of both.

Re-encode the image through a canvas before the upload. The canvas reads the
pixels alone, so the re-encoded file carries no EXIF block, no camera model, and
no GPS position. `browser-image-compression` does this work in a worker, applies
the EXIF orientation so the picture is upright, and defaults `preserveExif` to
`false`. Set `preserveExif: true` only where the product needs the metadata, and
write down the reason.

WARNING: a photograph from a phone often carries the position of the person who
took it. A file that keeps its GPS block, and that later reaches a public URL,
publishes where that person stood. Strip the metadata unless the product needs
it.

A browser cannot decode HEIC or HEIF through `<img>` or `createImageBitmap`, and
an iPhone produces both. Detect the format from the extension and the signature
before any canvas step, because the canvas step throws. Then either refuse the
file with a message that names the format, or convert it with `heic2any`. The
libheif decoder inside that package is about 270 kB, and the whole package is
larger, so load it only when a HEIC file arrives.

### The transport decision

| The condition | The decision | What forces the change |
| --- | --- | --- |
| A file of about 1 MB or less, sent with the other fields of a form | A Server Action or a Route Handler takes the `FormData` | The `serverActions.bodySizeLimit` default is 1 MB, and it counts the whole HTTP body, which holds the multipart boundaries and the part headers. Raise the limit a small amount only. The decision flips when the file approaches the limit, because every byte then passes through Node. |
| A file between 1 MB and the project threshold, with no need to resume | A Route Handler that streams to Django, or a direct `multipart/form-data` request to Django | It is one request, and it needs no storage credential. The Route Handler doubles the bandwidth. The direct request needs the CORS and the CSRF work that `references/cross-origin-and-bff-proxy.md` states. |
| A file above the project threshold, or any video | The browser sends the bytes to object storage with a presigned URL, and then calls a confirm endpoint | Neither Next.js nor Django carries the bytes. The cost is one more round trip and the risk of an orphaned object. The decision flips back where the deployment has no object storage. |
| A file of about 100 MiB or more, or a network that drops | A resumable protocol. `@uppy/tus` speaks tus 1.0.0, and `@uppy/aws-s3` sends an S3 multipart upload | Uppy sends a file of 100 MiB or less in one part, and a larger file in many parts. tus resumes after the tab closes. S3 multipart resumes a failed part, and it resumes after a reload only where the client stores the upload id and the part list. |

The threshold is a decision of the deployment, and it is not a constant. Read
the body limit of the Node process and the `client_max_body_size` of the reverse
proxy, take the smaller number, and record it in the repository. A rule that
states no number is a rule that nobody applies.

NEVER stream a large file through a Server Action or a Route Handler to reach
storage. The bandwidth is paid twice, the whole file sits in the memory of the
process, and the request meets the body limit as a 413.

`references/data-access-and-mutations.md` owns the choice between a Server
Component, a Server Action, and a Route Handler. This file owns the size at
which that choice stops being correct.

### Progress needs `XMLHttpRequest`

```ts
// Wrong: fetch carries the file, and the interface shows a spinner.
// Failure: fetch reports no upload progress. A request body stream needs
// duplex: "half", which is a Chromium-only option, and it still reports no
// progress. The bar sits at 0 percent for two minutes and then jumps to 100.
await fetch(url, { method: "PUT", body: file });
```

```ts
// Correct: XMLHttpRequest reports the bytes it has sent, and it cancels.
export function putWithProgress(
  url: string,
  file: File,
  onPercent: (percent: number) => void,
  signal: AbortSignal,
): Promise<void> {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    xhr.open("PUT", url);
    xhr.upload.onprogress = (e) => {
      if (e.lengthComputable) onPercent((e.loaded / e.total) * 100);
    };
    xhr.onload = () =>
      xhr.status < 300
        ? resolve()
        : reject(new Error(`The storage service refused the file: ${xhr.status}`));
    xhr.onerror = () => reject(new Error("The network dropped the upload."));
    signal.addEventListener("abort", () => xhr.abort(), { once: true });
    xhr.send(file);
  });
}
```

`XMLHttpRequest` is the older API, and it is the correct tool for this one job.
Take `fetch` for every request that carries no file.

Each file in a set needs its own percentage, its own cancel control, its own
retry control, and its own message. A set of ten files with one bar hides which
file failed.

An error message names the cause. "Upload failed" is not a message. "The file is
larger than the 25 MB limit" and "The network dropped the upload" are messages.

Announce the finish through the polite region that
`references/keyboard-focus-and-live-regions.md` owns. A progress bar that
announces each percentage floods a screen reader, and that file holds the rule.

### The presigned upload and the confirm call

The Django endpoint mints the credential and returns `{ url, fields }` for a POST
policy, or `{ url }` for a PUT. The sibling skill `django-api-contract` owns the
shape of that response, and the sibling skill `secure-code-auditor` owns the
scope of the credential. The client has three obligations.

1. Append every entry of `fields` to the `FormData` first, and append the `file`
   entry last. The storage service refuses the request when the file is not last.
2. Send the `Content-Type` that the policy allows. A POST policy states
   `content-length-range` and a `starts-with` condition on `$Content-Type`.
3. Let the runtime set the multipart header.
   `references/api-client-and-request-safety.md` owns that rule, and it holds
   here too.

A POST policy bounds the size and the type. A PUT needs the length in advance and
bounds nothing. Take the POST where the deployment must enforce a limit.

```ts
// Wrong: a rejection from storage reaches the user as the status code.
// Failure: the policy allows 10 MB, the file is 11 MB, and storage answers 403
// with an XML body. The user reads "Error 403" and changes nothing.
if (!response.ok) setError(`Error ${response.status}`);
```

```ts
// Correct: the XML body names the condition, and the interface names the fix.
if (!response.ok) {
  const body = await response.text();
  setError(
    body.includes("EntityTooLarge")
      ? "That file is above the 10 MB limit."
      : "The upload was refused. Please try again.",
  );
}
```

After storage accepts the bytes, call the confirm endpoint with the key that the
upload used. The backend then records the object. Until that call returns 2xx,
the object is in storage and no row points at it, which is the orphaned-object
state. Keep the file out of the interface, and keep the submit control inert,
until the confirm call succeeds. The lifecycle rule that removes an orphaned
object belongs to the backend.

A scan or a transcode runs after the confirm call. The contract exposes a status
for it. Render `pending scan` and `processing` as their own states, and never
present the file as usable while either one holds. The sibling skill
`django-async-jobs` owns the worker behind that status.

### The failures that reach the interface

| The failure | What the user sees | The response of the interface |
| --- | --- | --- |
| 413 from Next.js or from the reverse proxy | The upload stops near the start | Name the limit in the message, and move the path to a presigned upload |
| 403 with an XML body from storage | The upload finishes and then fails | Read the condition out of the body, and name it |
| 401 during a long upload | The upload fails after several minutes | Refresh the session and retry the part, as `references/session-and-token-lifecycle.md` states |
| The network drops | The percentage stops | Offer a retry. A resumable transport continues from the offset |
| The confirm call fails | Storage holds the bytes | Retry the confirm call. Keep the file out of the interface until it returns 2xx |

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those facts on 16 August 2026. A cell that holds no
date is a package with a current registry entry on that date. This file does not
state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `XMLHttpRequest` | A platform API, and no package. It is the one reliable source of upload progress. | Platform | — | Platform | None |
| Recommend | `magic-bytes.js` | The signature check in the browser. About 1.1 million downloads a week. | 1.13.1 | About 30 July 2026 | Active | None |
| Recommend | `browser-image-compression` | The resize and the re-encode before an upload. It applies the EXIF orientation and strips the metadata. | Current | Current | Active | None |
| Recommend | `@uppy/tus` and `@uppy/aws-s3` | The resumable upload, with the interface around it. | Current | Current | Active, by Transloadit | None |
| Recommend | `tus-js-client` | The tus 1.0.0 client, where the project needs no Uppy interface. | Current | Current | Active | None |
| Conditional | `react-dropzone` | Only where the drop zone above is not enough. Confirm that it still renders a real input. | Current | Current | Active | None |
| Conditional | `react-image-crop` or `cropperjs` | Only where the product crops to a fixed ratio. Both add bundle bytes. | Current | Current | Active | None |
| Conditional | `heic2any` | Only where an iPhone HEIC file must be converted rather than refused. The decoder is about 270 kB, and the package is larger. Load it on demand. | Current | Current | Active | None |
| Audit-only | `@uppy/aws-s3` with no stored state | A resume after a reload needs the upload id and the part list in storage. Wire that state, or take tus. | — | — | — | — |
| Reject | A helper that puts base64 in a JSON body | It inflates the bytes by about a third, it holds the whole file in memory at both ends, and it meets the body limit. | — | — | — | — |

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next.js 16.2.11 and 15.5.21 fixed CVE-2026-64646, an unbounded Server Action payload on the Edge runtime | `npm ls next` reports a version below the patch on its line | Upgrade. Until the upgrade lands, cap the request body at 5 MiB in the hosting layer |
| A request body stream still needs `duplex: "half"`, and only Chromium accepts it | A `duplex` option on a `fetch` call that carries a file | Take `XMLHttpRequest`, which reports progress and runs everywhere |

## Verification

```bash
# 1. Read the installed versions before you write code.
rg -n '"next"|"react"|"@uppy/|"magic-bytes.js"|"heic2any"' package.json

# 2. Find a file that travels as base64. This prints nothing.
rg -n 'toDataURL\(|btoa\(|readAsDataURL' -g '*.ts*' src/

# 3. Find a check that trusts the reported type. Read every hit, and confirm
#    that the comment about the server sits beside it.
rg -n -A3 'file\.type\s*===|files\[0\]\.type' -g '*.ts*' src/

# 4. Find the Server Action body limit. An upload path needs a stated decision.
rg -n 'bodySizeLimit' next.config.ts

# 5. Find an upload that carries no progress. Read every hit.
rg -n 'body:\s*(formData|form|file)\b' -g '*.ts*' src/

# 6. Find a file input that no label or no aria-label names. Read every hit.
rg -n -B4 'type="file"' -g '*.tsx' src/

# 7. Find a drop handler in a file that holds no file input.
rg --files-with-matches 'onDrop' -g '*.tsx' src/ \
  | xargs rg --files-without-match 'type="file"'

# 8. Find an upload that no confirm call follows. Read every hit.
rg -n 'presign|createPresignedPost|putWithProgress' -g '*.ts*' src/

# 9. Upload a file of 100 MB. Confirm that the percentage moves, that the
#    cancel control stops it, and that a reload behaves as the design states.

# 10. Rename an HTML page to image.png and upload it. Confirm that the server
#     refuses it and that the interface states why.

# 11. Disconnect the network in the middle of an upload. A resumable transport
#     continues from the offset, and any other transport fails with a message.

# 12. Complete an upload with the keyboard alone. Tab to the control, press
#     Enter or Space, and select the file.

# 13. Upload a photograph from a phone, then read the metadata of the stored
#     file. No EXIF block and no GPS position is present.
```

## Review checklist

- [ ] Does every drop zone hold a real `<input type="file">` that takes focus?
- [ ] Does a keyboard alone complete every upload?
- [ ] Does each client-side check carry a comment that states that the server
      re-validates?
- [ ] Does the client read the signature of the file, rather than the reported
      type alone?
- [ ] Does a file above the project threshold go to object storage, rather than
      through Next.js or Django?
- [ ] Does the repository record the threshold, and does it come from the body
      limit of the deployment?
- [ ] Does every upload show a percentage for each file, a cancel control, and a
      retry control?
- [ ] Does every upload error name its cause?
- [ ] Does the progress report come from `XMLHttpRequest`?
- [ ] Does a presigned POST send the policy fields first and the file last?
- [ ] Does the interface read the rejection body of the storage service, rather
      than showing the status code?
- [ ] Does a confirm call follow every direct upload, and does the interface hold
      the file back until it returns 2xx?
- [ ] Does the interface render a state for a pending scan and for a transcode?
- [ ] Is the metadata stripped from every user photograph, unless the product
      states a need for it?
- [ ] Is a HEIC file detected before any canvas step?

## Handoffs

- The request that carries the file, its timeout, and its abort signal →
  `references/api-client-and-request-safety.md`.
- The CORS preflight and the `X-CSRFToken` header on a direct request to Django →
  `references/cross-origin-and-bff-proxy.md`.
- The choice between a Server Action, a Route Handler, and a Server Component →
  `references/data-access-and-mutations.md`.
- The schema that states the type, the size, and the count of a file field, and
  the submit around it → `references/form-schema-and-field-binding.md` and
  `references/form-submission-and-server-errors.md`.
- The mutation that holds the upload, and the key that its success invalidates →
  `references/server-state-and-query-cache.md`.
- The refresh of a session during a long upload →
  `references/session-and-token-lifecycle.md`.
- The announcement of a finish, and the politeness of a progress region →
  `references/keyboard-focus-and-live-regions.md`. That domain holds a veto.
- The name of the file control, and the alternative text of a preview →
  `references/semantics-and-accessible-names.md`. That domain holds a veto.
- Criterion 2.5.7, and the second path beside a drag →
  `references/visual-and-motor-criteria.md`. That domain holds a veto.
- The preview of a picked file, and the object URL that it needs →
  `references/image-and-video-delivery.md`.
- The origin that serves the stored file back →
  `references/served-content-and-downloads.md`.
- The lockfile entry, the cooldown, and the justification for a new dependency →
  `references/dependencies-and-git-workflow.md`.
- The Content Security Policy over the page that holds the upload →
  `references/security-headers-and-csp.md`. The endpoint that receives the file
  is `references/exposed-endpoints-and-destinations.md`. That domain holds a
  veto.
- The words in an upload message and in a refusal →
  `references/error-and-empty-state-copy.md`.
- The bytes that an upload costs, and the budget over them →
  `references/performance-budgets-and-measurement.md`.
- The body limit at the reverse proxy, and the 413 that it returns →
  `references/runtime-process-and-reverse-proxy.md`.
- The handler that refuses a disguised file →
  `references/network-mocks-and-contract-tests.md`. The test that reads the
  message → `references/test-strategy-and-component-tests.md`.
- The server-side sniff, the virus scan, the name of a stored file, and the scope
  of a presigned credential → the sibling skill `secure-code-auditor`.
- The shape of the presigned response and of the confirm endpoint → the sibling
  skill `django-api-contract`.
- The worker that scans or transcodes the object, and its retries → the sibling
  skill `django-async-jobs`.
