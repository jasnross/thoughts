# Migrate Hardcoded Spec References to Template Syntax

## Overview

Migrate all hardcoded spec name references in `dotfiles/agent-config/spec/` body
content to the `{{ specs.<type>.<id>.name }}` keyed access syntax. This ensures
spec names resolve correctly when `agentspec sync` applies a prefix (e.g., `tw-`),
eliminates the manual `tw:gh-safe` prefix hack, and produces compile errors when
referenced specs are renamed or removed.

## Current State Analysis

The spec library contains ~30+ hardcoded references to other spec names scattered
across rules, skills, and fragments. When `agentspec sync` applies the `tw` prefix,
these hardcoded names don't match the actual model-facing names, causing the AI to
reference specs that don't exist under that name.

The `{{ specs.<type>.<id>.name }}` syntax is already implemented in agentspec and
documented in the README. Two files already use the iteration syntax
(`specs.agents` in `sub-agents.md` and `sub-agent-catalog.md`), but no files use
the keyed single-item access syntax yet.

### Key Discoveries:

- The keyed access syntax uses underscores for IDs: `gh-safe` → `gh_safe`,
  `code-reviewer` → `code_reviewer` (`spec/agents/code-reviewer.md:2`,
  `agentspec/README.md:272`)
- Template syntax works in both spec bodies and included fragments — they share
  the same MiniJinja context (`agentspec/README.md:305`)
- Three access patterns: `specs.skill.<id>.name`, `specs.agent.<id>.name`,
  `specs.rule.<id>.name` (`agentspec/src/templating/context.rs:28-29`)
- `gh-safe.sh` (the script filename) and `git-town sync` (CLI commands) are NOT
  spec names and must not be migrated
- Frontmatter `description:` fields are YAML, not template-rendered body content
  — they are out of scope

## Desired End State

Every spec-name reference in body content resolves through `{{ specs.*.name }}`
template syntax. No hardcoded spec names remain. `agentspec compile` and
`agentspec validate` pass cleanly.

### How to verify:

1. `agentspec validate` exits 0
2. `agentspec compile` exits 0 and produces output with correct resolved names
3. `agentspec sync --dry-run` shows expected `tw-` prefixed names
4. Grep for residual hardcoded spec names in body content returns no matches
   (excluding frontmatter `id:`, script filenames, and CLI commands)

## What We're NOT Doing

- **Fragment path references**: `{% include "review/prompt-contract.md" %}` uses
  filesystem paths, not spec names — unaffected by prefixing
- **Frontmatter fields**: `id:` and `description:` are YAML, not template-rendered
- **Script filenames**: `gh-safe.sh` is a bundled script, not a spec name
- **CLI commands**: `git-town sync`, `git-town continue`, `git-town branch`, etc.
  are external tool invocations, not spec references
- **agentspec code changes**: The template feature is already implemented
- **Adding new specs or changing spec behavior**: Pure reference migration only

## Implementation Approach

One batch migration. All changes are mechanical search-and-replace within body
content. A single `agentspec compile` run verifies everything. The changes group
naturally into two categories: skill-name references and agent-name references.

## Implementation

### Overview

Replace every hardcoded spec name reference in spec body content with the
corresponding `{{ specs.<type>.<id>.name }}` template expression.

### Migration Reference

