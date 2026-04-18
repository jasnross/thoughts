# Thoughts Cleanup Skill

## Problem Statement

The `thoughts/` directory accumulates idea and plan files over time, but there
is no mechanism to distinguish completed work from active or pending work. As
the number of files grows, it becomes harder to see what's still in progress.
Plans that have been fully implemented and ideas that have been realized sit
alongside active work, creating noise.

## Motivation

- **Signal-to-noise**: When browsing `plans/` or `ideas/`, completed items
  obscure the active ones. Moving completed items to `done/` subdirectories
  makes the active set immediately visible.
- **Subagent efficiency**: Skills like `tw-thoughts-locator` and
  `tw-thoughts-analyzer` search these directories. Fewer active files means
  faster, more relevant results.
- **Closure**: Explicitly marking work as done provides a sense of completion
  and makes it easier to audit what has been accomplished over time.
- **Low-friction housekeeping**: Without a dedicated skill, cleanup requires
  manually checking each file — tedious enough that it doesn't happen.

## Context

### Current State

- `thoughts/plans/` contains 46 plan files (as of 2026-04-13)
- `thoughts/ideas/` contains 38 idea files
- No `done/` subdirectories exist yet
- Files use `YYYY-MM-DD-slug.md` naming convention
- Neither plans nor ideas have YAML frontmatter or explicit status fields

### Plan Completion Signals

Plans use GitHub-flavored markdown checkboxes (`- [ ]` / `- [x]`) across
multiple sections (Changes Required, Tests, Success Criteria). A plan with all
checkboxes checked is clearly complete. Plans with partial completion are
ambiguous — they could be in-progress, abandoned, or completed outside the
checkbox workflow.

### Idea-to-Plan Relationship

Ideas and their corresponding plans typically share the same filename slug
(ignoring the date prefix), though the dates may differ. Plans also frequently
reference their originating idea file in a References section (e.g.,
`Idea document: thoughts/ideas/2026-04-12-foo.md`). In at least one known case,
the slugs diverge between idea and plan, so slug matching alone is not
sufficient.

### Skill Context

This would be a new skill in the `agent-config/spec/skills/` directory,
following the same frontmatter + markdown body pattern as existing skills like
`triage-todos` and `validate-plan`.

## Goals / Success Criteria

- [ ] Completed plans (all checkboxes checked) are moved to `plans/done/`
- [ ] Ideas linked to completed plans are moved to `ideas/done/`
- [ ] Partially-checked plans are handled with appropriate user interaction based on age
- [ ] Recent plans (< 3 days old) are skipped regardless of completion status
- [ ] Old plans (> 2 weeks) with partial completion are batch-presented for approval
- [ ] Idea-to-plan matching uses both slug comparison and content reference scanning
- [ ] The skill is safe to run repeatedly (idempotent) — already-moved items are not re-processed
- [ ] No files are moved without user awareness (fully-checked plans may auto-move, but the user sees a summary)

## Non-Goals (Out of Scope)

- **Deleting files**: This skill moves files to `done/`, it does not delete
  anything.
- **Modifying file contents**: Files are moved as-is. No status fields are added
  or checkboxes modified.
- **Research or learnings cleanup**: Only `plans/` and `ideas/` are in scope.
  The `research/` and `learnings/` directories have different lifecycle patterns.
- **Automated scheduling**: This is a user-invoked skill, not a background
  process. The user runs it when they want to tidy up.
- **Git operations**: The skill moves files but does not commit or push. The
  user handles version control separately.

## Proposed Behavior

### Classification Algorithm

The skill scans `plans/` and `ideas/` and classifies each file:

**Plans:**

1. Parse all checkboxes in the file (`- [ ]` and `- [x]`)
2. Calculate completion percentage
3. Classify based on completion + age:

| Completion | Age        | Action                                    |
| ---------- | ---------- | ----------------------------------------- |
| 100%       | Any        | Auto-move to `plans/done/`                |
| < 100%     | < 3 days   | Skip (assume active)                      |
| < 100%     | 3d – 2w    | Present individually for review           |
| < 100%     | > 2 weeks  | Batch-present for approval                |
| No boxes   | Any        | Flag for manual review (unusual for plans)|

**Ideas:**

1. Match each idea to a plan using:
   - **Slug matching**: Strip date prefix, compare slugs between `ideas/` and
     `plans/` files
   - **Content reference**: Scan plan files for references to the idea filename
     (e.g., `thoughts/ideas/YYYY-MM-DD-slug.md`)
2. If a matched plan is completed (moved to `done/`), the idea is also a
   candidate for moving
3. Unmatched ideas are presented to the user for review, grouped by age

### User Interaction Flow

1. **Scan**: Read all files in `plans/` and `ideas/`
2. **Classify**: Apply the algorithm above
3. **Report**: Present a summary showing:
   - Plans that will auto-move (fully checked)
   - Plans requiring individual review (middle-age, partial)
   - Old plans for batch approval
   - Ideas matched to completed plans
   - Unmatched/ambiguous ideas
4. **Confirm**: Get user approval before moving any files
5. **Move**: Execute the moves via `git mv` (preserves history) or plain `mv`
6. **Summary**: Report what was moved

### Directory Structure

```
thoughts/
  plans/
    done/           ← completed plans moved here
    active-plan.md
  ideas/
    done/           ← realized ideas moved here
    active-idea.md
```

## Constraints

- Must resolve `$THOUGHTS_DIR` using the standard `thoughts-dir-resolver`
  pattern (shared fragment)
- Must work correctly on first run when `done/` directories don't exist yet
- Date extraction from filenames must handle the `YYYY-MM-DD-slug.md` format
- Checkbox parsing must handle the full file, not just specific sections, since
  checkboxes appear in various locations within plans

## Decisions (Resolved)

- **File moves**: Use `git mv` when the thoughts directory is inside a git repo
  (preserves history, stages the move). Fall back to plain `mv` if git is not
  available or the directory is not tracked.
- **Preview/dry-run**: No separate dry-run mode. The normal flow includes a
  built-in preview step — the skill shows all proposed moves and waits for user
  confirmation before executing. Declining the confirmation effectively serves
  as a dry run.
- **Review detail for middle-age plans**: Show the plan's H1 title, checkbox
  completion ratio (e.g., "12/15 checked"), and file age. Concise enough to
  scan quickly while providing enough context to decide.
- **Unmatched ideas**: Report unmatched ideas in the run summary so the user is
  aware, but do not persist a tracking file. Keeps the skill stateless.

## References

- Existing skill specs: `agent-config/spec/skills/triage-todos/`,
  `agent-config/spec/skills/validate-plan/`
- Thoughts directory: `/Users/jasonr/Workspace/thoughts/`
- Shared fragment for THOUGHTS_DIR resolution: `thoughts/discover-dir.md`

## Affected Systems/Services

- `thoughts/plans/` directory
- `thoughts/ideas/` directory
- New `plans/done/` and `ideas/done/` subdirectories (to be created)
- Agent-config skill spec in `jasnross/dotfiles`
