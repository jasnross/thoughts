# agentspec Rules Support — Implementation Plan

## Overview

Add first-class `rules` spec support to agentspec so teams can write a rule once
in `spec/rules/<id>.md` and compile it to all four providers (Claude, Cursor,
Codex, OpenCode) rather than maintaining provider-specific rule files manually.

## Current State Analysis

agentspec has two spec kinds — `Agent` and `Skill` — modelled by a two-variant
`SpecKind` enum. Spec type is determined by load directory, not frontmatter. The
pipeline is exhaustive: `provider_supports_kind` (compile.rs:62-71) is a
compiler-enforced `match (Provider, SpecKind)` over every combination.

### Key Discoveries

- `SpecKind` enum at `src/types.rs:39-43` — adding `Rule` turns every
  incomplete match into a compiler error, guiding all required edits.
- `NormalizedSpec` at `src/types.rs:74-107` has no `paths` field — rules
  introduce this concept for the first time.
- `SpecConfig` at `src/config.rs:37-43` lists `agents_dir`, `skills_dir`,
  `fragments_dir` — a `rules_dir` field must be added.
- `load_canonical_specs` at `src/parse.rs:211-223` — rules use the flat-file
  loader pattern (like agents), not the subdirectory pattern (like skills).
- Both copies of `canonical.schema.json` must be kept in sync:
  `schemas/canonical.schema.json` and
  `agent-config/spec/schema/canonical.schema.json`.
- Claude and OpenCode adapters use `if spec.kind == SpecKind::Skill / Agent`
  branches with silent fallthrough — a new `Rule` kind must be explicitly
  handled to avoid silently routing rules through agent code paths.

## Desired End State

`agentspec compile` produces correct rule output for all four providers from a
single canonical `spec/rules/<id>.md` file. `agentspec check` detects drift.
`agentspec validate` catches malformed `paths:` and unknown targets.

### Verification

- `cargo build` succeeds with no warnings after adding `SpecKind::Rule`
- `cargo test` passes all existing and new integration tests
- `cargo clippy --all-targets` emits no lints
- A fixture with one unconditional and one path-scoped rule compiles to all
  four providers producing the correct output files and frontmatter

## What We're NOT Doing

- Codex directory-scoped `AGENTS.md` output — rules emit to
  `generated/codex/rules/<id>.md` only; users control placement
- OpenCode `opencode.json` read-modify-write — emit a standalone
  `generated/opencode/instructions.json` fragment; a future `sync` command
  will handle merging into the user's config
- Merging multiple rules into one AGENTS.md — one file per rule everywhere
- Codex Starlark execution rules (command allow/prompt/forbid policies)
- OpenCode path-scoping (`paths:` metadata is dropped for OpenCode)
- Rule priority / ordering
- `MissingDirectoryMapping` warning for file-extension-only Codex globs
  (output is flat and path-agnostic, so this warning is inapplicable)

## Implementation Approach

Add `SpecKind::Rule` to the type system first. The resulting compiler errors
then enumerate every location requiring a rules code path. Walk them top-down
through the pipeline: config → parse → validate → compile → adapters (per
provider) → integration tests.

---

## Phase 1: Schema and Types

### Overview

Establish the foundational data model. Every subsequent phase depends on
these types compiling cleanly.

### Changes Required

#### 1. `src/types.rs` — Add `SpecKind::Rule` and `paths` field

**File**: `src/types.rs`

- [x] Add `Rule` variant to `SpecKind` enum (currently lines 39-43):
  ```rust
  pub enum SpecKind {
      Agent,
      Skill,
      Rule,
  }
  ```
- [x] Add `paths: Option<Vec<String>>` field to `NormalizedSpec` (after the
  `kind` field, around line 80). Rules populate this; agents and skills leave
  it `None`:
  ```rust
  pub paths: Option<Vec<String>>,
  ```

