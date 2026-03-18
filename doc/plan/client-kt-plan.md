# wspulse Kotlin Client — Development Plan (`client-kt`)

> Status: **shipped v0.2.0** · Last updated: 2026-03-18
> Repo: `wspulse/client-kt` · Package: `com.github.wspulse:client-kt` (JitPack)

**Read before starting:**

- [Interface contract](../contracts/client-interface.md)
- [Behaviour contract](../contracts/client-behaviour.md)
- [Wire protocol](../../server/doc/protocol.md)
- [client-go] reference implementation: `client-go/client.go`, `client-go/options.go`

---

## Environments

| Target  | Min version | Transport          |
| ------- | ----------- | ------------------ |
| JVM     | Java 17     | Ktor CIO WebSocket |
| Android | API 26+     | Ktor CIO WebSocket |

Coroutine model: **kotlinx.coroutines** — all public API is `suspend fun` or non-blocking with `Deferred`.

---

## Project Structure

```
client-kt/
  src/
    main/kotlin/com/wspulse/client/
      WspulseClient.kt        # Client interface + WspulseClient implementation
      Frame.kt                # data class Frame(id, event, payload: Any?)
      ClientConfig.kt         # ClientConfig builder DSL
      Codec.kt                # Codec interface + JsonCodec (org.json)
      Errors.kt               # WspulseException sealed hierarchy
      Backoff.kt              # backoff(attempt, base, max): Duration
    test/kotlin/com/wspulse/client/
      BackoffTest.kt          # unit tests — backoff formula (4 tests)
      CodecTest.kt            # unit tests — JsonCodec (9 tests)
      ConfigValidationTest.kt # unit tests — config validation (15 tests)
      ClientIntegrationTest.kt # integration tests vs Go testserver (9 tests)
      WspulseClientResourceTest.kt # resource/lifecycle tests
  testserver/
    main.go                   # Go echo + reject testserver
    go.mod / go.sum
  build.gradle.kts
  settings.gradle.kts
  gradle/libs.versions.toml
  Makefile
  .github/
    workflows/
      ci.yml                  # lint + test on JDK 17 + 21 matrix
      cd.yml                  # GitHub Release on git tag
```

---

## Dependencies

Actual shipped dependencies (v0.2.0):

| Artifact                                        | Version    | Scope              |
| ----------------------------------------------- | ---------- | ------------------ |
| `org.jetbrains.kotlinx:kotlinx-coroutines-core` | `1.10.2`   | api                |
| `org.slf4j:slf4j-api`                           | `2.0.16`   | api                |
| `io.ktor:ktor-client-cio`                       | `3.4.1`    | implementation     |
| `io.ktor:ktor-client-websockets`                | `3.4.1`    | implementation     |
| `org.json:json`                                 | `20240303` | implementation     |
| `org.junit.jupiter:junit-jupiter`               | `5.11.4`   | testImplementation |
| `org.jetbrains.kotlinx:kotlinx-coroutines-test` | `1.10.2`   | testImplementation |

> Note: OkHttp and kotlinx-serialization were **not used**. Transport is Ktor CIO; serialisation is org.json (custom `JsonCodec`).

---

## Phase Breakdown

### P1 — Project Scaffold ✅

