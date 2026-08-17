# Live events and the cache merge

Zod 4, TanStack Query 5.101 or later, React 19.2.6 or later, against a Django
and DRF backend. This file owns the message that arrives on a push connection,
and what that message changes on the screen. The subjects are the event
envelope, the parse over it, and the type that this client cannot name. They
also include the write into the query cache and the flood over it. The last two
are the echo of a local change, and the drift that ends a feed in silence.

The connection that carries the message is
`references/push-transport-and-connection.md`. The cache that the message
writes into is `references/server-state-and-query-cache.md`. The parse over
every value that enters the program is
`references/boundary-validation-and-api-types.md`.

## Principle

A message from the network is a value from outside the program. The transport
proves nothing about its shape.

A feed that stops on one bad message is a feed that the server can end by
accident. One frame must never take the others with it.

A message that the client cannot name is a message from a newer server. Count
it, and let it pass.

Pushed data is the same data that a request produces. It belongs in the cache
entry that the request filled, and never in a second copy beside it.

A change that this client made comes back from the server. Two copies of one
change is a defect, and not a confirmation.

Data can arrive faster than a screen can paint. The screen then sets the rate,
and the connection does not.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The envelope, and who owns each half

Every message carries one envelope, and the envelope holds four fields. `type`
is the discriminator, and `id` names the event. `origin` names the client that
caused the event, and `payload` holds the record.

| The part | Who owns it |
| --- | --- |
| The envelope, its four fields, and the parse over them | This file |
| The fields inside `payload` | Domain 05, through `references/openapi-schema-and-codegen.md` and the generated types |
| The version and the deprecation of an event type | The sibling skill `django-api-contract` |
| The consumer that sends the event | The sibling skill `django-async-jobs` |

Type the payload from the generated types, so a renamed serializer field
becomes a compile error rather than a silent gap.
`references/boundary-validation-and-api-types.md` owns the rule that a
generated type states the shape and a Zod schema proves it.

### Every frame is parsed, and an unnamed type is counted

```ts
// Wrong: JSON.parse straight into the cache.
// Failure: one malformed frame throws inside the handler. The handler dies,
// the connection stays open, and the feed stops with no error on the screen. A
// frame of a type that this build does not know writes undefined over rows
// that were correct.
socket.onmessage = (event) =>
  queryClient.setQueryData(orderKeys.lists(), JSON.parse(event.data));
```

```ts
// Correct: src/features/orders/api/events.ts holds a discriminated union, a
// safeParse, and a counter for what this build cannot name.
import { z } from "zod";
import type { QueryClient } from "@tanstack/react-query";
import { assertNever } from "@/lib/assert-never";
import { ORIGIN_ID } from "@/lib/realtime/origin";
import { orderKeys } from "@/features/orders/api/orders";
import { OrderSchema } from "@/features/orders/api/schema";

const OrderEvent = z.discriminatedUnion("type", [
  z.object({
    type: z.literal("order.updated"),
    id: z.string(),
    origin: z.string().nullable(),
    payload: OrderSchema,
  }),
  z.object({
    type: z.literal("order.deleted"),
    id: z.string(),
    origin: z.string().nullable(),
    payload: OrderSchema.pick({ id: true }),
  }),
]);

let unnamedEvents = 0;

export function unnamedEventCount(): number {
  return unnamedEvents;
}

export function applyOrderEvent(frame: string, queryClient: QueryClient): void {
  let json: unknown;
  try {
    json = JSON.parse(frame);
  } catch {
    unnamedEvents += 1;
    return; // a malformed frame never throws out of the handler
  }

  const parsed = OrderEvent.safeParse(json);
  if (!parsed.success) {
    unnamedEvents += 1;
    return; // a type this build cannot name is news, and not a fault
  }

  const event = parsed.data;
  if (event.origin === ORIGIN_ID) return; // this client already holds the change

  switch (event.type) {
    case "order.updated":
      queryClient.setQueryData(orderKeys.detail(event.payload.id), event.payload);
      queryClient.invalidateQueries({ queryKey: orderKeys.lists() });
      return;
    case "order.deleted":
      queryClient.removeQueries({ queryKey: orderKeys.detail(event.payload.id) });
      queryClient.invalidateQueries({ queryKey: orderKeys.lists() });
      return;
    default:
      return assertNever(event);
  }
}
```

