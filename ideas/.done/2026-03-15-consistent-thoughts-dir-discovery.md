# Consistent `thoughts/` Directory Discovery Across Skills

## Problem Statement

Skills that reference the `thoughts/` directory each implement their own ad-hoc path
resolution strategy. `split-branch` walks up the directory tree looking for a git-backed
`thoughts/` repo. `implement-plan-worktree` hardcodes `~/Workspace/thoughts/`. Skills like
`thoughts-locator` and `thoughts-analyzer` assume the path is already known and do no
discovery at all. `create-idea` uses `./thoughts/` relative to CWD.

This inconsistency causes skills to silently resolve to different directories — or break
entirely — depending on where the user invokes them, whether they're inside a git worktree,
or whether a `thoughts/` directory exists at all.

## Motivation

- A single, canonical discovery mechanism means skills behave predictably regardless of
  invocation context.
- Worktree safety requires absolute paths; CWD-relative resolution is a latent bug in any
  skill that may run inside a worktree.
- Users shouldn't need to remember that one skill requires a specific CWD while another
  relies on a hardcoded path.
- A shared resolution algorithm makes it easy to add new skills that interact with
  `thoughts/` without re-inventing or copy-pasting discovery logic.

## Context

Two real usage patterns exist:

- **Workspace-level**: A single `thoughts/` repo lives alongside all project repos (e.g.,
  `~/Workspace/thoughts/`). Skills invoked from any project within the workspace can use it.
- **Project-local**: A `thoughts/` directory is scoped to a single repo (e.g.,
  `agent-config/thoughts/`). Skills invoked from within that repo should prefer it.

Both patterns can coexist in the same directory hierarchy. In practice, the nearest
`thoughts/` directory tends to be the right one — project-local takes precedence when
inside a project, workspace-level is found naturally when at the workspace root. Skills
also need to handle the case where no `thoughts/` directory exists yet (first-time setup).

When running inside a git worktree, the working directory shifts to the worktree path.
Any `thoughts/` path resolved at skill startup must be an absolute path — relative paths
silently resolve to wrong locations in this context.

## Goals / Success Criteria

- [ ] All skills that reference `thoughts/` use the same discovery mechanism
- [ ] The discovered path is always resolved to an absolute path at discovery time
- [ ] Skills work correctly when invoked from inside a git worktree
- [ ] Discovery behaves consistently whether `thoughts/` is workspace-level or project-local
- [ ] The mechanism handles the case where no `thoughts/` directory exists yet
- [ ] Zero additional configuration required for the common case

## Non-Goals (Out of Scope)

- Supporting multiple simultaneous `thoughts/` directories per invocation (skills pick one)
- Syncing or merging content across multiple `thoughts/` repos
- Changing the internal structure of `thoughts/` subdirectories (ideas/, plans/, etc.)
- Aggregating search results across child-project `thoughts/` directories (separate concern)

## Proposed Solutions

### Recommended: Directory Traversal + Env Var Override

Check `$THOUGHTS_DIR` first; if unset, walk up from CWD looking for a `thoughts/`
directory. Use the first one found (closest to workspace root wins). Resolve to an
absolute path immediately.

**Discovery algorithm:**
1. If `$THOUGHTS_DIR` is set, use it (already absolute) — skip traversal
2. Otherwise, start at CWD resolved to an absolute path
3. Walk up the tree; at each level check for a `thoughts/` directory
4. Use the first match as `$THOUGHTS_DIR` — proximity is the deciding factor
5. If no match found, apply the fallback behavior (see Decisions below)

**Note on git-backing (2026-03-18):** An earlier version of this algorithm
preferred git-backed `thoughts/` directories over closer non-git ones. This
caused misresolution when a project-local `thoughts/` (no `.git`) coexisted with
a workspace-level git-backed `thoughts/` higher in the tree — the algorithm
skipped the correct project-local directory. Git-backing is no longer a selection
factor; proximity alone determines the winner.

**Pros:**
- Truly zero-config for users who follow conventional directory layouts
- Env var override handles edge cases (CI, unusual layouts, per-session overrides) without
  complicating the common path
- Project-local `thoughts/` always wins when working inside a project, workspace-level
  is found naturally when at the workspace root

**Cons:**
- No explicit anchor means renaming or moving a `thoughts/` dir silently changes resolution

### Deferred: Marker File (`.thoughtsroot`)

If the traversal approach proves too ambiguous in practice, a `.thoughtsroot` marker file
could be introduced as an opt-in anchor. Skills would walk up looking for `.thoughtsroot`
instead of (or in addition to) a bare `thoughts/` directory. This adds one-time setup
friction but eliminates all ambiguity. Revisit if traversal causes real confusion.

