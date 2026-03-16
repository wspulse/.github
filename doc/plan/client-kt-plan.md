# wspulse Kotlin Client — Development Plan (`client-kt`)

> Status: planned · Last updated: 2026-03-13
> Repo: `wspulse/client-kt` · Package: `com.wspulse:client-kt`

**Read before starting:**

- [Interface contract](../contracts/client-interface.md)
- [Behaviour contract](../contracts/client-behaviour.md)
- [Wire protocol](../../server/doc/protocol.md)
- [client-go] reference implementation: `client-go/client.go`, `client-go/options.go`

---

## Environments

| Target  | Min version | Transport            |
| ------- | ----------- | -------------------- |
| JVM     | Java 11     | OkHttp 4.x WebSocket |
| Android | API 26+     | OkHttp 4.x WebSocket |

Coroutine model: **kotlinx.coroutines** — all public API is `suspend fun` or returns `Deferred`/`Flow`.

---

## Project Structure

```
client-kt/
  src/
    main/kotlin/com/wspulse/client/
      WspulseClient.kt        # main class + companion object factory
      Frame.kt                # @Serializable data class Frame
      ClientConfig.kt         # ClientConfig builder DSL
      Errors.kt               # WspulseException hierarchy
      Backoff.kt              # backoff(attempt, base, max): Duration
    test/kotlin/com/wspulse/client/
      BackoffTest.kt          # unit tests — backoff formula
      ClientTest.kt           # integration tests against wspulse/server
  build.gradle.kts
  settings.gradle.kts
  gradle/
    wrapper/
      gradle-wrapper.properties
  .github/
    workflows/
      ci.yml                  # build + lint + test on JDK 21
```

---

## Dependencies

| Artifact                                           | Version  | Scope              |
| -------------------------------------------------- | -------- | ------------------ |
| `com.squareup.okhttp3:okhttp`                      | `4.12.x` | api                |
| `org.jetbrains.kotlinx:kotlinx-coroutines-core`    | `1.9.x`  | api                |
| `org.jetbrains.kotlinx:kotlinx-serialization-json` | `1.7.x`  | api                |
| `org.junit.jupiter:junit-jupiter`                  | `5.11.x` | testImplementation |
| `org.jetbrains.kotlinx:kotlinx-coroutines-test`    | `1.9.x`  | testImplementation |

---

## Phase Breakdown

### P1 — Project Scaffold

- [ ] Create repo `wspulse/client-kt`
- [ ] `settings.gradle.kts`: `rootProject.name = "client-kt"`, `includeBuild(".")` for local dev
- [ ] `build.gradle.kts`:
  - Kotlin JVM 2.0, `jvmTarget = JavaVersion.VERSION_11`
  - Plugins: `kotlin("jvm")`, `kotlin("plugin.serialization")`, `maven-publish`, `signing`
  - All dependencies listed above
  - Maven Central publish config (`groupId = "com.wspulse"`, `artifactId = "client-kt"`)
- [ ] Gradle wrapper at Gradle 8.10
- [ ] CI workflow: `./gradlew check` on JDK 21

### P2 — Connect / Send / Close / Done

- [ ] **`Frame.kt`**:

  ```kotlin
  @Serializable
  data class Frame(
      val id: String? = null,
      val event: String? = null,
      val payload: JsonElement? = null,  // JsonElement covers any JSON value
  )
  ```

- [ ] **`Errors.kt`** — sealed exception hierarchy:

  ```kotlin
  sealed class WspulseException(msg: String) : Exception(msg)
  class ConnectionClosedException : WspulseException("connection is closed")
  class RetriesExhaustedException : WspulseException("max reconnect retries exhausted")
  class ConnectionLostException : WspulseException("connection lost")
  ```

- [ ] **`ClientConfig.kt`** — builder DSL:

  ```kotlin
  class ClientConfig {
      var onMessage: (Frame) -> Unit = {}
      var onDisconnect: (WspulseException?) -> Unit = {}
      var onReconnect: (attempt: Int) -> Unit = {}
      var onTransportDrop: (Exception) -> Unit = {}
      var autoReconnect: AutoReconnectConfig? = null  // null = disabled
      var heartbeat: HeartbeatConfig = HeartbeatConfig()
      var writeWait: Duration = 10.seconds
      var maxMessageSize: Long = 1 * 1024 * 1024
      var dialHeaders: Map<String, String> = emptyMap()
  }
  data class AutoReconnectConfig(val maxRetries: Int, val baseDelay: Duration, val maxDelay: Duration)
  data class HeartbeatConfig(val pingPeriod: Duration = 20.seconds, val pongWait: Duration = 60.seconds)
  ```

- [ ] **`WspulseClient.kt`** — class with companion factory:

  ```kotlin
  class WspulseClient private constructor(
      private val url: String,
      private val config: ClientConfig,
  ) {
      val done: CompletableDeferred<Unit> = CompletableDeferred()

      suspend fun send(frame: Frame): Unit { ... }
      suspend fun close(): Unit { ... }

      companion object {
          suspend fun connect(url: String, config: ClientConfig.() -> Unit = {}): WspulseClient { ... }
      }
  }
  ```

- [ ] Internal send buffer: `Channel<ByteArray>(capacity = 256, onBufferOverflow = DROP_OLDEST)`
  - Kotlin `Channel` with `DROP_OLDEST` handles head-drop natively.

- [ ] On initial connect failure (no autoReconnect): throw `ConnectionLostException`.

- [ ] `send()`:
  1. Check `done.isCompleted` → throw `ConnectionClosedException`
  2. JSON-encode `Frame` via `Json.encodeToString()`
  3. `channel.trySend(bytes)` (non-blocking due to DROP_OLDEST)

