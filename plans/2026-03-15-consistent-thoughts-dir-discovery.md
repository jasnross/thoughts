# Consistent `thoughts/` Directory Discovery Implementation Plan

## Overview

Replace all ad-hoc `thoughts/` path resolution in skills and agents with a single shared
Handlebars fragment (`thoughts/discover-dir`) that implements the canonical discovery
algorithm: env var check → walk up from workspace root → use closest `thoughts/` →
fallback prompt (write skills) or error (read skills).

## Current State Analysis

Sixteen skills and two agents reference `thoughts/` using six different strategies:

| Approach | Who Uses It |
|---|---|
| CWD-relative `./thoughts/` | create-idea, create-plan, create-learning-plan, create-pr, describe-pr, research-web, research-codebase, iterate-research, implement-plan, iterate-plan, follow-learning-plan |
| Walk-up traversal (partial) | split-branch (most complete, but walks from `$PROJECT_DIR` not workspace root) |
| Hardcoded `~/Workspace/thoughts/` | implement-plan-worktree |
| Reverse-derivation from plan path | validate-plan |
| Bare `thoughts/` (no discovery) | thoughts-locator |
| No discovery (caller provides path) | thoughts-analyzer (intentional — leave as-is) |

This causes silent misresolution when invoked from worktrees, sibling directories, or
non-conventional layouts.

## Desired End State

All skills and agents use `$THOUGHTS_DIR` (an absolute path resolved by the shared
fragment) to reference `thoughts/`. A user can override discovery by setting `$THOUGHTS_DIR`
in their environment. Behavior is consistent across Claude Code, OpenCode, Cursor, and Codex.

**Verification:**
- No skill or agent body references `./thoughts/`, `~/Workspace/thoughts/`, or any other
  hard-coded path
- All affected skills include `{{> thoughts/discover-dir}}` or
  `{{> thoughts/discover-dir writeable=true}}`
- `npm --prefix agent-config run build:agents` passes cleanly

### Key Discoveries

- **Fragment auto-registration**: `spec/fragments/thoughts/discover-dir.md` → partial name
  `thoughts/discover-dir` (build pipeline registers all `.md` files under `spec/fragments/`
  automatically via `scripts/lib/fragments.ts:23-47`)
- **Partial indentation gotcha**: fragment files must have no leading indentation on any
  line — Handlebars prepends the calling line's indent to every output line
  (`agent-docs/compiler-fragments.md`)
