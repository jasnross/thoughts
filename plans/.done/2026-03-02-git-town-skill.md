# Git-Town Skill & Split-Branch Integration — Implementation Plan

## Overview

Create a standalone `git-town` skill that registers stacked branch parent-child
relationships with git-town, and integrate it into the `split-branch` command so
that stacked branches created during a split are automatically tracked by
git-town's hierarchy.

## Current State Analysis

### Existing skill pattern: `gh-safe`

The only skill currently in `spec/skills/` is `gh-safe` — a directory bundle at
`spec/skills/gh-safe/` containing `SKILL.md` (with frontmatter) and a supporting
bash script. It targets `[claude, cursor]`, sets `user_invocable: false`, and
provides instructions that consuming commands (like `split-branch`) load by name.

### split-branch command

A 6-phase command at `spec/commands/split-branch.md` (536 lines). Phase 1 runs
pre-flight checks, Phase 4 creates branches via worktrees and verifies builds,
Phase 5 optionally creates PRs (loading `gh-safe`), and Phase 6 handles cleanup
and summary. The command tracks dependencies between groups (independent vs
stacked) throughout — this dependency graph is exactly what git-town needs.

### git-town in the dotfiles

- Global config template: `~/dotfiles/git-town.toml`
- Repo setup script: `~/dotfiles/scripts/setup-git-town.sh`
- Shell aliases: `gt` = `git-town`, `gtb` = `git-town branch`
- `.git-town.toml` is globally gitignored (per-repo local config)
- `unknown-type = "feature"` in config → manually-created branches default to
  feature type (no explicit type setting needed)

### Key Discoveries:

- `git-town set-parent <branch>` is fully non-interactive since v19.0.0 — sets
  the parent of the current branch to the given argument without prompting
  (`spec/commands/split-branch.md` consumers can use this directly)
- Skill frontmatter requires: `id`, `kind`, `description`, `version`; the
  `skill` block with `user_invocable` is optional
  (`spec/schema/canonical.schema.json:130-138`)
- Skills compile to `generated/{provider}/skills/{id}/SKILL.md` with only
  `name`, `description`, and optionally `user-invocable` in generated
  frontmatter (`scripts/lib/adapters/claude.ts:32-59`)
- All identifiers must be kebab-case (`agent-docs/canonical-command-conventions.md`)
- Supporting files in skill directories are auto-detected and copied with
  executable bit preserved (`scripts/lib/parse.ts:26-79`)

## Desired End State

After implementation:

1. `spec/skills/git-town/SKILL.md` exists with detection, registration, and
   verification procedures
2. `spec/commands/split-branch.md` detects git-town in Phase 1 and reports it in
   the pre-flight summary
3. `spec/commands/split-branch.md` Phase 6 registers stacked branch parents with
   git-town after worktree cleanup, then verifies with `git-town branch`
4. `npm --prefix agent-config run build:agents` succeeds, producing
   `generated/{claude,opencode,codex,cursor}/skills/git-town/SKILL.md`
5. Generated split-branch skills include the git-town integration text

### How to verify:

- Build passes without errors
- `generated/claude/skills/git-town/SKILL.md` exists with correct frontmatter
  (`name`, `description`, no `user-invocable` since default is fine)
- All 4 provider targets have the git-town skill
- Split-branch generated files include git-town detection in Phase 1 and
  registration in Phase 6

## What We're NOT Doing

- **Replacing `git branch` with `git-town append/hack`** — split-branch's
  worktree execution model stays unchanged
- **PR creation via `git-town propose`** — `gh`-based PR creation stays (richer
  descriptions, repo templates)
- **DAG/multi-parent support** — git-town only supports trees; the skill operates
  within that constraint
- **Managing git-town config** — `setup-git-town.sh` handles that; this skill
  only reads config to detect git-town
- **Running `git-town sync --stack`** — could trigger unexpected rebases; left as
  a user decision
- **Setting branch type** — `unknown-type = "feature"` config already handles
  this

## Implementation Approach

Two deliverables, designed together:

1. **The skill** (`spec/skills/git-town/SKILL.md`) — a pure-instruction skill
   (no supporting scripts) that documents detection, registration, and
   verification procedures. Provider-neutral, targeting all 4 providers.

2. **split-branch modifications** — two surgical insertions into the existing
   spec:
   - Phase 1: Add git-town detection as step 9 (renumber existing step 9 → 10)
   - Phase 6: Restructure to insert registration between cleanup and summary

---

## Phase 1: Create the git-town Skill

### Overview

