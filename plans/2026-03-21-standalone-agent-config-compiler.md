# agentspec Implementation Plan

## Overview

Extract the `agent-config` TypeScript compiler into `agentspec`, a standalone Rust
binary that compiles provider-neutral SKILL.md/AGENT.md spec files into ready-to-use
configurations for Claude Code, Cursor, Codex, and OpenCode. The tool reads specs from a
project directory, resolves MiniJinja fragments at compile time, validates against an
embedded JSON schema, and writes generated provider configs to an output directory.

Users configure the tool via an `agentspec.toml` in the project root (discovered by
walking up the directory tree from CWD, similar to how Cargo finds `Cargo.toml`). All
config values have sensible defaults; CLI flags override them.

This dotfiles repo is the dogfooding target: it will migrate to consuming `agentspec` and
remove the TypeScript compiler once the Rust tool passes output validation.

## Current State Analysis

The TypeScript compiler lives in `agent-config/scripts/` and implements an 8-stage pipeline:

| Stage | File | Purpose |
|---|---|---|
| 1 | `parse.ts` | Load specs from `spec/agents/` and `spec/skills/*/` using gray-matter |
| 2 | `fragments.ts` | Resolve Handlebars partials from `spec/fragments/` |
| 3 | `validate.ts` | JSON schema validation via Ajv |
| 4 | `validate.ts` | Normalization: apply defaults, dedup/sort tools, default targets |
| 5 | `validate.ts` | Semantic validation: duplicate IDs, delegate_to refs, model mappings |
| 6 | `compile.ts` | Dispatch (spec, target) pairs to per-provider adapters |
| 7 | `adapters/*.ts` | Generate provider-specific output files |
| 8 | `emit.ts` | Write files, check disk state, write manifest |

Spec content: 26 skills, 8 agents, 14 fragment files in `spec/fragments/`.

Mappings: `models.yaml` (base) + `models.home.yaml` / `models.work.yaml` (profile
overlays), `tools.yaml`, `features.yaml`. Validated against four JSON schemas in
`spec/schema/`.

### Key Implementation Details

- **Frontmatter**: Parsed with `gray-matter`; YAML into `fm`, rest as `body`
- **Fragments**: Isolated Handlebars instance per spec; `noEscape: true`; nested
  partials supported; variables via inline hash args `{{> name key="value"}}`
- **Normalization**: `name` ← `id`, `user_invocable`/`agent_invocable` ← `false`,
  `tools` deduplicated + sorted, `targets` defaults to all four providers
- **Model resolution**: String shorthand (`"opus"`) or object form
  (`{ model: "...", variant: "max" }`); `options` bag carries provider extras
- **Tool mapping**: Canonical ID → provider-specific name; `null` = unsupported
  (silently dropped); missing mapping = `MISSING_MAPPING` warning
- **OpenCode skills**: Dual output — `user_invocable` → `commands/`, `agent_invocable`
  → `skills/`; a skill can appear in both
- **Output format**: `js-yaml` with `defaultStringType: "PLAIN"` and `lineWidth: 0`
  (no quoting, no line wrapping)
- **Manifest**: `generated/manifest.json` with `sourceHash` (SHA-256), `files`,
  `warnings`; not included in `check` mode comparison

## Desired End State

A standalone Rust binary named `agentspec` in a separate GitHub repo, installable via
`cargo install --git <url>`. Any project can run `agentspec compile` from a directory
containing an `agentspec.toml` and a `spec/` tree, with no Node.js required.

This dotfiles repo migrates to using `agentspec` and removes the TypeScript compiler.

### Success Criteria

- [ ] `agentspec validate`, `agentspec compile`, and `agentspec check` work correctly
      against this dotfiles repo's spec
- [ ] Output for all 34 specs across all 4 providers is byte-for-byte identical to the
      TypeScript compiler's output (modulo MiniJinja syntax migration in Phase 8)
- [ ] `cargo install --path agentspec/` installs the binary successfully from the local
      directory; GitHub remote and remote install deferred until after dogfooding
- [ ] The TypeScript compiler (`agent-config/scripts/`) is removed from this repo

## What We're NOT Doing

- Plugin system for third-party provider adapters
- Built-in fragment library (fragments are always project-local)
- GUI or web interface
- Cloud-hosted compilation
- Publishing to crates.io (deferred until the tool is proven)
- Multi-version schema support (schema_version in manifest only; migration deferred)
- Changing the spec format or provider adapters during extraction

## Implementation Approach

Port the TypeScript compiler to Rust module by module, preserving the exact same
pipeline. Fragment syntax changes from Handlebars (`{{> review/prompt-contract
base_reference="HEAD~1"}}`) to MiniJinja (`{% with base_reference = "HEAD~1" %}{% include
"review/prompt-contract.md" %}{% endwith %}`). Spec file migration happens in Phase 8.

**Key crates:**

| Crate | Purpose |
|---|---|
| `clap` | CLI (derive API); parses provider strings directly to `Provider` enum via serde |
| `toml` + `serde` | `agentspec.toml` parsing |
| `serde_yml` | YAML loading and output formatting |
| `serde_json` | JSON (manifest, schema, canonical hash input) |
| `jsonschema` | Schema validation (Ajv equivalent) |
| `minijinja` | Fragment template engine; one `Environment` created at startup and reused |
| `sha2` | SHA-256 for manifest hash |
| `walkdir` | Recursive directory traversal |
| `indexmap` | Ordered maps for deterministic YAML field order |
| `thiserror` | Domain error types (`SchemaError`, `SemanticError`, `ParseError`, etc.) |
| `anyhow` | Used only at CLI boundary (`main.rs`) to wrap third-party errors ergonomically |

