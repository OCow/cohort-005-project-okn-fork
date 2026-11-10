# Live push transport for the bell

Research for issue #10 (map: issue #1). Question: how can the server live-push a new **Notification** to an open browser tab in this stack (React Router 8.3 framework mode, `react-router-serve`, one Node process, SQLite)? Compare SSE, WebSockets and long polling, and recommend one.

## Recommendation

**Use Server-Sent Events (SSE) from a React Router resource route.** Fan out through an in-process `EventEmitter` that lives on `globalThis`.

- A resource route `GET /api/notifications/stream` returns `new Response(readableStream, { headers: { "Content-Type": "text/event-stream", "Cache-Control": "no-cache, no-transform" } })`.
- Auth is the existing cookie session. The loader calls `getCurrentUserId(request)` and returns 401 when there is no user. That 401 also stops `EventSource` from retrying.
- An action writes the Notification row with better-sqlite3, which is synchronous. After the write commits, it calls `notificationBus.emit("user:<id>", payload)`. Each open stream for that user is subscribed to that key.
- On every connect and reconnect, the stream first sends a **snapshot** (the unread count and the latest Notification id). Missed events therefore never need replaying.
- The stream sends a `: keep-alive` comment every ~15–25 s. Subscriptions are cleaned up in the stream's `cancel()` and on `request.signal` abort.

The SSE pipeline works unchanged under both `react-router-serve` and `react-router dev`. I checked the source code and ran a harness built from the installed packages (see Verification). No new dependency and no custom server are needed.

## Why not the alternatives

| | SSE (resource route) | WebSockets | Long polling |
|---|---|---|---|
| Works with `react-router-serve` as is | **Yes.** The Express adapter has explicit `text/event-stream` handling. | **No.** `react-router-serve` keeps its `http.Server` in a local variable and gives no hook for `upgrade` (`@react-router/serve/dist/cli.js` L111–125). You would have to replace it with a custom Express server and a `ws` dependency. | Yes |
| Works in Vite dev as is | **Yes** (verified) | Needs a Vite plugin `configureServer` hook to attach to Vite's HTTP server. That server already carries Vite's HMR WebSocket. | Yes |
| Auth | Same-origin cookie is sent automatically, so `getCurrentUserId` works unchanged | Cookie arrives on the upgrade request, but you parse the session outside React Router | Same as normal loaders |
| Reconnection | Built into the browser (`retry:`, `Last-Event-ID`) | Hand-rolled | Inherent, but each event costs a new request |
| Direction needed | Server to client only, which is all the bell needs | Bidirectional (not needed) | Server to client |
| Cost | One open HTTP/1.1 connection per tab | One socket per tab | One held request per tab, plus re-request churn and the gap between polls |

WebSockets would only be worth it for bidirectional traffic, and the bell has none. They would also make us give up `react-router-serve`, which the map treats as the production runtime. Long polling has the same per-tab held connection as SSE but needs more code (cursor handling, re-arming, a race between response and next request). It adds nothing for this stack.

## How it fits this codebase

### Resource route

