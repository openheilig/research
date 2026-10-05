# Driving RAD's Granny runtime as an oracle

**Status:** Blocked — and the question it existed for was answered elsewhere
**Purpose:** Whether the shipped Granny converter can be made to read Sacred's
`.GRN` dialect, so that its output could serve as ground truth for our own
decoder — and what stopped it.

The idea was to avoid arguing about our `.GRN` decode by asking RAD's own code
what a file contains. The harness only **calls exported functions and reads
their output** — nothing decompiled, nothing derived from RAD's SDK source.
The binaries it drives come from a community archive and live in a scratch
directory; none is stored in any repository here. The harness itself is
[`tools/granny_oracle/`](https://github.com/openheilig/tools/tree/main/granny_oracle/).

## Two corrections worth keeping

> ~~`granny2.dll` is 2.5.0.11, and 2.5 cannot read Granny 1.x `.GRN`.~~ The
> DLL in use is **2.7.0.30**, renamed from `granny2_27.dll`. Version was never
> the obstacle.

> ~~The GRN reader lives in `gr2_viewer.exe`, and calling its exports faults
> because the EXE's CRT and C++ static initializers never run when it is
> mapped as a library, leaving a module global uninitialized.~~ The converter
> is `grn2gr2.dll` itself, whose export resolves and can be called directly —
> no dialog-driving and no EXE-as-library workaround. The fault was real; the
> cause was not. It is `grn2gr2.dll` null-dereferencing partway through
> parsing a genuine Sacred Gold object, as below.

## The crash

Every real-file invocation faults at one instruction:

```
wine: Unhandled page fault on read access to 00000000 at address 100061BF
0x100061bf grn2gr2+0x61bf: cmpl $0xca5e0103, (%eax)
EAX:00000000 EBX:0022fe00 ECX:00000000 EDX:00000000
```

The instruction compares against the object tag `0xCA5E0103` with `EAX = 0` —
the converter dereferences a pointer to a structure it expected to already
hold.

## What was ruled out, in order

Inputs were identity-checked before use — on-disk size plus md5, output
deleted on any mismatch — because an earlier scratch probe had reported a
false success that was really a stale file from the day before, silently
picked up after the converter no-op'd. The same class of error the log records
at rows 581 and 584.

| Hypothesis | Test | Verdict |
|---|---|---|
| A leading pak preamble confuses the parser | Full 186,842-byte pak entry vs a `--bare` slice from the `0xCA5E0000` root tag | **Refuted** — identical fault at the same address |
| The converter needs the model **paired** with its motion, as the community documentation states | Entries 589 (base) and 2845 (motion) in all four orders × bool values | **Refuted** — all 4 fault identically |
| Buffer shape — padding or declared-length truncation | Declared object length at `+0x10` is 185,584, exactly the slice size, already 16-byte aligned | **Not applicable** — nothing to vary |

**The control that makes this a result:** the same call against two
nonexistent paths returns cleanly, without crashing. So the fault is triggered
by real `.GRN` content being parsed, not by an unconditional setup error — yet
its location is completely insensitive to which file, in which order, with
which flag. Six real-file invocations, one instruction.

## Verdict

The most consistent reading is that this converter's parser does not handle a
structural feature of Sacred Gold's `.GRN` dialect at all — not a pairing,
addressing, or framing problem. **This build cannot be driven past that point
regardless of how it is called**, so no `.gr2` was ever produced and the
oracle yielded no skeleton dump and no vertex-count comparison.

The decode in [../formats/granny-grn.md](../formats/granny-grn.md) therefore
rests on the harness and parity route instead, which reached 3413 of 3421
clips without this oracle.

## Open

Nothing. The head-mesh vertex-count discrepancy (279 vs 280) this oracle was
built to settle was **closed on 2026-08-15 by a different instrument entirely**:
an `apitrace` capture of the retail character-select screen contains the
engine's own `glDrawElements` index array for that batch, and it resolves to
279 distinct vertices. See
[../formats/granny-grn.md](../formats/granny-grn.md).

The lesson is worth more than the answer. This harness spent three hypotheses
trying to make RAD's converter read a Sacred `.GRN`; the question fell out of
watching the shipped game draw the mesh, which needed no converter, no SDK and
no clean-room risk. Reach for what the game *does* before what a vendor tool
*might*.

---
Provenance: `tools/granny_oracle/granny_dump_skeleton.c` and `grn_extract.py`,
whose run modes produced every row above; findings log rows 581 and 584 for
the identity-check discipline this investigation adopted.
