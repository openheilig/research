# `creature.pak` — the creature type table

**Status:** Solved
**Purpose:** Every byte of a creature record, and what each one is called.

Reader: `engine/formats/creatures.gd`. Gate: `engine/checks/creature_check.gd`.

A **flat CIF table, not a `Sacred.Pak` container**: the magic passes but the
bytes at `0x100` are header, not an index, so reading it through `Pak` silently
misreads it. 474 records of 86 bytes from offset 256, and
`256 + 474*86 = 41020` = the file length is what fixes the stride.

## The map, transcribed from retail's own writer

Retail compiles this file from `SCRIPTS\CREATURE.TXT` and ships the function
that dumps it back out (`sub_8150646`). Its `fprintf` calls read every field
straight off the record, and its header comments are the documentation:

```
// CLASS: 0=UNK 1=HERO 2=MONSTER 3=NPC
// EXP:  <a>, <b>     exp = A+level*B
// BASE: <STK>, <RES>, <GES>, <REPHY>, <REMAG>, <CHARISMA>
```

| offset | type | field |
|---|---|---|
| `+0x00` | u32 | id — **also** the `items.pak` record naming the Granny model |
| `+0x04` | u8 | CLASS, the 1–15 enum the faction matrix indexes by |
| `+0x06` | u8 | FLAGS — `01` FLY, `02` BIG, `10` NOSHADOW, `20` GHOST, `40` BANANE, `80` KURVE |
| `+0x08` | u16 | EXP a |
| `+0x0a` | u16 | EXP b — awarded `= a + level*b` |
| `+0x0c` | u8×6 | BASE: STK, RES, GES, REPHY, REMAG, CHARISMA |
| `+0x14` | u8×2 | SKILLS |
| `+0x16` | u8×16 | SKILLSX — both index [skill-families.tsv](generated/skill-families.tsv) |
| `+0x26` | u16 | SPEED a |
| `+0x28` | u16 | SPEED b |
| `+0x2a` | 6×{u8,u8} | BONUS pairs — kind, target |
| `+0x36` | u8×6 | bonus value |
| `+0x3c` | u8×6 | bonus CLASS filter (same `CL_` enum) |
| `+0x42` | u8×5 | `Damping:RP` |
| `+0x47` | u8×5 | `Damping:RF` |
| `+0x4c` | u8×5 | `Damping:RM` |
| `+0x51` | u8×5 | `Damping:RG` |

`0x51 + 5 = 0x56 = 86` — every byte accounted for.

## Why this is a reading and not a fit

**The FLAGS byte is the structural proof.** The writer names six bits, and
across all 474 records **zero bits are set outside those six**. A wrong offset
does not produce that; it produces garbage bits. Mutation-testing the offset
from 6 to 7 collapses it to 0 of 474 flagged records.

Ranges corroborate: STK 2–80, GES 5–70, REPHY 1–80; SPEED ordered `a ≤ b` in
**472 of 474**; damping present on only 11–13 records per block.

**Two sources agree.** An outside table (`Creature.pak.txt`,
`SacredModdingStuff1.zip`) had described about 60 of the 86 bytes. This
transcription reproduces every field it names — id, class, flags, the xp pair,
the six attributes, the eighteen skill bytes, the speed pair, the six bonus
pairs and their values — and then accounts for the 26 bytes it left blank:
`+0x3c` is the bonus class filter and `+0x42` onward is the four damping
blocks. Independent agreement on the described part is what licenses the new
part.

That table calls `+0x26`/`+0x28` **walk** and **run**. Retail's writer prints
them as a bare pair and says nothing, so the names are recorded as a second
source rather than applied as a reading.

## A mesh name is not a key

`GHUL.GRN` is named by **two** records, ids 36 and 50, and they are not
duplicates:

| | base | exp | speed |
|---|---|---|---|
| id 36 | 35, 35, 25, 40, 0, 50 | (25, 50) | 50, 50 |
| id 50 | 33, 34, 24, 43, 0, 50 | (52, 48) | 80, 80 |

A reader that looks a creature up by its mesh gets whichever record it meets
first, and which is correct depends on what placed the creature —
`startcode.bin`'s hostile at `monster107` is body id **50**. Same shape as
`rust.bin`'s filename ambiguity and the `FormMeshBone` left/right pairing: the
human-readable label is not the identity. `Sacred.Creatures` is addressed by id
only.

## Open

**HP is not in this file, and neither are the attack and defence ratings** the
to-hit formula consumes. This table carries base *attributes*; retail derives
combat numbers from them through the kernel in
[../engine/combat-formulas.md](../engine/combat-formulas.md)
(`K(S) = (156−BalStatOff)·S/156 + BalStatOff + 9`, `BalStatOff = 20` from
`balance.bin`) plus the balance table. Which attribute becomes which rating is
the missing half.

What each SPEED value governs is unstated by retail; the outside naming
(walk/run) is unconfirmed.

---
Provenance: `sub_8150646` in `install/sacred`; findings log rows 693, 949, 950.