React Router: "A route becomes a resource route by convention when its module exports a loader or action but does not export a default component", and resource routes "can return `Response` instances" ([React Router docs, Resource Routes](https://reactrouter.com/how-to/resource-routes)). Register it outside `layout.app.tsx`, next to the other `api/*` routes in `app/routes.ts`, so no layout loader runs per connection. For example: `route("api/notifications/stream", "routes/api.notifications-stream.ts")`.

### Auth (cookie session)

- Sessions are `createCookieSessionStorage` with `cadence_session`, `httpOnly`, `sameSite: "lax"` (`app/lib/session.ts` L3–11). `EventSource` to a same-origin URL sends cookies by default; `withCredentials` only matters for cross-origin (MDN `EventSource.withCredentials`). The loader can therefore use `getCurrentUserId(request)` exactly like `app/routes/api.video-tracking.ts` L15–19.
- The spec says that if "res's status is not 200, or if res's Content-Type is not text/event-stream, then fail the connection", and a failed connection "does not attempt to reconnect" ([WHATWG HTML, Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html)). So returning `new Response("Unauthorized", { status: 401 })` cleanly stops a signed-out tab.
- **Caveat: the session is checked only at connect time.** Cookie sessions cannot be revoked server-side. After `api/logout` or the DevUI `api/switch-user` (`app/routes/api.switch-user.ts`), an already-open stream keeps pushing to the old user until the tab reconnects. Mitigation: open the `EventSource` in a `useEffect` keyed on `currentUser.id` from the `layout.app.tsx` loader (L174–205). It closes and reopens when the user changes. Both logout and switch-user redirect, which revalidates loaders.

### In-process emitter (action to open stream)

- One Node process (map #1 / ADR 0006), so a module-level `EventEmitter` is enough. Keep it in something like `app/lib/notification-bus.server.ts`, keyed per user (`"user:42"`). Call `setMaxListeners(0)`, because one listener per tab per user would otherwise trigger Node's 10-listener warning.
- **Store it on `globalThis`**: `globalThis.__notificationBus ??= new EventEmitter()`. In dev, `@react-router/dev` re-imports the server build through Vite's SSR module runner on every request (`@react-router/dev/dist/vite.js` L1655–1660). An HMR edit to the bus module or to anything that imports it would otherwise create a second emitter. Open streams would stay subscribed to the old one and miss events.
- Emit **after** the DB write. better-sqlite3 is synchronous, so the write has committed once `.run()`/`.get()` returns, unless it is inside a `db.transaction`. In that case emit after the transaction callback returns. The payload should be small, e.g. `{ unreadCount, notification: { id, ... } }`, or just a "changed" ping that tells the client to revalidate.
- This does **not** scale past one process. If the app ever runs more than one instance, replace the emitter with a cross-process channel (e.g. polling SQLite for `id > lastSeen`). The route and client code stay the same.

### Reconnection

- `EventSource` reconnects automatically after network errors. The server can set the delay with `retry: <ms>`. On reconnect the browser sends `Last-Event-ID` if an `id:` was received (WHATWG spec, as above).
- Recommended: **send a snapshot on every connect** (the current unread count, read from SQLite). That makes reconnection lossless with no replay logic. Optionally set `id: <notificationId>` so a later version could replay `WHERE id > Last-Event-ID`.
- Keep-alive: the spec suggests "a comment line (one starting with a ':' character) every 15 seconds or so" for proxies that drop idle connections.

### Cleanup on disconnect

Both pipelines cancel the response body when the socket closes, so the `ReadableStream`'s `cancel()` is the single place to unsubscribe and `clearInterval`. The harness logged `stream cancelled` for every client disconnect under both pipelines.
- `react-router-serve`: `@react-router/express` aborts `request.signal` on `res` `close` (`@react-router/express/dist/index.js` L47–48). `writeReadableStreamToWritable` cancels the reader when the writable closes (`@react-router/node/dist/index.js` L107–129).
- Vite dev: `@remix-run/node-fetch-server` `sendResponse` calls `reader.cancel()` on `res` `close` (`dist/lib/request-listener.js` L254–261).

## Buffering and compression

### `react-router-serve`

`react-router-serve` mounts `compression()` globally (`@react-router/serve/dist/cli.js` L113, `compression@1.8.2`).

- `compression`'s default filter compresses any type `compressible()` accepts. `mime-db@1.54.0` has no `text/event-stream` entry, so it falls through to the `^text\/` regex (`compressible/index.js` L23). **SSE responses therefore get compressed** (the harness showed `Content-Encoding: br`).
- This does not buffer events in practice. `@react-router/express` calls `res.flushHeaders()` for `text/event-stream` (`dist/index.js` L63). `writeReadableStreamToWritable` calls `writable.flush()` after every chunk (`@react-router/node/dist/index.js` L120–121), which is `compression`'s `res.flush()` → zlib flush. Measured: each event arrived about 500 ms apart, in real time.
- Still, **set `Cache-Control: no-cache, no-transform`**. `compression` skips any response whose `Cache-Control` contains `no-transform` (`compression/index.js` `shouldTransform`, L315–321). That avoids per-event brotli flush overhead and stops intermediaries from transforming or buffering the stream. Verified: with `no-transform`, `Content-Encoding` was absent and events still streamed.

### Vite dev

The dev server has no compression middleware. Vite's `@polka/compression` is only mounted in the **preview** server (`vite/dist/node/chunks/config.js` L35377). `sendResponse` writes each chunk directly. Verified streaming.

### Reverse proxy (if one is added in production)

Nginx buffers proxied responses by default. Send `X-Accel-Buffering: no` or set `proxy_buffering off`, and keep the keep-alive interval below `proxy_read_timeout` (default 60 s). The spec also notes that "HTTP chunking can have unexpected negative effects on the reliability of this protocol" with some legacy proxies.

### Node timeouts

Node's `server.requestTimeout` (300 s default) limits receiving the request, not a long response, and `server.timeout` defaults to 0. Neither ends an active SSE response. `react-router-serve` sets none of them.

## Risks and open points

1. **HTTP/1.1 six-connection limit.** MDN: "When not used over HTTP/2, SSE suffers from a limitation to the maximum number of open connections … per browser + domain … (6)." `react-router-serve` only speaks HTTP/1.1, so a user with 6+ Cadence tabs would starve their own page loads. Options:
   - put HTTP/2 TLS termination in front (a reverse proxy) in production;
   - have only one tab hold the stream (a `BroadcastChannel` leader, or a `SharedWorker`);
   - close the stream while `document.hidden` and reopen it on `visibilitychange`, with the snapshot on reconnect covering the gap.

   Pick one when splitting implementation tickets. The visibility approach is cheapest.
2. **Stale identity on open streams** after logout or user switch (see Auth). Handled client-side by keying on the user id.
3. **Dev HMR** can create duplicate emitters unless the bus lives on `globalThis`.
4. **Single-process assumption**, the same as ADR 0006. Multi-instance hosting needs a different fan-out.
5. **Graceful shutdown**: on SIGTERM, `react-router-serve` calls `server.close()` (`cli.js` L126–128), which waits for open connections. Long-lived SSE streams can hold shutdown open until the process is killed. This is acceptable; clients reconnect and get a snapshot.

## Verification

I wrote a harness (not committed) that loads the **installed** packages from the main checkout's `node_modules`:
- `express` + `compression()` + a replica of `@react-router/express`'s `sendRemixResponse`, calling the real `writeReadableStreamToWritable` from `@react-router/node`;
- a plain `http` server using the real `sendResponse` from `@remix-run/node-fetch-server` (the function `@react-router/dev` uses in Vite dev).

The handler returned a `ReadableStream` that sent an event every 500 ms. I measured arrival times from a Node `fetch` client:

| Pipeline | `Cache-Control` | `Content-Encoding` | Event arrivals (ms) | `cancel()` on disconnect |
|---|---|---|---|---|
| react-router-serve | `no-cache` | `br` | 156, 643, 1148, 1654, 2156 | yes |
| react-router-serve | `no-cache, no-transform` | none | 136, 627, 1139 | yes |
| Vite dev (`node-fetch-server`) | `no-cache` | none | 135, 632, 1140 | yes |

Not verified: a full `react-router build` plus `react-router-serve` run with a real route in this app. That would have meant changing app code, which was out of scope. The harness covers the exact middleware and write functions involved.

## Sources

- React Router, Resource Routes: https://reactrouter.com/how-to/resource-routes
- WHATWG HTML Living Standard, Server-sent events: https://html.spec.whatwg.org/multipage/server-sent-events.html
- MDN, EventSource: https://developer.mozilla.org/en-US/docs/Web/API/EventSource
- Source, as installed (versions pinned in `package.json` / lockfile):
  - `@react-router/serve@8.3.0` `dist/cli.js`
  - `@react-router/express@8.3.0` `dist/index.js`
  - `@react-router/node@8.3.0` `dist/index.js`
  - `@react-router/dev@8.3.0` `dist/vite.js`
  - `@remix-run/node-fetch-server@0.13.3` `dist/lib/request-listener.js`
  - `compression@1.8.2` `index.js`, `compressible@2.0.18`, `mime-db@1.54.0`
  - `vite@7.3.7` `dist/node/chunks/config.js`
- App code: `app/lib/session.ts`, `app/routes/api.video-tracking.ts`, `app/routes/api.switch-user.ts`, `app/routes/layout.app.tsx`, `app/db/index.ts`, `app/routes.ts`
