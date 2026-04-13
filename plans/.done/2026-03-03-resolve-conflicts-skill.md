# resolve-conflicts: Skill Implementation Plan

## Overview

Extract the conflict resolution workflow from `gt-sync` Phase 3 into a
standalone `resolve-conflicts` skill. The skill owns the full detect → identify
→ analyze → present → approve → write → verify → stage cycle. When invoked
directly by a user, it detects the in-progress git operation and offers
post-resolution actions. When loaded by another skill (e.g., `gt-sync`), it
accepts caller-provided context and returns results. `gt-sync` is then
refactored to delegate Phase 3 steps to the skill.

## Current State Analysis

### Existing infrastructure

- **`spec/skills/gt-sync/SKILL.md` Phase 3** — Steps 1–9 are the conflict
  resolution workflow to extract. Steps 1, 3–8 are universally reusable; steps
  2 and 9 and the outer loop are gt-sync-specific (branch reporting, round
  counter, `git-town continue`).
- **`spec/skills/git-town/SKILL.md`** and **`spec/skills/gh-safe/SKILL.md`** —
  Existing non-user-invocable skill patterns (`user_invocable: false`,
  `agent_invocable: true`). Both are directory-based
  (`spec/skills/<id>/SKILL.md`).
- **`spec/skills/gt-sync/SKILL.md`** and **`spec/skills/split-branch/SKILL.md`**
  — User-invocable skill patterns with full frontmatter including
  `capabilities.tools`, `execution.model_profile`, and `skill:` block.
- **`spec/schema/canonical.schema.json`** — Schema validation. Only `kind:
  "agent"` and `kind: "skill"` exist. Skills infer `kind` from path. Invocability
  is expressed via top-level `user_invocable` and `agent_invocable` booleans.
- **`mappings/tools.yaml`** and **`agent-docs/canonical-command-conventions.md`**
  — Tool names and naming conventions.

### Key discoveries

- All specs live in `spec/skills/<id>/SKILL.md` (directory-based). The former
  `spec/commands/` directory no longer exists.
- `kind: "skill"` is inferred by the build pipeline from the directory path —
  skills do not need `kind:` in their frontmatter.
- Invocability is two top-level booleans: `user_invocable` (appears in slash
  command menu) and `agent_invocable` (can be auto-invoked by agents). At least
  one must be `true`.
- Skills with `capabilities.tools` get tool mapping during build (`allowed-tools`
  in Claude output). Skills without tools (like `git-town`) inherit tools from
  the loading context.
- The `skill:` block holds optional fields: `accepts_args`, `args_schema`,
  `delegate_to`.
- The extraction boundary from gt-sync Phase 3 (confirmed from source):

  | Step | Content | Destination |
  |------|---------|-------------|
  | 1 | `git diff --name-only --diff-filter=U` | **skill** |
  | 2 | Branch name display, round context | stays in **gt-sync** |
  | 3 | Subagent dispatch, per-hunk analysis | **skill** |
  | 4 | Summary table + approval choices | **skill** |
  | 5 | Per-file review (Approve / Edit / Resolve manually) | **skill** |
  | 6 | Write resolved content | **skill** |
  | 7 | Verify no conflict markers remain | **skill** |
  | 8 | `git add <files>` | **skill** |
  | 9 | `git-town continue` + exit code routing + round counter | stays in **gt-sync** |
  | — | Outer conflict loop | stays in **gt-sync** |

- When invoked directly (`/resolve-conflicts`), the skill must detect which git
  operation is in progress. Detection uses sentinel files in `.git/`:
  - Rebase: `.git/rebase-merge/` or `.git/rebase-apply/` directory exists
  - Cherry-pick: `.git/CHERRY_PICK_HEAD` file exists
  - Stash pop: `.git/MERGE_HEAD` exists AND its value equals `refs/stash`
    (stash pop leaves MERGE_HEAD pointing to the stash commit)
  - Merge: `.git/MERGE_HEAD` exists (checked after stash-pop test)
  - Unknown: none of the above (resolve files and stage, skip continue offer)

