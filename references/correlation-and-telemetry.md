# Correlation and telemetry

Next.js 16.3, `pino`, `@vercel/otel`, W3C Trace Context, `web-vitals` 6.1.
This file owns the signals that leave the application beside an error report.
The subjects are the identifier that ties one screen to one Django log line,
and the trace that crosses to Django. They also include the structured log, and
the values that it must never carry. The last subjects are the transport of a
field metric, and the endpoint that receives a policy violation.

The capture of an error, the scrub before a report leaves the browser, and the
alert rule are `references/error-capture-and-reporting.md`. The metric numbers
and the budget are `references/performance-budgets-and-measurement.md`. The
content of the policy that produces a violation is
`references/security-headers-and-csp.md`.

## Principle

A user reports "it broke at about three o'clock". One identifier turns that
sentence into one request, one trace, and one log line. Without it, the team
reads an hour of logs for every report.

Generate the identifier once, at the first point that the request reaches the
application. Every layer after that reads it and never makes a second one.

A log line is a permanent record. A request body written "for debugging" is a
personal record with no retention policy, no access list, and no deletion path.

A trace that stops at the network boundary is two traces. It shows a slow
frontend and a fast backend, and it names no cause.

A metric that reaches no endpoint is a measurement that nobody made.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### One identifier, generated once and returned

```ts
// proxy.ts
// Correct: one identifier for the request. The caller's value wins, so a
// retry and a client-side trace keep the same reference.
import { NextResponse, type NextRequest } from "next/server";

const HEADER = "x-correlation-id";

export function proxy(req: NextRequest) {
  const id = req.headers.get(HEADER) ?? crypto.randomUUID();

  const headers = new Headers(req.headers);
  headers.set(HEADER, id);

  const res = NextResponse.next({ request: { headers } });
  res.headers.set(HEADER, id); // the browser reads it for the error reference
  return res;
}
```

```ts
// Wrong: the data layer makes its own identifier for each call.
// Failure: one page view produces four identifiers. The user quotes one, and
// it matches one request out of four. The other three are unreachable.
export async function getOrder(id: string) {
  return apiClient.get(`/orders/${id}/`, {
    headers: { "x-correlation-id": crypto.randomUUID() },
  });
}
```

The name of the header is one constant, and both sides read the same one. A
frontend that sends `x-correlation-id` and a Django layer that logs
`x-request-id` produces two records that no query joins.

The identifier reaches three places. It goes to Django on the request header,
and it comes back to the browser on the response header. It then reaches the
user, as the reference in an error surface. Where the failure is a server render,
the Next.js `digest` is that reference instead, and
`references/error-capture-and-reporting.md` owns the tag that ties the two
together.

`references/app-router-structure.md` owns `proxy.ts` as a route file and the
work that it may perform. `references/cross-origin-and-bff-proxy.md` owns the
proxy Route Handler in front of Django, which forwards the same header.

### The log carries metadata, and never a body

```ts
// lib/logger.ts
// Correct: the redaction list removes the secret whatever the call site logs.
import pino from "pino";

export const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  redact: [
    "req.headers.authorization",
    "req.headers.cookie",
    "*.password",
    "*.token",
  ],
  transport:
    process.env.NODE_ENV === "production"
      ? undefined // JSON on one line, which the aggregator parses
      : { target: "pino-pretty" },
});

export const withCorrelation = (id: string) => logger.child({ correlationId: id });
```

```ts
// Wrong: the whole request body reaches the aggregator.
// Failure: an address, an email, and a payment reference sit in a log store
// with a retention of months. A deletion request now covers a system that
// nobody listed, and the export of that store is a data incident.
logger.info({ body: await req.json() });
```

Log the method, the route, the status, the duration, and the correlation
identifier. That set answers every triage question that a body would answer.

The redaction list is the layer that does not depend on the discipline of a
call site. A path of `*.password` reaches a nested object, so a later refactor
cannot move a secret out of the list.

