# Implement-Plan Picker UI — Implementation Plan

## Overview

Convert free-text questions in the implement-plan workflow to structured picker UI instructions using the established provider-neutral pattern (`"your environment's selection/picker UI"`). This covers the shared `plans/implementation-workflow.md` fragment (section transitions + commit prompts), the `implement-plan-worktree` skill (pre-flight confirmation), and the `commit` skill (branch check).

## Current State Analysis

The `plans/implementation-workflow.md` fragment defines pause templates as code blocks with trailing free-text questions. These questions describe multiple possible responses in prose, but don't instruct the agent to use a picker UI. Meanwhile, many other skills and fragments consistently use the pattern:

```
Present options using your environment's selection/picker UI:
- **Option label** — description
```

The `implement-plan-worktree` skill already uses the picker pattern for its branch-exists and project-detection prompts, but its pre-flight confirmation (`"Proceed?"`) does not.

### Key Discoveries:

- The fragment's pause templates are code blocks that the agent outputs as status summaries. The trailing questions (e.g., `"Ready to proceed to [next section name]?"`) live *inside* these code blocks. The picker instructions must replace these trailing questions and be placed *after* the code block — the picker is a separate interaction, not part of the status output.
- The commit prompt augments the pause flow conditionally (only when `git status --porcelain` has output). The picker must handle both the "with commits" and "without commits" paths.
- The "execute multiple phases consecutively" mode (line 117) skips the entire pause-and-commit flow until the final phase. Since both the section-transition picker and the commit picker are part of this flow, they're naturally skipped too — no special handling needed.
- The deduplication guard (`"Present the choice exactly once — if using a selection UI or tool, do not also print the question as text"`) is used in other skills but isn't needed here because we're removing the free-text questions from the code blocks rather than having both.

## Desired End State

After implementation:
- Every discrete decision point in the shared fragment and the worktree skill uses the picker pattern where appropriate
- The code-block status summaries no longer contain trailing questions — they end with the last informational line
- Picker instructions follow each code block as a separate paragraph
- `agentspec compile` succeeds cleanly

## What We're NOT Doing

- Redesigning the pause template structure or verification approach
- Changing the commit skill's approval prompt (line 47) — this is a yes/no question, not a discrete-choice picker
- Adding picker UI to mismatch/stuck prompts (open-ended, free-text is appropriate)
- Changing the worktree branch-exists or project-detection prompts (already use the picker pattern)
- Changing the worktree cleanup offer (low-frequency, natural language is fine)

## Implementation Approach

Edit the three spec files to replace free-text trailing questions with picker instructions. The fragment has four decision points to convert (two section-transition variants plus two commit variants) plus the commit response handling, the worktree pre-flight, and the commit skill's branch check. Each conversion follows the same pattern: remove the question from the code block, add a picker instruction block after it.

## Implementation

### Overview

Replace free-text questions with picker UI instructions in the shared implementation workflow fragment and the worktree pre-flight confirmation.

### Changes Required:

#### 1. Section Transition — No Manual Steps (`spec/fragments/plans/implementation-workflow.md`)

**Current** (lines 71–86): The "no manual verification steps" code block ends with:
```
No manual verification steps for this section. Ready to proceed to [next section name]?
```

- [x] Split the trailing line in the code block. The current line reads `No manual verification steps for this section. Ready to proceed to [next section name]?` — keep `No manual verification steps for this section.` as the final line of the code block and remove only the `Ready to proceed to [next section name]?` part.
- [x] Add picker instructions after the code block:

  ```markdown
  Present options using your environment's selection/picker UI:
  - **Proceed to [next section name]** (Recommended) — all checks passed, continue to the next phase
  - **Hold** — I have feedback or changes before continuing
  ```

- [x] For single-section plans, the current text says to replace "Ready to proceed to [next section name]?" with "Implementation complete — anything else to address?" — update this rule to instead say: for single-section plans, replace the picker with:

  ```markdown
  Present options using your environment's selection/picker UI:
  - **Done** (Recommended) — implementation complete, nothing else to address
  - **Hold** — I have feedback or changes to address
  ```

#### 2. Section Transition — With Manual Steps (`spec/fragments/plans/implementation-workflow.md`)

**Current** (lines 53–68): The "has manual verification steps" code block ends with:
```
Let me know when manual testing is complete so I can proceed to [next section name].
```

- [x] Remove the trailing question from the code block. The last line inside the block should be the final manual verification item.
- [x] Add picker instructions after the code block:

  ```markdown
  Present options using your environment's selection/picker UI:
  - **Manual testing passed** — all manual verification steps confirmed
  - **Manual testing failed** — issues found during manual testing; describe what went wrong
  ```

  No `(Recommended)` on either option — the agent can't know the outcome. This picker is unchanged for single-section plans since neither option references the next section name.

#### 3. Commit Prompt — No Manual Steps (`spec/fragments/plans/implementation-workflow.md`)

**Current** (lines 90–97): When uncommitted changes are present and there are no manual steps, the trailing question is replaced with:
```
Ready to commit these changes and proceed to [next section name]?
You can also skip the commit to batch with later phases.
```

