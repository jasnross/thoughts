# Agentspec Binary Availability and Distribution

## Problem Statement

`agentspec` is a standalone Rust binary, but there is no streamlined release and distribution path that makes it easy for internal users to install and keep it updated. Today, availability depends too much on local build context and ad hoc setup, which increases onboarding friction and creates version drift across users.

Without a reliable release pipeline and a standard install channel, internal users face inconsistent setup experiences, slower adoption, and more support overhead when environment differences cause behavior mismatches.

## Motivation

- Reduce friction for internal users to get and update `agentspec`
- Improve consistency by making vetted binary versions easy to consume
- Increase confidence in rollouts through repeatable release automation
- Establish a foundation for additional distribution channels (for example `asdf`) after the initial Homebrew + `mise` path is stable

## Context

- `agentspec` is a standalone Rust binary in this repository
- Current user goal is to make binaries easily available to internal users first
- First step identified: CI support for building binaries and creating releases
- Release and distribution choices must support company-machine trust requirements and low ongoing maintenance

## Goals / Success Criteria

- [ ] Reproducible release CI builds and publishes versioned binaries
- [ ] Internal users can install/update via a documented, simple command path
- [ ] Homebrew distribution is available and documented for internal users
- [ ] Release process includes operational safeguards (ownership, validation, rollback guidance)
- [ ] Release outputs include SHA256 checksums, SBOM, and GitHub OIDC attestations
- [ ] Release validation gates include artifact smoke tests and Homebrew install verification

## Non-Goals (Out of Scope)

- Detailed implementation design for CI workflows, release jobs, or packaging internals
- Full multi-channel distribution rollout beyond initial Homebrew + `mise` support
- Public OSS growth strategy or broad external-user go-to-market work
- Refactoring unrelated `agentspec` compile/sync functionality

## Proposed Solutions (Optional)

### Selected Approach: Release Artifacts + Homebrew + `mise` in Milestone One

Implement a tag-driven release pipeline that publishes versioned binaries for macOS arm64, macOS x64, and Linux x64, then provides internal install/update through a Homebrew custom tap and `mise` `github:` backend usage.

Release publish criteria include SHA256 checksums, SBOM, GitHub OIDC attestations, artifact smoke tests, and Homebrew install verification, with release publication limited to protected `main` tags (manual hotfix path retained for exceptions).

Rationale: This maximizes internal usability while keeping maintenance manageable and meeting a high-trust security posture for company machine installs.

### Deferred Follow-Up: `asdf` Plugin Support

Defer official `asdf` support until a named owner commits to long-term plugin maintenance.

Rationale: This avoids immediate plugin lifecycle overhead while preserving a clear adoption path when ownership and demand are explicit.

## Constraints

- Internal users are the primary audience for the first milestone
- Homebrew support is required in initial delivery via a custom tap
- `mise` support is required in initial delivery via the `github:` backend
- Operational reliability is required, not just artifact generation
- Distribution choices should align with maintainability and low ongoing overhead
- Milestone-one releases must satisfy high-trust supply-chain requirements for company machine installs

## Open Questions

- Which SBOM format/tooling should be required for release gating?

## Decisions Made

- Use `vX.Y.Z` release tag format
- Milestone one ships stable releases only
- Publish only from protected `main` tags, with manual hotfix dispatch as exception
- Include Homebrew (custom tap) and `mise` (`github:` backend) in milestone one
- Defer `asdf` until a named owner commits to long-term maintenance
- Support latest release plus previous minor
- Require SHA256 checksums, SBOM, artifact smoke tests, and Homebrew install test before release completion
- Require GitHub OIDC attestations in milestone one

## References

- User direction from ideation discussion on CI releases, Homebrew-first distribution, and `asdf`/`mise` research
- Prior learning: `/Users/jasonr/dotfiles/thoughts/learnings/compiler-fragments.md`
- asdf docs: https://asdf-vm.com/manage/plugins.html
- asdf plugin authoring: https://asdf-vm.com/plugins/create.html
- mise backend guidance: https://mise.jdx.dev/dev-tools/backends/ and https://mise.jdx.dev/dev-tools/backends/asdf.html
- mise GitHub backend: https://mise.jdx.dev/dev-tools/backends/github.html
- Homebrew tap maintenance: https://docs.brew.sh/How-to-Create-and-Maintain-a-Tap
- GitHub artifact attestations: https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations-to-establish-provenance-for-builds

## Affected Systems/Services

- `agentspec` Rust binary build and release process
- CI provider workflows for artifact build/publish
- Distribution channel metadata and automation for Homebrew custom tap and `mise` `github:` backend
