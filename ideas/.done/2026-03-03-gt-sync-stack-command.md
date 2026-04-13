# gt-sync-stack: Interactive Stack Synchronization Command

## Problem Statement

When working with git-town stacked branches, running `git-town sync` can
surface merge conflicts that require manual resolution at each branch boundary.
Today, this is a manual process: run sync, notice it stopped, find the
conflicts, resolve them, stage files, run `git-town continue`, and repeat until
the full stack is synced. This is tedious and error-prone, especially for deep
stacks where multiple branches may conflict.

There is no automated assistant that can walk through a stack sync
interactively — detecting conflicts, reading both sides, suggesting
resolutions, and driving `git-town continue` after each resolution — while
keeping the user informed about progress through the stack.

## Motivation

- **Deep stacks amplify friction.** A 4-5 branch stack can require multiple
  stop-fix-continue cycles. Each cycle requires context-switching between
  understanding the conflict, resolving it, staging, and continuing.
- **Conflict resolution benefits from context.** An agent can read surrounding
  code, understand both sides of a conflict, and suggest intelligent
  resolutions — something a bare `git-town sync` cannot do.
- **Consistency with existing workflows.** The project already has interactive
  commands like `split-branch` that orchestrate multi-step git workflows. A
  sync command fits naturally alongside them.
- **Pre-flight safety.** Users sometimes run `git-town sync` with a dirty
  working tree or without realizing git-town isn't configured, leading to
  confusing errors. A command that checks preconditions first provides a better
  experience.

## Context

### Existing Infrastructure

- **`git-town` skill** (`spec/skills/git-town/SKILL.md`): Non-invocable
  reference skill covering detection, parent registration, stack operations
  (append, prepend, sync, diff-parent), and maintenance (compress, detach). The
  new command should reference this skill for detection and verification
  patterns.
- **`split-branch` command** (`spec/commands/split-branch.md`): Existing
  canonical command that demonstrates the interactive command pattern — phased
  execution, pre-flight checks, subagent delegation, user interaction. Serves
  as an architectural reference.
- **Canonical spec system**: Commands are defined in `spec/commands/` with YAML
  frontmatter and markdown body, compiled to provider-specific output via
  `npm run build:agents`.

### git-town sync Behavior

- `git-town sync --stack` syncs only branches in the current stack
  (parent-to-child order).
- `git-town sync --all` syncs all tracked branches.
- When a conflict occurs, sync stops with a non-zero exit code. Conflicted
  files have standard git conflict markers. After resolution, the user stages
  files and runs `git-town continue` to resume.
- `git-town sync` may change the current branch as it processes the stack.
- **Push behavior is not specified** — the command omits `--push` and
  `--no-push`, so git-town respects whatever the user has configured in
  `git-town.toml` (e.g., `push-new-branches = false`).

## Goals / Success Criteria

- [ ] **Pre-flight validation**: Detect missing/unconfigured git-town, dirty
      working tree, and other preconditions before starting sync.
- [ ] **Dirty tree handling**: When uncommitted changes exist, offer the user a
      choice to commit, stash, or abort.
- [ ] **Stack awareness**: Before syncing, show the current stack state
      (`git-town branch`) so the user knows what branches will be affected.
- [ ] **Conflict detection**: After `git-town sync` exits with conflicts,
      identify all conflicted files and present them clearly.
- [ ] **Intelligent resolution suggestions**: For each conflict, read both
      sides, understand the surrounding code context, and suggest a resolution.
      The user approves or modifies each suggestion.
- [ ] **Branch tracking**: After each `git-town continue` cycle, verify which
      branch we're on and report progress through the stack.
- [ ] **Completion summary**: On success, show a summary of what happened
      (branches synced, conflicts resolved) and display the final stack state
      via `git-town branch`.
- [ ] **Scope selection**: Default to `--stack` (current stack only), with
      support for syncing all branches (`--all`) when requested.

## Non-Goals (Out of Scope)

- **Automatic resolution without user approval.** The command suggests
  resolutions but never applies them without explicit user confirmation.
- **Stack creation or parent registration.** The existing `git-town` skill and
  manual workflows handle this. This command focuses solely on synchronization.
- **PR creation after sync.** That's a separate workflow (e.g., `git-town
  propose` or `gh pr create`). The command ends after sync completes.
- **Handling repos without git-town.** If git-town isn't installed or
  configured, the command stops immediately with a clear message. No fallback
  to plain git operations.

## Proposed Approach

### Command Spec as Canonical Command

Define `gt-sync-stack` as a canonical command in `spec/commands/gt-sync-stack.md`
following the same pattern as `split-branch`:

- YAML frontmatter with `kind: command`, tool capabilities, routing trigger
- Phased markdown body describing the interactive workflow
- References the existing `git-town` skill for detection patterns

### Workflow Phases (High-Level)

1. **Pre-flight**: Detect git-town, check working tree, show current stack
2. **Sync**: Run `git-town sync --stack` (or `--all`)
3. **Conflict loop**: If conflicts, identify files → suggest resolutions → user
   approves → stage → `git-town continue` → repeat until clean
4. **Completion**: Summary + `git-town branch` output

### Conflict Resolution Strategy

The command presents conflicts **file-by-file**. For each conflicted file:
1. Read the full file, identifying all conflict marker blocks (`<<<<<<<` …
   `=======` … `>>>>>>>`).
2. For each block (hunk), independently analyze the "ours" and "theirs" sides
   in the context of the surrounding code.
3. Produce a per-hunk recommendation with a brief rationale.
4. Assemble a proposed fully-resolved version of the file, incorporating all
   hunk recommendations.
5. Present this to the user — both the per-hunk breakdown and the final proposed
   file — for approval. The user may approve, modify, or resolve manually.

This approach keeps interaction at the file level (fewer round-trips) while
still surfacing conflict-region reasoning (so each decision is justified).

### Non-Conflict Failure Handling

If `git-town sync` fails for any reason other than merge conflicts (network
error, authentication failure, remote branch deleted, etc.), the command:
- Shows the raw git-town error output clearly
- States that the sync was interrupted and did not complete
- Stops — no attempt to recover or retry

Additionally, if git-town stops due to an unexpected error mid-sync, the command
surfaces `git-town undo` as an available escape hatch to reverse the operation.

## Constraints

- Must work within the canonical spec → generated output compiler pattern
- Must be provider-neutral in the canonical body (Claude, OpenCode, Codex,
  Cursor targets)
- Must reference the `git-town` skill for detection rather than duplicating
  that logic
- The command body should use provider-neutral tool references (mapped via
  `mappings/tools.yaml`)

## Open Questions

None remaining — all questions have been resolved and incorporated into the
Proposed Approach section above.

## References

- Existing git-town skill: `spec/skills/git-town/SKILL.md`
- split-branch command (architectural reference): `spec/commands/split-branch.md`
- git-town sync docs: `git-town help sync`
- Canonical schema: `spec/schema/canonical.schema.json`

## Affected Systems

- `spec/commands/gt-sync-stack.md` — new canonical command spec
- `generated/claude/commands/gt-sync-stack.md` — generated output (auto)
- `generated/opencode/commands/gt-sync-stack.md` — generated output (auto)
- `generated/codex/commands/gt-sync-stack.md` — generated output (auto)
- `generated/cursor/commands/gt-sync-stack.md` — generated output (auto)
- References `spec/skills/git-town/SKILL.md` (read-only dependency)
