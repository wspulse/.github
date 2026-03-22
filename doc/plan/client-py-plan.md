# wspulse Python Client — Development Plan (`client-py`)

> Status: planned · Last updated: 2026-03-13
> Repo: `wspulse/client-py` · Package: `wspulse-client` (PyPI)

**Read before starting:**

- [Interface contract](../contracts/client/interface.md)
- [Behaviour contract](../contracts/client/behaviour.md)
- [Wire protocol](../protocol.md)
- [client-go] reference implementation: `client-go/client.go`, `client-go/options.go`

---

## Environments

| Target      | Min version |
| ----------- | ----------- |
| CPython     | 3.11+       |
| Async model | `asyncio`   |

Concurrency model: **asyncio-first**. All public methods are `async`. Callbacks may be sync or coroutine functions — both are supported via `asyncio.create_task`.  
Dependencies: `websockets >= 12` (no other runtime dependency).

---

## Project Structure

```
client-py/
  wspulse/
    __init__.py             # public exports: connect, Frame, errors
    _client.py              # WspulseClient implementation + connect()
    _frame.py               # Frame dataclass
    _errors.py              # ConnectionClosedError, RetriesExhaustedError, ConnectionLostError
    _backoff.py             # backoff(attempt, base_delay, max_delay) -> float
    _options.py             # ClientOptions dataclass + defaults
  tests/
    conftest.py             # pytest fixtures: start/stop wspulse/server subprocess
    test_backoff.py         # unit tests — backoff formula
    test_client.py          # integration tests against wspulse/server
  pyproject.toml            # [project] + [tool.ruff] + [tool.mypy] + [tool.pytest.ini_options]
  README.md
  CHANGELOG.md
  .github/
    workflows/
      ci.yml                # lint + type-check + test on Python 3.11, 3.12, 3.13
```

---

## Dependencies

| Package          | Version  | Scope   |
| ---------------- | -------- | ------- |
| `websockets`     | `>=12`   | runtime |
| `pytest`         | `>=8`    | dev     |
| `pytest-asyncio` | `>=0.24` | dev     |
| `ruff`           | `>=0.4`  | dev     |
| `mypy`           | `>=1.10` | dev     |

---

## Phase Breakdown

### P1 — Project Scaffold

- [ ] Create repo `wspulse/client-py`
- [ ] `pyproject.toml`:

  ```toml
  [project]
  name = "wspulse-client"
  version = "0.1.0"
  requires-python = ">=3.11"
  dependencies = ["websockets>=12"]

  [tool.ruff]
  select = ["E", "F", "I", "UP"]
  target-version = "py311"

  [tool.mypy]
  strict = true
  python_version = "3.11"

  [tool.pytest.ini_options]
  asyncio_mode = "auto"
  ```

- [ ] Package layout: `wspulse/` directory with `py.typed` marker file (PEP 561)
- [ ] `__init__.py` exports: `connect`, `Frame`, `ConnectionClosedError`, `RetriesExhaustedError`, `ConnectionLostError`
- [ ] CI workflow: `ruff check` → `mypy` → `pytest` on Python 3.11, 3.12, 3.13

### P2 — Connect / Send / Close / Done

- [ ] **`_frame.py`**:

  ```python
  from dataclasses import dataclass, field
  from typing import Any

  @dataclass
  class Frame:
      id: str | None = None
      event: str | None = None
      payload: Any = None

  def _encode(frame: Frame) -> str:
      data = {k: v for k, v in {"id": frame.id, "event": frame.event, "payload": frame.payload}.items() if v is not None}
      return json.dumps(data)

  def _decode(raw: str) -> Frame:
      data = json.loads(raw)
      return Frame(id=data.get("id"), event=data.get("event"), payload=data.get("payload"))
  ```

- [ ] **`_errors.py`**:

  ```python
  class WspulseError(Exception): ...
  class ConnectionClosedError(WspulseError): ...
  class RetriesExhaustedError(WspulseError): ...
  class ConnectionLostError(WspulseError): ...
  ```

