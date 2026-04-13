# Code Review Loop Skill — Implementation Plan

## Overview

Add `spec/skills/code-review-loop/SKILL.md`, a new user- and agent-invocable skill
that auto-fixes unambiguous code-review findings in a loop, re-reviews after each
batch of fixes, and escalates only real decisions to the user. The skill reuses the
existing `code-reviewer` subagent and prompt contract unchanged; all orchestration
logic lives in the skill body.

## Current State Analysis

- `spec/skills/code-review/SKILL.md` — collaborative triage skill; presents every
  finding to the user one-by-one (fix now / skip / defer?). Explicitly states "do not
  auto-fix findings without user agreement on each one."
- `spec/skills/review-files/SKILL.md` — same collaborative pattern, scoped to
  files/globs/directories.
- `spec/agents/code-reviewer.md` — readonly subagent, returns structured output:
  Blockers / Major / Minor / Questions / Verification Gaps / Summary sections.
- No existing skill provides auto-fix or loop behaviour.

### Key Discoveries

- `spec/schema/canonical.schema.json:113` — `skill.delegate_to` is a kebab-case
  string for simple delegation. code-review-loop does **not** use `delegate_to`
  because it orchestrates a multi-pass loop rather than delegating once.
- `spec/skills/code-review/SKILL.md:8` — uses `model_profile: deep_review`. Same
  profile is correct for code-review-loop (complex multi-step orchestration).
- `spec/skills/code-review/SKILL.md:11` — code-review lists only `task` in
  capabilities. code-review-loop needs more: `bash, edit, read, task, todowrite, write`.
- `spec/skills/implement-plan/SKILL.md:9-19` — existing example of a skill that makes
  code edits; explicitly lists all tools it uses. Follow this pattern.
- `README.md:145` — "Skills infer `kind: skill` from their path — don't need to
  declare it."
- Build command: `npm --prefix agent-config run build:agents` (runs validate + compile
  - check in sequence).

## Desired End State

- `spec/skills/code-review-loop/SKILL.md` exists with valid frontmatter and complete
  skill body.
- `npm --prefix agent-config run build:agents` completes with no errors or warnings.
- Generated provider output files are present (at minimum under `generated/claude/`).

### Automated Verification

- [x] `npm --prefix agent-config run validate:agents` exits 0
- [x] `npm --prefix agent-config run build:agents` exits 0 with no errors

## What We're NOT Doing

- Modifying `spec/agents/code-reviewer.md` or its generated outputs
- Modifying `spec/skills/code-review/SKILL.md` or `spec/skills/review-files/SKILL.md`
- Adding a file-scoped variant — code-review-loop handles branch/diff scope only
  (that's `review-files`'s job). **Deviation from idea file**: the idea's success
  criteria lists "specific files via optional git ref argument" as a supported scope,
  but the optional argument in this plan is a git ref for base-branch selection only
  (not a file path list). File-scoped review is already covered by `review-files`.
- Reviewing uncommitted-only changes (no-base scope) — the skill always resolves
  a merge-base; if invoked on the default branch with no commits ahead of it,
  the merge-base equals HEAD and the reviewer sees no diff. This is a known
  limitation shared with `code-review`.
- Auto-committing fixes (user retains control of git operations)
- Supporting resume from an existing state file (always overwrites)
- Adding new review categories or severity levels

## Implementation Approach

Create one new file: `spec/skills/code-review-loop/SKILL.md` (and its parent
directory). Then run the build to validate frontmatter and generate provider outputs.
The skill body encodes all orchestration logic the main agent follows when
`/code-review-loop` is invoked.

---

## Phase 1: Create `spec/skills/code-review-loop/SKILL.md`

### Overview

Create the directory and write the canonical skill spec with the full frontmatter
and skill body specified below.

### Changes Required

#### 1. Create `spec/skills/code-review-loop/SKILL.md`

**File**: `spec/skills/code-review-loop/SKILL.md`