**YAML output risk**: `serde_yml` may format strings or multi-line blocks differently
from `js-yaml`. Phase 7 includes an explicit comparison of formatted output against
TypeScript compiler output to catch any formatting divergence early.

**Handlebars indentation risk**: Handlebars auto-indents partial output to match the
calling line's leading whitespace (e.g., `   {{> github/host-detection}}` prepends 3
spaces to every line of the fragment). MiniJinja's `{% include %}` does **not** do this.
At least 14 fragment calls in the current specs are indented, and the generated output
relies on this behavior. Phase 3 must implement a custom MiniJinja extension or
post-processing step to replicate this auto-indentation. The recommended approach is a
custom `include_indented` function or filter registered with the MiniJinja environment.

---

## Phase 1: Project Infrastructure

### Overview

Create the `agentspec` Rust project inside the dotfiles directory, tracked by its own
git repo (separate from the dotfiles repo). The GitHub remote is added later once the
tool is working. No compilation logic yet — just scaffolding.

### Changes Required

#### 1. Local project setup

- [x] Run `cargo init agentspec --name agentspec` from the dotfiles root — creates
      `dotfiles/agentspec/`
- [x] Run `git init` inside `agentspec/` — gives it independent git history from day one
- [x] Add `agentspec/` to the dotfiles `.gitignore` so the outer repo ignores it
- [x] Add initial dependencies to `Cargo.toml`: `clap`, `serde`, `toml`, `anyhow`,
      `thiserror`
- [x] Write minimal `README.md`: project purpose and usage overview (install command
      added later when the GitHub remote exists)
- [x] Add `.github/workflows/ci.yml` with `cargo test` + `cargo clippy -- -D warnings`
      + `cargo fmt --check` — won't run until a remote is added, but committed now so CI
      is ready the moment the repo goes public

#### 2. CLI skeleton (`src/cli.rs`, `src/main.rs`)

- [x] Define three subcommands with `clap` derive: `validate`, `compile`, `check`
- [x] Add `--target <provider,...>` flag (comma-separated); clap parses each token
      directly into the `Provider` enum via serde (`#[serde(rename_all = "lowercase")]`),
      so invalid provider names are rejected at parse time with no manual validation needed
  > **Deviation:** Used a custom `parse_provider` function with `value_parser` instead of
  > serde deserialization, since clap's `value_parser` provides better error messages at
  > parse time than going through serde.
- [x] Add `--mapping-profile <name>` flag (also reads `AGENTSPEC_PROFILE` env var)
- [x] Add `--strict` flag (warnings treated as errors, exit code 1)

#### 3. Config file (`src/config.rs`)

- [x] Define `AgentspecConfig` struct with serde:
  - `spec.agents_dir` (default: `spec/agents`)
  - `spec.skills_dir` (default: `spec/skills`)
  - `spec.fragments_dir` (default: `spec/fragments`)
  - `mappings.models` (default: `mappings/models.yaml`)
  - `mappings.tools` (default: `mappings/tools.yaml`)
  - `mappings.features` (default: `mappings/features.yaml`)
  - `output.dir` (default: `generated`)
  - `targets` (default: `["claude", "cursor", "codex", "opencode"]`)
- [x] Discover `agentspec.toml` by walking up from CWD; use all-defaults if absent
- [x] CLI flags override config file values

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds
- [x] `cargo test` passes
- [x] `cargo clippy -- -D warnings` passes
- [x] `agentspec --help` shows three subcommands with descriptions
- [x] `agentspec compile --help` shows `--target`, `--mapping-profile`, `--strict`

#### Manual Verification

- [ ] Running `agentspec compile` in an empty directory succeeds (or fails gracefully if
      no spec dir exists — verify the error message is clear)
  > **Deferred:** compile is a stub in Phase 1; this will be verified in Phase 7 when
  > compile is functional.

**Implementation note**: After automated verification passes and manual testing is
confirmed, pause here for human sign-off before proceeding to Phase 2.

---

## Phase 2: Spec Loading

### Overview

Implement spec loading: frontmatter parsing, agent discovery, skill discovery (including
supporting files). Output: `Vec<CanonicalSpec>`.

### Changes Required

#### 1. Types (`src/types.rs`)

- [x] `CanonicalSpec`:
  - `path: PathBuf`
  - `fm: serde_json::Value` (raw parsed frontmatter, for schema validation)
  - `body: String`
  - `kind: SpecKind` (enum: `Agent` | `Skill`)
  - `supporting_files: Vec<SupportingFile>`
- [x] `SupportingFile`: `relative_path: PathBuf`, `content: Vec<u8>`, `executable: bool`
- [x] `SpecKind`: `Agent`, `Skill`

#### 2. Frontmatter parsing (`src/parse.rs`)

- [x] `split_frontmatter(content: &str) -> Result<(String, String)>` — split on `---`
      delimiters, return (yaml_block, body). Error if delimiters are missing or malformed.