- [ ] **`_options.py`** — `ClientOptions` dataclass:

  ```python
  @dataclass
  class AutoReconnectOptions:
      max_retries: int = 10   # ≤ 0 = unlimited
      base_delay: float = 1.0   # seconds
      max_delay: float = 30.0   # seconds

  @dataclass
  class HeartbeatOptions:
      ping_period: float = 20.0  # seconds
      pong_wait: float = 60.0    # seconds

  @dataclass
  class ClientOptions:
      on_message: Callable[[Frame], Awaitable[None] | None] = field(default=lambda f: None)
      on_disconnect: Callable[[Exception | None], Awaitable[None] | None] = field(default=lambda e: None)
      on_reconnect: Callable[[int], Awaitable[None] | None] = field(default=lambda a: None)
      on_transport_drop: Callable[[Exception], Awaitable[None] | None] = field(default=lambda e: None)
      auto_reconnect: AutoReconnectOptions | None = None   # None = disabled
      heartbeat: HeartbeatOptions = field(default_factory=HeartbeatOptions)
      write_wait: float = 10.0      # seconds
      max_message_size: int = 1 << 20  # 1 MiB
      dial_headers: dict[str, str] = field(default_factory=dict)
  ```

- [ ] **`_client.py`** — `WspulseClient`:

  ```python
  class WspulseClient:
      def __init__(self, url: str, options: ClientOptions) -> None: ...
      async def send(self, frame: Frame) -> None: ...
      async def close(self) -> None: ...

      @property
      def done(self) -> asyncio.Event: ...

      async def __aenter__(self) -> "WspulseClient": ...
      async def __aexit__(self, *args: object) -> None: await self.close()
  ```

- [ ] `done`: `asyncio.Event` — `.set()` on permanent disconnect
- [ ] Internal send buffer: `asyncio.Queue(maxsize=256)` with head-drop:
  ```python
  async def _enqueue(self, data: str) -> None:
      while self._queue.full():
          self._queue.get_nowait()  # drop oldest (head-drop)
      self._queue.put_nowait(data)
  ```
- [ ] Write loop (Task): dequeue → `await asyncio.wait_for(ws.send(data), timeout=write_wait)`
- [ ] Read loop (Task): `async for message in ws: decode → _fire_callback(on_message, frame)`
- [ ] `_fire_callback(fn, *args)`: call `fn(*args)`; if result is a coroutine, `asyncio.create_task(result)`

- [ ] `connect(url, **opts)` factory function:
  ```python
  async def connect(url: str, **opts: Any) -> WspulseClient:
      options = ClientOptions(**opts) if opts else ClientOptions()
      client = WspulseClient(url, options)
      await client._dial_once()
      return client
  ```

  - Raises `ConnectionLostError` on initial dial failure (if `auto_reconnect=None`)
  - Also usable as `async with wspulse.connect(url) as c:`

### P3 — Auto-Reconnect + Backoff

- [ ] **`_backoff.py`**:

  ```python
  import random

  def backoff(attempt: int, base_delay: float, max_delay: float) -> float:
      shift = min(attempt, 62)
      raw = min(base_delay * (2 ** shift), max_delay)
      jitter = random.uniform(0.8, 1.2)
      return raw * jitter
  ```

- [ ] Reconnect loop (`asyncio.Task` stored on the client):
  1. Await `_dropped_event: asyncio.Event`
  2. Check `_closed` flag → exit loop
  3. `_dropped_event.clear()`
  4. `_fire_callback(on_transport_drop, err)` (only on unexpected drop)
  5. `await asyncio.sleep(backoff(attempt, base, max))`
  6. `_fire_callback(on_reconnect, attempt)`
  7. Try `_dial_once()`:
     - Success → reset `attempt`, restart read/write tasks
     - Fail → `attempt += 1`; if `attempt >= max_retries > 0` → `_fire_callback(on_disconnect, RetriesExhaustedError())` → `done.set()` → exit
  8. Go to step 1