Three rules bind this handler. `JSON.parse` runs inside a `try`, because a
proxy or a heartbeat can put a value that is not JSON on the connection. The
`safeParse` returns rather than throws, so one frame never ends the feed. The
counter is the only record that a frame was dropped, so a metric must read it.

The `default` branch calls `assertNever`, and it never runs. Zod rejects every
type outside the union before the switch, so an unnamed type returns at the
parse. `assertNever` is therefore a gate at compile time: it fails the build
when the union gains a member and the switch does not.
`references/type-modeling-and-narrowing.md` owns that function, and
`references/boundary-validation-and-api-types.md` owns the Zod calls.

### The write into the cache

`references/server-state-and-query-cache.md` owns the key factory, and it owns
the choice between `setQueryData` and `invalidateQueries` for a mutation. This
table states only what a pushed event changes about that choice.

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| The payload carries the whole record | `setQueryData` on the detail key. Add an invalidation of the list prefix where the server computes the order or a total. | The payload omits a field that a view reads, which the next row covers. | The cache holds a record that no request proved, until the entry goes stale. |
| The event names a record and carries no fields, or it changes a derived value | `invalidateQueries` on the prefix that the event touched | The payload grows to the whole record, which the row above covers. | One request for each event, so a flood of events becomes a flood of requests. |
| The events arrive faster than one for each frame of the display | Collect the frames, and write once for each frame | The rate falls under one event for each frame. | One frame of delay, and a buffer that a hidden tab holds until the return. |
| The event is the echo of a change that this client sent | Drop it, on the `origin` field | The client sends no optimistic write, so the echo is the first news of the change. | An id on every mutation, and a backend that echoes it on the event. |
| No view on the screen reads the subject of the event | Count it, and do nothing | A view starts to read that subject. | The counter is the only record, so a later reader must trust it. |

```ts
// Wrong: the cache write follows the invalidation.
// Failure: invalidateQueries marks the entry stale, and the setQueryData that
// follows clears that mark. The refetch never runs, so the total that the
// server computes keeps the value it held before the event.
queryClient.invalidateQueries({ queryKey: orderKeys.lists() });
queryClient.setQueryData(orderKeys.detail(id), payload);
```

```ts
// Correct: write first, and invalidate last.
queryClient.setQueryData(orderKeys.detail(id), payload);
queryClient.invalidateQueries({ queryKey: orderKeys.lists() });
```

An event writes into the entry that the request filled, under the key that
`references/server-state-and-query-cache.md` defines. NEVER hold the pushed
rows in a `useState` beside the cache. Two copies answer one question in two
ways, and only one of them refetches.

An async iterable that must reach the cache as one growing entry, such as the
tokens of a model, has a wrapper in TanStack Query.
`references/push-transport-and-connection.md` lists it in the library table,
with its experimental import name.

### The flood

```ts
// Wrong: one cache write for each event.
// Failure: a price feed sends 40 events each second. Each setQueryData starts
// a render, so the table commits 40 times each second. The scroll stutters,
// and typing in the filter above the table drops frames.
socket.onmessage = (event) => applyOrderEvent(event.data, queryClient);
```

```ts
// Correct: src/lib/realtime/coalesce.ts collects the frames, and it writes
// once for each frame of the display.
export function createCoalescer(flush: (frames: string[]) => void) {
  let buffer: string[] = [];
  let handle: number | null = null;

  return (frame: string) => {
    buffer.push(frame);
    if (handle !== null) return;
    handle = requestAnimationFrame(() => {
      handle = null;
      const batch = buffer;
      buffer = [];
      flush(batch);
    });
  };
}
```

Coalesce once the event rate passes about one event for each frame of the
display, which is about one every 16 ms at 60 Hz. No specification sets that
number, and it is a review threshold rather than a rule. Read the commit count
in the React Profiler before and after the change, and record the two numbers.

A hidden tab paints nothing, so `requestAnimationFrame` does not run and the
buffer holds. That behavior agrees with the rule in
`references/push-transport-and-connection.md` that a hidden tab stops its
connection.

### The echo of a change that this client made

```ts
// Wrong: the client applies its own change twice.
// Failure: onMutate writes the row optimistically, and the server broadcasts
// the same change to every connection. This client receives its own change and
// appends the row again, so the list shows the record twice until a refetch
// removes one of them.
useMutation({
  mutationFn: setOrderPaid,
  onMutate: async (order) => {
    /* the snapshot and the optimistic write */
  },
});
```

