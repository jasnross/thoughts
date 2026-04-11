# Implement-Plan-Worktree Skill Implementation Plan

## Overview

Create a new `implement-plan-worktree` skill that wraps the existing implement-plan
workflow with git worktree lifecycle management. This enables parallel plan
implementation from `~/Workspace/` — each Claude session gets an isolated branch
and worktree inside the target project, while `thoughts/plans/` files are shared
and accessed via absolute paths. The existing `implement-plan` skill is untouched.

## Current State Analysis

- `implement-plan` (`agent-config/spec/skills/implement-plan/SKILL.md`) is serial
  and single-worktree. It cd's into a project, implements phases, and tracks
  progress via checkboxes. No worktree awareness.
- `split-branch` (`agent-config/spec/skills/split-branch/SKILL.md`) is the primary
  worktree pattern reference: it creates worktrees with `git worktree add`, dispatches
  subagents per worktree, and cleans up with `git worktree remove` + `git worktree
  prune`. The full lifecycle is proven here.
- Skill compilation: `agent-config/scripts/` compiles `spec/skills/<id>/SKILL.md`
  to platform-specific outputs via `npm --prefix agent-config run build:agents`
  (= validate + compile + check).
- Plans embed absolute project paths inline (e.g., `/Users/jaross/Workspace/ops/bedrock-access-gateway/`).
- `thoughts/` is a separate git repo at `~/Workspace/thoughts/` — plan file updates
  use absolute paths, independent of project worktrees.
- `agent-docs-global/git-worktrees.md`: "never cd into worktree paths for transient
  ops" — this skill permanently lives in the worktree (one cd in, one cd out at
  cleanup) so the warning does not apply.
- `~/Workspace/` is not a git repo; `claude -w` is not viable from there.

## Desired End State

A user can run two concurrent Claude sessions from `~/Workspace/`, each implementing
a different plan (including plans targeting the same project), without branch conflicts:

```
Session A: /tw:implement-plan-worktree thoughts/plans/plan-a.md
Session B: /tw:implement-plan-worktree thoughts/plans/plan-b.md
```

Each session:
1. Detects the target project from the plan content, confirms with the user
2. Creates an isolated branch (`<username>/implement-<plan-slug>`) and worktree
   (`<project>/.worktrees/<plan-slug>/`)
3. Implements the plan fully from within that worktree
4. Updates `thoughts/plans/*.md` via absolute paths (shared, unaffected by worktrees)
5. Optionally pushes and creates a PR at completion
6. Cleans up the worktree after the PR is created (or keeps it on request)

### Key Discoveries:

- `split-branch/SKILL.md:332-366` — the authoritative worktree add/operate pattern
- `split-branch/SKILL.md:500-512` — cleanup: `git worktree remove` then `git worktree prune`
- `implement-plan/SKILL.md:38-116` — full implementation philosophy to carry forward
- `agent-config/CLAUDE.md` — must run `npm --prefix agent-config run build:agents`
  after spec changes; do not hand-edit `generated/`
- `agent-config/spec/schema/canonical.schema.json` — required frontmatter fields:
  `id`, `description`, `version`
- Branch naming: `<username>/implement-<plan-slug>` per `tw-git-conventions.md`
- `agent-docs-global/git-worktrees.md` — never use `git -C <worktree-path>` after
  cd'ing into the worktree; run plain `git` commands (working dir persists between
  Bash tool invocations)

## What We're NOT Doing

- Not modifying `implement-plan` — zero risk to the working skill
- Not implementing orchestrated parallel plans in a single session (Option B from the
  idea file) — that is a separate future effort
- Not adding worktree metadata to plan files or changes to `create-plan`
- Not supporting `claude -w` (requires being inside a git repo; `~/Workspace/` is not)
- Not auto-resolving merge conflicts between parallel worktrees
- Not automatically detecting when parallelism is needed — the user explicitly chooses
  this skill

## Implementation Approach

Create one new spec file: `agent-config/spec/skills/implement-plan-worktree/SKILL.md`.
Then rebuild. The skill body reuses the full implement-plan philosophy (same
verification approach, same checkbox protocol, same code review step) and adds a
worktree lifecycle wrapper around it.

---

## Phase 1: Create the `implement-plan-worktree` Skill Spec

### Overview

