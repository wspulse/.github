# wspulse Swift Client — Development Plan (`client-swift`)

> Status: planned · Last updated: 2026-03-13
> Repo: `wspulse/client-swift` · Package: `WspulseClient` (SPM)

**Read before starting:**

- [Interface contract](../contracts/client-interface.md)
- [Behaviour contract](../contracts/client-behaviour.md)
- [Wire protocol](../../server/doc/protocol.md)
- [client-go] reference implementation: `client-go/client.go`, `client-go/options.go`

---

## Environments

| Platform | Min version | Transport                 |
| -------- | ----------- | ------------------------- |
| iOS      | 16.0+       | `URLSessionWebSocketTask` |
| macOS    | 13.0+       | `URLSessionWebSocketTask` |
| watchOS  | 9.0+        | `URLSessionWebSocketTask` |
| tvOS     | 16.0+       | `URLSessionWebSocketTask` |

Concurrency model: **Swift Structured Concurrency** (`async/await`, `actor`). Swift 6 strict concurrency mode enabled.  
Dependencies: **zero** (stdlib-only).

---

## Project Structure

```
client-swift/
  Sources/
    WspulseClient/
      WspulseClient.swift       # public actor: init, connect(), send(), close(), done
      Frame.swift               # Codable Frame struct
      WspulseClientOptions.swift # WspulseClientOptions value type (Sendable)
      Errors.swift              # WspulseError enum
      Backoff.swift             # backoff(attempt:base:max:) → Duration
      AnyJSON.swift             # Codable type-erased JSON value for payload
      ConnectionActor.swift     # internal actor wrapping URLSessionWebSocketTask
  Tests/
    WspulseClientTests/
      BackoffTests.swift                  # unit tests — backoff formula
      FrameTests.swift                    # unit tests — Frame + AnyJSON
      CodecTests.swift                    # unit tests — JSONCodec
      ClientIntegrationTests.swift        # integration tests against wspulse/server
      ClientIntegrationTestsSupport.swift # test helpers + server lifecycle
  Package.swift
  .github/
    workflows/
      ci.yml                    # swift test on macos-14 + ubuntu-latest
```

---

## Phase Breakdown

### P1 — Project Scaffold

- [ ] Create repo `wspulse/client-swift`
- [ ] `Package.swift`:
  ```swift
  // swift-tools-version: 5.10
  platforms: [.iOS(.v16), .macOS(.v13), .watchOS(.v9), .tvOS(.v16)]
  targets: [
    .target(name: "WspulseClient"),
    .testTarget(name: "WspulseClientTests", dependencies: ["WspulseClient"]),
  ]
  swiftSettings: [.enableExperimentalFeature("StrictConcurrency")]
  ```
- [ ] CI workflow: `swift test` on `macos-14` and `ubuntu-latest` (Swift 5.10)
- [ ] `.swiftlint.yml` for style enforcement

### P2 — Connect / Send / Close / Done

- [ ] **`Frame.swift`** — `Codable`, `Sendable` struct:

  ```swift
  public struct Frame: Codable, Sendable {
      public var id: String?
      public var event: String?
      public var payload: AnyJSON?
      public init(id: String? = nil, event: String? = nil, payload: AnyJSON? = nil) { ... }
  }
  ```

- [ ] **`AnyJSON.swift`** — type-erased Codable value supporting `null`, `bool`, `number`, `string`, `array`, `object`:

  ```swift
  public enum AnyJSON: Codable, Sendable, Equatable {
      case null
      case bool(Bool)
      case number(Double)
      case string(String)
      case array([AnyJSON])
      case object([String: AnyJSON])
  }
  ```

  Implement `init(from:)` and `encode(to:)` by probing `KeyedDecodingContainer` types.

- [ ] **`Errors.swift`**:

  ```swift
  public enum WspulseError: Error, Sendable {
      case connectionClosed
      case retriesExhausted
      case connectionLost
  }
  ```

