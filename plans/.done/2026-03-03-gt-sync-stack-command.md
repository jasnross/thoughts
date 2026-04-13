# gt-sync Command Implementation Plan

## Overview

Create a new canonical command spec `spec/commands/gt-sync.md` that
provides an interactive, agent-assisted `git-town sync` workflow. The command
runs `git-town sync --stack` (or `--all`), detects merge conflicts, analyzes
each conflicted file with hunk-by-hunk reasoning, proposes resolutions for user
approval, and loops via `git-town continue` until the stack is clean.

## Current State Analysis

### Existing infrastructure

- **`spec/skills/git-town/SKILL.md`** — Non-invocable skill covering detection,
  parent registration, stack sync (`git-town sync --stack`/`--all`), and
  maintenance. The new command delegates all detection to this skill.
- **`spec/commands/split-branch.md`** — Architectural reference: phased
  execution, pre-flight checks, subagent delegation for context-heavy work,
  main-agent owns user interaction and sequencing.
- **`spec/schema/canonical.schema.json`** — `kind: command` requires a
  `command` block. Frontmatter fields `id`, `name`, `routing.trigger` must all
  be kebab-case and aligned.
- **`mappings/tools.yaml`** — Canonical tool names: `read`, `write`, `edit`,
  `grep`, `glob`, `bash`, `task`, `todowrite`, `ls`.
- **`agent-docs/canonical-command-conventions.md`** — `id`, `name`, filename,
  and routing trigger must all be identical kebab-case.

### Key git-town sync behavior

- `git-town sync --stack` syncs parent-to-child; `--all` syncs every tracked
  branch.
- When a conflict occurs, sync exits non-zero and leaves unmerged files in the
  working tree.
- Conflicted files are identified by: `git diff --name-only --diff-filter=U`
- After resolution: stage files with `git add`, then run `git-town continue`.
- `git-town undo` reverses the interrupted sync (escape hatch for non-conflict
  failures).
- Push behavior: omit `--push`/`--no-push` flags — sync respects the user's
  `git-town.toml` settings.

### Key Discoveries

- Conflict detection pattern: `git diff --name-only --diff-filter=U` lists
  exactly the unmerged files after a conflict stop.
- The `git-town` skill (loaded by the command) is the source of truth for
  binary detection and sync semantics.
- `split-branch` uses `model_profile: planning` and `disable_model_invocation:
  true` — same pattern applies here.
- `edit` tool should be included in capabilities (alongside `write`) since
  conflict resolution involves targeted in-place edits.
- Parent registration check: `git-town config get-parent` (for the current
  branch) must be non-empty before a `--stack` sync, because git-town needs
  the parent relationship to know what "the stack" is. For `--all`, no check
  needed — git-town already knows its tracked branch list.
- Conflict summary-first UX: analyze all conflicted files in parallel
  (subagents), then show a compact summary table. Offer "Approve All",
  "Review All", or "Review flagged only" before diving into per-file review.
  "Flagged" means the subagent couldn't produce a confident recommendation
  (e.g., structural conflict, both sides changed the same logic differently).
  This avoids presenting a wall of diffs immediately while keeping full
  detail available on demand.

## Desired End State

After implementation:

1. `spec/commands/gt-sync.md` exists with correct YAML frontmatter and
   a complete phased command body.
   > **Deviation:** Command renamed from `gt-sync-stack` to `gt-sync` per user request — shorter and matches `git-town sync` more directly.
2. `npm --prefix agent-config run build:agents` succeeds, producing
   `generated/{claude,opencode,codex,cursor}/skills/gt-sync/SKILL.md` for
   all four provider targets.
3. `npm --prefix agent-config run validate` succeeds with no errors.
4. The generated command content is coherent and actionable for an LLM agent.

### How to verify

- `generated/claude/skills/gt-sync/SKILL.md` exists with correct
  frontmatter (`name: gt-sync`, `description: ...`)
- All four provider targets have the skill
- Build reports the expected file count increase (4 new files, one per
  provider)

## What We're NOT Doing

- **Automatic conflict resolution without user approval.** The command only
  suggests — the user must approve every resolved file.
- **Fallback to plain git operations.** If git-town isn't installed or
  configured, the command stops immediately. No `git merge`/`git rebase`
  fallback.
- **Stack creation or parent registration.** If the current branch has no
  parent registered, the command stops and tells the user to run
  `git-town set-parent`. It does not attempt to register parents itself.
- **Full stack parent traversal.** Only the current branch's parent is
  checked pre-flight. Ancestor branches with missing parents are caught
  at runtime by git-town (surfaced via the non-conflict error handler).