| Hardcoded Name | Template Replacement |
|---|---|
| `gh-safe` (skill) | `{{ specs.skill.gh_safe.name }}` |
| `git-town` (skill) | `{{ specs.skill.git_town.name }}` |
| `resolve-conflicts` (skill) | `{{ specs.skill.resolve_conflicts.name }}` |
| `thoughts-dir-resolver` (skill) | `{{ specs.skill.thoughts_dir_resolver.name }}` |
| `code-reviewer` (agent) | `{{ specs.agent.code_reviewer.name }}` |
| `codebase-locator` (agent) | `{{ specs.agent.codebase_locator.name }}` |
| `codebase-analyzer` (agent) | `{{ specs.agent.codebase_analyzer.name }}` |
| `codebase-pattern-finder` (agent) | `{{ specs.agent.codebase_pattern_finder.name }}` |
| `web-search-researcher` (agent) | `{{ specs.agent.web_search_researcher.name }}` |
| `thoughts-locator` (agent) | `{{ specs.agent.thoughts_locator.name }}` |
| `thoughts-analyzer` (agent) | `{{ specs.agent.thoughts_analyzer.name }}` |
| `learnings-discoverer` (agent) | `{{ specs.agent.learnings_discoverer.name }}` |
| `discover-repo` (skill) | `{{ specs.skill.discover_repo.name }}` |
| `describe-pr` (skill) | `{{ specs.skill.describe_pr.name }}` |
| `create-plan` (skill) | `{{ specs.skill.create_plan.name }}` |
| `implement-plan` (skill) | `{{ specs.skill.implement_plan.name }}` |
| `validate-plan` (skill) | `{{ specs.skill.validate_plan.name }}` |
| `commit` (skill) | `{{ specs.skill.commit.name }}` |
| `review-files` (skill) | `{{ specs.skill.review_files.name }}` |
| `code-review` (skill) | `{{ specs.skill.code_review.name }}` |
| `learn` (skill) | `{{ specs.skill.learn.name }}` |
| `split-branch` (skill) | `{{ specs.skill.split_branch.name }}` |
| `tw:gh-safe` (prefixed skill) | `{{ specs.skill.gh_safe.name }}` |

### What NOT to migrate (false positives to skip):

- `gh-safe.sh` — script filename, not a spec name
- `git-town sync`, `git-town continue`, `git-town branch`, `git-town config`,
  `git-town set-parent`, `git-town propose`, `git-town undo`, `git-town help` —
  CLI commands
- `id:` values in YAML frontmatter — these define the spec
- `description:` values in YAML frontmatter — not template-rendered
- Self-referential text where a skill explains its own invocation syntax with
  examples that are templates themselves (e.g., `<skill-base-dir>/resolve-thoughts-dir.sh`)
  — these are filesystem paths, not spec names

### Changes Required:

#### 1. Rules

**File**: `spec/rules/git-conventions.md`

- [x] Line 106: `load the \`tw:gh-safe\` skill first` → `load the \`{{ specs.skill.gh_safe.name }}\` skill first`
- [x] Line 113: `without checking gh-safe first` → `without checking {{ specs.skill.gh_safe.name }} first`
- [x] Line 117: `# 1. Load tw:gh-safe` → `# 1. Load {{ specs.skill.gh_safe.name }}`
- [x] Line 122: `what gh-safe supports` → `what {{ specs.skill.gh_safe.name }} supports`

**File**: `spec/rules/knowledge-base.md`

- [x] Line 8: `` the `learnings-discoverer` subagent `` → `` the `{{ specs.agent.learnings_discoverer.name }}` subagent ``

**File**: `spec/rules/sub-agents.md`

- [x] Line 22: `` Dispatch `learnings-discoverer` before starting tasks `` → `` Dispatch `{{ specs.agent.learnings_discoverer.name }}` before starting tasks ``
- [x] Line 26: `` `codebase-locator` + `thoughts-locator` `` → `` `{{ specs.agent.codebase_locator.name }}` + `{{ specs.agent.thoughts_locator.name }}` ``

#### 2. Skills — gh-safe references

**File**: `spec/skills/create-pr/SKILL.md`

- [x] Line 29: `` Load the `gh-safe` skill `` → `` Load the `{{ specs.skill.gh_safe.name }}` skill ``
- [x] Line 31: `the **gh-safe** skill` → `the **{{ specs.skill.gh_safe.name }}** skill`

**File**: `spec/skills/describe-pr/SKILL.md`

- [x] Line 25: `` Load the `gh-safe` skill `` → `` Load the `{{ specs.skill.gh_safe.name }}` skill ``
- [x] Line 27: `the **gh-safe** skill` → `the **{{ specs.skill.gh_safe.name }}** skill`

**File**: `spec/skills/review-pr/SKILL.md`

