# Implement-Plan Skills: Structured Picker UI for User Prompts

## Problem Statement

The `implement-plan` and `implement-plan-worktree` skills include several points where the agent pauses to ask the user a question — section transitions, commit prompts, mismatch handling, and worktree setup decisions. Most of these are currently phrased as free-text questions embedded in code-block templates, which means the agent either renders them as plain text (requiring the user to type a response) or inconsistently decides whether to use a picker UI.

The codebase has a well-established provider-neutral pattern (`"your environment's selection/picker UI"`) used in 20+ locations across other skills and fragments, but the `plans/implementation-workflow.md` fragment — the shared core of both implement-plan skills — doesn't use it at all.

## Motivation

- **Reduced friction**: Picker UIs let users respond with a single click instead of typing. For repetitive prompts like section transitions (which fire after every phase), this adds up.
- **Clearer options**: The commit prompt currently describes three possible responses (approve, skip, request changes) in prose. A structured picker makes these discrete options explicit and unambiguous.
- **Consistency**: Other skills in the same codebase (resolve-conflicts, code-review-loop, split-branch, commit) already use the picker pattern for similar decision points. The implement-plan skills should follow suit.
- **Provider neutrality**: The existing pattern works across Claude Code (`AskUserQuestion`), Cursor, and OpenCode without naming any provider-specific tool.

## Context

### Current State

The `plans/implementation-workflow.md` fragment (included by both `implement-plan` and `implement-plan-worktree`) defines pause templates as code blocks with trailing questions:

- **No-manual sections**: `"Ready to proceed to [next section name]?"`
- **With-manual sections**: `"Let me know when manual testing is complete so I can proceed to [next section name]."`
- **Commit variant (no-manual)**: `"Ready to commit these changes and proceed to [next section name]?\nYou can also skip the commit to batch with later phases."`
- **Commit variant (with-manual)**: `"Manual verification confirmed. Ready to commit these changes and\nproceed to [next section name]?\nYou can also skip the commit to batch with later phases."`
- **Mismatch handling**: `"How should I proceed?"` (open-ended)

The `implement-plan-worktree` skill also has its own decision points:

- **Branch already exists**: Resume / Start fresh / Abort (already references picker UI)
- **Project detection — multiple repos**: Already references picker UI
- **Pre-flight confirmation**: `"Proceed?"` (yes/no, no picker)

### Established Pattern

The provider-neutral phrase used across the codebase is:

```
Present options using your environment's selection/picker UI:
```

Followed by a bulleted list of bold-labeled options. Some skills add a deduplication guard:

```
Present the choice exactly once — if using a selection UI or tool,
do not also print the question as text.
```

## Goals / Success Criteria

- [ ] Every discrete user decision point in both skills and the shared fragment uses the picker pattern where appropriate
- [ ] All picker instructions use the established provider-neutral wording
- [ ] The deduplication guard is applied where the picker would otherwise duplicate a code-block template
- [ ] No regressions in the single-section plan flow or the "execute multiple phases consecutively" flow
- [ ] `agentspec compile` succeeds cleanly after changes

## Non-Goals (Out of Scope)

- Redesigning the pause template structure or verification approach
- Changing the commit skill itself (it already uses the picker pattern)
- Adding picker UI to the mismatch/stuck prompts — these are open-ended situations where free-text is appropriate
- Changing how the `implement-plan-worktree` branch-exists or project-detection prompts work (they already use the picker pattern)

## Candidate Decision Points

Full audit of both skills and the shared fragment, with recommendations:

### Shared Fragment (`plans/implementation-workflow.md`)

| Decision Point | Current UX | Recommendation |
|---|---|---|
| Section transition (no manual steps) | Free-text: "Ready to proceed to [next section]?" | **Convert to picker**: Proceed to [next section] (Recommended) / Hold — I have feedback |
| Section transition (with manual steps) | Free-text: "Let me know when manual testing is complete..." | **Convert to picker**: Manual testing passed / Manual testing failed (no recommendation — agent can't know the outcome) |
| Commit prompt (no manual steps) | Free-text with skip hint in prose | **Convert to 3-option picker**: Commit and proceed (Recommended) / Skip commit / Request changes |
| Commit prompt (after manual verification) | Free-text with skip hint in prose | **Convert to 3-option picker**: Same as above |
| Mismatch handling | Open-ended: "How should I proceed?" | **Keep as free-text** — situation-dependent, not a discrete choice |

### `implement-plan-worktree/SKILL.md`

| Decision Point | Current UX | Recommendation |
|---|---|---|
| Branch already exists | Already uses picker (Resume/Start fresh/Abort) | **No change needed** |
| Multiple project repos detected | Already uses picker | **No change needed** |
| Pre-flight confirmation ("Proceed?") | Free-text yes/no | **Convert to picker**: Proceed (Recommended) / Abort |
| Worktree cleanup offer | Free-text mention: "Just say the word" | **Keep as free-text** — optional, low-frequency, natural language is fine |

### `implement-plan/SKILL.md`

| Decision Point | Current UX | Recommendation |
|---|---|---|
| No plan path provided | "Ask for one" (open-ended) | **Keep as free-text** — requires a file path, not a discrete choice |

## Constraints

- Must use provider-neutral language (`"your environment's selection/picker UI"`) — never reference `AskUserQuestion` or any other provider-specific tool name
- Changes are in `spec/fragments/` and `spec/skills/` — must pass `agentspec compile`
- The commit prompt's three options (approve, skip, request changes) need to stay clearly distinguishable since the handler logic branches on which one is chosen
- The "execute multiple phases consecutively" mode skips pauses until the final phase — picker instructions must not conflict with this flow

## Resolved Questions

- **Section-transition picker options**: Two options — Proceed / Hold. The user can type feedback after selecting Hold; a third "Request changes" option adds weight to a prompt that fires after every phase.
- **Manual-verification picker options**: Two options — Manual testing passed / Manual testing failed. No recommendation marker on either — the agent can't know the outcome, so recommending one would be presumptuous. No "Skip" option — if manual steps are in the plan, they were put there for a reason; users who want to skip can always type it.

## References

- Established picker pattern: `spec/fragments/interaction/question-phrasing.md`
- Deduplication guard example: `spec/skills/commit/SKILL.md:25-26`
- Shared implementation workflow: `spec/fragments/plans/implementation-workflow.md`
- Implement-plan skill: `spec/skills/implement-plan/SKILL.md`
- Implement-plan-worktree skill: `spec/skills/implement-plan-worktree/SKILL.md`

## Affected Systems/Services

- `spec/fragments/plans/implementation-workflow.md` — primary change target
- `spec/skills/implement-plan-worktree/SKILL.md` — pre-flight confirmation
- `generated/` — will be regenerated by `agentspec compile`