- [ ] **`WspulseClientOptions.swift`** — `Sendable` value type:

  ```swift
  public struct WspulseClientOptions: Sendable {
      public var onMessage: (@Sendable (Frame) -> Void)?
      public var onDisconnect: (@Sendable (Error?) -> Void)?
      public var onReconnect: (@Sendable (Int) -> Void)?
      public var onTransportDrop: (@Sendable (Error) -> Void)?
      public var autoReconnect: AutoReconnectOptions?
      public var heartbeat: HeartbeatOptions
      public var writeWait: Duration
      public var maxMessageSize: Int
      public var dialHeaders: [String: String]
      public init() { /* set all defaults */ }
  }
  public struct AutoReconnectOptions: Sendable {
      public var maxRetries: Int   // ≤ 0 = unlimited
      public var baseDelay: Duration
      public var maxDelay: Duration
  }
  public struct HeartbeatOptions: Sendable {
      public var pingPeriod: Duration = .seconds(20)
      public var pongWait: Duration = .seconds(60)
  }
  ```

- [ ] **`ConnectionActor.swift`** — internal `actor` wrapping `URLSessionWebSocketTask`:
  - Owns the `URLSessionWebSocketTask` reference
  - `func dial(url: URL, headers: [String: String]) async throws`
  - `func send(_ data: Data) async throws` — with timeout via `withThrowingTaskGroup`
  - `func close(code: URLSessionWebSocketTask.CloseCode)`
  - `func receive() async throws -> URLSessionWebSocketTask.Message`

- [ ] **`WspulseClient.swift`** — public `actor`:

  ```swift
  public actor WspulseClient {
      public let done: AsyncStream<Void>
      private let continuation: AsyncStream<Void>.Continuation

      public init(url: URL, options: WspulseClientOptions = .init()) { ... }
      public func connect() async throws { ... }
      public func send(_ frame: Frame) async throws { ... }
      public func close() async { ... }
  }
  ```

  - Internal send buffer: `[Data]` with `maxSize = 256`; oldest element dropped when full
  - `done`: `AsyncStream<Void>` backed by a `Continuation` stored on the actor; yield once then finish on permanent disconnect
  - `Task` for the read loop; `Task` for the write loop; both stored as `Task<Void, Never>` and cancelled in `close()`

- [ ] Read loop:

  ```swift
  while !Task.isCancelled {
      let message = try await connection.receive()
      let frame = try decode(message)
      options.onMessage?(frame)
  }
  ```

- [ ] `send(_ frame:)`:
  1. Check `_closed` flag → throw `WspulseError.connectionClosed`
  2. Encode to `Data` via `JSONEncoder`
  3. Append to buffer; if count > 256, remove first (`buffer.removeFirst()`)
  4. Signal write loop (via `AsyncStream` or `AsyncChannel`)

### P3 — Auto-Reconnect + Backoff

- [ ] **`Backoff.swift`**:

  ```swift
  func backoff(attempt: Int, base: Duration, max: Duration) -> Duration {
      let shift = min(attempt, 62)
      let raw = min(base * Int(pow(2.0, Double(shift))), max)
      let jitter = Double.random(in: 0.8...1.2)
      return raw * jitter
  }
  ```

  > `Duration` arithmetic: use `.components.seconds` + `nanoseconds` for precision.

- [ ] Reconnect loop (detached `Task` stored on the actor):
  1. Await `transportDropped: AsyncStream<Error>`
  2. If `_closed` → exit
  3. Call `options.onTransportDrop?(err)`
  4. `try await Task.sleep(for: backoff(attempt:base:max:))`
  5. Call `options.onReconnect?(attempt)`
  6. Try `connection.dial(...)`:
     - Success → reset `attempt`, restart read/write loops
     - Fail → `attempt += 1`; if `attempt >= maxRetries > 0` → call `options.onDisconnect?(WspulseError.retriesExhausted)` → finish `done`
  7. Go to step 1

- [ ] `close()` during reconnect: `reconnectTask.cancel()` → `Task.sleep` throws `CancellationError` → caught in loop → call `options.onDisconnect?(nil)` → finish `done`

### P4 — Heartbeat + Write Deadline

