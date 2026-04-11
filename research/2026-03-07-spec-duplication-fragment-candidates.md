---
date: 2026-03-07T13:08:58Z
git_commit: f8916dd14c6f8ea47479ebb2321739e8354536f1
branch: master
repository: dotfiles
topic: "Spec duplication analysis and Handlebars fragment candidates"
tags: [research, codebase, fragments, handlebars, duplication, consistency]
status: complete
last_updated: 2026-03-07
last_updated_by: assistant
---

# Research: Spec Duplication Analysis and Fragment Candidates

**Date**: 2026-03-07T13:08:58Z
**Git Commit**: f8916dd14c6f8ea47479ebb2321739e8354536f1
**Branch**: master
**Repository**: dotfiles (agent-config/)

## Research Question

Where can we remove duplication across agent and skill specs using Handlebars partials or template variables? The goal is to promote consistency between related skills, avoiding drift and ensuring similar skills behave in a consistent way.

## Summary

The codebase has **1 existing fragment** (`review/prompt-contract`) and significant duplication across **8 agents** and **28 skills**. The analysis identified **13 high-value fragment candidates** organized into 5 thematic clusters. The largest duplication clusters are:

1. **Codebase agent documentarian rules** (~3 agents, ~15 shared lines)
2. **Implementation workflow** between `implement-plan` and `implement-plan-worktree` (~100 shared lines)
3. **Research document conventions** across 3 research skills (~60 shared lines)
4. **PR authoring** between `create-pr` and `describe-pr` (~80 shared lines)
5. **Feedback processing workflow** between `review-annotations` and `review-pr-comments` (~50 shared lines)

## Fragment Candidates

### Cluster 1: Agent Behavioral Contracts

These fragments codify shared behavioral rules for sub-agents, preventing drift in how agents are instructed to behave.

#### Fragment 1: `agents/documentarian-mandate`

**Files**: `codebase-analyzer`, `codebase-locator`, `codebase-pattern-finder` (agents); also `research-codebase`, `iterate-research` (skills)

**Shared text** (the "CRITICAL" preamble + closing "REMEMBER" block):

Opening block (~7 lines):

```
## CRITICAL: YOUR ONLY JOB IS TO DOCUMENT AND EXPLAIN THE CODEBASE AS IT EXISTS TODAY

- DO NOT suggest improvements or changes unless the user explicitly asks for them
- DO NOT perform root cause analysis unless the user explicitly asks for them
- DO NOT propose future enhancements unless the user explicitly asks for them
- DO NOT critique the implementation or identify problems
- ONLY describe what exists, where it exists, how it works, and how components interact
- You are creating a technical map/documentation of the existing system
```

Closing block (~4 lines):

```
## REMEMBER: You are a documentarian, not a critic or consultant
...
```

**Parameterization**: The heading varies slightly per agent ("DOCUMENT AND EXPLAIN" vs "DOCUMENT AND SHOW EXISTING PATTERNS"), and each agent adds role-specific bullets. This could be a fragment with a `role` variable that conditionally includes extra bullets, or split into a core mandate + per-agent additions.

**Variant**: `research-codebase` and `iterate-research` use a condensed version in their "Important notes" closing section (3 bullets: CRITICAL/REMEMBER/NO RECOMMENDATIONS). This could be a separate `agents/documentarian-reminder` fragment.

#### Fragment 2: `agents/file-line-references`

**Files**: `codebase-analyzer`, `codebase-pattern-finder`, `code-reviewer` (agents)

**Shared text** (~1 line):

```
Always include file:line references for claims
```

**Assessment**: Small but frequently drifting. Currently uses slightly different wording in each file. A fragment would enforce a single canonical phrasing.

#### Fragment 3: `agents/no-critique-bullets`

**Files**: `codebase-analyzer`, `codebase-locator`, `codebase-pattern-finder`, `learnings-discoverer`, `code-reviewer` (agents)

**Shared text** (varies by agent but draws from the same pool):

```
- Don't suggest improvements to [unchanged code / file organization / patterns]
- Don't critique [design patterns / architecture / pattern quality]
- Don't recommend [best practices / refactoring / one pattern over another]
- Don't guess about [implementation / functionality / issues]
```

**Assessment**: Each agent picks ~3-4 bullets from this family. A fragment with a `context` variable could generate the appropriate variant. Alternatively, since these are tightly coupled with each agent's unique "What NOT to Do" section, they may be better as individual lines pulled from a shared list via variables.

---

### Cluster 2: Review Workflow