#### 2. `src/config.rs` — Add `rules_dir` to `SpecConfig`

**File**: `src/config.rs`

- [x] Add `rules_dir: PathBuf` field to `SpecConfig` struct (lines 37-43):
  ```rust
  pub rules_dir: PathBuf,
  ```
- [x] Add default value in `SpecConfig::default()` (lines 64-72):
  ```rust
  rules_dir: PathBuf::from("spec/rules"),
  ```

#### 3. `schemas/canonical.schema.json` — Add `rule` kind and `paths` property

Both copies must be updated identically.

**Files**:
- `schemas/canonical.schema.json`
- `agent-config/spec/schema/canonical.schema.json`

- [x] Add `"rule"` to the `kind` enum (currently `["agent", "skill"]`):
  ```json
  "kind": {
    "type": "string",
    "enum": ["agent", "skill", "rule"]
  }
  ```
- [x] Add `paths` property to the root object schema:
  ```json
  "paths": {
    "type": "array",
    "items": { "type": "string" },
    "description": "Glob patterns that scope when this rule is applied. Absent means unconditional."
  }
  ```

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds (compiler errors from incomplete `SpecKind` matches
  will appear here — that's expected; they will be fixed in Phase 3)
- [x] Both schema files are byte-for-byte identical for the `kind` and `paths`
  additions: `diff schemas/canonical.schema.json agent-config/spec/schema/canonical.schema.json`

---

## Phase 2: Parse

### Overview

Load rule spec files from `spec/rules/` with `SpecKind::Rule` assigned.

### Changes Required

#### 1. `src/parse.rs` — Add `load_rule_specs` and wire into `load_canonical_specs`

**File**: `src/parse.rs`

- [x] Add `load_rule_specs(rules_dir: &Path) -> Result<Vec<CanonicalSpec>>`
  following the `load_agent_specs` pattern (lines 80-113): recursive walk,
  collect `*.md` files, parse frontmatter + body, assign `kind: SpecKind::Rule`,
  empty `supporting_files`. Return empty vec if directory does not exist.
- [x] Update `load_canonical_specs` (lines 211-223) to resolve `rules_dir`
  from `config` and call `load_rule_specs`:
  ```rust
  let rules_dir = config.resolve(&config.spec.rules_dir);
  let rules = load_rule_specs(&rules_dir)?;
  agents.extend(rules);
  ```

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds after adding the new loader
- [x] `cargo test` passes existing tests (rule directory is absent in current
  fixture, so the early-return-on-missing-dir path is exercised)

---

## Phase 3: Validate and Dispatch

### Overview

Normalize `paths` from frontmatter, exclude rules from skill-only semantic
checks, and wire `SpecKind::Rule` into the compiler dispatch gate.

### Changes Required

#### 1. `src/validate.rs` — Normalize `paths` and update semantic validation

**File**: `src/validate.rs`

- [x] In `normalize_specs` (around line 100-262), extract `paths` from
  frontmatter JSON and assign to `NormalizedSpec.paths`:
  ```rust
  let paths: Option<Vec<String>> = fm
      .get("paths")
      .and_then(|v| serde_json::from_value(v.clone()).ok());
  ```
- [x] In `validate_semantics` (around lines 293-300), scope the skill
  invocability check to `SpecKind::Skill` only (it likely already is — verify
  the condition does not accidentally fire for `Rule`).

#### 2. `src/compile.rs` — Add `Rule` arms to `provider_supports_kind`

**File**: `src/compile.rs`

- [x] Add `Rule` arms to `provider_supports_kind` (lines 62-71). All four
  providers support rules:
  ```rust
  (Provider::Claude,    SpecKind::Rule) => true,
  (Provider::OpenCode,  SpecKind::Rule) => true,
  (Provider::Codex,     SpecKind::Rule) => true,
  (Provider::Cursor,    SpecKind::Rule) => true,
  ```

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds with zero compiler errors (all exhaustive matches
  are now complete)
- [x] `cargo clippy --all-targets` passes
- [x] `cargo test` passes

---

## Phase 4: Claude Adapter

### Overview

Emit `generated/claude/rules/<id>.md` with optional `paths:` frontmatter.
When `paths:` is absent the file has no frontmatter (unconditional rule).

### Changes Required

#### 1. `src/adapters/claude.rs` — Add rule output path

**File**: `src/adapters/claude.rs`

- [x] Add a rule branch before the existing `if spec.kind == SpecKind::Skill`
  check (line 46). Return early after emitting:
  ```rust
  if spec.kind == SpecKind::Rule {
      let mut fm = IndexMap::new();
      if let Some(desc) = &spec.description {
          fm.insert("description".to_string(), Value::String(desc.clone()));
      }
      if let Some(paths) = &spec.paths {
          fm.insert(
              "paths".to_string(),
              Value::Array(paths.iter().map(|p| Value::String(p.clone())).collect()),
          );
      }
      let content = if fm.is_empty() {
          format!("{}\n", spec.body.trim())
      } else {
          render_markdown_with_frontmatter(&fm, &spec.body)
      };
      let path = PathBuf::from(format!("generated/claude/rules/{}.md", spec.id));
      return (vec![GeneratedFile { provider: Provider::Claude, relative_path: path, contents: content.into_bytes() }], vec![]);
  }
  ```

### Success Criteria

#### Automated Verification

- [x] `cargo build` and `cargo clippy --all-targets` pass
- [x] Unit test (inline, in claude.rs) verifies:
  - [x] Rule with `paths:` produces YAML frontmatter with `paths:` list
  - [x] Rule without `paths:` produces a file with no frontmatter block

---

## Phase 5: Cursor Adapter

### Overview

Emit `generated/cursor/rules/<id>.mdc`. When `paths:` is present, use `globs:`.
When `paths:` is absent, use `alwaysApply: true`.

### Changes Required

#### 1. `src/adapters/cursor.rs` — Add rule output path

**File**: `src/adapters/cursor.rs`

- [x] Add a rule branch before existing skill logic. Return early after emitting:
  ```rust
  if spec.kind == SpecKind::Rule {
      let mut fm = IndexMap::new();
      if let Some(desc) = &spec.description {
          fm.insert("description".to_string(), Value::String(desc.clone()));
      }
      match &spec.paths {
          Some(paths) => {
              fm.insert(
                  "globs".to_string(),
                  Value::Array(paths.iter().map(|p| Value::String(p.clone())).collect()),
              );
          }
          None => {
              fm.insert("alwaysApply".to_string(), Value::Bool(true));
          }
      }
      let content = render_markdown_with_frontmatter(&fm, &spec.body);
      let path = PathBuf::from(format!("generated/cursor/rules/{}.mdc", spec.id));
      return (vec![GeneratedFile { provider: Provider::Cursor, relative_path: path, contents: content.into_bytes() }], vec![]);
  }
  ```

### Success Criteria

#### Automated Verification

- [x] `cargo build` and `cargo clippy --all-targets` pass
- [x] Unit test verifies:
  - [x] Rule with `paths:` emits `globs:` frontmatter key, no `alwaysApply`
  - [x] Rule without `paths:` emits `alwaysApply: true`, no `globs`
  - [x] Output file extension is `.mdc`

---

## Phase 6: Codex Adapter

### Overview

Emit `generated/codex/rules/<id>.md` as plain body — no frontmatter, no
path metadata. Codex rules are flat files; placement is user-controlled.

### Changes Required

#### 1. `src/adapters/codex.rs` — Add rule branch

**File**: `src/adapters/codex.rs`

- [x] Add a rule branch at the top of `adapt_codex`. Rules have no frontmatter
  and no model config; emit the body directly:
  ```rust
  if spec.kind == SpecKind::Rule {
      let content = format!("{}\n", spec.body.trim());
      let path = PathBuf::from(format!("generated/codex/rules/{}.md", spec.id));
      return (vec![GeneratedFile { provider: Provider::Codex, relative_path: path, contents: content.into_bytes() }], vec![]);
  }
  ```
  The remaining codex logic (which handles skills) is unchanged.

### Success Criteria

#### Automated Verification

- [x] `cargo build` and `cargo clippy --all-targets` pass
- [x] Unit test verifies:
  - [x] Output is raw body with no `---` frontmatter block
  - [x] Output path is `generated/codex/rules/<id>.md`

---

## Phase 7: OpenCode Adapter and `instructions.json`

### Overview

Emit `generated/opencode/rules/<id>/AGENTS.md` (plain body) per rule. After
all specs are compiled, aggregate rule paths into a standalone
`generated/opencode/instructions.json` fragment. `paths:` metadata is
intentionally dropped for OpenCode (it has no per-file activation trigger).

### Changes Required

#### 1. `src/adapters/opencode.rs` — Add rule branch

**File**: `src/adapters/opencode.rs`

- [x] Add a rule branch before the existing agent/skill logic. Emit plain body
  to a subdirectory following the existing skill directory convention:
  ```rust
  if spec.kind == SpecKind::Rule {
      let content = format!("{}\n", spec.body.trim());
      let path = PathBuf::from(format!(
          "generated/opencode/rules/{}/AGENTS.md",
          spec.id
      ));
      return (vec![GeneratedFile { provider: Provider::OpenCode, relative_path: path, contents: content.into_bytes() }], vec![]);
  }
  ```

#### 2. `src/adapters/opencode.rs` — Add `build_opencode_instructions`

**File**: `src/adapters/opencode.rs`

- [x] Add a public function that accepts all generated files and produces
  the `instructions.json` file if any OpenCode rule files are present:
  ```rust
  pub fn build_opencode_instructions(files: &[GeneratedFile]) -> Option<GeneratedFile> {
      let rule_paths: Vec<serde_json::Value> = files
          .iter()
          .filter(|f| {
              f.provider == Provider::OpenCode
                  && f.relative_path
                      .to_string_lossy()
                      .starts_with("generated/opencode/rules/")
          })
          .map(|f| serde_json::Value::String(f.relative_path.to_string_lossy().into_owned()))
          .collect();
      if rule_paths.is_empty() {
          return None;
      }
      let json = serde_json::json!({ "instructions": rule_paths });
      let content = serde_json::to_string_pretty(&json)
          .expect("instructions.json serialization is infallible")
          .into_bytes();
      Some(GeneratedFile {
          provider: Provider::OpenCode,
          relative_path: PathBuf::from("generated/opencode/instructions.json"),
          contents: content,
      })
  }
  ```

#### 3. `src/compile.rs` — Call `build_opencode_instructions` after the compile loop

**File**: `src/compile.rs`

- [x] After the main spec compilation loop in `compile_specs`, call
  `build_opencode_instructions` and append the result if present:
  ```rust
  if let Some(instructions_file) = build_opencode_instructions(&all_files) {
      all_files.push(instructions_file);
  }
  ```

### Success Criteria

#### Automated Verification

- [x] `cargo build` and `cargo clippy --all-targets` pass
- [x] Unit test verifies:
  - [x] A rule spec produces `generated/opencode/rules/<id>/AGENTS.md`
  - [x] `build_opencode_instructions` with one rule file produces
    `generated/opencode/instructions.json` containing the rule path
  - [x] `build_opencode_instructions` with no rule files returns `None`
  - [x] `paths:` field on the rule spec does not appear in any OpenCode output

---

## Phase 8: Integration Tests

### Overview

Add rule fixtures to the self-contained test fixture and assert correct
output for all four providers. One unconditional rule and one path-scoped rule.

### Changes Required

#### 1. `tests/fixtures/agent-config/spec/rules/` — Add fixture rule specs

**Files** (new):
- `tests/fixtures/agent-config/spec/rules/general-guidance.md`
- `tests/fixtures/agent-config/spec/rules/api-design.md`

- [x] Create `general-guidance.md` (unconditional — no `paths:`):
  ```markdown
  ---
  id: general-guidance
  description: General coding standards for this project
  compat:
    targets:
      - claude
      - cursor
      - codex
      - opencode
  ---

  # General Guidance

  All code must be reviewed before merging.
  ```
- [x] Create `api-design.md` (path-scoped):
  ```markdown
  ---
  id: api-design
  description: API design conventions for REST endpoints
  paths:
    - "src/api/**"
  compat:
    targets:
      - claude
      - cursor
      - codex
      - opencode
  ---

  # API Design Rules

  All endpoints must validate input at the boundary.
  ```

#### 2. `tests/pipeline.rs` — Add assertions for rule output

**File**: `tests/pipeline.rs`

- [x] Update `test_validate_fixture`: change hardcoded spec count from `3` to
  `5` (3 existing + 2 rules).
- [x] In `test_compile_generates_expected_files`, add assertions for:
  - [x] `generated/claude/rules/general-guidance.md` exists and contains no
    `paths:` frontmatter key
  - [x] `generated/claude/rules/api-design.md` exists and contains
    `paths:` with `src/api/**`
  - [x] `generated/cursor/rules/general-guidance.mdc` exists with
    `alwaysApply: true`
  - [x] `generated/cursor/rules/api-design.mdc` exists with
    `globs: ["src/api/**"]`
  - [x] `generated/codex/rules/general-guidance.md` exists with no
    frontmatter block
  - [x] `generated/codex/rules/api-design.md` exists with no frontmatter block
  - [x] `generated/opencode/rules/general-guidance/AGENTS.md` exists
  - [x] `generated/opencode/rules/api-design/AGENTS.md` exists
  - [x] `generated/opencode/instructions.json` exists and contains both rule
    paths in its `instructions` array
- [x] In `test_check_passes_after_compile`: no change needed (check uses
  compile output, both should stay in sync automatically).

### Success Criteria

#### Automated Verification

- [x] `cargo test` passes all tests including new integration tests
- [x] `cargo clippy --all-targets` passes with no warnings
- [x] `cargo fmt --check` passes

#### Manual Verification

- [x] Run `agentspec compile` against the `agent-config/` dotfiles directory
  and inspect the generated files to confirm real-world output looks correct

---

## Testing Strategy

### Unit Tests

Each adapter phase includes inline unit tests. Key edge cases:

- Claude: rule with `paths:` → frontmatter present; rule without `paths:` →
  no frontmatter block at all (not even empty `---\n---`)
- Cursor: `alwaysApply: true` vs `globs:` mutual exclusion
- OpenCode: `paths:` silently dropped from output; `instructions.json` only
  generated when at least one rule is present

### Integration Tests

The `tests/pipeline.rs` integration tests exercise the full binary against
the fixture directory and assert on exact file existence and content.

## Schema Embedding Note

After changing `canonical.schema.json`, the installed `agentspec` binary
will still enforce the old schema until `cargo install --path .` is re-run.
`cargo test` always tests the freshly built binary via
`env!("CARGO_BIN_EXE_agentspec")`, so tests are unaffected.

## References

- Idea file: `$THOUGHTS_DIR/ideas/2026-03-22-agentspec-rules-support.md`
- `SpecKind` enum: `src/types.rs:39-43`
- `provider_supports_kind`: `src/compile.rs:62-71`
- `SpecConfig`: `src/config.rs:37-43`
- `load_canonical_specs`: `src/parse.rs:211-223`
- Claude adapter skill branch: `src/adapters/claude.rs:46`
- OpenCode adapter skill branch: `src/adapters/opencode.rs:102`
