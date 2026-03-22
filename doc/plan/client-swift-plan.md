# wspulse Swift Client — Development Plan (`client-swift`)

> Status: v0.1.0 released · Last updated: 2026-03-22
> Repo: `wspulse/client-swift` · Package: `WspulseClient` (SPM)

**Read before starting:**

- [Interface contract](../contracts/client/interface.md)
- [Behaviour contract](../contracts/client/behaviour.md)
- [Wire protocol](../protocol.md)
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
      WspulseClient.swift            # public actor: init, connect(), send(), close(), done
      WspulseClient+Internal.swift   # internal loops, reconnect, helpers
      WspulseClientOptions.swift     # WspulseClientOptions value type (Sendable)
      ConnectionActor.swift          # internal actor wrapping URLSessionWebSocketTask
      Frame.swift                    # Codable Frame struct
      AnyJSON.swift                  # Codable type-erased JSON value for payload
      Codec.swift                    # WspulseCodec protocol + JSONCodec
      Errors.swift                   # WspulseError enum + LocalizedError
      Backoff.swift                  # backoff(attempt:base:max:) → Duration
  Tests/
    WspulseClientTests/
      BackoffTests.swift                  # unit tests — backoff formula
      FrameTests.swift                    # unit tests — Frame + AnyJSON
      CodecTests.swift                    # unit tests — JSONCodec
      OptionsTests.swift                  # unit tests — options validation
      ClientUnitTests.swift               # unit tests — client state machine
      ErrorTests.swift                    # unit tests — error descriptions
      ClientIntegrationTests.swift        # integration tests against wspulse/server
      ClientIntegrationTestsSupport.swift # test helpers + server lifecycle
  Package.swift
  .github/
    workflows/
      ci.yml                    # lint + unit tests (macos-15) + integration tests
      cd.yml                    # tag-triggered release
```

---

## Phase Breakdown

### P1 — Project Scaffold [DONE]

- [x] Create repo `wspulse/client-swift`
- [x] `Package.swift` with platforms, StrictConcurrency, SwiftLint integration
- [x] CI workflow: lint + unit tests on macos-15, integration tests with Go testserver
- [x] CD workflow: tag-triggered GitHub Release
- [x] `.swiftlint.yml` for style enforcement
- [x] Makefile: build, test, test-integration, lint, fmt, check, clean

### P2 — Connect / Send / Close / Done [DONE]

- [x] `Frame.swift` — `Codable`, `Sendable` struct with `id`, `event`, `payload`
- [x] `AnyJSON.swift` — type-erased `Codable` JSON enum (`null`, `bool`, `number`, `string`, `array`, `object`)
- [x] `Errors.swift` — `WspulseError` enum with `LocalizedError` conformance; cases: `connectionClosed`, `retriesExhausted`, `connectionLost`, `sendBufferFull`, `encodingFailed`
- [x] `Codec.swift` — `WspulseCodec` protocol + `JSONCodec` default, `FrameType` enum (`.text` / `.binary`)
- [x] `WspulseClientOptions.swift` — `Sendable` value type with callbacks, reconnect, heartbeat, codec, `os.Logger`
- [x] `ConnectionActor.swift` — internal actor wrapping `URLSessionWebSocketTask` with `ConnectionDelegate` for lifecycle
- [x] `WspulseClient.swift` — public actor: `connect()`, `send()`, `close()`, `done`
- [x] `WspulseClient+Internal.swift` — read/write/ping loops, reconnect, buffer drain (split for lint compliance)
- [x] Send buffer: `[Data]` with max 256; throws `sendBufferFull` when full
- [x] Write loop: `AsyncStream<Void>` signal channel, replaced on each connection cycle

### P3 — Auto-Reconnect + Backoff [DONE]

- [x] `backoff()` with equal jitter, negative attempt clamping, Int64 overflow protection
- [x] Reconnect loop: `handleTransportDrop` → `startReconnectLoop` → backoff → dial + ping verify
- [x] `close()` during reconnect: cancels reconnect task → `onDisconnect(nil)`
- [x] `reconnectExhausted()`: `onDisconnect(.retriesExhausted)` with full cleanup
- [x] Transport close before reconnect loop starts (early resource release)

### P4 — Heartbeat + Write Deadline [DONE]

- [x] Ping loop: `sendPing()` with pong timeout via `withThrowingTaskGroup`
- [x] CancellationError handling in task groups (not misclassified as transport drop)
- [x] `writeWait` enforced in `drainBuffer()` via task group timeout
- [x] `maxMessageSize` enforcement on `URLSessionWebSocketTask`
- [x] `encodingFailed` error for non-UTF8 data in text mode (not crash)

### P5 — Test Suite [DONE]

All 9 contract scenarios + 7 additional tests = **16 integration tests**.
See [integration-tests.md](../../client-swift/doc/integration-tests.md) for full matrix.

**99 unit tests** covering:
- Backoff formula, edge cases, negative attempts, overflow
- Frame Codable round-trip, AnyJSON encoding/decoding
- JSONCodec encode/decode
- Options validation (preconditions, defaults)
- Client state machine (buffer, close, send-after-close)
- Error descriptions and LocalizedError conformance

Test infrastructure:
- Shared Go testserver spawned via `Process` in class `setUp()`, killed in `tearDown()`
- macOS CI only (integration tests require Go + testserver siblings)
- `waitUntil` helper with timeout + throw on failure

---

## Key Implementation Notes

1. **`actor` vs `@MainActor`**: use a plain `actor` (not `@MainActor`) so the library is not main-thread-bound. Callers that update UI bridge back to `MainActor` themselves.
2. **`AnyJSON` payload**: avoid third-party `AnyCodable` packages — implement a minimal `AnyJSON` enum in the library to keep zero external dependencies.
3. **`Duration` arithmetic in Swift**: `Duration` does not support `*` by `Int` directly — use `Duration.seconds(seconds) + Duration.nanoseconds(nanos)` after computing the raw value as `Double`.
4. **Swift 6 strict concurrency**: every closure stored in `WspulseClientOptions` must be `@Sendable`. Closures that capture `XCTestExpectation` in tests must use `nonisolated(unsafe)` or `@Sendable` wrappers.
5. **`URLSessionWebSocketTask` and Linux**: `URLSession` WebSocket is not available on Linux (no Foundation networking). The library is therefore Apple-platforms only; CI runs on macOS only.
6. **Logging**: uses `os.Logger` (enabled by default). Users can replace via `WspulseClientOptions(logger:)`.
7. **StrictConcurrency in task groups**: `addTask` closures are nonisolated — capture actor-isolated properties into locals before creating child tasks.
8. **ConnectionDelegate lock safety**: handlers captured inside lock, invoked outside to prevent re-entrancy/deadlock.
9. **File split**: `WspulseClient.swift` (public API) + `WspulseClient+Internal.swift` (loops, reconnect) to stay under SwiftLint `type_body_length` limit.

---

## Publish Checklist (P6) [DONE]

- [x] README: quick-start (SPM dependency block + async/await example)
- [x] CHANGELOG.md with 0.1.0 entry
- [x] Tag `v0.1.0` on `main`; CD creates GitHub Release automatically
- [ ] DocC documentation: `swift package generate-documentation` (deferred)
- [ ] Swift Package Index submission (deferred)