```ts
// Correct: src/lib/realtime/origin.ts names this tab, the mutation sends the
// name, and the handler drops the event that carries it back.
"use client"; // reason: one id for one browser tab, and never for a request
export const ORIGIN_ID = crypto.randomUUID();
```

```ts
// The mutation carries the id, and the backend echoes it on the event.
mutationFn: (order: Order) =>
  api.PATCH("/api/orders/{id}/", {
    params: { path: { id: order.id } },
    body: { paid: true },
    headers: { "X-Origin-Id": ORIGIN_ID },
  }),
```

The value is created once for each tab, and no component renders it, so it
costs no hydration mismatch. `references/server-state-and-query-cache.md` owns
the optimistic update itself, with its snapshot, its rollback, and its
reconciliation at the end. This file owns only the echo that a push transport
adds to it.

Ask the backend team for the echo field before you write the optimistic path.
Where the backend cannot echo the id, drop the optimistic write and let the
event be the first news of the change. Two answers on the screen are worse than
one answer that arrives late.

### The resync after a gap

Every reconnect leaves a gap, and the events of that gap are lost. The
connection reports `open` again, and the cache holds the state of the moment
before the drop.

```ts
// Correct: the resync runs on open, and it names the subjects that the feed
// writes.
function onOpen(queryClient: QueryClient): void {
  queryClient.invalidateQueries({ queryKey: orderKeys.all });
}
```

Server-sent events need less. The browser sends the last `id:` back in the
`Last-Event-ID` header. A server that numbers its events therefore replays the
gap, and no refetch runs. A WebSocket carries no such mechanism, so the client
refetches instead. `references/push-transport-and-connection.md` owns both facts about
the transport.

Invalidate the prefix that the feed writes, and never the whole cache. An
unbounded invalidation on every reconnect turns a flapping network into a
request storm, and `references/server-state-and-query-cache.md` states that
rule for a mutation.

### What breaks when the contract drifts

| The change on the server | What the frontend sees |
| --- | --- |
| An event `type` string is renamed | Every frame of that type fails the parse. The counter rises, the screen stops updating, and no error appears. |
| A payload field is renamed, or its type changes | The frames of that one type fail. The other types keep working, so the defect looks like one broken view. |
| A new event type ships | The counter rises and nothing breaks. This is the design, and it is the reason an unnamed type never throws. |
| The `origin` field leaves the envelope | Every optimistic write meets its own echo, so rows appear twice. |
| The channel layer is down | The server raises on the fan-out. No event arrives, the connection stays open, and the view must show degraded rather than empty. |
| The envelope gains a required field | Every frame fails the parse at once, and the feed stops in full. |

The counter is the detector for the first three rows. Send it to the metric
that domain 21 `observability-and-resilience` owns, and alarm on a rise. A
contract test against the schema catches the first two rows before a deploy,
and domain 20 `testing-and-quality` owns that test.

### Version discipline

Read the installed versions before you write code.

TanStack Query 5.101 is the floor of this stack. The order of the two calls
above is a behavior of version 5. `setQueryData` clears the invalidated mark
that `invalidateQueries` sets. The write therefore goes first, and the
invalidation goes last.

Zod 4 is the pin. `z.discriminatedUnion` and `safeParse` are the calls that
this file needs. `references/boundary-validation-and-api-types.md` lists the
Zod 3 calls that Zod 4 replaced, and every one of them fails at run time.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

## Verification