Create `spec/skills/git-town/SKILL.md` — a standalone skill that other commands
can load to register stacked branch parent-child relationships with git-town.

### Changes Required:

#### 1. Create skill directory and spec

**File**: `spec/skills/git-town/SKILL.md` (new)

- [x] Create directory `spec/skills/git-town/`
- [x] Create `SKILL.md` with the following frontmatter:

```yaml
---
id: git-town
kind: skill
description: >
  Register stacked branch parent-child relationships with git-town. Provides
  detection logic for git-town availability, a non-interactive parent
  registration procedure using set-parent, and verification via git-town branch.
  Load this skill when creating branches that form dependency chains.
version: 1
skill:
  user_invocable: false
compat:
  targets: [claude, opencode, codex, cursor]
---
```

  > **Deviation:** Added `compat.targets` for all 4 providers — the plan's
  > frontmatter sample omitted it but the overview and verification both
  > specify all 4 targets. Without `compat.targets`, the build would use the
  > default target set from the mapping profile.

- [x] Write the skill body with these sections:

**Section 1 — Detection:**

Instructions for checking git-town availability in the current repo:

```bash
# 1. Binary in PATH
command -v git-town >/dev/null 2>&1

# 2. Repo has git-town config (any of these):
#    - git-town.toml in repo root
#    - .git-town.toml in repo root
#    - git-town section in .git/config
test -f git-town.toml || test -f .git-town.toml || git config --local --get-regexp '^git-town\.' >/dev/null 2>&1
```

Both conditions must be true for git-town to be considered active. Store the
result for later use.

**Section 2 — Parent Registration:**

For each stacked branch (a branch whose parent is another feature branch, not
the main/default branch), register the parent-child relationship:

```bash
# Checkout the child branch in the main working tree
git checkout <child-branch>

# Register its parent (non-interactive since git-town v19.0.0)
git-town set-parent <parent-branch>
```

Key rules:
- Only register branches whose parent is another feature branch (stacked).
  Branches based on `main`/`master`/the default branch don't need registration —
  git-town infers this automatically.
- Process branches in dependency order (parents before children) so that
  git-town's tracking is consistent at each step.
- After registration, return to the original branch.

**Section 3 — Verification:**

After all parents are registered, verify the hierarchy:

```bash
git-town branch
```

This prints a tree showing all tracked branches and their parent-child
relationships. Confirm the output matches the expected dependency graph.

**Section 4 — Error Handling:**

- If `set-parent` fails (non-zero exit), report the error and the branch it
  failed on. Do not continue registering subsequent branches — the parent chain
  may be inconsistent.
- If `git-town branch` output doesn't match expectations, warn the user but
  don't attempt to fix it automatically. The user can run `git-town set-parent`
  manually.

### Success Criteria:

#### Automated Verification:

- [x] File exists at `spec/skills/git-town/SKILL.md`
- [x] Frontmatter validates against schema: `npm --prefix agent-config run validate`
  > Validated 30 canonical specs successfully using mapping profile 'work'.
- [x] No other `.md` files in the directory (parse.ts requires exactly one)
- [x] No supporting files needed (pure instruction skill)

---

## Phase 2: Integrate git-town Detection into split-branch Phase 1

### Overview

Add a pre-flight step to split-branch that detects whether git-town is available
and active in the target repository. Report the result in the confirmation
summary so the user knows stacked branches will be registered.

### Changes Required:

#### 1. Add detection step to Phase 1

**File**: `spec/commands/split-branch.md`