- [x] Create directory `spec/skills/code-review-loop/`
- [x] Write `SKILL.md` with the exact frontmatter and body below

---

**Full frontmatter:**

```yaml
---
id: code-review-loop
description: Auto-fix unambiguous findings in a review loop; escalate decisions to the user.
version: 1
user_invocable: true
agent_invocable: true
execution:
  model_profile: deep_review
capabilities:
  tools:
    - bash
    - edit
    - grep
    - read
    - task
    - todowrite
    - write
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

**Rationale for each frontmatter choice:**

| Field           | Value              | Reason                                                                                                                                                                                                         |
| --------------- | ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`            | `code-review-loop` | Kebab-case, matches directory name per naming convention                                                                                                                                                       |
| `kind`          | _(omitted)_        | Skills infer `kind: skill` from path                                                                                                                                                                           |
| `delegate_to`   | _(omitted)_        | Not a simple delegation — orchestrates a multi-pass loop. Safe to omit: `validate.ts:127` only validates `delegate_to` when present; omitting it skips the check entirely                                      |
| `model_profile` | `deep_review`      | Same as `code-review`; complex orchestration requires deep reasoning                                                                                                                                           |
| `bash`          | ✓                  | Git operations (fetch, merge-base) and state file pass-summary appending via `>>` (step 4e). `bash` is used for appending because `write` overwrites the whole file — the two tools serve distinct I/O roles.  |
| `edit`          | ✓                  | Apply code fixes to source files                                                                                                                                                                               |
| `grep`          | ✓                  | Fix verification (e.g., confirming a renamed symbol is consistent across a file, checking for dangling references after an import removal). Zero cost to include; omitting it risks a rebuild cycle if needed. |
| `glob` / `ls`   | _(omitted)_        | The `code-reviewer` subagent handles all file discovery; the main agent applies targeted edits at specific file:line locations — no directory traversal needed.                                                |
| `read`          | ✓                  | Re-read state file at the start of each pass                                                                                                                                                                   |
| `task`          | ✓                  | Spawn the `code-reviewer` subagent                                                                                                                                                                             |
| `todowrite`     | ✓                  | Create/update todos for user-decision items and end-of-loop triage                                                                                                                                             |
| `write`         | ✓                  | Initialize the state file (full overwrite at start)                                                                                                                                                            |
| `accepts_args`  | `true`             | Optional git reference argument (same as `code-review`)                                                                                                                                                        |
| `args_schema`   | `string`           | Matches `code-review`; the arg is a git ref string (e.g., `main`, `develop`)                                                                                                                                   |

---

**Full skill body** (write exactly this markdown as the body after the frontmatter):

````markdown
# Code Review Loop

Spawn a specialized code-reviewer agent, auto-fix unambiguous findings, re-review
after each batch of fixes, and escalate real decisions to the user. Loops up to 3
passes per checkpoint before pausing to confirm with the user.

## Your Task

### 1. Understand the context and parse args

- Check what directory you're in (`pwd`)
- Determine if this is a project within a workspace
- Parse the optional git reference argument (if provided, treat it as `$BASE_BRANCH`)

### 2. Resolve base branch and merge-base

Use the same resolution logic as `code-review`:

- If the user passed a git reference argument, check if a remote-tracking version
  exists:

  ```bash
  if git rev-parse --verify "origin/$BASE_BRANCH" >/dev/null 2>&1; then
    git fetch origin "$BASE_BRANCH" --quiet 2>/dev/null
    MERGE_BASE=$(git merge-base HEAD "origin/$BASE_BRANCH")
  else
    MERGE_BASE=$(git merge-base HEAD "$BASE_BRANCH")
  fi
  ```

- If no argument was provided, determine `$BASE_BRANCH` from the remote default
  branch and fall back to `main`:

  ```bash
  BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
  BASE_BRANCH=${BASE_BRANCH:-main}
  git fetch origin "$BASE_BRANCH" --quiet 2>/dev/null
  MERGE_BASE=$(git merge-base HEAD "origin/$BASE_BRANCH")
  ```