- **PR creation after sync.** That's a separate workflow.
- **Handling repos without git-town.** Out of scope by design.
- **Retry logic for non-conflict failures.** Network errors, auth failures,
  deleted remotes → show error, surface `git-town undo`, stop.

---

## Phase 1: Write the Canonical Command Spec

### Overview

Create `spec/commands/gt-sync.md` with YAML frontmatter and a complete
phased command body following the `split-branch` architectural pattern.

### Changes Required

#### 1. Create spec file

**File**: `spec/commands/gt-sync.md` (new)

- [x] Create `spec/commands/gt-sync.md` with the following frontmatter:

```yaml
---
id: gt-sync
kind: command
name: gt-sync
description: Interactively sync a git-town stack, detecting merge conflicts and resolving them file-by-file with per-hunk reasoning and user approval
version: 1
execution:
  model_profile: planning
  mode: command
capabilities:
  tools:
    - read
    - write
    - edit
    - glob
    - grep
    - bash
    - task
    - todowrite
command:
  accepts_args: true
  args_schema: string
  disable_model_invocation: true
routing:
  trigger: gt-sync
compat:
  targets:
    - claude
    - opencode
    - codex
    - cursor
---
```

#### 2. Write the command body

The body is structured into four phases. All content must be provider-neutral:

- **No provider-specific tool names in prose.** Do not write `AskUserQuestion`,
  `Task`, `Read`, `Write`, `Bash` etc. — use plain language: "ask the user",
  "present the user with options and wait for selection", "read the file",
  "write the file". Tool names only appear in the capabilities frontmatter and
  (where unavoidable) in brief parenthetical references like "(via the `task`
  tool)".
- **No agent-type names.** Do not write `general-purpose` subagent — write
  "subagent" or "subagent (via the `task` tool)".
- Canonical tool names from `mappings/tools.yaml` may appear in `bash` code
  blocks where a shell command is being shown, but not in running prose.

##### Phase 1 — Pre-flight

- [x] **Determine scope**: Parse the args string. If `--all` is present, set
  `$SCOPE=--all`. Otherwise default to `$SCOPE=--stack`. Show the selected
  scope in the pre-flight summary.

- [x] **Detect git-town**: Load the `git-town` skill. Run detection using the
  skill's detection procedure:
  ```bash
  command -v git-town >/dev/null 2>&1
  test -f git-town.toml || test -f .git-town.toml || \
    git config --local --get-regexp '^git-town\.' >/dev/null 2>&1
  ```
  Both conditions must pass. If git-town is not active, stop immediately with
  a clear message: "git-town is not installed or not configured in this
  repository. Run `git-town help` to get started."

- [x] **Check parent registration** (for `--stack` scope only):
  ```bash
  git-town config get-parent
  ```
  If the output is empty AND the current branch is not `main`/`master`/the
  default branch, stop immediately:
  ```
  Branch '<current>' has no registered git-town parent.
  Before syncing, register its parent:
    git-town set-parent <parent-branch>
  ```
  Skip this check when `$SCOPE=--all` — git-town tracks all branches
  independently of the current branch's parent chain.

  Note: if an ancestor branch higher up the stack also lacks a parent,
  git-town will fail when it reaches that branch during sync. The
  non-conflict failure handler in Phase 2 will surface that error clearly.

- [x] **Check working tree**:
  ```bash
  git status --porcelain
  ```
  If the output is non-empty (dirty tree), present the user with three choices
  and wait for selection:
  1. **Commit** — stage and commit current changes before syncing
  2. **Stash** — run `git stash push -m "gt-sync: pre-sync stash"` and
     restore after sync with `git stash pop`
  3. **Abort** — stop without syncing

  If the user chooses **Commit**, ask for a commit message, then:
  ```bash
  git add -A
  git commit -m "<user message>"
  ```
  If the user chooses **Stash**, stash now. Track `$STASH_USED=true` and pop
  after a successful sync (see Phase 4). If stash pop fails, warn the user
  to resolve the conflict manually — do not abort the summary.

- [x] **Preview sync operations** (dry-run):
  ```bash
  git-town sync $SCOPE --dry-run
  ```
  Capture the output. This shows the rebase and push operations that sync
  will perform, giving the user a concrete preview before committing. If
  dry-run exits non-zero for any reason other than an unsupported flag
  (e.g., older git-town version), treat it as advisory: note that preview
  was unavailable and continue to confirmation. Do not block on dry-run
  failure.

