# Dynamic Shell Completions for Repo/Group Arguments

## Problem Statement

gityard's current shell completions are statically generated from the clap
schema. This means tab-completion only offers subcommand names and flags — it
cannot suggest repo aliases or group names because those are runtime data stored
in the registry (`repos.toml`). Users must remember or look up exact alias and
group names manually, which defeats much of the convenience a CLI tool should
provide.

## Motivation

- **Discoverability**: Users managing many repositories shouldn't need to run
  `gityard list` to remember an alias before typing `gityard remove <alias>`.
  Tab-completion should surface these values directly.
- **Correctness**: Completing from the actual registry eliminates typos in alias
  and group names, reducing errors from mistyped arguments.
- **CLI polish**: Dynamic completions are a standard expectation for modern CLI
  tools that operate on user-defined named resources.

## Context

- Static completions are generated via `clap_complete::generate()` in
  `src/main.rs:24-29`. The `completions` subcommand is handled before any
  config/registry loading, so it has no access to runtime data.
- The registry (`src/registry.rs`) stores repo entries in `repos.toml` with
  aliases (auto-derived from directory names) and group memberships. It provides
  methods like `list_groups()` and iteration over `RepoEntry` values.
- clap_complete supports dynamic completions via `CompleteEnv` +
  `ArgValueCompleter` behind the `unstable-dynamic` feature flag.
- Current dependencies: `clap = "4"` with `derive` feature,
  `clap_complete = "4"`.

## Goals / Success Criteria

- [ ] Tab-completing repo alias arguments (e.g., `gityard remove <tab>`)
      suggests registered repo aliases from the registry
- [ ] Tab-completing group name arguments (e.g., `gityard group remove <tab>`,
      `--group <tab>`) suggests registered group names from the registry
- [ ] Dynamic completions work for **all** commands/flags that accept repo
      aliases or group names: `remove`, `group add`, `group remove`, `--group`
      flags on `status`/`exec`/`worktrees`/`branches`/`trim`, and any other
      arguments that reference registry data
- [ ] The existing static `completions` subcommand is replaced by the
      `CompleteEnv`-based approach
- [ ] Completions work across bash, zsh, and fish (the shells clap_complete's
      `CompleteEnv` supports)

## Non-Goals (Out of Scope)

- Fancy UX extras like showing group members in completion descriptions,
  filtering already-added repos, or completing filesystem paths for `add`
- PowerShell or elvish support (unless trivially supported by `CompleteEnv`)
- Keeping the static `completions` subcommand as a fallback — this is a full
  replacement

## Constraints

- `CompleteEnv` + `ArgValueCompleter` requires the `unstable-dynamic` feature
  flag on `clap_complete`, which means the API may change in future clap
  releases. This is an accepted tradeoff for the functionality it provides.
- Dynamic completions require the binary to be invoked at tab-time to produce
  suggestions, so completion speed depends on how fast the registry can be
  loaded and queried. Registry loading should be fast for typical registry sizes
  (tens to low hundreds of repos).
- Shell setup instructions will change — users will need to update their shell
  config from the static script to the `CompleteEnv` registration pattern.

## Research Findings

These questions were originally open; research resolved them (2026-04-16).

### Integration with clap derive

`ArgValueCompleter` attaches to individual fields via `#[arg(add = ...)]` — no
restructuring of the `Cli` enum required. The completer must be `'static`, so it
cannot capture shared app state. Each completer loads the registry itself (e.g.,
reads `repos.toml` directly).

### User-facing shell setup

Users add a one-liner to their shell rc file:

- **bash**: `source <(COMPLETE=bash gityard)`
- **zsh**: `source <(COMPLETE=zsh gityard)`
- **fish**: `COMPLETE=fish gityard | source`

This replaces the static `completions` subcommand entirely. The setup is
self-correcting — new subcommands/args are picked up on shell restart.

### Performance

The binary is invoked on every tab press, but `CompleteEnv` short-circuits
before normal app init. The completer only needs to parse `repos.toml` — a small
file for typical usage. This should be imperceptible (single-digit ms).

### Migration gotchas

- `CompleteEnv::with_factory(cli).complete()` must be the **first line** of
  `main()` — anything writing to stdout before it will corrupt shell output.
- Binary name must match what the shell expects; verify
  `Command::get_bin_name()` returns `"gityard"`.
- The `unstable-dynamic` feature name is correct for clap_complete 4.x. The
  shell-side interface may change between versions, but the Rust API is
  relatively stable.

## Open Questions

_(None remaining — all resolved by research above.)_

## References

- [TODO.md entry](../../../jasnross/gityard/TODO.md) — original task description
- [clap_complete dynamic completions docs](https://docs.rs/clap_complete/latest/clap_complete/env/index.html)
- `src/main.rs:24-29` — current static completion generation
- `src/cli.rs` — all CLI argument definitions
- `src/registry.rs` — `Registry`/`RepoEntry` structs and persistence
