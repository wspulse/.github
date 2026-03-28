# Branching Strategy

Applies to all repositories in the wspulse organization.

---

## Branch Model

```
feature/*, feat/* ──→ develop ──→ main ──→ tag
bugfix/*           ──→ develop        ↑
refactor/*         ──→ develop    hotfix/* (emergency only)
fix/*              ──→ develop
chore/*            ──→ develop
```

### Branches

| Branch | Purpose | Direct push allowed? |
|--------|---------|---------------------|
| `main` | Released code. Every commit on main corresponds to a tagged release. | No |
| `develop` | Integration branch. Accumulates completed features for the next release. | No |
| `feature/*`, `feat/*` | New features | N/A (short-lived) |
| `bugfix/*` | Bug fixes | N/A (short-lived) |
| `refactor/*` | Restructuring without behaviour change | N/A (short-lived) |
| `fix/*` | Quick fixes (config, docs, CI) | N/A (short-lived) |
| `chore/*` | Maintenance, dependencies, CI/CD | N/A (short-lived) |
| `hotfix/*` | Emergency fixes on main (see below) | N/A (short-lived) |

### Day-to-day workflow

1. Create a branch from `develop` using the appropriate prefix.
2. Open a PR targeting `develop`. Full code review happens here.
3. CI must pass before merge.
4. Merge using **merge commit** (no squash, no rebase).

### Release workflow

1. Verify all intended PRs are merged to `develop`.
2. Update CHANGELOG: `[Unreleased]` to `[vX.Y.Z] - YYYY-MM-DD`.
3. Create a PR from `develop` to `main`. This is a **release checkpoint** — verify CI passes and scope is correct. No full re-review of code.
4. Merge using **merge commit**.
5. Tag on `main`: `git tag vX.Y.Z && git push origin vX.Y.Z`.
6. Merge `main` back into `develop` to keep them in sync:
   ```bash
   git checkout develop
   git merge main
   git push origin develop
   ```

---

## Hotfix Strategy

For critical bugs (security vulnerabilities, data loss) when `develop` contains unreleased work that is not ready to ship:

```
main (vX.Y.Z) ← hotfix/description ← fix the bug
                       ↓
               merge → main → tag vX.Y.(Z+1)
                       ↓
               merge → develop (sync the fix)
```

1. Create `hotfix/*` branch **from `main`** (not develop).
2. Fix the bug. Full code review and CI required.
3. Merge to `main`, tag the patch release.
4. Merge `main` back into `develop` to sync the fix.

This is the **only** case where a branch bypasses `develop`.

---

## Version Maintenance Policy

wspulse follows [Semantic Versioning](https://semver.org/). Within a major version (e.g., 1.x), there are no breaking API changes. Therefore:

- **Fix forward** — bugs are fixed in `develop` and released in the next version. Users upgrade to get the fix.
- **No backport branches** — old minor versions (e.g., 1.1.x, 1.2.x) do not receive patches. Users are expected to upgrade within the same major version.
- **Hotfix** is only for emergencies when `develop` is not in a releasable state.

| Scenario | Strategy |
|----------|----------|
| Bug found, develop is releasable | Fix in develop, release next version |
| Bug found, develop has incomplete work | Hotfix branch from main, patch release |
| Bug in old minor version | Users upgrade to latest (semver guarantees compatibility) |

---

## Merge Strategy

All merges use **merge commits**. Squash merge and rebase merge are not used. This preserves the full commit history and makes it easy to trace individual changes.

---

## CI Triggers

| Event | Branches |
|-------|----------|
| Push | `develop`, `main`, `feat/*`, `feature/*`, `bugfix/*`, `refactor/*`, `fix/*`, `chore/*`, `hotfix/*` |
| Pull request | targeting `develop` or `main` |
| Tag push | No CI trigger (tag is created after CI already passed on main) |

---

## Scope: docs repo

The `docs` repository is excluded from this branching model. It uses the simpler `feature/* → main` workflow with no `develop` branch.