Never pass an object from outside the program as the top-level merge object of
a log call. A key inside it can collide with `level`, `time`, or `msg`, and the
line that reaches the aggregator is then a different line. Nest it under a key
that this repository owns, as in `logger.info({ payload })`.

A child logger carries the correlation identifier onto every line that one
request writes. Create it once for each request, and pass it down.

`references/boundary-validation-and-api-types.md` owns the parse over a value
that enters the program.

### The trace crosses the boundary, or it is two traces

W3C Trace Context defines the `traceparent` header. Its value is four fields
that a hyphen separates, in this order.

| Field | Length | The rule |
| --- | --- | --- |
| version | 2 lowercase hex digits | `00` is the only defined version. `ff` is invalid. |
| trace id | 32 lowercase hex digits | It identifies one whole trace. All zeroes is invalid. |
| parent id | 16 lowercase hex digits | It identifies the caller's span. |
| trace flags | 2 lowercase hex digits | The low bit records the sample decision. |

One example value is
`00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`. A `tracestate`
header travels beside it and carries vendor entries.

```ts
// instrumentation.ts
// Correct: one call registers the tracer for the Node runtime and for the
// Edge runtime. The default propagator writes traceparent on every outbound
// fetch, so the Django span becomes a child of the Next.js span.
import { registerOTel } from "@vercel/otel";

export function register() {
  registerOTel({ serviceName: "storefront-web" });
}
```

```ts
// Wrong: the browser SDK sends no trace header to the backend origin.
// Failure: the trace ends at the browser. A slow page shows one long request
// with no detail, and the Django spans sit in a separate trace that nothing
// joins.
Sentry.init({ dsn, tracesSampleRate: 0.1 });
```

Set `tracePropagationTargets` to the Django origin, so the browser SDK attaches
the header to the requests that reach it and to no other host. A header on a
third-party host leaks the internal trace identifiers of the application.

`@sentry/nextjs` v8 and later build the server SDK on OpenTelemetry. Add
`@vercel/otel` where the spans must also reach a collector that is not the
reporter. Two tracers in one process need one owner for the propagator, so
state which package registers it.

`NEXT_OTEL_VERBOSE=1` makes Next.js emit every span rather than the default
set. Use it to confirm a trace, and never in production.

The Django instrumentation, the settings that enable it, and the propagator list
on that side belong to the backend. This file owns the header that the Next.js
side injects, and the configuration that decides which origins receive it. A
proxy that strips `traceparent` breaks the join, and
`references/cross-origin-and-bff-proxy.md` owns the forwarded header set.

### The field report needs a transport

```tsx
// Wrong: the callback sends the metric with fetch.
// Failure: the metrics arrive on a page that the reader keeps open, and they
// are lost on the page that the reader closes. A bounce reports nothing, so
// the worst sessions are the ones that never reach the endpoint.
useReportWebVitals((metric) => {
  void fetch("/api/vitals", { method: "POST", body: JSON.stringify(metric) });
});
```

```tsx
// Correct: a beacon survives the unload, and the payload carries no personal
// value.
useReportWebVitals((metric) => {
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,
    rating: metric.rating,
    route: window.location.pathname, // the pattern, never the query string
  });
  navigator.sendBeacon("/api/vitals", body);
});
```

`navigator.sendBeacon` hands the request to the browser, which sends it after
the document unloads. That is the only transport that reports the metric of a
session that ends on the measured page.

Send the route pattern, and never the full address. A query string carries a
search term, a filter value, and sometimes an identifier of a person.

Sample the report where the volume costs more than the answer. A metric needs
a percentile over a population, so a sample of one visit in ten still gives the
verdict.

`references/performance-budgets-and-measurement.md` owns which metrics the
project reports, the attribution build behind a cause, and the number that
each one must meet. This file owns how the report leaves the browser.

### The violation report arrives rather than leaves

A policy violation and a report from a browser reporting endpoint are signals
that arrive. Three rules hold for each of them.

The endpoint is a Route Handler with a size cap, and it accepts one method.
`references/exposed-endpoints-and-destinations.md` owns both rules, and that
domain holds a veto.

