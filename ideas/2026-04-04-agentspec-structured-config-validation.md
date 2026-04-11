# Structured Config Validation with Rich Error Reporting

## Problem Statement

agentspec's sync command validates configuration across multiple points — CLI
flags, TOML config sections, and cross-field relationships between them. Today,
each validation point uses `bail!()` independently, meaning users discover config
problems one at a time through fix-and-retry cycles. A user with 3 config issues
must run the command 3 times to find them all.

The error messages themselves are decent (they include fix suggestions), but
they're scattered across `config.rs`, `sync.rs`, and `emit.rs` with no
centralized validation pass. Adding a new check means finding the right spot to
insert another `bail!()` and crafting a one-off error message.

## Motivation

- **Better user experience**: Users should see all config problems at once, not
  discover them serially through repeated invocations.
- **Maintainability**: A centralized validation pass is easier to extend than
  scattered bail points. Each check becomes an enum variant with structured fix
  suggestions rather than an ad-hoc format string.
- **Consistency**: Fix suggestions should follow a predictable format — every
  error should tell the user both the TOML config fix and the CLI flag
  alternative (where applicable).

## Context

### Current validation points (5 total)

1. **`sync.rs`** — `--dest` requires `--provider` (CLI-only constraint)
2. **`sync.rs`** — No sync providers configured at all (TOML + CLI)
3. **`config.rs`** — Provider not configured + insufficient CLI flags
   (cross-field: TOML config state + CLI flag combination)
4. **`sync.rs`** — `mode = path` but no `dir` configured (cross-field)
5. **`emit.rs`** — File collision at sync destination without `--force`

These checks are inherently **cross-field and contextual** — validity depends on
combinations of CLI flags, TOML config sections, and provider state. Simple
per-field validation (email format, string length) doesn't apply here.

### Ecosystem research

- **`garde`** — Derive-based validation that collects all errors. Supports
  cross-field rules via a context struct. Limitation: custom validators receive
  one field + context, not the whole struct, so cross-field rules require copying
  relevant field values into the context. This ceremony adds up when most rules
  are inherently cross-field.
- **`miette`** — Rich diagnostic error presentation (help text, related errors,
  error codes). No validation logic — purely a presentation layer. The
  `#[related]` attribute collects multiple errors into one report. The
  `#[diagnostic(help("..."))]` attribute adds fix suggestions.
- **`figment`** — Config loading/merging with good deserialization error
  provenance, but no semantic cross-field validation.
- **Manual accumulation** — `Vec<Error>` pattern: run all checks, collect
  failures, report everything at once.

## Goals / Success Criteria

- [ ] All sync config validation errors are reported together in a single
      invocation (no more fix-and-retry cycles)
- [ ] Each error includes a structured fix suggestion covering both TOML config
      and CLI flag alternatives
- [ ] Adding a new validation check is a matter of adding an enum variant and a
      check function — no hunting for the right `bail!()` insertion point
- [ ] Error output is clear and well-formatted for terminal display
- [ ] No regression in error quality for existing validation points

## Non-Goals (Out of Scope)

- **Source-span pointing into TOML files** — Pointing to specific line/column in
  the config file would be nice but requires tracking source spans through
  parsing, which is a much larger change. The current approach of naming the
  field/section is sufficient.
- **Validating spec content** — Spec semantic validation (duplicate IDs, unknown
  presets) already has its own collect-all-errors pattern in `validate.rs`. This
  idea is specifically about sync/config validation.
- **Switching config loading to `figment`** — The current `toml` +
  `serde_path_to_error` approach works well for deserialization. This idea is
  about the semantic validation pass that runs *after* deserialization.

## Proposed Solutions

### Option A: Manual accumulation + `miette` (Recommended)

Skip `garde` (since almost all rules are cross-field) and use a manual
`validate_sync_config()` function that runs all checks into a
`Vec<SyncConfigError>`. Use `miette` for presentation — each error variant gets
`#[diagnostic(help("..."))]` and the top-level error uses `#[related]`.

```
[sync configuration] 2 errors found

  x sync is not configured for cursor
  help: add [sync.cursor] to agentspec.toml, or use --mode user|project

  x `--dest` requires an explicit `--provider`
  help: use --provider <provider> --dest <path>
```

**Pros**: Simple, readable, easy to extend. No ceremony around context structs.
**Cons**: Adds `miette` + `thiserror` dependencies.

### Option B: `garde` + `miette`

Use `garde` derive for validation logic with a context struct, then convert
`garde::Report` errors into `miette::Diagnostic` types.

**Pros**: Declarative validation on the struct.
**Cons**: Cross-field rules require copying field values into a context struct.
Most checks don't benefit from per-field derive since they're inherently
relational. More ceremony for little gain at current scale.

## Constraints

- Some checks are order-dependent (e.g., can't validate per-provider targets if
  no providers exist at all). The validation function should still short-circuit
  on fatal precondition failures.
- The emit-time collision check (#5) happens during file I/O, not at config
  validation time. It may need to remain a runtime error rather than being pulled
  into the upfront validation pass.

## Open Questions

- Should the `validate` subcommand also run sync config validation (when sync
  sections exist in TOML), or only spec content validation?
- Is `miette` worth the dependency, or would a simple custom `Display` impl on
  the error enum be sufficient? The project currently uses `anyhow` throughout.
- Should this pattern extend to compile config validation too, or is sync the
  only command with enough validation complexity to justify it?

## References

- [miette crate](https://docs.rs/miette/latest/miette/) — rich diagnostic errors
- [garde crate](https://docs.rs/garde/latest/garde/) — derive-based validation
- Current validation points: `src/config.rs:203`, `src/sync.rs:86,96`,
  `src/sync.rs:126`, `src/emit.rs:177`