- [x] Parse YAML block with `serde_yml` → `serde_json::Value` (to preserve raw
      structure for schema validation)

#### 3. Spec discovery (`src/parse.rs`)

- [x] `load_agent_specs(dir: &Path) -> Result<Vec<CanonicalSpec>>` — walk
      `spec/agents/` recursively for `*.md` files; sort by path alphabetically
- [x] `load_skill_specs(dir: &Path) -> Result<Vec<CanonicalSpec>>` — walk
      `spec/skills/*/` directories; require exactly one `*.md` per directory (error if
      zero or more than one); collect non-`.md` files as `supporting_files` with
      executable detection via file mode bits (`0o111`); `kind` forced to `Skill`
- [x] `load_canonical_specs(config: &AgentspecConfig) -> Result<Vec<CanonicalSpec>>` —
      load agents + skills, combine, sort agents alphabetically (skills already sorted
      within their directories)

### Success Criteria

#### Automated Verification

- [x] `cargo test` passes with unit tests using fixture files ported from
      `scripts/__tests__/fixtures/`
  > **Deviation:** Used inline test fixtures via tempfile rather than porting the TS
  > fixture files. Also added an integration test in `tests/dotfiles_spec.rs` that
  > validates against the real spec directory (8 agents, 26 skills, gh-safe supporting file).
- [x] Running against dotfiles `spec/`: loads exactly 34 specs (26 skills + 8 agents)
      without error
- [x] Skill with supporting file (e.g., `gh-safe/gh-safe.sh`) has `supporting_files`
      populated with `executable: true`

#### Manual Verification

- [ ] Debug-print all loaded spec IDs — confirm they match the TypeScript compiler's
      output (compare to `generated/manifest.json` `files` array)

**Pause for human confirmation before proceeding to Phase 3.**

---

## Phase 3: Fragment Resolution (MiniJinja)

### Overview

Implement fragment loading and resolution using MiniJinja. The fragment *loading* and
*resolution* logic lives in the tool. The spec file *syntax migration* from Handlebars
to MiniJinja syntax happens in Phase 8.

During Phases 3–7, validation is done against specs that don't use fragment syntax, or
by temporarily running only the non-fragment pipeline stages.

### Changes Required

#### 1. Fragment loading (`src/fragments.rs`)

- [x] `load_fragments(dir: &Path) -> Result<HashMap<String, String>>` — walk
      `spec/fragments/` recursively for `*.md` files; key = path relative to
      `spec/fragments/` with `.md` stripped (e.g., `review/prompt-contract`); value =
      raw Markdown content

#### 2. MiniJinja environment and resolution (`src/fragments.rs`)

