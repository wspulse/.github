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
  contracts/
    client-interface.md   # client API contract (all languages)
    client-behaviour.md   # client runtime behaviour contract
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

## Client Contracts

The `doc/contracts/` directory defines the interface and behaviour contracts that all first-party client libraries must implement:

- [client-interface.md](doc/contracts/client-interface.md) — public API surface (`connect`, `send`, `close`, `done`, options, callbacks)
- [client-behaviour.md](doc/contracts/client-behaviour.md) — runtime behaviour (reconnect, heartbeat, backpressure, shutdown sequence)

## License

[MIT](LICENSE)
