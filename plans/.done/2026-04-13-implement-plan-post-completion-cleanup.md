# Post-Completion Cleanup for implement-plan Skills

## Overview

Add a post-completion step to both `implement-plan` and `implement-plan-worktree`
skills that offers to move the completed plan file (and any linked idea files)
to the `.done/` subdirectory within `$THOUGHTS_DIR`. This reuses the archival
pattern established by the `cleanup-thoughts` skill, but in a targeted,
single-plan context — the implementing agent already knows which plan was
completed and can extract idea references directly.

## Current State Analysis

### implement-plan (`spec/skills/implement-plan/SKILL.md`)

- Includes `plans/implementation-workflow.md` at line 42, which defines the
  verification/pause/commit flow including the terminal "Done" picker
- Has a "Resuming Work" section (lines 44–52) but no post-completion logic
- Does not include `interaction/question-phrasing.md` (the fragment is included
  by the `create-plan` skill but not here)

### implement-plan-worktree (`spec/skills/implement-plan-worktree/SKILL.md`)

- Includes `plans/implementation-workflow.md` at line 146
- Has a "Completion and Cleanup" section (lines 148–167) that handles worktree
  removal — the plan archival should happen before this
- Includes `interaction/question-phrasing.md` at line 189

### implementation-workflow.md fragment (`spec/fragments/plans/implementation-workflow.md`)

- Defines the per-phase pause flow with "Done" as the terminal picker option
  for single-section plans and the final phase of multi-phase plans
- After the user selects "Done" (or "Commit and wrap up"), control returns to
  the skill-level instructions
- The fragment itself has no post-completion logic

### cleanup-thoughts skill (`spec/skills/cleanup-thoughts/SKILL.md`)

- Batch-oriented: scans all plans and ideas, classifies by age and completion
- Uses two-pass idea matching: (1) content references (`ideas/*.md` filenames
  in plan text), (2) slug suffix matching with ≥3-segment minimum
- Move mechanics: `mkdir -p` for `.done/` dirs, `git rev-parse` to choose
  `git mv` vs `mv`
- We can reuse the same matching and move patterns in simplified form

### Key Discoveries

- The `discover-dir.md` fragment is included by both skills without
  `writeable = true`, meaning `$THOUGHTS_DIR` resolution fails hard if the
  directory doesn't exist. Since we're post-implementation, the directory must
  already exist — this is fine.
- The worktree skill's cwd is inside the project worktree, not
  `$THOUGHTS_DIR`. The fragment must instruct the agent to `cd` into
  `$THOUGHTS_DIR` for git/move operations, following the user's git convention
  of separate `cd` + `git` calls.
- Both skills already have `bash` in their capabilities, which is needed for
  `mkdir -p`, `mv`, and `git mv`.

## Desired End State

After a plan's implementation is confirmed complete:

1. The agent offers to move the plan to `$THOUGHTS_DIR/plans/.done/`
2. If the plan references ideas, the agent also offers to move those to
   `$THOUGHTS_DIR/ideas/.done/`
3. The user confirms or declines with a single picker interaction
4. Moved files appear in `.done/` subdirectories, preserving their filenames
5. The feature is implemented as a shared fragment included by both skills

### How to Verify

- Run `agentspec compile` from `agent-config/` — no errors
- Read the generated output for both skills and confirm the post-completion
  cleanup section appears after the implementation workflow content
- Confirm the fragment uses MiniJinja conditionals correctly (e.g., the
  worktree-specific `cd` instruction)

## What We're NOT Doing

- Not adding batch classification, age-based rules, or review tiers — that's
  the `cleanup-thoughts` skill's domain
- Not touching the implementation-workflow fragment itself — the new logic is
  a separate concern that belongs after the implementation flow completes
- Not adding new tools to either skill's capabilities — both already have
  `bash`, `glob`, `grep`, and `read`
- Not creating git commits — the user handles version control separately
- Not modifying the `cleanup-thoughts` skill

## Implementation Approach

Create a new fragment and include it in both skills. The fragment handles:
detecting linked ideas, presenting a preview, getting confirmation, and
executing moves. A MiniJinja `worktree` flag controls whether the fragment
emits a `cd $THOUGHTS_DIR` instruction (needed when cwd is a project worktree).

## Implementation

### Overview

Create a shared fragment for post-completion cleanup and wire it into both
implement-plan skills.

### Changes Required

#### 1. New Fragment: `spec/fragments/plans/post-completion-cleanup.md`

**File**: `spec/fragments/plans/post-completion-cleanup.md` (new)

