# `.pax` hero saves

**Status:** Read
**Purpose:** How a hero save is framed, and where the character's own numbers
live inside it.

Section framing and the character stream both read;
`tools/parity/pax_diff.py` censuses a section type across the eight-hero corpus, and
`engine/checks/pax_check.gd` gates every section inflating to exactly its declared
size.

## Header

Magic `"AMH"`:

| Value | Build |
|---|---|
| `0x07484D41` | classic |
| `0x1B484D41` | Underworld |

Version word `0x101B` = 2.21 UW = Gold.

## Allocation table

At `0x100`, ten entries of twelve bytes:

```c
struct { DWORD DataType; DWORD Offset; DWORD UnpackedSize; }
```

## Payload framing

Each payload is framed by

```c
struct { DWORD 0xBAADC0DE; DWORD Size; BYTE[24]; }
```

followed by standard zlib — or stored raw when it is not compressed.

Section types observed: `0xC3`, `0xC4`, `0xC7`, `0xC8`, `0xCA`, `0xCB`.

## The `0xC7` character stream (Underworld offsets)

| Offset | Field |
|---|---|
| `0x03DD` | CharacterType |
| `0x03E1` | Experience |
| `0x03F9` | Skill IDs `[8]` |
| `0x0401` | Skill Levels `[8]` |
| `0x041B` | Gold |
| `0x042B` | Level |
| `0x04CB` | CA count |

Field widths, corrected 2026-08-16: the skill ids at `+0x3F9` and skill levels
at `+0x401` are `u8` arrays, not `u32`, and the CA count at `+0x4CB` is a `u16`.

### The combat-art LIST follows the count, and it is the live struct (row 1049)

`+0x4CD` onward, `count` records of **22 bytes** — the same struct retail keeps
in memory at the combat block's `+250` (row 1044), written out field for field.
The template is not a compact description of a starting art; it is a **snapshot
of the art already installed**.

| off | type | field |
|---|---|---|
| `+0x00` | u32 | kind — 1 spell, 2 combat art |
| `+0x04` | u16 | art id |
| `+0x06` | u8 | permanent level (runes) |
| `+0x07` | u8 | temporary level (items) |
| `+0x08` | u16 | flags, bit 0 = known |
| `+0x0A` | f32 | total regeneration, seconds |
| `+0x0E` | f32 | the per-art multiplier — **1.0** in all nineteen |
| `+0x12` | f32 | remaining — **0.0** in all nineteen: a new hero's arts are ready |

The eight templates carry **19 arts** between them: two each, except two
classes with three and four.

**Thirteen of the nineteen agree with the coefficient table to the last bit**
(`base + level*step`, row 1048); the other six are spells, whose curve lives in
a different record, or one of five that read as the table's value divided by
**exactly 1.12**.

> ⚠️ **The 1.12 is per ART, not per hero, and is unexplained.** `hero07`
> carries two arts that match and two that are off by it; `hero00`'s two are
> both off; `hero06` has one of each. So it is not an attribute, a class or a
> difficulty — those would move every art on a character together. The port
> uses the **saved** number and asserts the ratio is exactly 1.12 wherever it
> disagrees, so a misread record fails the gate instead of passing as another
> instance of this.


## The `0xC8` ids are `items.pak` record indices

Settled 2026-08-16 with the control arm `engine/probes/pax_c8.gd` was built
for. Over the union of 93 distinct ids from all eight `templates/*.ptx` and
both `save/*.pax` in the retail install:

| Reading | Resolves | Control (random ids, same range) |
|---|---|---|
| `items.pak` record index | 79/93 = **85%** | 34% |
| `global.res` numeric id | 60/93 = 65% | 63% — chance |

Per save: `hero06.pax` 39/41 = 95% (control 33%), `hero07.pax` 39/42 = 93%
(control 34%).

