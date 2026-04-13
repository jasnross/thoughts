# Code Review Loop Skill

## Problem Statement

The existing `code-review` skill provides a collaborative triage workflow: it
spawns a code-reviewer agent, presents findings one at a time, and asks the user
"fix now, skip, or defer?" for each. This is thorough but slow — most findings
have obvious fixes that don't require user deliberation. There's no mechanism to
re-review after applying fixes, so newly introduced issues (from the fixes
themselves) go undetected unless the user manually triggers another review.

## Motivation

- **Reduce friction for routine fixes**: Unused imports, missing error handling,
  style issues, and other clear-cut findings should be fixed without stopping to
  ask permission for each one.
- **Catch fix-introduced regressions**: Applying a batch of fixes can introduce
  new issues. A review loop catches these before the user considers the review
  "done."
- **Preserve user control for real decisions**: Design trade-offs, ambiguous
  fixes, and questions that require domain knowledge should still surface for
  collaborative discussion — just not the obvious stuff.
- **Complement, not replace**: The existing `code-review` skill remains
  available for users who prefer fully collaborative triage.

## Context

### Existing Skills

- **`code-review`** — Spawns a `code-reviewer` agent, resolves base branch and
  merge-base, presents findings collaboratively. Each finding goes through
  "fix now / skip / defer?" User controls every change.
- **`review-files`** — Same pattern but scoped to specific files/globs/directories
  rather than branch diffs.

Both skills delegate to the `code-reviewer` subagent and share the same prompt
contract:

```
Review request:
- Project path: <path>
- Scope: <uncommitted changes | changes against <base-ref> | specific files>
- Base reference: <git-ref or "none">
- Merge-base: <commit-sha or "none">
- Focus: <optional subdirectory or comma-separated files>
```

### Architecture

The new skill lives as a spec in `spec/skills/code-review-loop/SKILL.md` with
frontmatter-based invocability. It reuses the `code-reviewer` subagent and the
existing prompt contract — the difference is entirely in the orchestration logic
on the main agent side.

## Goals / Success Criteria

- [ ] Skill is both user-invocable (`/code-review-loop`) and agent-invocable
- [ ] Supports the same scope options as `code-review` (branch diff, uncommitted
      changes, specific files via optional git ref argument)
- [ ] Auto-fixes findings that have unambiguous fixes (regardless of severity)
- [ ] Stops and asks the user when a finding requires a decision (multiple valid
      approaches, design trade-offs, unclear intent)
- [ ] Re-reviews after applying fixes to catch introduced regressions
- [ ] Loops up to 3 passes per checkpoint; prompts user "continue?" at the cap
      (if actionable findings remain, another 3-pass window starts on approval)
- [ ] Presents skipped/deferred items at the end of the loop for collaborative
      triage using the same workflow as `code-review`
- [ ] Questions/Assumptions and Verification Gaps are surfaced for discussion
      (not auto-fixed) but do not block the loop
- [ ] Todos are created for user-decision items mid-loop and for all remaining
      items in the end-of-loop triage; auto-fixed items do not generate todos
- [ ] State written to `.code-review-loop.md` after each finding action; file
      re-read at the start of each pass so loop state survives context compression
- [ ] State file left in place after the loop as a review artifact

## Non-Goals (Out of Scope)

