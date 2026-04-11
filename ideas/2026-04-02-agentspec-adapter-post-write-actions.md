# Colocate Post-Write Actions With Adapters

## Problem Statement

The emit stage contains provider-specific post-write logic that violates the
principle that adapter-specific concerns belong in adapters. Today,
`patch_opencode_instructions` is ~90 lines of OpenCode-specific code in
`emit.rs` that patches `opencode.json` with rule file paths after sync writes.
The `ConfigPatch` enum in `plan.rs` is also OpenCode-specific. Adding a second
provider with post-write needs (e.g., Cursor plugin manifests — TODO #3) would
require adding another variant to the library crate's enum and another match arm
in emit, further scattering provider knowledge.

This matters for adapter authoring: someone implementing a new provider adapter
shouldn't need to touch emit or plan to add post-write behavior. All
provider-specific logic should be discoverable in one place.

## Motivation

- **Colocation**: The `patch_opencode_instructions` function is OpenCode-specific
  but lives in `emit.rs`, far from the OpenCode adapter. A reader looking at
  `adapters/opencode.rs` has no idea this post-write logic exists.
- **Adapter authoring**: Adding a new provider should mean implementing one adapter
  module. Currently it also means extending `ConfigPatch` in `plan.rs` and adding
  a match arm in `emit.rs`.
- **Consistency**: We just moved prefix/strip transforms into adapters for exactly
  this reason. Post-write actions are the remaining provider-specific leak in emit.

## Context

### Current flow

1. `sync_plan` in `sync.rs` creates `ConfigPatch::OpenCodeInstructions { rules_dest_dir, config_dir }`
   when it encounters OpenCode rules
2. `WritePlan` carries `patches: Vec<ConfigPatch>` alongside `writes: Vec<FileWrite>`
3. `emit.rs` `apply_patches` matches on `ConfigPatch::OpenCodeInstructions` and calls
   `patch_opencode_instructions`
4. `patch_opencode_instructions` reads/creates `opencode.json`, replaces agentspec-owned
   entries in the `instructions` array, preserves user-owned entries, and writes the file

### Why this can't simply move to compile time

Unlike frontmatter transforms (which operate on content the adapter produces),
post-write actions depend on **destination paths** — information that only exists
after plan construction. The adapter at compile time doesn't know where files will
be written. So the adapter can declare *what* should happen, but the *where* must
be resolved later (at plan time) and the *execution* happens at emit time.

### Related work

- The `AdapterConfig` pattern established this session shows how adapters can
  receive plan-time information via a config struct. A similar pattern could
  work for post-write action specifications.

## Goals / Success Criteria

- [ ] `patch_opencode_instructions` logic is colocated with the OpenCode adapter,
      not in emit
- [ ] `ConfigPatch` enum is either removed or replaced with something adapters
      produce, not something the library crate defines per-provider
- [ ] Emit has no provider-specific match arms — it executes post-write actions
      generically
- [ ] Adding a new provider's post-write action requires changes only in the
      adapter (and possibly plan construction), not in emit
- [ ] Post-write actions only execute for `sync`, not `compile`
- [ ] Existing behavior preserved: `opencode.json` patching works identically

## Non-Goals (Out of Scope)

- Implementing Cursor plugin manifest generation (TODO #3) — this idea enables
  it but doesn't implement it
- Building a general-purpose hook or event system — the goal is colocation, not
  extensibility for its own sake
- Changing what `patch_opencode_instructions` does — only where it lives and how
  it's invoked

## Proposed Solutions

### Chosen approach: Adapter-provided hooks

Adapters provide a hook function that emit calls with the destination path after
writing. The adapter owns the logic entirely — emit just calls the hook without
knowing what it does.

```
Adapter  →  provides hook (e.g., patch_opencode_instructions)
sync_plan →  attaches hook to the FileWrite or WritePlan
emit     →  calls hook.run(dest, dry_run) after writing the batch
```

- `patch_opencode_instructions` moves from `emit.rs` to `adapters/opencode.rs`
- `ConfigPatch` enum is replaced by an optional `Box<dyn PostWriteHook>` (or
  similar) on `WritePlan`
- Emit calls the hook if present, passing destination path and dry_run flag
- `sync_plan` attaches the hook when constructing OpenCode rules writes

Why this over alternatives:
- **Action descriptors** (rejected): spreads logic across three layers (adapter
  declares, plan resolves, emit executes) — worse fragmentation than today
- **Raw closures** (rejected): opaque, hard to test, can't dry-run
- **Move to sync_plan** (rejected): violates plan/emit separation, logic still
  outside adapter

### Rejected alternatives

**Action descriptors**: Adapter returns a `PostWriteAction` enum describing what
should happen; plan resolves paths; emit executes. This fragments the logic across
three modules — the whole point is colocation, and this makes it worse.

**Move patching into sync_plan**: Simplest change but violates plan/emit separation
(plans are data, emit is execution) and the patching logic still lives outside the
adapter.

## Constraints

- Post-write actions depend on destination paths (plan-time information)
- The plan-as-data model (`WritePlan` is a struct, not a set of side effects)
  should be preserved — plans should remain inspectable for dry-run
- `compile` command must NOT execute post-write actions (sync-only)
- Existing `patch_opencode_instructions` tests should remain (relocated, not deleted)

## Resolved Questions

1. **Should `CompileResult` carry post-write actions?** Not needed with the hook
   approach. The hook function lives in the adapter module. `sync_plan` imports
   it and attaches it to the `WritePlan`. No compile-time output needed.

2. **How does the plan resolve paths?** `sync_plan` already computes destination
   paths and `opencode_config_dir`. It passes these to the hook when attaching
   it. The adapter function receives concrete paths — no resolution needed inside
   the hook.

3. **Is `ConfigPatch` the right abstraction?** No. Replace it with a hook
   mechanism (`Box<dyn PostWriteHook>` or similar). The `ConfigPatch` enum and
   its provider-specific variants are deleted.

## Open Questions

1. **What's the hook trait signature?** The hook needs `dest_dir`, `dry_run`, and
   possibly other context (like the opencode config dir, which is distinct from
   the rules dest dir). Should the trait take a generic context struct, or should
   each hook capture what it needs via closure state?

2. **Where does `sync_plan` get the hook?** Currently `sync_plan` creates
   `ConfigPatch` directly. With hooks, it needs access to the adapter's hook
   function. Should adapters export a `post_write_hook()` function that
   `sync_plan` calls, or should the hook be returned alongside `GeneratedFile`
   from the adapter?

## References

- `src/emit.rs:238-322` — `patch_opencode_instructions` implementation
- `src/emit.rs:325-335` — `apply_patches` match on `ConfigPatch`
- `src/plan.rs:119,149-157` — `WritePlan.patches` and `ConfigPatch` enum
- `src/sync.rs:43-46` — `sync_plan` creating `ConfigPatch::OpenCodeInstructions`
- `src/sync.rs:168-178` — `opencode_config_dir` resolution
- `CLAUDE.md` — "Provider-specific logic belongs in adapters" principle
- `TODO.md` item #6 — this idea's originating TODO
