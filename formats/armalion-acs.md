# `ACS` — Armalion's compiled scripts, next to their source

**Status:** Read
**Purpose:** The one place in any Sacred build where the same script exists as
both text and bytecode, and what that pair proves.

Reader: `tools/formats/armascript.py`. Files:
`builds/prerelease/Armalion Build - 20-09-2001/…/Scripts/`.

## Why this matters

Every other script finding in this project is an argument from the
interpreter. The 2001-09-20 Armalion prerelease shipped `Scripts/SCRIPT.PAK`
— the compiled output — **in the same directory as the `script0N.txt` files
it was compiled from**. That is a Rosetta stone, and it is checkable.

It is not, however, a key to retail. See *What does not transfer* below.

## The script language

Sacred's scripts were written in a C-like language with **inline VM assembly**:

```c
init ()
{
  item_createOnPatch ("TYPE_NPC_SOLDIER_black_leather", 2802, 2834, 0);
  asm rsp; creature_setDialog (0, "npcDialog008");
  asm rsp; asm movi 1, 4; creature_setLevel (-1, -1);
  asm mov 1, 0;
  asm movi 0, 41;
  asm rsp; item_setLookup (-1, -1);
}

spTrigger5 ()
{
  if (!trigger_chkState (135,1))
    if (trigger_chkState(1,1))
    {
      playSfx ("Baum2.mp3",1,0);
      trigger_setState(1,2);
      trigger_resetState(127, 1);
      trigger_setState(127, 2);
      trigger_setState(135, 1);
    }
}
```

Two things follow immediately. The VM has **registers** (`mov`/`movi` take a
register number and a value) and a **stack pointer that the source resets by
hand** (`asm rsp` before most calls) — so arguments are pushed, and the author
was expected to know it. And `-1` is a sentinel argument meaning "the thing
just created", which is why so many calls read `(-1, -1)`.

## The container

`SCRIPT.PAK` is magic **`ACS` version 2** — exactly the `chunk.type==
CHUNK_TYPE_ACS` and `hdr.ver==2` asserts of the debug build, see
[../builds/armalion-source-tree.md](../builds/armalion-source-tree.md). It is
the **ordinary `.pak` blob layout**, and `tools/formats/pak.py` reads it
unmodified: 256-byte header, then a `{u32 flags, u32 offset, u32 size}` index
at `0x100`. 59 entries, one per script function.

```
char[32]  function name, NUL-padded     // "init", "spTrigger5", "npcDialog008"
u32       (not the code length -- spTrigger5 stores 44 in a 212-byte payload)
u32[]     code, to the end of the payload
```

Entry 0 is `invalid_function`, the interpreter's own error stub — the same
string the binary's `findScriptFunction` assert prints.

## The code

A flat `u32` stream in which **`0xff` introduces a call** and the next word is
the script API id, from the same table as
[armalion-script-api.tsv](../builds/armalion-script-api.tsv). `spTrigger5`
decodes line-for-line against its source:

| bytes | id | source |
|---|---|---|
| `ff 3b 87 01 …` | 59 | `trigger_chkState (135,1)` |
| `ff 3b 01 01 …` | 59 | `trigger_chkState (1,1)` |
| `ff 3d 03 01 00 "Baum2.mp3"` | 61 | `playSfx ("Baum2.mp3",1,0)` |
| `ff 39 01 02` | 57 | `trigger_setState(1,2)` |
| `ff 3a 7f 01` | 58 | `trigger_resetState(127, 1)` |
| `ff 39 7f 02` | 57 | `trigger_setState(127, 2)` |
| `ff 39 87 01` | 57 | `trigger_setState(135, 1)` |

Words between calls are the VM's own operations — the branches of the two
`if`s and the `asm` the source writes inline. They are **not decoded here**.

## The check

For every function present in both the container and the text, the ordered
list of API ids from the bytes must contain the ordered list of API names from
the source as a **suffix**. `armascript.py` reports:

> ACS v2, 59 functions; **26 of the 35 string-free functions match their source
> call-for-call**, 9 do not; 23 embed a string and are not checkable without
> the argument encoding, 1 has no source.

Mutating the `0xff` marker drops the count to nothing. That pins the marker
and the id space — and so **confirms `armalion-script-api.tsv`'s ids are
right**, independently of however they were assigned.

Ten of them are pinned *individually*, not just as a sequence: the id landed
where the source names that same function, in a function whose whole call list
aligned.

| id | function | | id | function |
|---|---|---|---|---|
| 15 | `view_setLocked` | | 52 | `creature_setLevel` |
| 35 | `creature_setFacing` | | 57 | `trigger_setState` |
| 39 | `creature_setDialog` | | 58 | `trigger_resetState` |
| 47 | `creature_setAlliance` | | 59 | `trigger_chkState` |
| 51 | `creature_morph` | | 67 | `printTextResource` |

No id maps to two names, which the tool checks and fails on. One id the table
has **no** entry for — 63 — lands where the source writes `trigger_chkState`.
That is a prediction from a gap, not a confirmation, and is reported
separately for exactly that reason.

Three honest limits:

- The check is a **suffix**, not an equality, because every compiled function
  opens with one call no source line asks for (usually id 77, sometimes 59,
  63, 9, 12, 30, 72). What that call is has not been established, so it is
  tolerated rather than explained.
- **It does not pin the header layout.** A suffix comparison ignores leading
  noise, so a 28-byte name field scores the same as 32. The 32-byte field is
  read off the hex dump.
- The 9 failures are all `npcDialog*`, which also exist as localized copies in
  `Scripts/us/` and `Scripts/de/`; the compiled copy need not be the one whose
  text is being compared.

## What does not transfer

Retail **re-encoded the bytecode entirely**. Retail's records are
`u16 opcode, u16 length` with a tagged argument list
([script-bytecode.md](script-bytecode.md)); Armalion's is a flat call stream
with a `0xff` marker. And retail **renumbered**: retail's trigger opcodes are
4, 5 and 39, where Armalion's `trigger_*` are 56-59; Armalion 67 is
`printTextResource` where retail 67 is a variable setter. Every sampled pair
disagrees.

So this gives a **vocabulary** — the 85-odd things a Sacred script can ask the
engine to do — and it constrains what retail's 141 opcodes can be. It does not
name one of them.

## Open

- The argument encoding. A call with an inline string emits an extra word
  before its arguments (`04` for a 4-argument `item_createOnPatch`, `03` for a
  3-argument `playSfx`) where a call without one does not
  (`trigger_setState(1,2)` is `ff 39 01 02` flat). "Argument count, emitted
  only when the VM must be told where the inline string starts" fits all three
  and has not been tested further.
- The leading call every function carries.
- The VM's own opcodes — the branch words between calls, and what `asm rsp`,
  `mov` and `movi` assemble to.

---
Provenance: `tools/formats/armascript.py`, whose ratchet fails below 26
aligned functions; the container read by the unmodified `pak.py`;
`spTrigger5` verified against its source by hand.
