# Neutralize `AskUserQuestion` References in Agent-Config Specs — Implementation Plan

## Overview

Replace provider-specific `AskUserQuestion` references in `agent-config/spec/` (and the sibling `AGENTS_GLOBAL.md`) with the established provider-neutral phrase `"your environment's selection/picker UI"`. This finishes a neutralization effort already completed across 30+ other spec locations and the shared `fragments/interaction/question-phrasing.md` fragment, per the `agent-config/CLAUDE.md` guideline: *"Keep canonical instruction text provider-neutral when possible."*

## Current State Analysis

The `agentspec` compiler generates Claude Code, Cursor, and OpenCode configs from canonical specs in `agent-config/spec/`. Most spec text is already provider-neutral, and there is a shared fragment (`fragments/interaction/question-phrasing.md`) that uses `"your environment's selection/picker UI"` and is included by 15 skills.

Five files still reference the Claude-specific `AskUserQuestion` tool name (16 occurrences in `spec/` plus 1 in `AGENTS_GLOBAL.md`):

| File | Occurrences | Notes |
| --- | --- | --- |
| `spec/rules/question-phrasing.md` | 9 | Includes 1 in the `description:` frontmatter; the rule doubles as a free-standing rule file and a sibling to the shared fragment |
| `spec/skills/create-idea/SKILL.md` | 3 | Question section, workflow step, and example transcript |
| `spec/skills/triage-todos/SKILL.md` | 3 | Includes Claude-specific detail: "supports a maximum of 4 options" and the auto-provided "Other" option |
| `spec/rules/git-conventions.md` | 1 | Main-branch detection paragraph |
| `agent-config/AGENTS_GLOBAL.md` | 1 | User-level rules file that duplicates guidance from `rules/question-phrasing.md` |

The compiler embeds tool-name mappings but does not transform concept-level references like `AskUserQuestion` — these pass through verbatim into all three provider outputs today.

### Key Discoveries:

- The neutral phrase `"your environment's selection/picker UI"` is the established pattern — it appears in 30+ locations across `spec/skills/`, `spec/fragments/`, and `spec/rules/` (verified via grep). Example call sites include `fragments/plans/implementation-workflow.md`, `skills/commit/SKILL.md`, and `skills/split-branch/SKILL.md`.
- The `fragments/interaction/question-phrasing.md` fragment is already neutral and does not need changes. It is the condensed, inline version imported by 15 skills; `rules/question-phrasing.md` is the longer standalone rule that needs to match its neutrality.
- `spec/rules/question-phrasing.md:58,79,82` use `AskUserQuestion(...)` as pseudocode in good/bad examples. Those need to be reworded as prose (e.g., `picker UI with question "…" and options with descriptions`) rather than preserved as call syntax, since a neutral pseudo-call doesn't read well.
- `triage-todos` references the Claude-specific 4-option limit and the auto-provided "Other" sink. Per the design decision for this plan, these are abstracted to generic picker-UI guidance while preserving the "top 3 + Show more groups" pagination behavior.
- The prior plan `thoughts/plans/2026-04-13-implement-plan-picker-ui.md` did the same kind of neutralization for the implementation-workflow fragment, confirming this is an ongoing, well-understood effort with the same pattern and verification approach.
- `agentspec` provides `agentspec validate`, `agentspec compile`, and `agentspec check` (compile + diff against `generated/`). After spec edits, `generated/` regenerates via `agentspec compile`.

## Desired End State

After implementation:

- No file under `agent-config/spec/` or `agent-config/AGENTS_GLOBAL.md` contains the string `AskUserQuestion`.
- All references have been rewritten using `"your environment's selection/picker UI"`, `"selection/picker UI"`, or `"picker UI"` where grammatically appropriate — matching the established pattern.
- The `description:` frontmatter field on `spec/rules/question-phrasing.md` no longer names `AskUserQuestion`.
- The `triage-todos` skill retains its 3-group + "Show more" pagination guidance and its "None — skip for now" option, but describes these as properties of any picker UI rather than quoting `AskUserQuestion` semantics.
- `agentspec validate` passes.
- `agentspec compile` regenerates `agent-config/generated/` cleanly, and `agent-config/generated/` no longer contains `AskUserQuestion` (currently 55 occurrences across 15 generated files).

