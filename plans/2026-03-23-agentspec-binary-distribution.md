# agentspec Binary Distribution Implementation Plan

## Overview

Implement a trusted, low-maintenance binary release and distribution workflow for `agentspec` that serves internal users first. Milestone one ships `release-please`-driven release orchestration plus GitHub Releases with reproducible multi-platform binaries, checksum + SBOM + attestation outputs, and documented installation via Homebrew custom tap and `mise` `github:` backend.

## Current State Analysis

`agentspec` is a buildable Rust binary with strong CI quality checks, but there is no release automation or standardized install channel for internal consumers.

### Key Discoveries:

- [ ] Binary versioning currently comes from crate metadata only (`agentspec/Cargo.toml:1` and `agentspec/Cargo.toml:3`), so release orchestration must enforce Cargo version, tags, and changelog consistency.
- [ ] Current CI runs lint/test only on push + PR (`agentspec/.github/workflows/ci.yml:3` and `agentspec/.github/workflows/ci.yml:21`), with no release job, artifacts, checksums, SBOM, or attestations.
- [ ] User docs currently describe compile/sync/validate/check usage (`agentspec/README.md:12`) but do not describe binary installs from release assets.
- [ ] The idea document already fixes milestone-one direction to Homebrew custom tap + `mise` `github:` backend and defers `asdf` until a named maintainer exists (`thoughts/ideas/2026-03-23-agentspec-binary-distribution.md:41` and `thoughts/ideas/2026-03-23-agentspec-binary-distribution.md:49`).
- [ ] SBOM format decision is now resolved for milestone one: SPDX JSON is required (chosen during planning iteration), which remains compatible with GitHub attestation workflows and independent of Homebrew/`mise` installer compatibility.

## Desired End State

Internal users install/update `agentspec` from vetted release artifacts (not ad hoc local builds), with a predictable operational process and supply-chain trust guarantees.

### Release/Distribution Contract

- [ ] `release-please` is the source of truth for version bump PRs, changelog generation, and release tag creation.
- [ ] Release tags use `vX.Y.Z` and must match `Cargo.toml` package version exactly.
- [ ] Protected `main` tag flow is the default publish path; manual hotfix publish path exists as an explicit exception workflow.
- [ ] Milestone one publishes binaries for macOS arm64, macOS x64, and Linux x64.
- [ ] Every published release includes `SHA256SUMS`, SPDX JSON SBOM, and GitHub OIDC provenance/attestations.
- [ ] Homebrew distribution uses a dedicated tap repository (separate from the source repo) with formula updates tied to release artifacts.
- [ ] Homebrew distribution uses a shared org tap repository convention (`jasnross/homebrew-tap`) with one formula per tool (`Formula/agentspec.rb`).
- [ ] Homebrew custom tap install/update is supported and verified for each release.
- [ ] `mise` `github:` backend install/update is supported and documented for each release.
- [ ] Support policy is latest release plus previous minor.

## What We're NOT Doing

- [ ] Building an official `asdf` plugin in milestone one (explicitly deferred until named long-term owner exists).
- [ ] Expanding to external/public distribution strategy beyond internal-user requirements.
- [ ] Refactoring unrelated `agentspec` compiler/sync features.
- [ ] Designing bespoke packaging formats beyond release binaries + Homebrew formula + `mise` metadata.
- [ ] Implementing broad release analytics/telemetry in this milestone.

## Shared Tap Convention

- [ ] Use a single shared tap repository for org-published CLIs: `jasnross/homebrew-tap`.
- [ ] Keep one formula file per distributable tool under `Formula/` (for this project: `Formula/agentspec.rb`).
- [ ] Define a clear CODEOWNERS/maintainer boundary so source-repo maintainers and tap maintainers are explicit.
- [ ] Require tap CI (`brew audit`/`brew test`) to pass before merging formula updates.
- [ ] Keep formula updates release-coupled: each update references one `release-please` version/tag and corresponding `SHA256SUMS` entries.

## Implementation Approach