Where the two readings disagree, `items.pak` is the right one. Ids 1224, 1227
and 1230 in `hero07.pax` are `Daemonia_Armor01_Shoes.grn`,
`Daemonia_Armor02_Belt.grn` and `Daemonia_Armor02_Helm.grn` — three distinct
items that the `global.res` reading collapses into three copies of the generic
word "Shield". Template ids 4007 and 4028 both read "Hair" under `global.res`
but `SeraHair01.grn` and `vlady_d_hair.grn` under `items.pak`.

One genuine mixed case: ids 7900–7909, the class weapons, sit on `items.pak`
records with empty names and resolve only in `global.res`.

## Note on the corpus

The eight-hero `.pax` corpus is third-party sample data, not part of the
retail install and not shipped in any of these repositories. The probes read
it from `$SACRED_CHARS`.

The retail install *does* ship save files of its own — `save/hero06.pax`,
`save/hero07.pax` (two level-29 heroes) and the eight `templates/hero0N.ptx`,
which are read in full above. The six section types this document lists are scoped
to the external corpus; the retail files carry eight, so they extend the list
rather than contradict it.

## `templates/hero0N.ptx` are the new-game characters

Reader: `engine/formats/hero.gd`. Gate: `engine/checks/pax_check.gd`.

The eight templates are `.pax` files in all but extension, and retail **never
parses them**: `cUI_Character::executeAction` copies the chosen one 512 bytes
at a time into `Save\HeroNN.pax` (the `fread`/`fwrite` loop at `0x853FA45`).
A template *is* a savegame, so reading it with `Sacred.Pax` is correct rather
than merely convenient.

Every class starts **level 1, 5000 gold, zero experience, exactly two skills at
level 1**:

| type | class | attributes (STK RES GES REPHY REMAG CHA) | skills |
|---|---|---|---|
| 1 | Seraphim | 22 19 25 22 22 17 | Magic Lore, Weapon Lore |
| 2 | Gladiator | 33 19 20 25 **0** 11 | Weapon Lore, Concentration |
| 3 | Battle Mage | 16 15 20 14 **30** 10 | Magic Lore, Meditation |
| 4 | Dark Elf | 26 18 26 20 0 16 | Weapon Lore, Concentration |
| 5 | Wood Elf | 13 13 **29** 21 24 25 | **Agility**, Weapon Lore |
| 6 | Vampiress | 26 22 21 22 0 21 | Weapon Lore, **Vampirism** |
| 8 | Dwarf | 26 18 25 24 0 **8** | Weapon Lore, Constitution |
| 9 | Daemon | 35 21 22 18 28 15 | Magic Lore, Weapon Lore |

Six `u16` attributes at `+0x3E5`, in `creature.pak`'s own order, duplicated
byte-identically at `+0x41F` — presumably base and current, equal because a
level-1 character's starting gear has not moved them. Nothing recovered says
so, so both are exposed and neither is named "current".

### `CharacterType` is a one-based class index

Established **twice, independently**. `global.res` slot `N-1` names it, and the
executable's own `GetTypeName` (`sub_815B3A2`, over 5624 68-byte records at
`0x8735AC0`) gives the same order — and it is that table which builds the
`bin/type_npc_*` directory name at runtime, since no lowercase tree name exists
as a string in the binary.

```
1 SERAPHIM   2 GLADIATOR  3 MAGICIAN  4 DARKELVE  5 ELVE
6 VAMPIRELADY  7 VAMPN_DO_NOT_USE  8 ZWERG  9 DAEMONIN
```

**Type 7 never appears** because `sub_8265BF6` opens with
`if (charType == 7) charType = 6;`.

The identifications are corroborated by the *characters*, not by the ordering:
the Vampiress is the only class with Vampirism, the Gladiator has the highest
STK and REMAG exactly 0, the Battle Mage the highest REMAG, the Wood Elf the
highest GES *and* the Agility skill, the Dwarf the lowest CHARISMA. Each is
asserted in the gate.

## Open


Nothing open.

---
Provenance: findings log rows tagged `pax-hero`; `tools/parity/pax_diff.py`;
`engine/checks/pax_check.gd`.

