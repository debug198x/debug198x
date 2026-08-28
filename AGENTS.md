# Debug198x

> Read [`PRINCIPLES.md`](PRINCIPLES.md) first. [`MANIFESTO.md`](MANIFESTO.md) is why the project exists.

The Debug198x cross-CPU debug-info format: an NDJSON sidecar carrying a
line↔address map, symbols, sections, and address-space qualifiers. See
[`../../AGENTS.md`](../../AGENTS.md) for umbrella context and cross-project
rules.

## What this is

A format, and a small crate for reading and writing it. An assembler writes a
`.debug198x` file beside its output; an emulator or debugger reads it to offer
source-level breakpoints and symbolised disassembly for machines whose
toolchains never had either.

User-facing overview is in [`README.md`](README.md).

## Why it lives here

Asm198x writes the format and Emu198x reads it, so neither end can own it
without making the other depend on a toolchain it does not need. It graduated
out of Asm198x under the same rule as a Format198x crate — see
[`../../decisions/formats-graduate-to-their-own-projects.md`](../../decisions/formats-graduate-to-their-own-projects.md).
That independence is the constraint to protect: the crate depends on `serde` and
`serde_json` and nothing else, and a change that would pull either end's
concerns into the format is the change to argue about.

## Changing the format

**The format is frozen at v1 (2026-08-18).** Additive change is free; a breaking
shape change needs a new decision in
[`decisions/debug198x-format.md`](decisions/debug198x-format.md), a
`format_version` bump, and a migration path. `format_version` still reads
`"0.1"` — that is the frozen v1 spec's version, not draft status, and bumping it
breaks the Emu198x importer's exact-match check.

Adding a variant to a `#[serde(untagged)]` enum is **not** additive: an
unrecognised variant is a hard parse error failing the whole file, where an
unknown record type or field is skipped.

Both ends have to agree, and the writer ships before the reader can rely on it.
Treat a field addition as a contract change affecting two repositories, not as a
local edit — state what an old reader does with a new file, and what a new
reader does with an old one.

## Naming

`debug198x`, org-prefixed like every published crate in the family. See
[`../../decisions/crate-naming.md`](../../decisions/crate-naming.md).
