# Network mocks and contract tests

MSW v2, `openapi-typescript`, `openapi-fetch`, drf-spectacular with OpenAPI
3.0.3, Vitest, Zod 4, Next.js 16.3, React 19.2.6 or later. This file owns the
answer that a test gives to a request, and the proof that the answer matches
the contract. The subjects are the one server that every test shares, and the
rule that an unhandled request fails the run. They also include the handler
that the generated types hold to the schema, and a named handler for each DRF
failure shape. The last subjects are the three pagination envelopes, the
fixture for each auth state, the mock socket, and the contract test.

The level that a test belongs to, and the component test that consumes these
handlers, are `references/test-strategy-and-component-tests.md`. The
interception inside a real browser is
`references/end-to-end-journeys-and-flake-control.md`. The schema artifact, the
generator, and the drift gate are `references/openapi-schema-and-codegen.md`.

## Principle

A test that reaches a real network tests two systems and reports on neither. It
fails when the other system is slow, and it passes when the other system is
wrong.

Mock at the boundary that this application does not own. Everything inside that
boundary runs for real: the client, the parse, the error normalizer, the cache,
and the component.

A mock is a claim about another system. A claim that nothing checks drifts away
from the system, and the suite keeps a memory of an API that no longer exists.

A request that no handler matches must stop the run. Silence at that point is a
test that quietly reaches production.

Every failure that the API can produce is a state that a user can meet. A state
with no handler is a state with no test.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### One server, started once, reset after each test

```ts
// Wrong: src/test/setup.ts starts the server with the default policy.
// Failure: the default reports an unhandled request as a warning. A renamed
// path falls through to the real network. On a laptop it reaches the
// developer's Django and passes. In CI it reaches nothing, and the failure
// names a timeout rather than the renamed path.
beforeAll(() => server.listen());
```

```ts
// Correct: src/test/setup.ts fails the run on any request that no handler
// matches, and returns the base handlers after each test.
import { afterAll, afterEach, beforeAll } from "vitest";
import { server } from "./msw/server";

beforeAll(() => {
  server.listen({ onUnhandledRequest: "error" });
});

afterEach(() => {
  server.resetHandlers();
});

afterAll(() => {
  server.close();
});
```

```ts
// Correct: src/test/msw/server.ts holds the one server for the project.
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);
```

`resetHandlers()` removes every override that a test added with `server.use()`.
Without it, an override from one test answers a request in the next one, and
the order of the file decides the result.

### The base handlers describe the happy path, and a test overrides one

Keep the base set small. It answers the requests that most tests need, and it
returns a successful body. A test that needs another answer states it in its own
body, and the reset removes it.

```ts
// Correct: the test states the one answer that its case needs.
it("renders the error state where the list request fails", async () => {
  server.use(
    http.get("*/api/orders/", () => HttpResponse.json({ detail: "Server error" }, { status: 500 })),
  );
  renderWithProviders(<OrderList />);
  expect(await screen.findByRole("alert")).toBeVisible();
});
```

The path pattern opens with `*` because this application holds two base URLs.
The server reaches Django directly, and the browser reaches the proxy route.
`references/cross-origin-and-bff-proxy.md` owns both values. A handler that
hard-codes one of them matches half of the requests, and the other half fails
the run under the error policy.

### The handler body carries the generated type

```ts
// Wrong: the body is a literal that nobody checks.
// Failure: the backend renames `total` to `total_amount`. The generated types
// change, the component fails to compile, and a person "fixes" the component
// against this handler. Every test passes, and production reads `undefined`.
http.get("*/api/orders/", () =>
  HttpResponse.json({ count: 1, results: [{ id: "1", total: "12.00" }] }),
);
```

```ts
// Correct: the body is typed from the schema, so a renamed field fails the
// typecheck inside the handler.
import { http, HttpResponse } from "msw";
import type { components } from "@/api/generated/schema";
import { buildOrder } from "@/test/factories/order";

type PaginatedOrderList = components["schemas"]["PaginatedOrderList"];

export const ordersHandler = http.get("*/api/orders/", () => {
  const body: PaginatedOrderList = {
    count: 1,
    next: null,
    previous: null,
    results: [buildOrder()],
  };
  return HttpResponse.json(body);
});
```