- [x] Line 26: `` Load the `gh-safe` skill `` → `` Load the `{{ specs.skill.gh_safe.name }}` skill ``
- [x] Line 28: `the **gh-safe** skill` → `the **{{ specs.skill.gh_safe.name }}** skill`

**File**: `spec/skills/review-pr-comments/SKILL.md`

- [x] Line 32: `` Load the `gh-safe` skill `` → `` Load the `{{ specs.skill.gh_safe.name }}` skill ``
- [x] Line 34: `the **gh-safe** skill` → `the **{{ specs.skill.gh_safe.name }}** skill`

**File**: `spec/skills/split-branch/SKILL.md`

- [x] Line 437: `` the `gh-safe` skill `` → `` the `{{ specs.skill.gh_safe.name }}` skill ``

#### 3. Skills — git-town and resolve-conflicts references

**File**: `spec/skills/split-branch/SKILL.md`

- [x] Line 83: `` the `git-town` skill `` → `` the `{{ specs.skill.git_town.name }}` skill ``
- [x] Line 519: `` the `git-town` skill `` → `` the `{{ specs.skill.git_town.name }}` skill ``

**File**: `spec/skills/gt-sync/SKILL.md`

- [x] Line 52: `` the `git-town` skill `` → `` the `{{ specs.skill.git_town.name }}` skill ``
- [x] Line 36: `` the `resolve-conflicts` skill `` → `` the `{{ specs.skill.resolve_conflicts.name }}` skill ``
- [x] Line 188: `` the `resolve-conflicts` skill `` → `` the `{{ specs.skill.resolve_conflicts.name }}` skill ``

#### 4. Skills — code-reviewer references

**File**: `spec/skills/review-pr/SKILL.md`

- [x] Line 56: `the **discover-repo** skill` → `the **{{ specs.skill.discover_repo.name }}** skill`
- [x] Line 18: `the code-reviewer agent` → `the {{ specs.agent.code_reviewer.name }} agent`
- [x] Line 137: `the code-reviewer agent` → `the {{ specs.agent.code_reviewer.name }} agent`
- [x] Line 152: `the code-reviewer agent` (heading) → `the {{ specs.agent.code_reviewer.name }} agent`
- [x] Line 154: `subagent_type: "code-reviewer"` → `subagent_type: "{{ specs.agent.code_reviewer.name }}"`
- [x] Line 161: `the code-reviewer agent` → `the {{ specs.agent.code_reviewer.name }} agent`
- [x] Line 165: `the code-reviewer's structured output` → `the {{ specs.agent.code_reviewer.name }}'s structured output`
- [x] Line 191: `The code-reviewer agent works` → `The {{ specs.agent.code_reviewer.name }} agent works`
- [x] Line 217: `the code-reviewer reaches` → `the {{ specs.agent.code_reviewer.name }} reaches`
- [x] Line 361: `the code-reviewer agent` → `the {{ specs.agent.code_reviewer.name }} agent`

**File**: `spec/skills/review-files/SKILL.md`

- [x] Line 17: `a specialized code-reviewer agent` → `a specialized {{ specs.agent.code_reviewer.name }} agent`
- [x] Line 63: `the code-reviewer agent` → `the {{ specs.agent.code_reviewer.name }} agent`
- [x] Line 64: `subagent_type: "code-reviewer"` → `subagent_type: "{{ specs.agent.code_reviewer.name }}"`
- [x] Line 65: `` that `code-reviewer` expects `` → `` that `{{ specs.agent.code_reviewer.name }}` expects ``
- [x] Line 76: `The code-reviewer agent` → `The {{ specs.agent.code_reviewer.name }} agent`

**File**: `spec/skills/code-review-loop/SKILL.md`

- [x] Line 95: `subagent_type: "code-reviewer"` → `subagent_type: "{{ specs.agent.code_reviewer.name }}"`

#### 5. Skills — research agent references

**File**: `spec/skills/create-plan/SKILL.md`