These fragments consolidate the review orchestration patterns shared by skills that delegate to the `code-reviewer` agent.

#### Fragment 4: `review/delegation-rules`

**Files**: `code-review`, `code-review-loop`, `review-files`, `review-pr` (skills)

**Shared text** (~4 lines):

```
**DO NOT** perform the review yourself — always spawn the code-reviewer agent.
The agent has specialized review logic, evaluation criteria, and output formatting.
```

Plus the triage collaborative rule:

```
The triage phase is collaborative — do not auto-fix findings without user agreement on each one.
```

**Assessment**: High value. Currently drifts between "spawn" vs "delegate to", "DO NOT" vs "Do not", hyphens vs em-dashes. A fragment locks these down.

#### Fragment 5: `review/triage-workflow`

**Files**: `code-review`, `review-files` (skills); partially `review-pr`

**Shared text** (~15 lines):

```
**Present the results**
- The code-reviewer agent will return structured findings
- Present them clearly to the user

**Work through findings collaboratively**
- Immediately transition into addressing findings — don't wait for the user to ask
- Work through each finding in severity order: Blockers → Major → Minor
- For each finding:
  - Present the issue briefly
  - Ask the user: fix now, skip, or defer?
- For Questions/Assumptions and Verification Gaps, discuss briefly for alignment
  but no code changes are needed
```

**Parameterization**: `code-review` adds todo creation instructions; `review-pr` adapts for PR comment posting context. A `mode` variable (e.g., `"interactive"` vs `"pr"`) could control these variants.

#### Fragment 6: `review/merge-base-resolution`

**Files**: `code-review`, `code-review-loop`, `review-pr` (skills)

**Shared text** (~15 lines of bash):

```bash
BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
BASE_BRANCH=${BASE_BRANCH:-main}

if git rev-parse --verify "origin/$BASE_BRANCH" >/dev/null 2>&1; then
  git fetch origin "$BASE_BRANCH" --quiet 2>/dev/null
  MERGE_BASE=$(git merge-base HEAD "origin/$BASE_BRANCH")
else
  MERGE_BASE=$(git merge-base HEAD "$BASE_BRANCH")
fi
```

**Assessment**: High value — this is the exact same git logic copied three times, with `code-review-loop` explicitly saying "Use the same resolution logic as `code-review`". Perfect fragment candidate.

#### Fragment 7: `review/feedback-processing`

**Files**: `review-annotations`, `review-pr-comments` (skills)

**Shared text** (~50 lines across multiple sections):

- Complexity assessment criteria (architectural changes, multi-file modifications, unclear requirements, substantial refactoring)
- Plan mode offer template
- User response options list (implement fix, additional context, already resolved, skip, plan mode)
- Resolution confirmation template ("Marking [item] #N as addressed")
- Visual separator pattern (━━━━ lines)
- Behavioral notes ("Don't be defensive", "Ask for clarification", "Propose concrete solutions", "One at a time")

**Parameterization**: `item_type` variable (`"annotation"` vs `"thread"`) controls the few word-level differences.

---

### Cluster 3: Research & Planning Workflow

These fragments consolidate the shared conventions across research and planning skills.

#### Fragment 8: `research/document-conventions`

**Files**: `research-codebase`, `research-web`, `iterate-research` (skills)

**Shared text** (~25 lines across multiple sections):

- Follow-up handling instructions (append to same document, update frontmatter fields)
- Critical ordering rules (read files first, wait for sub-agents, gather metadata before writing)
- Frontmatter consistency rules (always include, keep consistent, update on follow-up, snake_case)
- Research document storage convention (`thoughts/research/YYYY-MM-DD-description.md`)

**Assessment**: High value — these rules appear identically in 3 files. A single fragment would eliminate the drift risk.

#### Fragment 9: `research/sub-agent-catalog`

**Files**: `create-plan`, `research-codebase`, `iterate-research` (skills)

**Shared text** (~20 lines):

```
- Use the **codebase-locator** agent to find WHERE files and components live
- Use the **codebase-analyzer** agent to understand HOW specific code works
- Use the **codebase-pattern-finder** agent to find examples of existing patterns

- Start with locator agents to find what exists
- Then use analyzer agents on the most promising findings
- Run multiple agents in parallel when they're searching for different things
- Each agent knows its job — just tell it what you're looking for
- Don't write detailed prompts about HOW to search — the agents already know
```

**Parameterization**: `create-plan` additionally lists `thoughts-locator` and `thoughts-analyzer`. A `include_thoughts_agents` boolean variable could control this.

