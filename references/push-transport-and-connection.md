# Push transport and the connection

Next.js 16.3, React 19.2.6 or later, Django Channels 4.3, against a Django and
DRF backend. This file owns the decision to push data to the browser, and the
connection that carries it. The subjects are the transport choice, the
credential on the handshake, and the lifetime of the connection. They also
include the reconnect, the heartbeat, the close code, and the state that a
dropped connection renders.

The message that arrives on the connection is
`references/live-events-and-cache-merge.md`. The poll that most features take
instead is `references/server-state-and-query-cache.md`. The credential itself,
its storage, and its refresh are
`references/session-and-token-lifecycle.md`.

## Principle

A held connection is a resource that each user occupies for the life of the
tab. State what it buys before you open it. Most data that must stay fresh
stays fresh under a request that repeats.

A connection that the network can drop will be dropped. Design the second
connection before you write the first.

A retry at a fixed period turns one outage into a flood. Every client fails
together, so every client must return at a different moment.

A connection with no traffic looks the same as a connection that is gone. Send
something, or the application cannot tell the two apart.

A transport failure is a fact about the network. It is not a statement about
the identity of the user.

A view that renders an empty state when the connection drops reports a lie. It
says "no data", and the truth is "no news".

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### Choose the transport, and write the reason

The first row that matches decides the transport.

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| The data must stay fresh, and no user watches one value change second by second | The poll and the refetch on focus of `references/server-state-and-query-cache.md`. This is the default. | A user does watch one value change second by second, which the next three rows cover. | One request for each period, for each client. |
| The server sends and the client does not, the payload is text, and an update arrives every few seconds | Server-sent events, through the native `EventSource` or a fetch-based source | The client must also send on the same channel, so the WebSocket row applies. | The request is a GET, and the native `EventSource` sets no header. |
| Both sides send, and the delay must stay under one second | A WebSocket against Django Channels | The data travels one way only, so server-sent events are simpler and the proxy needs less. | You write the reconnect, the heartbeat, and the resync, and the proxy needs the `Upgrade` directives. |
| One long response arrives in parts, such as tokens, an export, or a report | A streamed `fetch`, read through `ReadableStream` and `TextDecoderStream`, in NDJSON | The feed is continuous rather than one response, so the two rows above apply. | You write the parser for a partial line. |
| A background job reports its state | Poll the status endpoint. Subscribe instead where a connection is already open for another reason. | The job runs for minutes and the user watches every step. | One request for each period, for each client that watches the job. |
| More than six streams are needed on one origin, over HTTP/1.1 | One connection that carries every subject, or a WebSocket | HTTP/2 serves the origin, which the next row covers. | One envelope that every subject shares, and a fan-out in the client. |
| More than six streams are needed, and HTTP/2 serves the origin | Server-sent events still serve. The multiplex removes the limit. | The origin falls back to HTTP/1.1 behind a proxy that does not negotiate HTTP/2. | You must confirm the protocol in production, and not in development alone. |
| The user opens many tabs of the same application | One tab holds the connection, and it fans out over `BroadcastChannel` | The connection count for a normal user stays inside the limit. | A leader election that the application writes, and a handover when that tab closes. |
| Unreliable datagrams are needed, for the latest browsers only | WebTransport. Audit-only for new work. | Never, while an older browser must work. | The specification is a W3C Working Draft, and the application needs a second transport beside it. |

Two numbers set the sixth and the seventh rows. A browser opens up to six
connections for each domain over HTTP/1.1, and the seventh stream waits for one
of the six to close. That limit belongs to the browser and not to the tab. Over
HTTP/2 the limit is the concurrent stream count instead. RFC 7540 section 6.5.2
recommends that count to be no smaller than 100, and Chrome and Firefox both
default to 100.

NEVER open a `new WebSocket` or a `new EventSource` with no written reason.
Put one line in the pull request. It answers one question: why not a 30-second
poll, or a refetch on focus?

### The credential travels on the handshake