## Constraints

- Discovery logic must be expressible in the natural language instructions of a skill
  (skills are LLM prompts, not shell scripts) — overly complex algorithms are hard to
  specify precisely
- Must not break existing skills that already work for users who have conventional layouts
- Solution must handle first-time setup gracefully (no `thoughts/` directory exists yet)

## Open Questions

Most resolved — one remaining open question below.

### Open Question

**How is "current working directory" defined across agents?**

The discovery algorithm walks up from CWD, but CWD has different meanings depending on
the agent and how it navigates during a session:

- The intended CWD for discovery is the **directory where the session was launched**
  (e.g., where the user opened Claude Code / OpenCode / Codex / Cursor), not whatever
  directory the agent happens to be in at a given moment mid-session.
- Agents may `cd` into worktrees, subdirectories, or sibling repos during a session —
  if CWD is sampled at the wrong moment, traversal resolves to the wrong root.

Questions to answer before planning:
- Does each target agent (Claude Code, OpenCode, Codex, Cursor) expose a reliable way to
  read the **session launch directory** (e.g., an env var like `$CLAUDE_PROJECT_DIR` or
  `$INIT_CWD`) distinct from the current shell CWD?
- If not, should the fragment instruct the agent to capture CWD at the very start of
  execution (before any navigation) and treat that as the stable root?
- Is the behavior consistent enough across all four targets to express in a single fragment,
  or will per-target conditionals be needed?

### Decisions

**When no `thoughts/` is found and `$THOUGHTS_DIR` is unset:**
- *Write skills* (e.g., `create-idea`, `split-branch`): Inspect the directory tree and
  surface candidate locations (sibling `thoughts/` repos, any `thoughts/` dirs found
  while traversing, and `./thoughts/` as a fresh option). Present these as a numbered list
  and allow the user to pick or type a custom path. Proceed with the chosen path.
- *Read-only skills* (e.g., `thoughts-locator`, `thoughts-analyzer`): Fail immediately with
  a clear error listing what paths were searched and how to fix it (create a `thoughts/`
  directory or set `$THOUGHTS_DIR`).

**Subagent discovery:** Calling skills resolve `$THOUGHTS_DIR` to an absolute path first
and pass it explicitly to subagents. Subagents do not implement their own discovery. This
prevents a subagent from resolving to a different `thoughts/` than its caller if CWD differs.

**Shared fragment:** Create `spec/fragments/thoughts/discover-dir.md` implementing the
full algorithm (env var check → walk up for `thoughts/` → fallback prompt/error). All
relevant skills include it via `{{> thoughts/discover-dir}}`. A `{{#if writeable}}`
conditional handles the write-vs-read-only prompt difference within the single fragment.

## References

- `spec/skills/split-branch/SKILL.md` — most complete existing discovery implementation
- `spec/skills/implement-plan-worktree/SKILL.md` — example of hardcoded path
- `spec/agents/thoughts-locator.md` — assumes path is pre-known
- `spec/agents/thoughts-analyzer.md` — assumes path is pre-known
- `spec/skills/create-idea/SKILL.md` — uses `./thoughts/` relative to CWD

## Affected Systems/Services

All skills and agents that reference `thoughts/` — every one currently uses a CWD-relative
path except `implement-plan-worktree` (hardcoded `~/Workspace/thoughts/`).

| Skill/Agent | Read/Write | Subdirs |
|---|---|---|
| `spec/skills/create-idea/` | Write | `ideas/` |
| `spec/skills/create-plan/` | Write | `plans/` |
| `spec/skills/create-learning-plan/` | Write | `plans/` |
| `spec/skills/create-pr/` | Write | `prs/$REPO/` |
| `spec/skills/describe-pr/` | Write | `prs/$REPO/` |
| `spec/skills/research-web/` | Write | `research/` |
| `spec/skills/research-codebase/` | Write | `research/` |
| `spec/skills/iterate-research/` | Write | `research/` |
| `spec/skills/split-branch/` | Read + Write | `plans/`, `splits/` |
| `spec/skills/implement-plan/` | Read | `plans/` |
| `spec/skills/implement-plan-worktree/` | Read | `plans/` |
| `spec/skills/iterate-plan/` | Read | `plans/` |
| `spec/skills/follow-learning-plan/` | Read | `plans/` |
| `spec/skills/validate-plan/` | Read | `plans/` |
| `spec/agents/thoughts-locator/` | Read | all |
| `spec/agents/thoughts-analyzer/` | Read | all |
