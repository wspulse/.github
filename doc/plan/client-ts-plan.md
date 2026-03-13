# wspulse TypeScript Client — Development Plan (`client-ts`)

> Status: P1–P4 complete · Last updated: 2026-03-14
> Repo: `wspulse/client-ts` · Package: `@wspulse/client`

**Read before starting:**

- [Interface contract](../contracts/client-interface.md)
- [Behaviour contract](../contracts/client-behaviour.md)
- [Wire protocol](../../server/doc/protocol.md)
- [client-go] reference implementation: `client-go/client.go`, `client-go/options.go`

---

## Environments

| Target      | WebSocket source                  |
| ----------- | --------------------------------- |
| Browser     | `globalThis.WebSocket` (built-in) |
| Node.js 20+ | `ws` package (peer dependency)    |

Module system: **ESM-first**, CJS compat shim via dual `exports` in `package.json`.

---

## Project Structure

```
client-ts/
  src/
    index.ts            # public re-exports
    client.ts           # WspulseClient implementation
    frame.ts            # Frame interface
    errors.ts           # ConnectionClosedError, RetriesExhaustedError, ConnectionLostError
    backoff.ts          # backoff(attempt, baseDelay, maxDelay): number
    options.ts          # ClientOptions interface + defaults
  test/
    backoff.test.ts     # unit tests — backoff formula
    client.test.ts      # integration tests against wspulse/server
  package.json
  tsconfig.json         # strict + NodeNext for src
  tsconfig.build.json   # emit to dist/
  vitest.config.ts
  .npmrc
  .github/
    workflows/
      ci.yml            # lint + type-check + test on Node 20 + 22
```

---

## Phase Breakdown

### P1 — Project Scaffold ✅

- [x] Create repo `wspulse/client-ts`
- [x] `package.json`:  name `@wspulse/client` v0.1.0, `type: "module"`, dual exports, peer dep `ws >=8`, dev deps typescript/vitest/eslint/prettier
- [x] `tsconfig.json`: strict, ES2022, NodeNext
- [x] `tsconfig.build.json`: declarationMap, outDir dist/
- [x] `vitest.config.ts` with environment node and coverage v8
- [x] CI workflow: lint → type-check → test on Node 20 and 22 (3-job matrix)
- [x] `.npmrc`: `provenance=true`

### P2 — Connect / Send / Close / Done ✅

- [x] **`frame.ts`** — Frame interface:

  ```ts
  export interface Frame {
    id?: string;
    event?: string;
    payload?: unknown;
  }
  ```

- [x] **`errors.ts`** — three error classes:

  ```ts
  export class ConnectionClosedError extends Error { ... }
  export class RetriesExhaustedError extends Error { ... }
  export class ConnectionLostError extends Error { ... }
  ```

- [x] **`options.ts`** — `ClientOptions` interface + `defaultOptions()`:
      | Option | Type | Default |
      | ---------------- | --------------------------------------- | ----------------- |
      | `onMessage` | `(frame: Frame) => void` | no-op |
      | `onDisconnect` | `(err: Error \| null) => void` | no-op |
      | `onReconnect` | `(attempt: number) => void` | no-op |
      | `onTransportDrop`| `(err: Error) => void` | no-op |
      | `autoReconnect` | `{ maxRetries, baseDelay, maxDelay }` | disabled |
      | `heartbeat` | `{ pingPeriod, pongWait }` (ms) | 20 000 / 60 000 |
      | `writeWait` | `number` (ms) | 10 000 |
      | `maxMessageSize` | `number` (bytes) | 1 MiB |
      | `dialHeaders` | `Record<string, string>` | `{}` |

- [x] **`client.ts`** — `WspulseClient` class (not exported directly — exposed via `connect()`):
  - Internal buffer: `Uint8Array[]` with `maxSize = 256`; head-drop on overflow
  - `done`: `Promise<void>` exposed via a stored resolver (`new Promise(res => this._resolve = res)`)
  - Read loop: `websocket.onmessage` → decode JSON → call `onMessage`
  - Write loop: dequeue from buffer → `websocket.send()` with `writeWait` timeout
  - `send(frame)`: JSON-encode then push to buffer; throw `ConnectionClosedError` if closed
  - `close()`: set `_closed = true`, close WS with code 1000, resolve `done`

- [x] **`index.ts`** — export `connect`, `Frame`, `Client` interface, all error classes

- [x] Entry point `connect(url, opts?)`:
  - Instantiate `WspulseClient`, call initial `dial()`
  - Throw on fail when `autoReconnect` disabled
  - Return client

### P3 — Auto-Reconnect + Backoff ✅

