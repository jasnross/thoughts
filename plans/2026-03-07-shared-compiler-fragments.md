# Shared Compiler Fragments Implementation Plan

## Overview

Add a Handlebars-based fragment mechanism to the skill compiler, enabling shared
instruction text across canonical skill definitions. Fragments live in
`spec/fragments/` and are resolved at compile time, producing self-contained
generated output. The first extraction target is the code-reviewer prompt
contract, which is duplicated (with drift) across four review skills.

## Current State Analysis

### Compiler Pipeline

The TypeScript compiler (`scripts/compile_agents.ts`) processes canonical specs
through a 7-stage pipeline:

1. Load mappings (`mappings.ts:64`)
2. Load specs (`parse.ts:84`) — gray-matter splits YAML frontmatter + Markdown body
3. Schema validation (`validate.ts:29`) — Ajv against `canonical.schema.json`
4. Normalization (`validate.ts:50`) — defaults, dedup, sort
5. Semantic validation (`validate.ts:108`) — cross-cutting constraints
6. Compilation (`compile.ts:14`) — dispatch to per-provider adapters
7. Emission (`emit.ts:23`) — write files + manifest

Fragment resolution will slot between stages 2 and 3: after `loadCanonicalSpecs()`
returns parsed specs (line 81) but before `validateSchema()` runs (line 83). At
that point, `spec.body` is a plain string ready for Handlebars rendering.

### Duplicated Prompt Contract

Four skills share the same code-reviewer prompt contract structure, but field
descriptions have drifted:

| Skill | Lines | Project path phrasing | Nullable syntax |
|---|---|---|---|
| `code-review` | `SKILL.md:82-89` | `<absolute-or-workspace-relative path>` | `"none"` in quotes |
| `code-review-loop` | `SKILL.md:121-128` | `<absolute-or-workspace-relative path>` | No `"none"` qualifier |
| `review-files` | `SKILL.md:78-85` | `<current working directory>` | Literal `none` |
| `review-pr` | `SKILL.md:101-108` | `<absolute path>` | `<MERGE_BASE sha>` mixed style |

### Key Discoveries

- No existing include/fragment mechanism in the codebase
- No compiler tests exist — changes must be verified carefully
- No `{{` or `}}` syntax exists in any current spec body, so Handlebars
  processing is safe to apply globally
- The project uses `tsx` for TypeScript execution and Node 20+ (has built-in
  `node:test` runner)
- Generated output preserves the body as-is — adapters only transform frontmatter
- Dependencies: `gray-matter`, `ajv`, `yaml` — no templating engine currently

## Desired End State

After this plan is complete:

1. The compiler supports Handlebars partials (`{{> fragment-name}}`) in canonical
   skill/agent bodies, resolved at compile time
2. A shared prompt contract fragment exists at `spec/fragments/review/prompt-contract.md`
3. All four review skills reference the shared fragment instead of duplicating
   the prompt contract
4. Generated output is byte-for-byte identical to current output (or
   functionally equivalent where drift is corrected)
5. Validation catches missing fragments and circular includes
6. A comprehensive test suite covers the compiler pipeline

### Verification

- `npm run build:agents` succeeds with no errors or warnings
- `npm run test` passes all tests
- `git diff generated/` shows no unintended changes (or shows only drift
  corrections that are intentional)

## What We're NOT Doing

- **Triage workflow extraction** — has meaningful variants requiring deeper
  parameterization design; deferred to a follow-up plan
- **Merge-base resolution extraction** — executable logic better suited as a
  shell script, not a compiler fragment
- **GitHub host detection extraction** — better folded into `gh-safe`
- **Runtime cross-references** — fragments are resolved at compile time only
- **Nested fragment includes** — fragments cannot include other fragments in
  this iteration (simplifies circular reference detection)
- **Non-review skill families** — solve the review family first, then evaluate
  broader applicability

## Implementation Approach