Use a phased rollout that starts with governance and licensing, then introduces release orchestration (`release-please`) and artifact automation, then distribution channels, then hard release gates and operational runbooks. Keep artifact naming and metadata stable so Homebrew and `mise` consumers can upgrade safely without frequent formula/schema churn.

---

## Phase 1: Governance, Licensing, and Release Policy

### Overview

Establish the legal and operational contract first so CI/release automation can encode policy rather than inventing it implicitly.

### Changes Required:

#### 1. Repository Licensing Baseline

**File**: `agentspec/LICENSE`

- [ ] Add MIT license text as the canonical repository license.

**File**: `agentspec/Cargo.toml`

- [ ] Add crate license metadata (`license = "MIT"`) so packaged metadata is explicit for downstream tooling.

#### 2. Release Policy + Ownership

**File**: `agentspec/README.md`

- [ ] Add a short release policy section describing tag format (`vX.Y.Z`), supported versions (latest + previous minor), and release ownership expectations.

**File**: `agentspec/CLAUDE.md`

- [ ] Add contributor-facing release policy notes so local maintainers follow one consistent process.

#### 3. Deferred `asdf` Policy

**File**: `agentspec/README.md`

- [ ] Document that `asdf` support is deferred and list the explicit adoption trigger (named owner + maintenance commitment).

### Success Criteria:

#### Automated Verification:

- [ ] `cargo build` passes after license metadata update.
- [ ] `cargo test` passes.
- [ ] `cargo clippy --all-targets` passes.
- [ ] `cargo fmt --check` passes.

#### Manual Verification:

- [ ] Maintainer review confirms MIT licensing is acceptable for internal distribution and dependency posture.
- [ ] Maintainer review confirms policy text matches team expectations for hotfix exceptions.

**Implementation Note**: After this phase and all automated checks pass, pause for human confirmation of policy/legal wording before proceeding.

---

## Phase 2: Release Artifact Pipeline and Supply-Chain Outputs

### Overview

Create `release-please`-driven release orchestration plus tag-triggered GitHub Actions that build release binaries, generate required trust artifacts, and publish release assets.

### Changes Required:

#### 1. `release-please` Orchestration

**File**: `agentspec/.github/workflows/release-please.yml`

- [ ] Add a `release-please` workflow that runs on `main` and manages release PR lifecycle.
- [ ] Configure Rust release type so `Cargo.toml` version and changelog are updated automatically.
- [ ] Ensure `release-please` emits `vX.Y.Z` tags and GitHub Releases from merged release PRs.
- [ ] Restrict release PR merge path to protected branch rules and required checks.

**File**: `agentspec/release-please-config.json`

- [ ] Add release-please configuration for package name, release type, and changelog sections aligned to Conventional Commits.

**File**: `agentspec/.release-please-manifest.json`

- [ ] Add manifest file to track current release version for the repository root package.

#### 2. Artifact Workflow

**File**: `agentspec/.github/workflows/release.yml`

- [ ] Keep tag-triggered artifact workflow for `v*.*.*` plus manual dispatch path for hotfix exceptions.
- [ ] Add preflight step to verify tag version equals `Cargo.toml` package version and matches `release-please` release metadata.
- [ ] Build binaries for target matrix: `aarch64-apple-darwin`, `x86_64-apple-darwin`, `x86_64-unknown-linux-gnu`.
- [ ] Package artifacts with stable naming convention (`agentspec-vX.Y.Z-<target>.tar.gz`).
- [ ] Upload binaries as GitHub Release assets.

#### 3. Checksums + SBOM (SPDX JSON)

**File**: `agentspec/.github/workflows/release.yml`

- [ ] Generate `SHA256SUMS` for all published archives.
- [ ] Generate SPDX JSON SBOM artifact for release outputs.
- [ ] Attach checksums and SBOM to GitHub Release assets.

#### 4. OIDC Attestations

**File**: `agentspec/.github/workflows/release.yml`

- [ ] Use GitHub OIDC-based artifact attestation action for each published binary artifact.
- [ ] Include SBOM attestation linkage for SPDX JSON output.

