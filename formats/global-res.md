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
later.

**The entry count is `first_offset / 16`, not `(first_offset − 4) / 16`.**
Entry 0's `offset` is four bytes *short* of the index end, and `first − 4` is
not even a multiple of 16 — it leaves a remainder of 12. The index runs to
`first + 4`, giving 369968/16 = **23,123** exactly. The last entry is real:
hash 2147266699, size 26, `Gero Wachholz`. Its `size` field lands on the four
bytes in front of entry 0's text, which is the same "u32 at offset that is not
the size" every payload carries.

`globalres.py`'s `range(4, first, 16)` produced 23,123 by overshooting rather
than by the rule, and the port's first reader used `i + 16 <= first` and lost
the last entry. Both now state `first + 4` explicitly.

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
| 9419 | `KO` (†) | Vampirism |
| 9424 | `KO` (†) | Concentration |
| 9425 | `BA` | Ballistics |
| 9427 | `BL` | Bloodlust |
| 9428 | `ZK` | Weapon Technology |
| 9431 | `HOM` | Hellpower (*Höllenmacht*) |
| 1100 | — | Attack Speed |
| 1107 | — | Regeneration |

Thirteen of thirteen resolve, and every one is a stat name, which is what makes
the hash trustworthy rather than merely plausible — a wrong hash returns
nothing at all, not thirteen coherent labels. († `KO` is the one prefix in that
table serving more than one skill; see the caution below.) The last two independently
confirm a reading taken from the code alone: `SP` was called a *speed* modifier
because `FUN_081f64e8` divides a duration by `1 + pct/100`, and the engine
labels that row **Attack Speed**.

Sweeping the neighbourhoods those ids sit in gives two contiguous blocks that
were not otherwise reachable. **9400-9432 is the whole skill list**, in order:

```
Heavenly Magic, Weapon Lore, Long-handled Weapons, Sword Lore, Axe Lore,
Dual Wielding, Ranged Combat, Agility, Parrying, Constitution, Armor,
Meditation, Blade Combat, Magic Lore, Fire Magic, Water Magic, Earth Magic,
Air Magic, Moon Magic, Vampirism, Trading, Riding, Disarming, Unarmed Combat,
Concentration, Ballistics, Trap Lore, Bloodlust, Weapon Technology,
Two-handed Weapons, Dwarven Lore, Hellpower, Forge Lore
```

with each skill's in-game description at `id + 50`. **1070-1199 is the
character sheet and item tooltips**, and it names the four damage channels
outright -- 1078-1081 are `Physical, Fire, Magic, Poison`, which is the
independent confirmation that `BalCharPD/FD/MD/GD` and `BalanceResPh/Fe/Ma/Gi`
are *physisch/Feuer/Magie/Gift* in that order.

`globalres.py` prints either block; they are not reproduced further here.

One caution the blocks make visible. A caller may reuse one balance triple for
several skills, so an abbreviation resolved from a *single* call site can be
mislabelled: `bal_KO*` serves three skills, not one. Count the call sites
before naming anything.

The whole family→skill map has since been closed properly, and not by
matching names to ids at all: **resource id = skill type + 9399**, stated by
the engine as a literal `add $0x24b7, %eax`, with the skill type indexing
`FUN_081f686a`'s jump table directly. See
[../engine/combat-formulas.md](../engine/combat-formulas.md) § *The family
prefixes*. `KO` turned out to be *Konzentration* — Concentration (9424) —
shared with Vampirism (9419) and Trap Lore (9426). Dwarven Lore is **not**
among them; that was an artefact of reading the switch cases in address order
when the jump table is a permutation.

## Why only numeric names resolve — closed

The by-name namespace holds only numeric names, and this is not an artefact of
retail. The Armalion prerelease ships **`Scripts/us/resource.txt`, the source
`RESOURCE.PAK` was compiled from**, and every resource in it is declared by
integer:

```
#pragma resources 8192
#pragma resource      1, 1,"OK"
#pragma resource     32, 1,"AMAZON"
#pragma resource     36, 1,"BORON PRIESTESS"
```

**376 of 376 declarations carry a numeric id**, none carries a word, the id
range is 1..8191, and the type field is `1` in every one. That is exactly the
376 populated slots of the 8192 recorded above.

So resources were addressed by number from the beginning. Armalion indexed
`RESOURCE.PAK` by that number directly; retail replaced the direct index with
a hash of the number's decimal string. Nothing was ever named in words, so
nothing was lost in the change — there is no missing vocabulary to look for.

Incidentally the file is the Armalion hero list, and it is *Das Schwarze
Auge* to the bone: `AMAZON`, `MAGE`, `ELF`, `WITCH`, `BORON PRIESTESS`,
`PHEX PRIESTESS`, `WARRIOR`, `DRUID`, `BORON PRIEST`, `PHEX THIEF` — Boron and
Phex being DSA deities. Retail's Seraphim and Gladiator replaced all of them.

## Open

**Not every `res:` operand is numeric, and some resolve to nothing.** Of the
1,521 `res:` display-name references the eight `startcode.bin` files carry,
144 are `res:D1Dorf_01` … `res:D1Dorf_18` — 18 word names repeated across all
eight character classes. They resolve under **neither** namespace: not by
slot, because the tail is not an integer, and not by hash, because no such
name is in the file (the substring `Dorf` appears in none of its 23,123
entries). The retail engine cannot resolve them either — its tag handler
`strtol`s the tail, which gives 0, and resource 0 does not exist. They are
dangling references in the shipped data.

That does not contradict the section above: `global.res` still contains only
numeric names. It is the *references* that are not all numeric.

---
Provenance: `tools/formats/globalres.py`, whose self-check runs on every
invocation; the hash and the load site from `sacred_orig` via
`tools/binary/ghidra/`; the absence of `RESOURCE.PAK` verified against two
independent retail installs.
