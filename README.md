# wspulse/.github

Organization-level shared resources for the [wspulse](https://github.com/wspulse) ecosystem.

> For the ecosystem overview, see the [org profile page](https://github.com/wspulse).

## Contents

```
.github/
  workflows/
    auto-assign.yml       # auto-assign PR author (inherited by all repos)
    go-ci.yml             # reusable Go CI workflow (fmt → lint → race test)
profile/
  README.md               # org profile page (github.com/wspulse)
doc/
  protocol.md           # wire protocol specification (shared)
  contracts/
    client/
      interface.md      # client API contract (all languages)
      behaviour.md      # client runtime behaviour contract
      integration-test-scenarios.md
    server/
      interface.md      # server API contract
      behaviour.md      # server behavioural guarantees
      integration-test-scenarios.md
  plan/
    client-lib-plan.md    # cross-language client plan overview
    client-ts-plan.md     # TypeScript/JS client development plan
    client-kt-plan.md     # Kotlin client plan (future)
    client-swift-plan.md  # Swift client plan (future)
    client-py-plan.md     # Python client plan (future)
CONTRIBUTING.md           # org-wide contribution guidelines
LICENSE                   # MIT
```

## Inherited Workflows

Workflows under `.github/workflows/` are automatically inherited by all repos in the `wspulse` org (unless overridden locally):

- **auto-assign.yml** — assigns the PR author on open
- **go-ci.yml** — reusable CI for Go modules (`fmt → lint → test -race`)

## Contracts

The `doc/contracts/` directory defines the interface and behaviour contracts for both server and client:

**Client:**
- [interface.md](doc/contracts/client/interface.md) — public API surface (`connect`, `send`, `close`, `done`, options, callbacks)
- [behaviour.md](doc/contracts/client/behaviour.md) — runtime behaviour (reconnect, heartbeat, backpressure, shutdown sequence)
- [integration-test-scenarios.md](doc/contracts/client/integration-test-scenarios.md) — shared test scenarios

**Server:**
- [interface.md](doc/contracts/server/interface.md) — public API surface (`Server`, `Connection`, options)
- [behaviour.md](doc/contracts/server/behaviour.md) — behavioural guarantees (hub serialization, callbacks, session resumption)
- [integration-test-scenarios.md](doc/contracts/server/integration-test-scenarios.md) — server test scenarios

**Wire Protocol:**
- [protocol.md](doc/protocol.md) — frame format, heartbeat, session resumption