- [x] **`backoff.ts`** — pure function matching the contract formula exactly (equal jitter `[0.5, 1.0]`):

  ```ts
  export function backoff(
    attempt: number,
    baseDelay: number,
    maxDelay: number,
  ): number {
    const exp = Math.min(baseDelay * 2 ** attempt, maxDelay);
    const half = exp / 2;
    return half + Math.random() * half;  // equal jitter [0.5, 1.0]
  }
  ```

- [x] Reconnect loop (in `client.ts`):
  1. `ws.onclose` / `ws.onerror` → fire `onTransportDrop(err)` (only if not user-initiated)
  2. `await sleep(backoff(attempt, base, max))`
  3. Fire `onReconnect(attempt)`
  4. Attempt `dial()`
  5. If success → reset `attempt`, set up new event handlers
  6. If fail → `attempt++`; if `attempt >= maxRetries > 0` → fire `onDisconnect(new RetriesExhaustedError())` → resolve `done`

- [x] `close()` during reconnect: cancel the pending `sleep` by rejecting the sleep-promise via an `AbortController`

- [x] `_closed` flag checked at every reconnect iteration

### P4 — Heartbeat + Write Deadline ✅

- [x] **Pong**: standard WebSocket library replies automatically — no manual pong needed.
- [x] **Heartbeat (Node.js `ws` only)**: `startHeartbeat(ws)` sends Ping every `pingPeriod` via `setInterval`; starts `pongWait` deadline timer after each Ping; resets deadline on Pong via `ws.on("pong", ...)`. Browser: no-op (browser handles ping/pong automatically).
- [x] **`writeWait`**: `sendWithTimeout(data, timeoutMs)` helper wraps `ws.send()` with `setTimeout` that closes WS on timeout. Used for control frames.
- [x] **`maxMessageSize`**: enforced in `onmessage` via `String(ev.data).length` check; closes with code 1009 and triggers `handleTransportDrop()` directly (detaches `onclose` first to avoid double-fire).
- [ ] Document browser limitation: `dialHeaders` is **not supported** in the browser WebSocket API.

### P5 — Test Suite (partially complete)

34 tests passing across 5 test files. Scenarios 1–7 and 9 covered using lightweight `ws.WebSocketServer` echo servers (no live `wspulse/server` required). Scenario 8 is N/A for single-threaded JS.

| #   | Scenario                                                      | Status |
| --- | ------------------------------------------------------------- | ------ |
| 1   | Connect → send → echo → close clean                           | ✅ Done |
| 2   | Server drops → onTransportDrop + onDisconnect (no reconnect)  | ✅ Done |
| 3   | Auto-reconnect: server drops → reconnects within maxRetries   | ✅ Done |
| 4   | Max retries exhausted → `onDisconnect(RetriesExhaustedError)` | ✅ Done |
| 5   | `close()` during reconnect → loop stops, `onDisconnect(null)` | ✅ Done |
| 6   | `send()` on closed client → `ConnectionClosedError`           | ✅ Done |
| 7   | Heartbeat pong timeout → transport drop                       | ✅ Done (short pongWait, server ignores pings) |
| 8   | Concurrent sends: no race / interleaving                      | N/A (single-threaded JS) |
| 9   | Concurrent close + transport drop → onDisconnect exactly once | ✅ Done |

Additional tests: done resolution, close idempotency, head-drop on buffer overflow, connect failure to unreachable host, message ordering, maxMessageSize enforcement.

- [ ] P5 stretch: integration tests against a live `wspulse/server` instance (vitest globalSetup with `go run .`)

---

## Key Implementation Notes

1. **Single-threaded safety**: JavaScript's event loop means no mutex is needed for the send buffer. However, async re-entrancy in the reconnect loop must be guarded with an `_isReconnecting` flag.
2. **`done` Promise**: backed by a `{ resolve, reject }` deferred captured at construction. Resolves (not rejects) in all cases — callers check `onDisconnect`'s argument for the error.
3. **Browser `dialHeaders` limitation**: the `WebSocket` constructor accepts only `url` and `protocols`. Raise an `Error` with a clear message if `dialHeaders` is set in a browser context.
4. **ESM/CJS dual build**: use `tsup` or `rollup` to produce both `dist/index.js` (ESM) and `dist/index.cjs`; both must include `.d.ts` declarations.
5. **`ws` package as peer dep**: do not import `ws` at the top level; dynamically detect environment: `typeof WebSocket !== "undefined" ? WebSocket : (await import("ws")).WebSocket`.

---

## Publish Checklist (P6)

- [ ] README: quick-start (browser + Node.js examples)
- [ ] CHANGELOG.md with version 0.1.0 entry
- [ ] `npm publish --access public --provenance` from CI on git tag
- [ ] GitHub release with NPM badge
- [ ] Register package at [npmjs.com](https://npmjs.com)
