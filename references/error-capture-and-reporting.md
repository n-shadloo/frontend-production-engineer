# Error capture and reporting

Next.js 16.3, React 19.2.6 or later, `@sentry/nextjs` 10.70.0,
`react-error-boundary`. This file owns the path from a thrown value to a
reporter. The subjects are the report inside a boundary, and the server hook that
sees what a client boundary cannot. They also include the client start file,
the failures that no boundary catches, and the `catch` block that neither
handles nor rethrows. The last subjects are what must never leave in a report,
the noise that must never raise an alert, and the source map.

The words that a boundary renders are
`references/error-and-empty-state-copy.md`. Where a boundary belongs in the
tree, and the shape of its fallback, are `references/suspense-and-actions.md`.
The route files themselves are `references/app-router-structure.md`. The
identifier that ties a report to a Django log line, and the structured log, are
`references/correlation-and-telemetry.md`.

## Principle

An error that nobody records did not happen. The team learns about it from a
user, weeks later, with no reproduction.

A boundary catches a render. It never catches an event handler, a timer, or a
rejected promise. Code that assumes otherwise ships a silent failure.

Every `catch` block makes one of three decisions. It handles the failure, it
reports it, or it rethrows it. A block that does none of the three deletes the
evidence.

A report that carries a person's data is a second incident. The reporter is a
third-party store with its own retention, its own access list, and its own
breach surface.

Noise is not evidence. An alert that nobody acts on trains a team to act on no
alert.

The production build hides the exception text from the reader. It must not hide
it from the team. The source map goes to the reporter, and never to the public
origin.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The boundary reports, and the digest ties it to the server

```tsx
// Wrong: the boundary renders a fallback and reports nothing.
// Failure: the segment is dead for the user and silent for the team. No
// report reaches the tracker, so the defect has no owner and no count.
"use client"; // reason: an error boundary needs client state

export default function Error() {
  return <p>Something went wrong</p>;
}
```

```tsx
// Correct: the boundary reports once, and the digest ties the report to the
// server log line.
"use client"; // reason: an error boundary needs client state
import { useEffect } from "react";
import * as Sentry from "@sentry/nextjs";

export default function Error({
  error,
  retry,
}: {
  error: Error & { digest?: string };
  retry: () => void;
}) {
  useEffect(() => {
    Sentry.captureException(error, { tags: { digest: error.digest } });
  }, [error]);

  return (
    <div role="alert">
      <h2>The orders did not load.</h2>
      <button type="button" onClick={retry}>
        Try again
      </button>
    </div>
  );
}
```

The report belongs in an effect, and not in the render. A render can run more
than once for one failure, and each run would send one event.

The `digest` is the only value that both sides hold. Next.js computes it on the
server, and it writes the value beside the stack trace in the server log. It
then forwards the value to the client, with the exception text removed. A
report with no digest tag is a client event that no server line matches.

`references/error-and-empty-state-copy.md` owns the words in this fallback, the
message map behind them, and the disclosure that shows the digest to the user.
This file owns the capture beside them.

### A boundary never sees an event handler

React error boundaries catch a failure in the render, in a lifecycle, and in a
constructor. A click handler, a `setTimeout` callback, and a rejected promise
run outside all three.

```tsx
// Wrong: the rejection escapes the component.
// Failure: the button does nothing, and the user clicks a second time. No
// boundary renders, so nothing on the screen states that the export failed.
// The global handler records a bare rejection that names no control.
"use client";

export function ExportButton({ onExport }: { onExport: () => Promise<void> }) {
  return <button type="button" onClick={() => void onExport()}>Export</button>;
}
```

```tsx
// Correct: the handler catches, reports, and renders its own failure state.
"use client";
import { useState } from "react";
import * as Sentry from "@sentry/nextjs";

export function ExportButton({ onExport }: { onExport: () => Promise<void> }) {
  const [failed, setFailed] = useState(false);

  async function run() {
    setFailed(false);
    try {
      await onExport();
    } catch (error) {
      Sentry.captureException(error); // report: the user keeps the page
      setFailed(true);
    }
  }

  return (
    <>
      <button type="button" onClick={() => void run()}>Export</button>
      {failed ? <p role="alert">The export failed. Try again.</p> : null}
    </>
  );
}
```

