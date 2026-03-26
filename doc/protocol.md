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
  "id":      "<uuid or empty>",
  "event":   "<string>",
  "payload": <any valid JSON value>
}
```

| Field     | Required | Description                                                            |
| --------- | -------- | ---------------------------------------------------------------------- |
| `id`      | No       | Opaque string used for ACK correlation. Omitted if empty.              |
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
  "id": "01JXABC",
  "event": "msg",
  "payload": { "text": "hello", "user": "alice" }
}
```

System event (server → client):

```json
{
  "id": "01JXABD",
  "event": "sys",
  "payload": { "event": "member_join", "user_id": "bob" }
}
```

Acknowledgement (client → server):

```json
{ "event": "ack", "payload": { "id": "01JXABC" } }
```

---

## Frame Format (Custom codec)

When a custom `Codec` is injected via `WithCodec`, messages are sent as
WebSocket **binary** or **text** frames depending on the codec's `FrameType()`
return value. The `Codec` interface handles encoding and decoding; wspulse is
agnostic to the on-wire format (Protobuf, MessagePack, CBOR, etc.).

---

## Heartbeat

wspulse uses a **dual heartbeat** model — both sides independently send
WebSocket Ping control frames and monitor Pong replies.

**Server → Client:** The server sends a **Ping** every `pingPeriod`
(default 10 s). Clients auto-reply with a **Pong** at the protocol layer
(gorilla, browsers, and other standard WebSocket libraries handle this
automatically). If the server receives no Pong within `pongWait`
(default 30 s), it closes the connection.

**Client → Server:** Native clients (Go, Node.js) **also** send their
own **Ping** every `pingPeriod` (default 20 s). The server auto-replies
with a **Pong** (gorilla default `PingHandler`). If the client receives
no Pong within `pongWait` (default 60 s), it closes the socket and
triggers a transport drop.

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
  |<----------- Ping --------------|  (server pingPeriod, default 10 s)
  |------------ Pong ------------->|  (auto-reply)
  |                                |
  |------------ Ping ------------->|  (client pingPeriod, default 20 s)
  |<----------- Pong --------------|  (auto-reply)
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

## Resume Intent

When a client reconnects after a transport failure, it MUST include
`resume=true` as a query parameter in the WebSocket upgrade URL:

```
ws://host/ws?token=abc&resume=true
```

This signals to the server that the client intends to resume a previously
suspended session. The `resume` parameter is a boolean flag; the
`connectionID` is conveyed separately via `ConnectFunc` (application-layer
responsibility, not part of this protocol).

### Server behaviour

The server evaluates the `resume` parameter in `ServeHTTP`, **before** the
WebSocket upgrade:

| Condition | Action |
| --- | --- |
| `resume=true`, session exists (any state) | Upgrade, resume session (existing behaviour) |
| `resume=true`, session does not exist | **HTTP 410 Gone** — do not upgrade |
| `resume` absent or false | Upgrade, create new session (existing behaviour) |

**HTTP 410 Gone** in the wspulse context means: the client requested session
resumption but the session no longer exists. The grace window has expired or
the server has restarted. Buffered messages from the previous session are
lost. The client MUST NOT automatically retry with `resume=true`.

### Race condition

In rare cases the HTTP pre-check may pass (session exists) but the session is
destroyed by the grace timer before the hub processes the registration. When
this happens, the server sends a WebSocket close frame with code **4100**
(`CloseSessionExpired`) and does not create a new session. The client handles
this identically to HTTP 410.

### Close code 4100 — CloseSessionExpired

| Code | Name | Meaning |
| --- | --- | --- |
| 4100 | `CloseSessionExpired` | Session resumption requested but the session no longer exists. Sent only in the upgrade-then-discover race case described above. |

### Metrics

The server emits `ResumeAttempt(roomID, connectionID, false)` for every
rejected resume — whether via HTTP 410 or close code 4100.

### Security

The `resume` parameter is an intent signal only. Session ownership is
verified by `ConnectFunc`, which authenticates the HTTP request and returns
the `connectionID`. **`ConnectFunc` MUST ensure that a reconnecting client is
the same identity that originally created the session.** Failure to do so
allows session hijacking: an attacker who passes `ConnectFunc` with a known
`connectionID` could resume another user's session and receive their buffered
messages.

wspulse does not enforce session ownership at the protocol layer. This is an
application-layer responsibility, analogous to HTTP session token security.

---

## Versioning

This protocol document is versioned alongside the wspulse module. Breaking
changes to the wire format will increment the major version of the module.