- [x] Create the fragment with the following content:

  **Section heading**: `## Archiving Completed Plans`

  **Trigger**: This section runs after the implementation is confirmed complete
  (the user selected "Done" or "Commit and wrap up" from the final picker).

  **Step 1 — Find linked ideas**: Instruct the agent to scan the plan file
  (which it has already read) for idea references. Use Grep on the plan file
  for lines matching `ideas/` followed by a `.md` filename. Extract just the
  `YYYY-MM-DD-slug.md` portion. Also perform slug suffix matching: extract
  the plan's slug (filename without date prefix and `.md`), then Glob for
  `$THOUGHTS_DIR/ideas/*.md` and compare slugs using the same ≥3-segment
  suffix matching rule from the cleanup-thoughts skill.

  **Step 2 — Present preview**: Show what will be moved:

  ```
  ## Archive Completed Plan

  Plan: [plan filename] → plans/.done/
  Linked ideas: [count, or "none found"]
  - [idea filename] → ideas/.done/
  ```

  If no linked ideas were found, omit the ideas list.

  **Step 3 — Get confirmation**: Present a picker with options:
  - **Archive** (Recommended) — move plan (and linked ideas) to `.done/`
  - **Skip** — leave files in place

  **Step 4 — Execute moves** (if user selected "Archive"):

  Conditional on `worktree` flag:
  - If `worktree` is true: instruct the agent to `cd` into `$THOUGHTS_DIR`
    first (separate Bash call), since cwd is inside the project worktree
  - If `worktree` is false: instruct the agent to `cd` into `$THOUGHTS_DIR`
    as well (the cwd is the project directory, not thoughts)

  Then:
  1. `mkdir -p "$THOUGHTS_DIR/plans/.done/"` (and `ideas/.done/` if ideas exist)
  2. Check if `$THOUGHTS_DIR` is in a git repo:
     `git rev-parse --git-dir 2>/dev/null`
  3. Use `git mv` if in a git repo, otherwise plain `mv`
  4. Move the plan file to `plans/.done/`
  5. Move any linked idea files to `ideas/.done/`

  **Step 5 — Report**: Brief confirmation of what was moved.

  The fragment file must end with a trailing newline.

#### 2. Include Fragment in implement-plan

**File**: `spec/skills/implement-plan/SKILL.md`

- [x] Add a fragment include after the `implementation-workflow.md` include
  (line 42) and before the `## Resuming Work` section (line 44):

  ```
  {% include "plans/post-completion-cleanup.md" %}
  ```

  No variables needed — `worktree` defaults to falsy.

#### 3. Include Fragment in implement-plan-worktree

**File**: `spec/skills/implement-plan-worktree/SKILL.md`

- [x] Add a fragment include after the `implementation-workflow.md` include
  (line 146) and before the `## Completion and Cleanup` section (line 148).
  > **Deviation:** Used a plain `{% include %}` without `{% with worktree = true %}`
  > — the fragment doesn't use the `worktree` variable since both variants
  > need the same `cd $THOUGHTS_DIR` instruction. Keeping it simple.

  ```
  {% include "plans/post-completion-cleanup.md" %}
  ```

#### Tests

- [x] Run `agentspec compile` from `agent-config/` — no errors
- [x] Run `agentspec check` to verify the generated output differs as expected
  (both skills should show the new section)
- [x] Inspect the generated output for `implement-plan` and confirm the
  "Archiving Completed Plans" section appears between the implementation
  workflow content and "Resuming Work"
- [x] Inspect the generated output for `implement-plan-worktree` and confirm
  the section appears between the implementation workflow content and
  "Completion and Cleanup"
- [x] Verify the `worktree` conditional renders correctly: the worktree variant
  should mention cd'ing to `$THOUGHTS_DIR` with an explicit note about leaving
  the worktree cwd
  > **Deviation:** No `worktree` conditional was needed — the fragment uses
  > identical instructions for both variants. Both always `cd` to
  > `$THOUGHTS_DIR` since neither skill's cwd is in the thoughts directory.

### Success Criteria

#### Automated Verification

- [x] `agentspec compile` exits zero with no errors or warnings
- [x] `agentspec validate` exits zero
- [x] Generated output for both skills contains the "Archiving Completed Plans"
  section text

#### Manual Verification

- [x] Review the generated output for both skills to confirm the section reads
  naturally in context and the `worktree` conditional produces the right
  instructions for each variant
  > **Deviation:** No `worktree` conditional needed — both variants use
  > identical instructions. Reviewed generated output for both skills;
  > section reads naturally in both contexts.

## References

- cleanup-thoughts skill: `spec/skills/cleanup-thoughts/SKILL.md` (idea matching logic at lines 136–165, move mechanics at lines 255–283)
- implementation-workflow fragment: `spec/fragments/plans/implementation-workflow.md` (completion flow at lines 90–126)
- MiniJinja fragment conventions: `agent-config/CLAUDE.md` and `thoughts/learnings/compiler-fragments.md`