#### 5. Guardrails and Permissions

**File**: `agentspec/.github/workflows/release.yml`

- [ ] Restrict publish permissions to least-privilege job permissions required for release + attest operations.
- [ ] Require mainline release path to originate from protected `main` tag workflow.

### Success Criteria:

#### Automated Verification:

- [ ] Workflow lints/validates (`gh workflow` syntax check or equivalent local validation path).
- [ ] `release-please` dry-run/validation confirms version/changelog and release PR generation behavior.
- [ ] Dry-run/rehearsal tag in a test namespace produces all three platform artifacts.
- [ ] `SHA256SUMS` file contains all published archives with correct digests.
- [ ] SPDX JSON SBOM artifact is generated and uploaded.
- [ ] Artifact attestations are generated and verifiable for each archive.

#### Manual Verification:

- [ ] Maintainer confirms release page contains binaries, checksums, SBOM, and attestation records.
- [ ] Maintainer confirms `release-please` release PR content (version bump + changelog sections) matches intended user-facing changes.
- [ ] Maintainer verifies artifact names are stable and appropriate for downstream Homebrew/`mise` consumption.

**Implementation Note**: After this phase and all automated checks pass, pause for human confirmation before enabling production tag publishing.

---

## Phase 3: Homebrew and `mise` Distribution Channels

### Overview

Connect published release artifacts to user-facing install channels with minimal ongoing maintenance burden.

### Changes Required:

#### 1. Homebrew Custom Tap Integration

**File**: `jasnross/homebrew-tap/Formula/agentspec.rb`

- [ ] Add/update formula to consume release tarballs and pinned SHA256 values for supported targets.
- [ ] Add `license "MIT"` in formula metadata.
- [ ] Ensure formula install path exposes `agentspec` binary correctly.

**File**: `jasnross/homebrew-tap/README.md`

- [ ] Document shared-tap usage and naming conventions for multi-tool formula ownership.
- [ ] Document required formula update checklist (version/tag parity, checksums, CI pass, maintainer approval).

**File**: `jasnross/homebrew-tap/CODEOWNERS`

- [ ] Add explicit ownership entries for `Formula/agentspec.rb` and shared tap governance files.

**File**: `agentspec/.github/workflows/release.yml`

- [ ] Add optional automation step to open/update tap formula PR (or clearly document manual update procedure if automation is deferred).

**File**: `agentspec/README.md`

- [ ] Document tap repository location and install command contract (`brew tap jasnross/tap` then `brew install agentspec`).
- [ ] Document why the tap is separate (decoupled ACLs, formula CI isolation, and cleaner release automation boundaries).

#### 2. `mise` GitHub Backend Path

**File**: `agentspec/README.md`

- [ ] Add canonical `mise` install/update instructions using the `github:` backend and release tag selection.
- [ ] Document expected archive naming assumptions used by `mise` matching.

#### 3. Artifact Compatibility Contract

**File**: `agentspec/README.md`

- [ ] Document archive/binary naming contract as stable API for distribution channels.
- [ ] Document supported platforms and support window (latest + previous minor).

### Success Criteria:

#### Automated Verification:

- [ ] Homebrew formula audit/test workflow passes in tap CI for updated version.
- [ ] Tap update PR references the exact `release-please` version/tag and published checksums.
- [ ] Tap update PR is approved by designated formula owner per `CODEOWNERS`.
- [ ] `mise` scripted install test succeeds against newly published release assets.
- [ ] `agentspec --version` output from both install paths matches released version.

#### Manual Verification:

- [ ] Maintainer installs via Homebrew tap on macOS and confirms command availability.
- [ ] Maintainer installs via `mise` on a clean environment and confirms upgrade path works.

**Implementation Note**: After this phase and all automated checks pass, pause for human confirmation that installer UX is acceptable before broad rollout.

---

## Phase 4: Validation Gates, Rollback, and Operational Safeguards

### Overview

Add objective release gates so publication only completes when artifact quality and installability are proven.