`references/session-and-token-lifecycle.md` owns where each credential lives.
This file owns the carrier that the handshake takes.

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| The frontend and Django answer on one origin | The cookie, which the browser sends on the handshake with no code | The two origins separate, which the next two rows cover. | The server must check the `Origin` header, because the handshake is not bound by the Same-Origin Policy. |
| The origins differ, and the client can send before it subscribes | A token in the first message after `open`. The server holds the connection unauthenticated until it validates the token. | The client must subscribe in the same frame, which the next row covers. | A short window where the connection is open and not authorized, and a server that must reject every other message inside it. |
| The origins differ, and the client cannot send a first message | The `Sec-WebSocket-Protocol` subprotocol value, which `new WebSocket(url, protocols)` sets | A proxy on the path rewrites or removes the subprotocol. | The header carries a value that is not a subprotocol, so the choice needs a comment. |
| The backend offers a query parameter and nothing else | A single-use ticket with a short life, and a comment that names the risk | Never. This is the last resort. | The URL reaches every access log on the path, so the ticket must expire in seconds. |

```ts
// Wrong: the access token rides in the query string.
// Failure: the URL reaches the access log of every proxy on the path, the
// Django log, and the history of the browser. A log that anyone may read now
// holds a credential that is valid for the whole life of the token.
const socket = new WebSocket(`wss://api.example.com/ws/feed/?token=${accessToken}`);
```

```ts
// Correct: src/lib/realtime/connect.ts asks for a ticket over HTTPS, and the
// ticket is valid once and for seconds.
import { requestWithAuth } from "@/lib/auth/refresh";
import { TicketSchema } from "@/lib/realtime/schema";

export async function connectWithTicket(url: string): Promise<WebSocket> {
  const response = await requestWithAuth("/api/realtime/ticket/", { method: "POST" });
  const ticket = TicketSchema.parse(await response.json());
  const socket = new WebSocket(url);
  socket.addEventListener("open", () => {
    socket.send(JSON.stringify({ type: "auth", ticket: ticket.value }));
  });
  return socket;
}
```

The cookie carrier depends on one server-side control. A WebSocket handshake is
an ordinary HTTP request, and the Same-Origin Policy does not stop it. A cookie
alone therefore authenticates any page that opens the connection. The backend
must check the `Origin` header on the handshake. The backend's security review
owns that check, and the backend owns the consumer that runs behind it. State the
requirement in the review, and fail the review where nobody can confirm it.

### One connection for the application

```tsx
// Wrong: the component body opens the connection.
// Failure: a new connection opens on every render, and no cleanup closes any
// of them. Strict Mode mounts twice in development, so the first connection
// reports "closed before the connection is established". In production a
// list of 50 rows opens 50 connections, and the sixth one waits.
function Feed() {
  const socket = new WebSocket("wss://api.example.com/ws/feed/");
  socket.onmessage = (event) => setItems(JSON.parse(event.data));
  return <FeedList items={items} />;
}
```

```tsx
// Correct: src/lib/realtime/socket-provider.tsx holds one connection in a ref,
// and the effect returns the cleanup that closes it.
"use client"; // reason: a WebSocket is a browser API, and the status is client state
import { createContext, useContext, useEffect, useRef, useState } from "react";
import { backoffDelay } from "@/lib/realtime/backoff";

export type ConnectionStatus = "connecting" | "open" | "reconnecting" | "failed";
type Listener = (frame: string) => void;

type SocketApi = {
  subscribe: (fn: Listener) => () => void;
  send: (message: unknown) => void;
};

const SocketContext = createContext<SocketApi | null>(null);
const StatusContext = createContext<ConnectionStatus>("connecting");