- Use `$MERGE_BASE` as the canonical baseline in reviewer prompts so the review
  remains stable even if `$BASE_BRANCH` advances during the loop.

### 3. Initialize the state file

Use the `write` tool to create `.code-review-loop.md` in the project root:

```markdown
# Code Review Loop

Scope: changes against <BASE_BRANCH> (merge-base: <MERGE_BASE>)
```

If a file already existed at `.code-review-loop.md`, notify the user:
"Replaced existing `.code-review-loop.md`."

Suggest to the user: "`.code-review-loop.md` is a hidden file left as a review
artifact. Add it to `.gitignore` if you don't want it committed."

Initialize loop state in working memory:

- `pass_number = 0` — incremented to 1 at the **start** of each pass (step 4a),
  so it always equals the current pass number during execution; used in state
  file headers ("## Pass 1", "## Pass 2", …)
- `passes_in_window = 0` — incremented at the **end** of each pass (step 4g),
  only when fixes were applied; tracks completed productive passes in the current
  checkpoint window; resets to 0 when the user approves continuation
- `seen_skipped = []` (file:line or description strings of user-skipped findings)
- `questions = []` (Questions/Verification Gaps collected during the loop)
- `total_fixed = 0`
- `fixes_this_pass = 0`

### 4. Main loop

Repeat steps 4a–4g until no fixes were applied in a pass (natural exit) or the
user declines to continue at the 3-pass checkpoint.

#### 4a. Start the pass

Increment `pass_number` by 1. Then re-read `.code-review-loop.md`. Collect all items listed under any
`**Skipped (user):**` bullet as the updated `seen_skipped` set. Skip any bullet
that reads exactly `- none` — that is the sentinel written when a pass had no
skipped items, not a real finding. The `- none` sentinel is used consistently
in all three state file sections (Fixed, Skipped, Questions) when a section is
empty; only the Skipped section is parsed here. This ensures seen_skipped
survives context compression between passes.

#### 4b. Spawn the code-reviewer agent

Use the Task tool with `subagent_type: "code-reviewer"` and the standard prompt
contract:

```
Review request:
- Project path: <absolute-or-workspace-relative path>
- Scope: changes against <base-ref>
- Base reference: <git-ref>
- Merge-base: <commit-sha>
- Focus: none
```

#### 4c. Filter findings

Set `fixes_this_pass = 0`. Remove any finding already in `seen_skipped` (match on
file:line identifier or description). Do not re-present user-skipped findings
during the current loop.

#### 4d. Process each remaining finding in severity order (Blockers → Major → Minor)

Apply the auto-fix heuristic to each finding:

**Auto-fix (apply immediately, no user confirmation):**

- Unused imports / dead code removal
- Missing error handling with an obvious pattern (e.g., unchecked error matching
  the surrounding code style)
- Typos in strings, comments, or variable names
- Style/formatting inconsistencies
- Missing null/undefined checks with an obvious guard pattern
- Simple type fixes

If auto-fixing:

1. Apply the fix using the Edit tool
2. Record the fix in working memory for the pass summary
3. Increment `total_fixed` and `fixes_this_pass`
4. Do NOT create a todo

**Stop and ask (decision required):**

- Architectural changes
- Performance trade-offs (caching strategy, algorithm choice)
- API design decisions
- Behavior changes affecting external contracts
- Security-sensitive changes where intent matters
- Any finding where the reviewer noted multiple valid approaches

If stopping to ask:

1. Create a todo describing the finding and the decision needed
2. Mark the todo as in-progress
3. Present the finding to the user and ask: "Fix with [approach], skip, or defer?"
4. **If fix agreed**: apply using Edit, mark todo complete, record in Fixed list
5. **If skip/defer**: mark todo complete with a note, add the finding to
   `seen_skipped`, record in Skipped list

