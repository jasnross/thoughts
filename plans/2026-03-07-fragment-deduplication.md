# Deduplicate Spec Files Using Handlebars Fragments

## Overview

Extract 11 shared instruction blocks from agent and skill specs into reusable Handlebars fragments. This eliminates wording drift, ensures consistency for critical logic (hunk algorithms, merge-base resolution), and reduces maintenance burden when updating shared concerns. The fragment infrastructure and first extraction (`review/prompt-contract`) are already in place — this plan focuses on the remaining candidates.

## Current State Analysis

The codebase has 8 agents and 26 skills in `spec/`. One fragment exists (`review/prompt-contract`), used by 4 review skills. The research document (`thoughts/research/2026-03-07-spec-duplication-fragment-candidates.md`) identified 13 high-value candidates. Two are being dropped (see below), leaving 11 to implement.

### Key Discoveries

- Fragment compiler is production-ready: `scripts/lib/fragments.ts` (72 lines), supports nested partials, hash arguments, `{{#if}}` conditionals, and `noEscape` mode
- Test suite at `scripts/__tests__/fragments.test.ts` (179 lines) covers all compiler behaviors
- `implement-plan` vs `implement-plan-worktree` share ~100 lines with only `<project-dir>` vs `<worktree-path>` differences plus em-dash drift
- `create-pr` vs `describe-pr` share ~57 lines of PR inline review logic, character-for-character identical except for `$PR_NUMBER` vs `{number}` and review body attribution text
- Host detection bash block is byte-for-byte identical across 5 skills
- Merge-base resolution in `code-review-loop` explicitly says "Use the same resolution logic as `code-review`"
- `research-codebase` and `iterate-research` share identical documentarian mandate, sub-agent catalog, and document convention blocks
- `review-annotations` and `review-pr-comments` differ only by `item_type` ("annotation" vs "thread") across ~50 shared lines

## Desired End State

After this plan is complete:

1. 12 new fragment files exist in `spec/fragments/` organized by cluster: `github/`, `plans/`, `research/`, `review/`, `agents/` (11 candidates produce 12 files because `github/host-detection` splits into an inner/outer pair)
2. All spec files that previously contained duplicated blocks now reference the corresponding fragment via `{{> category/name}}`
3. Generated output is functionally identical to pre-change output (drift corrections are intentional improvements, not behavioral changes)
4. Wording drift between related specs is eliminated for all shared concerns
5. Two candidates (`agents/no-critique-bullets`, `agents/file-line-references`) are explicitly documented as dropped with rationale

### Verification

- `npm --prefix agent-config run build:agents` succeeds with no errors
- `npm --prefix agent-config test` passes all tests
- `git diff generated/` shows only intentional drift corrections (em-dash normalization, minor wording improvements), never behavioral changes

## What We're NOT Doing

- **Small cross-cutting patterns**: "No attribution", "never git add -A", "read files fully" — these are 1-3 line patterns better served by template variables in a future effort
- **New instructions or behavioral changes**: Fragment content matches existing spec text with drift reconciled to the best version. The exception is **consistency normalization** — when a shared concern (e.g., delegation rules) exists in some skills but is missing from others, the fragment adds it to all consumers. These additions are explicitly called out in the plan as "intentional improvements."
- **`agents/no-critique-bullets` fragment**: Each agent's "What NOT to Do" section is intentionally tailored (analyzer has 12 bullets, locator has 10, pattern-finder has 10 with entirely different wording). The variation is by design, not drift.
- **`agents/file-line-references` fragment**: Single line with intentionally different phrasing per context. Extraction overhead exceeds value.
- **Compiler changes**: The compiler already supports everything needed (conditionals, variables, nested partials)

## Implementation Approach

Fragments are organized into 4 phases by cluster. Each phase creates all fragments for a cluster, migrates consuming specs, verifies the build, and pauses for review. This grouping minimizes context switching and lets each phase's build verification confirm a coherent set of changes.

**Drift reconciliation principle**: When duplicated text has drifted, the best version becomes the canonical fragment content. Em-dashes (`—`) are preferred over hyphens (`-`) for consistency. `[N]` is preferred over `[Number]` for brevity. Trailing clauses that add clarity are kept.

---

## Phase 1: GitHub & PR Operations

### Overview

Extract 4 fragments (including a nested inner/outer pair) for GitHub-related shared content. This cluster has the broadest file coverage (8 files) and includes the critical hunk line-number algorithm where correctness matters most.

### Changes Required

#### 1. Create `github/host-detection-code` inner fragment

**File**: `spec/fragments/github/host-detection-code.md`