export function SocketProvider({
  url,
  children,
}: {
  url: string;
  children: React.ReactNode;
}) {
  const socketRef = useRef<WebSocket | null>(null);
  const listeners = useRef(new Set<Listener>());
  const [status, setStatus] = useState<ConnectionStatus>("connecting");
  // One stable object, created once, that reads the two refs.
  const [api] = useState<SocketApi>(() => ({
    subscribe: (fn) => {
      listeners.current.add(fn);
      return () => {
        listeners.current.delete(fn);
      };
    },
    send: (message) => socketRef.current?.send(JSON.stringify(message)),
  }));

  useEffect(() => {
    let timer: ReturnType<typeof setTimeout> | undefined;
    let attempt = 0;
    let closedByUs = false;

    const connect = () => {
      const socket = new WebSocket(url);
      socketRef.current = socket;
      socket.onopen = () => {
        attempt = 0;
        setStatus("open");
      };
      socket.onmessage = (event: MessageEvent<string>) => {
        listeners.current.forEach((fn) => fn(event.data));
      };
      socket.onclose = (event: CloseEvent) => {
        if (closedByUs) return;
        if (event.code === 1008 || event.code === 4001 || event.code === 4003) {
          setStatus("failed"); // an answer about this identity. NEVER retry it.
          return;
        }
        setStatus("reconnecting");
        timer = setTimeout(connect, backoffDelay(attempt));
        attempt += 1;
      };
    };

    connect();
    return () => {
      closedByUs = true;
      clearTimeout(timer);
      socketRef.current?.close(1000, "unmount");
      socketRef.current = null;
    };
  }, [url]);

  return (
    <StatusContext value={status}>
      <SocketContext value={api}>{children}</SocketContext>
    </StatusContext>
  );
}

export function useConnectionStatus(): ConnectionStatus {
  return useContext(StatusContext);
}

export function useSocket(): SocketApi {
  const api = useContext(SocketContext);
  if (api === null) throw new Error("useSocket outside SocketProvider");
  return api;
}
```

Four rules bind the provider. The connection lives in a ref, so a render never
creates a second one. The two contexts are separate, so a component that only
sends does not re-render on each status change, and
`references/state-and-effects.md` owns that split. The effect returns a cleanup
that closes, which Strict Mode proves in development. Mount the provider once,
above every consumer, and never inside a list row.

`references/state-and-effects.md` owns the general rule for an effect and its
cleanup. It also owns `useEffectEvent`, which reads a value that must not start
a second connection. This file owns the connection that those rules govern.

### The reconnect spreads its attempts

```ts
// Wrong: a timer reconnects at a fixed period.
// Failure: the backend restarts, every client of the application returns at
// the same second, and the handshakes arrive as one flood. The backend fails
// again, and the flood repeats for as long as the outage lasts.
setInterval(() => {
  if (socket.readyState === WebSocket.CLOSED) socket = new WebSocket(url);
}, 1_000);
```

```ts
// Correct: src/lib/realtime/backoff.ts doubles the delay, caps it, and spreads
// the clients across the window.
const BASE_MS = 1_000;
const CAP_MS = 30_000;

export function backoffDelay(attempt: number): number {
  const ceiling = Math.min(BASE_MS * 2 ** attempt, CAP_MS);
  return ceiling / 2 + Math.random() * (ceiling / 2);
}
```

Half of the delay is fixed and half is spread, so two clients that fail
together return at two different moments. `Math.random()` is correct here,
because this function runs in a timer and never in a render.
`references/state-and-effects.md` states the render rule that it would break.

Every reconnect leaves a gap, and the messages of that gap are lost. Resync on
`open`, and `references/live-events-and-cache-merge.md` owns the invalidation
that the resync sends. The native `EventSource` reconnects on its own and
sends the `Last-Event-ID` header, so the server can replay the gap. A WebSocket
has no such mechanism, so the client refetches instead.

### The heartbeat proves that the connection is alive

A TCP connection can die with no close frame. `readyState` then stays `OPEN`,
nothing arrives, and the interface looks correct. Only traffic separates that
state from a quiet feed.

```ts
// Correct: src/lib/realtime/heartbeat.ts sends an application message, because
// the browser WebSocket API sends no protocol ping frame.
const PERIOD_MS = 25_000;