Write the canonical skill spec. This is the only source-level change required.
The spec body must cover: project detection, pre-flight checks, worktree creation,
the full implementation loop (running inside the worktree), and completion/cleanup.

### Changes Required:

#### 1. Skill Spec File

**File**: `agent-config/spec/skills/implement-plan-worktree/SKILL.md`

- [x] Create the directory `agent-config/spec/skills/implement-plan-worktree/`
- [x] Create `SKILL.md` with the following frontmatter and body:
  > **Deviation:** Pre-flight Checks and Worktree Creation sections were
  > restructured to `cd "$PROJECT_DIR"` first, then use plain `git` commands
  > instead of `git -C "$PROJECT_DIR"`. This aligns with the global
  > `tw-git-conventions.md` rule. `git -C` is retained only in Project
  > Detection for one-shot candidate path verification (before any `cd`).
  > Additionally restored content omitted during initial drafting: code-reviewer
  > example prompts, "Think deeply" bullet, "Keep It Useful" subsection, and
  > normalized arrow characters to unicode.

**Frontmatter:**

```yaml
---
id: implement-plan-worktree
description: >-
  Implement a technical plan in an isolated git worktree, enabling safe parallel
  plan execution without branch conflicts.
version: 1
user_invocable: true
agent_invocable: false
execution:
  model_profile: deep_review
capabilities:
  tools:
    - bash
    - edit
    - glob
    - grep
    - ls
    - read
    - task
    - todowrite
    - webfetch
    - websearch
skill:
  accepts_args: true
  args_schema: string
compat:
  targets:
    - claude
    - opencode
    - codex
    - cursor
---
```

**Skill body must cover the following sections:**

---

##### Section: Getting Started

Require a plan path argument. If none provided, ask for one.

Read the plan completely before doing anything else. Use the absolute path as
given (plan files live in `~/Workspace/thoughts/plans/` and must always be
accessed via their absolute path — the worktree working directory is inside
the project, not in `thoughts/`).

---

##### Section: Project Detection

Parse the plan body to identify the target project directory:

1. Scan for absolute paths that resolve to git repositories:
   - Extract candidate paths (e.g., `/Users/.../Workspace/<subdir>/`)
   - For each candidate: `git -C <path> rev-parse --git-dir 2>/dev/null`
2. Present results to the user:
   - If exactly one valid git repo path is found: show it and ask for confirmation
   - If multiple: list them, ask user to select
   - If none: ask user to provide the project path explicitly
3. Store confirmed path as `$PROJECT_DIR`

Derive additional values:

```bash
# Plan slug: strip date prefix and .md extension from the plan filename
# e.g. "2026-03-05-espio-calver-releases.md" -> "espio-calver-releases"
PLAN_SLUG=$(basename <plan-path> .md | sed 's/^[0-9]\{4\}-[0-9]\{2\}-[0-9]\{2\}-//')

USERNAME=$(whoami)
BRANCH_NAME="$USERNAME/implement-$PLAN_SLUG"
WORKTREE_PATH="$PROJECT_DIR/.worktrees/$PLAN_SLUG"
```

---

##### Section: Pre-flight Checks

Run checks in `$PROJECT_DIR` before creating anything:

```bash
# Verify it is a git repo
git -C "$PROJECT_DIR" rev-parse --git-dir

# Check if the branch already exists (would indicate a prior session)
git -C "$PROJECT_DIR" branch --list "$BRANCH_NAME"

# Check for uncommitted changes (user may want to stash or commit first)
git -C "$PROJECT_DIR" status --porcelain
```

If the branch already exists, pause and ask the user:
- Resume implementation in the existing worktree (if present), or
- Delete the branch and start fresh, or
- Abort

If uncommitted changes exist, warn the user but do not block.

Display a pre-flight summary and ask for confirmation before proceeding:

```
Worktree Setup

Plan:        <plan-slug>
Project:     $PROJECT_DIR
Branch:      $BRANCH_NAME
Worktree:    $WORKTREE_PATH

Proceed?
```

---

##### Section: Worktree Creation