- Replacing the existing `code-review` skill — both coexist
- Modifying the `code-reviewer` subagent's review logic or output format
- Adding new review categories or severity levels
- Auto-committing fixes (the user retains control over git operations)
- File-scoped variant (that's `review-files`; this skill covers branch/diff scope)

## Proposed Approach

### High-Level Flow

```
1. Resolve scope (same logic as code-review: base branch, merge-base, etc.)
2. Initialize state file `.code-review-loop.md` in the project root
   (always overwrite if one exists; notify user if an existing file was replaced)
3. LOOP (max 3 passes per checkpoint):
   a. Re-read state file to reconstruct seen_skipped (guards against context loss)
   b. Spawn code-reviewer agent with prompt contract
   c. Receive structured findings
   d. Filter out any findings already in seen_skipped
   e. For each remaining finding (severity order: Blockers → Major → Minor):
      - If fix is unambiguous → apply fix, append to state file under "Fixed"
        (no todo needed — appears in per-pass summary)
      - If fix requires decision → create a todo, stop, discuss with user:
          * Fix agreed → apply, mark todo complete, append to state file under "Fixed"
          * Skipped/deferred → mark todo complete with note, add to seen_skipped,
            append to state file under "Skipped"
      - If Questions/Assumptions or Verification Gaps → append to state file
        under "Questions"; add to end-of-loop list
   f. Print per-pass summary: "Pass N: fixed X issues — [brief list]"
   g. If fixes were applied → increment pass counter, re-review (goto 3a)
   h. If no fixes applied → exit loop (naturally done)
4. At pass cap (3): prompt user "N issues fixed across K passes. Continue?"
   - Yes → reset counter, continue from 3a
   - No → proceed to step 5
5. Create todos for all remaining items (re-read from state file if needed)
   - Work through collaboratively using code-review's triage workflow
   - Mark each todo complete as it's resolved
6. Final summary: total fixes applied across all passes, items skipped/discussed
   (state file remains as a review artifact for the user)

### State File Format

Written to `.code-review-loop.md` in the project root. Appended after each
finding action; re-read at the start of each pass to reconstruct loop state.

```markdown
## Pass 1

**Fixed:**
- src/api.ts:42 — unused import removed
- src/api.ts:88 — missing null check added

**Skipped (user):**
- src/api.ts:101 — extract helper function (deferred by user)

**Questions (end-of-loop):**
- Should retries use exponential backoff or fixed intervals?

## Pass 2

**Fixed:**
- src/api.ts:77 — error not propagated to caller
```
```

**Note on fix-introduced regressions**: If a fix introduces a new Blocker,
the next pass picks it up naturally — no special-casing or auto-revert logic.
The loop handles regressions the same way it handles any other finding.

### Auto-Fix vs. User Decision Heuristic

**Auto-fix** (unambiguous, single clear fix):
- Unused imports / dead code removal
- Missing error handling with obvious pattern (e.g., unchecked error in Go,
  missing try/catch matching surrounding code style)
- Typos in strings, comments, variable names
- Style/formatting issues
- Missing null/undefined checks with obvious guard pattern
- Simple type fixes

**Stop and ask** (ambiguous, multiple valid approaches):
- Architectural changes
- Performance trade-offs (caching strategy, algorithm choice)
- API design decisions
- Behavior changes that affect external contracts
- Security-sensitive changes where intent matters
- Anything where the reviewer noted multiple possible approaches

## Constraints

- Must reuse the existing `code-reviewer` subagent and prompt contract
  unchanged — no modifications to the reviewer's output format
- Skill ID, filename, and routing must be kebab-case and consistent:
  `code-review-loop`
- Must work across all compat targets: claude, opencode, codex, cursor

## Decisions

| Question | Decision |
|---|---|
| Deduplicate skipped findings across passes? | **Yes** — track seen_skipped; never re-ask mid-loop |
| Configurable pass cap? | **No** — fixed at 3; user can extend by answering "continue?" |
| Show auto-fix summaries per-pass or end-only? | **Per-pass** — brief list after each review pass |
| Fix introduces a new Blocker — revert or continue? | **Continue** — treat it like any other finding in the next pass |
| Create todos for auto-fixed items? | **No** — auto-fixes appear in per-pass summary only; todos for user-decision items and end-of-loop triage only |
| How to survive context compression? | **State file** — `.code-review-loop.md` written after each action, re-read at the start of each pass; left as a review artifact when done |
| Existing state file at startup? | **Always overwrite** — notify user if an existing file was replaced; resume is out of scope |

## References

- Existing skill: `spec/skills/code-review/SKILL.md`
- Existing skill: `spec/skills/review-files/SKILL.md`
- Code-reviewer agent: `code-reviewer` subagent type
- Naming convention: `agent-docs/canonical-command-conventions.md`