### Changes Required:

#### 1. Release Gate Jobs

**File**: `agentspec/.github/workflows/release.yml`

- [ ] Add artifact smoke test jobs per platform archive (extract + execute `agentspec --version`).
- [ ] Add Homebrew install verification gate before final release publish completion.
- [ ] Add checksum verification job that re-hashes downloaded release artifacts and compares with `SHA256SUMS`.

#### 2. Rollback and Incident Playbook

**File**: `agentspec/README.md`

- [ ] Add operational rollback guidance for bad release tags (deprecate formula update, cut replacement patch release, communicate install pinning).

**File**: `agentspec/CLAUDE.md`

- [ ] Add maintainer runbook checklist for release execution, failure handling, and hotfix exception process.

#### 3. Dependency License and Compliance Check

**File**: `agentspec/.github/workflows/ci.yml`

- [ ] Add dependency license scan/check step appropriate for Rust crates to detect unknown/incompatible licenses before release.

### Success Criteria:

#### Automated Verification:

- [ ] Release workflow blocks publish when any smoke test fails.
- [ ] Release workflow blocks publish when Homebrew verification fails.
- [ ] Release workflow blocks publish when checksum verification fails.
- [ ] CI license scan/check passes with explicit allowlist policy for internal use.

#### Manual Verification:

- [ ] Maintainer performs one controlled failure drill and confirms rollback instructions are actionable.
- [ ] Maintainer review confirms gates map to stated high-trust requirements in the idea document.

**Implementation Note**: After this phase and all automated checks pass, pause for human confirmation before declaring milestone one complete.

---

## Phase 5: Documentation and Adoption Completion

### Overview

Finalize user/admin docs so the process is self-serve and repeatable without tribal knowledge.

### Changes Required:

#### 1. End-User Installation Documentation

**File**: `agentspec/README.md`

- [ ] Add concise install/update/uninstall sections for Homebrew tap and `mise`.
- [ ] Add shared-tap install guidance with concrete command path (`brew tap jasnross/tap` then `brew install agentspec`).
- [ ] Add artifact integrity verification instructions (checksum + attestation verification path).
- [ ] Add supported platform matrix and troubleshooting notes for failed installs.

#### 2. Maintainer Documentation

**File**: `agentspec/CLAUDE.md`

- [ ] Add release lifecycle checklist (prepare tag, run release, verify outputs, update tap, announce).
- [ ] Add `release-please` maintainer flow (review/merge release PR, confirm generated tag/release, then validate artifact workflow outputs).
- [ ] Add explicit hotfix exception path and decision criteria.

#### 3. Milestone Completion Record

**File**: `thoughts/research/2026-03-23-agentspec-binary-distribution-rollout.md`

- [ ] Record final implementation decisions, known limitations, and deferred `asdf` follow-up trigger.

### Success Criteria:

#### Automated Verification:

- [ ] Documentation link checks (if configured) pass.
- [ ] `cargo fmt --check`, `cargo clippy --all-targets`, and `cargo test` continue passing after workflow/doc updates.

#### Manual Verification:

- [ ] A teammate who did not build from source can install and run `agentspec` using only the published docs.
- [ ] Maintainer confirms runbook is sufficient to execute the next release without ad hoc shell history.

**Implementation Note**: After this phase and all automated checks pass, pause for final human sign-off and then close the milestone.

---

## Phase 6: Cross-Repo Tap Update Automation

### Overview

Automate the handoff from published `agentspec` release assets to a ready-for-review
Homebrew tap PR so release and formula updates stay coupled with minimal manual
steps.

### Changes Required:

#### 1. Tap PR Automation Job

**File**: `agentspec/.github/workflows/release.yml`

- [ ] Add a post-publish job that checks out `jasnross/homebrew-tap` using a dedicated bot token.
- [ ] Parse release `SHA256SUMS` and deterministically update `Formula/agentspec.rb` version, URLs, and SHA256 values.
- [ ] Commit formula updates to a predictable branch name (for update-in-place reruns).
- [ ] Open or update a PR in `jasnross/homebrew-tap` with release-linked title/body.