export function createHeartbeat(socket: WebSocket) {
  let answered = true;
  const timer = setInterval(() => {
    if (!answered) {
      socket.close(4000, "no answer"); // the watchdog forces a reconnect
      return;
    }
    answered = false;
    socket.send(JSON.stringify({ type: "ping" }));
  }, PERIOD_MS);

  return {
    onPong: () => {
      answered = true;
    },
    stop: () => clearInterval(timer),
  };
}
```

Start the heartbeat on `open` inside the provider effect. Stop it in the
cleanup of that effect, and stop it again on each close. The message handler
calls `onPong()` when the parsed event type is `pong`, and
`references/live-events-and-cache-merge.md` owns that parse.

Set `PERIOD_MS` under the shortest idle timeout on the path. The Nginx
`proxy_read_timeout` default is 60 seconds. Cloudflare closes an idle
connection at about 100 seconds on every plan below Enterprise. A period of 25
seconds sits under both, and it leaves room for one lost answer.

The watchdog closes when one period passes with no answer. RFC 6455 reserves
the range 4000 to 4999 for the application, and this application takes 4000 for
the watchdog. That code is not one of the three that end the connection for
good, so the reconnect above runs. No specification sets the number of missed
answers that justifies a close. Record the value that you choose beside the
constant.

### The close code decides the answer

| The close code | What it reports | What the application does |
| --- | --- | --- |
| 1000 | A normal close | Nothing. The connection ended as it was asked to. |
| 1001 | The endpoint goes away, such as a restart or a navigation | Reconnect under the backoff. |
| 1006 | An abnormal close with no close frame | Reconnect under the backoff. NEVER end the session. |
| 1008 | A policy violation | Show the error. Do not reconnect. The next attempt gets the same answer. |
| 1011, 1012, 1013 | A server fault, a restart, or an instruction to try later | Reconnect under the backoff. |
| 4001, in the application range | The identity is not authenticated | Send the user to the answer that `references/route-protection-and-permissions.md` states. |
| 4003, in the application range | The identity may not read this subject | Show the permission state. Do not reconnect. |

RFC 6455 states that no endpoint sends 1006. The browser generates it locally
when the connection ends with no close frame, which is what a lost network
produces. A 1006 is therefore the most common close code of a mobile session,
and it says nothing about the identity of the user.

`references/session-and-token-lifecycle.md` owns the statuses that end a
session, and it states that a fault of the backend never ends one. This file
maps the close codes to the same answers. A 1006 that logs the user out is the
same defect as a 500 that logs the user out.

### The tab that nobody looks at

A hidden tab holds its connection and receives every message. The device pays
for the radio, the battery, and the render.

```ts
// Correct: the effect that owns the connection also owns the pause. This block
// goes inside the provider effect above, beside connect().
const onVisibilityChange = () => {
  if (document.hidden) {
    closedByUs = true; // a deliberate close, so no reconnect runs
    clearTimeout(timer); // a pending reconnect would open a second socket
    socketRef.current?.close(1000, "hidden");
    socketRef.current = null;
    return;
  }
  closedByUs = false;
  if (socketRef.current === null) connect(); // connect() resyncs on open
};

document.addEventListener("visibilitychange", onVisibilityChange);
// The cleanup of the same effect removes this listener.
```

`references/server-state-and-query-cache.md` states that Query stops a poll in
a background tab, because `refetchIntervalInBackground` defaults to `false`. A
connection has no such default, so this rule is the equivalent for a transport
that pushes. Keep the connection open only where the user must see the change
on the return to the tab. State that reason in a comment.

### A dropped connection is visible

```tsx
// Wrong: the view reads the cache and nothing else.
// Failure: the connection dropped ten minutes ago. The rows on the screen are
// the rows of ten minutes ago, and a feed with no events yet renders the empty
// state. The user reads "nothing happened", and the truth is "nothing arrived".
if (events.length === 0) return <FeedEmpty />;
return <FeedList events={events} />;
```

```tsx
// Correct: the connection status is a fifth state beside loading, error,
// empty, and ready.
"use client"; // reason: the status comes from a browser connection
import { useConnectionStatus } from "@/lib/realtime/socket-provider";

