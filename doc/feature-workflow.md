# Feature Workflow

Applies to all new features, enhancements, and design changes across the wspulse organization.

---

## Process

### 1. Plan (Idea)

Write initial thoughts to `doc/local/plan/<name>.md`. Include what, why, and rough how. This is local only (git-ignored) — low-cost exploration.

### 2. Quick discussion

Brief feasibility and value check. Is this solving a real problem? Is there a simpler alternative?

### 3. Go / No-go

Kill the idea or proceed. Most ideas should be killed here — fewer items done well beats many items done poorly.

### 4. Layer check

Determine responsibility:
- **Transport layer** (wspulse implements): connection management, message delivery, routing, backpressure, session resumption, heartbeat, codec.
- **Application layer** (user implements, wspulse documents): rate limiting, ACK/delivery guarantees, request-response correlation, auth token refresh, broadcast filtering, message ordering beyond TCP.

If it belongs to the application layer, write a docs recipe showing how to implement it using wspulse's building blocks (router middleware, callbacks, Payload). Do not build it into the library.

### 5. Issue RFC

Open a GitHub issue on `wspulse/.github` with:
- Summary: what and why
- Scope: which repos are affected
- Impact assessment: breaking change? wire protocol change? contract update?
- Assign priority label (`p0`–`p3`) and milestone

### 6. Design discussion

Detailed design in the issue thread:
- API surface (function signatures, option names, error types)
- Cross-SDK parity (same option/behaviour across all client SDKs)
- Contract and protocol spec updates needed
- Edge cases and failure modes

### 7. Task

- Create feature branch from `develop`
- Implement with tests
- Add CHANGELOG `[Unreleased]` entry
- Open PR following the repo's PR template
- Cross-repo changes: execute in dependency order (core → server → testserver → clients → metrics → docs)
