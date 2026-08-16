# Script bytecode — `StartCode.bin` / `FunkCode.bin`

**Status:** Read
**Purpose:** The record framing and argument encoding of the script bytecode,
taken from the interpreter rather than fitted to the data.

Reader:
`tools/formats/startcode.py`. Opcode tables: [script-opcodes.md](generated/script-opcodes.md)
(static) and [script-opcodes-behaviour.md](generated/script-opcodes-behaviour.md)
(behavioural).

Files live at `bin/TYPE_NPC_*/{Start,Funk}Code.bin`.

## Record framing

```
u16 OPCODE
u16 LENGTH      // counts these four bytes
<tagged argument list>
```

Confirmed by the engine itself, which reads the length as
`movsx eax, WORD PTR [ebx+0x2]` at `0x0826af24`.

## The opcode table

The jump table is at `0x086f4298`, **141 entries**. The dispatcher is
`0x0826e8ce`; its default case `0x082720dd` is `mov ecx,1; ret` — **an opcode
with no case is silently ignored.**

`script-opcodes.md` carries the static side: handler address, `.rodata`
literals inside each handler, RTTI classes, and a name wherever the strings
say what the handler does. `script-opcodes-behaviour.md` carries the other
side — the most common operand tag signature and the share of records using
it (1.0 means the opcode has exactly one shape), plus the English text each
`res:N` operand resolves to in `global.res`.

## The argument tag table

The tag byte dispatches through a **162-entry** jump table at `0x086f3fbc`
into **52 handlers**. Each handler's cursor advance was recovered by
following control flow from its entry to the shared epilogue
(`tools/binary/tagwidths.py`); width = advance − 1, the one byte being the tag.

Three rules complete it, and each cost several failed attempts:

1. **66 tags fall through to the default handler** at `0x0826e0bc`, which is
   just `inc [esi]; jmp epilogue`. So every unlisted tag below `0xa2` is a
   **zero-width marker**. Omitting these is what made an earlier
   engine-derived table score *worse* than a fitted one.
2. A tag `>= 0xa2` fails the bounds check and exits the record.
3. **VARIANT** tags read a `u32` and, when it is the sentinel `0xfffffffe`,
   take a NUL-terminated string instead of their fixed payload.

## The fifth tag class, and why the metric was wrong

> A **NUMSTR** class was added after "parse rate" was found to be hollow.
> Tags `0x48` `0x49` `0x4a` `0x5d` `0x5e` `0x6d` `0x6e` (handler
> `0x0826cb0e`) and `0x7a` (handler `0x0826de28`) carry a `u32` **and** a
> NUL-terminated string. A control-flow walk cannot see that — it reports the
> `strcpy` and stops — so they were typed STR, the `u32`'s own bytes were
> eaten as the string, and the rest of the record parsed from the wrong
> offset. **Nothing failed**, because every unlisted tag is a zero-width
> no-op, so the garbage was absorbed silently.
>
> The honest metric is therefore not "parses" but **consumes its body
> exactly**, or stops on an END tag.

## `vectoren.bin`

`FunkCode`'s symbol table. Decoded: 5684 sectors, 0 orphans.

## Related

`builds/armalion-script-api.tsv` holds the script API surface extracted from
the Armalion debug build — a second, independent view of the same interpreter.
[`builds/armalion-source-tree.md`](../builds/armalion-source-tree.md) adds that
build's own names for the VM (`cScriptCompiler`, `cScriptInterpreter`,
`cScriptVM`, `cScriptLoader`, in `armaSource/scripts/`) and its invariants:
`fp->code[ip]<256` states that an opcode is a byte indexing a byte array,
`serialID<exportFunctions.size()` that the API dispatches by index into a
vector rather than a fixed switch, and `chunk.type==CHUNK_TYPE_ACS` with
`hdr.ver==2` name the compiled container.

**The Armalion ids do not transfer.** Armalion 67 is `printTextResource` while
retail opcode 67 is a variable setter, and every sampled pair disagrees the
same way. That table is a *vocabulary* for naming retail opcodes, not a key to
them; a join needs a shared invariant, not a shared index.

## The `res:` tag

`0x7a` is NUMSTR like its seven siblings -- `u32` then NUL-string, advance
`len+6`, arithmetic identical -- but its handler `0x0826de28` is a *separate*
case and its tail is a different thing. Where the siblings test the stored
`u32` for a negative sentinel and take a second string, `0x7a` does not branch
on the `u32` at all. It `strncasecmp`s the string it just wrote against
`"res:"` (`0x086f3d65`, 4 chars) and, on a match, copies the remainder and
`strtol`s it base 10. **`0x7a` is where a `res:N` operand becomes a resource
id**, and the resolution happens in the tag handler, not in the opcode.