## Desired End State

After implementation:

1. `spec/skills/resolve-conflicts/SKILL.md` exists with correct frontmatter
   and complete body covering both standalone invocation and caller-loaded use.
2. `spec/skills/gt-sync/SKILL.md` Phase 3 is refactored: steps 3–8 removed,
   replaced with "Load and invoke the `resolve-conflicts` skill."
3. `npm run build:agents` succeeds with no errors.
4. All four provider targets generated for the new skill.
5. `gt-sync`'s generated output reflects the refactored Phase 3.

### How to verify

- `generated/claude/skills/resolve-conflicts/SKILL.md` exists
- `generated/claude/skills/resolve-conflicts/SKILL.md` frontmatter has
  `name: resolve-conflicts` and includes `allowed-tools` (user-invocable skills
  with tools get tool mapping)
- `generated/claude/skills/gt-sync/SKILL.md` Phase 3 references the
  `resolve-conflicts` skill and no longer contains steps 3–8 inline

## What We're NOT Doing

- **Automatic conflict resolution without user approval.** Skill only suggests.
- **Binary file resolution.** Binary conflicts are flagged and deferred.
- **Retry/undo after a failed continue.** The skill surfaces the error and
  stops.
- **Detecting the specific stash entry index.** We always reference
  `stash@{0}` — the convention when stash pop conflicts is that the entry
  remains at index 0.
- **Modifying the `git-town` skill.** No changes there.

---

## Phase 1: Create the resolve-conflicts Skill

### Overview

Create `spec/skills/resolve-conflicts/SKILL.md` — a unified skill that:
- When loaded by another skill (e.g., `gt-sync`): accepts optional caller
  context, resolves all unmerged files through the analyze → approve → write →
  verify → stage cycle (with a partial-resolution loop), and returns a summary.
- When invoked directly by a user (`/resolve-conflicts`): detects the
  in-progress git operation, runs the resolution workflow, and offers
  post-resolution actions (continue command or stash drop).

### Changes Required

#### 1. Create the skill directory and spec file

**File**: `spec/skills/resolve-conflicts/SKILL.md` (new directory + file)

- [x] Create `spec/skills/resolve-conflicts/SKILL.md` with the following
  frontmatter:

```yaml
---
id: resolve-conflicts
description: >
  Resolve merge conflicts in the working tree: identifies unmerged files,
  analyzes each conflict hunk via parallel subagents, presents a summary
  with confidence ratings, collects user approval, writes resolved content,
  verifies no conflict markers remain, and stages the results. When invoked
  directly, detects the in-progress git operation and offers to continue it
  after resolution.
version: 1
user_invocable: true
agent_invocable: true
execution:
  model_profile: planning
capabilities:
  tools:
    - read
    - write
    - edit
    - grep
    - bash
    - task
    - todowrite
skill:
  accepts_args: false
compat:
  targets:
    - claude
    - opencode
    - codex
    - cursor
---
```

- [x] Write the skill body with the following sections:

##### Title and opening

```markdown
# resolve-conflicts — Interactive Conflict Resolution
```

Followed by a 2–3 sentence summary: what the skill does, when to load it,
and the two invocation modes (direct and caller-loaded).

##### Caller interface

Document the optional context a caller may pass in:

- **Operation name** (optional, string) — displayed in the conflict summary
  header (e.g., `"git-town sync"`, `"git rebase"`). When provided, the skill
  skips operation detection and uses this value directly. Defaults to
  auto-detection if omitted.
- **File list** (optional, list of paths) — restricts resolution to a subset
  of conflicted files. If omitted, the skill resolves all unmerged files.

Document what the skill returns to the caller:

- List of resolved file paths (staged)
- Count of conflicts resolved
- List of files skipped for manual resolution (if any)

##### Agent architecture