- [x] **Confirm**: Present a consolidated pre-flight summary including the
  dry-run output and ask the user to confirm before proceeding. Example format:

  ```
  gt-sync Pre-flight

  Scope:     --stack (current stack only)
  Stack:
    main
    └── feature/auth-refactor (current)
        └── feature/auth-tests

  Preview:
    [git-town sync --dry-run output]

  Working tree: clean

  Proceed with sync?
  ```

  When `--all` is selected, the scope line reads `--all (all tracked branches)`.
  When dry-run output is unavailable, omit the Preview section entirely.

##### Phase 2 — Sync

- [x] **Run sync**:
  ```bash
  git-town sync $SCOPE
  ```
  Capture the exit code.

  - **Exit 0**: sync completed without conflicts. Jump to Phase 4 (Completion).
  - **Non-zero exit**: check for conflicts (next step).

- [x] **Distinguish conflict stop from other failure**:
  ```bash
  git diff --name-only --diff-filter=U
  ```
  - If this returns one or more files → merge conflicts detected. Enter
    Phase 3 (Conflict Loop).
  - If this returns nothing → non-conflict failure. Show the raw git-town
    error output, state that the sync was interrupted and did not complete,
    surface `git-town undo` as an escape hatch, and stop.

##### Phase 3 — Conflict Loop

This phase repeats until `git-town sync` / `git-town continue` exits cleanly.

- [x] **Identify conflicted files**:
  ```bash
  git diff --name-only --diff-filter=U
  ```

- [x] **Report conflict location**: Show which branch the conflict occurred on:
  ```bash
  git branch --show-current
  ```
  Display: "Conflict on branch `<branch>` — N file(s) to resolve."

- [x] **Analyze conflicted files** (main agent dispatches subagents):

  For each conflicted file, dispatch a subagent (via the `task` tool) in
  parallel with:
  - The full file content (with conflict markers)
  - Instruction to identify every `<<<<<<< … ======= … >>>>>>>` block
  - Instruction to analyze "ours" (above `=======`) and "theirs" (below
    `=======`) for each block in the context of surrounding code
  - Instruction to produce:
    (a) a confidence level for each hunk: `clear` (one side is obviously
        correct) or `needs-review` (structural or ambiguous conflict)
    (b) per-hunk recommendation with rationale
    (c) a fully-resolved version of the file incorporating all recommendations
  - The subagent must NOT write the file — it returns all of the above to the
    main agent

  Wait for all subagents to complete before presenting to the user.

- [x] **Present conflict summary** (main agent):

  After all analyses are done, show a compact summary table:

  ```
  Conflict on branch feature/auth-refactor — 3 files

  File                          Hunks  Status
  ────────────────────────────  ─────  ────────────────────────
  src/auth/login.ts             2      clear (ours preferred)
  src/auth/session.ts           1      clear (theirs preferred)
  src/config/auth.yaml          1      needs review
  ```

  Then present the user with the following options and wait for selection:
  - **Approve clear** — apply all `clear` recommendations immediately; queue
    `needs-review` files for one-at-a-time review
  - **Review all** — go through every file one at a time
  - **Approve all** — accept all recommendations as-is (including
    `needs-review` files)

- [x] **Process files per user choice**:

  **For files being approved without review:** write the resolved content
  directly using the `write` tool.

  **For files being reviewed** (one at a time):
  1. Show the per-hunk breakdown (which side was chosen and why, with
     surrounding code context)
  2. Show the proposed fully-resolved file content
  3. Ask the user:
     - **Approve** — accept the proposal as-is
     - **Edit** — user provides modified content or instructions; incorporate
       their changes before writing
     - **Resolve manually** — skip writing; user will edit the file themselves
       before staging

- [x] **Write resolved files**: For each approved file, write the resolved
  content using the `write` tool (replacing the conflict-marker version).

- [x] **Stage resolved files**:
  ```bash
  git add <file1> <file2> ...
  ```
  Use specific file paths — never `git add -A` or `git add .`.

- [x] **Continue sync**:
  ```bash
  git-town continue
  ```
  Capture the exit code. Apply the same logic as Phase 2:
  - Exit 0 → Phase 4 (Completion)
  - Non-zero + unmerged files → repeat Phase 3 (another conflict stop, possibly
    on a different branch in the stack)
  - Non-zero + no unmerged files → non-conflict failure; show error, surface
    `git-town undo`, stop

  Before each `git-town continue`, increment a conflict-round counter and
  display progress: "Continuing sync (round N)…"

##### Phase 4 — Completion

