# Degradation and health checks

Next.js 16.3, React 19.2.6 or later, TanStack Query v5.101 or later. This file
owns what the application does while a dependency is down. The subjects are the
gate that stops calling a dead backend, the last-good render on a failed
refresh, and the offline state. They also include the deploy that must not
break an open tab. The last subject is the health route and what it proves.

The retry rule for one request, the backoff, and `Retry-After` are
`references/api-client-and-request-safety.md`. The query key, the cache entry,
and the four states of a data view are
`references/server-state-and-query-cache.md`. The words in a degraded message
and in an offline message are `references/error-and-empty-state-copy.md`.

## Principle

A dependency fails for a period, and the application decides what the user sees
during that period. That decision is design work, and a default of "an empty
screen" is the decision that nobody made.

Retry helps one request that failed by chance. It hurts every request when the
dependency is down. The second case needs a gate, and not a longer backoff.

Stale data with a label beats no data with an apology, except where the
staleness is the danger. A balance, a permission, and a stock count are wrong
when they are old.

Offline is a state of the network, and empty is a state of the data. A user who
reads "No records" while the train is in a tunnel believes the records are
gone.

A deploy replaces the files that an open tab still expects. The tab is a client
of the previous release, and the release must plan for it.

A health check that answers 200 whatever happens is a check that lies. It
reports the process, which was never the question.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### A dead backend takes one signal, and not one for each call

Neither Next.js nor React carries a circuit-breaker primitive. The gate below
is a small piece of state that the client module owns.

```ts
// Wrong: the client retries every failed call, and no gate sees the outage.
// Failure: one dashboard with twelve widgets sends twelve requests, each one
// retried three times. The user waits through the whole backoff before the
// first message appears, and the backend receives thirty-six requests while it
// tries to restart.
export const queryClient = new QueryClient({
  defaultOptions: { queries: { retry: 3 } },
});
```

```ts
// Correct: a gate that opens on a measured failure rate, and closes after one
// successful probe.
let openedAt: number | null = null;
const OPEN_FOR_MS = 30_000;

export function backendIsDown() {
  if (openedAt === null) return false;
  if (Date.now() - openedAt < OPEN_FOR_MS) return true;
  openedAt = null; // let one request through, which closes or reopens the gate
  return false;
}

export function recordOutage() {
  openedAt ??= Date.now();
}
```

Set the threshold from a measurement, and never from a guess. Read the failure
rate that the reporter records for the client, pick a rate that no healthy
period reaches, and open the gate above it.

The gate serves the user before it serves the backend. It turns a wait of
several backoff periods into one immediate message that names the cause.

A gate that never closes is an outage of its own. Let one request through after
the period, and use its result as the decision.

`references/api-client-and-request-safety.md` owns the retry rule for one
request, the exponential backoff, the cap, and `Retry-After`. This file owns
the rule that stops the retries when the failure is not one request.

### Last-good data beats an empty view, until it does not

A view loaded, and the refresh failed. The cache still holds the previous
answer.

Keep that answer on the screen, and mark it. The label states that the value is
from an earlier moment, and the control offers a new attempt. A view that
throws away good data because a refresh failed shows less than it holds, for the
whole outage.

Three kinds of data refuse this rule. A balance, a permission, and a count that
governs a purchase are wrong when they are old. A stale render of one of them
causes the harm that the refresh was for. Show an explicit failure for those,
and never a stale number.

State the choice in the code, for each query that a view depends on. A reader
of the file learns which of the two rules the value takes.

`references/server-state-and-query-cache.md` owns the cache entry, the key, and
the four states of a data view. `references/error-and-empty-state-copy.md` owns
the words of the label.

### Offline is not empty

TanStack Query v5 tracks the connection with `onlineManager`. It listens to the
`online` and `offline` events of the window, and it starts in the online state.
It no longer reads `navigator.onLine`, because that value reports a false
negative on some browsers.

```tsx
// Wrong: one branch for no data.
// Failure: the reader loses the network, and the screen says "No orders yet".
// The reader believes the account is empty, and calls support.
if (data.length === 0) return <EmptyState />;
```

