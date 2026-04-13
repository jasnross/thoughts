# Thoughts Cleanup Skill Implementation Plan

## Overview

Create a new `cleanup-thoughts` skill in `agent-config/spec/skills/` that scans
`$THOUGHTS_DIR/plans/` and `$THOUGHTS_DIR/ideas/` for completed items and moves
them to `done/` subdirectories. The skill classifies plans by checkbox completion
and age, matches ideas to their corresponding plans via slug and content
reference scanning, and presents an interactive preview before moving any files.

## Current State Analysis

- 46 plan files in `thoughts/plans/`, 38 idea files in `thoughts/ideas/`
- No `done/` subdirectories exist yet
- Files use `YYYY-MM-DD-slug.md` naming convention with no YAML frontmatter
- Plans use `- [ ]` / `- [x]` checkboxes across multiple sections
- Some plans have checkboxes in narrative sections ("Current State", "What We're
  NOT Doing") that are never checked even after full implementation
- Idea files never have checked checkboxes — their `- [ ]` items are goals, not
  work items
- Idea-to-plan references in plans use varied labels (`Idea file:`,
  `Idea document:`, `Idea doc:`, `Related idea:`) and varied path formats
  (relative, absolute, `$THOUGHTS_DIR`-based)
- Some ideas are referenced by multiple plans; some plans have no corresponding
  idea

### Key Discoveries

- Plan `2026-03-23-agentspec-binary-distribution.md` has 149 unchecked checkboxes
  all in narrative sections — a naive "all checked?" check would never classify
  this as complete. The decision is to not try to be smart about section parsing;
  plans with any unchecked boxes go through interactive review tiers.
- Slug matching between ideas and plans is unreliable in ~20% of cases (slugs
  diverge, e.g., `parallel-plan-implementation-with-worktrees` vs
  `implement-plan-worktree-skill`). Content reference scanning (grep for
  `thoughts/ideas/` in plan files) catches these cases.
- Multiple plans can reference the same idea file. The idea should only be moved
  to `done/` when all referencing plans are complete.

## Desired End State

A compiled, working skill that:

1. Can be invoked via `/cleanup-thoughts` (or whatever the compiled name becomes)
2. Scans plans and ideas, classifies each by completion + age
3. Presents an interactive summary before moving anything
4. Moves files using `git mv` (with `mv` fallback) to `plans/done/` and
   `ideas/done/`
5. Reports what was moved and what was skipped

Verification: `agentspec compile` succeeds with the new skill included, and the
generated output is valid.

## What We're NOT Doing

- Deleting files — moves only
- Modifying file contents — no status fields added, no checkboxes changed
- Cleaning up `research/` or `learnings/` directories
- Git commit/push operations
- Automated/scheduled execution — user-invoked only
- Section-aware checkbox parsing — too fragile; all unchecked boxes count

## Implementation Approach

Create a single `SKILL.md` file following the established pattern: YAML
frontmatter + markdown instruction body. The skill uses the
`thoughts/discover-dir.md` fragment (with `writeable = true` to create `done/`
directories) and the `interaction/question-phrasing.md` fragment. The
instruction body walks the agent through a scan → classify → preview → confirm →
move workflow.

## Implementation

### Overview

Create the skill spec directory and SKILL.md file. The skill is entirely
prompt-based — it instructs the agent on the algorithm, not code that runs
independently.

### Changes Required

#### 1. Create Skill Directory and SKILL.md

**File**: `agent-config/spec/skills/cleanup-thoughts/SKILL.md`

- [x] Create the directory `agent-config/spec/skills/cleanup-thoughts/`
- [x] Write the `SKILL.md` file with frontmatter and instruction body
  > **Deviation:** Added `tasks` tool to capabilities and restructured Steps 1-3
  > to delegate file reading to two parallel subagents (plan scanner + idea
  > scanner). The main agent only sees compact summaries, avoiding context bloat
  > from reading 40+ files directly. This was not in the original plan but was
  > identified during manual testing as a necessary optimization.

**Frontmatter:**

```yaml
---
id: cleanup-thoughts
description: >-
  Move completed plans and their linked ideas into done/ subdirectories within
  the thoughts directory.
user_invocable: true
agent_invocable: false
execution:
  preset: balanced
capabilities:
  tools:
    - bash
    - glob
    - grep
    - read
    - write
---
```

Rationale for field choices:
- `agent_invocable: false` — this is a housekeeping skill, not something other
  skills should trigger
- `preset: balanced` (Sonnet) — file scanning and classification doesn't need
  Opus-level reasoning; the interactive review is simple enough for Sonnet
- `write` tool needed to create `done/` directories; `bash` needed for
  `git mv` / `mv` operations

**Instruction body structure:**

The body should include these sections in order:

1. **Title and role** — "You are a housekeeping skill that identifies completed
   work in the thoughts directory and moves it to done/ subdirectories."

2. **What you do / What you don't do** — Scope boundaries matching the idea
   doc's non-goals

3. **THOUGHTS_DIR discovery** — via fragment include:
   ```
   {% with writeable = true %}{% include "thoughts/discover-dir.md" %}{% endwith %}
   ```

4. **Step 1: Scan Plans** — Read all `.md` files in `$THOUGHTS_DIR/plans/`
   (excluding `done/` subdirectory). For each file:
   - Extract the date from the filename (`YYYY-MM-DD` prefix)
   - Calculate file age from today's date
   - Count total `- [ ]` (unchecked) and `- [x]` (checked) checkboxes
   - Extract the H1 title (first `# ` line)
   - Extract the slug (filename without date prefix and `.md` extension)

5. **Step 2: Scan Ideas** — Read all `.md` files in `$THOUGHTS_DIR/ideas/`
   (excluding `done/`). For each file:
   - Extract the date and slug from the filename
   - Record the full filename for matching

6. **Step 3: Match Ideas to Plans** — Two-pass matching:
   - **Pass 1 — Content reference**: Grep all plan files (both active and those
     about to move to `done/`) for references containing `thoughts/ideas/` or
     `ideas/`. Match against idea filenames. Handle varied path formats
     (relative, absolute, `$THOUGHTS_DIR`-based).
   - **Pass 2 — Slug fallback**: For unmatched ideas, compare the slug portion
     (after stripping `YYYY-MM-DD-` prefix) against plan slugs. Use substring
     matching since slugs sometimes have extra prefixes (e.g., idea slug
     `agentspec-unify-emit-sync` should match plan slug `unify-emit-sync`).
   - Record the mapping, noting that one idea may map to multiple plans and
     vice versa.

7. **Step 4: Classify Plans** — Apply the decision matrix:

   | Unchecked boxes | Age       | Classification              |
   | --------------- | --------- | --------------------------- |
   | 0               | Any       | `auto-move`                 |
   | > 0             | < 3 days  | `skip` (assume active)      |
   | > 0             | 3d – 14d  | `review-individual`         |
   | > 0             | > 14 days | `review-batch`              |
   | No checkboxes   | Any       | `review-individual` (unusual)|

   Plans with no checkboxes at all are unusual and warrant individual review
   rather than auto-moving or batch-approving.

8. **Step 5: Classify Ideas** — For each idea:
   - If matched to one or more plans and ALL matched plans are classified as
     `auto-move`: classify the idea as `auto-move`
   - If matched to plans but some are not `auto-move`: classify as
     `skip` (plan not done yet)
   - If unmatched: classify as `unmatched` (report in summary, do not move)

9. **Step 6: Present Preview** — Show a structured summary to the user:

   ```
   ## Cleanup Preview

   ### Auto-Move (fully completed)
   Plans: [count]
   - plan-title (15/15 checked, 30 days old) → plans/done/
   - ...
   Ideas linked to completed plans: [count]
   - idea-title → ideas/done/

   ### Individual Review (3-14 days old, partially checked)
   - plan-title (12/15 checked, 8 days old)
   - ...

   ### Batch Review (>14 days old, partially checked)
   [count] old plans with unchecked items:
   - plan-title (3/20 checked, 45 days old)
   - ...

   ### Skipped (recent, < 3 days old)
   - plan-title (5/10 checked, 1 day old)
   - ...

   ### Unmatched Ideas (no corresponding plan found)
   - idea-title (60 days old)
   - ...
   ```

10. **Step 7: Get Confirmation** — Use your environment's selection/picker UI
    for each review tier:

    - **Auto-move items**: Single yes/no confirmation for the whole batch.
      "Move [N] completed plans and [M] linked ideas to done/?"
    - **Individual review items**: Present each one using your environment's
      selection/picker UI with options: "Move to done", "Skip". Show title,
      checkbox ratio, and age in the question text.
    - **Batch review items**: Single yes/no confirmation. "Move [N] old
      partially-checked plans to done/? These are all >14 days old."

    Include the question-phrasing fragment:
    ```
    {% include "interaction/question-phrasing.md" %}
    ```

11. **Step 8: Execute Moves** — For each approved file:
    - Check if `$THOUGHTS_DIR/plans/done/` or `$THOUGHTS_DIR/ideas/done/`
      exists; create with `mkdir -p` if not
    - Check if the thoughts directory is inside a git repo:
      `git -C "$THOUGHTS_DIR" rev-parse --git-dir 2>/dev/null`
    - If git repo: use `git mv <source> <dest>`
    - If not git repo: use `mv <source> <dest>`
    - When moving an idea, also move the idea if its linked plan(s) were all
      moved in this run

12. **Step 9: Report Summary** — After all moves complete:
    ```
    ## Cleanup Complete

    Moved to plans/done/: [N] plans
    Moved to ideas/done/: [M] ideas
    Skipped (recent): [count]
    Skipped (user declined): [count]
    Unmatched ideas (not moved): [count]
    ```

#### Tests

- [x] `agentspec compile` succeeds with the new skill included
- [x] `agentspec validate` passes without errors
- [ ] `agentspec check` shows the new skill in generated output
  > **Deviation:** `agentspec check` does not exist as a subcommand. Verified
  > manually that the new skill appears in `generated/` across all 3 providers
  > (claude, cursor, opencode).
- [x] The generated skill output contains the resolved fragment content (no raw
  `{% include %}` tags remain)
- [x] The `specs.skill.*` references in the generated output resolve to actual
  skill names (no unresolved template variables)

### Success Criteria

#### Automated Verification

- [x] `agentspec validate` passes: run from `agent-config/`
- [x] `agentspec compile` succeeds: run from `agent-config/`
- [ ] `agentspec check` shows expected diff: the new skill appears in generated
  output with no other unexpected changes
  > **Deviation:** `agentspec check` does not exist. Verified manually that the
  > skill appears in `generated/` for all 3 providers with no unexpected changes.
- [x] No raw MiniJinja template syntax in generated output: grep generated
  files for `{%` and `{{` — only resolved content should remain

#### Manual Verification

- [ ] Invoke the skill in a Claude Code session and verify it correctly scans
  `thoughts/plans/` and `thoughts/ideas/`, classifying files as expected
- [ ] Verify the preview summary is readable and accurate
- [ ] Verify that declining confirmation does not move any files
- [ ] Verify that approving moves files to the correct `done/` subdirectories

## References

- Idea document: `thoughts/ideas/2026-04-13-thoughts-cleanup-skill.md`
- Pattern reference: `agent-config/spec/skills/triage-todos/SKILL.md` (scan →
  classify → present → route pattern)
- Fragment: `agent-config/spec/fragments/thoughts/discover-dir.md`
- Fragment: `agent-config/spec/fragments/interaction/question-phrasing.md`
- Compiler docs: `agent-config/CLAUDE.md`
