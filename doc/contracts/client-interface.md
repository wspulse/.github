# wspulse Client Interface Contract

> Version: 0 (unstable — aligned with protocol v0)
> Applies to: all `wspulse/client-*` libraries

This document defines the **language-agnostic API surface** that every wspulse client library must expose. Naming and syntax are adapted per language; semantics are fixed.

For wire-level details see [`server/doc/protocol.md`](https://github.com/wspulse/server/blob/main/doc/protocol.md).
For behavioural contracts (lifecycle, callbacks, reconnect semantics) see [`client-behaviour.md`](./client-behaviour.md).

---

## Core Concepts

### Frame

The minimal transport unit. All fields are optional at the wire layer.

| Field     | Type           | Description                                            |
| --------- | -------------- | ------------------------------------------------------ |
| `id`      | string         | Opaque correlation ID. Omit if not needed.             |
| `event`   | string         | Application-defined event name.                        |
| `payload` | any JSON value | Opaque body. The client does not interpret this field. |

### Codec

Encodes and decodes Frames for WebSocket transmission. Every client library must accept an optional Codec and ship a default JSON codec.

| Concept       | Requirement                                                                             |
| ------------- | --------------------------------------------------------------------------------------- |
| **Encode**    | Serialize a Frame into wire data (`string` for text frames, `bytes` for binary frames). |
| **Decode**    | Deserialize received wire data into a Frame.                                            |
| **FrameType** | Declare whether the codec produces text frames (opcode 1) or binary frames (opcode 2).  |

The default codec (`JSONCodec`) serializes Frames as JSON text frames. Custom codecs (e.g. Protocol Buffers) may use binary frames. The codec determines the WebSocket message type used for sending and the expected format for receiving.

### Client

The object returned by the entry-point function. Must expose:

| Concept   | Requirement                                                                                                                                                    |
| --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Send**  | Enqueue a Frame for delivery. Returns/throws `ConnectionClosedError` if the client is closed. Returns/throws `SendBufferFullError` if the send buffer is full. |
| **Close** | Permanently terminate the connection and stop any reconnect loop. Idempotent.                                                                                  |
| **Done**  | A signal (channel / Promise / Event / AsyncStream) that resolves when permanently disconnected.                                                                |

---

## Entry Point

```
connect(url, options) → Client
```

- `url`: WebSocket URL string (e.g. `wss://host/ws?token=…`)
- `options`: language-specific options object / builder / named parameters
- Returns a connected Client, or throws/returns an error if the initial connection fails.
- **Initial dial failure is always fatal** — even when `autoReconnect` is enabled. Auto-reconnect only activates after a successful initial connection. Initial failures typically indicate configuration errors (wrong URL, auth failure, server unreachable) that retries cannot fix. The caller must handle the error explicitly (e.g. retry with corrected config, surface to the user, or abort).

---

## Options (Required)

Every implementation must support these options:

| Option            | Type / Signature                    | Default     | Description                                                                                                                                                                                                    |
| ----------------- | ----------------------------------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `onMessage`       | `(Frame) → void`                    | no-op       | Called for every inbound frame.                                                                                                                                                                                |
| `onDisconnect`    | `(error\|null) → void`              | no-op       | Called on permanent disconnect. `null`/`nil` = clean close. See behaviour doc for full semantics.                                                                                                              |
| `onReconnect`     | `(attempt: int) → void`             | no-op       | Called at each reconnect attempt. `attempt` is 0-based.                                                                                                                                                        |
| `onTransportDrop` | `(error) → void`                    | no-op       | Called each time the underlying transport drops (before any reconnect).                                                                                                                                        |
| `autoReconnect`   | `(maxRetries, baseDelay, maxDelay)` | disabled    | Enable exponential backoff reconnect. `maxRetries ≤ 0` = unlimited.                                                                                                                                            |
| `heartbeat`       | `(pingPeriod, pongWait)`            | 20 s / 60 s | Client-side Ping/Pong interval. The client sends Ping every `pingPeriod` and closes the socket if no Pong arrives within `pongWait`. Browser clients: no-op (browser handles Ping/Pong at the protocol level). |
| `writeWait`       | duration                            | 10 s        | Deadline for a single write operation.                                                                                                                                                                         |
| `maxMessageSize`  | bytes (int)                         | 1 MiB       | Max inbound message size. Connection closed if exceeded.                                                                                                                                                       |
| `dialHeaders`     | map\<string, string\>               | none        | Extra HTTP headers sent during WebSocket upgrade.                                                                                                                                                              |
| `codec`           | Codec                               | JSONCodec   | Wire-format codec for encoding/decoding Frames. Custom codecs enable binary protocols.                                                                                                                         |

---

## Backoff Formula

All implementations must use this formula:

```
delay       = min(baseDelay × 2^attempt, maxDelay)
jitter      = uniform random in [0.5, 1.0]   (equal jitter)
result      = delay × jitter
```

This matches `client-go` exactly. Any deviation is a bug.

---

## Error / Exception Values

Each language maps these sentinel concepts to its own error type. The names below are conceptual:

| Concept                 | When raised / returned                                                    |
| ----------------------- | ------------------------------------------------------------------------- |
| `ConnectionClosedError` | `send()` called after the client is permanently closed.                   |
| `SendBufferFullError`   | `send()` called when the internal send buffer is full.                    |
| `RetriesExhaustedError` | Passed to `onDisconnect` when max reconnect retries are exhausted.        |
| `ConnectionLostError`   | Passed to `onDisconnect` when the server drops and auto-reconnect is off. |

---

## Language Mapping

| Concept          | Go (`client-go`)                       | TypeScript (`client-ts`)           | Kotlin (`client-kt`)                     | Swift (`client-swift`)                   | Python (`client-py`)                            |
| ---------------- | -------------------------------------- | ---------------------------------- | ---------------------------------------- | ---------------------------------------- | ----------------------------------------------- |
| Entry point      | `Dial(url, ...opts) (Client, error)`   | `connect(url, opts): Client`       | `WspulseClient.connect(url, config)`     | `WspulseClient(url:options:).connect()`  | `async with wspulse.connect(url, **opts) as c:` |
| Send             | `client.Send(Frame) error`             | `client.send(frame): void`         | `suspend client.send(frame)`             | `await client.send(frame)`               | `await client.send(frame)`                      |
| Close            | `client.Close() error`                 | `client.close(): void`             | `suspend client.close()`                 | `await client.close()`                   | `await client.close()`                          |
| Done signal      | `client.Done() <-chan struct{}`        | `client.done: Promise<void>`       | `client.done: CompletableDeferred<Unit>` | `client.done: AsyncStream<Void>`         | `client.done: asyncio.Event`                    |
| Frame type       | `core.Frame{ID, Event, Payload}`       | `{ id?, event?, payload? }`        | `data class Frame(id, event, payload)`   | `struct Frame { id, event, payload }`    | `@dataclass Frame(id, event, payload)`          |
| Codec interface  | `core.Codec` (Encode/Decode/FrameType) | `Codec` (encode/decode/binaryType) | `Codec` (encode/decode/frameType)        | `WspulseCodec` (encode/decode/frameType) | `Codec` (encode/decode/frame_type)              |
| Default codec    | `core.JSONCodec`                       | `JSONCodec`                        | `JsonCodec`                              | `JSONCodec`                              | `JSONCodec`                                     |
| ConnectionClosed | `ErrConnectionClosed` (from core)      | `ConnectionClosedError`            | `ConnectionClosedException`              | `WspulseError.connectionClosed`          | `ConnectionClosedError`                         |
| SendBufferFull   | `ErrSendBufferFull` (from core)        | `SendBufferFullError`              | `SendBufferFullException`                | `WspulseError.sendBufferFull`            | `SendBufferFullError`                           |
| RetriesExhausted | `ErrRetriesExhausted`                  | `RetriesExhaustedError`            | `RetriesExhaustedException`              | `WspulseError.retriesExhausted`          | `RetriesExhaustedError`                         |
| ConnectionLost   | `ErrConnectionLost`                    | `ConnectionLostError`              | `ConnectionLostException`                | `WspulseError.connectionLost`            | `ConnectionLostError`                           |