```tsx
// Correct: the network state and the data state are two questions.
const online = useOnlineStatus(); // onlineManager.subscribe under the hook

if (!online) return <OfflineState onRetry={refetch} />;
if (data.length === 0) return <EmptyState />;
```

`navigator.onLine` reports that an interface is up, and not that a server is
reachable. A captive portal, a broken tunnel, and a dead backend all report
`true`. Pair the event with a real probe, which the health route below
provides.

`networkMode` decides what a query and a mutation do while the browser is
offline. The default is `'online'`, which pauses the work rather than failing
it. The other values are `'always'` and `'offlineFirst'`.

A paused mutation resumes on reconnect. Call `resumePausedMutations()` when the
connection returns, and confirm that the mutation is safe to send late. A
mutation that carries no idempotency key is not, and
`references/api-client-and-request-safety.md` owns that rule.

An offline surface states three things. It names the cause, it says that the
work is not lost, and it offers a new attempt.

### A deploy must not break an open tab

A build names its JavaScript chunks by a hash. The next build produces
different names and deletes the previous files. A tab that a reader opened
before the deploy still asks for the old names, receives a 404, and raises
`ChunkLoadError`. The page then does nothing at all.

```ts
// next.config.ts
// Correct: the client detects the mismatch and performs a full navigation,
// which loads the current build.
import type { NextConfig } from "next";

const buildId = process.env.GIT_SHA ?? "development";

const nextConfig: NextConfig = {
  deploymentId: buildId,
  generateBuildId: async () => buildId,
};

export default nextConfig;
```

```ts
// Wrong: neither key is set, and a try block around the import is the plan.
// Failure: the first navigation after a deploy fails on a deleted chunk. The
// reader sees an inert page, and a reload is the only recovery that works.
const Chart = dynamic(() => import("./chart").catch(() => null));
```

`deploymentId` makes Next.js send the identifier of the build on its responses.
The client compares it, and it performs a hard navigation on a mismatch rather
than a client-side transition that would fail. `generateBuildId` keeps the
chunk names equal across every instance of one release, which matters as soon
as more than one process serves the application.

Serve `/_next/static` with an immutable cache directive, so the reader who
already has a chunk never asks for it again.

A Server Action carries an encryption key that a build generates. Where the key
changes between deploys, an open tab sends an action that the new build cannot
read. The call then fails for that reader alone. Set
`NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` to one value across the deploys of one
release line. The value is a secret, and
`references/secret-boundary-and-supply-chain.md` owns where it may sit.

`references/release-pipeline-and-rollback.md` owns the rollout itself and the
rollback. `references/runtime-process-and-reverse-proxy.md` owns the cache
directive at that layer. This file owns the detection of the mismatch and the
configuration that prevents it.

### The health route verifies the chain

```ts
// app/api/health/route.ts
// Correct: the route answers for the chain, and it states nothing else.
import { NextResponse } from "next/server";

export const runtime = "nodejs";
export const dynamic = "force-dynamic"; // a cached health check reports the past

export async function GET() {
  try {
    // The probe calls the backend directly. It carries no session, it parses
    // no body, and it must not retry, so the typed client is the wrong tool.
    const res = await fetch(`${process.env.DJANGO_URL}/api/health/`, {
      signal: AbortSignal.timeout(3000),
      cache: "no-store",
    });
    if (!res.ok) {
      return NextResponse.json({ status: "unhealthy" }, { status: 503 });
    }
    return NextResponse.json({ status: "healthy" });
  } catch {
    // handle: any failure to reach Django is an unhealthy answer
    return NextResponse.json({ status: "unhealthy" }, { status: 503 });
  }
}
```

```ts
// Wrong: the handler answers for itself.
// Failure: Django is down, the probe is green, and no rollback starts. The
// first report of the outage is a user, and the deploy that caused it already
// finished.
export async function GET() {
  return Response.json({ status: "ok" });
}
```

The timeout is not optional. A check that hangs is worse than a check that
fails, because the probe waits and the orchestrator waits with it.

The body states the verdict and nothing more. A version string, an environment
name, an internal address, and a dependency list are each a value that a public
endpoint gives away for free.