**Questions/Assumptions and Verification Gaps:**

- Add to `questions` list for end-of-loop discussion
- Record in Questions list for the pass summary
- Do NOT block the loop or ask the user now

#### 4e. Append the pass summary to the state file

After all findings are processed for this pass, use bash `>>` to append a summary
block to `.code-review-loop.md`. Write `- none` as the sole bullet in any section
that has no items (this is the sentinel that `seen_skipped` reconstruction skips).

The block must follow this structure:

```
## Pass N

**Fixed:**
- src/api.ts:42 — unused import removed
- src/api.ts:88 — missing null check added

**Skipped (user):**
- none

**Questions (end-of-loop):**
- none
```

#### 4f. Print per-pass summary to the user

```
Pass N: fixed X issues — <comma-separated list of what was fixed>
```

If no new actionable findings were found (all filtered or none returned):

```
Pass N: no new issues found — review complete.
```

#### 4g. Decide whether to continue

- If `fixes_this_pass > 0`:
  - Increment `passes_in_window` (note: `pass_number` was already incremented at
    the start of this pass in step 4a; only `passes_in_window` is incremented here)
  - If `passes_in_window < 3`: loop back to 4a
  - If `passes_in_window >= 3`: prompt the user:

    ```
    <total_fixed> total issues fixed so far.
    Completed <passes_in_window> passes in this window.
    The loop has reached the 3-pass checkpoint. Continue reviewing?
    ```

    - If yes: reset `passes_in_window = 0`, loop back to 4a
    - If no: proceed to step 5

- If `fixes_this_pass == 0`: exit the loop, proceed to step 5

### 5. End-of-loop triage

Work through remaining items collaboratively using the same triage workflow as
`code-review`:

1. Re-read `.code-review-loop.md` to collect all items under any
   `**Skipped (user):**` section
2. If there are skipped items:
   - Create a todo for each
   - For each item: present it, ask "fix now, skip, or defer?"
   - If fixing: apply the change, mark todo complete
   - If skipping/deferring: mark todo complete with a note
3. If there are items in `questions`:
   - Create a todo for each
   - Discuss them with the user for alignment (no code changes expected for
     Questions/Verification Gaps)
   - Mark todos complete as each is resolved

### 6. Final summary

Print:

```
## Code Review Loop Complete

- Passes completed: N
- Total fixes applied: X
- Items deferred/skipped: Y
- Questions discussed: Z

State file: .code-review-loop.md (kept as review artifact)
```

The `.code-review-loop.md` file is intentionally left in place.

## Important Notes

- **DO NOT** perform the review yourself — always spawn the code-reviewer agent
- **Auto-fix only when the fix is unambiguous** — when there are multiple valid
  approaches, stop and ask
- **Re-read the state file at the start of every pass** — this is your resilience
  against context compression
- **Do not create todos for auto-fixed items** — they appear in the per-pass
  summary only
- **seen_skipped prevents re-asking mid-loop** — but skipped items ARE surfaced
  again in the end-of-loop triage
- **Questions/Verification Gaps do not block the loop** — collect and discuss at
  the end
````

> **State file design note (plan rationale, not part of SKILL.md)**: The pass
> summary is written at the END of each pass, not after each individual finding
> (deviation from idea file which specifies per-finding writes). This keeps the
> format clean and provides the primary resilience goal: surviving context
> compression _between_ passes.
>
> **Known trade-off**: If context compression occurs _mid-pass_, fixes applied
> before compression will not appear in the state file for that pass. The actual
> file edits persist (the code is already changed), but the state record will be
> incomplete. The agent re-reads the state file at the start of the next pass and
> re-runs the reviewer — the reviewer will not re-report already-fixed issues, so
> the loop continues correctly. The only loss is the per-pass summary entry for
> those fixes.

---

### Success Criteria

#### Automated Verification

