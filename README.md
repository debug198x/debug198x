# debug198x

[![CI](https://github.com/debug198x/debug198x/actions/workflows/ci.yml/badge.svg)](https://github.com/debug198x/debug198x/actions/workflows/ci.yml)
[![crates.io](https://img.shields.io/crates/v/debug198x.svg)](https://crates.io/crates/debug198x)

The Debug198x cross-CPU debug-info format: an NDJSON sidecar carrying a
line↔address map, symbols, sections, and address-space qualifiers.

An assembler writes a `.debug198x` file next to its output; an emulator or
debugger reads it to show source-level breakpoints and symbolised disassembly
for machines whose toolchains never had either.

## Reading a sidecar costs you serde

The crate depends on `serde` and `serde_json` and nothing else. That is the
point of it existing separately: a debugger that wants symbol names should not
have to build an assembler, its parser, or its CPU dialects to get them.

```toml
[dependencies]
debug198x = "0.1"
```

## Who writes and reads it

The direction is one-way, deliberately:

| | |
|---|---|
| **Writes** | [Asm198x](https://github.com/asm198x/asm198x) |
| **Reads** | [Emu198x](https://github.com/emu198x/emu198x) |

Neither is a dependency of this crate, and this crate must never gain a
dependency on either. The format is the contract between them; if it needed one
of them to be understood, it would not be a format.

## The format

The format is **frozen at v1**. New record types and new fields may be added
without a version break, and a conforming reader skips unknown record types and
ignores unknown fields. A breaking shape change requires a `format_version`
bump and a migration path.

`format_version` reads `"0.1"` and denotes that frozen v1 specification. It is
deliberately not `"1.0"`: the freeze changed the promise rather than the wire
shape, and consumers validate the string by exact match.

| | |
|---|---|
| **Specification** — write a reader or a writer against this | [`debug198x.md`](https://github.com/asm198x/docs/blob/main/debug198x.md) |
| **Governance** — why it is this way, and how it may change | [`decisions/debug198x-format.md`](decisions/debug198x-format.md) |

## History

This crate began inside [Asm198x](https://github.com/asm198x/asm198x) and moved
here on 2026-08-27, once Emu198x shipped a reader and the format had a consumer
that was not its own producer. Its git history came with it. Released versions
restart at 0.1.0; the 0.0.x line was Asm198x's lockstep version and mostly
recorded that project's releases rather than changes to this format.

## Licence

GPL-2.0-or-later. See [`LICENSE`](LICENSE).