- [x] Line 78: `the **codebase-locator** agent` → `the **{{ specs.agent.codebase_locator.name }}** agent`
- [x] Line 79: `the **codebase-analyzer** agent` → `the **{{ specs.agent.codebase_analyzer.name }}** agent`
- [x] Line 80: `the **thoughts-locator** agent` → `the **{{ specs.agent.thoughts_locator.name }}** agent`
- [x] Line 136: `**codebase-locator**` → `**{{ specs.agent.codebase_locator.name }}**`
- [x] Line 137: `**codebase-analyzer**` → `**{{ specs.agent.codebase_analyzer.name }}**`
- [x] Line 138: `**codebase-pattern-finder**` → `**{{ specs.agent.codebase_pattern_finder.name }}**`
- [x] Line 141: `**thoughts-locator**` → `**{{ specs.agent.thoughts_locator.name }}**`
- [x] Line 142: `**thoughts-analyzer**` → `**{{ specs.agent.thoughts_analyzer.name }}**`

**File**: `spec/skills/create-learning-plan/SKILL.md`

- [x] Line 82: `**web-search-researcher**` → `**{{ specs.agent.web_search_researcher.name }}**`
- [x] Line 83: `**codebase-pattern-finder**` → `**{{ specs.agent.codebase_pattern_finder.name }}**`
- [x] Line 84: `**codebase-analyzer**` → `**{{ specs.agent.codebase_analyzer.name }}**`
- [x] Line 423: `**web-search-researcher**` → `**{{ specs.agent.web_search_researcher.name }}**`
- [x] Line 424: `**thoughts-locator**` → `**{{ specs.agent.thoughts_locator.name }}**`
- [x] Line 427: `**codebase-pattern-finder**` → `**{{ specs.agent.codebase_pattern_finder.name }}**`
- [x] Line 428: `**codebase-analyzer**` → `**{{ specs.agent.codebase_analyzer.name }}**`

**File**: `spec/skills/research-web/SKILL.md`

- [x] Line 64: `the **web-search-researcher** agent` → `the **{{ specs.agent.web_search_researcher.name }}** agent`

**File**: `spec/skills/vet-repo/SKILL.md`

- [x] Line 73: `(codebase-analyzer)` → `({{ specs.agent.codebase_analyzer.name }})`
- [x] Line 93: `(codebase-analyzer)` → `({{ specs.agent.codebase_analyzer.name }})`
- [x] Line 112: `(codebase-analyzer)` → `({{ specs.agent.codebase_analyzer.name }})`
- [x] Line 127: `(codebase-analyzer)` → `({{ specs.agent.codebase_analyzer.name }})`
- [x] Line 137: `(codebase-analyzer)` → `({{ specs.agent.codebase_analyzer.name }})`
- [x] Line 148: `(codebase-analyzer)` → `({{ specs.agent.codebase_analyzer.name }})`

#### 6. Skills — slash command references

Slash commands like `/describe-pr` use the spec's model-facing name, which gets
prefixed during sync. These need migration so the suggested command matches the
actual registered name.

**File**: `spec/skills/create-pr/SKILL.md`

- [x] Line 60: `` `/describe-pr` `` → `` `/{{ specs.skill.describe_pr.name }}` ``
- [x] Line 168: `describe-pr conventions` → `{{ specs.skill.describe_pr.name }} conventions`
- [x] Line 174: `` `/describe-pr` `` → `` `/{{ specs.skill.describe_pr.name }}` ``

**File**: `spec/skills/create-idea/SKILL.md`

- [x] Line 179: `` `/create-plan` `` → `` `/{{ specs.skill.create_plan.name }}` ``

**File**: `spec/skills/split-branch/SKILL.md`

- [x] Line 22: `` `/implement_plan` `` → `` `/{{ specs.skill.implement_plan.name }}` ``
- [x] Line 22: `` `/commit` `` → `` `/{{ specs.skill.commit.name }}` ``
- [x] Line 42: `` `/split_branch` `` → `` `/{{ specs.skill.split_branch.name }}` ``

**File**: `spec/skills/validate-plan/SKILL.md`

- [x] Line 222: `` `/implement_plan` `` → `` `/{{ specs.skill.implement_plan.name }}` ``
- [x] Line 223: `` `/commit` `` → `` `/{{ specs.skill.commit.name }}` ``
- [x] Line 224: `` `/validate_plan` `` → `` `/{{ specs.skill.validate_plan.name }}` ``
- [x] Line 225: `` `/describe_pr` `` → `` `/{{ specs.skill.describe_pr.name }}` ``