export function Feed({ events, isPending, isError }: FeedProps) {
  const status = useConnectionStatus();

  if (isPending) return <FeedSkeleton rows={10} />;
  if (isError) return <FeedError />;

  return (
    <>
      {status === "open" ? null : <FeedDegraded status={status} />}
      {events.length === 0 ? (
        <FeedEmpty live={status === "open"} />
      ) : (
        <ol aria-busy={status === "reconnecting"}>
          {events.map((event) => (
            <li key={event.id}>{event.title}</li>
          ))}
        </ol>
      )}
    </>
  );
}
```

`references/server-state-and-query-cache.md` owns the four states of a data
view. This file adds the fifth, and the fifth changes the empty state: an empty
list under a broken connection is not an empty list. Announce the change of
status without a move of the focus.
`references/keyboard-focus-and-live-regions.md` owns the politeness level and
the screen-reader semantics of that announcement.

### One long response, read in parts

```ts
// Correct: src/lib/realtime/ndjson.ts reads one response in parts. The abort
// signal is the only stop, because a stream has no single deadline.
export async function* readNdjson(response: Response): AsyncGenerator<unknown> {
  if (response.body === null) return;
  const reader = response.body.pipeThrough(new TextDecoderStream()).getReader();
  let rest = "";

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    rest += value;
    const lines = rest.split("\n");
    rest = lines.pop() ?? ""; // the last part can be half a line
    for (const line of lines) {
      if (line.length > 0) yield JSON.parse(line);
    }
  }
}
```

```ts
// The caller owns the stop.
const controller = new AbortController();
const response = await fetch("/api/reports/export/", { signal: controller.signal });
for await (const row of readNdjson(response)) {
  applyRow(row); // the parse over each row is the file below
}
```

Two rules bind this pattern. The last part of a chunk can be half a line, so
the parser holds it and joins it to the next chunk. Each row is a value from
outside the program, so it needs a parse, and
`references/live-events-and-cache-merge.md` owns that parse.

`references/api-client-and-request-safety.md` owns the deadline on an ordinary
request, and it states that every request carries one. A streamed response has
no single deadline, because the response is correct for as long as it produces
rows. `controller.abort()` is the only stop, so the component that starts the
stream must own the controller and abort it on unmount.

### Server-sent events, and the header problem

```ts
// Wrong: the native EventSource where a bearer token must travel.
// Failure: EventSource sends a GET and sets no custom header, so the token has
// nowhere to go but the query string, and the URL reaches every access log on
// the path. A backend that reads the header alone answers 401, and the browser
// reconnects into the same 401 on its own schedule.
const source = new EventSource(`/api/notifications/?token=${accessToken}`);
```

```ts
// Correct: prefer the same-origin topology, where the cookie travels with no
// header at all.
const source = new EventSource("/api/notifications/");
source.addEventListener("message", (event: MessageEvent<string>) => applyEvent(event.data));
source.addEventListener("error", () => setStatus("reconnecting"));
```

```ts
// Correct, where a header must travel: a fetch-based source that stops on a
// 401 rather than reconnecting into it.
import { fetchEventSource } from "@microsoft/fetch-event-source";

