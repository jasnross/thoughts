---
date: 2026-04-03T12:33:07Z
git_commit: 4ecfaf5aecbf45e5b5f2f521d57e253a12b006bc
branch: main
repository: agentspec
topic: "Homebrew release pipeline: what's built, what's pending, what's next"
tags: [research, codebase, homebrew, release, distribution, release-please, github-actions]
status: complete
last_updated: 2026-04-03
last_updated_by: claude
---

# Research: Homebrew Release Pipeline Status

**Date**: 2026-04-03T12:33:07Z
**Git Commit**: 4ecfaf5aecbf45e5b5f2f521d57e253a12b006bc
**Branch**: main
**Repository**: agentspec

## Research Question

Where does the agentspec Homebrew release pipeline stand? Was the plan to
release via Homebrew completed?

## Summary

The release infrastructure is **fully built and operational** but **no release
has been published yet**. Every component from the 6-phase plan is implemented
in code: release-please orchestration, multi-platform artifact builds with
supply-chain outputs, Homebrew install gate, cross-repo tap PR automation, CI
license checks, and end-user documentation. A release-please PR for v0.2.0 is
currently open. The Homebrew tap repo exists with a placeholder formula
(version 0.1.0, dummy SHA256 values). Merging the release-please PR is the
trigger that starts the first real release.

## Detailed Findings

### Release Infrastructure (Fully Implemented)

All three GitHub Actions workflows are in place:

- **CI** (`.github/workflows/ci.yml`): lint, test, format, plus a parallel
  `cargo deny check licenses` job backed by `deny.toml` with a 10-license
  allowlist.
- **Release Please** (`.github/workflows/release-please.yml`): runs on push to
  `main` and manual dispatch. Uses `googleapis/release-please-action` pinned to
  SHA. Config in `release-please-config.json` (Rust release type, `vX.Y.Z`
  tags, conventional commit changelog sections) and
  `.release-please-manifest.json` (currently `"0.1.0"`).
- **Release Artifacts** (`.github/workflows/release.yml`): triggered by
  `v*.*.*` tag push or `workflow_dispatch` (hotfix path). Contains four jobs:

  | Job | Purpose |
  |---|---|
  | `build` | Cross-compile for `aarch64-apple-darwin`, `x86_64-apple-darwin`, `x86_64-unknown-linux-gnu`; archive as `agentspec-vX.Y.Z-<target>.tar.gz`; per-archive smoke test (`--version` check) |
  | `homebrew-gate` | Constructs a temporary formula from build artifacts and runs `brew install` + version verification on macOS |
  | `publish` | Tag/Cargo.toml parity check, mainline ancestry guard, SHA256SUMS generation + self-verification, SPDX JSON SBOM via anchore/sbom-action, `gh release upload`, OIDC attestations for archives and SBOM |
  | `tap-pr` | Checks out `jasnross/homebrew-tap` via `HOMEBREW_TAP_PAT`, patches `Formula/agentspec.rb` with new version + SHA256 values, opens or updates a PR on a deterministic branch (`automation/agentspec-<tag>`) |

  All `uses:` references are SHA-pinned. Permissions are scoped per-job.

### Homebrew Tap Repository

`jasnross/homebrew-tap` exists as a public repo. It contains
`Formula/agentspec.rb` at version `0.1.0` with **placeholder SHA256 values**
(`0000...`, `1111...`, `2222...`). The formula structure is complete: platform
dispatching (macOS arm64/x64, Linux x64), `bin.install`, and a version test.

The formula has never been updated by the automated `tap-pr` job because no
release has been published yet. The first successful release run will open a PR
that replaces the placeholder SHA values with real digests.

### Release-Please PR (Open)

PR #1 (`chore(main): release agentspec 0.2.0`) has been open since 2026-03-25.
It bumps the version from 0.1.0 to 0.2.0 and includes a full changelog
covering features, fixes, refactors, docs, tests, and chores accumulated since
the project's inception. Merging this PR will:

1. Update `Cargo.toml` version to `0.2.0`
2. Update `.release-please-manifest.json` to `"0.2.0"`
3. Populate `CHANGELOG.md` with the first real release entry
4. Create a `v0.2.0` tag, which triggers `release.yml`

### Documentation (Complete)

`README.md` already documents:

- Homebrew install/upgrade/uninstall via `brew tap jasnross/tap` +
  `brew install agentspec`
- `mise` install via `github:jasnross/agentspec@latest`
- Supported platforms
- Release artifact naming contract
- Checksum and attestation verification commands
- Release policy (tag format, support window, hotfix path)
- Rollback guidance for bad releases

`CLAUDE.md` contains a maintainer release runbook with a checklist.

### Governance and Licensing

`Cargo.toml` declares `license = "MIT"`. However, **no LICENSE file exists** at
the project root. The formula in `homebrew-tap` also declares `license "MIT"`.

### Plan Phase Completion Matrix

| Phase | Description | Status |
|---|---|---|
| 1 | Governance, licensing, release policy | Done (except LICENSE file) |
| 2 | Release artifact pipeline + supply chain | Done |
| 3 | Homebrew + mise distribution channels | Done (tap repo + formula + docs + tap-pr automation) |
| 4 | Validation gates + rollback | Done (smoke tests, homebrew gate, checksum verification, rollback docs) |
| 5 | Documentation + adoption | Done (README install docs, CLAUDE.md runbook) |
| 6 | Cross-repo tap PR automation | Done (tap-pr job in release.yml) |

### What Has Not Happened Yet

1. **No release has been published.** The release-please PR for v0.2.0 is open
   but not merged.
2. **No tags exist** in the repository.
3. **CHANGELOG.md is empty** (placeholder header only).
4. **The Homebrew formula has placeholder SHA values** and is not installable.
5. **No LICENSE file** at the project root (despite `Cargo.toml` declaring MIT).

## Code References

- `.github/workflows/release.yml` -- full release artifact + tap automation pipeline
- `.github/workflows/release-please.yml` -- version bump orchestration
- `.github/workflows/ci.yml` -- lint/test/license-check
- `release-please-config.json` -- Rust release type, changelog config
- `.release-please-manifest.json` -- current version `0.1.0`
- `deny.toml` -- cargo-deny license allowlist
- `README.md:146-246` -- release policy, install docs, rollback guidance
- `CLAUDE.md` -- release runbook checklist
- `Cargo.toml:3` -- `version = "0.1.0"`
- `Cargo.toml:6` -- `license = "MIT"`
- External: `jasnross/homebrew-tap` -- `Formula/agentspec.rb` (placeholder)
- External: PR #1 -- `chore(main): release agentspec 0.2.0` (open since 2026-03-25)

## Architecture Documentation

The release pipeline follows a chain of dependencies:

```
merge to main
  -> release-please creates/updates PR
     -> merge release PR
        -> release-please creates vX.Y.Z tag
           -> release.yml triggers
              -> build (3 targets, smoke tests)
              -> homebrew-gate (brew install verification)
              -> publish (checksums, SBOM, attestations, asset upload)
              -> tap-pr (patches formula, opens PR on homebrew-tap)
```

The hotfix exception path bypasses the release-please flow:
`workflow_dispatch` with a tag input triggers the same `release.yml` pipeline
but skips the mainline ancestry guard.

## Open Questions

- Is the missing LICENSE file a known deferral or an oversight?
- Is v0.2.0 the intended first public release, or should v0.1.0 be released
  first to validate the pipeline end-to-end?
- Has the `HOMEBREW_TAP_PAT` secret been configured in the repository's GitHub
  Actions secrets?
