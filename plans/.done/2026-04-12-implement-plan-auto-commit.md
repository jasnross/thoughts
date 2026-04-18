# Auto-Commit After Each Phase in Implement-Plan Skills

## Overview

Add a commit step to the shared `implementation-workflow.md` fragment so that
after each phase completes verification and code review, the agent offers to
commit the changes before proceeding. This gives per-phase atomic commits,
cleaner working trees, and better resumability — without modifying the commit
skill or the individual implement-plan skill files.

## Current State Analysis

The post-phase sequence in `implementation-workflow.md` (lines 29–89) currently
runs:

1. Verify success criteria, check off `- [ ]` boxes
2. Spawn `{{ specs.agent.code_reviewer.name }}` agent, fix blockers
3. Pause for human verification (present summary, wait for go-ahead)
4. Proceed to next phase

There is no commit step. Uncommitted changes accumulate across phases and
pauses.

### Key Discoveries:

- The fragment is included by both `implement-plan` and
  `implement-plan-worktree` via `{% include "plans/implementation-workflow.md" %}`
  with template variables `project_path_label` and optionally `worktree`
- Cross-skill references use `{{ specs.skill.<id>.name }}` (e.g.,
  `{{ specs.skill.commit.name }}`) — established pattern in `validate-plan`,
  `split-branch`, and `gt-sync`
- Skill delegation uses prose: "Load the `<skill-name>` skill" — seen in
  `gt-sync` → `resolve-conflicts` and `review-pr` → `gh-safe`
- The fragment already has a consecutive-phase guard: "If instructed to execute
  multiple phases consecutively, skip the pause until the last one" (line 89) —
  the commit deferral hooks into this same condition
- The commit skill (`spec/skills/commit/SKILL.md`) has `agent_invocable: true`,
  confirming it is designed to be loaded by other skills

## Desired End State

After each phase/section completes verification and code review, the workflow
checks for uncommitted changes and, if present, combines the existing pause
message with a commit prompt. The user can approve (triggering the commit
skill's full workflow), request changes first (loop back), or skip the commit
(proceed uncommitted). When consecutive phases are executed with pauses
skipped, commits are also deferred to the final pause.

### Verification:

- Run `agentspec compile` from `agent-config/` — no errors
- Run `agentspec check` — diff shows only the expected changes in generated
  output for `implement-plan` and `implement-plan-worktree`
- Read the generated output for both skills and confirm the commit step appears
  correctly with resolved template variables

## What We're NOT Doing

- Modifying the commit skill itself
- Adding auto-push behavior after commits
- Adding commit behavior to skills other than implement-plan variants
- Changing the fragment's template variable interface (no new `{% with %}` vars
  needed)

## Implementation Approach

The commit step integrates into the existing pause-for-verification section.
Rather than adding a separate step, we restructure the pause message to include
the commit option — reducing interaction round-trips while keeping the user in
control.

## Implementation

### Overview

Modify the "Verification Approach" section of `implementation-workflow.md` to
add a commit-and-proceed step after code review. The pause message is
restructured to present three paths: commit and proceed, request changes, or
skip commit.

### Changes Required:

#### 1. Add commit step to post-phase workflow

**File**: `agent-config/spec/fragments/plans/implementation-workflow.md`

The current "Pause for human verification" bullet in the Verification Approach
section is replaced with a restructured flow that integrates the commit prompt.
The new sequence after the code review bullet:

- [x] Add a **"Commit changes"** bullet after the code review bullet,
      before the pause. This step:

  1. Checks for uncommitted changes (`git status --porcelain`). "Uncommitted
     changes" means any output from this command — staged, unstaged, or
     untracked files.
  2. If no output, skip the commit prompt entirely and use the existing
     pause templates from the current fragment as-is (no commit language
     added). This handles edge cases where verification-only phases produce
     no changes.
  3. If there is output, augment the pause flow with a commit prompt as
     described below