An internal probe reaches the application from inside the network. It therefore
cannot see a failure of the name resolution, of the certificate, or of the
content network in front of it. Pair it with an external check that requests
the same route from outside the infrastructure.

The Django health endpoint, the process that serves it, and the go-live gate
belong to the backend. This file owns the Next.js route and what the frontend
probe proves.

### The libraries

The table gives each package its rule and its maintenance status. Read the
installed version from `package.json` before you write code.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `@tanstack/react-query` | `onlineManager`, `networkMode`, and `resumePausedMutations` are the offline surface. `references/server-state-and-query-cache.md` owns the cache design over them. | 5.101 or later | Current | TanStack, active | None |
| Recommend | `next` configuration keys | `deploymentId` and `generateBuildId` are the skew fix. No library replaces them. | 16.3 | Current | Vercel, active | None |
| Conditional | An external uptime service | Only where the team needs a request from outside its own infrastructure. An internal probe cannot see a name, a certificate, or an edge failure. | — | — | — | — |
| Reject | A health handler that answers 200 for the process | It reports that Node is running, which was never the question. | — | — | — | — |
| Reject | A retry over every call while a backend is down | It multiplies the load and delays the message to the user. | — | — | — | — |
| Reject | `navigator.onLine` as the only offline test | It reports the interface, and not the reachability of a server. | — | — | — | — |
| Reject | A `catch` around a dynamic import as the skew fix | It hides the cause, and the configuration keys are the fix. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| The whole screen waits, then fails at once | Every widget retried against a dead backend | Stop Django, and count the requests in the network panel | Open a gate on the failure rate, and fail fast |
| A page that held data now holds nothing | A failed refresh discarded the cache entry | Break one request after the first load | Render the last good answer with a label |
| A stale balance renders during an outage | The last-good rule reached a value that must be current | Read which queries take the rule | Show an explicit failure for that query |
| "No records" appears in a tunnel | One branch serves the empty state and the offline state | Toggle the offline mode in the browser tools | Split the two states, and render a distinct offline surface |
| The connection returns and a mutation is lost | Nothing resumed the paused mutations | Go offline, submit, then reconnect | Call `resumePausedMutations` on reconnect |
| The first click after a deploy does nothing | A deleted chunk answered 404 | Read the reporter for a rise after each release | Set `deploymentId` and `generateBuildId` |
| One reader's Server Actions fail after a deploy | The action encryption key changed between builds | Compare the failures against the release time | Pin `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` across the deploys |
| The probe is green and the application is broken | The route answers for the process | Stop Django, and read the status of the route | Verify the chain, and answer 503 |
| The probe hangs and the orchestrator waits | The dependency call carries no timeout | Make Django accept the connection and never answer | Bound the call with `AbortSignal.timeout` |
| An outage of the name resolution reaches nobody | The only check runs inside the network | Read where the checks run from | Add a request from outside the infrastructure |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| TanStack Query v5 dropped `navigator.onLine` from `onlineManager` | `rg -n 'navigator.onLine' src/` reports a hit | Subscribe to `onlineManager`, and pair it with a probe |
| TanStack Query v5 defaults `networkMode` to `'online'` | A mutation that fails rather than pauses while offline | State the mode for each mutation, and resume on reconnect |
| Next 14.1.4 promoted `deploymentId` out of `experimental` | `rg -n -A2 'experimental' next.config.*` reports `deploymentId` | Move the key to the top level |
| Next 16 serves `proxy.ts` on the Node runtime | A health route that a proxy rule rewrites for the Edge runtime | Keep the route on the Node runtime, which the `runtime` export states |
| Next 16.3 names the boundary prop `retry` | A degraded surface that calls `reset()` | Rename the prop and the handler to `retry` |

## Verification