Verification: `grep -rn 'AskUserQuestion' agent-config/spec agent-config/AGENTS_GLOBAL.md agent-config/generated` returns no results.

## What We're NOT Doing

- Not touching the 15 skills that already include the shared `fragments/interaction/question-phrasing.md` — they are already neutral.
- Not modifying `fragments/interaction/question-phrasing.md` itself — already neutral.
- Not adding any new fragments or MiniJinja conditionals. Provider-specific branching was considered and rejected in favor of uniform neutral text (matching the rest of the codebase).
- Not editing the global user-rules file at `~/.claude/CLAUDE.md` — this plan scopes to the `agent-config/` source tree only.
- Not hand-editing files under `agent-config/generated/` — these regenerate via `agentspec compile`.
- Not renaming `question-phrasing` rules/fragments or restructuring their organization.
- Not changing the `agentspec` compiler or adding new tool-name mappings.

## Implementation Approach

Five files to edit. Each edit is a local string substitution or a small paragraph rewrite — no new sections, no restructuring. After the edits, run `agentspec validate` → `agentspec compile` → `agentspec check`, then grep for residual references.

## Implementation

### Overview

Rewrite the 17 `AskUserQuestion` references across 5 files using the established neutral phrasing, then regenerate and verify.

### Changes Required:

#### 1. `spec/rules/question-phrasing.md` (9 occurrences)

**File**: `agent-config/spec/rules/question-phrasing.md`

- [x] Update `description:` frontmatter (line 3): replace `AskUserQuestion for multiple choice` with `selection/picker UI for multiple choice`.
- [x] Rewrite the "Recommendations" section (line 14): change `When using \`AskUserQuestion\`, mark the recommended option as the first in the list and append "(Recommended)" to its label.` → `When using a selection/picker UI, mark the recommended option as the first in the list and append "(Recommended)" to its label.`
- [x] Rewrite the "Multiple-Choice Questions" opening (lines 50–52): change `**use the \`AskUserQuestion\` tool** instead of listing choices inline as text. The tool renders a picker UI that lets the user select an option with one click rather than typing a response.` → `**use your environment's selection/picker UI** instead of listing choices inline as text. The picker lets the user select an option with one click rather than typing a response.`
- [x] Rewrite the good example (line 58): change `✅ GOOD: Use AskUserQuestion with labeled options and descriptions` → `✅ GOOD: Use a selection/picker UI with labeled options and descriptions`.
- [x] Rewrite the fallback sentence (line 62): change `Reserve inline text questions for cases where \`AskUserQuestion\` is unavailable` → `Reserve inline text questions for cases where a selection/picker UI is unavailable`.
- [x] Rewrite the "Self-Contained Questions" opening (line 67): change `The \`AskUserQuestion\` picker can obscure text output that precedes it.` → `The selection/picker UI can obscure text output that precedes it.`
- [x] Rewrite the bulleted reference (line 76): change `not in the same response as the \`AskUserQuestion\` call.` → `not in the same response as the picker UI call.`
- [x] Rewrite the pseudocode examples (lines 79, 82) as prose rather than keeping `AskUserQuestion(...)` call syntax:
  - Bad: `❌ BAD:  [paragraph explaining three approaches] → picker UI asking "Which approach?"`
  - Good: `✅ GOOD: picker UI with question "Which approach for X? …" and options with descriptions`

#### 2. `spec/skills/create-idea/SKILL.md` (3 occurrences)

**File**: `agent-config/spec/skills/create-idea/SKILL.md`

These references describe gathering open-ended clarifying questions (problem space, scope, success criteria), which are typically not discrete-option questions. Replace with neutral phrasing that does not presume a picker UI.