**File**: `spec/skills/resolve-conflicts/SKILL.md`

- [x] Line 28: `` `/resolve-conflicts` `` → `` `/{{ specs.skill.resolve_conflicts.name }}` ``
- [x] Line 65: `` `/resolve-conflicts` `` → `` `/{{ specs.skill.resolve_conflicts.name }}` ``
- [x] Line 244: `` `/resolve-conflicts` `` → `` `/{{ specs.skill.resolve_conflicts.name }}` ``

**File**: `spec/skills/review-files/SKILL.md`

- [x] Lines 28, 36-39: `/review-files` → `/{{ specs.skill.review_files.name }}`
  (usage example block — replace all 5 occurrences)

**File**: `spec/skills/code-review/SKILL.md`

- [x] Line 66: `/code-review` → `/{{ specs.skill.code_review.name }}`
- [x] Line 75: `/code-review` → `/{{ specs.skill.code_review.name }}`

**File**: `spec/skills/create-plan/SKILL.md`

- [x] Line 608: `/create-plan` → `/{{ specs.skill.create_plan.name }}`

**File**: `spec/skills/learn/SKILL.md`

- [x] Line 38: `` `/learn` `` → `` `/{{ specs.skill.learn.name }}` ``
- [x] Line 40: `` `/learn` `` → `` `/{{ specs.skill.learn.name }}` ``

**File**: `spec/fragments/review/arg-parsing.md`

- [x] Line 5: `` `/review-files` `` → `` `/{{ specs.skill.review_files.name }}` ``

#### 7. Fragments

**File**: `spec/fragments/review/delegation-rules.md`

- [x] Line 1: `the code-reviewer agent` → `the {{ specs.agent.code_reviewer.name }} agent`

**File**: `spec/fragments/plans/implementation-workflow.md`

- [x] Line 35: `subagent_type: "code-reviewer"` → `subagent_type: "{{ specs.agent.code_reviewer.name }}"`

**File**: `spec/fragments/research/sub-agent-catalog.md`

- [x] Line 9: `` `learnings-discoverer` `` → `` `{{ specs.agent.learnings_discoverer.name }}` ``
- [x] Line 9: `` `thoughts-locator` `` → `` `{{ specs.agent.thoughts_locator.name }}` ``
- [x] Line 9: `` `thoughts-analyzer` `` → `` `{{ specs.agent.thoughts_analyzer.name }}` ``

**File**: `spec/fragments/thoughts/discover-dir.md`

- [x] Line 9: `` the `thoughts-dir-resolver` skill `` → `` the `{{ specs.skill.thoughts_dir_resolver.name }}` skill ``
- [x] Line 37: `the thoughts-dir-resolver helper` → `the {{ specs.skill.thoughts_dir_resolver.name }} helper`

**File**: `spec/fragments/workspace/discover-root.md`

- [x] Line 6: `` the `thoughts-dir-resolver` skill `` → `` the `{{ specs.skill.thoughts_dir_resolver.name }}` skill ``

#### Additional References Found During Implementation

> **Deviation:** The following references were not in the original plan but were
> discovered during grep verification. They have all been migrated.

**File**: `spec/skills/code-review-loop/SKILL.md`

- [x] Line 20: `code-reviewer agent` (prose intro) → `{{ specs.agent.code_reviewer.name }} agent`
- [x] Line 93: `#### 4b. Spawn the code-reviewer agent` (heading) → `{{ specs.agent.code_reviewer.name }}`
- [x] Line 222: `` `code-review` `` (skill reference) → `` `{{ specs.skill.code_review.name }}` ``

**File**: `spec/skills/code-review/SKILL.md`

- [x] Line 15: `code-reviewer agent` (prose) → `{{ specs.agent.code_reviewer.name }} agent`
- [x] Line 32: `### 3. Spawn the code-reviewer agent` (heading) → `{{ specs.agent.code_reviewer.name }}`
- [x] Line 81: `The code-reviewer agent` (prose) → `The {{ specs.agent.code_reviewer.name }} agent`