- [ ] `close()` during reconnect: set `_closed = True`, cancel the reconnect task; catch `asyncio.CancelledError` in the loop → `_fire_callback(on_disconnect, None)` → `done.set()`

### P4 — Heartbeat + Write Deadline

- [ ] **Pong**: the `websockets` library handles WebSocket pong replies automatically — no manual handling needed.
- [ ] **`pong_wait`**: `websockets.connect()` accepts `ping_interval` and `ping_timeout` parameters that implement exactly this behaviour:
  ```python
  ws = await websockets.connect(
      url,
      ping_interval=options.heartbeat.ping_period,
      ping_timeout=options.heartbeat.pong_wait,
      max_size=options.max_message_size,
      additional_headers=options.dial_headers,
  )
  ```
  Map `ClientOptions` heartbeat fields to `websockets` parameters directly.
- [ ] **`write_wait`**: already handled via `asyncio.wait_for(ws.send(data), timeout=write_wait)` in the write loop.
- [ ] **`max_message_size`**: passed as `max_size` to `websockets.connect()` — the library closes the connection automatically if exceeded.
- [ ] **`dial_headers`**: passed as `additional_headers` to `websockets.connect()`.

### P5 — Test Suite

All 8 shared scenarios from the behaviour contract:

| #   | Scenario                                           | Test approach                                                                               |
| --- | -------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| 1   | Connect → send → echo → close clean                | `pytest-asyncio`; server via `asyncio.create_subprocess_exec`; assert `on_disconnect(None)` |
| 2   | Server drops → on_transport_drop + on_disconnect   | Force-close connection; verify callback order with `asyncio.Event`s                         |
| 3   | Auto-reconnect: reconnects within max_retries      | Server drops once; assert `on_reconnect(0)` then successful receive                         |
| 4   | Max retries exhausted → `RetriesExhaustedError`    | Always-fail server; `max_retries=2`                                                         |
| 5   | `close()` during reconnect → `on_disconnect(None)` | Call `close()` in `on_transport_drop`; assert `done.is_set()`                               |
| 6   | `send()` on closed → `ConnectionClosedError`       | Call after `close()`; `pytest.raises()`                                                     |
| 7   | No pong within pong_wait → reconnect               | Short `pong_wait`; server ignores pings                                                     |
| 8   | Concurrent sends: no race                          | `asyncio.gather(*[client.send(f) for _ in range(100)])`; verify delivery order              |

- `conftest.py`: `autouse=True` session-scoped fixture that starts `wspulse/server` via `asyncio.create_subprocess_exec` and tears it down after tests
- All tests annotated `@pytest.mark.asyncio` (or `asyncio_mode = "auto"` in `pyproject.toml`)

---

## Key Implementation Notes

1. **`asyncio.Queue` head-drop**: `Queue.get_nowait()` removes the oldest item without awaiting — this is the idiomatic head-drop for `asyncio`. Guard with `self._queue.full()`.
2. **Callback dispatch**: callbacks may be synchronous or `async`. Check `asyncio.iscoroutinefunction(fn)` and dispatch via `asyncio.create_task(fn(*args))` or direct call accordingly. Errors in callbacks must be caught and logged, never propagated to the client.
3. **`connect()` as both factory and context manager**: return a `_ConnectContext` class that implements both `__await__` (for `await connect(url)`) and `__aenter__`/`__aexit__` (for `async with connect(url)`).
4. **Type annotations**: use `from __future__ import annotations` everywhere; prefer `X | None` over `Optional[X]`. `mypy --strict` must pass.
5. **`websockets` v12 API**: use `websockets.connect()` (not the legacy `websockets.client.connect()`). The connection is an async context manager; in the reconnect loop, re-enter it after each drop.

---

## Publish Checklist (P6)

- [ ] README: quick-start (factory pattern + context manager pattern)
- [ ] CHANGELOG.md with 0.1.0 entry
- [ ] `python -m build` → `twine upload dist/*` via CI on git tag
- [ ] GitHub release with PyPI badge
- [ ] Register on [PyPI](https://pypi.org) with `trusted publishing` (OIDC, no API tokens)