The body is a value from outside the program, so it takes a parse before
anything reads it. A report is a public endpoint, and any host can post to it.

The count is a signal, and a rise in it is the alert. A single violation from
one browser extension is noise, and
`references/error-capture-and-reporting.md` owns the rule that separates the
two.

`references/security-headers-and-csp.md` owns the policy itself, the
report-only run before an enforced policy, and the directive set. This file
owns the endpoint that receives the reports and the rule over their count.

### The libraries

The table gives each package its rule and its maintenance status. Read the
installed version from `package.json` before you write code.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `pino` | The structured log on the Node runtime. `redact` removes a secret whatever the call site passes. | Current | Current | Pino team, active | None |
| Recommend | `pino-pretty` | The readable transport, in development alone. Production writes one JSON line for each event. | Current | Current | Pino team, active | None |
| Recommend | `@vercel/otel` | One `registerOTel` call for the Node runtime and the Edge runtime. It defaults to the W3C propagator. | Current | Current | Vercel, active | None |
| Recommend | `web-vitals`, the attribution build | The field measurement behind the transport in this file. `references/performance-budgets-and-measurement.md` owns the numbers. | 6.1.0 | Current | Google, active | None |
| Conditional | `@opentelemetry/sdk-node` | Only where the project needs a sampler or an exporter that `@vercel/otel` does not expose. Guard the import, because it does not run on the Edge runtime. | Current | Current | OpenTelemetry, active | None |
| Conditional | `winston` | Only where a repository already standardised on it. `pino` is the default for new work. | Current | Current | Community, active | None |
| Reject | A log line that carries a request body | It writes a personal record with no retention policy. | — | — | — | — |
| Reject | A second correlation identifier for each outbound call | It produces one identifier for each request and joins nothing. | — | — | — | — |
| Reject | `NEXT_OTEL_VERBOSE=1` in production | It emits every span, and the volume buys no answer. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| The reference of a user matches no Django line | The two sides read a different header name | Search the Django log for one identifier from a response header | Take one constant for the name, and read it on both sides |
| One page view produces four identifiers | The data layer generates one for each call | Read the log lines of one navigation | Generate in `proxy.ts`, and forward the value |
| An address sits in the log store | A call logged the request body | Read the aggregator for a known field name | Log the metadata, and add the field to the redaction list |
| A log line lost its level | An untrusted object was the top-level merge object | Read one line whose `level` is not a number | Nest the object under a key that this repository owns |
| The trace ends at the browser | `tracePropagationTargets` omits the Django origin | Read one trace, and count the spans after the request | Add the origin, and confirm the header on the request |
| The Django spans sit in their own trace | A proxy strips `traceparent` | Read the inbound headers at the Django layer | Forward the header at every hop |
| The field metrics under-report a slow page | The transport is `fetch`, and the reader left | Compare the report count against the page-view count | Send with `navigator.sendBeacon` |
| A search term appears in the metric store | The report carried the full address | Read one stored report | Send the route pattern alone |
| The violation endpoint accepts a large body | The Route Handler states no cap | Post a large body, and read the status | Cap the body, which `references/exposed-endpoints-and-destinations.md` owns |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next 16 renamed `middleware.ts` to `proxy.ts`, on the Node runtime | `test -f middleware.ts` succeeds | Move the identifier generation into `proxy.ts` |
| Next 15.3 and later detect `instrumentation.ts` | `rg -n 'instrumentationHook' next.config.*` reports a hit | Delete the key |
| `@sentry/nextjs` v8 and later carry OpenTelemetry in the server SDK | Two packages register a propagator | Name the one owner of the propagator, and read the installed version |
| W3C Trace Context defines version `00` alone | A `traceparent` value that starts with another version | Emit `00`, and treat `ff` as invalid |
| `web-vitals` 6 carries the long animation frame in the attribution build | An import from `web-vitals` where the report needs a cause | Import from `web-vitals/attribution` |

## Verification