#### 2. Token and Permissions Contract

**File**: `agentspec/.github/workflows/release.yml`

- [ ] Introduce and document a dedicated secret (e.g., `HOMEBREW_TAP_PAT`) scoped only to required tap repo operations.
- [ ] Keep least-privilege permissions in the source repo workflow while using explicit token only for cross-repo git/PR actions.
- [ ] Fail clearly when token/permissions are missing, with actionable error output.

#### 3. Tap Repository Intake Contract

**File**: `homebrew-tap/.github/workflows/ci.yml`

- [ ] Ensure tap CI always runs `brew audit` and `brew test` on formula update PRs.
- [ ] Ensure CODEOWNERS-required review remains enforced for `Formula/agentspec.rb`.

**File**: `homebrew-tap/README.md`

- [ ] Document the automation contract (source workflow opens PR; tap CI/review gates merge).
- [ ] Document manual fallback steps when automation fails.

### Success Criteria:

#### Automated Verification:

- [ ] A release run creates or updates exactly one tap PR with correct version and SHA values.
- [ ] Re-running the same release workflow updates the existing tap PR branch instead of creating duplicates.
- [ ] Tap PR CI passes (`brew audit` + `brew test`) without manual formula edits.
- [ ] Source release workflow surfaces the tap PR URL in summary output.

#### Manual Verification:

- [ ] Maintainer confirms the generated tap PR is reviewable without additional artifact lookup.
- [ ] Maintainer verifies fallback path is still usable when bot token is unavailable.

**Implementation Note**: After this phase and automated checks pass, this becomes the default path; keep manual tap updates only as documented exception handling.

---

## Testing Strategy

### Unit/Static Validation:

- [ ] Workflow-level static checks for all added GitHub Actions files.
- [ ] `release-please` config and manifest consistency validation for Rust package versioning.
- [ ] Formula-level lint/audit checks in the Homebrew tap pipeline.
- [ ] Rust project quality gates remain green (`cargo fmt --check`, `cargo clippy --all-targets`, `cargo test`).

### Integration/Release Validation:

- [ ] End-to-end dry release run produces all expected artifacts and metadata files.
- [ ] `release-please` run creates a correct release PR and, after merge, emits the expected tag/release trigger.
- [ ] Attestation verification command succeeds for published artifacts.
- [ ] Homebrew install test from tap succeeds against new release.
- [ ] `mise` install test from GitHub release succeeds against new release.

### Manual Testing Steps:

- [ ] Create a release candidate tag and run full release workflow in controlled mode.
- [ ] Install the candidate via Homebrew tap and run `agentspec --version`.
- [ ] Install the candidate via `mise` and run `agentspec --version`.
- [ ] Verify checksum and attestation artifacts from a separate machine/session.
- [ ] Execute documented rollback drill for a simulated broken release.

## Performance Considerations

- [ ] Use Rust build caching in release jobs to keep per-tag runtime reasonable.
- [ ] Keep smoke tests minimal (`--version` + one basic command) so reliability gates do not create excessive release latency.
- [ ] Avoid rebuilding identical artifacts across validation jobs by reusing uploaded artifacts.

## Migration Notes

- [ ] Existing local `cargo install --path .` remains valid for development workflows.
- [ ] Internal user onboarding should switch from local-build instructions to Homebrew/`mise` release channels after milestone completion.
- [ ] `asdf` support remains intentionally deferred; do not imply official support until ownership is assigned.

## References

- [ ] Idea document: `thoughts/ideas/2026-03-23-agentspec-binary-distribution.md`
- [ ] Existing CI workflow baseline: `agentspec/.github/workflows/ci.yml`
- [ ] Existing CLI/version metadata baseline: `agentspec/Cargo.toml`
- [ ] Existing user documentation baseline: `agentspec/README.md`
- [ ] Prior sync implementation plan pattern: `thoughts/plans/2026-03-22-agentspec-sync-command.md`