- [x] Modify the **existing pause message templates** to add commit
      language when uncommitted changes are present. The existing templates
      are kept as the base — not replaced — with commit-related lines added:

  For sections **with** manual verification steps, the existing pause
  template is unchanged — it stays focused on manual verification with no
  commit language. After the user confirms manual verification passed,
  present the commit prompt as a follow-up message:

  ```
  Manual verification confirmed. Ready to commit these changes and
  proceed to [next section name]?
  You can also skip the commit to batch with later phases.
  ```

  For sections with **no** manual verification steps and **uncommitted
  changes**, modify the existing pause template by replacing its trailing
  question ("Ready to proceed to [next section name]?") with the commit
  variant:

  ```
  Ready to commit these changes and proceed to [next section name]?
  You can also skip the commit to batch with later phases.
  ```

  For single-section plans, replace "proceed to [next section name]" with
  "wrap up implementation".

- [x] Add instructions for handling user responses at each stage:

  **At the manual verification pause** (sections with manual steps only):

  - **User confirms testing passed**: Proceed to the commit prompt (below).
  - **User reports a failure or requests changes**: Make the requested
    changes, re-run verification as needed, then re-present the manual
    verification pause message.

  **At the commit prompt** (reached directly for no-manual sections, or
  after manual verification is confirmed):

  1. **User approves commit**: Load the `{{ specs.skill.commit.name }}` skill
     if not already loaded, and follow its full workflow (branch check, diff
     review, message drafting, user approval, execution). Then proceed to
     the next phase.
  2. **User requests changes**: Make the requested changes, re-run
     verification as needed, then re-present the commit prompt.
  3. **User skips commit**: Proceed to the next phase with uncommitted
     changes. Note that skipped commits accumulate — the next phase's pause
     will cover all uncommitted changes. This also applies when skip and
     consecutive-phase deferral combine: if a user skips commit in phase 1,
     then instructs consecutive execution for phases 2–3, the final pause
     covers all three phases' changes. This is expected behavior.

- [x] Update the **consecutive-phase guard** (the existing sentence: "If
      instructed to execute multiple phases consecutively, skip the pause
      until the last one") to explicitly include commits:

  ```
  If instructed to execute multiple phases consecutively, skip the pause
  and commit until the last one. At the final phase, present the combined
  pause/commit prompt covering all accumulated changes.
  ```

- [x] Add a note near the commit step (inline, adjacent to the commit
      prompt instructions) clarifying that the existing "Do not check off
      manual testing steps unless the user confirms" rule still applies —
      committing does not imply manual verification passed.

#### 2. Verify no template variable changes needed

**File**: `agent-config/spec/skills/implement-plan/SKILL.md`
**File**: `agent-config/spec/skills/implement-plan-worktree/SKILL.md`

- [x] Confirm that no changes are needed to the `{% with %}` blocks in either
      skill file — the commit skill reference uses `{{ specs.skill.commit.name }}`
      which is a global template variable, not passed via `{% with %}`

### Success Criteria:

#### Automated Verification:

- [x] `agentspec validate` exits 0 (validates specs and frontmatter without
      writing output — catches schema errors that `compile` would also catch,
      but runs faster as a quick smoke test)
- [x] `agentspec compile` exits 0
- [x] `agentspec check` shows changes only in the generated output for skills
      that include `implementation-workflow.md`
  > **Deviation:** `agentspec check` subcommand does not exist in the current
  > version. Verified via `git diff --name-only` after compile — only
  > implement-plan and implement-plan-worktree generated output changed
  > (across all 3 providers), plus the source fragment.

#### Manual Verification:

- [x] Read the generated `implement-plan` output and confirm the commit step
      text appears with `{{ specs.skill.commit.name }}` resolved to the actual
      skill name
- [x] Read the generated `implement-plan-worktree` output and confirm the same
- [x] Verify the consecutive-phase guard text includes the commit deferral
      language in both generated outputs
- [x] Review both outputs and confirm the commit workflow reads naturally and
      covers: commit prompt, skip option, change request loop,
      consecutive-phase deferral, and the no-uncommitted-changes path

## References

- Idea document: `thoughts/ideas/2026-04-12-implement-plan-auto-commit.md`
- Fragment: `agent-config/spec/fragments/plans/implementation-workflow.md`
- Commit skill: `agent-config/spec/skills/commit/SKILL.md`
- Cross-skill reference pattern: `agent-config/spec/skills/validate-plan/SKILL.md:222-225`
- Skill delegation pattern: `agent-config/spec/skills/gt-sync/SKILL.md:188-191`