Skills are loaded into the **main agent's context**, not dispatched as
subagents. "Loading a skill" means the main agent reads its instructions and
follows them directly — there is no separate process or boundary. Because the
main agent always owns the conversation with the user, all user-facing steps
in this skill (presenting the summary table, asking for approval, per-file
review) are executed by the main agent and are valid regardless of which
context loaded the skill.

The one exception is analysis: the main agent dispatches **one subagent per
conflicted file** in parallel (via the `task` tool) for the heavy context work
of reading and reasoning about conflict markers. Subagents must not write files
or ask the user questions — if blocked, they return their findings to the main
agent and let it decide what to ask.

##### Direct invocation: Pre-flight

_This section applies only when invoked directly (`/resolve-conflicts`). When
loaded by another skill that provides an operation name, skip to the resolution
workflow._

**Step 1: Check for active conflicts**

```bash
git diff --name-only --diff-filter=U
```

If the output is empty, stop immediately with: "No merge conflicts detected in
the working tree. Nothing to resolve."

**Step 2: Detect which operation is in progress**

Run these checks in order (first match wins):

```bash
# Rebase in progress
GIT_DIR=$(git rev-parse --git-dir)
test -d "$GIT_DIR/rebase-merge" || test -d "$GIT_DIR/rebase-apply"

# Cherry-pick in progress
test -f "$GIT_DIR/CHERRY_PICK_HEAD"

# Stash pop in progress (MERGE_HEAD points to same commit as stash)
MERGE_HEAD=$(cat "$GIT_DIR/MERGE_HEAD" 2>/dev/null)
STASH_REF=$(git rev-parse refs/stash 2>/dev/null)
test -n "$MERGE_HEAD" && test "$MERGE_HEAD" = "$STASH_REF"

# Merge in progress
test -f "$GIT_DIR/MERGE_HEAD"
```

Set `$OPERATION` to one of: `rebase`, `cherry-pick`, `stash-pop`, `merge`,
or `unknown` (if none of the above match).

**Step 3: Show pre-flight summary**

Display the detected state and list conflicted files before proceeding.
Example:

```
Resolve Conflicts

Operation:  git rebase (in progress)
Conflicts:  3 files
  src/auth/login.ts
  src/auth/session.ts
  src/config/auth.yaml

Proceed with resolution?
```

For `unknown` operation: "Could not determine which git operation is in
progress. Files will be resolved and staged; you will need to run the
appropriate continue command manually."

##### Resolution workflow

_This section applies to all invocation modes._

**Step 1 — Identify conflicted files**

```bash
git diff --name-only --diff-filter=U
```

If the caller provided a file list, intersect with this output. If no
conflicted files are found (or none match the restricted list), return
immediately to the caller with zero conflicts resolved.

**Step 2 — Analyze conflicted files**

For each conflicted file, dispatch a subagent in parallel with:
- The full file content (with conflict markers)
- Instruction to identify every `<<<<<<< … ======= … >>>>>>>` block
- Instruction to analyze "ours" (above `=======`) and "theirs" (below
  `=======`) in the context of surrounding code
- Instruction to produce:
  (a) confidence per hunk: `clear` (one side obviously correct) or
      `needs-review` (structural or ambiguous)
  (b) per-hunk recommendation with rationale
  (c) a fully-resolved version of the file
- The subagent must NOT write the file — returns results to the main agent

Wait for all subagents to complete before presenting.

**Step 3 — Present conflict summary**

Show a compact summary table. The header includes the operation name
(from caller context or auto-detection):

```
Conflict during <operation name> — N files

File                          Hunks  Status
────────────────────────────  ─────  ────────────────────────
src/auth/login.ts             2      clear (ours preferred)
src/auth/session.ts           1      clear (theirs preferred)
src/config/auth.yaml          1      needs review
```

Present three options and wait for user selection:
- **Approve clear** — apply `clear` recommendations; queue `needs-review`
  files for one-at-a-time review
- **Review all** — go through every file one at a time
- **Approve all** — accept all recommendations including `needs-review` files

**Step 4 — Process files per user choice**

**For files approved without review:** write resolved content directly.