```bash
# 1. Read the installed versions before you write code.
rg -n '"next"|"pino"|"@vercel/otel"|"web-vitals"' package.json

# 2. Confirm one owner for the identifier. Read every hit.
rg -n 'x-correlation-id|correlationId' src/ proxy.ts

# 3. Find a second correlation identifier outside the proxy. This prints
#    nothing. An idempotency key is a different value, so the filter is needed.
rg -n -B2 'randomUUID' src/ | rg -i 'correlation|request-id'

# 4. Confirm the redaction list. Read every path in it.
rg -n -A8 'redact' src/lib/logger.ts

# 5. Find a log call that passes a body. Read every hit.
rg -n 'logger\.(info|warn|error|debug)\(\{\s*(body|payload|data)\b' src/

# 6. Confirm the tracer registration. Read the hit.
rg -n 'registerOTel|serviceName' instrumentation.ts src/instrumentation.ts

# 7. Confirm the propagation targets. Read the list.
rg -n 'tracePropagationTargets' src/ instrumentation-client.ts

# 8. Confirm the beacon transport for the field report. Read the hit.
rg -n 'sendBeacon' src/

# 9. Find a field report that carries the full address. This prints nothing.
rg -n -B4 'sendBeacon' src/ | rg 'location\.(href|search)'

# 10. Read one response header set from the running application, and confirm
#     that the correlation header is present.
curl -s -D - -o /dev/null "$APP_ORIGIN/" | rg -i 'correlation'

# 11. Send one request with a known identifier, then search the Django log for
#     that value. One line matches.

# 12. Open one trace after a page load. The Django spans are children of the
#     Next.js span, in one trace.
```

## Review checklist

- [ ] Does `proxy.ts` generate the correlation identifier, and does it reuse a
      value that the caller sent?
- [ ] Does one constant hold the header name, and do both sides read it?
- [ ] Does the response carry the identifier back to the browser?
- [ ] Does the user-facing error reference hold that identifier, or the digest
      that ties to it?
- [ ] Does the logger state a redaction list that covers the authorization
      header, the cookie header, a password, and a token?
- [ ] Is a request body absent from every log call?
- [ ] Does every log call nest an object from outside the program under a key
      that this repository owns?
- [ ] Does a child logger carry the correlation identifier onto each line of
      one request?
- [ ] Does production write one JSON line for each event, with the pretty
      transport in development alone?
- [ ] Does `instrumentation.ts` register a tracer with a service name?
- [ ] Does `tracePropagationTargets` name the Django origin, and no
      third-party host?
- [ ] Does one package own the propagator?
- [ ] Does the field report leave with `navigator.sendBeacon`?
- [ ] Does that report carry the route pattern rather than the full address?
- [ ] Does the report endpoint parse its body before anything reads it?
- [ ] Is `NEXT_OTEL_VERBOSE` absent from the production environment?

## Handoffs

- The capture of an error, the scrub in `beforeSend`, the source map, and the
  alert rule → `references/error-capture-and-reporting.md`.
- The gate over a dead backend, the offline state, and the health route →
  `references/degradation-and-health-checks.md`.
- `proxy.ts` as a route file, `instrumentation.ts`, and the work that each one
  may perform → `references/app-router-structure.md`.
- The proxy Route Handler in front of Django, and the header set that it
  forwards → `references/cross-origin-and-bff-proxy.md`.
- The metrics that the project reports, the attribution build, and the number
  that each metric must meet →
  `references/performance-budgets-and-measurement.md`.
- The policy content, the directive set, and the report-only run →
  `references/security-headers-and-csp.md`.
- The size cap and the method list on a Route Handler →
  `references/exposed-endpoints-and-destinations.md`. That domain holds a veto.
- The parse over a value that enters the program →
  `references/boundary-validation-and-api-types.md`.
- The rule that no credential sits in a `NEXT_PUBLIC_` variable →
  `references/secret-boundary-and-supply-chain.md`. That domain holds a veto.
- The Django instrumentation, the propagator on that side, and the server
  process that emits the spans → the backend. This file owns the header that the
  Next.js side injects.
- The slow endpoint behind a long server span → the backend.