The rule holds for a `void` call on a promise. It holds for an event listener
that a component adds, and for a callback that a third-party library invokes.

### Every catch handles, reports, or rethrows

```ts
// Wrong: two blocks that delete the evidence.
// Failure: the request failed, the function returned a default, and the
// screen renders an empty list. The tracker holds nothing, so the team reads
// the defect as "no records" for as long as it takes a user to complain.
try {
  return await listOrders();
} catch {
  return [];
}

void refreshToken().catch(() => {});
```

```ts
// Correct: each block states its decision on one line.
try {
  return await listOrders();
} catch (error) {
  // rethrow: the segment boundary renders the failure, and reports it.
  throw error;
}

void refreshToken().catch((error) => {
  // report: a failed refresh is not fatal here, and the next call retries.
  Sentry.captureException(error);
});
```

A comment of three words is the whole cost. Write `handle:`, `report:`, or
`rethrow:` and the reason.

A `console.log` is not a report. It reaches a browser console that nobody
reads, and a server console whose lines nobody aggregates.

### The server hook catches what a client boundary cannot

`error.tsx` is a Client Component. It renders the fallback for a failed Server
Component, and it never receives the original exception. Without the hook
below, a throw inside a nested Server Component, a Route Handler, a Server
Action, or `proxy.ts` reaches the server console alone.

```ts
// instrumentation.ts, at the project root or in src/
// Correct: the runtime loads its own config, and the hook forwards every
// server error to the reporter.
import * as Sentry from "@sentry/nextjs";

export async function register() {
  if (process.env.NEXT_RUNTIME === "nodejs") {
    await import("./sentry.server.config");
  }
  if (process.env.NEXT_RUNTIME === "edge") {
    await import("./sentry.edge.config");
  }
}

export const onRequestError = Sentry.captureRequestError;
```

```ts
// Wrong: the file registers the SDK and exports no hook.
// Failure: a throw in a nested Server Component renders the segment fallback,
// writes one line to the server console, and reaches no tracker. The team
// finds out from a user.
import * as Sentry from "@sentry/nextjs";

export async function register() {
  if (process.env.NEXT_RUNTIME === "nodejs") {
    await import("./sentry.server.config");
  }
}
```

Next.js calls `onRequestError` with the error, the request path, the method,
the headers, and a context object. The context names the route path and the
route type, which is `render`, `route`, `action`, or `proxy`.

React can reprocess an error from a Server Component, so the value that the
hook receives is not always the instance that the code threw. Correlate on
`error.digest`, and never on object identity.

`references/app-router-structure.md` owns `instrumentation.ts` as a route file.
This file owns what it exports.

### The client start file sends no personal data

```ts
// instrumentation-client.ts
// Correct: personal data stays off, the replay masks every text node, and the
// known noise never reaches an alert.
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  sendDefaultPii: false,
  tracesSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  replaysSessionSampleRate: 0.1,
  integrations: [
    Sentry.replayIntegration({ maskAllText: true, blockAllMedia: true }),
  ],
  ignoreErrors: [/ResizeObserver loop/, "Non-Error promise rejection captured"],
  denyUrls: [/^chrome-extension:\/\//i, /^moz-extension:\/\//i],
  beforeSend(event, hint) {
    const original = hint.originalException as Error | undefined;
    if (original?.name === "AbortError") return null; // a cancelled request
    if (event.request?.cookies) delete event.request.cookies;
    if (event.request?.headers) delete event.request.headers["authorization"];
    return event;
  },
});

export const onRouterTransitionStart = Sentry.captureRouterTransitionStart;
```

```ts
// Wrong: the two settings that produce a data incident.
// Failure: the network address of every user, the session cookie, and the
// text of a password field reach a third-party store. The retention is the vendor's,
// and a deletion request now covers two systems.
Sentry.init({
  dsn,
  sendDefaultPii: true,
  integrations: [Sentry.replayIntegration({ maskAllText: false })],
});
```

