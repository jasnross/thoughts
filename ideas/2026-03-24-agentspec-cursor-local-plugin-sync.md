# Agentspec Sync to Local Cursor Plugin

## Problem Statement

Developers using `agentspec` can generate useful output, but there is no clearly defined workflow to consistently sync that output into a local Cursor plugin installation. Today, this requires a repeated manual copy/edit loop into `~/.cursor/plugins/local/<plugin-name>`, including hand-maintaining plugin metadata at `.cursor-plugin/plugin.json`.

The manual workflow increases friction and introduces avoidable errors, especially around the Cursor plugin metadata file structure (`.cursor-plugin/plugin.json`) and keeping local plugin state consistent across iterations. This slows down development feedback loops when testing plugin changes in Cursor.

## Motivation

- Reduce repetitive manual work in the local plugin test loop.
- Make local plugin iteration faster and less error-prone.
- Ensure a minimal valid Cursor plugin structure is produced consistently, including required metadata.
- Improve reliability and confidence when re-running sync as content evolves.

## Context

- Cursor supports loading local plugins from `~/.cursor/plugins/local`.
- Cursor plugins require `.cursor-plugin/plugin.json` at the plugin root; docs indicate `name` is required and additional metadata is optional but useful.
- `agentspec sync` appears to cover most syncing needs already, but metadata generation/maintenance for the Cursor plugin metadata file (`.cursor-plugin/plugin.json`) is currently a gap in the desired workflow.
- Local development needs two sync modes:
  - Copy mode for explicit generated output snapshots.
  - Symlink mode that links the plugin directory into `~/.cursor/plugins/local/<plugin-name>` so Cursor reflects source changes immediately.
- Full plugin-directory sync/link behavior may not be supported by current `agentspec sync` behavior and may require new capability.
- Primary use case is local development and testing, not plugin marketplace publishing.

## Goals / Success Criteria

- [ ] `agentspec`-driven sync produces a valid `.cursor-plugin/plugin.json` in the local plugin output.
- [ ] Synced output loads as a Cursor local plugin after reload.
- [ ] Re-running sync is idempotent (no drift/duplication from repeat runs).
- [ ] Plugin `name` is supported via explicit input, with deterministic inference fallback when not provided.
- [ ] Workflow supports both copy mode and symlink mode for local Cursor plugin targets.
- [ ] In symlink mode, updates to plugin source are reflected in Cursor's local plugin path without re-copying files.
- [ ] Invalid or missing metadata is surfaced with clear, actionable error messages.

## Non-Goals (Out of Scope)

- Automatic publishing to Cursor Marketplace.
- Team marketplace packaging/distribution workflows.
- Remote distribution/version rollout concerns.
- Broader plugin release process automation beyond local development sync.

## Proposed Solutions (Optional)

### Option A: Extend `agentspec sync` with Cursor plugin metadata file support

Brief description: Keep `agentspec sync` as the main entry point and add first-class generation/validation behavior for the Cursor plugin metadata file (`.cursor-plugin/plugin.json`) in local plugin targets.

Pros:
- Preserves a single command workflow.
- Keeps sync and metadata concerns in one place.

Cons:
- Adds Cursor-specific behavior into existing sync semantics.

### Option B: Add a companion metadata-file helper integrated into sync workflows

Brief description: Keep core sync mostly unchanged and add a helper step/command focused on generating or validating `.cursor-plugin/plugin.json` for local plugin use.

Pros:
- Clear separation of generic sync vs. Cursor-specific metadata concerns.
- Easier to reason about `plugin.json`-only validation paths.

Cons:
- Potentially more than one command in the workflow unless tightly integrated.

## Constraints

- Must work with Cursor's local plugin directory contract (`~/.cursor/plugins/local`).
- Must preserve required plugin root structure expected by Cursor.
- Must handle both copy-based output and symlink-based output safely and predictably.
- Must keep focus on local iteration workflow and avoid scope expansion into publishing/distribution.
- Should not require manual edits to `.cursor-plugin/plugin.json` in the common case.

## Decisions (Resolved)

- Required fields in `.cursor-plugin/plugin.json`: only `name` is required for this local workflow.
- Name source: explicit sync input takes precedence; inference is fallback only.
- Overwrite behavior:
  - If target contains an existing `.cursor-plugin/plugin.json`, overwrite by default.
  - If target exists but does not contain `.cursor-plugin/plugin.json`, error by default.
  - Existing `agentspec` overwrite controls (`--force` and `allow_overwrite`) can bypass default protections.
- Validation timing: run both pre-sync validation (fail fast) and post-sync validation (verify final output).
- Multi-variant behavior: each variant maps to its own plugin directory and unique plugin name, with collision checks.
- Sync modes: support both copy mode and symlink mode for local plugin workflows.

## Current-State Answers (Pre-Planning)

- Current `agentspec sync` does **not** support syncing/linking a full plugin root directory as one unit.
  - Sync is currently per-kind (`agents`, `skills`, `rules`, `commands`) with separate destination paths.
  - `--dest` maps to `<dest>/agents`, `<dest>/skills`, `<dest>/rules`, and `<dest>/commands`, not a plugin-root abstraction.
- Symlink mode currently supports directory symlinks for individual entries (for example each Cursor skill folder), but not a single symlink for an entire plugin directory tree.
- Collision handling semantics already exist and can be reused if directory-level sync is added:
  - Default: collision errors on user-owned paths.
  - `--force` / `allow_overwrite`: permit overwrite with backup behavior.
- Symlink implementation currently uses Unix symlink APIs, so symlink-mode behavior is Unix-oriented today.

## Open Questions

- None currently; ready for planning and design.

## References

- Cursor plugins docs (including local testing): https://cursor.com/docs/plugins
- Cursor plugin metadata file reference (`.cursor-plugin/plugin.json`): https://cursor.com/docs/reference/plugins.md
- Relevant learning on current `agentspec` compiler context: `/Users/jasonr/dotfiles/thoughts/learnings/compiler-fragments.md`

## Affected Systems/Services

- `agentspec` CLI sync workflow.
- Local Cursor plugin filesystem location: `~/.cursor/plugins/local`.
- Cursor plugin metadata file at `.cursor-plugin/plugin.json`.