```bash
# 1. Read the installed versions before you write code.
node -p "require('@tanstack/react-query/package.json').version"
node -p "require('zod/package.json').version"

# 2. Find a raw parse into state or into the cache. This must print nothing.
rg -n 'JSON\.parse\((event|e|msg)\.data\)' src/

# 3. Find a cache write from a message handler with no parse beside it.
rg -n -B6 'setQueryData|invalidateQueries' src/ | rg -i 'onmessage|addEventListener\("message"'

# 4. Find the event schema and the counter. Both must exist.
rg -n 'discriminatedUnion|safeParse' src/features/*/api/events.ts
rg -n 'unnamedEventCount|unnamedEvents' src/

# 5. Find an invalidation that runs before a write on the same subject.
rg -n -A3 'invalidateQueries' src/ | rg 'setQueryData'

# 6. Find pushed rows held beside the cache. Read every hit.
rg -n -B4 'useState' src/ | rg -i 'event|message|socket|feed'

# 7. Find an unbounded invalidation on a reconnect. This must print nothing.
rg -n 'invalidateQueries\(\)' src/

# 8. The typecheck proves the switch against the union.
pnpm typecheck

# 9. Confirm that a malformed frame does not end the feed. Send a frame that
#    is not JSON, and read the view. It still renders, and the counter rises.

# 10. Confirm that an unnamed type is ignored. Send {"type":"not.known"}, and
#     read the view. Nothing changes, and the counter rises.

# 11. Confirm that no row appears twice. Make an optimistic edit with the
#     connection open, and count the rows. There is one.

# 12. Confirm the resync. Drop the connection, change a record on the backend,
#     restore the connection, and read the view. It holds the new value.
```

## Review checklist

- [ ] Does every inbound frame pass through `safeParse` before it reaches the
      cache or the state?
- [ ] Is `JSON.parse` inside a `try`, so a frame that is not JSON returns
      rather than throws?
- [ ] Is the envelope a discriminated union on `type`?
- [ ] Does the payload type come from the generated types?
- [ ] Is a type that this build cannot name ignored and counted, and never
      thrown?
- [ ] Does a counter record every dropped frame, and does a metric read it?
- [ ] Does the union `switch` end in a `default` that calls `assertNever`?
- [ ] Does each event write with `setQueryData`, or invalidate a prefix, under
      the rule in the table?
- [ ] Does every write run before the invalidation of the same subject?
- [ ] Are the pushed rows in the query cache, and never in a `useState` beside
      it?
- [ ] Does a feed above one event for each frame coalesce into one write?
- [ ] Is the coalesce threshold recorded with a commit count from the React
      Profiler?
- [ ] Does every mutation carry an origin id, and does the handler drop its own
      echo?
- [ ] Does the reconnect invalidate the prefix that the feed writes, rather
      than the whole cache?
- [ ] Does an empty view under a broken connection render degraded rather than
      empty?

## Handoffs

- The transport, the handshake, the reconnect, the heartbeat, the close code,
  and the degraded state → `references/push-transport-and-connection.md`.
- The key factory, the cache times, the mutation and its invalidation, the
  optimistic update, and the four states of a data view →
  `references/server-state-and-query-cache.md`.
- The parse doctrine, the Zod 4 calls, the DRF envelopes, and the nullable
  field → `references/boundary-validation-and-api-types.md`.
- `assertNever`, the discriminated union, and the branded id →
  `references/type-modeling-and-narrowing.md`.
- The generated `paths` and `components` types behind the payload →
  `references/openapi-schema-and-codegen.md`.
- The one typed client that the mutation sends through, and
  `normalizeApiError` → `references/api-client-and-request-safety.md`.
- The state that no query owns, and the effect that subscribes →
  `references/state-and-effects.md`.
- The identity that scopes a key, and the cache that a logout clears →
  `references/session-and-token-lifecycle.md` and
  `references/route-protection-and-permissions.md`.
- The row that a live table adds or removes while the user reads it →
  `references/data-table-and-server-driven-state.md`.
- The progress event of an upload or a download →
  `references/file-upload-and-transport.md`.
- The politeness of an announcement when a row arrives →
  `references/keyboard-focus-and-live-regions.md`. That domain holds a veto.
- The words of a live notification, and the channel that carries it →
  `references/error-and-empty-state-copy.md`.
- The render cost of a high-frequency feed →
  `references/paint-and-interaction-cost.md`. The budget over it →
  `references/performance-budgets-and-measurement.md`.
- The metric behind the counter, and the alarm on it → domain 21
  `observability-and-resilience`. Not integrated yet.
- The MSW `ws` handler, the malformed-frame test, and the unnamed-type test →
  domain 20 `testing-and-quality`. Not integrated yet. Three assertions prove
  this domain. They are a view that survives a malformed frame, a counter that
  rises on an unnamed type, and one row after an optimistic edit.
- The event envelope as a published surface, its version, and its deprecation →
  the sibling skill `django-api-contract`.
- The consumer that sends the event, and the idempotency of the work behind it
  → the sibling skill `django-async-jobs`.
- The payload cost of a feed → the sibling skill
  `django-performance-optimizer`.