```bash
# 1. Update .gitignore in the project (prevents .worktrees/ from appearing as
#    untracked in git status). This is done in the main working tree and is
#    NOT committed as part of the implementation branch — tell the user they
#    can commit it separately at any time.
grep -qxF '.worktrees/' "$PROJECT_DIR/.gitignore" 2>/dev/null \
  || echo '.worktrees/' >> "$PROJECT_DIR/.gitignore"

# 2. Fetch and resolve the default branch
git -C "$PROJECT_DIR" fetch origin
DEFAULT_BRANCH=$(git -C "$PROJECT_DIR" symbolic-ref refs/remotes/origin/HEAD \
  | sed 's@^refs/remotes/origin/@@')

# 3. Create branch and worktree from the remote default branch
git -C "$PROJECT_DIR" worktree add "$WORKTREE_PATH" \
  -b "$BRANCH_NAME" "origin/$DEFAULT_BRANCH"
```

Confirm success, then cd into the worktree (one Bash call):

```bash
cd "$WORKTREE_PATH"
```

After this cd, all git commands run as plain `git` (no `-C` flags) — the
working directory persists between Bash tool invocations.

---

##### Section: Implementation (inside worktree)

From this point, the skill follows the complete implement-plan workflow:

- Read the plan completely (via its absolute path — NOT relative to the worktree)
- Check for existing checkmarks to identify the resume point
- Read all files and tickets mentioned in the plan
- Create a todo list to track progress
- Implement each phase fully before moving to the next
- Adapt to what is found in the codebase; the plan is a guide, not a contract

After each phase:

1. Run all automated success criteria from the plan
2. Check off completed `- [ ]` checkboxes in the plan file (via absolute path)
3. Spawn a code-reviewer subagent:
   - Use `subagent_type: "code-reviewer"` via the Task tool
   - Provide project path (the worktree path), scope, and base reference
   - Fix all blockers immediately
   - Surface major/minor findings to the user in the pause message
4. Pause for human manual verification using this format:

```
Phase [N] Complete - Ready for Manual Verification

Automated verification passed:
- [List automated checks that passed]

Code review completed:
- Blockers fixed: [N] (or "none found")
- Major issues: [N] (or "none found")
- Minor issues: [N] (or "none found")
[If major/minor issues exist, include them with file:line references]

Please perform the manual verification steps listed in the plan:
- [List manual verification items from the plan]

Let me know when manual testing is complete so I can proceed to Phase [N+1].
```

When things don't match the plan, stop and present clearly:

```
Issue in Phase [N]:
Expected: [what the plan says]
Found: [actual situation]
Why this matters: [explanation]

How should I proceed?
```

Plan update protocol — after each phase, before pausing:
- Check off each completed `- [ ]` → `- [x]`
- If implementation diverged from the plan, add a deviation note:

  ```markdown
  - [x] Add retry logic to `src/api/client.ts`
    > **Deviation:** Implemented in `src/api/http.ts` instead — `client.ts` was
    > refactored into `http.ts` since the plan was written.
  ```

Do not check off manual verification steps unless the user confirms they passed.

---

##### Section: Completion and Cleanup

When all phases are complete, present options:

```
Implementation Complete

Branch:   $BRANCH_NAME
Worktree: $WORKTREE_PATH

Next steps:
1. Push branch and create PR (recommended)
2. Push branch only (no PR yet)
3. Keep local only — no push

What would you like to do?
```

If pushing: push from within the worktree using `git push -u origin $BRANCH_NAME`.
For PR creation: load the `gh-safe` skill and follow its syntax. Use draft PRs by
default unless the user specifies otherwise.

After the push/PR decision, offer cleanup:

```
Worktree cleanup:
1. Remove worktree now (recommended after PR is created)
2. Keep worktree for further work
```

If removing, cd out first, then clean up:

```bash
cd "$PROJECT_DIR"
git worktree remove "$WORKTREE_PATH"
git worktree prune
```

---

##### Section: Resuming Work

If the plan already has checkmarks and the branch/worktree exist:

```bash
# Re-enter the worktree
cd "$WORKTREE_PATH"
```

Trust completed work. Pick up from the first unchecked `- [ ]` item.
Verify previous work only if something looks wrong.

---

##### Section: Important Notes

- **Plan files must be accessed via absolute paths** — the worktree working
  directory is inside the project, not in `thoughts/`. Always use the full
  absolute path when reading or updating the plan file.
- **Never use `git -C` after cd'ing into the worktree** — plain `git` commands
  run in the correct context. `git -C` causes each command to require individual
  approval.
