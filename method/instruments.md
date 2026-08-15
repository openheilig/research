# Instruments — what to reach for, and what each one cost to learn

**Status:** Standing
**Purpose:** Which instrument to reach for, and the trap each one sets.

Techniques that repaid the time spent building them. The rule that decides
when their output counts is in [discipline.md](discipline.md); this document
is only about the tools themselves.

## Capture the game's own API calls

`apitrace` records every GL call the retail process makes, and it composes
with the harness that drives the game, so a question about *how retail draws
something* can be answered by reading what retail asked the driver to do
rather than by matching pixels. The floor overlay's two-texture blend in
[../formats/world-sectors.md](../formats/world-sectors.md) was settled this
way in one capture.

Two practical limits, both met the hard way:

- **A trace is enormous** — roughly 12 GB for a 100-second run to a landmark.
  Dump it, grep the dump into a small index, and delete the trace. A 562k-line
  index of GL state calls is a few MB and answers most later questions without
  re-running anything.
- **The output path must contain no spaces.** The launcher leaves its runner
  variable unquoted, so a path with a space fails in a way that does not name
  itself.

## Use a real disassembler

Work done with raw `objdump` cost time twice over: once on instruction
desync in the middle of a handler, and once on confusing two adjacent state
calls. A headless IDA run does not desync. Use it for anything longer than a
single basic block, and keep `objdump` for the quick look.

Ghidra is not part of this project's toolchain — the `.rep` projects left in
the workspace are dead.

## Finding the code that reads a field

To locate the consumer of a per-corner field, scan `.text` for **four byte
loads at four consecutive displacements inside one 64-byte window**. The
shape is distinctive enough that it found `+0x18`'s only reader immediately.

This does not work for flag bytes. Their consumer usually copies the byte to
a stack slot first, so a scan for `[reg+disp]` misses the site entirely — look
for the copy, not the test.

## Draw the field

When a census will not separate a field's meaning, render it: one colour per
value, over the world, and look at where it is set. Two `WldxEntry` fields
were settled this way after statistics stalled on both. A distribution hides
spatial structure; a picture of the same data does not.

## Related

- [naming-oracle.md](naming-oracle.md) — turning function naming from a
  heuristic into a mechanism.
- [discipline.md](discipline.md) — read before quoting any number these
  instruments produce.

---
Provenance: each entry is a technique this project ran and paid for; the
findings log rows for the roof-cutaway capture, the `+0x18` reader hunt, and
the `+0x1e`/`+0x1f` overlays carry the specific measurements.