- **No cross-provider env var for workspace root**: all 4 providers (Claude Code, OpenCode,
  Cursor, Codex) communicate workspace root as prose in the LLM system prompt, not as a
  shell env var. `$CLAUDE_PROJECT_DIR` is only available in hooks/subprocesses — it is
  empty inside Task subagents (issue #26429). The fragment references "the Working
  directory from your session context" (provider-neutral)
- **Session-level persistence via shell export**: after discovery, the fragment exports
  `THOUGHTS_DIR` to the shell environment. All 4 providers use a persistent bash shell
  within a session, so subsequent skill invocations find `$THOUGHTS_DIR` already set in
  Step 1 and skip re-discovery. Users can also pre-set `$THOUGHTS_DIR` in their shell
  profile for a permanent override
- **split-branch already close**: it walks up from `$PROJECT_DIR` (set to `pwd` early in
  the skill) looking for a git-backed `thoughts/`. The fragment replaces just its
  discovery block (lines 66–72); all other `$THOUGHTS_DIR` references in that skill
  remain unchanged
- **validate-plan uses reverse-derivation**: derives workspace root as
  `parent(parent(plan_file))`. After this plan: workspace root = `parent($THOUGHTS_DIR)`,
  which is simpler and equally correct. The single-project vs. multi-project detection
  logic is unchanged
- **`implementation-workflow.md` `{{#if worktree}}` note**: concerns using the plan
  file's absolute path in worktrees — distinct from `$THOUGHTS_DIR` discovery. Evaluate
  in Phase 4 whether the note needs updating to reference `$THOUGHTS_DIR` or can be removed
- **`sub-agent-catalog.md` `{{#if include_thoughts_agents}}`**: currently no skill
  activates it. Update in Phase 4 to instruct callers to pass `$THOUGHTS_DIR` explicitly
  to those subagents

## What We're NOT Doing

- Supporting multiple simultaneous `thoughts/` directories per skill invocation
- Syncing or merging content across `thoughts/` repos
- Changing the internal structure of `thoughts/` subdirectories (ideas/, plans/, etc.)
- Adding automated behavioral tests for the discovery algorithm (it's natural language prose)
- Updating `thoughts-analyzer` — it intentionally has no discovery; it receives document
  paths from callers

## Implementation Approach

Create the fragment first, then update skills by category (write, then read), then agents
and existing fragments. Run a build after each phase to catch regressions before touching
the next batch.

---

## Phase 1: Create the Discovery Fragment

### Overview

Create `spec/fragments/thoughts/discover-dir.md` implementing the canonical algorithm.
This is the only new file in the plan — all other changes update existing files.

### Changes Required

#### 1. Create `spec/fragments/thoughts/discover-dir.md`

- [x] Create the file at `spec/fragments/thoughts/discover-dir.md`
- [x] No line in the file has leading indentation (partial indentation gotcha)
- [x] File ends with a trailing newline

**Full fragment content** (write exactly this):

~~~markdown
## Discover `$THOUGHTS_DIR`

Before accessing any `thoughts/` content, resolve `$THOUGHTS_DIR` to an absolute path
using this algorithm:

**Step 1 — Check env var override:**
If `$THOUGHTS_DIR` is already set (non-empty), use it as-is — it is already an absolute
path. Skip steps 2–5. (This catches both values the user set before the session and values
exported by earlier skill invocations within this session.)

**Step 2 — Identify workspace root:**
Use the workspace root from your session context — the "Working directory" shown in your
environment information. This is the directory where the session was launched (not the
current shell working directory, which may have shifted during worktree navigation or
subdirectory exploration). Store this as your traversal starting point.

**Step 3 — Walk up the tree:**
Starting from the workspace root, check each directory level for a `thoughts/`
subdirectory. Walk all the way up to the filesystem root if needed. Use the first
`thoughts/` directory found — the one closest to the workspace root is the correct
match. Proximity is the deciding factor; whether the directory is a git repository
does not affect selection.

**Step 4 — Export the result:**
- Resolve the selected path to an absolute path. Store as `$THOUGHTS_DIR`.
- Export it to the shell environment: run `export THOUGHTS_DIR=<absolute-path>` via bash.
  This ensures subsequent skill invocations within the same session find `$THOUGHTS_DIR`
  already set in Step 1 and skip re-discovery entirely.

**Step 5 — If no `thoughts/` directory was found anywhere in the traversal:**
{{#if writeable}}
Identify candidate locations for a new `thoughts/` directory:
- Any sibling directories named `thoughts/` visible near the workspace root
- A fresh `thoughts/` at the workspace root itself

Present the candidates as a numbered list, allow the user to pick one or enter a custom
path, and create the directory if it does not exist. Use the chosen absolute path as
`$THOUGHTS_DIR` and export it: `export THOUGHTS_DIR=<chosen-path>`.
{{else}}
Fail immediately with a clear error:

    Error: Could not locate a thoughts/ directory.

    Searched from: <workspace root from your session context>

    To fix: create a thoughts/ directory at or above your workspace root,
    or set $THOUGHTS_DIR to the correct absolute path before invoking this skill.

{{/if}}
~~~

### Success Criteria

#### Automated Verification

- [x] Build passes with no errors: `npm --prefix agent-config run build:agents`

#### Manual Verification

- [ ] Read `spec/fragments/thoughts/discover-dir.md` — confirm no lines have leading
  indentation and the file ends with a newline
- [ ] Confirm partial name `thoughts/discover-dir` is usable by checking the build output
  for a skill that includes it (validated in Phase 2)

---

## Phase 2: Update Write Skills

### Overview

Replace all ad-hoc `./thoughts/` references in write skills with
`{{> thoughts/discover-dir writeable=true}}`. After discovery runs, every path reference
to `thoughts/` in the skill body changes to use `$THOUGHTS_DIR`.

**Placement rule**: The fragment opens with a `## Discover \`$THOUGHTS_DIR\`` heading, so
it must be placed at the **top level of the skill body** — never inside a numbered list
item or indented block. The `{{> thoughts/discover-dir ...}}` call itself must appear at
**column 0** (no leading spaces or list indentation) in the SKILL.md source. Handlebars
prepends the calling line's indentation to every line of the fragment — an indented call
site would corrupt the heading and all subsequent lines.

**Positioning**: Insert the fragment before the first numbered step or section that
references `thoughts/`. If the skill's first reference to `thoughts/` is inside a numbered
list, place the fragment before that list (i.e., before the `1.` that starts it).

**Restructuring when needed**: Some skills (notably `split-branch`) embed their existing
discovery block inside a numbered step as inline prose. In these cases, pull the discovery
block out entirely and place the fragment include as a standalone top-level section
immediately before the numbered steps begin. The numbered steps themselves do not change;
only the discovery content moves out of them.

**Path substitution rule**: Replace every instance of `./thoughts/`, `thoughts/`, or any
other relative `thoughts` path with `$THOUGHTS_DIR/` (absolute). Remove any instructions
that say "relative to the current working directory" or `mkdir -p ./thoughts/...` — the
fragment's fallback handles directory creation for write skills.

**split-branch exception**: This skill already uses `$THOUGHTS_DIR` throughout. Only
replace the discovery block at lines 66–72 with `{{> thoughts/discover-dir writeable=true}}`
(placed before step 1 at column 0, not inside step 1's prose). The note block at lines
68–72 (explaining absolute path usage after `cd $PROJECT_DIR`) stays in place inside step
1 — it is about referencing `$THOUGHTS_DIR` correctly, not about discovery itself.

### Changes Required

- [x] `spec/skills/create-idea/SKILL.md`
  - [x] Insert `{{> thoughts/discover-dir writeable=true}}` before the save step
  - [x] Replace `./thoughts/ideas/` with `$THOUGHTS_DIR/ideas/`
  - [x] Remove "relative to CWD" language from path instructions

- [x] `spec/skills/create-plan/SKILL.md`
  - [x] Insert `{{> thoughts/discover-dir writeable=true}}` before Step 4 (plan writing)
  - [x] Replace `./thoughts/plans/` with `$THOUGHTS_DIR/plans/`
  - [x] Remove `mkdir -p ./thoughts/plans` instruction and "relative to CWD" language

- [x] `spec/skills/create-learning-plan/SKILL.md`
  - [x] Insert `{{> thoughts/discover-dir writeable=true}}` before plan-writing step
  - [x] Replace `./thoughts/plans/` with `$THOUGHTS_DIR/plans/`
  - [x] Remove "relative to CWD" language

- [x] `spec/skills/create-pr/SKILL.md`
  - [x] Insert `{{> thoughts/discover-dir writeable=true}}` before first `thoughts/` reference
  - [x] Replace `./thoughts/prs/$REPO_NAME/` with `$THOUGHTS_DIR/prs/$REPO_NAME/`

- [x] `spec/skills/describe-pr/SKILL.md`
  - [x] Insert `{{> thoughts/discover-dir writeable=true}}` before first `thoughts/` reference
  - [x] Replace all `./thoughts/prs/{repo}/` references with `$THOUGHTS_DIR/prs/{repo}/`

- [x] `spec/skills/research-web/SKILL.md`
  - [x] Insert `{{> thoughts/discover-dir writeable=true}}` before Step 5 (save step)
  - [x] Replace `./thoughts/research/` with `$THOUGHTS_DIR/research/`
  - [x] Remove `mkdir -p ./thoughts/research` instruction and "relative to CWD" language

- [x] `spec/skills/research-codebase/SKILL.md`
  - [x] Insert `{{> thoughts/discover-dir writeable=true}}` before Step 6 (save step)
  - [x] Replace `./thoughts/research/` with `$THOUGHTS_DIR/research/`
  - [x] Remove `mkdir -p ./thoughts/research` instruction and "relative to CWD" language

- [x] `spec/skills/iterate-research/SKILL.md`
  - [x] Insert `{{> thoughts/discover-dir writeable=true}}` before first `thoughts/` reference
  - [x] Replace `thoughts/research/` with `$THOUGHTS_DIR/research/`

- [x] `spec/skills/split-branch/SKILL.md`
  - [x] Replace the discovery block at lines 66–72 with `{{> thoughts/discover-dir writeable=true}}`
  - [x] Do NOT modify any other `$THOUGHTS_DIR` references in the skill — they are correct as-is

### Success Criteria

#### Automated Verification

- [x] Build passes: `npm --prefix agent-config run build:agents`

#### Manual Verification

- [ ] Spot-check `generated/claude/skills/create-idea/SKILL.md` — discovery prose appears
  before the save step; path references use `$THOUGHTS_DIR/ideas/`
- [ ] Spot-check `generated/claude/skills/split-branch/SKILL.md` — old walk-up block
  replaced by fragment prose; all other `$THOUGHTS_DIR` references intact

---

## Phase 3: Update Read Skills

### Overview

Replace all ad-hoc `thoughts/` path references in read-only skills with
`{{> thoughts/discover-dir}}` (no `writeable` parameter — defaults to the read-only
error behavior on not-found). For `validate-plan`, additionally replace the
reverse-derivation logic with `workspace_root = parent($THOUGHTS_DIR)`.

**Placement rule**: Same as Phase 2 — `{{> thoughts/discover-dir}}` at column 0, before
any step or section that references `thoughts/`, never inside a numbered list item. For
`implement-plan`, the first `thoughts/` reference is in the intro prose (line 35), so
insert the fragment before `## Getting Started` at the top level. For skills where the
first reference is inside a numbered step, pull the fragment out to a standalone section
before that step's list.

### Changes Required

- [x] `spec/skills/implement-plan/SKILL.md`
  - [x] Insert `{{> thoughts/discover-dir}}` before `## Getting Started`
  - [x] Replace `./thoughts/plans/` on line 35 with `$THOUGHTS_DIR/plans/`

- [x] `spec/skills/implement-plan-worktree/SKILL.md`
  - [x] Insert `{{> thoughts/discover-dir}}` before first `thoughts/` reference
  - [x] Replace hardcoded `~/Workspace/thoughts/plans/` (line 43) with `$THOUGHTS_DIR/plans/`
  - [x] Remove the explanation that `~/Workspace/thoughts/plans/` is the canonical location

- [x] `spec/skills/iterate-plan/SKILL.md`
  - [x] Insert `{{> thoughts/discover-dir}}` before first `thoughts/` reference
  - [x] Replace `thoughts/plans/` with `$THOUGHTS_DIR/plans/`

- [x] `spec/skills/follow-learning-plan/SKILL.md`
  - [x] Insert `{{> thoughts/discover-dir}}` before first `thoughts/` reference
  - [x] Replace `./thoughts/plans/` with `$THOUGHTS_DIR/plans/`

- [x] `spec/skills/validate-plan/SKILL.md`
  - [x] Insert `{{> thoughts/discover-dir}}` at the top of `## Initial Setup` (before step 1)
  - [x] Replace Step 3 (lines 48–60) — the reverse-derivation block — with: workspace root
    is `parent($THOUGHTS_DIR)` (the directory containing the `thoughts/` directory). Keep
    the single-project vs. multi-project detection logic unchanged — only the source of
    the root path changes from "parse the plan file path" to "parent of `$THOUGHTS_DIR`"

### Success Criteria

#### Automated Verification

- [x] Build passes: `npm --prefix agent-config run build:agents`

#### Manual Verification

- [ ] Spot-check `generated/claude/skills/implement-plan-worktree/SKILL.md` — confirm
  `~/Workspace/thoughts/` is gone; discovery prose present
- [ ] Spot-check `generated/claude/skills/validate-plan/SKILL.md` — confirm
  reverse-derivation text replaced; workspace root derivation uses `parent($THOUGHTS_DIR)`

---

## Phase 4: Update Agents and Existing Fragments

### Overview

Update `thoughts-locator` to document caller-provided `$THOUGHTS_DIR`. Update
`implementation-workflow.md` and `sub-agent-catalog.md` for consistency with the new model.

### Changes Required

#### 1. `spec/agents/thoughts-locator.md`

- [x] Add a note near the top of the agent spec (before other instructions) stating: the
  caller is responsible for discovering `$THOUGHTS_DIR` and passing it explicitly in the
  subagent prompt. This agent does not run its own discovery.
- [x] Replace any bare `thoughts/` path references with `$THOUGHTS_DIR`

#### 2. `spec/fragments/plans/implementation-workflow.md`

- [x] Review the `{{#if worktree}}` block at line 75:
  `"Always use the plan's absolute path — the worktree working directory is inside the
  project, not in thoughts/."`
- [x] Since `implement-plan-worktree` will now resolve `$THOUGHTS_DIR` as an absolute path
  via the discovery fragment, update this note to reference `$THOUGHTS_DIR` instead of
  the generic `thoughts/` reference (e.g., "Always use the plan's absolute path from
  `$THOUGHTS_DIR/plans/` — ..."), or remove it if it is fully redundant given the
  discovery fragment's output is always absolute

#### 3. `spec/fragments/research/sub-agent-catalog.md`

- [x] In the `{{#if include_thoughts_agents}}` block, add an instruction: before
  dispatching `thoughts-locator` or `thoughts-analyzer`, ensure `$THOUGHTS_DIR` has been
  resolved by the calling skill (via the discovery fragment or env var), then pass
  `$THOUGHTS_DIR` explicitly in the subagent prompt — subagents do not run their own discovery

### Success Criteria

#### Automated Verification

- [x] Build passes: `npm --prefix agent-config run build:agents`

#### Manual Verification

- [ ] Spot-check `generated/claude/agents/thoughts-locator.md` — note about caller-provided
  `$THOUGHTS_DIR` is present; no bare `thoughts/` path references remain
- [ ] Spot-check `generated/claude/skills/implement-plan-worktree/SKILL.md` — confirm
  the `{{#if worktree}}` note in the rendered output is consistent with absolute
  `$THOUGHTS_DIR` paths

---

## Phase 5: Full Build and Final Verification

### Overview

Full build across all providers, grep-based regression check, and cross-provider
spot-check.

### Changes Required

- [x] Run full build: `npm --prefix agent-config run build:agents`
- [x] Grep `generated/` for remaining hard-coded paths (should return no matches from
  skill/agent bodies)
- [x] Spot-check two additional providers beyond Claude

### Success Criteria

#### Automated Verification

- [x] Full build passes: `npm --prefix agent-config run build:agents`
- [x] No remaining `./thoughts/` in generated skill bodies:
  `grep -r '\./thoughts/' generated/`
  (expected: no matches in skill body content; path-only strings in descriptions are fine)
- [x] No remaining `~/Workspace/thoughts/` references:
  `grep -r 'Workspace/thoughts' generated/`
  (expected: zero matches)

#### Manual Verification

- [ ] Spot-check `generated/cursor/skills/create-plan/SKILL.md` — discovery prose present
- [ ] Spot-check `generated/codex/skills/research-web/SKILL.md` — discovery prose present
- [ ] Spot-check `generated/opencode/commands/split-branch.md` — old walk-up block gone;
  discovery prose present

---

## Testing Strategy

### Automated Tests

The fragment build pipeline has no automated behavioral test for discovery prose (it is
natural language, not executable code). Build success and grep checks are the automated
gates.

### Manual Testing Steps

1. From a git worktree path inside a project, invoke a write skill (e.g. `create-idea`)
   — confirm it finds the correct `thoughts/` rather than resolving to the worktree's
   local path
2. Set `$THOUGHTS_DIR=/tmp/custom-thoughts` and invoke any skill — confirm it skips
   discovery and uses the override path
3. Within a session, invoke one skill (let it run discovery and export `THOUGHTS_DIR`),
   then invoke a second skill — confirm the second skill's Step 1 finds the exported
   value and skips traversal
3. Invoke a read-only skill (e.g. `implement-plan`) from a directory with no `thoughts/`
   ancestor — confirm the error output lists the searched paths and gives actionable fix
   instructions

## References

- Idea document: `thoughts/ideas/2026-03-15-consistent-thoughts-dir-discovery.md`
- Fragment system agent-docs: `agent-docs/compiler-fragments.md`
- Best existing discovery: `spec/skills/split-branch/SKILL.md:66`
- Fragment resolution code: `scripts/lib/fragments.ts`
- Build pipeline: `package.json` (`build:agents` script at line 11)
- Claude Code workspace root research: `$CLAUDE_PROJECT_DIR` empty in subagents
  (issue #26429); workspace root available as prose in `<env>` system prompt block