- [x] Line 69: change `Use AskUserQuestion to gather missing information. Good questions explore:` → `Present questions using your environment's selection/picker UI to gather missing information. Good questions explore:`
  > **Deviation:** Initial rewrite used `Ask clarifying questions to gather missing information`, which stripped the picker-UI intent. User feedback during manual verification called this out. Revised to preserve picker-UI specificity using the established neutral phrasing.
- [x] Line 172 (workflow step): change `2. **Ask**: Use AskUserQuestion to clarify gaps (2-3 questions at a time)` → `2. **Ask**: Use your environment's selection/picker UI to present 2-3 questions at a time`
  > **Deviation:** Same reason as above — initial rewrite dropped picker specificity; revised.
- [x] Line 201 (example transcript): change `[Uses AskUserQuestion with 2-3 targeted questions]` → `[Presents 2-3 targeted questions using the selection/picker UI]`
  > **Deviation:** Same reason as above — initial rewrite dropped picker specificity; revised.
- [x] **Added during implementation:** Append `{% include "interaction/question-phrasing.md" %}` at the end of `create-idea/SKILL.md` (scope expansion per user direction). Rationale: 15 other skills already include this fragment; `create-idea` is a skill entirely about asking questions yet did not include it, which was a consistency gap. Expanding scope to close that gap was chosen over keeping the diff minimal.

#### 3. `spec/skills/triage-todos/SKILL.md` (3 occurrences)

**File**: `agent-config/spec/skills/triage-todos/SKILL.md`

- [x] Line 179–180: change `Present the ranked groups to the user using \`AskUserQuestion\`. Order groups from lowest ambiguity (most actionable) to highest ambiguity (most vague).` → `Present the ranked groups to the user using your environment's selection/picker UI. Order groups from lowest ambiguity (most actionable) to highest ambiguity (most vague).`
- [x] Lines 188–192 (the 4-option constraint paragraph): replace
  ```
  `AskUserQuestion` supports a maximum of 4 options. If there are more than 3
  groups, present the top 3 (by ambiguity rank) and include a 4th option:
  "Show more groups" which re-presents the next batch. Always include a
  "None — skip for now" path via the "Other" option (automatically provided by
  `AskUserQuestion`).
  ```
  with
  ```
  Picker UIs typically support a limited number of options. If there are more
  than 3 groups, present the top 3 (by ambiguity rank) and include a 4th option:
  "Show more groups" which re-presents the next batch. Always include a
  "None — skip for now" option so the user can bail out without selecting a
  group.
  ```

#### 4. `spec/rules/git-conventions.md` (1 occurrence)

**File**: `agent-config/spec/rules/git-conventions.md`

- [x] Line 27: change `If on \`main\` or \`master\`, **stop and ask the user** (using \`AskUserQuestion\`) whether to:` → `If on \`main\` or \`master\`, **stop and ask the user** (presenting options via your environment's selection/picker UI) whether to:`

#### 5. `agent-config/AGENTS_GLOBAL.md` (1 occurrence)

**File**: `agent-config/AGENTS_GLOBAL.md`

- [x] Lines 12–14: change
  ```
  - When a question has discrete options (not yes/no), use the `AskUserQuestion`
    tool instead of listing choices inline as text. The picker UI lets the user
    select with one click rather than typing a response.
  ```
  to
  ```
  - When a question has discrete options (not yes/no), present them using your
    environment's selection/picker UI instead of listing choices inline as
    text. This lets the user select with one click rather than typing a
    response.
  ```

#### 6. Regenerate and verify

- [x] From `agent-config/`, run `agentspec validate` — passes cleanly.
- [x] From `agent-config/`, run `agentspec compile` — regenerates `generated/` for all three providers.
- [x] ~~From `agent-config/`, run `agentspec check` — confirms the diff between compiled output and on-disk `generated/` is empty after step above.~~
  > **Deviation:** The `agentspec check` subcommand no longer exists (only `validate`, `compile`, `sync`, `completions` are available). Substituted an equivalent guarantee: `agentspec compile` writes directly into `generated/`, and the git diff confirms spec and generated outputs are in sync (`git status` shows both spec files and their regenerated counterparts modified together).