The corpus cannot see this: **0 of 2053** `0x7a` operands carry a negative
`u32`, against the siblings' 22. Both readings consume identical bytes on every
shipped record, so the distinction is read off the interpreter. Applying it
changes no parse today and is still the rule the engine implements.

Read twice, in two builds: `sacred_orig` at `0x0826de28` and
`sacred-1.0.02-final` at `0x0826cf35`, whose `.rodata` sits `0x6ca0` lower. The
two disassemble instruction-for-instruction alike.

## What quest logic actually uses — 21 opcodes, 98%

The 141-entry dispatch table is the format's size; it is not the size of the
job. Walking every hook of every quest in the `gladiator` tree — 603 quests,
five hook slots each, resolved through `vectoren.bin` section 1 into
`funkcode.bin` — gives **475,345 bytes over 14,017 records**, and those records
use **53 distinct opcodes, of which 21 cover 98.1%**:

| | | |
|---|---|---|
| 53 (26.3%) | 67 (12.0%) | 63 (11.0%) |
| 3 (8.2%) | 23 (6.3%) | 76 (5.0%) |
| 1 (3.9%) | 86 (3.5%) | 58 (3.3%) |
| 22 (3.2%) | 59 (3.2%) | 66 (2.1%) |
| 132 (1.5%) | 62 (1.5%) | 5 (1.4%) |
| 46 (1.3%) | 68 (1.2%) | 64 (1.1%) |
| 4 (0.9%) | 75 (0.8%) | 8 (0.4%) |

Nine already carry handler-derived names — `CreateNPC` 1, `CreateOBJ` 8,
`Teleport` 46, `IsNotVarBit` 58, `QuestkompassPos` 63, `QuestkompassOBJ` 64,
`atmo_rg` 67/75/76 — and opcode 23 is the named-position declaration already
read by `engine/formats/startcode.gd`.

**The tail is safe to ignore, and that is a property of retail rather than a
concession.** The dispatcher's default case is `mov ecx,1; ret`: an opcode with
no case is skipped by its length field and does nothing. An interpreter that
implements the head of this distribution and length-skips the rest behaves the
way the shipped engine behaves on its own unhandled opcodes.

Quest 1101, the one every shipped `questcode.bin` seeds, walks out as 19
records of `OnEnter` (ten `CreateNPC` priests at `pos_Priester1..10`, a ritual
object, a victim at `pos_Steinkreis`, compass to `Res:17112`, three log lines,
a timer, a flag), 3 of `OnExit` and 1 of `OnLose`, with a zero-byte `Trigger`
— so entry is driven from outside the quest. Every operand is either a literal
string in the record or a `res:` id resolving through `global.res`; none of it
is fitted. See findings log rows 936–937.

## Open

The FORMAT is closed; the SEMANTICS are not. **96** opcodes have no verified
meaning beyond what their string payloads suggest, and the 66 zero-width tags
are presumably operators whose identity sits in handlers already located.

Six were named this round from strings reachable *within two calls* of the
handler rather than inside it — 17 award experience, 20 quest-in-sector
trigger, 25 `CreateDynamicQuest`, 120 play movie, 121 `AUTOSAVE`, 126 NPC
refusal speech — each requiring that the string be reachable from exactly one
opcode **and** that `opsem.py`'s behavioural profile agree. They are marked
`[reached]` in the generated table to keep them distinguishable from the 34
named by a handler's own strings.

That two-filter rule is what makes the tier usable. Opcode 79 uniquely reaches
`cCreature::equipment_reset() EquipmentRef unknown?!` and would have been
named "reset equipment" on the strings alone — but its operands are
`(res:TEXT, small id)` carrying quest prose, so the string belongs to
something deeper. It is left unnamed.

The remaining lever for the rest is the decompiler, which has not been pointed
at these handlers. Harvesting *named callees* instead of strings does not work
here: the handlers reach C++ objects almost entirely through vtable dispatch,
so 115 of 141 reach no named function at all and the 26 that do nearly all
reach the same shared pair.

---
Provenance: `tools/formats/startcode.py`, `tagwidths.py`, `opcodes.py`, `opsem.py` module
docstrings; findings log rows 718-723.

