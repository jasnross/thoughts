# Add File Path Support to Code Review Skills

## Overview

Update `code-review` and `code-review-loop` skills to accept explicit file
paths as arguments in addition to git references. A new shared fragment
encapsulates the arg-parsing heuristic so both skills stay in sync.

## Current State Analysis

- `code-review` (step 1) parses args only as a git reference (`$BASE_BRANCH`)
- `code-review-loop` (step 1) does the same
- Both already use the `review/prompt-contract` fragment which supports
  `scope="specific files"` with `base_reference="none"` / `merge_base="none"`
- `code-review` already has example prompt contract invocations for specific
  files (lines 99-106) but the arg-parsing logic never routes to that path
- `review-files` exists as a separate skill for glob/directory/file-based
  review and remains unchanged

### Key Discoveries:

- `spec/fragments/review/prompt-contract.md:1-5` — already supports file-focused scope
- `spec/skills/code-review/SKILL.md:99-106` — has example for specific files but no routing
- `spec/fragments/review/merge-base-resolution.md` — merge-base logic that should be skipped for file mode
- Fragment conventions from agent-docs: no leading indentation, trailing newline, single quotes for values containing double quotes

## Desired End State

Both `code-review` and `code-review-loop` accept arguments that are either:
- **File paths**: one or more existing files → file-mode review (no git diff)
- **A git reference**: a branch/tag/SHA → branch-mode review (existing behavior)
- **No arguments**: default branch resolution (existing behavior)

A shared fragment (`review/arg-parsing`) handles the detection heuristic and is
used by both skills in their step 1. When file mode is detected, merge-base
resolution is skipped entirely.

### How to verify:

- `npm --prefix agent-config run build:agents` succeeds with no errors
- Generated output for both skills includes the new arg-parsing instructions
- Generated output for both skills includes file-mode routing logic
- The `review/merge-base-resolution` fragment remains unchanged
- The `review-files` skill remains unchanged

## What We're NOT Doing

- Not adding glob or directory expansion to `code-review`/`code-review-loop`
  (that's `review-files`'s job)
- Not modifying `review-files`
- Not modifying the `code-reviewer` agent spec
- Not modifying `review/prompt-contract` or other existing fragments
- Not adding flags like `--files` — detection is heuristic-based

## Implementation Approach

Create a new fragment that both skills include in their step 1. The fragment
replaces the current "Parse the optional git reference argument" instruction
with a heuristic that checks whether args are file paths or a git reference.
Each skill then branches: file mode skips merge-base resolution and passes
file-specific prompt contract params; branch mode continues as before.

## Phase 1: Create the Shared Arg-Parsing Fragment

### Overview

Create `spec/fragments/review/arg-parsing.md` containing the heuristic
detection logic that both skills will share.

### Changes Required:

#### 1. New Fragment: `spec/fragments/review/arg-parsing.md`

**File**: `spec/fragments/review/arg-parsing.md` (new)

- [x] Create the fragment with the following heuristic logic:

```markdown
Parse the arguments to determine the review mode:

1. If no arguments were provided → **branch mode** (default branch resolution)
2. If arguments were provided, check each one:
   - If any argument contains glob characters (`*`, `?`) → stop and tell the
     user: "Glob patterns are supported by `/review-files`. Use that for
     glob-based review."
   - If an argument exists as a file on disk → it's a **file path**
   - Otherwise → treat as a **git reference**
3. If arguments are a mix of file paths and non-file tokens → stop and tell
   the user: "Arguments appear to be a mix of file paths and git references.
   Please provide either file paths or a single git reference."
4. If all arguments resolved as file paths → **file mode**
   - Set `$REVIEW_MODE = files`
   - Set `$FILE_LIST` to the space-separated list of resolved file paths
5. If a single argument resolved as a git reference → **branch mode**
   - Set `$REVIEW_MODE = branch`
   - Set `$BASE_BRANCH` to the argument
6. If multiple arguments resolved as git references → stop and tell the user:
   "Multiple git references provided. Please provide a single base branch or
   use file paths."
```

### Success Criteria:

#### Automated Verification:

- [x] File exists at `spec/fragments/review/arg-parsing.md`
- [x] Fragment compiles without errors: `npm --prefix agent-config run build:agents`

---

## Phase 2: Update `code-review` Skill

### Overview

Replace the inline arg-parsing in step 1 with the new fragment and add
file-mode routing that skips merge-base resolution and uses file-specific
prompt contract params.

### Changes Required:

#### 1. Update Step 1: Arg Parsing

**File**: `spec/skills/code-review/SKILL.md`

- [x] Replace the current step 1 content:
  ```
  - Check what directory you're in (use `pwd`)
  - Determine if this is a project within a workspace
  - Parse the optional git reference argument (if provided, treat it as `$BASE_BRANCH`)
  ```
  With:
  ```
  - Check what directory you're in (use `pwd`)
  - Determine if this is a project within a workspace
  - {{> review/arg-parsing}}
  ```

#### 2. Add File-Mode Routing to Step 2

**File**: `spec/skills/code-review/SKILL.md`