- [x] Create repo `wspulse/client-kt`
- [x] `settings.gradle.kts`: `rootProject.name = "client-kt"`
- [x] `build.gradle.kts`:
  - Kotlin JVM 2.3.0 (≠ plan's 2.0), `jvmToolchain(17)` (≠ plan's Java 11)
  - Plugins: `kotlin("jvm")`, `ktlint`, `jacoco`, `maven-publish`
  - No `kotlin("plugin.serialization")` — org.json used instead
  - JitPack publishing (≠ plan's Maven Central)
- [x] Gradle wrapper at Gradle 8.12
- [x] CI workflow: `./gradlew check` on JDK 17 + 21 matrix
- [x] CD workflow: GitHub Release on git tag (no Maven Central publish)
- [x] Makefile with `check`, `test`, `lint`, `fmt`, `test-cover`, `test-integration`, `clean`

### P2 — Connect / Send / Close / Done ✅

- [x] **`Frame.kt`**: `data class Frame(id, event, payload: Any?)` — `payload` is `Any?` (not `JsonElement?`); org.json used instead of kotlinx-serialization
- [x] **`Codec.kt`**: `Codec` interface + `JsonCodec` using org.json; `FrameType` enum
- [x] **`Errors.kt`**: `sealed class WspulseException`; added `SendBufferFullException` (not in plan); `ConnectionClosedException`, `RetriesExhaustedException(attempts)`, `ConnectionLostException(cause?)`
- [x] **`ClientConfig.kt`**: builder DSL with `autoReconnect`, `heartbeat`, `writeWait`, `maxMessageSize`, `dialHeaders`; all callbacks present
- [x] **`WspulseClient.kt`**: `Client` interface + `WspulseClient` implementation
  - `connect()` ✅, `send()` (non-blocking, throws `SendBufferFullException` on full) ✅, `close()` ✅, `done: Deferred<Unit>` ✅
  - Send buffer: `Channel<ByteArray>(256)` — throws `SendBufferFullException` instead of `DROP_OLDEST` (≠ plan; aligned with client-go contract)
  - Scope: `CoroutineScope(SupervisorJob() + Dispatchers.IO)` ✅

### P3 — Auto-Reconnect + Backoff ✅

- [x] **`Backoff.kt`**: `internal fun backoff(attempt, base, max): Duration` with equal jitter — matches client-go formula ✅
- [x] Reconnect loop: exponential backoff, `maxRetries` limit, `onReconnect`, `onDisconnect(RetriesExhaustedException)` ✅
- [x] `close()` during reconnect → `onDisconnect(null)` ✅

### P4 — Heartbeat + Write Deadline ✅

- [x] Heartbeat: Ktor CIO `pingInterval` / `pingWait` (not OkHttp `pingIntervalMillis`) ✅
- [x] `writeWait`: `withTimeout(writeWait)` around Ktor send ✅
- [x] `maxMessageSize`: close with code 1009 when exceeded ✅

### P5 — Test Suite ✅

14 integration tests + 4 backoff unit tests + 9 codec unit tests + 15 config validation tests + resource tests.

> **Canonical integration test coverage matrix:** [`client-kt/doc/integration-tests.md`](../../../client-kt/doc/integration-tests.md)

### P6 — Publish ✅ (via JitPack, not Maven Central)

- [x] README with quick-start, Android ViewModel example, API reference
- [x] CHANGELOG.md with v0.1.0 and v0.2.0 entries
- [x] CD workflow on git tag → GitHub Release (auto-generated notes)
- [x] JitPack badge on README; published at `com.github.wspulse:client-kt`
- [x] ~~Maven Central publish~~ — N/A (JitPack used instead)
- [x] ~~GPG signing~~ — N/A (JitPack does not require signing)

### P7 — Integration Test Gap ✅

Migrated to the [shared testserver](testserver-plan.md) with HTTP control API. All scenarios covered including pong timeout (scenario 7).

> **Canonical integration test coverage matrix:** [`client-kt/doc/integration-tests.md`](../../../client-kt/doc/integration-tests.md)

- [x] Migrate to shared testserver (see [testserver-plan.md](testserver-plan.md) Step 2)
- [x] Add kick test (server-initiated disconnect via control API)
- [x] Delete `client-kt/testserver/` after migration
- [x] Scenarios 3, 4, 5, 7 — all done

---

## Deviations from Original Plan

| Area                 | Plan                                | Actual                                      |
| -------------------- | ----------------------------------- | ------------------------------------------- |
| Transport            | OkHttp 4.x                          | Ktor CIO                                    |
| Serialisation        | kotlinx-serialization (JsonElement) | org.json (custom JsonCodec, `Any?` payload) |
| JVM minimum          | Java 11                             | Java 17                                     |
| Kotlin version       | 2.0                                 | 2.3.0                                       |
| Publishing           | Maven Central + GPG signing         | JitPack (no signing required)               |
| Send buffer overflow | DROP_OLDEST                         | Throw `SendBufferFullException`             |
| SLF4J logging        | Not planned                         | Added as `api` dep in v0.2.0                |
