# wspulse Hub Interface Contract

> Version: 0 (unstable — aligned with protocol v0)
> Applies to: `wspulse/hub`

This document defines the **public API surface** of the wspulse hub library. Implementation details (goroutine model, hub internals) are out of scope here — see [`hub/doc/internals.md`](https://github.com/wspulse/hub/blob/main/doc/internals.md) for those.

For wire-level details see [`protocol.md`](https://github.com/wspulse/.github/blob/main/doc/protocol.md).
For behavioural guarantees see [`behaviour.md`](./behaviour.md).

---

## Core Types

### Frame

Re-exported from `wspulse/core`. The minimal transport unit — all fields are optional at the wire layer.

| Field     | Go Type  | Description                                          |
| --------- | -------- | ---------------------------------------------------- |
| `Event`   | `string` | Application-defined event name.                      |
| `Payload` | `[]byte` | Opaque body. The hub does not interpret this field.  |

### Codec

Re-exported from `wspulse/core`. Encodes and decodes Frames for WebSocket transmission.

| Method      | Signature                 | Description                                     |
| ----------- | ------------------------- | ----------------------------------------------- |
| `Encode`    | `(Frame) ([]byte, error)` | Serialize a Frame into wire data.               |
| `Decode`    | `([]byte) (Frame, error)` | Deserialize received wire data into a Frame.    |
| `FrameType` | `() int`                  | WebSocket message type: text (1) or binary (2). |

The default codec is `JSONCodec` (text frames, JSON payload).

---

## Hub Interface

The primary interface for managing WebSocket connections. Embeds `http.Handler`.

```go
type Hub interface {
    http.Handler

    Send(connectionID string, f Frame) error
    Broadcast(roomID string, f Frame) error
    Kick(connectionID string) error
    GetConnections(roomID string) []Connection
    Close()
}
```

| Method           | Description                                                                                                   |
| ---------------- | ------------------------------------------------------------------------------------------------------------- |
| `ServeHTTP`      | Handles HTTP Upgrade requests. Calls `ConnectFunc` to authenticate and assign room/connection IDs.            |
| `Send`           | Enqueues a Frame for a specific connection. Returns `ErrConnectionNotFound` if the connection does not exist. |
| `Broadcast`      | Enqueues a Frame for every active connection in a room. Returns `ErrHubClosed` if the hub has shut down.      |
| `Kick`           | Forcefully closes a connection, bypassing the resume window. Returns `ErrConnectionNotFound` if not found.    |
| `GetConnections` | Returns a snapshot of all connections in a room, including suspended sessions within the resume window.       |
| `Close`          | Gracefully shuts down the hub: sends close frames, drains registrations, fires `OnDisconnect` for all.        |

---

## Connection Interface

Represents a logical WebSocket session. The underlying transport may be swapped transparently on reconnect when `WithResumeWindow` is configured. All methods are safe for concurrent use.

```go
type Connection interface {
    ID() string
    RoomID() string
    Send(f Frame) error
    Close() error
    Done() <-chan struct{}
}
```

| Method   | Description                                                                                     |
| -------- | ----------------------------------------------------------------------------------------------- |
| `ID`     | Unique connection identifier provided by `ConnectFunc`.                                         |
| `RoomID` | Room this connection belongs to, as provided by `ConnectFunc`.                                  |
| `Send`   | Enqueues a Frame for delivery. Returns `ErrConnectionClosed` or `ErrSendBufferFull` on failure. |
| `Close`  | Initiates graceful shutdown of the session.                                                     |
| `Done`   | Channel closed when the session is terminated.                                                  |

---

## ConnectFunc

Authenticates incoming HTTP Upgrade requests and assigns room and connection identifiers.

```go
type ConnectFunc func(r *http.Request) (roomID, connectionID string, err error)
```

- Returning a non-nil `error` rejects the upgrade with HTTP 401 and the error text as the response body.
- If `connectionID` is empty, the hub assigns a random UUID.
- Use a non-empty `connectionID` when the application needs deterministic IDs (e.g. for `Hub.Send` and `Hub.Kick`).

---

## Constructor

```go
func NewHub(connect ConnectFunc, options ...HubOption) Hub
```

- `connect` must not be nil (panics if nil — setup-time programmer error).
- Panics if `pingInterval <= writeTimeout` after all options are applied.
- Starts the hub event loop immediately.
- Returns a ready-to-use `Hub`.

---

## Hub Options

All options are set via functional option builders. Invalid values panic at setup time (programmer error).

| Option                       | Signature / Type                 | Default         | Valid Range           |
| ---------------------------- | -------------------------------- | --------------- | --------------------- |
| `WithOnConnect(fn)`          | `func(Connection)`               | --              | --                    |
| `WithOnMessage(fn)`          | `func(Connection, Frame)`        | --              | --                    |
| `WithOnDisconnect(fn)`       | `func(Connection, error)`        | --              | --                    |
| `WithOnTransportDrop(fn)`    | `func(Connection, error)`        | --              | --                    |
| `WithOnTransportRestore(fn)` | `func(Connection)`               | --              | --                    |
| `WithPingInterval(d)`        | `time.Duration`                  | 20 s            | (writeTimeout, 1m]    |
| `WithWriteTimeout(d)`        | `time.Duration`                  | 10 s            | (0, 30s]              |
| `WithMaxMessageSize(n)`      | `int64`                          | 512 B           | [1, 64 MiB]           |
| `WithSendBufferSize(n)`      | `int`                            | 256 frames      | [1, 4096]             |
| `WithResumeWindow(d)`        | `time.Duration`                  | 0 (disabled)    | [0, +∞)               |
| `WithCodec(c)`               | `Codec`                          | `JSONCodec`     | non-nil               |
| `WithMetrics(mc)`            | `MetricsCollector`               | `NoopCollector` | non-nil               |
| `WithLogger(l)`              | `*zap.Logger`                    | `zap.NewNop()` | non-nil                |

### Callback execution context

| Callback             | Goroutine                           | Notes                                                                                   |
| -------------------- | ----------------------------------- | --------------------------------------------------------------------------------------- |
| `OnConnect`          | Separate goroutine (spawned by hub) | Fires after successful registration.                                                    |
| `OnMessage`          | Connection's `readPump` goroutine   | Must return quickly; use a goroutine for heavy work. Single-threaded per connection.    |
| `OnDisconnect`       | Separate goroutine (spawned by hub) | Fires once per session. With resume window: only after grace expires without reconnect. |
| `OnTransportDrop`    | Separate goroutine (spawned by hub) | Fires when transport dies and session enters SUSPENDED. Only when `resumeWindow > 0`.   |
| `OnTransportRestore` | Separate goroutine (spawned by hub) | Fires when a suspended session resumes via reconnect. Only when `resumeWindow > 0`.     |

---

## Config Validation

All option values are validated at setup time. Invalid configuration panics immediately — it is a programmer error, not a runtime condition.

All validation panic messages must use the prefix `wspulse:` followed by a space, then the function name and a colon.

### Validation Rules

| #   | Option               | Field            | Condition                     | Panic message                                                               |
| --- | -------------------- | ---------------- | ----------------------------- | --------------------------------------------------------------------------- |
| 1   | `NewHub`             | `connect`        | `!= nil`                      | `wspulse: NewHub: connect must not be nil`                                  |
| 2   | `NewHub`             | cross-constraint | `pingInterval > writeTimeout` | `wspulse: NewHub: pingInterval must be greater than writeTimeout`           |
| 3   | `WithPingInterval`   | `d`              | `> 0`                         | `wspulse: WithPingInterval: duration must be positive`                      |
| 4   | `WithPingInterval`   | `d`              | `<= 1 m`                      | `wspulse: WithPingInterval: duration exceeds maximum (1m0s)`                |
| 5   | `WithWriteTimeout`   | `d`              | `> 0`                         | `wspulse: WithWriteTimeout: duration must be positive`                      |
| 6   | `WithWriteTimeout`   | `d`              | `<= 30 s`                     | `wspulse: WithWriteTimeout: duration exceeds maximum (30s)`                 |
| 7   | `WithMaxMessageSize` | `n`              | `>= 1`                        | `wspulse: WithMaxMessageSize: n must be at least 1`                         |
| 8   | `WithMaxMessageSize` | `n`              | `<= 64 MiB`                   | `wspulse: WithMaxMessageSize: n exceeds maximum (64 MiB)`                   |
| 9   | `WithSendBufferSize` | `n`              | `>= 1`                        | `wspulse: WithSendBufferSize: n must be at least 1`                         |
| 10  | `WithSendBufferSize` | `n`              | `<= 4096`                     | `wspulse: WithSendBufferSize: n exceeds maximum (4096)`                     |
| 11  | `WithResumeWindow`   | `d`              | `>= 0`                        | `wspulse: WithResumeWindow: duration must be non-negative`                  |
| 12  | `WithCodec`          | `codec`          | `!= nil`                      | `wspulse: WithCodec: codec must not be nil`                                 |
| 13  | `WithMetrics`        | `mc`             | `!= nil`                      | `wspulse: WithMetrics: collector must not be nil`                           |
| 14  | `WithLogger`         | `l`              | `!= nil`                      | `wspulse: WithLogger: logger must not be nil`                               |

Notes:

- Rule #2 is a cross-option constraint enforced by `NewHub` after all options have been applied. `WithPingInterval` and `WithWriteTimeout` individually only check their own bounds.
- `WithResumeWindow(0)` is valid — it disables session resumption (the default).
- `maxMessageSize` minimum is 1. The hub always enforces a message size limit.
- Callback options (`WithOnConnect`, `WithOnMessage`, `WithOnDisconnect`, `WithOnTransportDrop`, `WithOnTransportRestore`) accept `nil` — nil means no callback registered.
- Upper bounds are intentionally conservative for a v0 library. They can be relaxed in future versions without breaking existing callers.
- Boundary values (e.g. `sendBufferSize = 4096`, `writeTimeout = 30s`) are valid and must be accepted.

---

## Sentinel Errors

### Hub-only errors

| Error                      | When returned / passed                                                                              |
| -------------------------- | --------------------------------------------------------------------------------------------------- |
| `ErrConnectionNotFound`    | `Hub.Send` or `Hub.Kick` when `connectionID` has no active connection.                              |
| `ErrDuplicateConnectionID` | Passed to `OnDisconnect` when an existing session is kicked by a new registration with the same ID. |
| `ErrHubClosed`             | `Hub.Broadcast` (and potentially other methods) after `Hub.Close()`.                                |

### Re-exported from core

| Error                 | When returned                                                  |
| --------------------- | -------------------------------------------------------------- |
| `ErrConnectionClosed` | `Connection.Send` after the session is closed.                 |
| `ErrSendBufferFull`   | `Connection.Send` when the per-connection send buffer is full. |

---

## Language Mapping

Currently Go only. Structured for future hub ports.

| Concept       | Go (`hub`)                                                |
| ------------- | --------------------------------------------------------- |
| Entry point   | `NewHub(connect, ...opts) Hub`                            |
| Hub interface | `Hub` (embeds `http.Handler`)                             |
| Connection    | `Connection` interface                                    |
| Frame         | `Frame` (re-exported from `core`)                         |
| Codec         | `Codec` interface (re-exported from `core`)               |
| Default codec | `JSONCodec`                                               |
| Auth hook     | `ConnectFunc func(*http.Request) (string, string, error)` |
| Options       | `HubOption` functional options                            |
| Logger        | `*zap.Logger`                                             |