`sendDefaultPii` is `false` by default, and the code states it. From
`@sentry/nextjs` 10.4 that flag also gates the network address, so an explicit
`false` is the record that nobody turned it on.

Session Replay masks every text node and blocks every media element by
default. `maskAllText: false` is safe on a surface that renders no personal
value and no credential, and nowhere else.

`beforeSend` is the last gate. It runs on every event, and a value that it
deletes never leaves the browser. Delete the cookies, the `authorization`
header, and any request body that the SDK attached.

`references/secret-boundary-and-supply-chain.md` owns what must never cross to
the browser. This file owns what must never leave it in a report.

### The source map reaches the reporter and not the origin

Upload the client source maps to the reporter at build time, and delete them
from the build output in the same step. A `.js.map` that the public origin
serves gives every reader the original source of the application.

`sourcemaps.deleteSourcemapsAfterUpload` performs the deletion, and current
`@sentry/nextjs` sets it by default. State it in `withSentryConfig` anyway, so
a later upgrade cannot change the behavior in silence.

`hideSourceMaps` is dead. It removed the `sourceMappingURL` comment and left
the map file in place, which hid the map from a browser and from nobody else.
`@sentry/nextjs` v9 removed the option.

### The global handlers report a rejection

`Sentry.init` installs a `window.onerror` handler and an `unhandledrejection`
handler. Both report to the SDK, so a rejected promise that no `catch` reaches
still produces an event.

Two things break this. Code that assigns `window.onerror` after the SDK starts
replaces the handler. A project with no SDK has no handler at all, and every
rejection is silent.

Confirm that the project holds one owner for each of the two handlers. Where
the project reports without an SDK, add both handlers, and send the event to
the same endpoint that the boundary uses.

### An alert has an owner and an action

An alert rule states three things. It names the condition, it names the person
who answers it, and it names what that person does.

A rule that fails any of the three is noise, and the team learns to close it
unread.

Three conditions are worth an alert. They are the rate of a new failure, a new
issue in a paid flow, and a rise in a counter that a release changed. A 404, a
`ResizeObserver loop` message, an extension error, and a cancelled request are
not conditions.

Filter the noise at the SDK, and not at the alert rule. `ignoreErrors` and
`denyUrls` drop the event before it costs a quota. A rule that filters after
the event still pays for it, and it still shows the issue in the list.

`references/live-events-and-cache-merge.md` and
`references/push-transport-and-connection.md` emit the counters for a live
feed. This file owns the rule over them.

### The libraries

The table gives each package its rule and its maintenance status. Read the
installed version from `package.json` before you write code.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `@sentry/nextjs` | The reporter for the browser and for the Node runtime. It instruments a Server Component, a Route Handler, and a Server Action through `onRequestError`. The floor is 10.27.0. | 10.70.0 | August 2026 | Sentry, active | GHSA-6465-jgvq-jhgp, fixed in 10.27.0 |
| Recommend | `react-error-boundary` | The boundary for one widget inside a segment. The route files are boundaries for a segment alone. | Current | Current | Community, active | None |
| Conditional | GlitchTip | Only where the team self-hosts and needs a flat cost. It accepts the Sentry wire format, so the client code does not change and the DSN does. It carries no session replay and no profile. | Current | Current | GlitchTip, active | None |
| Reject | `sentry.client.config.ts` as a new file | `instrumentation-client.ts` replaced it. The rename is safe on every supported Next.js version. | — | — | — | — |
| Reject | `hideSourceMaps` | It removed the comment and served the map. `@sentry/nextjs` v9 removed the option. | — | — | — | — |
| Reject | `experimental.instrumentationHook` | Next.js detects the file. The key does nothing since 15.3. | — | — | — | — |
| Reject | A `catch` block with a `console.log` and no rethrow | It reads as a report and reaches no aggregator. | — | — | — | — |