const controller = new AbortController();
await fetchEventSource("/api/notifications/", {
  headers: { Authorization: `Bearer ${ticket}` },
  signal: controller.signal,
  onopen: async (response) => {
    if (response.status === 401 || response.status === 403) {
      throw new Error("not authorised"); // terminal, so the library stops
    }
  },
  onmessage: (event) => applyEvent(event.data),
});
```

Take the same-origin topology first. `references/cross-origin-and-bff-proxy.md`
states the rewrite and the Route Handler proxy that produce it, and both remove
the need for a header on the stream. A fetch-based source is the answer where
the topology cannot change.

The frame format is `data:`, `event:`, `id:`, and `retry:`, and a blank line
ends each frame. The WHATWG HTML standard defines it, and Django does not. The
browser sends the last `id:` back in the `Last-Event-ID` header on the
reconnect, so a server that numbers its events can replay the gap.

### What the path in front of Node must carry

The frontend writes none of this configuration. It states what the client
depends on, so that a review can fail on it.

| The layer | What the client depends on | The failure when it is absent |
| --- | --- | --- |
| Nginx, for a WebSocket | `proxy_http_version 1.1`, `proxy_set_header Upgrade $http_upgrade`, and `proxy_set_header Connection "upgrade"` | The handshake never reaches 101, and every connection fails at once |
| Nginx, for an idle connection | `proxy_read_timeout` above the heartbeat period. The documented default is 60 seconds. | The connection closes with 1006 after exactly 60 seconds of quiet |
| Nginx, for server-sent events | `proxy_buffering off`, or the response header `X-Accel-Buffering: no` | The events arrive together at the end, so the feed works in development and batches in production |
| Cloudflare | A client heartbeat under the idle close. Only the Enterprise plan configures the period. | The connection closes with 1006 about every 100 seconds |
| The ASGI server | Daphne or Uvicorn behind the proxy, and a channel layer that is up | A `group_send` raises on the server, no event arrives, and the view must show degraded rather than empty |

`references/runtime-process-and-reverse-proxy.md` owns the Nginx file itself.
The backend owns the ASGI process and the health check. This file owns the list
above, and the `curl -N` command in the verification block that proves the third
row.

### The Django seam

The client connects to `wss://<host>/ws/<path>/`. Channels routes the
connection through `ProtocolTypeRouter` and then `URLRouter`, and the consumer
is usually `AsyncJsonWebsocketConsumer`. A group over the channel layer fans
one event out to many connections through `group_send`.

Django 4.2 added async iterator support to `StreamingHttpResponse` under ASGI,
and the release notes name server-sent events as a use case. An SSE view
therefore takes an `async def` generator. A synchronous generator under ASGI is
consumed in full before the response starts, which defeats the stream.

The client depends on three response headers on an SSE endpoint:
`Content-Type: text/event-stream`, `Cache-Control: no-cache`, and
`X-Accel-Buffering: no`. The Django documentation prescribes none of them, and
the third is an Nginx directive that travels as a header. Confirm all three
against the real deployment with `curl -N` before you depend on the stream.

The backend owns the consumer, the queue, the worker, and the event as a
published surface. This file owns the transport, the handshake, and what the
browser does when the connection ends.

### The libraries

The table gives each library its latest version, its maintenance status, and
its open advisories. The package registry and the advisory database supplied
those facts on 16 August 2026. Read the advisory database again before you
install any row of it.

| Tier | Library | The rule | Latest version | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- |
| Recommend | The web platform — `WebSocket`, `EventSource`, `ReadableStream`, `TextDecoderStream` | Take these first. Reach for a package only for the reconnect and the buffer. | Baseline | The web platform | None |
| Recommend | `partysocket` | The wrapper for a WebSocket that must reconnect and buffer. It is the maintained fork of `reconnecting-websocket`. | 1.3.0, released 23 June 2026 | Active, at Cloudflare | None known |
| Recommend | Zod | The parse over each frame. `references/live-events-and-cache-merge.md` owns the schema. | 4.x, which is the pin of this stack | Active | None known |
| Recommend | TanStack Query | The cache that a pushed event writes into. | 5.101 or later, which is the pin of this stack | Active | None known |
| Conditional | `@microsoft/fetch-event-source` | Server-sent events where a header must travel, or where the request must be a POST. It also pauses on `document.hidden`. Current but in decline. | 2.x | Low release activity, at Azure. Community forks exist. | None known |
| Conditional | `experimental_streamedQuery` from TanStack Query | An async iterable that must arrive in the query cache as one entry, such as tokens from a model. The import name carries the `experimental_` prefix, and the API can change with no major release. | In version 5 | Active, and experimental | None known |
| Conditional | WebTransport | Unreliable datagrams over HTTP/3, for the latest browsers only. Do not mandate it. | Baseline newly available since March 2026 | The specification is a W3C Working Draft | None known |
| Audit-only | `reconnecting-websocket` | An existing install only. Plan the move to `partysocket`. In decline. | 4.4.0, about six years old | Dormant | None known |
| Reject | `socket.io` against Django Channels | It runs its own Engine.IO handshake and framing over a `/socket.io/` path, and it does not speak RFC 6455 to `AsyncJsonWebsocketConsumer`. Take the native `WebSocket` or `partysocket`. | — | Active, and wrong for this backend | — |