- [x] `npm --prefix agent-config run validate:agents` exits 0
- [x] `npm --prefix agent-config run build:agents` exits 0 with no errors
- [x] `spec/skills/code-review-loop/SKILL.md` passes frontmatter schema validation
      (no `additionalProperties` errors, required fields present)
- [x] Generated output exists for at least the `claude` target
      (e.g., `generated/claude/skills/code-review-loop/` or equivalent path)

#### Manual Verification

- [x] The frontmatter is consistent: `id`, directory name, and invocation trigger
      are all `code-review-loop` (no snake_case or camelCase drift)
- [x] The skill body accurately reflects the decisions table in the idea file,
      with the following intentional deviations documented in the plan: - **Deviation — state file write granularity**: idea says "write after
      each finding action"; plan writes the pass summary at END of each pass
      (see step 4e rationale note in the skill body) - **Deviation — "specific files" scope**: idea lists it as supported;
      plan intentionally excludes it (see "What We're NOT Doing") - **Deviation — counter-tracking consistency and triage verification**:
      SKILL.md ensures all fix paths increment counters. Step 4d "fix agreed"
      adds `increment total_fixed and fixes_this_pass` (plan body omitted
      these, which would cause user-agreed fixes to not be counted). Step 5
      adds `triage_fixes = 0` tracking, increments `total_fixed` and
      `triage_fixes` on triage fixes, allows questions to warrant code fixes
      (more flexible than plan's "no code changes expected"), and adds
      step 5.4 to re-enter the main loop after triage fixes for verification.
      Together these close gaps where fixes would go uncounted or unreviewed. - **Deviation — `delegate_to` and explicit spawn instruction**: plan
      omits `delegate_to` (rationale: multi-pass orchestration, not simple
      delegation). SKILL.md adds `delegate_to: code-reviewer` in frontmatter
      and explicit `subagent_type: "code-reviewer"` in body to match sibling
      skill conventions (`code-review`, `review-files`).
      All other decisions must match: - seen_skipped prevents re-asking mid-loop - Pass cap is fixed at 3 (not configurable) - Auto-fixes produce no todos - State file is always overwritten at startup - State file is left in place as a review artifact - Fix introduces a new Blocker → treat it like any other finding in the
      next pass (no auto-revert logic)

---

## Phase 2: Build and Validate

### Overview

Run the build pipeline to validate frontmatter, compile provider outputs, and
confirm the new skill appears in generated files.

### Changes Required

- [x] Run `npm --prefix agent-config run build:agents` from the repo root

### Success Criteria

#### Automated Verification

- [x] Build exits 0 with no errors
- [x] No schema validation warnings related to `code-review-loop`
- [x] Generated files for all 4 compat targets (`claude`, `opencode`, `codex`,
      `cursor`) are present or noted as unsupported by a target's adapter

#### Manual Verification

- [x] Review generated output for the `claude` target — confirm the skill body is
      rendered correctly (frontmatter stripped, body preserved)

---

## Testing Strategy

This skill is pure instruction text — there is no executable code to unit-test.
Correctness is verified by:

1. **Schema validation** (`validate:agents`) — catches malformed frontmatter
2. **Build success** (`build:agents`) — catches adapter-level issues
3. **Prose review** — reading the generated output to confirm the skill body is
   coherent and matches the design decisions from the idea file

Functional testing (actually invoking `/code-review-loop` and running the loop)
is out of scope for this plan; it belongs in a separate integration exercise.

## References

- Idea file: `thoughts/ideas/2026-03-03-code-review-loop-skill.md`
- Existing skill (code-review): `spec/skills/code-review/SKILL.md`
- Existing skill (review-files): `spec/skills/review-files/SKILL.md`
- Code-reviewer agent spec: `spec/agents/code-reviewer.md`
- Naming convention: `agent-docs/canonical-command-conventions.md`
- Schema: `spec/schema/canonical.schema.json`
- README: `README.md` (build commands, directory layout, frontmatter rules)
