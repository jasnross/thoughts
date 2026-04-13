# Standalone Agent Config Compiler

## Problem Statement

The `agent-config/` compiler in this dotfiles repo is a useful tool for maintaining
skills and agent configurations across multiple AI coding tools (Claude Code, Cursor,
Codex, OpenCode) from a single canonical source. However, it is tightly coupled to
this dotfiles repo's directory layout, making it impossible for other projects or people
to adopt the same workflow without forking or reimplementing the tooling.

Extracting the compiler into a standalone Rust binary would allow anyone managing AI
agent configurations to benefit from the spec-driven compilation approach without
depending on this dotfiles repo or requiring a Node.js runtime.

## Motivation

- **Reusability**: The compilation pipeline (spec loading → fragment resolution →
  schema validation → provider adaptation → output generation) is entirely generic.
  Only the spec _content_ is project-specific.
- **Runtime independence**: A self-contained binary (no Node.js requirement) removes
  a significant adoption barrier for people who don't have a Node.js toolchain.
- **Standard format**: A standalone tool can establish and document the canonical
  `SKILL.md`/`AGENT.md` spec format as a public standard that other projects can
  adopt with confidence.
- **Dogfooding**: This dotfiles repo will migrate to consuming the standalone tool
  early in development — reducing maintenance burden and providing real-world
  validation during the build.

## Context

### Current architecture

The compiler lives in `agent-config/scripts/` and implements a TypeScript pipeline:

1. **Spec loading**: Reads `spec/skills/<id>/SKILL.md` and `spec/agents/<id>/AGENT.md`
   (YAML frontmatter + Markdown body)
2. **Fragment resolution**: Resolves Handlebars partials (`{{> category/name}}`) from
   `spec/fragments/` at compile time, before schema validation
3. **Schema validation**: Validates resolved specs against `spec/schema/canonical.schema.json`
4. **Provider adaptation**: Applies provider-specific mappings and adapters for Claude
   Code, Cursor, Codex, and OpenCode
5. **Output generation**: Writes generated provider configs to `generated/`

### What's dotfiles-specific vs. generic

| Component | Ownership |
|---|---|
| Compilation pipeline | Generic → ships with the tool |
| Provider adapters (Claude, Cursor, Codex, OpenCode) | Generic → ships with the tool |
| Spec format (SKILL.md / AGENT.md structure) | Owned by the tool, documented publicly |
| JSON schema (`canonical.schema.json`) | Owned by the tool, shipped with the binary |
| Fragment resolution | Generic → ships with the tool |
| Spec content (`spec/skills/`, `spec/agents/`, `spec/fragments/`) | Stays in user's project |

### Prior art in this repo

- Fragment system implemented and documented in `thoughts/learnings/compiler-fragments.md`
- Fragment deduplication work in `thoughts/ideas/2026-03-07-fragment-deduplication.md`
- Shared compiler fragments idea in `thoughts/ideas/2026-03-07-shared-compiler-fragments.md`

## Decisions

- **Language**: Rust. Native binary, no runtime dependency, `cargo install --git <url>`
  as the install path. Publishing to crates.io deferred until proven.
- **Templating engine**: MiniJinja (replaces Handlebars). Jinja2-style syntax, runtime
  template loading, actively maintained, covers all current Handlebars use cases
  (includes with variable passing, nested partials). Spec files using `{{> ...}}` syntax
  will be migrated to MiniJinja syntax as part of the port.
- **Fragment library**: Project-local only. No built-in fragments ship with the tool.
- **dotfiles migration timing**: Migrate this repo early — use it as the dogfooding
  target during development rather than waiting for a stable v1.0.

## Goals / Success Criteria

- [ ] The compiler exists as a standalone Rust binary in a separate OSS repo
- [ ] Any project can use the tool by pointing it at a directory of spec files — no
      knowledge of this dotfiles repo required
- [ ] The tool ships with built-in provider adapters for Claude Code, Cursor, Codex,
      and OpenCode
- [ ] The canonical spec format (SKILL.md / AGENT.md + JSON schema) is defined and
      documented by the tool, not by this dotfiles repo
- [ ] The fragment system (compile-time transclusion with variable passing) is included
      and documented, using MiniJinja as the template engine
- [ ] Three commands are supported: `validate` (dry-run), `compile` (write output),
      `check` (compare expected vs. disk without writing — useful for CI)
- [ ] Mapping profiles (`--mapping-profile` / env var) allow users to switch between
      model configurations (e.g., home vs. work) without editing spec files
- [ ] This dotfiles repo migrates to using the standalone tool (removes its own compiler)
- [ ] Installation via `cargo install --git <url>` or building from source (no Node.js required)

## Non-Goals (Out of Scope)

- Plugin system for third-party provider adapters (may be valuable later; start with built-ins)
- Built-in fragment library (fragments are always project-local)
- GUI or web interface
- Cloud-hosted compilation service
- Spec content (the actual skill/agent definitions) — those stay per-project
- Changing the existing spec format or provider adapters during the extraction (extract
  first, then iterate)
- Publishing to crates.io or any package registry (deferred until the tool is proven)

## Implementation Approach

Port the TypeScript compiler to Rust, module by module. The existing pipeline structure
maps cleanly to Rust:

- **Key crates**: `minijinja` (fragment resolution), `serde`/`serde_yaml`/`serde_json`
  (schema handling and YAML serialization), `jsonschema` (AJV equivalent), frontmatter
  parsing (custom or via crate)
- The TypeScript implementation remains in this dotfiles repo during the transition and
  is removed once the Rust tool is validated against the full spec set

## Constraints

- The standalone tool must produce output _identical_ to the current compiler (at least
  for the skills/agents in this dotfiles repo) so migration can be validated
- The fragment system capability must be preserved; MiniJinja syntax will replace
  Handlebars `{{> ...}}` syntax, requiring a one-time migration of spec files
- The spec format must be backwards-compatible with existing specs in this repo during
  the migration period
- The tool should be usable with a single command (e.g., `agent-config compile ./spec`)

## Open Questions

1. **Project config**: How does a user project tell the tool which providers to target,
   where to write output, and where specs live? Options: convention over configuration
   (fixed directory names), a config file (`agent-config.toml`), or CLI flags. Needs design.

2. **Project name**: What is this tool called? Needs a name that communicates the purpose
   without being tied to "dotfiles" or "agent-config" specifically.

3. **Versioning / schema evolution**: If the spec format evolves in the tool, how do
   existing projects upgrade? Does the tool embed a version declaration in output files?
   Does it support multiple schema versions?

## Affected Systems

- `agent-config/` — the current compiler; replaced by the standalone tool once validated
- This dotfiles repo becomes a consumer (adds the tool as a dependency)
- Potentially: the fragment deduplication work in progress (`thoughts/plans/2026-03-07-fragment-deduplication.md`)
  could target the standalone tool instead of the local compiler