The typecheck is the mechanism. `pnpm typecheck` fails on the renamed field,
inside the handler and inside the component, on the same run.
`references/openapi-schema-and-codegen.md` owns the command that regenerates
the types, and the gate that runs it in CI.

A generator can write the handler set from the same schema. Orval emits MSW
handlers beside its client, and `msw-auto-mock` emits them from an OpenAPI
document. Take one where the endpoint count makes the hand-written set a
maintenance cost. A generated handler set cannot drift from the contract,
because both come from one artifact.

### A named handler for each failure shape

The API produces more than one failure. Each one renders a different tree, so
each one needs a handler that a test can name.

| The response | What the interface must do | The handler |
| --- | --- | --- |
| 400 with a field dictionary | Put each message beside its own control | `fieldErrors` |
| 400 with `non_field_errors` | Put the message in the form-level region | `formError` |
| 401 | Refresh once, then send the reader to the sign-in route | `unauthorized` |
| 403 | State the permission, and never name a record the account may not read | `forbidden` |
| 404 | Render the not-found state of the route | `missing` |
| 409 | State the conflict, and offer the current value | `conflict` |
| 429 with `Retry-After` | Wait for the stated period, and never retry a POST with no idempotency key | `rateLimited` |
| 500 | Render the error state with a working retry control | `serverError` |
| A network error | Render the same error state, with no status code on the screen | `networkError` |
| A response past the deadline | Abort at the client deadline, and report the abort | `slowResponse` |

```ts
// Correct: src/test/msw/failures.ts. Each export is one shape, and each one
// is reusable across features.
import { HttpResponse, delay, http } from "msw";

export const fieldErrors = http.post("*/api/orders/", () =>
  HttpResponse.json(
    { quantity: ["Ensure this value is less than or equal to 10."] },
    { status: 400 },
  ),
);

export const rateLimited = http.post("*/api/orders/", () =>
  HttpResponse.json(
    { detail: "Request was throttled." },
    { status: 429, headers: { "Retry-After": "30" } },
  ),
);

export const networkError = http.get("*/api/orders/", () => HttpResponse.error());

export const slowResponse = http.get("*/api/orders/", async () => {
  await delay(15_000);
  return HttpResponse.json({ count: 0, next: null, previous: null, results: [] });
});
```

`references/boundary-validation-and-api-types.md` owns the shape of each DRF
body, and `references/api-client-and-request-safety.md` owns the
`normalizeApiError` that turns it into an `ApiError`. This file owns only the
handler that produces the body. The server side of every one of these responses
belongs to the backend.

The 400 test is the one that most suites omit. It is not optional. A field
message that reaches a toast rather than its control is a defect that only this
test finds. `references/form-submission-and-server-errors.md` states the rule
that the test asserts.

### The three pagination envelopes

DRF publishes page-number, limit-offset, and cursor pagination. The envelope
differs, and a handler that answers one style proves nothing about another.

```ts
// Correct: the helper builds the envelope, and the next value is a URL.
export function paginate<T>(
  results: readonly T[],
  { count = results.length, next = null as string | null } = {},
) {
  return { count, next, previous: null, results: [...results] };
}

export const ordersPageOne = http.get("*/api/orders/", () =>
  HttpResponse.json(paginate(pageOneRows, { count: 42, next: "/api/orders/?page=2" })),
);
```

The client follows the `next` URL and never computes an offset, which
`references/server-state-and-query-cache.md` states as a rule. A handler that
returns `next: null` on every page hides a client that computes the page
number. Give the first page a real `next` value, and assert that the second
request carries the query that the first response named.

A cursor endpoint carries no `count`. A table that reads `count` for its page
control breaks against it. Where the backend uses cursor pagination, the
handler returns no `count`, and the table test asserts the control that
`references/data-table-and-server-driven-state.md` requires in that case.

### One fixture for each auth state

```ts
// Correct: src/test/msw/auth.ts. Each export is a whole state, and a test
// takes one.
export const anonymous = [
  http.get("*/api/auth/session/", () => new HttpResponse(null, { status: 401 })),
];

export const authenticated = [
  http.get("*/api/auth/session/", () =>
    HttpResponse.json({ id: "u_1", email: "agent@example.test", roles: ["agent"] }),
  ),
];

export const expiredToken = [
  http.get("*/api/orders/", () => new HttpResponse(null, { status: 401 })),
  http.post("*/api/auth/refresh/", () => HttpResponse.json({ access: "fresh" })),
];

export const insufficientPermission = [
  http.get("*/api/orders/", () =>
    HttpResponse.json({ detail: "You do not have permission." }, { status: 403 }),
  ),
];
```