#### Fragment 10: `plans/implementation-workflow`

**Files**: `implement-plan`, `implement-plan-worktree` (skills)

**Shared text** (~100 lines):

- Implementation Philosophy block ("Plans are carefully designed, but reality can be messy...")
- Mismatch presentation template ("STOP and think deeply... Expected/Found/Why")
- Verification Approach (read AGENTS.md, run success criteria, code-reviewer dispatch)
- Updating the Plan section (Mark Completion checkboxes, Document Deviations with example, Keep It Useful)
- "If You Get Stuck" section

**Assessment**: Highest single-pair duplication in the entire codebase (~100 lines). `implement-plan-worktree` differs only in path references (`<project-dir>` vs `<worktree-path>`) and punctuation (hyphens vs em-dashes). A `project_path_label` variable handles the difference.

---

### Cluster 4: GitHub & PR Operations

#### Fragment 11: `github/host-detection`

**Files**: `create-pr`, `describe-pr`, `split-branch`, `gh-safe`, `review-pr`, `review-pr-comments` (skills)

**Shared text** (~8 lines):

```
Invoke the **gh-safe** skill to load the GitHub CLI wrapper script path and syntax reference.
All `gh` operations in this command use the `gh-safe` script — never call `gh` directly.

REMOTE_URL=$(git remote get-url origin)
case "$REMOTE_URL" in
  *git.taservs.net*) GH_HOST=git.taservs.net ;;
  *) GH_HOST=github.com ;;
esac
```

**Assessment**: This appears in 6 files. High value — ensures the host detection logic stays consistent.

#### Fragment 12: `github/pr-inline-review`

**Files**: `create-pr`, `describe-pr`, `gh-safe` (skills)

**Shared text** (~40 lines):

- Comment-worthy locations list (non-obvious decisions, subtle behavioral changes, security, performance)
- Hunk line number computation algorithm (@@ header parsing, +new starting line, line counting rules)
- `diff-lines` validation step
- JSON payload format for `post-review`

**Assessment**: The hunk parsing algorithm is particularly dangerous to have duplicated — a bug fix in one place could easily miss the other two. High value for correctness.

#### Fragment 13: `github/pr-description-rules`

**Files**: `create-pr`, `describe-pr` (skills)

**Shared text** (~10 lines):

```
- Be thorough but concise — descriptions should be scannable
- Do NOT use hard line breaks within prose paragraphs or bullet points in the PR description
- Focus on the "why" as much as the "what"
- Include any breaking changes or migration notes prominently
- If the PR touches multiple components, organize the description accordingly
- Do NOT include Claude attribution or co-author information in any output
```

Plus the ticket fabrication warning:

```
If the template requests a ticket/issue/task number and you have not identified one,
ask the user whether they have a ticket number. If they don't, use "N/A".
Do not guess or fabricate a ticket reference.
```

---

### Cross-Cutting Patterns (Lower Priority)

These patterns appear across cluster boundaries and are smaller in scope. They may be better addressed via template variables in a shared config rather than full fragments.

| Pattern                                     | Files                                                    | Lines           |
| ------------------------------------------- | -------------------------------------------------------- | --------------- |
| "Read files fully — never use limit/offset" | 6+ skills                                                | ~2-3 lines each |
| "Read AGENTS.md/CLAUDE.md in each area"     | 7 skills                                                 | ~2 lines each   |
| "No Claude attribution"                     | `commit`, `create-pr`, `describe-pr`, `split-branch`     | ~2-3 lines each |
| "Never use git add -A or git add ."         | `commit`, `resolve-conflicts`, `split-branch`, `gt-sync` | ~1 line each    |
| "Wait for ALL sub-agents to complete"       | 6+ skills                                                | ~1 line each    |
| "Subagents must not ask the user questions" | `split-branch`, `resolve-conflicts`                      | ~1 line each    |

## Prioritized Implementation Order

Based on duplication volume, drift risk, and consistency impact:

