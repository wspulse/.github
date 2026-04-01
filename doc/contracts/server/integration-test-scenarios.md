# wspulse Server Integration Test Scenarios

> Version: 0 (unstable — aligned with protocol v0)
> Applies to: `wspulse/server`

This document defines the **integration test scenarios** that the wspulse server must cover. These tests validate the server's behavioural guarantees from the perspective of a connected client.

For API surface see [`interface.md`](./interface.md).
For behavioural guarantees see [`behaviour.md`](./behaviour.md).
For wire-level details see [`protocol.md`](https://github.com/wspulse/.github/blob/main/doc/protocol.md).

---

## Test Infrastructure

Server integration tests use real WebSocket connections. The test harness creates a `wspulse.Server` instance, binds it to a local `httptest.Server`, and connects clients via `gorilla/websocket` (or `client-go`).

---

## Scenario Matrix

| #   | Scenario                              | Behaviour Reference                     | Pass Condition                                                                             |
| --- | ------------------------------------- | --------------------------------------- | ------------------------------------------------------------------------------------------ |
| 1   | Connect and receive OnConnect         | Callback ordering                       | `OnConnect` fires exactly once with valid `Connection`.                                    |
| 2   | Send frame to connection              | `Server.Send`                           | Client receives the frame with correct `Event`, `Payload`.                                 |
| 3   | Broadcast to room                     | `Server.Broadcast`                      | All connections in the room receive the frame; connections in other rooms do not.           |
| 4   | Client disconnect (clean close)       | Teardown                                | `OnDisconnect` fires with `nil` error. `Connection.Done()` closes.                        |
| 5   | Client disconnect (transport drop)    | Teardown                                | `OnDisconnect` fires with non-nil error. `Connection.Done()` closes.                      |
| 6   | Session resume within window          | Resume window interaction               | Client reconnects, no `OnConnect`/`OnDisconnect` fires, buffered frames replayed.          |
| 7   | Session resume window expires         | Resume window interaction               | `OnDisconnect` fires after grace period. Subsequent reconnect creates new session.         |
| 8   | Kick bypasses resume window           | Kick semantics                          | `OnDisconnect` fires immediately. No suspended state.                                      |
| 9   | Kick non-existent connection          | Kick semantics                          | Returns `ErrConnectionNotFound`.                                                            |
| 10  | Graceful shutdown                     | Graceful shutdown                       | All clients receive close frames. `OnDisconnect` fires for every session. No goroutine leaks. |
| 11  | OnTransportDrop fires on suspend      | Callback ordering, Resume window        | `OnTransportDrop(conn, err)` fires when transport dies and `resumeWindow > 0`. `err` is non-nil. |
| 12  | OnTransportRestore fires on resume    | Callback ordering, Resume window        | `OnTransportRestore(conn)` fires when suspended session resumes. Same `Connection` object.    |
| 13  | Transport callbacks skip without resume | Callback ordering                     | Neither `OnTransportDrop` nor `OnTransportRestore` fires when `resumeWindow == 0`.            |

---

## Additional Tests

| Test                                  | What It Covers                                                                              |
| ------------------------------------- | ------------------------------------------------------------------------------------------- |
| ConnectFunc rejection                 | `ConnectFunc` returns error → HTTP 401 response, no session created, no callbacks fire.     |
| Duplicate connectionID                | New connection with same ID kicks old session with `ErrDuplicateConnectionID`.              |
| Send to unknown connectionID          | `Server.Send` returns `ErrConnectionNotFound`.                                              |
| Broadcast after Close                 | `Server.Broadcast` returns `ErrServerClosed`.                                               |
| Send buffer full (direct Send)        | `Connection.Send` returns `ErrSendBufferFull` when buffer is at capacity.                   |
| Broadcast backpressure (drop-oldest)  | Slow consumer's oldest frame is dropped; other consumers receive all frames.                |
| Heartbeat timeout                     | Server closes connection after no Pong within `pongWait`. `OnDisconnect` fires.             |
| OnMessage ordering                    | Frames sent by client arrive at `OnMessage` in order.                                       |
| Resume buffer replay ordering         | Frames buffered during suspension are replayed in original order after reconnect.            |
| GetConnections includes suspended     | `GetConnections` returns suspended sessions within the resume window.                       |
| Empty connectionID gets UUID          | When `ConnectFunc` returns empty `connectionID`, server assigns a non-empty UUID.            |
| Concurrent Send from multiple goroutines | Multiple goroutines calling `Server.Send` concurrently — no data race, all frames delivered. |
| OnConnect/OnDisconnect goroutine safety | Callbacks run in separate goroutines and do not block the hub event loop.                   |
| OnTransportDrop then OnDisconnect ordering | `OnTransportDrop` fires first, then `OnDisconnect` after grace expires. No overlap.       |
| OnTransportRestore before OnMessage     | No `OnMessage` from the new transport fires until after `OnTransportRestore` completes.     |

---

## Minimum Test Count

| Category        | Count |
| --------------- | ----- |
| Scenario matrix | 13    |
| Additional      | 15    |
| **Total**       | **28** |

---

## Goroutine Leak Detection

All server integration tests must use `go.uber.org/goleak` in `TestMain` to verify that no goroutines are leaked after test completion. This catches:

- Orphaned `readPump` / `writePump` goroutines.
- Hub goroutine not exiting after `Server.Close()`.
- Grace timer goroutines not cancelled on shutdown.