- [x] Update step 2 to be conditional on `$REVIEW_MODE`:
  - If `$REVIEW_MODE = branch` → proceed with merge-base resolution as before
  - If `$REVIEW_MODE = files` → skip merge-base resolution entirely

#### 3. Update Step 3: Prompt Contract Dispatch

**File**: `spec/skills/code-review/SKILL.md`

- [x] Ensure the prompt contract invocation uses the correct params for each mode:
  - **Branch mode** (existing): `scope="changes against <base-ref>"`,
    `base_reference="<git-ref>"`, `merge_base="<sha>"`, `focus="none"`
  - **File mode** (new): `scope="specific files"`,
    `base_reference="none"`, `merge_base="none"`,
    `focus="<comma-separated file paths from $FILE_LIST>"`

- [x] The existing example blocks (lines 62-107) should be updated to show
  both branch-mode and file-mode examples clearly, with the file-mode examples
  reflecting the new arg-parsing flow

### Success Criteria:

#### Automated Verification:

- [x] Compiles without errors: `npm --prefix agent-config run build:agents`
- [x] Generated output for `code-review` includes the arg-parsing heuristic
- [x] Generated output for `code-review` includes file-mode routing
- [x] Generated output for `code-review` retains branch-mode behavior

---

## Phase 3: Update `code-review-loop` Skill

### Overview

Apply the same arg-parsing and routing changes to `code-review-loop`, with
an additional adjustment to the state file initialization to reflect file-based
scope.

### Changes Required:

#### 1. Update Step 1: Arg Parsing

**File**: `spec/skills/code-review-loop/SKILL.md`

- [x] Replace the current step 1 content (same change as `code-review`):
  ```
  - Check what directory you're in (use `pwd`)
  - Determine if this is a project within a workspace
  - Parse the optional git reference argument (if provided, treat it as `$BASE_BRANCH`)
  ```
  With:
  ```
  - Check what directory you're in (use `pwd`)
  - Determine if this is a project within a workspace
  - {{> review/arg-parsing}}
  ```

#### 2. Add File-Mode Routing to Step 2

**File**: `spec/skills/code-review-loop/SKILL.md`

- [x] Update step 2 to be conditional on `$REVIEW_MODE`:
  - If `$REVIEW_MODE = branch` → proceed with merge-base resolution as before
  - If `$REVIEW_MODE = files` → skip merge-base resolution entirely

#### 3. Update Step 3: State File Initialization

**File**: `spec/skills/code-review-loop/SKILL.md`

- [x] Update the state file initialization to reflect file-based scope when
  in file mode:
  - **Branch mode** (existing):
    ```markdown
    Scope: changes against <BASE_BRANCH> (merge-base: <MERGE_BASE>)
    ```
  - **File mode** (new):
    ```markdown
    Scope: specific files — <comma-separated file paths>
    ```

#### 4. Update Step 4b: Prompt Contract Dispatch

**File**: `spec/skills/code-review-loop/SKILL.md`

- [x] Ensure the prompt contract invocation in step 4b uses the correct
  params for each mode (same logic as `code-review` phase 2, step 3)

### Success Criteria:

#### Automated Verification:

- [x] Compiles without errors: `npm --prefix agent-config run build:agents`
- [x] Generated output for `code-review-loop` includes the arg-parsing heuristic
- [x] Generated output for `code-review-loop` includes file-mode routing
- [x] Generated output for `code-review-loop` retains branch-mode behavior
- [x] State file template includes file-mode scope variant

---

## Phase 4: Build and Validate

### Overview

Run the full build pipeline and verify all generated output is correct.

### Changes Required:

- [x] Run `npm --prefix agent-config run build:agents`
- [x] Verify no warnings or errors
- [x] Spot-check generated output for both skills to confirm:
  - Arg-parsing fragment is expanded inline (no `{{>` references remain)
  - Branch-mode examples are preserved
  - File-mode routing is present
  - Prompt contract params are correct for both modes

### Success Criteria:

#### Automated Verification:

- [x] `npm --prefix agent-config run build:agents` exits 0
- [x] No `{{>` or `{{` syntax remains in generated output for either skill

#### Manual Verification:

- [ ] Generated `code-review` skill reads naturally with both modes documented
- [ ] Generated `code-review-loop` skill reads naturally with both modes documented
- [ ] No other generated skills or agents were unexpectedly modified

---

## Testing Strategy

### Manual Testing Steps:

1. Invoke `/code-review src/some-file.ts` — should enter file mode
2. Invoke `/code-review main` — should enter branch mode (existing behavior)
3. Invoke `/code-review` with no args — should use default branch (existing behavior)
4. Invoke `/code-review src/*.ts` — should suggest using `/review-files`
5. Invoke `/code-review-loop src/some-file.ts` — should enter file mode with correct state file
6. Invoke `/code-review-loop` with no args — should use default branch (existing behavior)

## References

- Existing file-mode skill: `spec/skills/review-files/SKILL.md`
- Prompt contract fragment: `spec/fragments/review/prompt-contract.md`
- Merge-base fragment: `spec/fragments/review/merge-base-resolution.md`
- Code-reviewer agent: `spec/agents/code-reviewer.md`
- Fragment conventions: `agent-docs/compiler-fragments.md`
