# agentspec Binary Distribution Rollout (Milestone One)

## Decisions Implemented

- Release orchestration uses `release-please` with manifest config in the
  source repository (`release-please-config.json`,
  `.release-please-manifest.json`, `.github/workflows/release-please.yml`).
- Artifact publishing is tag-driven (`vX.Y.Z`) with a guarded hotfix exception
  path (`workflow_dispatch`) in `.github/workflows/release.yml`.
- Published artifact contract is stable:
  - `agentspec-vX.Y.Z-aarch64-apple-darwin.tar.gz`
  - `agentspec-vX.Y.Z-x86_64-apple-darwin.tar.gz`
  - `agentspec-vX.Y.Z-x86_64-unknown-linux-gnu.tar.gz`
- Supply-chain outputs include `SHA256SUMS`, SPDX JSON SBOM, and GitHub OIDC
  attestations.
- Homebrew distribution is via shared tap repository (`jasnross/homebrew-tap`)
  with `Formula/agentspec.rb` ownership tracked in `CODEOWNERS`.
- `mise` installation path is documented via `github:` backend against release
  assets.

## Safeguards Added

- Release workflow performs:
  - tag/version parity check against `Cargo.toml`
  - release metadata validation from GitHub API
  - protected mainline tag ancestry guard (except explicit hotfix dispatch)
  - per-target archive smoke tests
  - checksum self-verification gate
  - Homebrew install gate using a temporary formula from release archives
- CI includes dependency license checks (`cargo deny check licenses`).

## Known Limitations

- Homebrew formula updates are still manual (automation deferred).
- Homebrew gate in release workflow validates installability from generated
  release artifacts, not from the merged tap formula PR.
- Tap CI local verification may depend on local Command Line Tools version.

## Deferred Work

- `asdf` plugin support remains deferred until a named long-term maintainer is
  assigned and commits to upkeep.

## Follow-up

- Consider automating tap PR creation after release assets are published.
- Consider adding release documentation link checks if docs tooling is adopted.
