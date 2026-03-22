# wspulse Server Interface Contract

> Version: 0 (unstable — aligned with protocol v0)
> Applies to: `wspulse/server`

This document defines the **public API surface** of the wspulse server library. Implementation details (goroutine model, hub internals) are out of scope here — see [`server/doc/internals.md`](https://github.com/wspulse/server/blob/main/doc/internals.md) for those.

For wire-level details see [`protocol.md`](https://github.com/wspulse/.github/blob/main/doc/protocol.md).
For behavioural guarantees see [`behaviour.md`](./behaviour.md).

---

## Core Types

### Frame

Re-exported from `wspulse/core`. The minimal transport unit — all fields are optional at the wire layer.

| Field     | Go Type  | Description                                            |
| --------- | -------- | ------------------------------------------------------ |
| `ID`      | `string` | Opaque correlation ID. Omit if not needed.             |
| `Event`   | `string` | Application-defined event name.                        |
| `Payload` | `[]byte` | Opaque body. The server does not interpret this field. |

### Codec

Re-exported from `wspulse/core`. Encodes and decodes Frames for WebSocket transmission.

| Method        | Signature                                     | Description                                                  |
| ------------- | --------------------------------------------- | ------------------------------------------------------------ |
| `Encode`      | `(Frame) ([]byte, error)`                     | Serialize a Frame into wire data.                            |
| `Decode`      | `([]byte) (Frame, error)`                     | Deserialize received wire data into a Frame.                 |
| `FrameType`   | `() int`                                      | WebSocket message type: text (1) or binary (2).              |

The default codec is `JSONCodec` (text frames, JSON payload).

---

## Server Interface

The primary interface for managing WebSocket connections. Embeds `http.Handler`.

```go
type Server interface {
    http.Handler

    Send(connectionID string, f Frame) error
    Broadcast(roomID string, f Frame) error
    Kick(connectionID string) error
    GetConnections(roomID string) []Connection
    Close()
}
```

| Method           | Description                                                                                                    |
| ---------------- | -------------------------------------------------------------------------------------------------------------- |
| `ServeHTTP`      | Handles HTTP Upgrade requests. Calls `ConnectFunc` to authenticate and assign room/connection IDs.             |
| `Send`           | Enqueues a Frame for a specific connection. Returns `ErrConnectionNotFound` if the connection does not exist.  |
| `Broadcast`      | Enqueues a Frame for every active connection in a room. Returns `ErrServerClosed` if the server has shut down. |
| `Kick`           | Forcefully closes a connection, bypassing the resume window. Returns `ErrConnectionNotFound` if not found.     |
| `GetConnections` | Returns a snapshot of all connections in a room, including suspended sessions within the resume window.         |
| `Close`          | Gracefully shuts down the server: sends close frames, drains registrations, fires `OnDisconnect` for all.      |

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

| Method   | Description                                                                  |
| -------- | ---------------------------------------------------------------------------- |
| `ID`     | Unique connection identifier provided by `ConnectFunc`.                      |
| `RoomID` | Room this connection belongs to, as provided by `ConnectFunc`.               |
| `Send`   | Enqueues a Frame for delivery. Returns `ErrConnectionClosed` or `ErrSendBufferFull` on failure. |
| `Close`  | Initiates graceful shutdown of the session.                                  |
| `Done`   | Channel closed when the session is terminated.                               |

---

## ConnectFunc

Authenticates incoming HTTP Upgrade requests and assigns room and connection identifiers.

```go
type ConnectFunc func(r *http.Request) (roomID, connectionID string, err error)
```

- Returning a non-nil `error` rejects the upgrade with HTTP 401 and the error text as the response body.
- If `connectionID` is empty, the server assigns a random UUID.
- Use a non-empty `connectionID` when the application needs deterministic IDs (e.g. for `Server.Send` and `Server.Kick`).

---

## Constructor

```go
func NewServer(connect ConnectFunc, options ...ServerOption) Server
```

- `connect` must not be nil (panics if nil — setup-time programmer error).
- Starts the hub event loop immediately.
- Returns a ready-to-use `Server`.

---

## Server Options

All options are set via functional option builders. Invalid values panic at setup time (programmer error).

| Option                        | Signature / Type                       | Default              | Valid Range            |
| ----------------------------- | -------------------------------------- | -------------------- | ---------------------- |
| `WithOnConnect(fn)`           | `func(Connection)`                     | --                   | --                     |
| `WithOnMessage(fn)`           | `func(Connection, Frame)`              | --                   | --                     |
| `WithOnDisconnect(fn)`        | `func(Connection, error)`              | --                   | --                     |
| `WithHeartbeat(ping, pong)`   | `(time.Duration, time.Duration)`       | 10 s / 30 s          | (0, 5m] / (ping, 10m] |
| `WithWriteWait(d)`            | `time.Duration`                        | 10 s                 | (0, 30s]               |
| `WithMaxMessageSize(n)`       | `int64`                                | 512 B                | [1, 64 MiB]            |
| `WithSendBufferSize(n)`       | `int`                                  | 256 frames           | [1, 4096]              |
| `WithResumeWindow(d)`         | `time.Duration`                        | 0 (disabled)         | [0, 3m]                |
| `WithCodec(c)`                | `Codec`                                | `JSONCodec`          | non-nil                |
| `WithCheckOrigin(fn)`         | `func(*http.Request) bool`             | allow all            | non-nil                |
| `WithLogger(l)`               | `*zap.Logger`                          | `zap.NewNop()`       | non-nil                |

### Callback execution context

| Callback         | Goroutine                                | Notes                                                                                 |
| ---------------- | ---------------------------------------- | ------------------------------------------------------------------------------------- |
| `OnConnect`      | Separate goroutine (spawned by hub)      | Fires after successful registration.                                                  |
| `OnMessage`      | Connection's `readPump` goroutine        | Must return quickly; use a goroutine for heavy work. Single-threaded per connection.   |
| `OnDisconnect`   | Separate goroutine (spawned by hub)      | Fires once per session. With resume window: only after grace expires without reconnect. |

---

## Sentinel Errors

### Server-only errors

| Error                      | When returned / passed                                                              |
| -------------------------- | ----------------------------------------------------------------------------------- |
| `ErrConnectionNotFound`    | `Server.Send` or `Server.Kick` when `connectionID` has no active connection.        |
| `ErrDuplicateConnectionID` | Passed to `OnDisconnect` when an existing session is kicked by a new registration with the same ID. |
| `ErrServerClosed`          | `Server.Broadcast` (and potentially other methods) after `Server.Close()`.          |

### Re-exported from core

| Error                  | When returned                                                   |
| ---------------------- | --------------------------------------------------------------- |
| `ErrConnectionClosed`  | `Connection.Send` after the session is closed.                  |
| `ErrSendBufferFull`    | `Connection.Send` when the per-connection send buffer is full.  |

---

## Language Mapping

Currently Go only. Structured for future server ports.

| Concept          | Go (`server`)                                              |
| ---------------- | ---------------------------------------------------------- |
| Entry point      | `NewServer(connect, ...opts) Server`                       |
| Server interface | `Server` (embeds `http.Handler`)                           |
| Connection       | `Connection` interface                                     |
| Frame            | `Frame` (re-exported from `core`)                          |
| Codec            | `Codec` interface (re-exported from `core`)                |
| Default codec    | `JSONCodec`                                                |
| Auth hook        | `ConnectFunc func(*http.Request) (string, string, error)`  |
| Options          | `ServerOption` functional options                          |
| Logger           | `*zap.Logger`                                              |
