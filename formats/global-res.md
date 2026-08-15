# `global.res` — the text resource tree

**Status:** Read
**Purpose:** The container behind every piece of text the engine shows, and the two
different ways the same file is addressed.

Reader: `tools/formats/globalres.py`. File: `scripts/<lang>/global.res`,
2.6 MB, 23,123 entries in the English build.

## Layout

```
u32  'SZ\0\0'
                       // index, 16 bytes per entry, starting at +4
u32  name_hash         // the KEY -- see below
u32  offset            // to the payload
u32  0
u32  size              // payload length in bytes
...
                       // payloads: UTF-16LE at offset+4, `size` bytes
```

The `u32` sitting *at* `offset` is not the size; the text starts four bytes
later. Entry 0's `offset` is where the index ends, so the entry count is
`(first_offset − 4) / 16`.

## Two namespaces, one file

Conflating these is the trap, and it cost an iteration.

**By slot.** `res:N` in the script bytecode is the **slot index** — the N'th
index entry. Slot 0 is `Seraphim`, slot 1 `Gladiator`, slot 18101 `Settler`.

**By name.** Everything *inside the engine* reaches the same tree through a
hash of a resource's **name**, which is the entry's first `u32`. A caller
holding a number prints it to decimal first, so resource `9400` is the entry
whose name hashes like the four characters `"9400"` — `Heavenly Magic`.

The two disagree completely. Slot 9400 is a paragraph of quest prose about an
antidote at Faeries Crossing; resource 9400 is `Heavenly Magic`.

## The hash

`FUN_080ae4d2`, verbatim:

```c
uint name_hash(const char *s) {
    uint h = 0;
    for (; *s; s++) h = (h*0x71 + toupper(*s)) % 0x3b9ac9f7;
    return h & 0x7fffffff;
}
```

`0x3b9ac9f7` is 999999991, prime. `toupper` makes lookup case-insensitive.

That final mask is why a caller may pass a **negative** id. `FUN_084c2e06`
branches on the sign: a value with the sign bit set is a key that is *already
hashed* and is used directly after `& 0x7fffffff`; a non-negative one is a
number to be stringified and hashed. One branch, two namespaces.

> **The modulus is not verified by the data.** It is read off the
> disassembly, but `h` only exceeds it at five characters or more, and nothing
> in the shipped file reaches that: no non-numeric name resolves at all, and no
> numeric id above 9999 does either. Mutating the modulus leaves
> `globalres.py`'s self-check green; mutating the *multiplier* breaks it. Said
> here so the green is not read as more than it is.

## Where the tree comes from

`main` builds it at startup, at `0x0809af0d`:

```
sprintf(buf, ".\SCRIPTS\%s\global.res", language);
FUN_080ae09e(0x0890ac48, buf);        // 0x0890ac48 is the tree root
```

`FUN_084c2d6c` is the singleton wrapper every UI caller goes through, and it
is constructed with the literal `".\SCRIPTS\RESOURCE.PAK"`. **No such file
ships.** Neither the LGP Linux install nor the extracted retail Gold Windows
disc has one; both have only `scripts/us/global.res`. It is an Armalion-era
container that retail dropped, and the string is vestigial — the wrapper reads
the tree `main` already loaded from `global.res`.

The Armalion prerelease still has its `Scripts/RESOURCE.PAK`, and it is a
different, simpler format: `"RES" 0x01`, a `u32` slot count of 8192, zeros to
`0x100`, then 8192 × 12-byte `[type, offset, size]` slots **direct-indexed by
resource id** — no hashing — then payloads of `[u32 type][u32 len][u32 0]
[ASCII]`. 376 of the 8192 slots are populated. Its ids stop at 8191 and
retail's run past 9400, so it does not resolve retail's ids; what it does give
is the vocabulary, and it is openly *Das Schwarze Auge* — `COURAGE`,
`INTUITION`, `DEFTNESS`, alongside `Rondrakamm`, `Tuzak knife`, `Boron sichel`
and `Thorwalder shield`.

## Worked example

The character sheet passes a resource id beside each stat family, and those
ids expand the abbreviations that name half the balance table:

| id | family | resolves to |
|---|---|---|
| 9400 | `HM` | Heavenly Magic |
| 9414 | `FM` | Fire Magic |
| 9415 | `WM` | Water Magic |
| 9416 | `EM` | Earth Magic |
| 9417 | `LM` | Air Magic (*Luftmagie*) |
| 9418 | `MM` | Moon Magic |
| 9419 | `KO` | Vampirism |
| 9425 | `BA` | Ballistics |
| 9427 | `BL` | Bloodlust |
| 9428 | `ZK` | Weapon Technology |
| 9431 | `HOM` | Hellpower (*Höllenmacht*) |
| 1100 | — | Attack Speed |
| 1107 | — | Regeneration |

Thirteen of thirteen resolve, and every one is a stat name, which is what makes
the hash trustworthy rather than merely plausible — a wrong hash returns
nothing at all, not thirteen coherent labels. The last two independently
confirm a reading taken from the code alone: `SP` was called a *speed* modifier
because `FUN_081f64e8` divides a duration by `1 + pct/100`, and the engine
labels that row **Attack Speed**.

## Open

- The by-name namespace appears to hold only numeric names. No plain-word name
  tried resolves, so how the non-numeric resources of the Armalion build were
  carried over — or whether they simply were not — is unread.

---
Provenance: `tools/formats/globalres.py`, whose self-check runs on every
invocation; the hash and the load site from `sacred_orig` via
`tools/binary/ghidra/`; the absence of `RESOURCE.PAK` verified against two
independent retail installs.