```bash
# 1. Read the installed versions before you write code.
rg -n '"next"|"@tanstack/react-query"' package.json

# 2. Confirm the skew keys. Read both.
rg -n 'deploymentId|generateBuildId' next.config.ts

# 3. Confirm the action encryption key across the deploys.
rg -n 'NEXT_SERVER_ACTIONS_ENCRYPTION_KEY' .env.example

# 4. Find the only-offline test. This prints nothing.
rg -n 'navigator\.onLine' src/

# 5. Confirm the offline surface. Read every hit.
rg -n 'onlineManager|networkMode|resumePausedMutations' src/

# 6. Confirm the health route, its runtime, and its timeout.
rg -n -A12 'export async function GET' src/app/api/health/route.ts

# 7. Find a health body that states more than the verdict. Read every hit.
rg -n 'version|env|host|NODE_ENV' src/app/api/health/route.ts

# 8. Stop Django, then read the status of the health route. Expect 503.
curl -s -o /dev/null -w '%{http_code}\n' "$APP_ORIGIN/api/health"

# 9. With Django stopped, open each primary surface. Each one names the cause
#    and offers a new attempt, and no surface signs the reader out.

# 10. With Django stopped, count the requests that one dashboard sends. The
#     count stops rising after the gate opens.

# 11. Set the browser to the offline mode, and read each data view. The offline
#     surface is distinct from the empty state.

# 12. Submit one mutation while offline, then reconnect. The mutation runs once.

# 13. Build, deploy, then reload a tab that the previous build served. The tab
#     performs a full navigation rather than a failed transition.
```

## Review checklist

- [ ] Does a run of failures open a gate that stops the calls?
- [ ] Does the threshold of that gate come from a measured failure rate?
- [ ] Does the gate let one request through after its period?
- [ ] Does a failed refresh keep the last good answer with a label on it?
- [ ] Does every query that carries a balance, a permission, or a purchase
      count show an explicit failure instead?
- [ ] Does the offline surface differ from the empty state?
- [ ] Does the offline test read the connection events rather than
      `navigator.onLine` alone?
- [ ] Does a real probe confirm reachability beside those events?
- [ ] Does each mutation state its `networkMode`?
- [ ] Does a reconnect resume the paused mutations?
- [ ] Is every mutation that resumes late idempotency-keyed?
- [ ] Are `deploymentId` and `generateBuildId` both set from one build
      identifier?
- [ ] Does `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` hold one value across the
      deploys of a release line?
- [ ] Does `/api/health` call the backend and answer 503 when it fails?
- [ ] Does that call carry a timeout?
- [ ] Does the health body state the verdict alone, with no version, no
      environment, and no internal address?
- [ ] Does a check run from outside the infrastructure beside the internal one?

## Handoffs

- The capture of an error, the scrub before a report leaves, and the alert rule
  → `references/error-capture-and-reporting.md`.
- The correlation identifier, the structured log, and the trace →
  `references/correlation-and-telemetry.md`.
- The retry rule for one request, the exponential backoff, the cap,
  `Retry-After`, and the idempotency key →
  `references/api-client-and-request-safety.md`.
- The query key, the cache entry, the four states of a data view, and the
  `retry` option over an `ApiError` →
  `references/server-state-and-query-cache.md`.
- The words of a degraded message, of an offline message, and of an empty
  state → `references/error-and-empty-state-copy.md`.
- The boundary that renders a failure, and its recovery control →
  `references/suspense-and-actions.md`.
- `next.config.ts` as a file, the route files, and the render mode →
  `references/app-router-structure.md`.
- The size cap and the method list on the health Route Handler →
  `references/exposed-endpoints-and-destinations.md`. That domain holds a veto.
- The place of a secret such as the action encryption key →
  `references/secret-boundary-and-supply-chain.md`. That domain holds a veto.
- The reconnect of a live connection, its heartbeat, and its degraded state →
  `references/push-transport-and-connection.md`.
- The journey that runs against a failed route, and the interception that
  produces it → `references/end-to-end-journeys-and-flake-control.md`.
- The Docker image → `references/build-output-and-container-image.md`. The
  reverse proxy and the immutable cache directive on `/_next/static` →
  `references/runtime-process-and-reverse-proxy.md`. The rollout and the
  rollback → `references/release-pipeline-and-rollback.md`.
- The Django health endpoint, the ASGI process behind it, and the go-live gate →
  the backend. This file owns the Next.js route and what the frontend probe
  proves.
- The rate limit that produces a 429, and the throttle behind it → the
  backend's security review.
