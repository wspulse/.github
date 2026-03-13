# wspulse

A modular WebSocket library ecosystem for Go — minimal, production-ready, and easy to integrate with any HTTP router.

## Architecture

```mermaid
graph LR
    core["<b>core</b><br/>Frame · Codec · Router"]
    server["<b>server</b><br/>Server · Connection · Hub"]
    client-go["<b>client-go</b><br/>Client · Dial · Backoff"]
    server --> core
    client-go --> core
```

| Module                                            | Description                                                                                              |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| [core](https://github.com/wspulse/core)           | Shared types (`Frame`, `Codec`, sentinel errors) and Gin-style event router. Zero external dependencies. |
| [server](https://github.com/wspulse/server)       | WebSocket server: room routing, session resumption, heartbeat, backpressure.                             |
| [client-go](https://github.com/wspulse/client-go) | Go client: auto-reconnect, exponential backoff, lifecycle callbacks.                                     |

## Where to Start

| I want to…                   | Go to                                                                                   |
| ---------------------------- | --------------------------------------------------------------------------------------- |
| Build a WebSocket server     | [server README](https://github.com/wspulse/server#readme)                               |
| Connect from Go              | [client-go README](https://github.com/wspulse/client-go#readme)                         |
| Route frames by event name   | [core/router](https://github.com/wspulse/core#router)                                   |
| Understand the wire protocol | [server/doc/protocol.md](https://github.com/wspulse/server/blob/main/doc/protocol.md)   |
| Understand server internals  | [server/doc/internals.md](https://github.com/wspulse/server/blob/main/doc/internals.md) |
| Contribute                   | [CONTRIBUTING.md](../CONTRIBUTING.md)                                                   |

## Key Features

- **Room-based routing** — connections are partitioned into rooms; broadcast targets a single room
- **Session resumption** — configurable grace window; transparent WebSocket swap on reconnect
- **Event router** — Gin-style middleware chain for dispatching frames by event name (`core/router`)
- **Pluggable codecs** — JSON by default; swap in Protobuf, MessagePack, or any custom encoding
- **Auto-reconnect** — client-side exponential backoff with configurable retries
- **Any HTTP router** — standard `http.Handler`; works with net/http, Gin, Chi, Echo, etc.

## Roadmap

First-party client libraries are planned for TypeScript/JS, Kotlin, Swift, and Python — all implementing the same [wire protocol](https://github.com/wspulse/server/blob/main/doc/protocol.md) and [behaviour contract](doc/contracts/client-behaviour.md).

## Status

`server` is at **v0.2.1**; `core` and `client-go` are at **v0.2.0** — APIs are being stabilized. Breaking changes may occur before v1.

## License

All modules are released under the [MIT License](https://github.com/wspulse/core/blob/main/LICENSE).