**Handlebars** is the chosen templating engine because:
- It provides partials (`{{> name}}`), variables (`{{value}}`), and conditionals
  (`{{#if}}`) natively — all needed for current and upcoming use cases
- It's JS-native, logic-less by default, and widely known
- Provider-specific conditionals will be valuable in future fragment work

Fragment resolution is a **pre-processing step** inserted into the existing
pipeline. All specs pass through Handlebars rendering; specs without template
syntax are unchanged (Handlebars is a no-op on plain text).

---

## Phase 1: Establish Compiler Test Suite

### Overview

Add a comprehensive test suite for the existing compiler pipeline using Node's
built-in `node:test` runner and `tsx` for TypeScript execution. This creates a
safety net before modifying the compiler.

### Changes Required

#### 1. Add test npm script

**File**: `package.json`

- [x] Add `"test"` script: `"tsx --test scripts/__tests__/*.test.ts"`
  > **Deviation:** Used unquoted single-level glob (`*.test.ts`) instead of
  > quoted `**` glob. The quoted `**` glob was not expanded by `tsx` (it treats
  > the literal path as a filename). Since all test files live at one directory
  > level, the simpler glob works correctly with shell expansion.

#### 2. Create test fixtures

**Directory**: `scripts/__tests__/fixtures/`

- [x] Create `scripts/__tests__/fixtures/spec/skills/test-skill/SKILL.md` — a
      minimal valid skill spec with frontmatter and body
- [x] Create `scripts/__tests__/fixtures/spec/skills/test-skill-with-support/SKILL.md` —
      a skill with a supporting file
- [x] Create `scripts/__tests__/fixtures/spec/skills/test-skill-with-support/helper.sh` —
      an executable supporting file (mode 0o755)
- [x] Create `scripts/__tests__/fixtures/spec/agents/test-agent.md` — a minimal
      valid agent spec
- [x] ~~Create `scripts/__tests__/fixtures/spec/schema/canonical.schema.json`~~ — skipped
  > **Deviation:** Tests use real project schemas and mappings via `PROJECT_ROOT`
  > instead of fixture copies. This ensures tests break when schemas/mappings
  > change (desirable). Removed fixture `spec/schema/` and `mappings/` dirs.
- [x] ~~Create `scripts/__tests__/fixtures/mappings/`~~ — skipped (see above)
  > **Deviation:** `loadProjectMappings()` in `helpers.ts` loads real mappings
  > with caching. The `makeMinimalMappings()` inline helper was also removed.

#### 3. Test: parse module

**File**: `scripts/__tests__/parse.test.ts`

- [x] Test `loadCanonicalSpecs()` loads agent specs from `spec/agents/`
- [x] Test `loadCanonicalSpecs()` loads skill specs from `spec/skills/`
- [x] Test skill specs get `kind: "skill"` injected regardless of frontmatter
- [x] Test supporting files are collected with correct `relativeName` and `mode`
- [x] Test skill directory with no `.md` file throws an error
- [x] Test skill directory with multiple `.md` files throws an error
- [x] Test missing `spec/agents/` directory is handled gracefully (returns empty)
- [x] Test missing `spec/skills/` directory is handled gracefully (returns empty)

#### 4. Test: validate module

**File**: `scripts/__tests__/validate.test.ts`

- [x] Test `validateSchema()` passes for valid frontmatter
- [x] Test `validateSchema()` reports errors for missing required fields
- [x] Test `validateSchema()` reports errors for invalid field types
- [x] Test `normalizeSpecs()` sets defaults (name from id, invocability false,
      targets to ALL_PROVIDERS)
- [x] Test `normalizeSpecs()` deduplicates and sorts tools
- [x] Test `normalizeSpecs()` throws on missing required string fields
- [x] Test `validateSemantics()` detects duplicate IDs
- [x] Test `validateSemantics()` detects empty body
- [x] Test `validateSemantics()` detects skills with no invocability flags
- [x] Test `validateSemantics()` validates `delegate_to` references
- [x] Test `validateSemantics()` validates model profile mappings exist

