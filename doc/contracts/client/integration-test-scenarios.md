# wspulse Integration Test Scenarios Contract

> Version: 0 (unstable — aligned with protocol v0)
> Applies to: all `wspulse/client-*` libraries

This document defines the **integration test scenarios** that every wspulse client library must cover against a live `wspulse/server` instance. Per-language test names and error types vary; the behaviour being tested is fixed.

For wire-level details see [`protocol.md`](https://github.com/wspulse/.github/blob/main/doc/protocol.md).
For API surface see [`interface.md`](./interface.md).
For behavioural requirements see [`behaviour.md`](./behaviour.md).

---

## Shared Testserver

All client libraries test against the same Go binary located at [`testserver/`](../../../testserver/). It runs a standard `wspulse/server` with control endpoints for test orchestration.

### Control API

| Method | Endpoint        | Purpose                           |
| ------ | --------------- | --------------------------------- |
| GET    | `/health`       | Liveness probe                    |
| POST   | `/kick?id=<id>` | Server-initiated disconnect       |
| POST   | `/shutdown`     | Stop WS listener (all dials fail) |
| POST   | `/restart`      | Rebind WS listener on same port   |

### WebSocket Query Parameters

| Param             | Effect                                                         |
| ----------------- | -------------------------------------------------------------- |
| `?reject=1`       | Server returns HTTP 403 on upgrade (simulates auth failure)    |
| `?room=<id>`      | Join a specific room                                           |
| `?id=<id>`        | Register a session ID (required for `/kick`)                   |
| `?ignore_pings=1` | Server suppresses Pong replies (for heartbeat timeout testing) |

---

## Scenario Matrix

Every client must implement tests for the following 9 scenarios. Error type names are conceptual — use the language-appropriate mapping from `interface.md`.

| #   | Scenario                                                            | Behaviour Reference                  | Query Params      | Notes                                   |
| --- | ------------------------------------------------------------------- | ------------------------------------ | ----------------- | --------------------------------------- |
| 1   | Connect → send → echo → close clean                                 | Lifecycle: INIT → CONNECTED → CLOSED | —                 | Basic happy-path                        |
| 2   | Server drops → `onTransportDrop` + `onDisconnect` (no reconnect)    | `onTransportDrop`, `onDisconnect`    | —                 | autoReconnect disabled                  |
| 3   | Auto-reconnect: server drops → reconnects within maxRetries         | Auto-Reconnect steps 1–5             | `?id=…`           | Use `/kick` to trigger drop             |
| 4   | Max retries exhausted → `onDisconnect(RetriesExhaustedError)`       | Auto-Reconnect step 7                | `?id=…`           | Use `/shutdown` to make all dials fail  |
| 5   | `close()` during reconnect → loop stops, `onDisconnect(nil)`        | `close()` Semantics                  | `?id=…`           | Call `close()` from `onReconnect`       |
| 6   | `send()` on closed client → `ConnectionClosedError`                 | `send()` Semantics                   | —                 |                                         |
| 7   | Heartbeat pong timeout → `ConnectionLostError`                      | Heartbeat: client-side               | `?ignore_pings=1` | Short `pingPeriod`/`pongWait` for speed |
| 8   | Concurrent sends: no data race or interleaving                      | `send()` Semantics: ordering         | —                 | See [Threading note](#threading-note)   |
| 9   | Concurrent `close()` + transport drop → `onDisconnect` exactly once | `close()` Semantics: idempotent      | —                 |                                         |

### Threading Note

Scenario 8 verifies that concurrent `send()` calls do not cause data races or message interleaving.

- **Multi-threaded runtimes** (Go, Kotlin, Swift): implement as a scenario matrix test — launch N goroutines/coroutines each sending M frames, verify all arrive intact and in per-sender order.
- **Single-threaded runtimes** (JavaScript/TypeScript): mark scenario 8 as N/A in the scenario matrix. Instead, add an equivalent test in the Additional Tests section (e.g. fire N sends via `Promise.all`, verify ordering and completeness).

The concurrent-send behaviour **must** be tested in every client; only the placement differs.

---

## Additional Tests

Beyond the scenario matrix, every client must include these tests:

| Test                      | What It Covers                                                               | Query Params |
| ------------------------- | ---------------------------------------------------------------------------- | ------------ |
| Frame field round-trip    | All `Frame` fields (`id`, `event`, `payload`) survive encode → wire → decode | —            |
| Server rejection handling | Server returns HTTP error on upgrade                                         | `?reject=1`  |
| Message ordering          | Send N frames, receive them in the same order                                | —            |
| Room routing              | Connect with `?room=<id>`, verify isolation                                  | `?room=…`    |
| Server-initiated kick     | `POST /kick?id=…` → `onDisconnect(error)`                                    | `?id=…`      |

---

## Minimum Test Count

| Runtime model   | Scenario matrix | Additional tests | Total |
| --------------- | --------------- | ---------------- | ----- |
| Multi-threaded  | 9               | 5                | 14    |
| Single-threaded | 8 (sc.8 = N/A)  | 6 (+concurrent)  | 14    |

All clients must reach a minimum of **14 integration tests**.

---

## Testserver Lifecycle

Each client library is responsible for starting/stopping the testserver during its test run. Common strategies:

- **Go**: `exec.Command` in `TestMain`.
- **TypeScript**: vitest `globalSetup` / `globalTeardown`.
- **Kotlin**: `ProcessBuilder` in `@BeforeAll` / `@AfterAll`.
- **Swift**: `Process` in `setUp` / `tearDown`.

The testserver binary must be built from `testserver/main.go` before the test suite starts. Ports are assigned via `PORT` and `CONTROL_PORT` environment variables.