**For files being reviewed** (one at a time):
1. Show per-hunk breakdown (which side chosen, why, with context)
2. Show proposed fully-resolved content
3. Ask: **Approve** / **Edit** (user provides changes) / **Resolve manually**
   (skip; user will edit the file before staging)

**Step 5 — Write resolved files**

For each approved file, write the resolved content replacing the
conflict-marker version.

**Step 6 — Verify no conflict markers remain**

For each written file:

```bash
grep -n '^<<<<<<<\|^=======\|^>>>>>>>' <file>
```

If any markers are found, do not stage that file — report which lines still
contain markers and ask the user whether to re-edit or resolve manually.
Only proceed to staging once all written files are marker-free.

**Step 7 — Stage resolved files**

```bash
git add <file1> <file2> ...
```

Specific file paths only — never `git add -A` or `git add .`.

##### Partial-resolution loop

After staging, run `git diff --name-only --diff-filter=U` again. If unmerged
files remain (because the user chose "Resolve manually" for some), present
them and ask:
- **Re-analyze now** — the user has edited the file; dispatch a fresh subagent
  and repeat steps 2–7 for the remaining files
- **Stop here** — return to the caller with the remaining files listed as
  skipped

Loop until all files are resolved or the user stops.

##### Direct invocation: Post-resolution

_This section applies only when invoked directly (`/resolve-conflicts`). When
loaded by another skill, return results to the caller and stop._

Based on `$OPERATION` and whether any files were skipped:

**If all files resolved:**

| `$OPERATION` | Offer |
|---|---|
| `rebase` | Run `git rebase --continue`? |
| `merge` | Run `git merge --continue`? |
| `cherry-pick` | Run `git cherry-pick --continue`? |
| `stash-pop` | Run `git stash drop stash@{0}`? (explain: stash pop = apply + drop; apply succeeded, drop was skipped because of the conflict — drop it now to clean up) |
| `unknown` | Show the staged files and remind the user to run their continue command manually |

Present each offer as a confirmation. If the user declines, show the command
they would need to run manually.

**If some files were skipped (manual resolution):**

Show the remaining unmerged files and the commands to run after resolving them:

```
N file(s) still unresolved:
  src/config/auth.yaml

After resolving manually:
  git add src/config/auth.yaml
  git rebase --continue   (or the appropriate continue command)
```

Do not run the continue command if any files remain unmerged.

### Success Criteria

#### Automated Verification

- [x] `spec/skills/resolve-conflicts/SKILL.md` exists
- [x] Schema validates: `npm run validate:agents`
- [x] `generated/claude/skills/resolve-conflicts/SKILL.md` exists
- [x] Generated frontmatter has `name: resolve-conflicts`, `description`,
  and `allowed-tools` (user-invocable skills with tools get tool mapping)
- [x] `generated/opencode/skills/resolve-conflicts/SKILL.md` exists
- [x] `generated/codex/skills/resolve-conflicts/SKILL.md` exists
- [x] `generated/cursor/skills/resolve-conflicts/SKILL.md` exists

#### Manual Verification

- [x] Read `generated/claude/skills/resolve-conflicts/SKILL.md` and confirm:
  - Caller interface is clearly documented (what to pass in, what comes back)
  - Direct invocation pre-flight (detection logic) is clearly separated from
    the shared resolution workflow
  - The stash-pop detection (MERGE_HEAD vs refs/stash comparison) is correctly
    described
  - Post-resolution offers match the operation table
  - Steps 1–7 and the partial-resolution loop are coherent and actionable
  - Subagent dispatch instructions are unambiguous
  - No provider-specific tool names appear in the body text

---

## Phase 2: Refactor gt-sync + Build Verification

### Overview

Update `spec/skills/gt-sync/SKILL.md` Phase 3 to delegate the conflict
resolution steps to the `resolve-conflicts` skill. The outer loop, round
counter, branch reporting, and `git-town continue` call remain in gt-sync.
Rebuild and verify all generated output.

### Changes Required

#### 1. Refactor gt-sync Phase 3

