# User-Facing Documentation Plan

> Status: planned
> Created: 2026-03-22

## Goal

Create documentation aimed at **SDK consumers** (application developers), separate
from the existing `doc/contracts/` which targets **SDK implementers**.

## Audience Distinction

| Aspect   | Implementer docs (current)                          | User docs (new)                              |
| -------- | --------------------------------------------------- | -------------------------------------------- |
| Location | `.github/doc/contracts/`                            | `.github/docs/` (TBD)                        |
| Reader   | SDK maintainer / contributor                        | App developer using the SDK                  |
| Content  | Exact error messages, validation rules, wire format | Concepts, recipes, recommended values        |
| Tone     | Prescriptive ("must", "panic if")                   | Instructive ("use X when Y", "we recommend") |

## Proposed Structure

```
docs/
├── getting-started.md         # Install + first connection (language tabs)
├── guides/
│   ├── configuration.md       # All options with defaults, recommended values, and gotchas
│   ├── reconnection.md        # Auto-reconnect behaviour, backoff, maxRetries semantics
│   ├── heartbeat.md           # pingPeriod/pongWait tuning, browser limitations
│   ├── error-handling.md      # Four error types — what they mean, how to handle each
│   └── custom-codec.md        # Replace JSONCodec with protobuf/msgpack
├── server/
│   ├── getting-started.md     # Server setup, ConnectFunc, rooms
│   ├── configuration.md       # Server options with defaults and guidance
│   └── session-resumption.md  # ResumeWindow — when and how to use it
├── api/                       # Per-language API reference (or links to generated docs)
│   ├── go.md
│   ├── typescript.md
│   ├── kotlin.md
│   └── swift.md
└── migration/
    └── v0.2-to-v0.3.md       # Breaking change upgrade guide
```

## Content Principles

1. **Scenario-driven** — organize by what the user wants to do, not by internal structure
2. **Language tabs** — shared concepts in one place, code samples per language in tabs
3. **Recommended values** — don't just list ranges, tell users what to pick and why
4. **Copy-pasteable** — every guide includes a working code snippet
5. **No internal details** — never mention panic messages, constant names, hub goroutines

## Tooling (Deferred)

Initial content as plain Markdown in the repo. When content stabilizes, migrate to a
doc site (MkDocs Material or Docusaurus) with:

- Language tab support
- Search
- Versioned docs

## Prerequisite Completed

- [x] Remove "Contract & Protocol" section from all SDK READMEs (user-facing
      README should not link to implementer contracts)

## Open Questions

- Where to host: `.github/docs/` vs dedicated `wspulse/docs` repo?
- Whether to include server docs in the same site or separate
- API reference: generate from source (GoDoc, TypeDoc, Dokka, DocC) or hand-write?
