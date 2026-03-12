# wspulse

A modular WebSocket library ecosystem for Go — minimal, production-ready, and easy to integrate with any HTTP router.

## Modules

All modules share [core](https://github.com/wspulse/core) — shared types (`Frame`, `Codec`, sentinel errors) and a Gin-style event router. Zero external dependencies; pulled in automatically.

| Module                                            | Description                                                                  |
| ------------------------------------------------- | ---------------------------------------------------------------------------- |
| [server](https://github.com/wspulse/server)       | WebSocket server: room routing, session resumption, heartbeat, backpressure. |
| [client-go](https://github.com/wspulse/client-go) | Go client: auto-reconnect, exponential backoff, callbacks.                   |

## At a Glance

### Server

Basic broadcast with session resumption:

```go
var srv server.Server

srv = server.NewServer(
    func(r *http.Request) (roomID, connectionID string, err error) {
        return r.URL.Query().Get("room"), r.URL.Query().Get("uid"), nil
    },
    server.WithOnMessage(func(conn server.Connection, f server.Frame) {
        srv.Broadcast(conn.RoomID(), f)
    }),
    server.WithResumeWindow(30 * time.Second),
)
http.Handle("/ws", srv)
```

### Event Routing with `core/router`

Both server and client libraries can use `core/router` for Gin-style event dispatching with middleware:

```go
// ── server side ──
rtr := router.New()
rtr.Use(router.Recovery())

rtr.On("chat.message", func(c *router.Context) {
    srv.Broadcast(c.Connection.RoomID(), c.Frame)
})
rtr.On("chat.join", func(c *router.Context) {
    welcome := server.Frame{Event: "chat.welcome", Payload: []byte(`"hello"`)}
    c.Connection.Send(welcome)
})

srv = server.NewServer(connectFunc,
    server.WithOnMessage(func(conn server.Connection, f server.Frame) {
        rtr.Dispatch(conn, f) // server.Connection satisfies router.Connection
    }),
)
```

### Client — Go

```go
rtr := router.New()
rtr.Use(router.Recovery())

rtr.On("chat.message", func(c *router.Context) {
    fmt.Printf("[msg] %s\n", c.Frame.Payload)
})
rtr.On("chat.welcome", func(c *router.Context) {
    fmt.Println("joined!")
})

c, _ := client.Dial("ws://localhost:8080/ws?room=r1&uid=alice",
    client.WithOnMessage(func(f wspulse.Frame) {
        rtr.Dispatch(nil, f) // client has no router.Connection; pass nil
    }),
    client.WithAutoReconnect(5, time.Second, 30*time.Second),
)
defer c.Close()
```

## Key Features

- **Room-based routing** — connections are partitioned into rooms; broadcast targets a single room
- **Session resumption** — configurable grace window; transparent WebSocket swap on reconnect
- **Pluggable codecs** — JSON by default; swap in Protobuf, MessagePack, or any custom encoding
- **Event router** — Gin-style middleware chain for dispatching frames by event name (`core/router`)
- **Auto-reconnect** — client-side exponential backoff with configurable retries
- **Any HTTP router** — standard `http.Handler`; works with net/http, Gin, Chi, Echo, etc.

## Documentation

- [Wire Protocol](https://github.com/wspulse/server/blob/main/doc/protocol.md) — frame format, heartbeat, session resumption
- [Server Internals](https://github.com/wspulse/server/blob/main/doc/internals.md) — hub event loop, goroutine model, backpressure

## Status

All modules are at **v0.2.0** — API is being stabilized. Breaking changes may occur before v1.

## License

All modules are released under the [MIT License](https://github.com/wspulse/core/blob/main/LICENSE).