**File**: `spec/skills/gt-sync/SKILL.md`

Current Phase 3 structure (9 steps):
1. Identify conflicted files
2. Report conflict location (gt-sync-owned)
3. Analyze conflicted files
4. Present conflict summary
5. Process files per user choice
6. Write resolved files
7. Verify no conflict markers remain
8. Stage resolved files
9. Continue sync (gt-sync-owned)

- [x] Remove steps 3–8 from Phase 3 (the content being extracted)
- [x] Replace with a single step: "Load the `resolve-conflicts` skill. Invoke
  it, passing `"git-town sync"` as the operation name. Wait for the skill to
  complete and receive the list of resolved and skipped files."
- [x] Keep step 1 (identify conflicted files — the loop entry check)
- [x] Keep step 2 (report conflict location: current branch + round counter)
- [x] Keep step 9 (continue sync: `git-town continue`, exit code routing, round
  counter increment)
- [x] Keep the outer loop envelope ("This phase repeats until `git-town
  continue` exits cleanly")
- [x] Renumber steps after the edit so the numbering is contiguous

After refactoring, Phase 3 should have these steps:
1. Identify conflicted files (`git diff --name-only --diff-filter=U`)
2. Report conflict location (branch name + round counter)
3. Load and invoke the `resolve-conflicts` skill
4. Continue sync (`git-town continue`, exit code routing)

#### 2. Run the build

- [x] Run `npm run build:agents`
- [x] Confirm no validation errors
- [x] Confirm no compilation errors

#### 3. Verify generated output

- [x] `generated/claude/skills/resolve-conflicts/SKILL.md` exists
- [x] `generated/claude/skills/gt-sync/SKILL.md` Phase 3 no longer contains
  the inline analysis, summary, and staging steps — only the 4-step structure
  above
- [x] `generated/opencode/skills/resolve-conflicts/SKILL.md` exists
- [x] `generated/codex/skills/resolve-conflicts/SKILL.md` exists
- [x] `generated/cursor/skills/resolve-conflicts/SKILL.md` exists

### Success Criteria

#### Automated Verification

- [x] Full build passes: `npm run build:agents`
- [x] No warnings or errors in build output
- [x] New skill exists in all four provider targets

#### Manual Verification

- [x] Read `generated/claude/skills/gt-sync/SKILL.md` Phase 3 and confirm it
  loads the `resolve-conflicts` skill rather than inlining the resolution logic
- [x] Confirm Phase 3 in gt-sync still clearly describes the gt-sync-specific
  loop: round counter, `git-town continue`, and conflict re-detection
- [x] Confirm the `resolve-conflicts` skill body is complete and coherent end-
  to-end on its own (readable without gt-sync context)

---

## Testing Strategy

### Automated Tests

- Schema validation: `npm run validate:agents`
- Build compilation: `npm run build:agents`
- No unit tests to add (project validates via schema + build only)

### Manual Testing Steps

1. Read `generated/claude/skills/resolve-conflicts/SKILL.md` from top to
   bottom as if you're an LLM receiving this skill — verify it's self-contained
   and unambiguous about:
   - The two invocation modes (direct vs caller-loaded)
   - Caller interface, subagent dispatch, and partial-resolution loop
   - Operation detection and post-resolution offers (direct mode)
2. Read the refactored `generated/claude/skills/gt-sync/SKILL.md` Phase 3 and
   verify the skill reference doesn't leave any gaps in the conflict loop

## References

- Idea document: `thoughts/ideas/2026-03-03-resolve-conflicts-skill.md`
- Source of extraction: `spec/skills/gt-sync/SKILL.md` (Phase 3)
- Skill pattern references: `spec/skills/git-town/SKILL.md`,
  `spec/skills/gh-safe/SKILL.md`
- User-invocable skill reference: `spec/skills/split-branch/SKILL.md`
- Schema: `spec/schema/canonical.schema.json`
- Tool mappings: `mappings/tools.yaml`
- Naming conventions: `agent-docs/canonical-command-conventions.md`