The `expiredToken` fixture is the one that proves the single-flight refresh.
Render a view that fires three concurrent requests, answer each with 401, and
assert that the refresh endpoint received one request and not three.
`references/session-and-token-lifecycle.md` states that rule, and it holds a
veto through the auth gate.

Never write a real credential into a fixture, and never read one from the
environment inside a test.
`references/secret-boundary-and-supply-chain.md` owns that rule, and it holds a
veto.

### The socket is mocked at the same boundary

MSW carries a `ws` namespace for a WebSocket connection. Confirm the link
signature and the event names against the installed version. This file states
the shape of the calls, and the installed package is the authority on its API.

```ts
// Correct: the link intercepts the connection, and the test drives the frames.
import { ws } from "msw";

const orders = ws.link("wss://*/ws/orders/");

export const orderStream = orders.addEventListener("connection", ({ client }) => {
  client.send(JSON.stringify({ type: "order.updated", data: buildOrder() }));
});
```

Three assertions prove the transport domain. The first is one connection after
a navigation, and never two. The second is a reconnect delay that grows between
attempts. The third is a degraded state on a close, and never an empty list.
`references/push-transport-and-connection.md` owns all three rules.

Three more assertions prove the merge domain. A malformed frame leaves the view
standing. An unnamed event type raises a counter rather than an exception. An
optimistic edit that the server echoes produces one row and not two.
`references/live-events-and-cache-merge.md` owns those three.

```ts
// Wrong: the test sends a frame that the schema accepts.
// Failure: the parse is never exercised. A production frame with a missing
// field reaches the cache, the view renders `undefined`, and no test moved.
client.send(JSON.stringify({ type: "order.updated", data: buildOrder() }));
```

```ts
// Correct: the test sends the frame that the backend can send by mistake.
client.send(JSON.stringify({ type: "order.updated", data: { id: null } }));
expect(await screen.findByRole("row", { name: /ORD-1001/ })).toBeVisible();
expect(droppedFrameCounter).toBe(1);
```

### The contract test

Two gates protect the contract, and they catch different failures.

The drift gate regenerates the types and compiles. It catches a renamed field,
a removed endpoint, and a changed type.
`references/openapi-schema-and-codegen.md` owns it, and `oasdiff` gates the
breaking change.

The contract test runs at run time. It catches a handler whose body the
application would reject, and a value that the schema permits and the parse
refuses.

```ts
// Wrong: nothing compares the handler body with the parse that production
// runs.
// Failure: the handler returns `total: 12`, the serializer returns
// `total: "12.00"`, and the Zod schema expects a string. Every component test
// passes on a number, and the first real response fails the parse.
export const ordersHandler = http.get("*/api/orders/", () =>
  HttpResponse.json({ count: 1, next: null, previous: null, results: [{ total: 12 }] }),
);
```

```ts
// Correct: every fixture passes the boundary parse that the application uses.
import { describe, expect, it } from "vitest";
import { orderSchema } from "@/features/orders/schema";
import { orderFixtures } from "@/test/factories/order";

describe("the order fixtures satisfy the boundary schema", () => {
  it.each(orderFixtures)("accepts %#", (fixture) => {
    expect(orderSchema.safeParse(fixture).success).toBe(true);
  });
});
```

The parse is the one that the application runs, imported and not copied.
`references/boundary-validation-and-api-types.md` owns it. A second copy of the
rules inside the test proves that the copy agrees with itself.

### When a test hits the real backend

Three cases justify a real Django instance, and no other case does.

- The contract test that proves the generated client against a running server,
  once for each release rather than once for each pull request.
- The end-to-end journey, which
  `references/end-to-end-journeys-and-flake-control.md` owns.
- A behavior that only the server produces, such as a database constraint.

The instance is seeded to a known state before the run. The seed is a Django
management command or a test-only endpoint, and the backend owns both. This file
owns only the call that the frontend makes to reach that state. It also owns the
rule that no frontend test writes to a shared instance.

Read the schema from the committed artifact, and never from a live address. A
build that reads a moving target does not repeat.

### The browser worker