- [ ] **Pong**: `URLSessionWebSocketTask` responds to WebSocket pings automatically — no manual pong needed.
- [ ] **`pingPeriod` / `pongWait`**: `URLSession` does not expose ping/pong events at the `URLSessionWebSocketTask` level. Use `webSocketTask.sendPing(pongReceiveHandler:)` in a loop within the write loop:
  ```swift
  Task {
      while !Task.isCancelled {
          try await Task.sleep(for: options.heartbeat.pingPeriod)
          try await withThrowingTaskGroup(of: Void.self) { group in
              group.addTask { try await sendPing() }
              group.addTask {
                  try await Task.sleep(for: pongWait)
                  throw WspulseError.connectionLost  // pong timeout
              }
              let _ = try await group.next()
              group.cancelAll()
          }
      }
  }
  ```
- [ ] **`writeWait`**: wrap `connection.send()` in `withThrowingTaskGroup` with a cancelling timeout task (same pattern as ping).
- [ ] **`maxMessageSize`**: in the receive loop, check `data.count` / `string.utf8.count`; call `connection.close(code: .messageTooLong)` if exceeded.

### P5 — Test Suite

All 8 shared scenarios from the behaviour contract:

| #   | Scenario                                                | Test approach                                                                     |
| --- | ------------------------------------------------------- | --------------------------------------------------------------------------------- |
| 1   | Connect → send → echo → close clean                     | `XCTestExpectation`; start local server via `Process`; assert `onDisconnect(nil)` |
| 2   | Server drops → onTransportDrop + onDisconnect           | Control API `POST /kick`; verify callback order with two expectations             |
| 3   | Auto-reconnect: reconnects within maxRetries            | Control API `POST /kick`; assert `onReconnect(0)` then successful receive         |
| 4   | Max retries exhausted → `WspulseError.retriesExhausted` | Control API `POST /shutdown`; `maxRetries = 2`                                    |
| 5   | `close()` during reconnect → `onDisconnect(nil)`        | Control API `POST /kick`; call `close()` in `onTransportDrop`                     |
| 6   | `send()` on closed → `WspulseError.connectionClosed`    | `XCTAssertThrowsError` after `close()`                                            |
| 7   | No pong within pongWait → reconnect                     | Deferred — requires testserver `?ignore_pings=1` support                          |
| 8   | Concurrent sends: no race                               | 100 concurrent `async let` calls to `send()`; verify all arrive                   |

- Test infra: `setUp()` launches the [shared testserver](testserver-plan.md) via `Process` from `../testserver`; parses `READY:<ws_port>:<control_port>` from stderr; `tearDown()` terminates it
- macOS CI only for integration tests (Linux CI for unit tests)
- HTTP helper: `controlPost(_ path: String, params: [String: String])` calls control port endpoints

---

## Key Implementation Notes

1. **`actor` vs `@MainActor`**: use a plain `actor` (not `@MainActor`) so the library is not main-thread-bound. Callers that update UI bridge back to `MainActor` themselves.
2. **`AnyJSON` payload**: avoid third-party `AnyCodable` packages — implement a minimal `AnyJSON` enum in the library to keep zero external dependencies.
3. **`Duration` arithmetic in Swift**: `Duration` does not support `*` by `Int` directly — use `Duration.seconds(seconds) + Duration.nanoseconds(nanos)` after computing the raw value as `Double`.
4. **Swift 6 strict concurrency**: every closure stored in `WspulseClientOptions` must be `@Sendable`. Closures that capture `XCTestExpectation` in tests must use `nonisolated(unsafe)` or `@Sendable` wrappers.
5. **`URLSessionWebSocketTask` and Linux**: `URLSession` WebSocket is not available on Linux (no Foundation networking). The library is therefore Apple-platforms only; document this clearly. CI runs unit-only tests (backoff) on Linux, integration tests on macOS.

---

## Publish Checklist (P6)

- [ ] README: quick-start (SPM dependency block + async/await example)
- [ ] CHANGELOG.md with 0.1.0 entry
- [ ] Tag `0.1.0` on `main`; Swift Package Index (swiftpackageindex.com) auto-discovers from GitHub
- [ ] DocC documentation: `swift package generate-documentation`
- [ ] GitHub release with SPI badge