- [x] Confirm `generated/` no longer contains `AskUserQuestion` (prior count: 55 occurrences across 15 files).

### Success Criteria:

#### Automated Verification:

- [x] `agentspec validate` succeeds from `agent-config/` (no frontmatter or semantic errors).
- [x] `agentspec compile` succeeds from `agent-config/` and writes updated `generated/` tree.
- [x] ~~`agentspec check` from `agent-config/` reports a clean state (no diff between compiled output and committed `generated/`).~~
  > **Deviation:** Subcommand does not exist — see deviation note above. Equivalent verification: `git diff --stat` shows spec and generated/ files modified together with proportional line counts (19 files, 95/94 insertions/deletions, all confined to the substituted phrases).
- [x] `grep -rn 'AskUserQuestion' agent-config/spec` returns no results.
- [x] `grep -n 'AskUserQuestion' agent-config/AGENTS_GLOBAL.md` returns no results.
- [x] `grep -rn 'AskUserQuestion' agent-config/generated` returns no results.

#### Manual Verification:

- [x] Spot-read the three generated rule/skill files for Claude (`generated/claude/rules/tw-question-phrasing.md`, `generated/claude/rules/tw-git-conventions.md`, `generated/claude/skills/tw-triage-todos/SKILL.md`) to confirm the new phrasing reads naturally and preserves the original guidance intent (manual-only: subjective prose quality judgment).
  > **Note:** On first pass the user flagged that `create-idea` had lost picker-UI specificity. Revised inline references and added the shared `interaction/question-phrasing.md` fragment (see deviation note in §2). Re-verified on second pass and confirmed passing.

## Post-Review Polish (added during implementation)

After code review surfaced 6 minor prose issues, the following additional edits were applied before commit:

- [x] `spec/rules/question-phrasing.md:71,73` — Replaced schema-coupled field names (`question` field / `description` fields) with plain-language references (`the question text` / `each option's description`). Rationale: those field names were carried over from the `AskUserQuestion` schema and don't make sense in a provider-neutral rule.
- [x] `spec/rules/question-phrasing.md:76` — Changed `not in the same response as the picker UI call` → `not in the same response that presents the picker`. Rationale: "picker UI call" is awkward without a specific tool to call.
- [x] `spec/rules/question-phrasing.md:79,82` — Fixed fragment grammar in BAD/GOOD examples. Prefixed `picker UI` with an article (`a picker UI` / `A picker UI`) so both read as proper noun phrases.
- [x] `spec/rules/git-conventions.md:27` — Factored the long picker-UI parenthetical out of the lead sentence. Moved the instruction `Present the choice using your environment's selection/picker UI.` to follow the list of options, matching the reading flow of the original terse `(using AskUserQuestion)` form.
- [x] `spec/skills/triage-todos/SKILL.md:188–192` — Tightened the softened premise. Changed `typically support a limited number of options` → `often cap options at around 4` so the claim lines up with the specific 3-group + 4th-option prescription that follows.
- [x] `spec/skills/triage-todos/SKILL.md:191–192` — Changed colloquial `bail out` → `decline` to match the tone of the rest of the doc.
- [x] Re-ran `agentspec validate` and `agentspec compile` after polish — both pass, 0 residual `AskUserQuestion` references anywhere in `agent-config/`.

## References

- Prior neutralization effort: `thoughts/plans/2026-04-13-implement-plan-picker-ui.md`
- Canonical neutral fragment: `agent-config/spec/fragments/interaction/question-phrasing.md`
- Compiler guidelines: `agent-config/CLAUDE.md` ("Keep canonical instruction text provider-neutral when possible")
- Established pattern example: `agent-config/spec/skills/split-branch/SKILL.md:46,79,311,416,428`
- Files being modified:
  - `agent-config/spec/rules/question-phrasing.md`
  - `agent-config/spec/rules/git-conventions.md`
  - `agent-config/spec/skills/create-idea/SKILL.md`
  - `agent-config/spec/skills/triage-todos/SKILL.md`
  - `agent-config/AGENTS_GLOBAL.md`
