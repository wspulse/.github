# wspulse Client Library Plan — Multi-Language

> Status: draft · Last updated: 2026-03-12

Roadmap for wspulse client libraries beyond Go. The API and behaviour requirements are fully specified in the contract docs — this file tracks **what** to build and **where things stand**.

**Contracts (read before implementing):**

- [Interface contract](../contracts/client-interface.md) — API surface, options, error types, language mapping
- [Behaviour contract](../contracts/client-behaviour.md) — lifecycle, callback semantics, reconnect algo, test scenarios
- [Wire protocol](https://github.com/wspulse/server/blob/main/doc/protocol.md) — frame format, heartbeat, session resumption

---

## Guiding Principles

1. **Protocol parity** — implement the JSON wire protocol exactly as specified.
2. **Behaviour parity** — lifecycle, callbacks, and reconnect semantics match the behaviour contract.
3. **Idiomatic API** — naming and concurrency patterns follow the target language's conventions.
4. **Zero breaking changes to the wire format** — never change frame encoding without a protocol version bump.
5. **Minimal dependencies** — stdlib or one well-maintained WebSocket lib per language.

---

## Progress

Phases per library:

| Phase | Description                               |
| ----- | ----------------------------------------- |
| P1    | Repo created, go.mod / package manifest   |
| P2    | Connect / Send / Close / Done             |
| P3    | Auto-reconnect + backoff                  |
| P4    | Heartbeat + write deadline                |
| P5    | All shared test scenarios pass            |
| P6    | Published (pkg registry + GitHub release) |

| Library             | P1  | P2  | P3  | P4  | P5  | P6  |
| ------------------- | --- | --- | --- | --- | --- | --- |
| `client-go` (Go)    | ✅  | ✅  | ✅  | ✅  | ✅  | ✅  |
| `client-ts` (TS/JS) | ⬜  | ⬜  | ⬜  | ⬜  | ⬜  | ⬜  |
| `client-kt`     | ⬜  | ⬜  | ⬜  | ⬜  | ⬜  | ⬜  |
| `client-swift`      | ⬜  | ⬜  | ⬜  | ⬜  | ⬜  | ⬜  |
| `client-py`     | ⬜  | ⬜  | ⬜  | ⬜  | ⬜  | ⬜  |

---

## Prioritisation

1. **TypeScript** — covers browser + Node.js; most common full-stack use case.
2. **Kotlin** — Android + JVM; coroutine model maps cleanly to the Go goroutine model.
3. **Swift** — iOS/macOS; zero external deps via `URLSessionWebSocketTask`.
4. **Python** — asyncio model maps cleanly; broad adoption.

---

## Planned Libraries

### TypeScript / JavaScript (`wspulse/client-ts`)

- **Environments**: Browser (native WebSocket API) + Node.js (`ws` package)
- **Module system**: ESM-first, CJS compat shim
- **Package name**: `@wspulse/client`
- **Key idioms**:
  - Callbacks match Go client (no observable difference to consumers)
  - `send(frame: Frame): void` — throws `ConnectionClosedError` if closed
  - `close(): void`
  - `done`: `Promise<void>` (resolves on permanent disconnect — replaces Go channel)
  - Auto-reconnect backoff: same formula as Go
  - Typed `Frame`: `{ id?: string; event?: string; payload?: unknown }`
- **Dependencies**: `ws` (Node.js only — peer dep); zero browser deps
- **Repo**: `wspulse/client-ts`
- **Status**: planned

### Python (`wspulse/client-py`)

- **Environments**: Python 3.11+, async-first (`asyncio`)
- **Package name**: `wspulse-client` (PyPI)
- **Key idioms**:
  - `async with wspulse.connect(url, **opts) as client:` context manager
  - `await client.send(frame)` — raises `ConnectionClosedError` if closed
  - `await client.close()`
  - `done: asyncio.Event`
  - Callbacks are coroutines or sync callables (wrapped internally with `asyncio.create_task`)
  - Auto-reconnect: same backoff formula, `asyncio.sleep` between attempts
- **Dependencies**: `websockets` (>=12)
- **Repo**: `wspulse/client-py`
- **Status**: planned

### Kotlin (`wspulse/client-kt`)

- **Environments**: JVM + Android
- **Package name**: `com.wspulse:client-kt` (Maven Central)
- **Key idioms**:
  - `WspulseClient.connect(url, config): WspulseClient`
  - Coroutine-based: `suspend fun send(frame: Frame)`; `suspend fun close()`
  - `val done: CompletableDeferred<Unit>`
  - Callbacks: `config.onMessage`, `config.onDisconnect`, etc. (lambda in builder DSL)
  - Auto-reconnect with same backoff formula
- **Dependencies**: `okhttp` (WebSocket), `kotlinx-coroutines-core`
- **Repo**: `wspulse/client-kt`
- **Status**: planned

### Swift (`wspulse/client-swift`)

- **Environments**: iOS 16+ / macOS 13+ / watchOS 9+ / tvOS 16+
- **Package name**: `WspulseClient` (Swift Package Manager)
- **Key idioms**:
  - `WspulseClient(url:options:)` — plain initialiser; call `connect()` to start
  - `async/await`: `func send(_ frame: Frame) async throws`; `func close() async`
  - `done: AsyncStream<Void>` or `withTaskCancellationHandler` integration
  - Callbacks as closures in `WspulseClientOptions` value type (builder pattern via `init` params)
  - `onMessage: ((Frame) -> Void)?`; `onDisconnect: ((Error?) -> Void)?`; etc.
  - Auto-reconnect with same backoff formula; retry loop runs inside a detached `Task`
  - `Sendable`-conformant types throughout (Swift 6 strict concurrency)
- **Dependencies**: `URLSessionWebSocketTask` (stdlib, zero external deps)
- **Repo**: `wspulse/client-swift`
- **Status**: planned

---

## Shared Test Scenarios (All Languages)

Every client lib must pass these behavioural tests (tested against `wspulse/server`):

| #   | Scenario                                                                    |
| --- | --------------------------------------------------------------------------- |
| 1   | Connect, send frame, receive echo, close cleanly → `OnDisconnect(nil)`      |
| 2   | Server drops connection → `OnTransportDrop` fires, then `OnDisconnect`      |
| 3   | Auto-reconnect: server drops, client reconnects within maxRetries           |
| 4   | Auto-reconnect: max retries exhausted → `OnDisconnect(ErrRetriesExhausted)` |
| 5   | `close()` during reconnect loop → loop stops, `OnDisconnect(nil)`           |
| 6   | Send on closed client → returns / raises `ConnectionClosedError`            |
| 7   | Heartbeat: no Pong within pongWait → server closes → client reconnects      |
| 8   | Concurrent sends: no data race / message interleaving                       |

---

## Repo Naming Convention

| Language   | Repo                    | Module / Package               |
| ---------- | ----------------------- | ------------------------------ |
| Go         | `wspulse/client-go`     | `github.com/wspulse/client-go` |
| TypeScript | `wspulse/client-ts`     | `@wspulse/client`              |
| Kotlin     | `wspulse/client-kt` | `com.wspulse:client-kt`    |
| Swift      | `wspulse/client-swift`  | `WspulseClient` (SPM)          |
| Python     | `wspulse/client-py` | `wspulse-client` (PyPI)        |

---

## Prioritisation

1. **TypeScript** — highest demand; covers both browser and Node.js; completes the most common full-stack use case.
2. **Kotlin** — Android + JVM; builder DSL maps cleanly to the Go options pattern.
3. **Swift** — iOS/macOS native; zero external deps via `URLSessionWebSocketTask`.
4. **Python** — broad adoption; asyncio model maps cleanly to the Go goroutine model.

Each language will be developed independently on its own repo. No shared scaffolding is required — common only the wire protocol.