- **If something fails mid-worktree**, do not try to clean up automatically.
  Report the error, cd back to `$PROJECT_DIR`, and let the user decide whether
  to fix or remove the worktree manually.
- **The `.gitignore` addition** (`.worktrees/`) is uncommitted in the main working
  tree. Mention it to the user once; they can commit it to the project at any time.

---

### Success Criteria:

#### Automated Verification:

- [x] Skill spec file exists: `agent-config/spec/skills/implement-plan-worktree/SKILL.md`
- [x] Schema validation passes: `npm --prefix agent-config run validate:agents`
- [x] Full build passes: `npm --prefix agent-config run build:agents`
- [x] Generated Claude skill exists and is non-empty:
  `agent-config/generated/claude/skills/implement-plan-worktree/SKILL.md`
- [x] Generated opencode skill exists:
  `agent-config/generated/opencode/skills/implement-plan-worktree/SKILL.md`
- [x] Generated codex skill exists:
  `agent-config/generated/codex/skills/implement-plan-worktree/SKILL.md`
- [x] Generated cursor skill exists:
  `agent-config/generated/cursor/skills/implement-plan-worktree/SKILL.md`
- [x] No validation warnings in build output

#### Manual Verification:

- [ ] Invoke `/tw:implement-plan-worktree` against a real plan — pre-flight summary
  is displayed with correct project path, branch name, and worktree path
- [ ] Worktree is created at `<project>/.worktrees/<plan-slug>/`
- [x] `.worktrees/` is appended to `.gitignore` in the project root
  > **Deviation:** Removed — the skill no longer auto-updates `.gitignore`. Any repo using worktrees is assumed to have its own ignore configuration.
- [ ] Branch is named `<username>/implement-<plan-slug>`
- [ ] Plan checkbox updates write to the correct absolute path in `thoughts/plans/`
  (not a copy inside the worktree)
- [ ] Parallel invocation: two sessions, same project, different plans — each gets
  its own branch and worktree with no conflicts

**Implementation Note**: After Phase 1 passes all automated verification, pause here
for manual confirmation that the worktree behavior is correct before proceeding.

---

## Phase 2: Build, Verify, and Distribute

### Overview

Compile the new skill spec into platform-specific generated files, review the
outputs, and confirm the skill is reachable in Claude Code.

### Changes Required:

#### 1. Run the Full Build

- [ ] Run: `npm --prefix agent-config run build:agents`
- [ ] Confirm exit code 0
- [ ] Confirm no errors or unexpected warnings

#### 2. Review the Generated Claude Output

**File**: `agent-config/generated/claude/skills/implement-plan-worktree/SKILL.md`

- [ ] Frontmatter contains `name`, `description`, `allowed-tools`, and `model`
- [ ] `disable-model-invocation` is absent (skill is user-invocable)
- [ ] Body matches the spec body
- [ ] Spot-check one additional platform output (opencode or codex) for consistent
  structure

#### 3. Check the Distribution Path

- [ ] Inspect the dotfiles install/link mechanism (check `Makefile`, install scripts,
  or symlinks in `~/.claude/`) to understand how generated skills reach Claude Code
- [ ] Confirm the new skill is accessible after any required install step
- [ ] If a manual copy or symlink step is needed, run it and verify

### Success Criteria:

#### Automated Verification:

- [ ] `npm --prefix agent-config run build:agents` exits 0 with no errors
- [ ] All four platform-specific generated files are present and non-empty
- [ ] `git diff --name-only` shows only the new spec file and new generated files
  (no unintended changes to existing skills)

#### Manual Verification:

- [ ] Skill appears in the Claude Code skill list after the install step
- [ ] `/tw:implement-plan-worktree` is invokable and displays its getting-started
  message without error

---

## References

- Skill to model: `agent-config/spec/skills/implement-plan/SKILL.md`
- Worktree pattern: `agent-config/spec/skills/split-branch/SKILL.md:325-510`
- Git conventions (no `git -C`, cd separately): `~/Workspace/thoughts/plugin/rules/tw-git-conventions.md`
- Worktree gotchas: `agent-docs-global/git-worktrees.md`
- Schema: `agent-config/spec/schema/canonical.schema.json`
- Build command: `npm --prefix agent-config run build:agents` (per `agent-config/CLAUDE.md`)
- PR creation: `agent-config/spec/skills/gh-safe/SKILL.md`
