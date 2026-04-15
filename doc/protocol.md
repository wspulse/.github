# wspulse Wire Protocol

Version: 0 (unstable — evolves until v1)

---

## Transport

wspulse uses RFC 6455 WebSocket as the transport layer. Clients connect using
any standard WebSocket library; no custom handshake is required beyond the
standard HTTP Upgrade.

---

## Frame Format (JSON codec — default)

Every message exchanged between server and client is a **Frame** encoded as a
JSON object in a WebSocket **text** frame:

```json
{
  "event":   "<string>",
  "payload": <any valid JSON value>
}
```

| Field     | Required | Description                                                            |
| --------- | -------- | ---------------------------------------------------------------------- |
| `event`   | No       | Application-defined string classifying the frame purpose.              |
| `payload` | No       | Any valid JSON value. The wspulse layer does not interpret this field. |

All fields are optional at the transport layer. Their semantics are defined by
the application layer built on top of wspulse.

### Suggested event values

These are conventions only — applications may use any string:

| Value   | Meaning                                        |
| ------- | ---------------------------------------------- |
| `"msg"` | User-generated data message                    |
| `"sys"` | System or control event                        |
| `"ack"` | Acknowledgement of a previously received frame |

### Example frames

Chat message (server → client):

```json
{
  "event": "msg",
  "payload": { "msgId": "01JXABC", "text": "hello", "user": "alice" }
}
```

System event (server → client):

```json
{
  "event": "sys",
  "payload": { "event": "member_join", "user_id": "bob" }
}
```

Acknowledgement (client → server):

```json
{ "event": "ack", "payload": { "ref": "01JXABC" } }
```

> **Note:** Correlation IDs such as `msgId` and `ref` above are
> application-layer concerns carried inside `payload`. The wspulse
> transport layer does not interpret or generate these fields.

---

## Frame Format (Custom codec)

When a custom `Codec` is injected via `WithCodec`, messages are sent as
WebSocket **binary** or **text** frames depending on the codec's `FrameType()`
return value. The `Codec` interface handles encoding and decoding; wspulse is
agnostic to the on-wire format (Protobuf, MessagePack, CBOR, etc.).

---

## Heartbeat

wspulse uses a **server-initiated heartbeat** model — the server sends
WebSocket Ping control frames and monitors Pong replies.

**Server → Client:** The server sends a **Ping** every `pingInterval`
(default 20 s) via a dedicated `pingPump` goroutine. Each Ping blocks
synchronously until the Pong reply arrives or `writeTimeout` (default 10 s)
elapses. If no Pong arrives within `writeTimeout`, the server closes the
connection. The constraint `pingInterval > writeTimeout` is always enforced.

Standard WebSocket clients (browsers, `coder/websocket`, gorilla, and other
libraries) auto-reply Pong at the protocol layer — no application-level
handling is needed. Native clients (Go, Node.js, etc.) may also send their
own Pings for client-side dead-connection detection; the server auto-replies.

> **Browser note:** The browser WebSocket API does not expose Ping/Pong
> control frames. Browser clients rely entirely on the server-side
> heartbeat for liveness detection.

---

## Connection Lifecycle

```
Client                           Server
  |                                |
  |--- HTTP GET /ws (Upgrade) ---->|
  |<-- 101 Switching Protocols ----|
  |                                |
  |     [frames exchanged]         |
  |                                |
  |<----------- Ping --------------|  (server pingInterval, default 20 s)
  |------------ Pong ------------->|  (auto-reply)
  |                                |
  |--------- Close frame --------->|  (normal close by client)
  |<-------- Close frame ----------|
  |                                |
```

Abnormal closure (network drop, server-side Kick) terminates the TCP
connection and triggers the `OnDisconnect` callback on the server side.

---

## Session Resumption

When `WithResumeWindow` is configured, a disconnected client may reconnect
using the same `connectionID` (as returned by `ConnectFunc`) within the resume
window. The server transparently swaps the underlying WebSocket and replays
any frames buffered during the gap. **No changes to the wire protocol are
required** — the reconnect is a standard HTTP Upgrade with the same
`connectionID` negotiated via `ConnectFunc`.

The server does **not** fire `OnConnect` or `OnDisconnect` callbacks during a
successful resume. From the application layer's perspective, the `Connection`
never dropped.

---

## Versioning

This protocol document is versioned alongside the wspulse module. Breaking
changes to the wire format will increment the major version of the module.