| Priority | Fragment                        | Lines Saved     | Files Affected | Drift Risk                                       |
| -------- | ------------------------------- | --------------- | -------------- | ------------------------------------------------ |
| 1        | `plans/implementation-workflow` | ~100            | 2              | High — worktree variant will drift from standard |
| 2        | `github/pr-inline-review`       | ~40 × 3         | 3              | Critical — hunk algorithm must stay in sync      |
| 3        | `research/document-conventions` | ~25 × 3         | 3              | Medium — rules already drifting slightly         |
| 4        | `review/feedback-processing`    | ~50             | 2              | Medium — annotation/thread variants diverging    |
| 5        | `github/host-detection`         | ~8 × 6          | 6              | Low per-file but high breadth                    |
| 6        | `agents/documentarian-mandate`  | ~15 × 3 + 3 × 2 | 5              | Medium — agents already showing wording drift    |
| 7        | `review/merge-base-resolution`  | ~15 × 3         | 3              | High — git logic must stay in sync               |
| 8        | `research/sub-agent-catalog`    | ~20 × 3         | 3              | Low — stable text                                |
| 9        | `review/delegation-rules`       | ~4 × 4          | 4              | Medium — punctuation/wording drift               |
| 10       | `github/pr-description-rules`   | ~10 × 2         | 2              | Low — mostly stable                              |
| 11       | `review/triage-workflow`        | ~15 × 2         | 2-3            | Medium — code-review has extra todo logic        |
| 12       | `agents/no-critique-bullets`    | ~4 × 5          | 5              | Low — intentionally varied per agent             |
| 13       | `agents/file-line-references`   | ~1 × 3          | 3              | Low — single line                                |

## Architecture Considerations

### Variable design

Fragments that serve multiple contexts need carefully designed variables:

- `plans/implementation-workflow`: needs `project_path_label` (e.g., `"<project-dir>"` vs `"<worktree-path>"`)
- `review/feedback-processing`: needs `item_type` (e.g., `"annotation"` vs `"thread"`)
- `research/sub-agent-catalog`: needs `include_thoughts_agents` boolean
- `agents/documentarian-mandate`: needs `role_description` for the heading variant

### Handlebars syntax in bash blocks

Some fragments contain bash code blocks with `${VARIABLE}` syntax. Since Handlebars interprets `{{...}}` but not `${...}`, bash variables are safe. However, any future bash that uses `{{` would conflict — this is a known constraint documented in `agent-docs/compiler-fragments.md`.

## Code References

- `spec/fragments/review/prompt-contract.md` — only existing fragment
- `spec/agents/codebase-analyzer.md:28-36, 161-165` — documentarian mandate (opening + closing)
- `spec/agents/codebase-locator.md:27-34, 140-144` — documentarian mandate variant
- `spec/agents/codebase-pattern-finder.md:28-36, 251-255` — documentarian mandate variant
- `spec/skills/implement-plan/SKILL.md:53-164` — implementation workflow (full block)
- `spec/skills/implement-plan-worktree/SKILL.md:155-297` — implementation workflow duplicate
- `spec/skills/create-pr/SKILL.md:196-260` — inline review comments section
- `spec/skills/describe-pr/SKILL.md:111-175` — inline review comments duplicate
- `spec/skills/gh-safe/SKILL.md:130-140` — hunk algorithm third copy
- `spec/skills/code-review/SKILL.md:42-63` — merge-base resolution
- `spec/skills/code-review-loop/SKILL.md:52-67` — merge-base resolution duplicate
- `spec/skills/research-codebase/SKILL.md:207-221` — document conventions
- `spec/skills/research-web/SKILL.md:169-187` — document conventions variant
- `spec/skills/iterate-research/SKILL.md:240-265` — document conventions duplicate
- `spec/skills/review-annotations/SKILL.md:96-166` — feedback processing workflow
- `spec/skills/review-pr-comments/SKILL.md:163-237` — feedback processing duplicate
- `spec/skills/create-pr/SKILL.md:36-48` — gh-safe loading + host detection
- `spec/skills/review-pr/SKILL.md:37-50` — gh-safe loading + host detection duplicate
- `spec/skills/review-pr-comments/SKILL.md:47-58` — gh-safe loading + host detection duplicate

## Open Questions

1. **Fragment granularity**: Should `plans/implementation-workflow` be one large ~100-line fragment, or broken into sub-fragments (philosophy, mismatch, verification, updating)?

2. **Conditional content in fragments**: Some fragments need conditional sections (e.g., `research/sub-agent-catalog` with optional thoughts agents). Handlebars supports `{{#if variable}}` blocks — should we use this, or keep fragments simpler and accept some remaining duplication?

3. **Cross-cluster patterns**: The small cross-cutting patterns (read files fully, no attribution, etc.) each appear as ~1-3 lines across many files. Are these worth fragmenting, or is the invocation overhead (`{{> ...}}`) not worth it for single-line patterns?

4. **Fragment file organization**: The current convention uses `spec/fragments/<category>/<name>.md`. Proposed categories: `agents/`, `review/`, `research/`, `plans/`, `github/`. Does this align with how the fragments will be maintained?