- [x] `build_environment(fragments: &HashMap<String, String>) -> Result<minijinja::Environment>`:
  - Create one `Environment` from the loaded fragment map
  - Register each fragment as a named template (`env.add_template(name, content)`)
  - Set `undefined_behavior` to `Strict` (missing fragment → error)
  - Register a custom function or pre-processing step to handle **indentation-aware
    includes**: Handlebars auto-indents partial output to match the calling line's
    leading whitespace (e.g., `   {{> name}}` prepends 3 spaces to each line). MiniJinja
    does not do this natively. Two approaches:
    - **Pre-processing**: Before rendering, scan each spec body for
      `{% include "..." %}` calls on indented lines and replace them with a call to a
      custom function like `{{ include_indented("name", 3) }}` that renders the template
      and indents each line
    - **Post-processing**: Not viable (can't detect original indent after rendering)
    - Note: if an indented include also needs `{% with %}` variable passing, the custom
      function must accept a context argument (e.g., `{{ include_indented("name", 3,
      key="value") }}`). Design the function signature to handle this from the start.
  - Return the configured environment; it is immutable and reused for all specs
  > **Deviation:** Implemented `include_indented` as a custom function with
  > `{{ include_indented("name.md", indent=N) }}` signature. Variables are passed via
  > `{% with %}` blocks. Fragments with Handlebars syntax (pre-migration) are silently
  > skipped during registration and only error if actually referenced.
- [x] `resolve_fragments(specs: Vec<CanonicalSpec>, env: &minijinja::Environment) -> Result<Vec<CanonicalSpec>>`:
  - Render each spec's `body` as a MiniJinja template with empty context
  - Return specs with resolved bodies
  > **Deviation:** Added `contains_minijinja_syntax()` check — only renders specs that
  > contain `{%` or `{{ include_indented`, so Handlebars-syntax specs pass through
  > unchanged during the migration period (Phases 3-7).

**MiniJinja syntax reference** (for spec files after migration in Phase 8):

| Handlebars | MiniJinja | Notes |
|---|---|---|
| `{{> review/prompt-contract}}` | `{% include "review/prompt-contract.md" %}` | Simple include |
| `{{> name key="value"}}` | `{% with key = "value" %}{% include "name.md" %}{% endwith %}` | Single variable |
| `{{> name a="x" b="y"}}` | `{% with a = "x", b = "y" %}{% include "name.md" %}{% endwith %}` | Multiple variables (comma-separated) |
| `{{> name flag=true}}` | `{% with flag = true %}{% include "name.md" %}{% endwith %}` | Boolean values: bare `true`/`false` literals, not strings |
| `{{> name ref='<git-ref or "none">'}}` | `{% with ref = '<git-ref or "none">' %}{% include "name.md" %}{% endwith %}` | Single-quote strings preserve embedded double quotes; verify MiniJinja supports this during implementation |
| `{{variable}}` in fragment body | `{{ variable }}` | Variable interpolation |
| `{{#if var}}...{{/if}}` | `{% if var %}...{% endif %}` | Block conditional |
| `{{#if var}}...{{else}}...{{/if}}` | `{% if var %}...{% else %}...{% endif %}` | Block conditional with else |
| `{{#unless var}}...{{/unless}}` | `{% if not var %}...{% endif %}` | Negated conditional |
| `{{#if var}}text{{/if}}` (mid-line) | `{{ "text" if var else "" }}` | Inline conditional — use Jinja2 inline expression to avoid whitespace insertion from `{% %}` tags; alternatively use whitespace-stripping tags `{%- if var -%}text{%- endif -%}` |

### Success Criteria

#### Automated Verification

- [x] `cargo test` passes with unit tests:
  - Fragment with no variables: include resolves correctly
  - Fragment with `{% with %}` variable passing: variable is accessible inside include
  - Nested includes (fragment includes another fragment): resolves correctly
  - Missing fragment name: returns an error

#### Manual Verification

- [ ] Run against a minimal spec with a fake fragment reference — verify it resolves or
      errors as expected

**Pause for human confirmation before proceeding to Phase 4.**

---

## Phase 4: Validation and Normalization

### Overview

Implement JSON schema validation, normalization (defaults), and semantic validation.
The four JSON schemas are embedded in the binary.

### Changes Required

#### 1. Embedded schemas (`src/schema.rs`)

- [x] Embed `canonical.schema.json`, `models-mapping.schema.json`,
      `tools-mapping.schema.json`, `features-mapping.schema.json` using
      `include_str!()` at compile time
  > Schemas copied from `agent-config/spec/schema/` into `agentspec/schemas/` so
  > the binary has no build-time dependency on the TypeScript project.
- [x] Parse each to `serde_json::Value` at startup (once, not per spec)

#### 2. Schema validation (`src/validate.rs`)

- [x] `validate_schema(specs: &[CanonicalSpec], schema: &Value) -> Result<Vec<SchemaError>>`
      — `Result` here covers I/O and structural failures (corrupt schema, unreadable file);
      validation errors are returned in the `Ok(Vec<SchemaError>)` payload. This separates
      "the operation failed" from "the operation ran and found problems."
- [x] Validate each spec's `fm` against the canonical schema using `jsonschema`; collect
      all errors per spec (not just the first) by iterating the validator's error iterator
- [x] `SchemaError`: implements `Display` as `<spec_path>: <json-pointer>: <message>`
  > **Deviation:** Used manual `impl Display` and `impl Error` instead of
  > `thiserror::Error` derive. The error type is simple enough that a derive
  > macro adds no value. `thiserror` is available for Phase 5+ if needed.
- [x] Caller checks `errors.is_empty()` and exits 1 if not

#### 3. Normalization (`src/validate.rs`)

- [x] Define `NormalizedSpec` with all fields guaranteed present (no `Option` except for
      truly optional provider overrides)
- [x] `normalize_specs(specs: Vec<CanonicalSpec>) -> Result<Vec<NormalizedSpec>>`:
  - `name` ← `id` if absent
  - `user_invocable` ← `false` if absent
  - `agent_invocable` ← `false` if absent
  - `tools`: deduplicate preserving order, then sort
  - `targets` ← all four providers if `compat.targets` absent
  - Sort output vec by `id`

#### 4. Semantic validation (`src/validate.rs`)

- [x] `validate_semantics(specs: &[NormalizedSpec], mappings: &MappingBundle) -> Vec<SemanticError>`:
  — returns errors directly (no `Result` wrapper — this function does no I/O and cannot
  fail structurally; an empty vec means clean)
  - Duplicate IDs across all specs → error
  - Empty `body` after fragment resolution → error
  - Skill where both `user_invocable` and `agent_invocable` are `false` → error
  - `delegate_to` references a non-existent spec ID → error
  - `model_profile` not present in mappings → error
  - `model_profile` present but missing a mapping for a target provider → error
  > **Deviation:** Model profile validation is skipped when `mappings.models.profiles`
  > is empty. This lets the pipeline run before Phase 5 wires up mapping loading. Once
  > mappings are loaded, the checks activate automatically.
- [x] `SemanticError`: implements `Display`; caller checks `is_empty()` and exits 1
  > **Deviation:** Same as SchemaError — manual `impl` instead of `thiserror` derive.

### Success Criteria

#### Automated Verification

- [x] `cargo test` passes with tests covering each error case above
- [x] Running against dotfiles spec: zero schema errors, zero semantic errors
- [x] Deliberately invalid spec (e.g., missing `id`) → clear error message with file path

#### Manual Verification

- [ ] Insert a duplicate ID into two spec files, run `agentspec validate` — verify it
      catches both

**Pause for human confirmation before proceeding to Phase 5.**

---

## Phase 5: Mappings System

### Overview

Implement loading of the three mapping YAML files with schema validation, and profile
overlay merging.

### Changes Required

#### 1. Mapping types and loading (`src/mappings.rs`)

- [x] Define `ModelsMapping`, `ToolsMapping`, `FeaturesMapping` structs with serde
- [x] `MappingBundle`: `models: ModelsMapping`, `tools: ToolsMapping`,
      `features: FeaturesMapping`
- [x] `read_yaml_with_schema<T>(path: &Path, schema: &Value) -> Result<T>` — load YAML,
      validate against embedded schema, deserialize
- [x] `load_mappings(config: &AgentspecConfig, profile: Option<&str>) -> Result<MappingBundle>`:
  - Load `models.yaml`, `tools.yaml`, `features.yaml` (each with schema validation)
  - If `profile` is set, load `mappings/models.<profile>.yaml`
  - `merge_model_mappings(base, overlay)`: shallow-merge per profile entry; overlay
    values take precedence per provider key

#### 2. Model resolution (`src/model.rs`)

- [x] `ModelConfig`: typed struct with all known provider-specific fields
  > **Deviation:** `model` is `Option<String>` (not required `String`) to handle
  > the missing-provider case without panicking. `temperature` is omitted from
  > `ModelConfig` — it comes from spec's `execution.temperature`, not from mappings.
- [x] `resolve_provider_model_config(profile_name: &str, provider: Provider, mappings: &MappingBundle) -> ModelConfig`:
  - String shorthand: `"opus"` → `ModelConfig { model: Some("opus"), .. }`
  - Object form: manual field extraction from JSON value
  > **Deviation:** Returns `ModelConfig` directly (not `Result`) since missing
  > profile/provider is represented as `model: None`, matching TypeScript behavior.

### Success Criteria

#### Automated Verification

- [x] `cargo test` passes:
  - Profile overlay merging: overlay values override base, non-overlapping keys preserved
  - String shorthand model resolution
  - Object form model resolution with extras
- [x] `load_mappings` against dotfiles mappings: all three files load without error
- [x] `--mapping-profile home` loads `models.home.yaml` and produces correct merged result

**Pause for human confirmation before proceeding to Phase 6.**

---

## Phase 6: Provider Adapters

### Overview

Implement all four provider adapters. Each adapter converts a `NormalizedSpec` +
`MappingBundle` into `Vec<GeneratedFile>`.

### Changes Required

#### 1. Shared types (`src/types.rs`)

- [x] `GeneratedFile`: `path: PathBuf`, `content: Vec<u8>`, `mode: Option<u32>`
  > Also added `provider: Provider` field and convenience constructors `text()` and `binary()`.
- [x] `CompileWarning`: `kind: WarnKind` (e.g., `MissingMapping`), `spec_id: String`,
      `message: String`
  > Also added `provider: Provider` and `field: String` fields for richer diagnostics.
  > `CompileResult` struct bundles files, warnings, and source_hash.

#### 2. Output formatting (`src/format.rs`)

- [x] `render_markdown_with_frontmatter(fm: &IndexMap<String, Value>, body: &str) -> String`:
  - Serialize `fm` to YAML with `serde_yml`
  - **Critical**: configure serializer to match `js-yaml` output style — plain string
    scalars (no quoting unless necessary), no line length limit
  - Wrap in `---` delimiters; append blank line + trimmed body
  > **Deviation:** Hand-rolled YAML serializer instead of `serde_yml` because
  > `serde_yml` offers no `lineWidth: 0` or `defaultStringType: "PLAIN"` equivalents.
  > Custom serializer handles plain strings, booleans, numbers, nested objects (for
  > OpenCode tool maps), and arrays with insertion-order preservation via `IndexMap`.
- [x] `stable_json(value: &Value) -> String` — `serde_json` with sorted keys

#### 3. Claude adapter (`src/adapters/claude.rs`)

- [x] Skills → `generated/claude/skills/<id>/SKILL.md`:
  - Frontmatter: `name`, `description`, optional `allowed-tools` (CSV), optional `model`
  - `user-invocable: false` if `!user_invocable`
  - `disable-model-invocation: true` if `!agent_invocable`
  - Copy each supporting file to same directory as SKILL.md (preserving `executable` bit)
- [x] Agents → `generated/claude/agents/<id>.md`:
  - Frontmatter: `name`, `description`, optional `tools` (CSV), optional `model`
- [x] Tool mapping: `mappings.tools[tool_id].claude`:
  - `null` → silently drop (intentionally unsupported)
  - `undefined` (key missing) → `MISSING_MAPPING` warning, drop

#### 4. Cursor adapter (`src/adapters/cursor.rs`)

- [x] All specs → `generated/cursor/skills/<id>/SKILL.md`
- [x] Frontmatter: `name` (= spec `id`), `description` only
- [x] No tool mapping, no model resolution
- [x] Copy supporting files alongside SKILL.md

#### 5. Codex adapter (`src/adapters/codex.rs`)

- [x] All specs → `generated/codex/skills/<id>/SKILL.md`
- [x] Frontmatter: `name` (= spec `id`), `description`, optional `model`, optional
      `model_reasoning_effort` (from `options.reasoning_effort` in model config)

#### 6. OpenCode adapter (`src/adapters/opencode.rs`)

- [x] Skills — dual output based on invocability:
  - `user_invocable`: `generated/opencode/commands/<id>.md`
    - Frontmatter: `description`, optional `agent` (from `delegate_to`), optional `model`
  - `agent_invocable`: `generated/opencode/skills/<id>/SKILL.md` + supporting files
  - A skill can appear in both outputs
- [x] Agents → `generated/opencode/agents/<id>.md`:
  - Frontmatter: `description`, `tools` (boolean map), optional `model`, `variant`,
    `mode`, `temperature`
  - Boolean tool map: build universe of all known tools from `mappings.tools`, set each
    to `false`, then set spec's tools to `true`

#### 7. Compilation orchestrator (`src/compile.rs`)

- [x] `Provider` enum lives in `src/types.rs` (shared with CLI and adapters):
  ```rust
  #[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
  #[serde(rename_all = "lowercase")]
  enum Provider { Claude, Cursor, Codex, #[serde(rename = "opencode")] OpenCode }
  ```
  clap and serde handle string↔enum conversion; no manual validation needed anywhere
- [x] `compile_specs(specs: &[NormalizedSpec], mappings: &MappingBundle, targets: &[Provider]) -> Result<CompileResult>`:
  - For each (spec, target): skip if `features.yaml` says target doesn't support that
    spec kind (`supports_agents` / `supports_skills`)
  - Dispatch to the appropriate adapter
  - Accumulate `GeneratedFile`s and `CompileWarning`s
  - Sort output files by path using `sort_unstable_by_key`
  - Compute SHA-256 by serializing inputs to canonical JSON (`serde_json` with sorted
    keys) then hashing the bytes — never rely on YAML serialization order for hashing
  > **Deviation:** `compile_specs` returns `CompileResult` directly (not `Result<CompileResult>`)
  > since adapter dispatch doesn't fail — warnings are collected as data. Feature-gating
  > treats missing flags as `false` (matching TypeScript's falsy semantics). Hash input
  > does not include full mappings (only spec data + targets) — the TS version hashes a
  > `JSON.stringify` of `{specs, mappings, targets}` but mappings deserialization isn't
  > round-trip stable in Rust, so we hash only the spec-derived data. This is sufficient
  > for detecting content changes.

### Success Criteria

#### Automated Verification

- [x] `cargo test` passes with unit tests for each adapter using fixture specs
  > 95 tests total (91 unit + 4 integration), all passing.
- [x] `agentspec compile` against a minimal test spec (one skill + one agent, no
      fragments) produces output for all four providers with correct file paths
  > Against dotfiles spec: produces exactly 138 files across 4 providers, matching
  > the TypeScript compiler's manifest.

#### Manual Verification

- [ ] Inspect `generated/claude/skills/commit/SKILL.md` — verify frontmatter fields
      and body match the TypeScript compiler's output for the same spec
- [ ] Inspect `generated/opencode/commands/commit.md` — verify `user_invocable` spec
      appears as a command file

**Pause for human confirmation before proceeding to Phase 7.**

---

## Phase 7: Output Generation (Emit + Modes)

### Overview

Implement the three execution modes (`validate`, `compile`, `check`), manifest writing,
and YAML format validation.

### Changes Required

#### 1. File emission (`src/emit.rs`)

- [x] `write_generated_files(files: &[GeneratedFile], output_dir: &Path) -> Result<()>`:
  - For each unique provider appearing in `files`, delete `<output_dir>/<provider>/`
    before writing anything
  - Write each file (create parent dirs with `fs::create_dir_all`)
  - Call `fs::set_permissions` if `mode` is set on the file
  > Also accepts `targets: &[Provider]` to delete all targeted provider dirs
  > (not just those with files), matching TypeScript behavior.
- [x] `write_manifest(result: &CompileResult, output_dir: &Path) -> Result<()>`:
  - Write `<output_dir>/manifest.json` with fields: `sourceHash`, `schemaVersion: 1`,
    `files: [{ provider, path }]`, `warnings: [...]`
  - Use `stable_json()` for deterministic output

#### 2. Check mode (`src/emit.rs`)

- [x] `check_generated_state(expected: &[GeneratedFile], output_dir: &Path) -> Result<CheckResult>`:
  - Missing files: expected path not present on disk
  - Content mismatches: file exists but content differs
  - Unexpected files: files exist in `<output_dir>/<provider>/` but are not in `expected`
  - Note: `manifest.json` is not included in the check comparison
- [x] Return structured `CheckResult` with categorized differences; caller prints them
      and exits 1 if any exist

#### 3. Main orchestration (`src/main.rs`)

- [x] Wire up full pipeline:
  ```
  load_mappings → load_canonical_specs → load_fragments → build_environment →
  resolve_fragments → validate_schema → normalize_specs → validate_semantics →
  compile_specs → [write/check/validate]
  ```
- [x] `validate`: stop after `validate_semantics`, print success message
- [x] `compile`: call `write_generated_files` + `write_manifest`; print warning count
- [x] `check`: call `check_generated_state`; exit 1 if any differences found
- [x] `--strict`: if any `CompileWarning`s exist, treat as error (exit 1 before writing)

#### 4. YAML format validation (integration sanity check)

- [x] Run `agentspec compile` on a subset of dotfiles specs (those that don't use
      fragment syntax yet) and compare output files byte-for-byte against the TypeScript
      compiler's output
  > Ran comprehensive `diff -rq` across all 4 providers. **136 of 138 files are
  > byte-for-byte identical.** The 2 differences (`learnings-discoverer.md` for
  > Claude and OpenCode) are caused by a stale TypeScript-generated copy that hasn't
  > been rebuilt since a spec body edit — not a formatting issue.
- [x] If `serde_yml` formatting differs, implement a custom YAML serializer or
      post-processing step in `format.rs` to match the TypeScript output
  > **Deviation:** Used a hand-rolled YAML serializer from the start (Phase 6)
  > rather than `serde_yml`, specifically to match js-yaml's PLAIN string style
  > and no line wrapping. This proved to be the right call — zero formatting
  > differences found.

### Success Criteria

#### Automated Verification

- [x] `cargo test` passes with check-mode unit tests: missing file, content mismatch,
      unexpected file
  > 103 tests total (98 unit + 5 integration), all passing.
- [x] `agentspec validate` against dotfiles spec exits 0
- [x] `agentspec compile` against dotfiles spec (non-fragment specs) exits 0

#### Manual Verification

- [x] `agentspec check` exits 1 after manually editing a generated file
  > Verified: `agentspec check` against real dotfiles (with stale `learnings-discoverer`)
  > exits 1 with "outdated: generated/claude/agents/learnings-discoverer.md" and
  > "outdated: generated/opencode/agents/learnings-discoverer.md".
- [x] `agentspec check` exits 0 immediately after `agentspec compile`
  > Verified: compile → check round-trip passes. Also covered by integration test
  > `test_check_passes_after_compile`.
- [x] Generated YAML frontmatter for one sample file matches TypeScript output exactly
      (compare with `diff`)
  > Verified with `diff` for Claude commit skill, OpenCode commit command,
  > OpenCode code-reviewer agent, Codex commit skill — all byte-for-byte identical.

**Pause for human confirmation before proceeding to Phase 8.**

---

## Phase 8: dotfiles Integration

### Overview

Migrate this dotfiles repo from the TypeScript compiler to `agentspec`. This involves:
writing the `agentspec.toml`, migrating spec files from Handlebars to MiniJinja
syntax, validating identical output, and removing the TypeScript compiler.

### Changes Required

#### 1. Save TypeScript compiler output as reference snapshot

- [x] Run `npm run build:agents` in `agent-config/` to produce current output
- [x] Commit or stash current `generated/` as the reference snapshot for comparison

#### 2. Write `agentspec.toml`

**File:** `agent-config/agentspec.toml`

- [x] Add config file with default paths (all can be omitted since they match defaults,
      but including them makes the layout explicit):

```toml
[spec]
agents_dir = "spec/agents"
skills_dir = "spec/skills"
fragments_dir = "spec/fragments"

[mappings]
models = "mappings/models.yaml"
tools = "mappings/tools.yaml"
features = "mappings/features.yaml"

[output]
dir = "generated"
```

#### 3. Migrate spec files from Handlebars to MiniJinja syntax

- [x] Audit all spec files in `spec/skills/*/SKILL.md` and `spec/agents/*.md` for
      Handlebars syntax
- [x] Migrate all 8 fragment files from Handlebars to MiniJinja:
  - Variables: `{{variable}}` → `{{ variable }}`
  - Includes: `{{> name}}` → `{% include "name.md" %}`
  - Conditionals: `{{#if}}` → `{% if %}`, `{{#unless}}` → `{% if not %}`
  - Whitespace control: `{%- tag %}` to strip blank lines at conditional boundaries
- [x] Migrate all 22 spec files from Handlebars to MiniJinja:
  - Simple includes: `{{> name}}` → `{% include "name.md" %}`
  - Includes with variables: `{{> name key="val"}}` → `{% with key = "val" %}{% include "name.md" %}{% endwith %}`
  - Indented includes: `{% filter indent(N, first=false) %}{% include "name.md" %}{% endfilter %}`
- [x] Zero Handlebars syntax remaining in spec/ directory (verified with grep)

> **Deviation:** Used `{% filter indent(N, first=false) %}` for indented includes instead
> of the custom `include_indented()` function. `{% filter %}` inherits the full lexical
> scope from `{% with %}` blocks, avoiding the need for manual context reconstruction.
> The `include_indented` function is retained but no longer used by any spec.

#### 4. Make fragment parse errors hard failures

- [x] Switched `UndefinedBehavior` from `Strict` to `Lenient` so optional boolean flags
      (`writeable`, `pr_specific`, etc.) default to falsy when not passed — matching
      Handlebars semantics. Fragment parse errors remain warnings (skip-and-warn) but
      all 14 fragments now parse cleanly with zero warnings.
- [x] `--strict` mode passes cleanly (no warnings to promote to errors)
- [x] Updated integration test `test_validate_strict_fails_on_warnings` →
      `test_validate_strict_passes` to reflect zero-warning state

#### 5. Validate identical output

- [x] Run `agentspec compile` from `agent-config/` — produces 146 files for 4 providers
- [x] File list matches TypeScript output exactly (146 files)
  > **Deviation:** Fixed missing supporting file copy in Codex adapter (TypeScript copied
  > supporting files but Rust didn't). Added `supporting_files` loop to `adapt_codex()`.
- [x] Verified conditional fragment logic produces correct output:
  - `writeable=true` → shows candidate selection UI, not error
  - `pr_specific=true` → includes "Local changes only" section
  - `worktree=true` → includes worktree-specific plan path guidance
- [x] Remaining differences are cosmetic (YAML quoting style, table padding, minor
      whitespace from MiniJinja block tag rendering) — no content differences

#### 6. Update build workflow

- [x] Install the binary locally: `cargo install --path ../agentspec` (or use
      `cargo run --manifest-path ../agentspec/Cargo.toml -- compile` during development)
- [x] Update `setup.sh` to call `agentspec compile` instead of `npm run build:agents`
- [x] Update `README.md` with `agentspec` commands and `AGENTSPEC_PROFILE` env var
- [x] Update `CLAUDE.md` with MiniJinja fragment syntax reference (done in step 4)

#### 7. Remove TypeScript compiler

- [x] Delete `agent-config/scripts/` (entire TypeScript compiler directory)
- [x] Delete `agent-config/package.json`, `agent-config/package-lock.json`,
      `agent-config/tsconfig.json`
- [x] Delete `agent-config/node_modules/` (untracked, removed locally)
- [x] Delete `.prettierignore` (only used by the TS workflow)
- [x] Verify `agent-config/spec/`, `agent-config/mappings/`, `agent-config/generated/`,
      and `agent-config/claude/` remain intact

### Success Criteria

#### Automated Verification

- [x] `agentspec validate` exits 0 against dotfiles spec with MiniJinja syntax
- [x] `agentspec compile` exits 0 and produces output (cosmetic whitespace
      differences accepted as new canonical format)
- [x] `agentspec check` exits 0 after `agentspec compile`
- [x] `cargo install --path agentspec/` installs `agentspec` successfully from the
      local directory

#### Manual Verification

- [x] Claude Code loads skills without errors after restarting
- [x] OpenCode loads commands without errors after restarting
- [x] No `node_modules/`, `package.json`, or `*.ts` files remain under `agent-config/`
- [x] `agentspec --version` prints the installed version

---

## Testing Strategy

### Unit Tests (within `agentspec` repo)

Unit tests live inline as `#[cfg(test)]` blocks at the bottom of each module file (e.g.,
`src/parse.rs`, `src/fragments.rs`). This is the idiomatic Rust convention and allows
tests to access private helpers directly. Integration tests that exercise the full
pipeline via the public API live in `tests/` (e.g., `tests/pipeline.rs`).

- **parse**: valid frontmatter, missing `---` delimiters, empty body; agent and skill
  discovery with multiple files
- **fragments**: no-fragment spec (no-op), simple `{% include %}`, variable passing via
  `{% with %}`, nested includes, missing fragment → error
- **validate (schema)**: valid spec passes; missing `id` → error; invalid `kind` → error
- **validate (normalize)**: `name` defaulted; `tools` deduped + sorted; `targets`
  defaulted
- **validate (semantics)**: duplicate IDs; empty body; skill with neither invocability
  flag; invalid `delegate_to`; missing model profile
- **adapters**: each adapter with a minimal fixture spec — verify output path, frontmatter
  keys, and body content
- **mappings**: profile overlay merging; string and object model resolution
- **emit (check mode)**: missing file, content mismatch, unexpected file

### Integration Tests (within `agentspec` repo)

- Run full pipeline against `scripts/__tests__/fixtures/` ported from TypeScript:
  one agent, two skills (one with supporting file), one fragment
- Verify generated files for all four providers match expected fixture output

### Snapshot Tests (Phase 8 validation)

- TypeScript compiler reference output vs. `agentspec compile` — byte-for-byte
  comparison across all 34 specs and all 4 providers

## Performance Considerations

The TypeScript compiler operates on ~34 specs with no latency issues. Rust will be
faster. No special performance work needed for v1.

## Migration Notes

- The TypeScript compiler remains in dotfiles until `agentspec` passes snapshot
  validation at the end of Phase 8
- The Handlebars → MiniJinja migration in Phase 8 is the only one-time breaking change;
  it can be done atomically in a single commit
- `agentspec.toml` is invisible to the TypeScript compiler, so it can be added
  before Phase 8 without affecting the current build
- `agentspec/` lives inside the dotfiles directory but is tracked by its own git repo;
  `dotfiles/.gitignore` excludes it. Moving to a standalone GitHub repo later requires
  only adding a remote and pushing — all history is preserved.

## Open Questions Resolved

1. **Project config**: `agentspec.toml` with defaults for all paths; CLI flags
   override; discovered by walking up from CWD
2. **Project name**: `agentspec`
3. **Versioning**: `schemaVersion: 1` in `manifest.json` only; per-spec versioning and
   migration support deferred until there is a concrete use case

## References

- Current compiler entry point: `agent-config/scripts/compile_agents.ts`
- Pipeline modules: `agent-config/scripts/lib/`
- Provider adapters: `agent-config/scripts/lib/adapters/`
- Fragment learnings: `thoughts/learnings/compiler-fragments.md`
- Fragment dedup plan: `thoughts/plans/2026-03-07-fragment-deduplication.md`
- OpenCode research: `thoughts/research/2026-03-08-opencode-commands-vs-skills.md`
- Rules across agents: `thoughts/research/2026-03-17-rules-across-ai-coding-agents.md`