`references/dependencies-and-git-workflow.md` owns the rule that a new
dependency states its replacement, its size, and its maintenance status.

### Version discipline

Read the installed versions before you write code.

Django Channels 4.3 is the pin for the WebSocket seam. Django 4.2 is the floor
for an SSE view, because the async iterator support arrived there.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states. Next.js 16.3 is the framework pin.

WebTransport reached Baseline newly available in March 2026, so the latest
Chrome, Edge, Firefox, and Safari carry it. An older browser does not, and the
specification is still a W3C Working Draft. Audit an existing use of it, and
mandate it nowhere.

`partysocket` 1.3.0 of 23 June 2026 replaces `reconnecting-websocket` 4.4.0,
which has taken no release for about six years.

## Verification

```bash
# 1. Read the installed versions before you write code.
node -p "require('next/package.json').version"
node -p "require('react/package.json').version"

# 2. Find a connection opened inside a component. This must print nothing.
rg -n 'new WebSocket\(|new EventSource\(' src/app src/components src/features

# 3. Find every connection in the application. Read every hit, and confirm
#    that one provider holds it.
rg -n 'new WebSocket\(|new EventSource\(|fetchEventSource\(' src/

# 4. Find a credential in a socket URL. Read every hit.
rg -n -e 'wss?://[^"'"'"'`]*token=' -e 'ws/.*\?.*token' src/

# 5. Find a reconnect at a fixed period. This must print nothing.
rg -n -B3 -A3 'setInterval' src/ | rg -i 'websocket|eventsource|reconnect'

# 6. Find an effect that opens a connection and returns no cleanup.
rg -l 'new WebSocket\(' src/ | xargs rg --files-without-match '\.close\('

# 7. Confirm that server-sent events are not buffered. The lines arrive one at
#    a time, and not together at the end.
curl -N -H 'Accept: text/event-stream' "$NEXT_PUBLIC_API_BASE_URL/api/notifications/"

# 8. Confirm the handshake. The answer is 101, and it echoes the upgrade.
npx wscat -c "wss://staging.example.com/ws/feed/"

# 9. Confirm one connection after a navigation. Open the network panel, filter
#    on WS, navigate away, and return. There is one open connection.

# 10. Confirm the reconnect. Stop the backend, and read the delay between the
#     attempts in the network panel. It doubles, and it stops at the cap.

# 11. Confirm the heartbeat period. Leave the tab open and quiet for three
#     minutes. The connection stays open.

