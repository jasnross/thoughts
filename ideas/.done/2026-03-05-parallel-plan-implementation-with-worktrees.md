# Parallel Plan Implementation with Git Worktrees

## Problem Statement

When working on multiple features or plans simultaneously in the same repo, the current approach is to clone the repo multiple times and run separate Claude Code instances in each clone. This works but causes friction: duplicated disk space, clones drifting out of sync with each other and upstream, and mental overhead tracking which clone has what work.

The `implement-plan` skill currently assumes a single serial workflow — one plan, one branch, one working directory. It has no awareness of parallel execution or worktree isolation, even though Claude Code now has first-class support for both.

## Motivation

- **Parallel productivity**: Working on 2-3 independent efforts simultaneously is a common pattern — e.g., a feature on repo A while a bugfix lands on repo B, or two independent features on the same repo.
- **Reduced friction**: Git worktrees share the object store, refs, and config. No more `git fetch` in 3 different clones to stay current. No more 2 GB of duplicated `.git` directories.
- **Claude Code already supports this**: The `--worktree` CLI flag and `isolation: "worktree"` subagent parameter exist today but the implement-plan skill doesn't leverage them.
- **Better coordination**: Worktrees share refs instantly — a branch created in one is visible in all others. This makes cross-plan awareness possible without pushing to a remote.

## Context

### Current State

The `implement-plan` skill (`agent-config/spec/skills/implement-plan/SKILL.md`) operates as a single-threaded workflow:

1. Read a plan from `thoughts/plans/`
2. Implement phases sequentially
3. Pause for human verification between phases
4. Track progress via checkboxes in the plan file

For parallel work, the user manually clones the repo N times and runs `claude` in each clone independently. There is no shared coordination, no awareness between agents, and no tooling to manage the lifecycle.

### Claude Code's Worktree Capabilities

Claude Code provides three mechanisms for parallel execution:

1. **`claude --worktree <name>` (or `-w`)**: Launches a full interactive session in an isolated worktree at `.claude/worktrees/<name>`. Creates a branch `worktree-<name>` from the default remote branch. On exit, auto-cleans if no changes were made; prompts to keep/remove if changes exist.

2. **`isolation: "worktree"` on subagents**: When dispatching a sub-agent via the Task/Agent tool, setting `isolation: "worktree"` gives that agent its own worktree automatically. The worktree is cleaned up when the agent finishes (if no changes). This enables a single orchestrating agent to farm out parallel work.

3. **`WorktreeCreate` / `WorktreeRemove` hooks**: Customizable hooks that fire during worktree lifecycle events. Can be used to integrate with non-git VCS, set up dependencies, or perform custom cleanup.

### Git Worktree Model

- Worktrees share: object store, refs/branches/tags, stash, config, hooks
- Worktrees isolate: working directory, index (staging area), HEAD, in-progress operations (merge, rebase, cherry-pick)
- Key constraint: **a branch can only be checked out in one worktree at a time** — git enforces this to prevent ref conflicts
- Cleanup: `git worktree remove <path>` or `git worktree prune` for stale entries

## Goals / Success Criteria

- [ ] The implement-plan skill can orchestrate parallel implementation of multiple plans (or multiple phases of one plan) using worktree-isolated sub-agents
- [ ] A user can launch separate `claude --worktree` sessions to implement different plans independently, with each session aware it's in a worktree context
- [ ] The skill handles worktree lifecycle: creation, branch naming, cleanup, and merge/PR workflow
- [ ] Plan files remain the source of truth for progress tracking, even when work happens in worktrees
- [ ] The approach works for the common case of 2-3 parallel efforts with occasional file overlap

## Non-Goals (Out of Scope)

- Building a full multi-agent orchestration framework — leverage what Claude Code already provides
- Solving merge conflicts automatically — surface them, let the human decide
- Supporting non-git VCS (SVN, Perforce) — git worktrees only
- Replacing the existing serial implement-plan workflow — this should be additive, not a rewrite
- Agent teams / inter-agent communication protocols — that's a separate, larger effort

## Proposed Solutions (Optional)

### Option A: Worktree-Aware Implement-Plan (Separate Sessions)

Update implement-plan to detect when it's running inside a worktree (`git rev-parse --git-common-dir` differs from `--git-dir`) and adapt its behavior:

- Use the worktree branch name for commits
- Know that the "main" working tree may have other work in progress
- Offer to create a PR from the worktree branch when a plan is complete

The user launches parallel sessions manually via `claude -w plan-a` and `claude -w plan-b`.

**Pros**: Simple, leverages existing Claude Code features, user controls parallelism directly.
**Cons**: No orchestration — each session is fully independent, user manages coordination.

### Option B: Orchestrated Parallel Plans (Single Session)

Add a new mode or companion skill where a single Claude Code session can dispatch multiple plan implementations as worktree-isolated sub-agents:

```
Main agent reads plans A, B, C
├── Task(isolation="worktree") → implement plan A
├── Task(isolation="worktree") → implement plan B
└── Task(isolation="worktree") → implement plan C
```

The main agent coordinates, reviews results, and merges.

**Pros**: Centralized coordination, can review cross-plan interactions, single point of control.
**Cons**: More complex, sub-agent context limits may constrain large plans, verification pauses are harder to manage across parallel agents.

### Option C: Both (Layered Approach)

Option A as the foundation (worktree-awareness in implement-plan), with Option B as an optional orchestration layer for when you want single-session control.

**Pros**: Flexible, incremental, covers both use cases.
**Cons**: More surface area to maintain.

## Constraints

- Must work with the existing `thoughts/plans/` convention and plan file format
- Branch naming must follow the `<username>/<description>` convention from git rules
- Worktree branches must be unique per worktree (git enforces this)
- Sub-agents dispatched with `isolation: "worktree"` have limited context windows — large plans may need to be chunked
- The skill must remain compatible with non-worktree usage (backward compatible)

## Open Questions

- How should plan file updates work when the plan file lives in the main worktree but implementation happens in a sub-worktree? Should the plan file be committed in the worktree branch, or updated in the main tree?
- Should worktree branch names encode the plan name (e.g., `jaross/implement-plan-a`) for traceability?
- How do verification pauses work in the orchestrated model (Option B)? Does the main agent pause all sub-agents, or let them continue independently?
- What happens when two parallel plans need to modify the same file? Should the skill detect this upfront and warn, or handle it at merge time?
- Should `.claude/worktrees/` be added to `.gitignore` as part of project setup, or is that a per-project decision?
- Is the sub-agent context window sufficient for implementing full plan phases, or do we need a chunking strategy?

## References

- Claude Code worktree docs: `claude --worktree` CLI flag, `isolation: "worktree"` subagent parameter
- Claude Code hooks: `WorktreeCreate` and `WorktreeRemove` hook events for custom worktree lifecycle
- Git worktree docs: https://git-scm.com/docs/git-worktree
- Current implement-plan skill: `agent-config/spec/skills/implement-plan/SKILL.md`

## Affected Systems/Services

- `agent-config/spec/skills/implement-plan/SKILL.md` — primary skill to update
- Potentially a new companion skill for orchestrated parallel execution
- User's `.gitignore` files (to exclude `.claude/worktrees/`)
- Branch naming conventions (must accommodate worktree branch patterns)