`msw/browser` registers a service worker, and it serves the same handlers to a
real browser. It is the right tool for Storybook, where each story needs an
answer with no backend. Playwright has its own interception, and
`references/end-to-end-journeys-and-flake-control.md` states when each one
applies. The worker file is a generated asset under `public/`. Regenerate it
after an MSW upgrade, because a stale worker and a new library disagree in
silence.

### The libraries

The table gives each package its rule and its maintenance status. This skill
cannot confirm a version number or a release date for them, so no cell states
one. Read the installed version from `package.json` before you write code.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `msw` | Version 2 or later. The `http` namespace for a request, the `ws` namespace for a socket, and `HttpResponse` for the body. | Current | Current | Mock Service Worker, active | None |
| Recommend | `msw/node` | The server for the Vitest run, under `onUnhandledRequest: "error"`. | Ships with `msw` | Current | Mock Service Worker, active | None |
| Recommend | `msw/browser` | The worker for Storybook, and for a browser run that needs the same handlers. | Ships with `msw` | Current | Mock Service Worker, active | None |
| Conditional | Orval mock generation | Where the endpoint count makes a hand-written handler set a maintenance cost. It emits the handlers beside the client, from one schema. | Current | Current | Orval team, active | None |
| Conditional | `msw-auto-mock` | The same purpose, for a project that generates its client another way. | Current | Current | Community, active | None |
| Conditional | A seeded Django instance | Only for the release contract test, and for the end-to-end journey. The backend owns the seed. | — | — | — | — |
| Reject | A stub of the API client module | It skips the parse, the deadline, the header, and the error normalizer. | — | — | — | — |
| Reject | `nock` in a browser-facing suite | It intercepts the Node HTTP layer, which the browser path does not use. | — | — | — | — |
| Reject | `fetch-mock` beside MSW | Two interception layers in one suite give two answers for one request. | — | — | — | — |
| Reject | A hand-copied response body from a browser tab | It is a snapshot of one moment, and nothing holds it to the schema. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| The suite passes locally and times out in CI | An unhandled request reaches a developer's Django | Run with the network disabled | Set `onUnhandledRequest: "error"` |
| One test passes alone and fails after another | A `server.use()` override survives the test | Run the file with a reversed order | Call `server.resetHandlers()` in `afterEach` |
| Half the requests fail the error policy | The handler path holds one base URL | Read the unhandled path in the failure | Open the path pattern with `*` |
| Every test passes and production reads `undefined` | The handler body is an untyped literal | Rename a field in the schema and run the typecheck | Type the body from `components["schemas"]` |
| The parse throws on the first real response | The fixture holds a number where the API sends a string | Run the fixtures through the boundary schema | Add the contract test over the fixtures |
| The second page request never fires | Every handler returns `next: null` | Read the request log of the test | Give page one a real `next` URL |
| The refresh endpoint receives three requests | The single-flight guard is absent | Answer three concurrent calls with 401 | Fix the client, and keep the assertion |
| A frame with a missing field crashes the view | No test sends a malformed frame | Send `{ id: null }` from the mock socket | Parse every frame, and count the dropped frame |
| The worker serves a stale response after an upgrade | The generated worker file predates the library | Compare the worker version against `package.json` | Regenerate the worker asset |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| MSW 2 replaced `rest` with `http` | `rg -n 'from "msw"' src/ \| rg 'rest'` reports a hit | Import `http`, and match the method name |
| MSW 2 replaced `res(ctx.json())` with a returned `HttpResponse` | `rg -n 'ctx\.(json\|status)' src/` reports a hit | Return `HttpResponse.json(body, { status })` |
| MSW 2 replaced `setupServer` request arguments with a `{ request }` object | A handler that reads `req.body` | Read `await request.json()` |
| The default unhandled policy is a warning | `rg -n 'server\.listen\(\)' src/` reports a hit | Pass `{ onUnhandledRequest: "error" }` |
| The generated worker asset ages with the library | The worker version and `package.json` disagree | Regenerate the asset after each upgrade |
| OpenAPI 3.0.3 ignores `nullable` beside a `$ref` | A fixture that sends `null` where the type promises a value | Add `.nullable()` in the boundary schema, which `references/boundary-validation-and-api-types.md` owns |

## Verification