- [x] Create fragment with just the bash case statement (no heading, no trailing prose):

  ````markdown
  ```bash
  REMOTE_URL=$(git remote get-url origin)
  case "$REMOTE_URL" in
    *git.taservs.net*) GH_HOST=git.taservs.net ;;
    *) GH_HOST=github.com ;;
  esac
  ```
  ````

  This inner fragment is the single source of truth for the host detection logic. It is used directly by `split-branch` and `gh-safe` (which have their own surrounding prose), and nested inside the outer `github/host-detection` fragment for the standard case.

#### 2. Create `github/host-detection` outer fragment

**File**: `spec/fragments/github/host-detection.md`

- [x] Create fragment that nests the inner fragment with the code block and trailing instruction (no heading):

  ```markdown
  {{> github/host-detection-code}}

  Store `$GH_HOST` and pass it as the first argument to all `gh-safe` calls.
  ```

  > **Deviation:** Removed the `### Detect the GitHub host` heading from the outer fragment. Each skill uses a different heading format (numbered list bold text vs `###` headings), so the heading stays inline in each skill and the fragment provides only the shared content (code block + trailing instruction).

  **Design note**: The gh-safe load instruction that precedes host detection varies meaningfully across skills (e.g., `create-pr` says "read-only", others say "all `gh` operations"). Only the host detection block + trailing instruction are extracted into the outer fragment. The gh-safe load instruction stays inline in each skill.

#### 3. Update skills to use host-detection fragments

- [x] **`spec/skills/create-pr/SKILL.md`**: Keep step 2 heading inline, replace bash block + trailing sentence with `{{> github/host-detection}}`
- [x] **`spec/skills/describe-pr/SKILL.md`**: Keep step 2 heading inline, replace bash block + trailing sentence with `{{> github/host-detection}}`
- [x] **`spec/skills/review-pr/SKILL.md`**: Keep `### 2. Detect the GitHub host` heading inline, replace content with `{{> github/host-detection}}`
- [x] **`spec/skills/review-pr-comments/SKILL.md`**: Keep `### Detect the GitHub host` heading inline, replace content with `{{> github/host-detection}}`
- [x] **`spec/skills/split-branch/SKILL.md`**: Replace the inline bash case statement with `{{> github/host-detection-code}}`, keeping the surrounding subagent context prose inline
- [x] **`spec/skills/gh-safe/SKILL.md`**: Replace the bash case statement under "Detecting the host for the current repository" with `{{> github/host-detection-code}}`, keeping the surrounding reference documentation prose inline

#### 4. Create `github/pr-description-rules` fragment

**File**: `spec/fragments/github/pr-description-rules.md`

- [x] Create fragment with this content, reconciling drift between `create-pr` and `describe-pr`. The canonical version uses `describe-pr`'s broader phrasing ("This command works across different repositories") and `create-pr`'s "Do NOT include Claude attribution" bullet:

  ```markdown
  - Be thorough but concise — descriptions should be scannable
  - Do NOT use hard line breaks (manual newlines) within prose paragraphs or bullet points in the PR description. Write each paragraph or bullet as a single unwrapped line and let the GitHub UI handle text wrapping. Hard line breaks create awkward mid-sentence breaks at different viewport widths.
  - Focus on the "why" as much as the "what"
  - Include any breaking changes or migration notes prominently
  - If the PR touches multiple components, organize the description accordingly
  - Do NOT include Claude attribution or co-author information in any output
  ```

#### 5. Update skills to use `github/pr-description-rules`

- [x] **`spec/skills/create-pr/SKILL.md`**: Replace the matching bullets in the "Important notes" section with `{{> github/pr-description-rules}}`. Keep skill-specific bullets inline.
- [x] **`spec/skills/describe-pr/SKILL.md`**: Replace the matching bullets in the "Important notes" section with `{{> github/pr-description-rules}}`. Keep skill-specific bullets inline. Also fixed em-dash drift ("repositories - always" → "repositories — always").

#### 6. Create `github/pr-inline-review` fragment

**File**: `spec/fragments/github/pr-inline-review.md`