- [x] Rewrite the `**When uncommitted changes are present**` lead-in and the no-manual-steps sub-bullet (fragment lines 90–97) entirely. Currently the lead-in says "replace the trailing question in the appropriate template with the commit variant" — but after change 1, there is no trailing question to replace. The new instruction should say: when uncommitted changes are present, after presenting the code block (which now ends without a question), present a commit picker instead of the section-transition picker. Leave lines 99–105 (the manual-steps sub-bullet) for change 4 to handle. The single-section override at line 107 currently applies to both sub-bullets — rewrite it here to reference the picker label rather than inline text replacement. The rewritten override must cover both the no-manual commit picker (this change) and the post-manual-verification commit picker (change 4), since both use the same "Commit and proceed" / "Commit and wrap up" label substitution:

  ```markdown
  Present options using your environment's selection/picker UI:
  - **Commit and proceed** (Recommended) — commit changes, then continue to [next section name]
  - **Skip commit** — proceed with uncommitted changes; batch with later phases
  - **Request changes** — make adjustments before committing
  ```

- [x] For single-section plans, the "proceed to [next section name]" text becomes "wrap up implementation" — update the picker label accordingly:

  ```markdown
  - **Commit and wrap up** (Recommended) — commit changes and wrap up implementation
  ```

#### 4. Commit Prompt — After Manual Verification (`spec/fragments/plans/implementation-workflow.md`)

**Current** (lines 99–105): After the user confirms manual verification passed, a follow-up commit prompt is presented:
```
Manual verification confirmed. Ready to commit these changes and
proceed to [next section name]?
You can also skip the commit to batch with later phases.
```

- [x] Replace this free-text follow-up with picker-driven branching after the user selects from the manual-verification picker (change 2):
  - **"Manual testing passed" + uncommitted changes**: present the commit picker from change 3. The "Manual verification confirmed" prefix is no longer needed since the user just selected it from a picker.
  - **"Manual testing passed" + no uncommitted changes**: for multi-phase plans, proceed directly to the next section. For single-section plans, report implementation complete. No additional picker is needed — the user already confirmed readiness by selecting "passed."
  - **"Manual testing failed"**: ask the user to describe what went wrong, fix the issues, re-run verification, then re-present the manual-verification picker. This preserves the existing implicit behavior.

#### 5. Commit Response Handling (`spec/fragments/plans/implementation-workflow.md`)

**Current** (lines 109–113): Three response paths described in prose.

- [x] Update the response handling to reference picker selections instead of free-text responses. The three paths remain the same but are now triggered by picker selection rather than interpreting free-text:
  1. **User selects "Commit and proceed"**: Load the commit skill and follow its full workflow, then proceed to the next phase.
  2. **User selects "Request changes"**: Make the requested changes, re-run verification as needed, then re-present the commit picker.
  3. **User selects "Skip commit"**: Proceed to the next phase with uncommitted changes.

#### 6. Pre-flight Confirmation (`spec/skills/implement-plan-worktree/SKILL.md`)

**Current** (lines 93–104): The prose instruction and code block for the pre-flight summary. The code block (lines 95–104) ends with `Proceed?`.

- [x] Remove `Proceed?` from the code block.
- [x] Add picker instructions after the code block:

  ```markdown
  Present options using your environment's selection/picker UI:
  - **Proceed** (Recommended) — create the worktree and start implementation
  - **Cancel** — stop without changes
  ```

#### 7. Commit Skill Branch Check (`spec/skills/commit/SKILL.md`)

**Current** (lines 20–27): When on `main` or `master`, the skill asks the user to choose between creating a feature branch or proceeding on main. It uses a numbered list format (`1. Create a feature branch first (recommended)` / `2. Proceed on main anyway`) with the dedup guard, rather than the standard bold-label picker pattern.

- [x] Convert the numbered list to the standard picker format:

  ```markdown
  Present options using your environment's selection/picker UI:
  - **Create a feature branch** (Recommended) — suggest a concrete name using `<username>/<description>` format
  - **Proceed on main** — commit directly to the current branch
  ```

- [x] Keep the existing dedup guard (`"Present the choice exactly once — if using a selection UI or tool, do not also print the question as text"`) and the `"Do not continue to step 1 until the user responds"` instruction.

#### 8. Compile and Verify

- [x] Run `agentspec compile` from `agent-config/` and fix any issues
- [x] Spot-check the generated output for the affected skills (implement-plan, implement-plan-worktree, commit) to confirm picker instructions appear correctly

### Success Criteria:

#### Automated Verification:

- [x] `agentspec compile` succeeds with no errors
- [x] ~~`agentspec check`~~ `git diff --name-only` shows changes only in the expected generated files (implement-plan, implement-plan-worktree, and commit skill outputs)
  > **Deviation:** `agentspec check` is not a valid subcommand. Used `git diff --name-only` instead, which confirmed changes are limited to the 3 spec files and their generated outputs across all 3 providers.
- [x] No references to `AskUserQuestion` or other provider-specific tool names in the changed files: `grep -r "AskUserQuestion" spec/fragments/plans/ spec/skills/implement-plan*/ spec/skills/commit/`

#### Manual Verification:

- [ ] Review the generated output for `implement-plan`, `implement-plan-worktree`, and `commit` — confirm picker instructions read naturally in context and the code-block templates no longer contain trailing questions
- [ ] Verify the "execute multiple phases consecutively" instruction (line 117) still makes sense with the new picker wording

## References

- Idea document: `thoughts/ideas/2026-04-13-implement-plan-picker-ui.md`
- Established picker pattern: `spec/fragments/interaction/question-phrasing.md`
- Commit skill (branch check): `spec/skills/commit/SKILL.md`
- Shared implementation workflow: `spec/fragments/plans/implementation-workflow.md`
- Implement-plan skill: `spec/skills/implement-plan/SKILL.md`
- Implement-plan-worktree skill: `spec/skills/implement-plan-worktree/SKILL.md`