```bash
# 1. Read the installed versions before you write code.
rg -n '"msw"|"zod"|"openapi-typescript"' package.json

# 2. Confirm that an unhandled request fails the run.
rg -n 'onUnhandledRequest' src/test/

# 3. Confirm that the handlers reset after each test.
rg -n 'resetHandlers' src/test/

# 4. Find a handler that hard-codes a base URL. Read every hit.
rg -n 'http\.(get|post|patch|put|delete)\("' src/test/ | rg -v '"\*'

# 5. Find an untyped handler body. Read every hit.
rg --files-with-matches 'HttpResponse.json' src/test/ \
  | xargs rg --files-without-match 'components\["schemas"\]'

# 6. Confirm that a handler exists for each failure shape.
rg -n 'status: (400|401|403|404|409|429|500)' src/test/msw/

# 7. Find a legacy MSW 1 idiom. This prints nothing.
rg -n 'ctx\.json|ctx\.status|from "msw".*\brest\b' src/

# 8. Find a real credential in a fixture. This prints nothing.
rg -in 'password|secret|token: "[a-z0-9]{16}' src/test/

# 9. Run the contract test over the fixtures.
pnpm vitest run src/test/contract

# 10. Regenerate the types, then compile. An error is drift.
pnpm api:generate && pnpm typecheck

# 11. Run the suite with the network unavailable. It still passes.
pnpm test

# 12. Confirm that the mock socket sends a malformed frame in at least one
#     test.
rg -n 'client\.send' src/
```

## Review checklist

- [ ] Does one server serve the whole project, started in one setup file?
- [ ] Is `onUnhandledRequest` set to `error`?
- [ ] Does `resetHandlers()` run after each test?
- [ ] Does every handler path open with `*`, so both base URLs match?
- [ ] Does every handler body carry a type from the generated schema?
- [ ] Does the project hold a named handler for the 400 field dictionary, the
      401, the 403, and the 404?
- [ ] Does it also hold one for the 409, the 429, the 500, the network error,
      and the slow response?
- [ ] Does a test assert that a 400 field message lands beside its control?
- [ ] Does the first page of a paginated handler carry a real `next` URL?
- [ ] Does the project hold a fixture for the anonymous, the authenticated, the
      expired, and the refused state?
- [ ] Does a test prove that three concurrent 401s send one refresh request?
- [ ] Is every fixture free of a real credential?
- [ ] Does a mock socket test send a malformed frame, and does the view survive
      it?
- [ ] Does a contract test run every fixture through the boundary parse that
      the application uses?
- [ ] Is the boundary schema imported into the test, rather than copied?
- [ ] Does every real backend run against a seeded, isolated state?
- [ ] Does the schema come from the committed artifact, rather than a live
      address?
- [ ] Is no module of this repository replaced by a stub?

## Handoffs

- The level that a test belongs to, the provider helper, the factory, and the
  determinism rules → `references/test-strategy-and-component-tests.md`.
- The interception inside a real browser, the stored sign-in state, and the
  seeded journey → `references/end-to-end-journeys-and-flake-control.md`.
- The gate order and the coverage report in CI →
  `references/merge-gates-and-quality-signals.md`.
- The schema artifact, the generator, the drift gate, and `oasdiff` →
  `references/openapi-schema-and-codegen.md`.
- The DRF envelope shapes, the `nullable` trap, and the boundary parse →
  `references/boundary-validation-and-api-types.md`.
- `normalizeApiError`, the deadline, the retry rule, and the idempotency key →
  `references/api-client-and-request-safety.md`.
- The two base URLs and the proxy Route Handler →
  `references/cross-origin-and-bff-proxy.md`.
- The query key, the `next` URL, and the optimistic write →
  `references/server-state-and-query-cache.md`.
- The session, the single-flight refresh, and the cookie attributes →
  `references/session-and-token-lifecycle.md`.
- The connection count, the reconnect delay, and the close code →
  `references/push-transport-and-connection.md`.
- The malformed frame, the unnamed event type, and the merge into the cache →
  `references/live-events-and-cache-merge.md`.
- The field error that must land beside its control →
  `references/form-submission-and-server-errors.md`.
- The credential that must never reach a fixture →
  `references/secret-boundary-and-supply-chain.md`. That domain holds a veto.
- The serializer, the status code, the pagination class, and the error envelope
  on the server → the backend. This file changes nothing on the server.
- The seed command, the factory on the server, and the transactional reset → the
  backend.