#### 5. Test: compile module

**File**: `scripts/__tests__/compile.test.ts`

- [x] Test `compileSpecs()` dispatches to correct adapter per provider
- [x] Test `compileSpecs()` respects target filtering
- [x] Test `compileSpecs()` skips specs whose kind is unsupported by a provider
- [x] Test `compileSpecs()` produces deterministic `sourceHash`

#### 6. Test: format module

**File**: `scripts/__tests__/format.test.ts`

- [x] Test `renderMarkdownWithFrontmatter()` produces correct YAML + body output
- [x] Test `stableJson()` sorts keys deterministically

#### 7. Test: emit module

**File**: `scripts/__tests__/emit.test.ts`

- [x] Test `writeGeneratedFiles()` creates files at correct paths
- [x] Test `writeGeneratedFiles()` sets executable mode when specified
- [x] Test `writeGeneratedFiles()` cleans target directories before writing
- [x] Test `checkGeneratedState()` reports missing files
- [x] Test `checkGeneratedState()` reports content mismatches
- [x] Test `checkGeneratedState()` reports unexpected files

#### 8. Test: model module

**File**: `scripts/__tests__/model.test.ts`

- [x] Test `resolveProviderModelConfig()` returns model from plain string
- [x] Test `resolveProviderModelConfig()` extracts model and options from object
- [x] Test `resolveProviderModelConfig()` returns undefined for missing provider

### Success Criteria

#### Automated Verification

- [x] `npm run test` passes all tests
- [x] Tests cover all existing compiler modules: parse, validate, compile,
      format, emit, model