- [x] Create fragment covering the full inline review workflow shared between `create-pr` and `describe-pr`. This includes:
  - Comment-worthy locations list
  - CRITICAL hunk line-number algorithm
  - Per-comment instructions
  - `diff-lines` validation step
  - JSON payload example
  - `post-review` command + trailing skip note

  Variables:
  - `{{review_body_text}}` — `Automated review notes from create-pr` or `from describe-pr`

  **Normalization**: Both `create-pr` and `describe-pr` will use `$PR_NUMBER` as the shell variable for the PR number. `describe-pr` currently uses `{number}` placeholders but already resolves the PR number in step 4. As a pre-step before fragment extraction, normalize `describe-pr` to capture `PR_NUMBER` and use `$PR_NUMBER` consistently. This eliminates the need for a `{{pr_number_ref}}` variable — the fragment uses `$PR_NUMBER` directly.

  **Design note**: `gh-safe/SKILL.md` has a condensed version of the hunk algorithm (~12 lines vs ~57 lines) in a different context (reference documentation, not workflow steps). It stays inline because extracting it would require either a "god fragment" with extensive conditionals or a separate fragment for just the algorithm — and gh-safe's version is intentionally condensed for its reference-doc format.

#### 7. Update skills to use `github/pr-inline-review`

- [x] **`spec/skills/describe-pr/SKILL.md`** (pre-step): Normalized PR number references from `{number}` to `$PR_NUMBER` / `${PR_NUMBER}` throughout the file. Added `PR_NUMBER` capture step in step 4. Used `${PR_NUMBER}` (with braces) in file path contexts to prevent variable name ambiguity with adjacent underscores.
- [x] **`spec/skills/create-pr/SKILL.md`**: Replaced the inline review section (step 13 content) with `{{> github/pr-inline-review review_body_text="Automated review notes from create-pr"}}`. Kept the step number heading and introductory sentence inline.
- [x] **`spec/skills/describe-pr/SKILL.md`**: Replaced the inline review section (step 12 content) with `{{> github/pr-inline-review review_body_text="Automated review notes from describe-pr"}}`. Kept the step number heading and introductory sentence inline.

#### 8. Build and verify

- [x] Run `npm --prefix agent-config run build:agents`
- [x] Run `npm --prefix agent-config test`
- [x] Review `git diff generated/` — changes are: trailing whitespace on blank lines (from Handlebars partial indentation), review-pr lost one introductory line ("Determine which GitHub host this repo targets:"), describe-pr `{number}` → `$PR_NUMBER`/`${PR_NUMBER}` normalization, added "Do NOT include Claude attribution" bullet to describe-pr, em-dash fixes

### Success Criteria

#### Automated Verification

- [x] `npm --prefix agent-config run build:agents` succeeds
- [x] `npm --prefix agent-config test` passes (53 tests, including 2 new conditional tests)
- [x] Generated output diff shows only intentional drift corrections

#### Manual Verification

- [x] Review each fragment file for clarity and completeness
- [x] Spot-check one generated create-pr output to confirm inline review section renders correctly
- [x] Verify `gh-safe/SKILL.md` was NOT modified (its condensed version is intentionally different)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 2.

---

## Phase 2: Implementation & Research

### Overview

Extract 3 fragments for the implementation workflow and research convention patterns. This phase tackles the largest single duplication (`plans/implementation-workflow` at ~100 lines) and the research document conventions shared across 3 skills.

### Changes Required

#### 1. Create `plans/implementation-workflow` fragment

**File**: `spec/fragments/plans/implementation-workflow.md`