**File**: `spec/skills/iterate-plan/SKILL.md`

- [x] Line 92: `**codebase-locator**` → `**{{ specs.agent.codebase_locator.name }}**`
- [x] Line 93: `**codebase-analyzer**` → `**{{ specs.agent.codebase_analyzer.name }}**`
- [x] Line 94: `**codebase-pattern-finder**` → `**{{ specs.agent.codebase_pattern_finder.name }}**`

**File**: `spec/fragments/plans/implementation-workflow.md`

- [x] Line 34: `Spawn a code-reviewer agent` (prose) → `Spawn a {{ specs.agent.code_reviewer.name }} agent`

**File**: `spec/skills/gt-sync/SKILL.md`

- [x] Line 38: `` `resolve-conflicts` skill `` (prose) → `` `{{ specs.skill.resolve_conflicts.name }}` skill ``

**File**: `spec/skills/create-plan/SKILL.md`

- [x] Line 454: `` `implement-plan` command `` → `` `{{ specs.skill.implement_plan.name }}` command ``
- [x] Line 506: `` `create-plan` and `implement-plan` `` → `` `{{ specs.skill.create_plan.name }}` and `{{ specs.skill.implement_plan.name }}` ``

**File**: `spec/skills/resolve-conflicts/SKILL.md` (found in code review)

- [x] Line 29: `` `gt-sync` `` → `` `{{ specs.skill.gt_sync.name }}` ``
- [x] Line 34: `` `gt-sync` `` → `` `{{ specs.skill.gt_sync.name }}` ``

#### Tests

- [x] Run `agentspec validate` from `agent-config/` — must pass with no errors
- [x] Run `agentspec compile` from `agent-config/` — must succeed and produce
  output in `generated/` with resolved names
- [x] Spot-check a generated file (e.g., `generated/claude/rules/tw-git-conventions.md`)
  to confirm `tw:gh-safe` was replaced with `tw-gh-safe` (or whatever the prefix resolves to)
- [x] Run `agentspec sync --dry-run` — must show expected prefixed names
- [x] Grep verification: search `spec/` body content for residual hardcoded spec
  names. Use a pattern that excludes frontmatter `id:`/`description:`, script
  filenames (`gh-safe.sh`), CLI commands (`git-town sync|continue|branch|config|set-parent|propose|undo|help|compress|detach|ship|rename`),
  and `{% include %}` paths. Expect zero matches.

### Success Criteria:

#### Automated Verification:

- [x] `agentspec validate` passes: `cd agent-config && agentspec validate`
- [x] `agentspec compile` passes: `cd agent-config && agentspec compile`
- [x] `agentspec sync --dry-run` shows prefixed names
- [x] Grep for residual hardcoded spec names in body content returns zero results
  (see grep verification pattern in Tests section)

## Prerequisite: agentspec Binary Update

> **Deviation:** The installed `agentspec` binary (v0.1.0 via mise/homebrew) did
> not include the keyed access feature (`specs.skill.<id>.name`). The binary was
> rebuilt from source (`cargo install --path agentspec/`) to include commit
> `a662802 feat(templating): add prefix-aware keyed spec access in templates`.
> Without this update, `agentspec validate` fails with "undefined value" errors
> on any `{{ specs.skill.*.name }}` or `{{ specs.agent.*.name }}` expressions.

## Performance Considerations

Template resolution adds negligible overhead — it runs once at compile time, not
at runtime. The `{{ specs.*.name }}` expressions are simple key lookups into a
pre-built context map.

## References

- Prefix-aware spec references implementation plan: `thoughts/plans/2026-04-11-agentspec-prefix-aware-spec-references.md`
- Idea document for this migration: `thoughts/ideas/2026-04-11-dotfiles-migrate-hardcoded-spec-references.md`
- agentspec README (Built-in variables): `agentspec/README.md:260-306`
- Keyed access test fixture: `agentspec/tests/fixtures/agent-config/spec/skills/basic-skill/SKILL.md:13`
