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

## Open

The FORMAT is closed; the SEMANTICS are not. 102 opcodes have no verified
meaning beyond what their string payloads suggest, and the 66 zero-width tags
are presumably operators whose identity sits in handlers already located. One
handler is unread by name: `0x0826de28`, serving tag `0x7a`, may or may not
carry the negative-sentinel second string that its sibling does -- no record
in the corpus can decide it.

---
Provenance: `tools/formats/startcode.py`, `tagwidths.py`, `opcodes.py`, `opsem.py` module
docstrings; findings log rows 718-723.