`@sentry/nextjs` 10.11.0 through 10.26.0 sent the `Authorization` header and
the `Cookie` header that earlier versions removed, where `sendDefaultPii` was
`true`. 10.27.0 fixed it. The pinned version is above the fix, and
`sendDefaultPii: false` with a `beforeSend` gate is the layer that does not
depend on a version number.

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| A click does nothing, and the tracker is empty | A boundary does not catch an event handler | Throw inside an `onClick`, and read the tracker | Catch in the handler, report, and render a failure state |
| A Server Component error reaches no tracker | `instrumentation.ts` exports no `onRequestError` | Throw in a nested Server Component of a production build | Export `Sentry.captureRequestError` from the hook |
| The tracker holds a client event that no server line matches | The report carries no digest | Read one issue, and search the server log for its identifier | Tag the capture with `error.digest` |
| One failure produces many identical events | The capture sits in the render | Read the event count of one issue against the user count | Move the capture into an effect that depends on the error |
| A user address, a cookie, or a password appears in an event | `sendDefaultPii: true`, or an unmasked replay | Open one event, and read the request section | Set `sendDefaultPii: false`, mask the replay, and delete the fields in `beforeSend` |
| The public origin serves a `.js.map` | The upload ran, and the deletion did not | Request one map path from the origin, and read the status | Set `sourcemaps.deleteSourcemapsAfterUpload` |
| Nobody reads the alerts | The rules fire on a 404 and on an extension error | Count the issues that nobody assigned | Filter at the SDK, and give each remaining rule an owner |
| The tracker is empty and the application is broken | An empty `catch` returned a default | Read every `catch` block in the diff | Handle, report, or rethrow, with the reason on one line |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next 16.3 names the boundary prop `retry` | `rg -n 'reset' -g 'error.tsx' -g 'global-error.tsx' src/` reports a hit | Rename the prop and the handler to `retry` |
| Next 16.3 adds `catchError` to `next/error` for a boundary inside a component | A widget with no boundary of its own | Read the signature from the installed build, then use it or `react-error-boundary`. `references/suspense-and-actions.md` owns the placement |
| Next 15 added `onRequestError`, and Next 16 keeps it | `rg -n 'onRequestError' instrumentation.ts` prints nothing | Export `Sentry.captureRequestError` |
| Next 15.3 ignores `experimental.instrumentationHook` | `rg -n 'instrumentationHook' next.config.*` reports a hit | Delete the key |
| `@sentry/nextjs` v8 rebuilt the server SDK on OpenTelemetry | `rg -n 'withSentryApi' .` reports a hit | Call `wrapApiHandlerWithSentry` |
| `@sentry/nextjs` v9 removed `hideSourceMaps`, and deletes a client map after upload | `rg -n 'hideSourceMaps' .` reports a hit | Delete the option, and set `sourcemaps.deleteSourcemapsAfterUpload` |
| `@sentry/nextjs` v10.4 gates the network address behind `sendDefaultPii` | The option is absent from the init call | State `sendDefaultPii: false` |
| `@sentry/nextjs` 10.11.0 through 10.26.0 leaked two headers | `rg -n '"@sentry/nextjs"' package.json` reports a version under 10.27.0 | Upgrade to 10.27.0 or later |
| `captureFeedback` supersedes `captureUserFeedback` | `rg -n 'captureUserFeedback' src/` reports a hit | Call `captureFeedback`, and pass the identifier of the associated event |

## Verification

