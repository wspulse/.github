# Contributing to wspulse

Thank you for your interest in contributing to **wspulse** — a production-ready WebSocket library ecosystem for Go. This document covers conventions shared across all modules. Each repo may have its own `CONTRIBUTING.md` with additional rules.

## Modules

| Module                         | Repository                                                |
| ------------------------------ | --------------------------------------------------------- |
| `github.com/wspulse/core`      | [wspulse/core](https://github.com/wspulse/core)           |
| `github.com/wspulse/server`    | [wspulse/server](https://github.com/wspulse/server)       |
| `github.com/wspulse/client-go` | [wspulse/client-go](https://github.com/wspulse/client-go) |

## Before You Start

- Open an issue to discuss significant changes before starting work.
- For bug fixes, write a failing test that reproduces the issue before modifying production code. The PR must include this test.
- For new features, confirm scope and API design in an issue first.

## Development Setup

Requires: **Go 1.26+**, [golangci-lint](https://golangci-lint.run/), [goimports](https://pkg.go.dev/golang.org/x/tools/cmd/goimports).

```bash
git clone https://github.com/wspulse/<repo>
cd <repo>
go mod tidy
```

## Pre-Commit Checklist

Run `make check` before every commit. It runs in order:

1. `make fmt` — formats all source files
2. `make lint` — runs `go vet` and `golangci-lint`; must pass with zero warnings
3. `make test` — runs tests with `-race`; must pass

If any step fails, do not commit.

## Branching Strategy

- **Never push directly** to `main` or `develop`.
- Use prefixed branches: `feature/`, `bugfix/`, `refactor/`, `fix/`.
- All changes go through pull requests.

## Commit Messages

Follow the format in each project's `.github/instructions/commit-message-instructions.md`:

```
<type>: <subject>

1.<reason> → <change>
```

All commit messages must be in English.

## Naming Conventions

- Use full words for all exported identifiers — `Connection` not `Conn`, `Configuration` not `Cfg`.
- Local/parameter scope: idiomatic Go short names are fine (`conn`, `fn`, `err`, `ok`, `n`, `i`, `v`).
- Allowed short forms for exported identifiers: `ID`, `URL`, `HTTP`, `API`, `JSON`, `Msg`, `Err`, `Buf`.

## API Compatibility

All modules follow semantic versioning. Any change that removes, renames, or alters the signature of an exported symbol is a **breaking change** and requires a major version bump.

- Before removing a symbol, mark it `// Deprecated: use Xxx instead.` in a minor release.
- Adding a method to an exported interface is also a breaking change.

## Pull Request Guidelines

- One PR per logical change.
- Do not reformat code unrelated to your change.
- All CI checks must pass before review.
- Describe what changed and why, not just what the diff shows.

## Module-Specific Rules

Each repo may have additional constraints:

- **core**: zero external dependencies — only Go standard library allowed.
- **server**: all session state mutations must go through the hub event loop (hub serialization rule).
- **client-go**: `Close()` must not leak goroutines — every goroutine has an explicit exit condition.

See each repo's `CONTRIBUTING.md` for full details.