- [x] Create a single fragment covering all 5 shared sections: Implementation Philosophy, Mismatch template, Verification Approach, Updating the Plan (with Mark Completion, Document Deviations, Keep It Useful sub-sections), and If You Get Stuck.

  Variables:
  - `{{project_path_label}}` — `<project-dir>` in implement-plan, `<worktree-path>` in implement-plan-worktree

  Conditional:
  - `{{#if worktree}}` — gates the worktree-specific note about using absolute paths for plan file access in the "Updating the Plan" section

  **Drift reconciliation decisions**:
  - Em-dashes (`—`) everywhere (implement-plan has mixed hyphens; worktree is consistent)
  - `[N]` not `[Number]` for count placeholders (shorter, used by worktree version)
  - Keep "future readers benefit from knowing why" clause from implement-plan (adds clarity)
  - Use `(see [Updating the Plan](#updating-the-plan))` cross-reference and `[Updating the Plan](#updating-the-plan)` link unconditionally in the fragment. The anchor `#updating-the-plan` is derived from heading text, not level, so it resolves correctly in both skills regardless of whether the heading is `##` or `###`. As a pre-step, normalize `implement-plan-worktree` to use the same link syntax (it currently uses plain text "the Updating the Plan section below").
  - Keep "Plans use `- [ ]` checkboxes at every level — implementation steps, sub-phase tasks, automated verification, and manual verification" (full sentence from implement-plan; worktree's shorter version loses useful detail)
  - Use em-dash `—` not double-hyphen `--` in the Deviation example

  **Heading levels** (heading-level parameterization deferred — only one fragment has this problem; revisit if future fragments need it): The fragment uses `##`/`###` headings (matching implement-plan's current structure). Handlebars does not adjust heading levels, so both consuming skills receive the same levels.

  For `implement-plan`, this matches the current output exactly — no change.

  For `implement-plan-worktree`, the fragment's `##` headings will become peers of the skill's own `## Implementation (inside worktree)` section rather than children. This is a structural change to worktree's generated output. **Decide during implementation**: if the peer-heading structure reads acceptably in the generated output, keep the single fragment. If it's confusing, split into 5 sub-fragments (one per section: Philosophy, Mismatch, Verification, Updating the Plan, If You Get Stuck) so each skill can include them at the appropriate heading level. The research analysis suggested this as a fallback.

  > **Decision:** Kept the single fragment with `##`/`###` headings. The peer-heading structure reads acceptably — the sections are logically independent concerns that flow naturally as peers of `## Implementation (inside worktree)`. The `## If You Get Stuck` section was already at `##` level in the worktree version, so only Philosophy, Verification, and Updating the Plan changed from `###` to `##`.

#### 2. Update implement-plan skills

- [x] **`spec/skills/implement-plan/SKILL.md`**: Replace lines ~51-164 (Implementation Philosophy through the end of Resuming Work preamble) with:

  ```
  {{> plans/implementation-workflow project_path_label="<project-dir>"}}
  ```

  Keep the "Getting Started" and "Resuming Work" sections inline — they differ for good reason. Getting Started has different setup flows (implement-plan front-loads everything; worktree delegates to Project Detection / Pre-flight / Worktree Creation). Resuming Work has a small shared core (~3 lines) but worktree wraps it with `cd "$WORKTREE_PATH"`. As a minor cleanup, add implement-plan's motivational line ("Remember: You're implementing a solution, not just checking boxes...") to implement-plan-worktree's Resuming Work section — that's drift, not intentional omission.

- [x] **`spec/skills/implement-plan-worktree/SKILL.md`**: Replace lines ~153-297 (Implementation Philosophy through If You Get Stuck) with:
  ```
  {{> plans/implementation-workflow project_path_label="<worktree-path>" worktree=true}}
  ```
  Keep the worktree-specific sections (Project Detection, Pre-flight Checks, Worktree Creation, Completion and Cleanup, Resuming Work, Important Notes) inline.

#### 3. Create `research/document-conventions` fragment

**File**: `spec/fragments/research/document-conventions.md`

- [x] Create fragment covering the shared "Important notes" closing rules. These appear in `research-codebase`, `research-web`, and `iterate-research` as two sub-sections: "Critical ordering" and "Frontmatter consistency".

  **Pre-step — normalize step references**: Before extracting, update all 3 consuming skills to replace step-number cross-references with descriptive language. For example, "Always read mentioned files fully (step 2)" becomes "Always read mentioned files fully before dispatching agents." This eliminates fragile step-number coupling — the fragment stays correct even when a skill's step ordering changes.

  Conditional:
  - `{{#if include_git_commit_example}}` — research-codebase and iterate-research include `git_commit` in the snake_case example; research-web omits it
  - `{{#if include_iterate_rules}}` — gates 3 extra bullets in iterate-research: "Preserve original question", "Overwrite, don't create new", "Track lineage"

  **Drift reconciliation**: Use `research-codebase`'s version as the base (most complete). research-web's version is a strict subset.

#### 4. Update research skills

- [x] **`spec/skills/research-codebase/SKILL.md`**: Normalize step-number references to descriptive language, then replace the "Critical ordering" and "Frontmatter consistency" blocks in "Important notes" (lines ~211-221) with:

  ```
  {{> research/document-conventions include_git_commit_example=true}}
  ```

  Keep the other "Important notes" bullets inline (they're skill-specific).

- [x] **`spec/skills/research-web/SKILL.md`**: Normalize step-number references to descriptive language, then replace the matching blocks in "Important notes" (lines ~177-187) with:

  ```
  {{> research/document-conventions}}
  ```

- [x] **`spec/skills/iterate-research/SKILL.md`**: Normalize step-number references to descriptive language, then replace the matching blocks in "Important notes" (lines ~256-265) with:
  ```
  {{> research/document-conventions include_git_commit_example=true include_iterate_rules=true}}
  ```

#### 5. Create `research/sub-agent-catalog` fragment

**File**: `spec/fragments/research/sub-agent-catalog.md`

- [x] Create fragment covering the shared sub-agent dispatch block from research-codebase and iterate-research. This includes the "For codebase research" agent list, the "IMPORTANT: All agents are documentarians" note, the "For web research" section, and the "use these agents intelligently" guidelines.

  Conditional:
  - `{{#if include_thoughts_agents}}` — gates the thoughts-locator and thoughts-analyzer entries that `create-plan` additionally lists

  **Note**: `create-plan` is a skill defined in the thoughts-workflow plugin, not in this repo. This fragment will serve `research-codebase` and `iterate-research` immediately. `create-plan` can adopt it later when thoughts-workflow is updated.

#### 6. Update research skills

- [x] **`spec/skills/research-codebase/SKILL.md`**: Replace the sub-agent dispatch block (lines ~76-96) with:
  ```
  {{> research/sub-agent-catalog}}
  ```
- [x] **`spec/skills/iterate-research/SKILL.md`**: Replace the sub-agent dispatch block (lines ~108-129) with:
  ```
  {{> research/sub-agent-catalog}}
  ```

#### 7. Build and verify

- [x] Run `npm --prefix agent-config run build:agents`
- [x] Run `npm --prefix agent-config test`
- [x] Review `git diff generated/` — drift corrections confirmed: em-dash normalization, `[Number]` → `[N]`, step-number references → descriptive language, heading level promotion in worktree, added cross-reference links, "future readers benefit from knowing why" clause, motivational line added to worktree Resuming Work, trailing whitespace on blank lines from Handlebars indentation. No behavioral changes.

### Success Criteria

#### Automated Verification

- [x] `npm --prefix agent-config run build:agents` succeeds
- [x] `npm --prefix agent-config test` passes (53 tests)
- [x] Generated output diff shows only intentional drift corrections

#### Manual Verification

- [x] Review `plans/implementation-workflow` fragment to confirm heading levels work correctly in both consuming skills
- [x] Spot-check generated implement-plan-worktree output to verify worktree-specific conditional content appears
- [x] Verify research skills' "Important notes" sections still read naturally with the fragment invocation

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 3.

---

## Phase 3: Review Workflow

### Overview

Extract 4 fragments for review orchestration patterns. These fragments consolidate the delegation rules, merge-base resolution, triage workflow, and feedback processing shared across review skills.

### Changes Required

#### 1. Create `review/merge-base-resolution` fragment

**File**: `spec/fragments/review/merge-base-resolution.md`

- [x] Create fragment covering the full merge-base resolution logic from `code-review`: both the "user-passed arg" branch and the "no arg" branch. Use `code-review-loop`'s unified format for the no-arg block (4 lines in one block rather than code-review's split presentation).

  Content covers:
  - If user passed a git reference: check for remote-tracking version, fetch, compute merge-base
  - If no argument: detect base branch from `refs/remotes/origin/HEAD`, fall back to `main`, fetch, compute merge-base
  - Trailing note about using `$MERGE_BASE` as the canonical baseline

  **Design decision**: `review-pr` keeps its own 2-line inline version (`git fetch origin "$PR_BASE_BRANCH"` / `MERGE_BASE=$(git merge-base HEAD "origin/$PR_BASE_BRANCH")`). Its context is fundamentally different — it already knows the base branch from PR metadata and doesn't need the detection/fallback logic. Forcing it into a fragment with conditionals would add complexity for no benefit.

#### 2. Update review skills

- [x] **`spec/skills/code-review/SKILL.md`**: Replace step 2 "Resolve base branch and merge-base commit" content (lines ~38-66) with:

  ```
  {{> review/merge-base-resolution}}
  ```

  Keep the step number heading inline.

- [x] **`spec/skills/code-review-loop/SKILL.md`**: Replace step 2 "Resolve base branch and merge-base" content (lines ~46-71) with:
  ```
  {{> review/merge-base-resolution}}
  ```
  Keep the step number heading and "Use the same resolution logic as `code-review`:" intro inline — or remove the intro since the fragment IS that shared logic now.

#### 3. Create `review/delegation-rules` fragment

**File**: `spec/fragments/review/delegation-rules.md`

- [x] Create fragment covering the shared delegation instruction. Canonical version (reconciling drift):

  ```markdown
  - **DO NOT** perform the review yourself — always spawn the code-reviewer agent
  - The agent has specialized review logic, evaluation criteria, and output formatting
  - The triage phase is collaborative — do not auto-fix findings without user agreement on each one
  ```

  **Drift reconciliation**: Use em-dash (`—`) everywhere (most files already use it; `code-review` uses a hyphen). Keep the 3 core bullets that are truly shared. The "thin orchestrator" description varies per skill (code-review says "makes it easy for users to get reviews", review-files says "resolve files, guard the count, then delegate") — leave these skill-specific descriptions inline.

#### 4. Update review skills

- [x] **`spec/skills/code-review/SKILL.md`**: Replace the matching 3 bullets in "Important Notes" (lines ~161-164) with `{{> review/delegation-rules}}`. Keep the skill-specific "thin orchestrator" bullet inline.
      **Note**: code-review currently has 4 bullets; only the 3 shared ones are replaced. The orchestrator description stays.

- [x] **`spec/skills/code-review-loop/SKILL.md`**: Replace the first bullet in "Important Notes" (line ~274) with `{{> review/delegation-rules}}`. Keep the 5 loop-specific bullets inline.
      **Note**: code-review-loop only shares the first bullet ("DO NOT perform the review yourself"). The delegation fragment replaces that one bullet and adds the 2 other shared bullets that code-review-loop was missing. This is an intentional improvement — code-review-loop should have the same delegation rules as the other review skills.

- [x] **`spec/skills/review-files/SKILL.md`**: Replace the matching 3 bullets in "Important Notes" (lines ~105-108) with `{{> review/delegation-rules}}`. Keep the skill-specific orchestrator bullet inline.

- [x] **`spec/skills/review-pr/SKILL.md`**: Replace the first bullet in "Important Notes" (line ~236) with `{{> review/delegation-rules}}`. Keep the PR-specific bullets inline.
      **Note**: review-pr currently shares only the first bullet. Adding the full delegation rules is an intentional improvement for consistency.

#### 5. Create `review/triage-workflow` fragment

**File**: `spec/fragments/review/triage-workflow.md`

- [x] Create fragment covering the shared triage workflow from `code-review` and `review-files`. Both skills will include TODO tracking unconditionally — `review-files` currently omits it, but this is drift, not intentional. TODO tracking prevents findings from being lost during lengthy triage sessions. Canonical version (no conditionals):

  ```markdown
  - Immediately transition into addressing findings — don't wait for the user to ask
  - Create a todo for each finding so none are lost during triage
  - Work through each finding in severity order: Blockers → Major → Minor
  - For each finding:
    - Mark the todo as in-progress
    - Present the issue briefly
    - Ask the user: fix now, skip, or defer?
    - If fixing, make the change, then mark the todo as completed
    - If skipped or deferred, mark the todo as completed with a note
  - For Questions/Assumptions and Verification Gaps, discuss briefly for alignment
    but no code changes are needed — still track and complete their todos
  - If the review found no issues, skip this step
  ```

  **Design decision**: `review-pr` is excluded — its triage uses Include/Edit/Exclude options for building a PR review payload, which is a fundamentally different paradigm from fix/skip/defer. Forcing both into one fragment would create a "god fragment".

#### 6. Update triage skills

- [x] **`spec/skills/code-review/SKILL.md`**: Replace step 7 "Work through findings collaboratively" content (lines ~146-157) with:

  ```
  {{> review/triage-workflow}}
  ```

  Keep the step heading inline.

- [x] **`spec/skills/review-files/SKILL.md`**: Replace step 7 "Work through findings collaboratively" content (lines ~93-101) with:
  ```
  {{> review/triage-workflow}}
  ```
  Keep the step heading inline.

#### 7. Create `review/feedback-processing` fragment

**File**: `spec/fragments/review/feedback-processing.md`

- [x] Create fragment covering the shared feedback processing workflow from `review-annotations` and `review-pr-comments`. This includes:
  - Complexity assessment criteria (architectural changes, multi-file modifications, unclear requirements, substantial refactoring)
  - Plan mode offer template
  - User response options list
  - Resolution confirmation template
  - Important Notes behavioral bullets (don't be defensive, ask for clarification, propose concrete solutions, one at a time)

  Variables:
  - `{{item_type}}` — `annotation` or `thread` (controls wording throughout)
  - `{{item_type_plural}}` — `annotations` or `threads`

  Conditional:
  - `{{#if pr_specific}}` — gates `review-pr-comments`-specific bullets in Important Notes ("Local changes only", "Respect outdated threads")

  **Drift reconciliation**:
  - Opening behavioral bullet: use "Address feedback constructively" (review-pr-comments' version is more professional than review-annotations' "Be receptive to feedback")
  - "One at a time" bullet: normalize to "One {{item_type}} at a time — don't rush; each {{item_type}} deserves focused attention"
  - User response options: use review-pr-comments' slightly more detailed versions ("Ask you to implement the proposed fix", "Provide additional context or a different approach", "Say it's already resolved")

#### 8. Update feedback skills

- [x] **`spec/skills/review-annotations/SKILL.md`**: Replace the "Assessing Complexity" section through "Important Notes" (lines ~95-166) with:

  ```
  {{> review/feedback-processing item_type="annotation" item_type_plural="annotations"}}
  ```

  Keep the preceding sections (Find Annotations, Create TODOs, Work Through Each Annotation) inline — they have annotation-specific structure that differs from the PR comments version.

  > **Deviation:** The plan said to keep Confirm Resolution and Batch Removal inline, but Confirm Resolution is truly shared content (identical logic in both skills). It was included in the fragment. Batch Removal stays inline AFTER the fragment output, which reorders Important Notes to appear before Batch Removal. This is acceptable since Important Notes contains behavioral guidelines that should be read before the procedural cleanup step. The "Step 4:"/"Step 5:" prefixes were dropped from remaining sections since the fragment breaks the original numbering.

- [x] **`spec/skills/review-pr-comments/SKILL.md`**: Replace the "Assessing Complexity" section through "Important Notes" (lines ~163-239) with:
  ```
  {{> review/feedback-processing item_type="thread" item_type_plural="threads" pr_specific=true}}
  ```
  Keep the preceding sections (Identify PR, Create TODOs, Work Through Each Thread, Show local code context, Handle divergence, Analyze and propose) inline. Final Summary + Next steps stays inline AFTER the fragment output.

  > **Deviation:** Same structural approach as review-annotations — Confirm Resolution is shared and included in the fragment. Final Summary + Next steps is skill-specific and placed after the fragment. "Step 4:"/"Step 5:" prefixes dropped.

#### 9. Build and verify

- [x] Run `npm --prefix agent-config run build:agents`
- [x] Run `npm --prefix agent-config test`
- [x] Review `git diff generated/` — 53 generated files changed (-4 net lines). Drift corrections: em-dash normalization, added delegation bullets in code-review-loop and review-pr, TODO tracking added to review-files triage, wording improvements in feedback skills ("Address feedback constructively", more detailed user response options), reordered Important Notes before Batch Removal/Final Summary in annotation/thread skills.

### Success Criteria

#### Automated Verification

- [x] `npm --prefix agent-config run build:agents` succeeds
- [x] `npm --prefix agent-config test` passes (53 tests)
- [x] Generated output diff shows only intentional drift corrections and consistency improvements

#### Manual Verification

- [x] Review merge-base fragment to confirm it handles both user-arg and no-arg branches clearly
- [x] Review triage-workflow fragment to confirm conditional todo tracking renders correctly
- [x] Review feedback-processing fragment to confirm `item_type` substitutions read naturally
- [x] Verify review-pr was NOT modified for triage (it uses a different paradigm)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 4.

---

## Phase 4: Agent Behavioral Contracts

### Overview

Evaluate and potentially extract 1 fragment for the documentarian mandate shared across agent specs and research skills. This phase also formally documents the decision to drop 2 candidates.

### Changes Required

#### 1. Evaluate `agents/documentarian-mandate` fragment

- [x] Read the 5 consuming files side-by-side and decide between two options:

  **Option A — Extract heading + 3 shared bullets**:
  - Create `spec/fragments/agents/documentarian-mandate.md` containing the `## CRITICAL: YOUR ONLY JOB IS TO {{role_description}}` heading and the 3 universally shared "DO NOT" bullets (suggest improvements, root cause analysis, propose future enhancements)
  - Variable: `{{role_description}}` — `DOCUMENT AND EXPLAIN THE CODEBASE AS IT EXISTS TODAY` (default) or `DOCUMENT AND SHOW EXISTING PATTERNS AS THEY ARE` (pattern-finder)
  - Each consumer adds its role-specific bullets inline after the fragment invocation
  - For the REMEMBER closing block: extract only the shared heading `## REMEMBER: You are a documentarian, not a critic or consultant`; the paragraph text is intentionally different per agent, so it stays inline
  - The research skills (`research-codebase`, `iterate-research`) are the strongest case — they share the block identically

  **Option B — Don't extract (drop candidate)**:
  - The overlap is only 3-4 lines across files that are already architecturally different
  - Document the rationale in a deviation note, alongside the other 2 dropped candidates
  - This is a valid outcome — not every candidate justifies extraction

#### 2. If proceeding with extraction (Option A)

> **Decision: Option B chosen — skipping this step.** See step 3 for rationale.

- ~~**`spec/agents/codebase-analyzer.md`**~~
- ~~**`spec/agents/codebase-locator.md`**~~
- ~~**`spec/agents/codebase-pattern-finder.md`**~~
- ~~**`spec/skills/research-codebase/SKILL.md`**~~
- ~~**`spec/skills/iterate-research/SKILL.md`**~~

#### 3. Document dropped candidates

- [x] Add a note in the idea file or a deviation note in this plan documenting:
  - **`agents/no-critique-bullets` (candidate 12)**: Dropped. Each agent's "What NOT to Do" section is intentionally tailored to its specific role. codebase-analyzer has 12 prohibitions including security and performance; codebase-locator has 10 focused on structure; codebase-pattern-finder has 10 entirely reworded for patterns. The variation is by design. Extracting would require extensive parameterization that defeats the purpose.
  - **`agents/file-line-references` (candidate 13)**: Dropped. Single line per file with intentionally different phrasing: "Always include file:line references for claims" (analyzer), "Include file:line references" + "Full file paths — With line numbers" (pattern-finder, two separate locations), "Include specific file paths and line numbers for reference" (research skills). The variation reflects each agent's context. Fragment overhead exceeds value.
  - **`agents/documentarian-mandate` (candidate 11)**: Dropped (Option B). Only 3 bullets shared across 5 codebase-focused files; `research-web` shares nothing (entirely different role description and bullets). The 3 agents each have 5-7 role-specific "DO NOT" bullets with intentionally different wording. REMEMBER closing paragraphs are deliberately unique per agent (map/navigator, technical writer, pattern librarian). Only `research-codebase` and `iterate-research` are identical, but a fragment for 2 consumers of 7 lines adds overhead without meaningful drift protection.

#### 4. Build and verify

- [x] Run `npm --prefix agent-config run build:agents` — already passing from Phase 3; no code changes in Phase 4
- [x] Run `npm --prefix agent-config test` — already passing from Phase 3; no code changes in Phase 4
- [x] Review `git diff generated/` — no additional changes (Option B = no extraction)

### Success Criteria

#### Automated Verification

- [x] `npm --prefix agent-config run build:agents` succeeds
- [x] `npm --prefix agent-config test` passes
- [x] Generated output diff shows no additional changes (Option B chosen, no extraction)

#### Manual Verification

- [x] Fragment was dropped: deviation note documents the rationale (see step 3 above)
- [x] No agent's "What NOT to Do" section was inadvertently modified (no spec changes in Phase 4)

**Implementation Note**: This is the final phase. All 13 candidates from the research document have been evaluated — 10 implemented as 12 fragment files (including the `github/host-detection-code` inner/outer pair), 3 explicitly dropped with rationale (`documentarian-mandate`, `no-critique-bullets`, `file-line-references`).

---

## Testing Strategy

### Unit Tests

- Existing fragment tests (`scripts/__tests__/fragments.test.ts`) cover the compiler mechanics: variable substitution, missing partials, isolation, no-escape, nested partials
- **Risk mitigation**: The test suite does not cover `{{#if}}` conditional behavior (only syntax errors). Four fragments rely on conditionals (`plans/implementation-workflow`, `research/document-conventions`, `research/sub-agent-catalog`, `review/feedback-processing`). Add a conditional test case to `fragments.test.ts` before or during Phase 1 to verify that `{{#if variable}}` works correctly with hash arguments passed to partials.
- New fragments are tested implicitly through `build:agents` — if a fragment has a syntax error or missing variable, compilation fails

### Integration Tests

- `npm --prefix agent-config run build:agents` is the primary integration test — it runs validate + compile + check
- `git diff generated/` after each phase verifies output correctness
- `npm --prefix agent-config run check:agents` verifies generated files match compiled state

### Manual Testing Steps

1. After each phase, review `git diff generated/` for unexpected changes
2. Spot-check one generated file per fragment to confirm the resolved content appears correctly
3. Verify all fragment files follow conventions: no leading indentation, trailing newline, no `{{` outside intentional Handlebars syntax

## Performance Considerations

Handlebars compilation is sub-millisecond per template. Adding 12 fragment files to the existing 1 increases the registered partial count from 1 to 13, but each spec only resolves the fragments it references (typically 1-3). The full build should remain under 1 second for fragment resolution.

## References

- Research document: `thoughts/research/2026-03-07-spec-duplication-fragment-candidates.md`
- Idea document: `thoughts/ideas/2026-03-07-fragment-deduplication.md`
- Previous plan (infrastructure + first fragment): `thoughts/plans/2026-03-07-shared-compiler-fragments.md`
- Existing fragment: `spec/fragments/review/prompt-contract.md`
- Compiler fragment docs: `agent-docs/compiler-fragments.md`
- Fragment compiler: `scripts/lib/fragments.ts`
- Fragment tests: `scripts/__tests__/fragments.test.ts`
- Build command: `npm --prefix agent-config run build:agents`
- Test command: `npm --prefix agent-config test`
