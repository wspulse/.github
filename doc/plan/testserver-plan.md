# wspulse Shared Test Server — Development Plan (`testserver`)

> Status: planned · Last updated: 2026-03-17
> Location: `wspulse/testserver/` (workspace root)

---

## Motivation

client-go tests scenarios 3–5 (auto-reconnect, retries exhausted, close-during-reconnect) by embedding `wspulse/server` in-process, calling `srv.Kick()` and `srv.Close()` directly. Non-Go clients (client-kt, client-ts, client-swift) run their testserver as an **external process** and can only interact via the WebSocket wire.

The current per-client testservers (`client-kt/testserver/`, `client-ts/testserver/`) are near-identical echo+reject servers with no ability to simulate:

- Server-initiated connection drops (kick)
- Full server shutdown / restart (for retries exhaustion)
- Suppressed pong replies (for heartbeat timeout)

Rather than duplicating control logic across three (and growing) testserver copies, this plan extracts a **single shared testserver** with an HTTP control API.

---

## Architecture

```
testserver/
  main.go       # dual-port server: WebSocket + HTTP control
  go.mod        # module github.com/wspulse/testserver
  go.sum
```

### Dual-Port Design

The testserver listens on **two ports** bound to `127.0.0.1`:

| Port         | Protocol                         | Purpose                                     |
| ------------ | -------------------------------- | ------------------------------------------- |
| WS port      | WebSocket (via `wspulse/server`) | Client connections, echo, reject            |
| Control port | Plain HTTP                       | Test orchestration: kick, shutdown, restart |

**Why two ports?**

- `srv.Close()` tears down the WS listener — the control port must survive to accept `/restart`.
- Clean separation: WebSocket clients never see control endpoints.

### Startup

```
$ go run ./testserver
READY:<ws_port>:<control_port>
```

Printed to **stderr** (same convention as existing testservers). Each client's test harness parses this line to discover both ports.

---

## Control API

All endpoints are plain HTTP on the control port. No authentication (localhost-only, test-use only).

| Method | Path        | Query Params        | Effect                                                          | Use Case                                        |
| ------ | ----------- | ------------------- | --------------------------------------------------------------- | ----------------------------------------------- |
| `GET`  | `/health`   | —                   | `200 OK`                                                        | Verify control server is alive                  |
| `POST` | `/kick`     | `id=<connectionID>` | Calls `srv.Kick(id)`                                            | Scenario 3: trigger reconnect                   |
| `POST` | `/shutdown` | —                   | Calls `srv.Close()` + closes WS listener                        | Scenario 4: make all reconnect dials fail       |
| `POST` | `/restart`  | —                   | Creates new `wspulse.Server` + new WS listener on **same port** | Scenario 3/4: allow reconnects to succeed again |

### Responses

All control endpoints return JSON:

```json
{ "ok": true }
```

On error (e.g., kick unknown ID):

```json
{ "ok": false, "error": "connection not found" }
```

HTTP status: `200` on success, `400` or `500` on error.

---

## WebSocket Behaviour (unchanged)

Query params on the WS endpoint (same as current testservers):

| Param       | Effect                                         |
| ----------- | ---------------------------------------------- |
| `reject=1`  | `ConnectFunc` returns error → HTTP 401         |
| `room=<id>` | Assigns connection to room (default: `"test"`) |
| `id=<id>`   | Sets `connectionID` (default: auto-generated)  |

All inbound frames are echoed back to sender.

---

## Scenario Coverage

| #   | Scenario                   | Control API calls                             | Notes                                                             |
| --- | -------------------------- | --------------------------------------------- | ----------------------------------------------------------------- |
| 3   | Auto-reconnect success     | `POST /kick?id=X`                             | Server stays up; client reconnects to same server                 |
| 4   | Max retries exhausted      | `POST /shutdown`                              | All dial attempts fail; `onDisconnect(RetriesExhausted)`          |
| 5   | `close()` during reconnect | `POST /kick?id=X` then client calls `close()` | Same as #3 setup; test calls `close()` before reconnect completes |
| 7   | Pong timeout               | Future: `?ignore_pings=1` query param         | Requires per-conn `SetPingHandler` override; deferred             |

### Scenario 7 — Deferred

Suppressing pong requires overriding gorilla/websocket's `SetPingHandler` at the per-connection level. This can be done inside `wspulse/server`'s `WithOnConnect` by accessing the underlying `*websocket.Conn`, but it depends on internal server API that may not be exposed. Defer until the server exposes a hook or the testserver embeds gorilla directly.

---

## Migration Steps

### Step 1 — Build shared testserver

- [ ] Create `testserver/` at workspace root
- [ ] `go.mod`: `module github.com/wspulse/testserver`, depend on `github.com/wspulse/server`
- [ ] Implement dual-port main.go with control API
- [ ] Verify: `go build ./testserver && ./testserver` prints `READY:<ws>:<ctl>`
- [ ] Verify: existing echo + reject behaviour works

### Step 2 — Migrate client-kt

- [ ] Update `ProcessBuilder` in `ClientIntegrationTest.kt` to point to `../testserver`
- [ ] Parse `READY:<ws>:<ctl>` to get both ports
- [ ] Add helper function `controlPost(path, params)` for HTTP calls to control port
- [ ] Add integration tests for scenarios 3, 4, 5
- [ ] Delete `client-kt/testserver/`
- [ ] Update `Makefile` to build shared testserver
- [ ] Update CI workflow

### Step 3 — Migrate client-ts

- [ ] Update `test/global-setup.ts` to point to `../testserver`
- [ ] Parse `READY:<ws>:<ctl>` to get both ports; write both to `.server-url` / `.control-url`
- [ ] Add helper function for HTTP calls to control port
- [ ] Add integration tests for scenarios 3, 4, 5
- [ ] Delete `client-ts/testserver/`
- [ ] Update `Makefile` and CI workflow

### Step 4 — Prepare for client-swift

- [ ] Document testserver setup in client-swift-plan.md P5
- [ ] `XCTestCase.setUp()` launches `../testserver` via `Process`
- [ ] Same `READY:<ws>:<ctl>` parsing pattern

---

## Key Design Decisions

1. **Same WS port on restart**: `/restart` rebinds to the same port so clients' configured URL remains valid. Implemented via `SO_REUSEADDR` + retry loop.
2. **No auth on control API**: localhost-only, ephemeral port, test-use only. No need for tokens.
3. **Stateless restart**: `/restart` creates a fresh `wspulse.Server` — no session state carried over. This matches the "server crashed and restarted" scenario.
4. **Graceful shutdown sequencing**: `/shutdown` blocks until `srv.Close()` returns and the WS listener is closed, ensuring subsequent dials fail deterministically.