```bash
# 1. Read the installed versions before you write code.
rg -n '"next"|"@sentry/nextjs"|"react-error-boundary"' package.json

# 2. Confirm the server hook. This must print one hit.
rg -n 'onRequestError' instrumentation.ts src/instrumentation.ts

# 3. Confirm that every boundary file reports. Read every file that prints.
rg --files-without-match 'captureException' -g 'error.tsx' -g 'global-error.tsx' src/

# 4. Find the stale boundary prop. This prints nothing.
rg -n 'reset' -g 'error.tsx' -g 'global-error.tsx' src/

# 5. Find an empty catch. This prints nothing.
rg -n 'catch\s*(\([^)]*\))?\s*\{\s*\}|\.catch\(\(\) => \{\}\)' src/

# 6. Find a catch whose only statement is a console call. Read every hit.
rg -n -A2 '\} catch' src/ | rg 'console\.'

# 7. Confirm the personal-data setting. Read the hit.
rg -n 'sendDefaultPii' src/ instrumentation-client.ts

# 8. Confirm the replay mask. Read both options.
rg -n 'maskAllText|blockAllMedia' src/ instrumentation-client.ts

# 9. Confirm the scrub gate. Read the body.
rg -n -A10 'beforeSend' src/ instrumentation-client.ts

# 10. Confirm that the build deletes the client maps.
rg -n 'deleteSourcemapsAfterUpload|hideSourceMaps' next.config.ts

# 11. Read the status of one built source map from the deployed origin.
curl -s -o /dev/null -w '%{http_code}\n' "$APP_ORIGIN/_next/static/chunks/$CHUNK.js.map"

# 12. Throw inside a nested Server Component of a production build, and read
#     the tracker. One event arrives, and it carries the digest.

# 13. Throw inside a click handler, and read the tracker. One event arrives.

# 14. Open one captured event, and read its request section. No cookie, no
#     authorization header, and no request body is present.

# 15. Read the alert rules. Each one names an owner and an action.
```

## Review checklist

- [ ] Does every `error.tsx` and `global-error.tsx` file capture the error in
      an effect?
- [ ] Does each capture carry the digest as a tag?
- [ ] Does every boundary file take the `retry` prop of Next 16.3?
- [ ] Does `instrumentation.ts` export `onRequestError`?
- [ ] Does every `catch` block handle, report, or rethrow, with the reason on
      one line?
- [ ] Is a `catch` block whose only statement is a console call absent?
- [ ] Does every event handler, timer, and async callback that can fail carry
      its own catch and its own report?
- [ ] Is `sendDefaultPii` stated as `false`?
- [ ] Does the replay set `maskAllText` and `blockAllMedia` to `true`?
- [ ] Does `beforeSend` delete the cookies and the authorization header?
- [ ] Does the SDK drop a cancelled request, a `ResizeObserver loop` message,
      and an extension error before it sends?
- [ ] Is the installed `@sentry/nextjs` 10.27.0 or later?
- [ ] Does the build delete the client source maps after the upload?
- [ ] Does the public origin answer a `.js.map` request with a 404?
- [ ] Does the project own one `window.onerror` handler and one
      `unhandledrejection` handler?
- [ ] Does every alert rule name a condition, an owner, and an action?

## Handoffs

- The correlation identifier, the structured log, the trace, and the transport
  of a field report → `references/correlation-and-telemetry.md`.
- The gate over a dead backend, the offline state, the version skew, and the
  health route → `references/degradation-and-health-checks.md`.
- The words in a fallback, the message map, and the digest disclosure →
  `references/error-and-empty-state-copy.md`.
- Where a boundary sits in the tree, the fallback shape, and the boundary
  around one widget → `references/suspense-and-actions.md`.
- `error.tsx`, `global-error.tsx`, `not-found.tsx`, and `instrumentation.ts` as
  route files → `references/app-router-structure.md`.
- The `"use client"` directive on a boundary file →
  `references/server-and-client-components.md`.
- The rule that no exception text and no unrendered record reaches the browser
  → `references/exposed-endpoints-and-destinations.md`. What must never cross
  to the browser, and the audit of a new dependency, are
  `references/secret-boundary-and-supply-chain.md`. That domain holds a veto.
- `ApiError`, the normalizer, and the retry rule over a status →
  `references/api-client-and-request-safety.md`.
- The connection status, the reconnect count, and the parse counter that feed
  an alert → `references/push-transport-and-connection.md` and
  `references/live-events-and-cache-merge.md`.
- The consent that must arrive before a session replay starts →
  `references/consent-gate-and-cookie-inventory.md`. The same scrub on the path
  that an analytics event takes →
  `references/event-taxonomy-and-tracking-plan.md`.
- The test that proves a boundary renders and recovers →
  `references/test-strategy-and-component-tests.md`.
- The server-side guarantee that no response body carries a stack trace → the
  sibling skill `secure-code-auditor`. This file owns the browser side and the
  Node side of the Next.js application.