- [x] Add a new step 9 after the existing step 8 ("Discover build/test
  command") and before the existing step 9 ("Confirm with user"):

```markdown
9. **Detect git-town:**

   Load the `git-town` skill. Check if git-town is available and active in this
   repository using the skill's detection procedure. Store the result as
   `$GIT_TOWN_ACTIVE` (true/false).

   This is a silent detection step — do not prompt the user. If git-town is
   not detected, the split proceeds normally without registration.
```

- [x] Renumber the existing step 9 ("Confirm with user") to step 10

- [x] Update the pre-flight summary (in the new step 10) to include a git-town
  line:

```
Branch Split Pre-flight

Branch:    feature/big-change
Base:      main @ abc1234
Files:     23 changed (14 added, 7 modified, 2 deleted)
Commits:   8 commits since base
Project:   my-project
Directory: /full/path/to/my-project
Build cmd: sbt compile
Test cmd:  sbt test
git-town:  active — will register stacked branches

Proceed with analysis?
```

When git-town is not detected, the line should read:
`git-town:  not detected`

### Success Criteria:

#### Automated Verification:

- [x] Spec validates: `npm --prefix agent-config run validate`
  > Validated 30 canonical specs successfully.
- [x] Build succeeds: `npm --prefix agent-config run build:agents`
  > Generated 104 files, check confirmed up-to-date.
- [x] Generated split-branch files for all 4 providers include the git-town
  detection step in Phase 1
- [x] Pre-flight summary includes the `git-town:` line in the template

---

## Phase 3: Integrate git-town Registration into split-branch Phase 6

### Overview

Restructure split-branch Phase 6 to insert git-town parent registration after
worktree cleanup but before summary generation. This ensures: (a) only verified
branches are registered (no stale parents), (b) registration runs in the main
working tree (predictable behavior), (c) the summary document includes git-town
status.

### Changes Required:

#### 1. Restructure Phase 6 step ordering

**File**: `spec/commands/split-branch.md`

- [x] Update the Phase 6 subagent strategy note to reflect the new structure:

```markdown
> **Subagent strategy:** The main agent handles worktree cleanup (step 1) and
> git-town registration (step 2) directly — both are lightweight sequential
> operations. Then dispatch a single `general-purpose` subagent for summary
> document generation (step 3). Provide it with: `$PROJECT_DIR`, all branch
> results (names, bases, file counts, build status, PR URLs), and
> `$GIT_TOWN_ACTIVE` plus verification output if applicable. The main agent
> then presents the final summary (step 4) to the user directly.
```

- [x] Reorder Phase 6 steps to:

**Step 1: Clean Up Worktrees** (moved from current step 2, content unchanged):

```markdown
### 1. Clean Up Worktrees

\```bash
# Remove each worktree
git worktree remove "$WORKTREE_BASE/part-N"  # for each

# Remove the base directory
rmdir "$WORKTREE_BASE"

# Prune stale worktree references
git worktree prune
\```
```

**Step 2: Register git-town Parents** (NEW):

```markdown
### 2. Register git-town Parents

**Skip this step if `$GIT_TOWN_ACTIVE` is false or there are no stacked
branches.**

Load the `git-town` skill and follow its parent registration procedure for each
stacked branch. The main agent handles this directly (not a subagent) — it
requires sequential `git checkout` + `git-town set-parent` in the main working
tree.

For each stacked branch, in dependency order (parents before children):

\```bash
git checkout <stacked-branch>
git-town set-parent <parent-branch>
\```

After all registrations, return to the original source branch:

\```bash
git checkout <original-feature-branch>
\```

Verify the hierarchy:

\```bash
git-town branch
\```

Confirm the output matches the expected dependency graph from Phase 3. If
verification fails, warn the user but continue to the summary step — the user
can fix parent relationships manually with `git-town set-parent`.

Store the `git-town branch` output as `$GIT_TOWN_VERIFY_OUTPUT` for inclusion in
the summary document.
```

**Step 3: Generate Split Summary Document** (moved from current step 1):

- [x] Update the summary document template to include a git-town section. After
  the existing "Dependency Graph" section, add:

```markdown
## git-town Status

<if $GIT_TOWN_ACTIVE>
Stacked branch parents registered. Verification output:

\```text
<$GIT_TOWN_VERIFY_OUTPUT>
\```

Useful commands:
- `git-town branch` — view the branch hierarchy
- `git-town sync --stack` — sync all branches in the stack
- `git-town propose --stack` — create PRs for the entire stack (if not already created)

<else>
git-town not detected. To register branches manually:
\```bash
git checkout <stacked-branch>
git-town set-parent <parent-branch>
\```
</if>
```

**Step 4: Present Final Summary** (moved from current step 3):

- [x] Update the final summary table to include a git-town status row:

```
Split Complete: feature/big-change → 3 branches

Branch                                      | Base    | Files | Build | PR
--------------------------------------------|---------|-------|-------|--------
feature/big-change--part-1-data-models      | main    | 4     | pass  | #142
feature/big-change--part-2-api-endpoints    | part-1  | 3     | pass  | #143
feature/big-change--part-3-cleanup          | main    | 2     | pass  | #144

git-town: 2 stacked branches registered (part-2 → part-1 → main)
Merge order: part-3, then part-1, then part-2 (retarget to main after part-1 merges)

Original branch preserved: feature/big-change
Summary saved: $THOUGHTS_DIR/splits/YYYY-MM-DD-big-change.md
```

When git-town is not active, the `git-town:` line reads:
`git-town: not detected (stacked branches not registered)`

### Success Criteria:

#### Automated Verification:

- [x] Spec validates: `npm --prefix agent-config run validate`
  > Validated 30 canonical specs successfully.
- [x] Build succeeds: `npm --prefix agent-config run build:agents`
  > Generated 104 files, check confirmed up-to-date.
- [x] Generated split-branch files for all 4 providers include the restructured
  Phase 6 with git-town registration step
- [x] Summary document template includes the git-town section
- [x] Final summary includes the git-town status line

---

## Phase 4: Build and Verify

### Overview

Compile all specs and verify the generated output is correct for all 4 provider
targets.

### Changes Required:

#### 1. Run the build

- [x] Run `npm --prefix agent-config run build:agents`
- [x] Verify no validation errors
- [x] Verify no compilation errors

#### 2. Verify generated git-town skill

- [x] `generated/claude/skills/git-town/SKILL.md` exists with frontmatter:
  `name: git-town`, `description: ...` (no `user-invocable` since default is
  acceptable)
  > **Deviation:** Generated output includes `user-invocable: false` because
  > the spec explicitly sets `user_invocable: false` and the Claude adapter
  > emits it when set. This is correct and consistent with gh-safe.
- [x] `generated/opencode/skills/git-town/SKILL.md` exists
- [x] `generated/codex/skills/git-town/SKILL.md` exists
- [x] `generated/cursor/skills/git-town/SKILL.md` exists
- [x] No supporting files in any generated git-town skill directory (pure
  instruction skill)
- [x] Body content is identical across all 4 providers (provider-neutral
  instructions)

#### 3. Verify generated split-branch updates

- [x] All 4 generated split-branch files include git-town detection in Phase 1
- [x] All 4 generated split-branch files include the restructured Phase 6 with
  git-town registration step
- [x] Pre-flight summary template includes `git-town:` line
- [x] Final summary template includes `git-town:` line

### Success Criteria:

#### Automated Verification:

- [x] Full build passes: `npm --prefix agent-config run build:agents`
  > Validated 30 specs, generated 104 files, check confirmed up-to-date.
- [x] No warnings or errors in build output
- [x] File count: 4 new files (one `SKILL.md` per provider for the git-town
  skill)
- [x] Diff check: `git diff --stat` shows changes only in expected files
  (`spec/skills/git-town/SKILL.md`, `spec/commands/split-branch.md`, and
  `generated/` files)
  > `generated/` is gitignored — verified via `check:agents` instead.

#### Manual Verification:

- [ ] Read `generated/claude/skills/git-town/SKILL.md` and confirm it contains
  clear, actionable instructions for detection, registration, and verification
- [ ] Read a generated split-branch file and confirm the Phase 1 and Phase 6
  changes are coherent with the rest of the spec

---

## Testing Strategy

### Automated Tests:

- Schema validation via `npm --prefix agent-config run validate`
- Build compilation via `npm --prefix agent-config run build:agents`
- No unit tests to add (the agent-config project validates via schema + build)

### Manual Testing Steps:

1. In a repo with git-town active, run `git-town set-parent --help` to confirm
   the positional argument is supported (already verified in the idea doc, but
   worth a sanity check)
2. Create a test stacked branch, run `git-town set-parent <parent>` manually,
   verify `git-town branch` shows the correct hierarchy
3. Review the generated skill content to ensure the instructions are clear enough
   for an LLM agent to follow without ambiguity

## Performance Considerations

None — the git-town detection and registration steps are lightweight (a few git
commands). No impact on build performance or split-branch execution time.

## Migration Notes

No migration needed. The git-town integration is purely additive:
- Repos without git-town: split-branch detects `$GIT_TOWN_ACTIVE = false` and
  skips all git-town steps. No behavior change.
- Repos with git-town: stacked branches are automatically registered. Users who
  previously ran `git-town set-parent` manually after splits no longer need to.

## References

- Idea document: `thoughts/ideas/2026-03-02-git-town-skill.md`
- gh-safe skill (pattern to follow): `spec/skills/gh-safe/SKILL.md`
- split-branch spec (to modify): `spec/commands/split-branch.md`
- Skill schema: `spec/schema/canonical.schema.json`
- Skill types: `scripts/lib/types.ts:43-80`
- Skill parser: `scripts/lib/parse.ts:26-79`
- Claude adapter: `scripts/lib/adapters/claude.ts:32-59`
- git-town research: `thoughts/research/2026-03-01-git-town-vs-git-branchless.md`
- git-town set-parent docs: https://www.git-town.com/commands/set-parent.html
