# Auto-Commit After Each Phase in Implement-Plan Skills

## Problem Statement

The `implement-plan` and `implement-plan-worktree` skills complete entire
phases of work without committing. This leaves large amounts of uncommitted
changes in the working tree across pauses, making it harder to resume sessions,
review incremental progress, and reason about what changed in each phase.

The existing `commit` skill already handles branch detection, commit message
drafting, and user approval — but the implement-plan skills never invoke it.

## Motivation

- **Resumability**: If a session crashes or is interrupted mid-implementation,
  uncommitted work is lost. Committing after each phase creates natural
  checkpoints.
- **Reviewability**: Atomic, per-phase commits make it easier to review what
  changed and why, both during implementation and after the fact.
- **Clean working tree**: Each phase starts with a clean slate, reducing
  confusion about which changes belong to which phase.
- **Bisectability**: Per-phase commits that each represent a working state
  enable `git bisect` if something breaks later.

## Context

### Current Workflow (implementation-workflow.md fragment)

The shared implementation workflow fragment defines the post-phase sequence as:

1. Run success criteria checks, mark checkboxes
2. Spawn code reviewer agent
3. Fix blockers
4. Pause for human verification
5. Proceed to next phase

There is no commit step anywhere in this sequence.

### Commit Skill

The `commit` skill (`spec/skills/commit/SKILL.md`) handles:

- Checking the current branch (prompts to create a feature branch if on
  main/master)
- Reviewing changes via `git status` and `git diff`
- Planning logical commit grouping
- Presenting the plan to the user for approval
- Executing the commit

### Worktree Variant

The `implement-plan-worktree` skill creates a dedicated feature branch during
worktree setup (e.g., `jasonr/plan-slug`). The commit skill's branch check
will see this feature branch and proceed without prompting — no special handling
needed.

## Goals / Success Criteria

- [ ] After each phase/section completes verification + code review, the
      workflow asks the user if they're ready to commit and proceed
- [ ] If the user requests changes before committing, the changes are made and
      the commit offer is re-presented (loop until committed)
- [ ] Commits only happen when the phase represents an atomic, working state
      (no partial or broken commits)
- [ ] The commit skill's existing branch-check logic is used as-is (handles
      both main-branch and feature-branch scenarios)
- [ ] Both `implement-plan` and `implement-plan-worktree` skills gain this
      behavior via the shared `implementation-workflow.md` fragment
- [ ] The commit step is clearly documented in the workflow so agents understand
      when and why it happens

## Non-Goals (Out of Scope)

- Changing the commit skill itself — it already has the needed functionality
- Auto-pushing to remote after commits — that's a separate concern
- Committing partial/incomplete phases — the phasing already ensures atomicity
- Adding commit behavior to other skills beyond implement-plan variants

## Proposed Solution

Integrate the commit step into the shared `implementation-workflow.md` fragment,
slotting it into the existing post-phase sequence between code review and the
pause message. The combined pause-and-commit flow would look like:

1. Run success criteria checks, mark checkboxes
2. Spawn code reviewer agent, fix blockers
3. **Check for uncommitted changes** (`git status --porcelain`)
4. **If changes exist**: combine the verification pause with a commit prompt —
   present the phase completion summary and ask if the user is ready to commit
   and proceed
5. **If user requests changes**: make fixes, then re-offer the commit
6. **On approval**: invoke the commit skill's workflow (review diff, draft
   message, present plan, execute)
7. Proceed to next phase

Since both `implement-plan` and `implement-plan-worktree` include
`implementation-workflow.md` via `{% include %}`, the change propagates to both
skills automatically.

## Constraints

- The commit behavior lives in the shared fragment, not duplicated in each
  skill file
- Must not break the existing pause-for-verification flow — the commit is an
  addition, not a replacement
- The commit skill's full workflow (including user approval of the commit
  message) must be followed — no silent/automatic commits

## Resolved Questions

- **Opt-out support**: Yes — the user can skip the commit at the pause point
  and let changes accumulate across phases. This supports cases where phases are
  tightly coupled and a single commit makes more sense.
- **Consecutive phase mode**: When the user instructs consecutive phase
  execution (skipping pauses until the last one), commits are also deferred to
  the final pause. This is consistent with the skip-pause intent — if you're
  running phases back-to-back, you don't want to stop for commits either.

## Affected Systems/Services

- `agent-config/spec/fragments/plans/implementation-workflow.md` — primary
  change location
- `agent-config/spec/skills/implement-plan/SKILL.md` — inherits via include
- `agent-config/spec/skills/implement-plan-worktree/SKILL.md` — inherits via
  include
- `agent-config/spec/skills/commit/SKILL.md` — referenced but not modified