# 12. Confirm the degraded state. Stop the backend with rows on the screen.
#     The view marks the data stale, and it does not render the empty state.
```

## Review checklist

- [ ] Does every `new WebSocket` and every `new EventSource` carry a written
      reason to prefer it over a poll?
- [ ] Is the transport chosen from the decision table, and is the choice
      recorded?
- [ ] Is there one connection for the application, in a provider, held in a
      ref?
- [ ] Is the connection absent from every component body and from every list
      row?
- [ ] Does the effect return a cleanup that closes the connection?
- [ ] Does the credential travel as a cookie, a first message, or a
      subprotocol, rather than in the query string?
- [ ] Does a query-string ticket expire in seconds, and does it carry a comment
      that names the risk?
- [ ] Does the review record that the backend checks the `Origin` header on the
      handshake?
- [ ] Does the reconnect double the delay, cap it, and spread it with a random
      part?
- [ ] Does the reconnect resync the data of the gap on `open`?
- [ ] Is there a heartbeat under the shortest idle timeout on the path?
- [ ] Does a watchdog close the connection when no answer arrives?
- [ ] Do 1006, 1011, 1012, and 1013 reconnect, and do only 1008, 4001, and 4003
      reach the auth or permission state?
- [ ] Does the connection stop while the tab is hidden, or does a comment state
      why it must not?
- [ ] Does a dropped connection render a degraded state rather than an empty
      state?
- [ ] Does the streamed response hold a partial line, and does one
      `AbortController` stop it?
- [ ] Are the proxy directives for `Upgrade`, `proxy_read_timeout`, and
      `proxy_buffering` confirmed against the real deployment?
- [ ] Do the three SSE response headers arrive through the real proxy?

## Handoffs

- The frame that arrives, its parse, the unknown type, and the write into the
  query cache → `references/live-events-and-cache-merge.md`.
- The poll, `refetchInterval`, the four states of a data view, and the key
  factory → `references/server-state-and-query-cache.md`.
- The credential itself, where it lives, the single-flight refresh, and the
  status that ends a session → `references/session-and-token-lifecycle.md`.
  This file owns only the carrier on the handshake and the close code.
- The redirect after an unauthenticated close, and the permission state →
  `references/route-protection-and-permissions.md`. That file states that the
  connection never trusts an identity that the client sends.
- The same-origin topology, the rewrite, and the Route Handler proxy that
  removes the header problem → `references/cross-origin-and-bff-proxy.md`.
- The deadline, the retry rule, and `normalizeApiError` on an ordinary request
  → `references/api-client-and-request-safety.md`.
- The rules for an effect, its cleanup, `useEffectEvent`, and
  `useSyncExternalStore` → `references/state-and-effects.md`.
- The `"use client"` directive that this provider carries, and the provider in
  its own wrapper → `references/server-and-client-components.md`.
- The `<Suspense>` boundary and the error boundary above a live view →
  `references/suspense-and-actions.md`.
- The store that code outside React reads through `getState()` →
  `references/client-and-url-state.md`.
- The politeness level of a live announcement, and the focus that must not move
  → `references/keyboard-focus-and-live-regions.md`. That domain holds a veto.
- The progress feed of an upload →
  `references/file-upload-and-transport.md`. This file owns the transport under
  it.
- The download that a stream produces →
  `references/served-content-and-downloads.md`.
- The row that a live table adds while the user reads it →
  `references/data-table-and-server-driven-state.md`.
- The words of a degraded message and of a reconnect message →
  `references/error-and-empty-state-copy.md`.
- The connection uptime, the reconnect count, and the alert over them →
  `references/error-capture-and-reporting.md`. This file emits the status, and
  that file owns the rule that gives an alert an owner and an action. The
  transport of a field report is `references/correlation-and-telemetry.md`.
- The Nginx file, the TLS termination, and the reverse proxy in front of Node →
  `references/runtime-process-and-reverse-proxy.md`.
- The MSW `ws` namespace and the fixture for a dropped connection →
  `references/network-mocks-and-contract-tests.md`. The Playwright `routeWebSocket`
  mock → `references/end-to-end-journeys-and-flake-control.md`. Three
  assertions prove this domain. They are one connection after a navigation, a
  reconnect delay that grows, and a degraded state on a close.
- The consumer, the channel layer, the queue, and the worker behind the
  connection → the backend. This file owns the transport for the state of a job
  and the interface for its progress.
- The `Origin` check on the handshake, the permission class on the consumer, and
  the rate limit → the backend's security review.
- The ASGI server, the health check, and the go-live gate → the backend.