- [x] Test fixtures are self-contained (don't depend on real spec files)
  > **Deviation:** Tests intentionally use real project schemas and mappings
  > (via `PROJECT_ROOT`) so changes to those files surface as test failures.
  > Only spec fixtures (test-skill, test-agent) are test-specific.

#### Manual Verification

- [ ] Review test coverage to confirm key edge cases are exercised
- [ ] Verify test fixture specs are realistic enough to catch real issues

---

## Phase 2: Integrate Handlebars and Implement Fragment Resolution

### Overview

Add the `handlebars` npm dependency, build a fragment loader that registers
`spec/fragments/` files as Handlebars partials, and wire resolution into the
pipeline between spec loading and schema validation. Add validation for missing
fragments and circular references.

### Changes Required

#### 1. Add Handlebars dependency

**File**: `package.json`

- [x] Add `"handlebars": "^4.7.8"` to `devDependencies`
- [x] Run `npm install`

#### 2. Create the fragment resolver module

**File**: `scripts/lib/fragments.ts`

- [x] Implement `loadFragments(baseDir: string): Promise<Map<string, string>>`
  - Walk `spec/fragments/` recursively for `.md` files
  - For each file, compute its partial name from its path relative to
    `spec/fragments/` with the `.md` extension stripped
    (e.g., `spec/fragments/review/prompt-contract.md` → `review/prompt-contract`)
  - Return a `Map<partialName, content>`
  - If `spec/fragments/` does not exist, return an empty map (graceful no-op)

- [x] Implement `resolveFragments(specs: CanonicalSpec[], fragments: Map<string, string>): CanonicalSpec[]`
  - Import Handlebars and create an **isolated instance** via `Handlebars.create()`
    (avoids polluting the global Handlebars state, important for testability)
  - **Disable HTML escaping**: pass `{ noEscape: true }` to `hbs.compile()`.
    > **Deviation:** The plan specified setting `hbs.noEscape = true` on the
    > instance, but that property doesn't propagate to partials. Passing
    > `noEscape` as a compile option works correctly.
  - Register all fragments as partials: `handlebars.registerPartial(name, content)`
  - For each spec, compile and render `spec.body` through Handlebars with an
    **empty context** `{}`. Partials receive their context from inline hash
    arguments (e.g., `{{> review/prompt-contract project_path="..."}}`)
  - Return new `CanonicalSpec[]` with resolved bodies (do not mutate originals)
  - On Handlebars errors (missing partial, syntax error), throw with
    `${spec.sourcePath}: fragment resolution failed: ${error.message}`

- [x] Implement `validateFragments(fragments: Map<string, string>): string[]`
  - Check for nested fragment includes using **Handlebars AST parsing**
    (not naive string scanning, which would false-positive on documentation
    text that mentions `{{>` syntax):
    - Parse each fragment with `Handlebars.parse(content)` — this is a
      stateless read-only operation on the global Handlebars module (no
      isolated instance needed; it does not register partials or modify state)
    - Walk the AST looking for `PartialStatement` nodes
    - If any are found, report an error: nested fragments are not supported
      in this iteration

#### 3. Wire fragment resolution into the pipeline

**File**: `scripts/compile_agents.ts`

- [x] Import `loadFragments`, `validateFragments`, and `resolveFragments`
      from `./lib/fragments.js`
- [x] After `loadCanonicalSpecs()` (line 81) and before `validateSchema()`
      (line 83), add fragment resolution
- [x] Pass `resolvedSpecs` (instead of `rawSpecs`) to all subsequent stages

#### 4. Create the `spec/fragments/` directory

- [x] Create `spec/fragments/.gitkeep` (or the first real fragment in Phase 3)

#### 5. Add fragment resolution tests

**File**: `scripts/__tests__/fragments.test.ts`

- [x] Test `loadFragments()` loads `.md` files from `spec/fragments/`
- [x] Test `loadFragments()` computes partial names correctly (nested paths,
      extension stripped)
- [x] Test `loadFragments()` returns empty map when directory doesn't exist
- [x] Test `resolveFragments()` replaces `{{> partial-name}}` with fragment content
- [x] Test `resolveFragments()` passes variables to partials
      (`{{> partial-name var="value"}}`)
- [x] Test `resolveFragments()` leaves specs without template syntax unchanged
- [x] Test `resolveFragments()` throws on missing partial reference
- [x] Test `resolveFragments()` throws on Handlebars syntax errors
- [x] Test `resolveFragments()` does NOT HTML-escape angle brackets in variable
      values (verifies `noEscape` is set — e.g., `<absolute path>` must not
      become `&lt;absolute path&gt;`)
- [x] Test `resolveFragments()` uses an isolated Handlebars instance (calling
      it twice does not leak partials between invocations)
- [x] Test `validateFragments()` detects nested fragment includes via AST
      parsing (not string scanning)
- [x] Test end-to-end: spec with fragment produces expected resolved body

#### 6. Add fragment test fixtures

**Directory**: `scripts/__tests__/fixtures/spec/fragments/`

- [x] Create `test/greeting.md` containing `Hello, {{name}}!`
      (the `{{name}}` variable is provided via hash args when the partial is
      invoked: `{{> test/greeting name="World"}}`)
- [x] Create `test/static.md` containing plain text (no variables)
- [x] Create `test/nested-bad.md` containing a `PartialStatement` node
      (e.g., `Intro: {{> test/static}}`) for nested include detection test

### Success Criteria

#### Automated Verification

- [x] `npm run test` passes all tests (existing + new fragment tests)
- [x] `npm run build:agents` succeeds with no changes to generated output
      (fragments exist but no skills use them yet)
- [x] Fragment resolution is a no-op for specs that don't use `{{` syntax

#### Manual Verification

- [ ] Review `fragments.ts` for correct error messages on missing/invalid
      fragments
- [ ] Verify Handlebars does not mangle spec bodies that contain markdown
      code fences, shell variables (`$VAR`), or angle-bracket placeholders
      (`<value>`)

**Implementation Note**: After completing this phase and all automated
verification passes, pause here for manual confirmation that Handlebars
rendering preserves existing spec bodies exactly before proceeding.

---

## Phase 3: Extract the Prompt Contract Fragment

### Overview

Create the shared prompt contract fragment, replace the duplicated blocks in
all four review skills with Handlebars partial references, and verify generated
output matches.

### Changes Required

#### 1. Capture current generated output for diff comparison

- [ ] Ensure `generated/` is clean before starting (no uncommitted changes).
      After compilation, use `git diff generated/` to compare against the
      committed baseline. This is the simplest and most reliable approach.

#### 2. Create the prompt contract fragment

**File**: `spec/fragments/review/prompt-contract.md`

- [x] Create the fragment with Handlebars variables for context-specific values.
      This becomes the **canonical version** of the prompt contract. The field
      names and ordering are the single source of truth; each skill provides
      its own context-specific placeholder descriptions via hash arguments:

  ```markdown
  Review request:
  - Project path: {{project_path}}
  - Scope: {{scope}}
  - Base reference: {{base_reference}}
  - Merge-base: {{merge_base}}
  - Focus: {{focus}}
  ```

**Important note on code fences**: In each skill below, the `{{> ...}}` partial
call appears inside a markdown code fence (triple backticks). Handlebars does
not understand markdown — it resolves the partial reference regardless of
surrounding syntax. The code fence is part of the *spec body* and will appear
in generated output; only the `{{> ...}}` directive is replaced.

#### 3. Update `code-review/SKILL.md`

**File**: `spec/skills/code-review/SKILL.md`

- [x] Replace lines 82-89 (the prompt contract code block) with:
  > **Deviation:** Used single-quoted hash args (`'...'`) for `base_reference`
  > and `merge_base` since their values contain double quotes (`"none"`).
  > The plan used backslash-escaped quotes (`\"`) which Handlebars doesn't support.

  ````
  ```
  {{> review/prompt-contract
      project_path="<absolute-or-workspace-relative path>"
      scope="<uncommitted changes | changes against <base-ref> | specific files>"
      base_reference="<git-ref or \"none\">"
      merge_base="<commit-sha or \"none\">"
      focus="<optional subdirectory or comma-separated files>"
  }}
  ```
  ````

  Note: The code fence (` ``` `) is part of the skill body, not Handlebars
  syntax. Handlebars resolves the `{{> ...}}` inside the fence, producing
  the same text that was there before.

#### 4. Update `code-review-loop/SKILL.md`

**File**: `spec/skills/code-review-loop/SKILL.md`

- [x] Replace lines 121-128 (the prompt contract code block) with:

  ````
  ```
  {{> review/prompt-contract
      project_path="<absolute-or-workspace-relative path>"
      scope="changes against <base-ref>"
      base_reference="<git-ref>"
      merge_base="<commit-sha>"
      focus="none"
  }}
  ```
  ````

#### 5. Update `review-files/SKILL.md`

**File**: `spec/skills/review-files/SKILL.md`

- [x] Replace lines 78-85 (the prompt contract code block) with:

  ````
  ```
  {{> review/prompt-contract
      project_path="<current working directory>"
      scope="specific files"
      base_reference="none"
      merge_base="none"
      focus="<comma-separated list of resolved file paths>"
  }}
  ```
  ````

#### 6. Update `review-pr/SKILL.md`

**File**: `spec/skills/review-pr/SKILL.md`

- [x] Replace lines 101-108 (the prompt contract code block) with:

  ````
  ```
  {{> review/prompt-contract
      project_path="<absolute path>"
      scope="changes against <PR_BASE_BRANCH>"
      base_reference="<PR_BASE_BRANCH>"
      merge_base="<MERGE_BASE sha>"
      focus="none"
  }}
  ```
  ````

#### 7. Compile and verify

- [x] Run `npm run build:agents`
- [x] Diff generated output against the pre-change state:
  ```bash
  git diff generated/
  ```
  > Zero diff — generated output is byte-for-byte identical.
- [x] If the diff shows only whitespace or the intentional drift corrections
      noted in the plan, accept the changes
- [x] No unexpected differences found

#### 8. Add integration test for prompt contract fragment

**File**: `scripts/__tests__/fragments.test.ts` (append to existing)

- [x] Test that a spec body containing `{{> review/prompt-contract ...}}` with
      all five variables produces the expected `Review request:` block
- [x] Test that each variable is correctly substituted (no leftover `{{}}`)

### Success Criteria

#### Automated Verification

- [x] `npm run build:agents` succeeds with no errors
- [x] `npm run test` passes all tests (53/53)
- [x] Generated output for all four review skills matches the pre-change
      snapshot — zero diff
- [x] `npm run check:agents` passes (generated files are up-to-date)

#### Manual Verification

- [ ] Review each of the four updated skill files to confirm the partial
      invocation reads naturally and is maintainable
- [ ] Verify the fragment file at `spec/fragments/review/prompt-contract.md`
      is clean and self-documenting
- [ ] Spot-check one generated skill output to confirm the resolved prompt
      contract appears correctly in the final markdown

**Implementation Note**: After completing this phase and all automated
verification passes, pause here for manual confirmation before proceeding
to Phase 4.

---

## Phase 4: Documentation and Cleanup

### Overview

Update project documentation to cover the fragment mechanism, add an agent-docs
learning entry, and ensure the schema/README reflect the new `spec/fragments/`
directory.

### Changes Required

#### 1. Update README.md

**File**: `README.md`

- [x] Add `spec/fragments/` to the Directory Layout section
- [x] Add a "Fragments" section explaining usage, syntax, and constraints

#### 2. Update CLAUDE.md

**File**: `CLAUDE.md`

- [x] Add guidance for working with fragments (location, naming, regeneration)

#### 3. Add agent-docs learning entry

**File**: `agent-docs/compiler-fragments.md`

- [x] Document the fragment mechanism with key details and gotchas
  (noEscape, isolated instances, partial indentation, hash arg quoting,
  trailing newlines, no nested fragments, AST-based validation)

#### 4. No temp cleanup needed

Snapshot comparison uses `git diff`, so there are no temp files to clean up.

### Success Criteria

#### Automated Verification

- [x] `npm run build:agents` still succeeds
- [x] `npm run test` still passes (53/53)

#### Manual Verification

- [ ] README fragment section is clear and complete
- [ ] CLAUDE.md guidance is concise and actionable
- [ ] Agent-docs entry captures the key gotchas for future sessions

---

## Testing Strategy

### Unit Tests (Phase 1 + Phase 2)

- All existing compiler modules: parse, validate, compile, format, emit, model
- Fragment-specific: loading, resolution, variable substitution, error handling
- Edge cases: missing fragments, empty fragment directory, specs with no
  template syntax, Handlebars syntax errors

### Integration Tests (Phase 3)

- End-to-end: spec with prompt contract partial produces expected output
- Generated output equivalence: before/after diff comparison

### Manual Testing Steps

1. Run `npm run build:agents` and verify clean output
2. Spot-check `generated/claude/skills/code-review/SKILL.md` for correct
   prompt contract rendering
3. Verify `npm run check:agents` passes (generated files match compiled state)
4. Confirm Handlebars doesn't mangle existing specs by reviewing a non-fragment
   skill's generated output

## Performance Considerations

Handlebars compilation is fast (sub-millisecond per template). With ~30 specs
and a handful of fragments, the added compile step is negligible. No caching
is needed initially.

## References

- Idea document: `thoughts/ideas/2026-03-07-shared-compiler-fragments.md`
- Compiler entry point: `scripts/compile_agents.ts`
- Pipeline insertion point: `compile_agents.ts:81-83` (between load and validate;
  line numbers refer to pre-change state — imports added in Phase 2 will shift these)
- Prompt contract locations (line numbers refer to pre-change state):
  - `spec/skills/code-review/SKILL.md:82-89`
  - `spec/skills/code-review-loop/SKILL.md:121-128`
  - `spec/skills/review-files/SKILL.md:78-85`
  - `spec/skills/review-pr/SKILL.md:101-108`
- Handlebars docs: https://handlebarsjs.com/guide/
