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

The compiler's API lookup is **case-insensitive**: the source writes
`dialogMgr_Reset` where the table recovered from the binary's own strings
holds `dialogMgr_reset`, and it still compiled to id 10.

## The container

`SCRIPT.PAK` is magic **`ACS` version 2** — exactly the `chunk.type==
CHUNK_TYPE_ACS` and `hdr.ver==2` asserts of the debug build, see
[../builds/armalion-source-tree.md](../builds/armalion-source-tree.md). It is
the **ordinary `.pak` blob layout**, and `tools/formats/pak.py` reads it
unmodified: 256-byte header, then a `{u32 flags, u32 offset, u32 size}` index
at `0x100`. 59 entries, one per script function.

```
char[32]  function name, NUL-padded     // "init", "spTrigger5", "npcDialog008"
u32       code length IN WORDS
u32[n]    code
```

The length word **is** the code length, in words — an earlier reading of this
file said it was not. `payload_bytes == 4 * length` holds for all 59 entries
and the reader now asserts it. That assertion also **pins the 32-byte name
field**, which used to be read off a hex dump: the identity holds at NAMELEN
32 and scores 0/59 at 24, 28, 36 and 40. The same number is what the debug
build prints as `sid[N] func[…] size[N]` in its `DEBUG.LOG`.

Entry 0 is `invalid_function`, the interpreter's own error stub — the same
string the binary's `findScriptFunction` assert prints.

## The code

A flat `u32` stream.

**`0xff` introduces a call** and the next word is the script API id, from the
same table as [armalion-script-api.tsv](../builds/armalion-script-api.tsv).
`spTrigger5` decodes line-for-line against its source:

| bytes | id | source |
|---|---|---|
| `ff 3b 87 01 …` | 59 | `trigger_chkState (135,1)` |
| `ff 3b 01 01 …` | 59 | `trigger_chkState (1,1)` |
| `ff 3d 03 01 00 "Baum2.mp3"` | 61 | `playSfx ("Baum2.mp3",1,0)` |
| `ff 39 01 02` | 57 | `trigger_setState(1,2)` |
| `ff 3a 7f 01` | 58 | `trigger_resetState(127, 1)` |
| `ff 39 7f 02` | 57 | `trigger_setState(127, 2)` |
| `ff 39 87 01` | 57 | `trigger_setState(135, 1)` |

**`33 0 0 0 <cond> <target>` is a six-word branch.** `target` is a word index
into the same function; the function's own length means "jump to the end", and
no target in the container points outside its function. `cond` is **16 to jump
when the test was false** — a plain `if (x)` — and **17 to jump when it was
true** — `if (!x)`. Nothing else appears. The twin functions `spTrigger20` and
`spTrigger21`, whose sources differ only in two constants, isolate it exactly:

```
  0  ff 3b            CALL 59 trigger_chkState
  2  134  1 / 2       its arguments
  4  33 0 0 0 17 17   if (!…) -> jump-if-true to word 17 = end of function
 10  ff 3f            CALL 63 playSpeechResource
 12  1098 / 1099      its argument
 13  ff 39            CALL 57 trigger_setState
 15  134  1 / 2       its arguments
```

**Inline strings are NUL-terminated and padded to a word**, so the `u32` walk
stays in phase across them — which is why the 23 string-carrying functions are
now checkable at all. The field is the smallest multiple of 4 **strictly
greater than length+1**, so there is always at least one NUL and always a
partial or whole zero word at the end; that holds for all 118 inline strings.

What `33`'s three zero operands select, and what `asm rsp`, `mov` and `movi`
assemble to, are still undecoded.

## The check

`armascript.py` compares, for every function present in both the container and
the text, the ordered list of API ids from the bytes against the ordered list
of call names in the source — as an **equality**, and separately the branch
condition codes against the source's own `!`. It reports:

> ACS v2, 59 functions; **58 match their source call-for-call**, 0 do not,
> 1 has no source; branch condition codes match in **57** functions, disagree
> in 0, 1 not comparable.

The one function without source is `invalid_function`. Every constant in the
decoding is falsifiable and was falsified as a control:

| mutation | 58 call matches becomes | 57 branch matches becomes |
|---|---|---|
| `NAMELEN` 32 → 28 | the header assertion raises | — |
| call marker `0xff` → `0xfe` | **7** | 57 |
| branch opcode 33 → 34 | 58 | **17** |
| `cond` 16/17 swapped | 58 | **17**, with 40 explicit disagreements |
| `script03.txt` included | **55** | 55 |

**There is no compiler preamble.** An earlier version of this document
reported that every compiled function opens with one call no source line asks
for, "usually id 77". That was an artefact of counting only source names the
API table already knew: the table has gaps, so a real leading call to an
untabled name looked like an unexplained extra. Counting every call in the
source removes it, and the comparison is an equality rather than a suffix.

**`script03.txt` is excluded**, and must be. It redefines `init`,
`spTrigger40` and `spTrigger41` with different bodies, and the prerelease's
own `DEBUG.LOG` shows the engine compiled `SCRIPT00`, `SCRIPT01`, `SCRIPT02`
and `SCRIPT04` only. Including it makes those three appear to disagree with a
bytecode that was never compiled from them — which is exactly what the older
`spTrigger40`/`spTrigger41` "mismatch" was.

## What the ids are

**35 ids are pinned to exactly one name**, with no id mapping to two. 26 of
them agree with `armalion-script-api.tsv`, which independently confirms that
table; **9 fill gaps it does not cover**:

| id | function | | id | function |
|---|---|---|---|---|
| 8 | `console_getInput` | | 38 | `creature_setBehaviour` |
| 12 | `engine_setMode` | | 63 | `playSpeechResource` |
| 26 | `item_createOnPatch` | | 64 | `playMusic` |
| 30 | `item_getLookup` | | 72 | `questSetFlag` |
| | | | 77 | `questCheckFlagAnd` |

Id 63 corrects an earlier prediction in this document, which read it as
`trigger_chkState`. That prediction came from the same suffix-alignment
artefact: `playSpeechResource` was missing from the table, so dropping it from
the source list shifted everything after it by one.

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

- **The argument encoding, narrowed but not closed.** When a string is the
  call's **first** argument it is hoisted to the end and an extra word is
  emitted first whose value is the **total argument count**: 30 calls fit,
  e.g. `item_createOnPatch ("TYPE_FX_MAGICMARKER",4022,6173,0)` is
  `ff 26 | 4 | 4022 6173 0 | "TYPE_FX_MAGICMARKER"`, and
  `playSfx ("Baum2.mp3",1,0)` is `ff 3d | 3 | 1 0 | "Baum2.mp3"`. When the
  string is **already last** there is still one extra word, but its value is
  not the argument count and not a constant — `creature_setDialog (0, "…")`
  emits `0 0`, while `creature_onDeath (0, "onDeathSkeleton01")` emits `0 1`.
  Every string-last call in this container has all-zero scalar arguments
  except that one, so there is not enough variation here to decide what the
  word is. `creature_isItemEquiped (1,"OFFIZIERSRÜSTUNG")` in `script04.txt`
  would discriminate, but its string is not present in `SCRIPT.PAK`.
- The VM's own opcodes other than the branch: `33`'s three zero operands, and
  what `asm rsp`, `mov` and `movi` assemble to.

---
Provenance: `tools/formats/armascript.py`, whose ratchets fail below 58
aligned functions and 57 aligned branch sets, and whose header assertion
raises on a wrong name-field size; the container read by the unmodified
`pak.py`; the mutation controls tabled above; `spTrigger5`, `spTrigger20` and
`spTrigger21` verified against their sources by hand.
