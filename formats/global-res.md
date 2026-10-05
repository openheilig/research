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

## The symbolic namespace is NOT in this install

**This section previously concluded that the symbolic namespace was absent
from the install. That was wrong, and the cause was our own arithmetic — see
[The hash](#the-hash). Corrected 2026-08-16, findings log row 954.**

Both namespaces resolve, and the same `Res:` prefix carries both: a decimal
payload is a **slot index**, anything else is a **name to hash**.

| operand | resolves |
|---|---|
| NPC names in `startcode.bin`, all eight trees | **1521 of 1521** (was 1377) |
| static `QuestBook` keys, all eight trees | **319 of 1242** |
| `UI_QUICKSAVE`, `INVENTAR_RES_PHYSICAL`, `EWT_Schwert` | yes |

`DQ_BAUER_BEGRUESSUNG_LOG` is *"The peasants need my help."* The quest log
reads in English.

**The 144 `res:D1Dorf_01`..`_18` references are not dangling.** They were
recorded as "references the retail game cannot resolve either". They are a
village roster: `D1Dorf_01` is *Smith*, `_02` *Healer*, `_03` *Bartender*,
`_18` *Witch*, and `_04`..`_17` fourteen *Peasant*s.

### The unresolved remainder is composed, not missing

The 923 `QuestBook` keys that do not resolve are **built at runtime**:

```
DQ_BRINGE_ITEM+Var(DQ_2604)+_LOG
```

Substituting the variable gives a real key. `DQ_BRINGE_ITEM1_LOGTITLE` is
*"The Magic Cure"*, `DQ_BRINGE_ITEM2_LOGTITLE* is *"The Father's Sword."*, and
`..._LOGHEADER` gives the objective line. 61 distinct templates are built this
way. So this is a **VM feature** — the interpreter must substitute before it
looks up — and not a data gap.

> **A resolution rate is hollow until it names which operand it measured.** A
> first pass here reported "3506 of 3506 numeric operands inside `Dialog:`
> procedures resolve" and concluded dialogue was available. It had scraped
> every `res:` operand in the procedure body — and was reporting the text of
> `SetButton res:1024`, a UI button, as the dialogue. The line itself is
> opcode 26, and it is symbolic.

## The hash

`sub_80ACC3E`, verbatim — and **it is 32-bit and it overflows on purpose**:

```c
uint name_hash(const char *s) {
    int v = 0;                                  /* int32, and it WRAPS */
    for (; *s; s++) v = (int32_t)(113*v + toupper(*s)) % 999999991;
    return v & 0x7fffffff;
}
```

Three details carry the whole thing, and getting any of them wrong produces a
hash that works on short names and fails on long ones:

- **`113*v` passes 2^32 and wraps.** `v` can be as large as 999999990, so the
  product reaches ~1.1e11. The first character at which this can happen is the
  **fifth**.
- **The `%` is x86 `idiv`** — C truncated division, so the remainder takes the
  *dividend's* sign and `v` is genuinely negative between iterations.
- **The mask is applied once at the end**, not per iteration, so a negative
  final `v` comes back as `v + 0x80000000`.

`toupper` runs in the C locale on the **sign-extended** byte, making lookup
case-insensitive: `hash("EWT_Schwert") == hash("ewt_schwert") == 444989805`.

That final mask is why a caller may pass a **negative** id. `sub_84C1FD0`
branches on the sign: a value with the sign bit set is a key that is *already
hashed* and is used directly after `& 0x7fffffff`; a non-negative one is a
number to be stringified and hashed. One branch, two namespaces.

### How this was got wrong, and why nothing caught it

The port reimplemented the line above in GDScript, whose ints are **64-bit**.
Nothing wrapped. The two hashes therefore **agree for exactly four characters
and diverge from the fifth**.

Every resource name in the shipped files is a 3–5 digit number. So the numeric
namespace resolved, looked like proof, and the symbolic namespace — every key
of which is longer — missed silently and was written up as absent from the
install. The gate was green throughout because every name it hashed was
`"9400"`-shaped.

An earlier version of this document came within one sentence of the bug:

> *"The modulus is not verified by the data. It is read off the disassembly,
> but `h` only exceeds it at five characters or more, and nothing in the
> shipped file reaches that."*

The five-character boundary was noticed and read as a reason the modulus was
**untested**, when it was also the boundary past which the implementation was
**wrong**.

`checks/resources_check.gd` now carries the mutation control: the naive 64-bit
hash must **agree** with retail's at four characters and **miss the table** at
twelve. Removing the wrap fails the gate instead of quietly restoring the old
conclusion.

## The first u32 is a count, not a magic

There is no `SZ\0\0` signature. The loader (`sub_80AE930`) reads the first
word as the **entry count**, and `23123 == 0x00005A53` — the "magic" was that
number's bytes. A reader that checks for `'SZ'` therefore accepts only a
`.res` holding exactly 23,123 entries. The index is exactly `4 + count*16`
bytes, which is the structural check that replaced it.

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

## The TXT source format, from `SacredFilesRes`, 2026-08-25

A third-party `.res` ↔ `.txt` converter recovered from VK. Its executable
names the conversion in **three stages** and an entry-type enum, neither of
which we had:

    Stage 1. Hash order:      Stage 2. Tabulators:      Stage 3. Text lines:

    CET_SECTION  CET_PARAM
    CET_LISTNAME  CET_LISTDATA  CET_LIST_LIST  CET_LIST_WORD
    CET_WORDNAME  CET_WORDDATA  CET_WORD_WORD  CET_WORD_LIST

Errors it raises: `ERROR_EQUALS_SIGN`, `ERROR_MAX_LENGTH`, `ERROR_TAG_UNKNOWN`,
`ERROR_INCORRECT_NAME`, `ERROR_INCORRECT_ENTRY_TYPE`, `ERROR_NULL_POINTER`.
It refuses any source not in **UTF-16 LE** — "Should be: CP-1200 (UTF-16 LE)" —
which matches the payload encoding recorded under [Layout](#layout).

**Stage 1 being "hash order" is an independent confirmation** that the index is
ordered by the name hash rather than by id or by insertion — arrived at here
by reading the file, and by a different author from a different direction.

> The tool is third-party and its names are its author's reading of the format,
> not Ascaron's. Nothing above was tested by round-tripping a file through it.

**A second, independent tool carries the same enum, 2026-08-25.** Raven Rock's
`srr.dll` holds **10 of 10 `CET_` names and 7 of 7 `ERROR_` names identically**,
and it names `global.res` and builds a `%s%sscripts` path — so it parses the
resource tree at runtime with the same library. Two unrelated Sacred tools
sharing the vocabulary makes it the community's settled reading of the format
rather than one author's private naming. It still is not Ascaron's.

### 50 symbolic names recovered

Its `hash-0.txt` is a name list, and **all 50 resolve against our own
`global.res`**.

> ~~The resolved hashes and retail text were checked in as
> `generated/global-res-symbolic-names.tsv`.~~ **Publication correction,
> 2026-10-05:** that extracted dialogue table crossed the authored-findings
> boundary. It is removed from every published revision, not merely HEAD;
> the original research history remains privately preserved. Read the user's
> own `global.res` at runtime; no replacement dialogue dataset is distributed.

They are one quest chain, `E3Q01`, with `cptHawkwood`, `groomJohn`,
`brideSarah`, `wPriest`, `wHighPriest` and `ranger`, plus the
`E3_Teleporter` resource. Retail's file keeps only the hash, so
these are vocabulary that cannot be enumerated from the container — every
symbolic name recovered has to come from outside it.

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