- [x] **Restore stash** (if `$STASH_USED=true`):
  ```bash
  git stash pop
  ```
  If this fails (pop conflict), warn the user: "Stash restore encountered a
  conflict. Your pre-sync changes are in stash[0] — resolve manually with
  `git stash pop`." Do not abort the summary.

- [x] **Show completion summary**. Example:

  ```
  Sync Complete

  Scope:    --stack
  Branches synced: 3
    ✓ main (no conflicts)
    ✓ feature/auth-refactor (1 conflict resolved)
    ✓ feature/auth-tests (2 conflicts resolved)

  Total conflicts resolved: 3 (across 2 files)
  ```

  Track the count of resolved conflicts across all loop iterations.

- [x] **Show final stack state**:
  ```bash
  git-town branch
  ```

#### 3. Agent architecture notes (included in the command body)

- [x] Add a brief "Agent Architecture" section near the top (following
  `split-branch` pattern) that describes:
  - **Main agent** owns: pre-flight, user interaction, sequencing, staging,
    `git-town continue` invocation, completion summary
  - **Subagents** (via the `task` tool) for: per-file conflict analysis
    (context-heavy, parallelizable)
  - Subagents must not ask the user questions — if blocked, report back to the
    main agent
  - Language is provider-neutral throughout; no provider-specific tool names
    appear in prose

### Success Criteria

#### Automated Verification

- [x] `spec/commands/gt-sync.md` exists
- [x] Schema validates: `npm --prefix agent-config run validate`
  > **Deviation:** Actual script name is `npm run validate:agents` (not `npm --prefix agent-config run validate`)
- [x] No naming drift: `id`, `name`, `routing.trigger`, and filename are all
  `gt-sync`

---

## Phase 2: Build and Verify

### Overview

Run the build pipeline and confirm all four provider targets are generated
correctly.

### Changes Required

#### 1. Run the build

- [x] Run `npm --prefix agent-config run build:agents`
  > **Deviation:** Ran as `npm run build:agents` (already in agent-config dir)
- [x] Confirm no validation errors
- [x] Confirm no compilation errors

#### 2. Verify generated output

- [x] `generated/claude/skills/gt-sync/SKILL.md` exists with correct
  frontmatter (`name: gt-sync`, `description: ...`)
- [x] `generated/opencode/skills/gt-sync/SKILL.md` exists
- [x] `generated/codex/skills/gt-sync/SKILL.md` exists
- [x] `generated/cursor/skills/gt-sync/SKILL.md` exists
- [x] No supporting files in any generated directory (pure-instruction command)
- [x] Build reports 4 new files (one per provider)
  > **Deviation:** Build reports total of 108 files (all specs), not incremental count. Verified 4 new gt-sync files exist via glob.

### Success Criteria

#### Automated Verification

- [x] Full build passes: `npm --prefix agent-config run build:agents`
- [x] No warnings or errors in build output

#### Manual Verification

- [ ] Read `generated/claude/skills/gt-sync/SKILL.md` and confirm the
  phased content is coherent: pre-flight → sync → conflict loop → completion
- [ ] Confirm the conflict resolution loop's "analyze → present → approve →
  stage → continue" cycle is clearly described and actionable for an LLM
- [ ] Confirm provider-neutrality: no Claude/OpenCode-specific tool names in
  the body text

---

## Testing Strategy

### Automated Tests

- Schema validation via `npm --prefix agent-config run validate`
- Build compilation via `npm --prefix agent-config run build:agents`
- No unit tests to add (the project validates via schema + build only)

### Manual Testing Steps

1. In a repo with git-town active and a stacked branch with an upstream
   conflict, invoke `/gt-sync` and verify the pre-flight summary shows
   the correct stack
2. Simulate a conflict by creating divergent changes on parent and child
   branches; verify the conflict loop correctly identifies files, presents
   per-hunk reasoning, and stages after approval
3. Verify `--all` arg changes the scope line in the pre-flight summary
4. Verify that invoking in a repo without git-town produces a clean stop
   message (no cryptic errors)

## References

- Idea document: `thoughts/ideas/2026-03-03-gt-sync-stack-command.md`
- Architectural reference: `spec/commands/split-branch.md`
- git-town skill: `spec/skills/git-town/SKILL.md`
- Schema: `spec/schema/canonical.schema.json`
- Tool mappings: `mappings/tools.yaml`
- Naming conventions: `agent-docs/canonical-command-conventions.md`
- Prior git-town plan: `thoughts/plans/2026-03-02-git-town-skill.md`
