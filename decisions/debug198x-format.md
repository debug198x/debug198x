# Decision: the Debug198x format's evolution policy and freeze governance

**Status:** Active. The format is **frozen at v1** as of 2026-08-18.

**Date:** 2026-07-06.

## What this governs

The Debug198x cross-CPU debug-info format — the `.debug198x` NDJSON sidecar
written by asm198x (`--debug`) and read by consumers, first the Emu198x
importer. The format itself is specified in the org docs repo
([`debug198x.md`](https://github.com/asm198x/docs/blob/main/debug198x.md)),
written for external implementers against the conformance fixture corpus
(`crates/asm198x/tests/fixtures/debug198x/` in
[asm198x](https://github.com/asm198x/asm198x), enforced always-on by
`tests/debug198x_fixtures.rs`). That page describes the format; this record
governs it — why it is the way it is, and under what rules it may change.

Sibling governance in asm198x:
[`core-contract-freeze.md`](https://github.com/asm198x/asm198x/blob/main/decisions/core-contract-freeze.md)
(the draft-then-freeze pattern this record instantiates),
[`v1-scope.md`](https://github.com/asm198x/asm198x/blob/main/decisions/v1-scope.md),
[`packaging-and-cpu-roadmap.md`](https://github.com/asm198x/asm198x/blob/main/decisions/packaging-and-cpu-roadmap.md),
[`syntax-stance.md`](https://github.com/asm198x/asm198x/blob/main/decisions/syntax-stance.md)
(the dialect surface `header.dialect` names).

## The evolution policy

- **Additive, skip-unknown.** New record types and new fields are added without
  a version break; a conforming reader skips unknown `t` values and ignores
  unknown fields (both spec-normative, both fixture-exercised — AE5). The
  `format_version` bumps incompatibly only for a breaking shape change, which
  now requires a new decision here.
- **Untagged enums are the exception, and the trap.** `Space` is
  `#[serde(untagged)]`, so an unrecognised *variant* is not skipped — it is a
  hard parse error failing the whole file:

  ```
  {"space":{"segment":7,"window":2}}  ->  data did not match any variant of untagged enum Space
  {"t":"quantum","whatever":1}        ->  skipped, as designed
  ```

  A new variant of an existing enum is neither a new record type nor a new
  field, so the additive promise does not cover it. Any untagged enum in this
  format must therefore carry a catch-all, or its set of shapes closes
  permanently at the freeze. `Space::Unknown` is that catch-all: it holds an
  unrecognised shape verbatim, the record still resolves through its section,
  and a rewritten file preserves it. The cost is strictness — a misspelled
  qualifier lands in the catch-all rather than being rejected — which is the
  right trade for a data contract whose consumers must not refuse files they
  only partly understand.
- **Decimal wire, hex rendering.** Numbers stay decimal JSON integers;
  presentation belongs to tools (KTD3).
- **No fabricated address data.** A producer emits `space` qualifiers only from
  actual placement, at both symbol and section level; flat CPUs carry nothing
  extra (AE3).

## What the freeze commits to

From 2026-08-18 the additive-evolution guarantee (R11) is irreversible. New
record types and new fields are added without a version break. A **breaking**
shape change requires a new decision in this record, a `format_version` bump,
and a migration path.

Fixture growth stays additive and independent of the freeze (R10).

### `format_version` stays `"0.1"`

The freeze changed the promise, not the shape, so bumping the version string
would misinform every reader: the spec's own rule is *branch on `format_version`
for a breaking change; additive changes never bump it*, and a reader seeing the
number move is entitled to conclude a shape it knows has changed.

It would also break real consumers for nothing. The Emu198x importer validates
`format_version` by **exact match** against the crate constant
(`crates/emu198x-shell/src/debug_info.rs:163`), so a bump makes every
post-freeze sidecar unreadable to any build pinned before it, including the
released Emu198x v0.3.0.

`FORMAT_VERSION` therefore stays `"0.1"`, and `"0.1"` denotes the frozen v1
specification. Changing that pairing is itself a breaking change.

## Address spaces

**`{slot, page}` is the only qualifier shape.** It is fixture-pinned,
cross-checked against a real `Memory128K`, and consumer-exercised. `Paged`
earns its place because two pages genuinely share a CPU address.

**`bank` was withdrawn before the freeze.** Nothing emitted it, nothing consumed
it, and it is redundant on this model: a 65816 address resolves to its full
24-bit value (the fixture's `FARBUF` is 98304 = `$018000`), so the bank byte is
the top 8 bits of something the reader already holds. With the catch-all in
place, adding a real banked shape once a 65816 consumer has a requirement is
safe, and a file carrying the old shape still loads.

**`Section` carries an optional `space`.** Paging was expressible but not
*discoverable*: `space` lived only on address-kind symbols, so a consumer
holding a real paging state had no stated section → (slot, page) mapping to
build a base map from — only the inference "scrape any symbol out of the
section", which a section holding nothing but `line` records defeats entirely.
A record's own `space` is the finer truth where it carries one; the section's is
the section-wide default. The field is additive on the wire and skipped when
absent.

**The base map is the paging channel.** Map only the sections paged in and both
banks resolve correctly. A consumer that maps two pages into one slot at once
has described a state the hardware cannot be in, and will read the resulting
record-order answer as a reader bug.

**`Eq` is not available** on `Space`, `SymbolKind`, `Symbol`, `Section` or
`DebugInfo`, nor on asm198x's `DebugData` and `AssemblyResult`, because
arbitrary JSON has no total equality. `PartialEq` remains everywhere; `Header`
and `LineSpan` keep `Eq`.

## Trimmed leading gaps become a `reserved` section

On the asl-family dialects `p2bin` starts the image at the lowest *written*
address, so a leading `org` gap or reservation is absent from the file rather
than padding it. That moves `Assembly::origin` above where the source's own
addresses start, and `SymbolKind::Label` carries an **unsigned** section-relative
`offset` — so a label in the dropped region, such as `buf: ds 256` before the
code, has no representable location.

The dropped region is therefore its own section: `name: "reserved"`, `base` at
the source's original origin, contributing no bytes. Symbols below the boundary
are placed in it; line spans for it are dropped, since a span describes bytes
that exist. This is a usage decision, not a format change — it uses `sections`
as the list it already is, the way a BSS region always has.

Two consequences that bite later:

- **`main` keeps section id 0.** Numbering the reserved section 0 reads better
  in isolation but renumbers the one section every other sidecar has, changing
  the output of every flat assemble on every dialect to accommodate a case
  almost no source hits. A sidecar with no leading gap is byte-identical to
  what shipped before; the reserved section takes the next free id.
- **One section does not mean one contiguous image.** A `reserved` section has a
  `base` and no bytes, so anything mapping sections to file offsets by
  accumulating lengths must skip it. Already true of the ca65 and vasm linked
  paths; now true of the flat path, where a consumer is likeliest to have
  assumed otherwise.

`AssemblyResult::reserved_prefix` carries the boundary to the emission layer,
omitted from the JSON when zero.

## Coverage and accepted gaps

The corpus covers: z80-spectrum (flat engine + entry symbol), 6502-c64 (acme),
6502-nes (ca65 linker, multi-segment, non-CPU sections), 68000-amiga (vasm
hunks, relocatable), 65816 (24-bit constant), cp1610-intellivision (the one
word-addressed family — decle units), the hand-authored spectrum128-banked shape
fixture, and two multi-file families (z80-spectrum-multifile: flat include +
incbin; 6502-nes-multifile: ca65-linked included CHR data) pinning per-file line
records and ordered `sources`.

`Header.sources` is populated in the producer's `FileId` order — `sources[0]` is
the root input, one entry per included file in first-inclusion order — so one id
space spans the core contract and the sidecar (KTD2). An `incbin` payload is one
`line` record covering the whole payload at the directive's position; binary
assets never appear in `sources`.

**Accepted gap:** the asl-syntax flat chips (8080, 6800, 1802, 8048, SC/MP, F8,
2650, TMS7000, PDP-11, TMS9900, Z8000) share the flat engine's single capture
path with the covered Z80/6502 families and introduce no new record shape. They
are accepted as a class, gaining fixtures incrementally as consumers reach them.

## Document the contract where the consumer reads it

The banked contract lived on the spec page and in a test, but not on `BaseMap`
or its three lookups. The first consumer read the crate, not the page, and got
it wrong. Rustdoc on the type a consumer touches is part of the contract, not a
courtesy.

## Drift triggers

Re-consult this record before:

- adding, renaming, or removing any field or record type in
  `crates/debug198x/src/lib.rs` — spec page and fixtures move in the same
  change. Additive is still free; a breaking shape change needs a new decision
  here, a `format_version` bump, and a migration path. **"It is still 0.1, so it
  is still draft" is the misreading to watch for** — the string is the frozen
  v1 spec's version;
- **bumping `FORMAT_VERSION`** — deliberately unchanged at the freeze. Moving it
  tells every reader a shape they know has changed *and* breaks the Emu198x
  importer's exact-match check. Its own decision, with the consumer coordinated
  first;
- **adding a variant to any `#[serde(untagged)]` enum** — not covered by the
  additive promise, and a hard parse error for readers that predate it;
- making a consumer depend on a field the spec marks informational
  (`tool`/`tool_version`);
- emitting a `space` qualifier from a new machine — the paged shape is
  specified; population must match actual hardware placement, and the Emu198x
  paging cross-check is the arbiter for the Spectrum 128;
- treating the `--sym`/`--listing` text renderings as part of this format — they
  are CLI conveniences over the record and carry no stability promise.