- [ ] `close()`:
  1. Set closed flag
  2. Cancel internal coroutine scope
  3. Close OkHttp WebSocket with code 1000
  4. Await `done` (it will be completed by the teardown path)

- [ ] Write coroutine: consume `channel`, send via `webSocket.send(ByteString)` inside `withTimeout(writeWait)`

### P3 — Auto-Reconnect + Backoff

- [ ] **`Backoff.kt`**:

  ```kotlin
  fun backoff(attempt: Int, base: Duration, max: Duration): Duration {
      val shift = minOf(attempt, 62)
      val raw = minOf(base * (1L shl shift), max)
      val jitter = 0.8 + Random.nextDouble() * 0.4
      return raw * jitter
  }
  ```

- [ ] Internal `CoroutineScope(SupervisorJob() + Dispatchers.IO)` owns all goroutines
- [ ] Reconnect loop (coroutine):
  1. Await `droppedChannel` signal from OkHttp `onFailure`/`onClosed`
  2. Check closed flag → exit
  3. Fire `config.onTransportDrop(err)` (only on unexpected drop)
  4. `delay(backoff(attempt, base, max))`
  5. Fire `config.onReconnect(attempt)`
  6. Try `dial()`:
     - Success → reset `attempt`, resume send coroutine with new `webSocket`
     - Fail → `attempt++`; if `attempt >= maxRetries > 0` → fire `onDisconnect(RetriesExhaustedException())` → `done.complete(Unit)` → exit
  7. Go to step 1

- [ ] `close()` during reconnect: cancel the coroutine scope; `CancellationException` is caught in the loop → fire `onDisconnect(null)` → `done.complete(Unit)`

### P4 — Heartbeat + Write Deadline

- [ ] **OkHttp Pong**: `WebSocketListener.onMessage` does not handle pings; OkHttp replies automatically — no manual handling.
- [ ] **`pongWait`**: when OkHttp fires `onMessage(webSocket, text)`, also schedule a `withTimeout(pongWait)` coroutine. OkHttp does not expose ping/pong events directly — use `webSocket.send(ByteString.EMPTY)` as a ping and check `onMessage` / `onFailure` timing. _Alternative_: rely on OkHttp's built-in `pingIntervalMillis` config.
  - **Recommended**: pass `OkHttpClient.Builder().pingInterval(pingPeriod)` to OkHttp; this causes OkHttp to send pings automatically and close the connection if no pong arrives within `pongWait`. Document this in the README.
- [ ] **`writeWait`**: wrap `webSocket.send()` in `withTimeout(writeWait)`.
- [ ] **`maxMessageSize`**: check `text.length` or `bytes.byteLength` in `onMessage`; if exceeded, call `webSocket.cancel()`.

### P5 — Test Suite

All 8 shared scenarios from the behaviour contract:

| #   | Scenario                                                          | Test approach                                                                             |
| --- | ----------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| 1   | Connect → send → echo → close clean                               | `runTest { }` block; start local server via `ProcessBuilder`; assert `onDisconnect(null)` |
| 2   | Server drops → onTransportDrop + onDisconnect                     | Server force-closes; verify callback order with `CompletableDeferred` collect             |
| 3   | Auto-reconnect: reconnects within maxRetries                      | Server drops once; assert `onReconnect(0)` then successful message                        |
| 4   | Max retries exhausted → `onDisconnect(RetriesExhaustedException)` | Always-fail server; `maxRetries=2`                                                        |
| 5   | `close()` during reconnect → `onDisconnect(null)`                 | Call `close()` in `onTransportDrop` callback                                              |
| 6   | `send()` on closed → `ConnectionClosedException`                  | Call `send()` after `close()`; catch exception                                            |
| 7   | No pong within pongWait → reconnect                               | Use OkHttp `pingInterval` + short `pongWait`; server ignores pings                        |
| 8   | Concurrent sends: no race                                         | Launch 100 coroutines all calling `send()`; verify delivery order                         |

- Test infra: start `wspulse/server` with `ProcessBuilder` in `@BeforeAll`; shut down with `process.destroy()` in `@AfterAll`
- Use `kotlinx-coroutines-test` `runTest` for all async tests

---

## Key Implementation Notes

1. **OkHttp WebSocket is callback-based**, not coroutine-native. Bridge via `Channel` and `CompletableDeferred`:
   - `onOpen` → complete a `connectedDeferred`
   - `onMessage` → decode + call `onMessage` callback
   - `onFailure` / `onClosed` → send to `droppedChannel`
2. **Serialisation**: use `kotlinx.serialization` with `Json { ignoreUnknownKeys = true }`. `payload` field typed as `JsonElement?` preserves arbitrary JSON without pre-defining structure.
3. **`Channel(DROP_OLDEST)`**: Kotlin `Channel` with `BUFFERED` capacity and `DROP_OLDEST` overflow strategy is the idiomatic head-drop implementation — mirrors the Go `send` channel with manual head-drop.
4. **Thread safety**: OkHttp callbacks run on OkHttp's thread pool; all state mutations go through the coroutine scope (post to the scope's dispatcher). Never touch shared state directly from OkHttp callbacks.
5. **Jetpack compatibility (Android)**: all public API is dispatcher-agnostic — callers inject context via `withContext(Dispatchers.Main)` in their own code. Do not `Dispatchers.Main` internally.

---

## Publish Checklist (P6)

- [ ] README: quick-start + builder DSL example
- [ ] CHANGELOG.md with 0.1.0 entry
- [ ] `./gradlew publishToSonatype closeAndReleaseStagingRepository` via CI on git tag
- [ ] GitHub release with Maven Central badge
- [ ] Sign artifacts with GPG key stored as GitHub Actions secret
