# Install inventory

**Status:** Read
**Purpose:** Every file a *Sacred Gold* (Linux) install ships, what it is, and
whether anything can read it — one table instead of eleven documents.

This is the "what is on the disc" index. Each row points at the document that
owns the format; where no document exists, the row carries the measurement
itself. Audited 2026-08-16 across 928 files in 17 directories, 18 GB.

Sizes are `stat` bytes. Counts are measured, not declared — where a header's
declared count disagrees with the payload, the row says so.

## The short version

Of the install's 18 GB, **1.7 GB in `pak/` plus 320 MB in `world/` is the
game**; the rest is media (`mp3/`, `movie/`), the runtime (`lib/`, `xfonts/`,
`share/`) and duplicate script trees. Nine of the twelve `pak/` files and all
of `world/` are read in production by the port. The largest previously
undocumented body was **audio**, and it is now decoded end to end.

## `pak/` — the asset containers

Every `.pak` is the same container: `char[3] magic; u8 version; u32 count;`
zeros to `0x100`, then a `count`-long index of `{u32 flags; u32 offset; u32
size}`, then payloads. Details and the exceptions: [pak-containers.md](pak-containers.md).

| File | Bytes | Magic | What it holds | Reader | State |
|---|---|---|---|---|---|
| `texture.pak` | 860 648 179 | `TEX` v3 | 25 535 index slots; the entire texture corpus | `engine/formats/texture.gd` | Solved |
| `sound.pak` | 455 621 361 | `SND` v1 | 50 000 slots, **6 598** payloads: 3 194 RIFF/WAVE + 3 404 Ogg Vorbis | `tools/formats/pak.py` (container only) | Solved, unread by the engine |
| `models.pak` | 434 340 164 | `MDL` v3 | 4 993 entries: 1 572 meshes + 3 421 motions, Granny 1.x chunks | `engine/formats/models.gd` | Read |
| `world/floor.pak` | 187 968 064 | `OBJ` v1 | 6 713 136 fixed 16-byte terrain cells | `engine/formats/world.gd` | Solved |
| `world/static.pak` | 78 889 320 | `OBJ` v1 | 1 038 014 uniform 64-byte object placements | `engine/formats/statics.gd` | Solved |
| `mixed.pak` | 14 336 144 | `MIX` v0 | 32 096 sprite slots, 209 956 tiles | `engine/formats/mixed.gd` | Solved |
| `tiles.pak` | 6 850 288 | `ISO` v3 | 90 132 fixed 64-byte tile records | `engine/formats/tiles.gd` | Solved |
| `items.pak` | 4 587 776 | `ITM` v5 | 32 768 slots, **17 408** populated 128-byte object definitions | `engine/formats/items.gd` | Read |
| `items03.pak` | 4 587 776 | `ITM` v5 | overlay — **one** real record (see below) | same | Solved |
| `sndprofiles.pak` | 1 605 888 | `SPF` v1 | 8 192 slots, **175** populated 184-byte sound-selection profiles | none | Solved, unread |
| `weapon.pak` | 1 572 582 | `WPN` v8 | **flat** table (no blob index): 4 883 × 258 B from `0x100` | none | Partial |
| `motions.pak` | 678 010 | `MHP` v1 | 3 423 × 198 B = `u32 id; u16 id; float slot[48]` | none | Solved |
| `creature.pak` | 41 020 | `CIF` v0 | **flat** table: 474 × 86 B | `engine/formats/creatures.gd` | Solved |
| `models03.pak` | 61 752 | `MDL` v3 | overlay — 4 entries | generic | Solved |
| `texture03.pak` | 27 361 | `TEX` v3 | overlay — 3 entries | generic | Solved |
| `texture.tmp` | 2 043 320 | `TXL` v1 | **not a pak** — the texture manager's merged name/offset cache | none | Solved |
| `models.tmp` | 2 755 924 | — | derived cache of the merged `models.pak` + `models03.pak` | none | Solved |
| `*.bmp` (6) | 1.5–2.4 M each | — | plain Windows BMP 1024×768, load/save screens | Godot native | Solved |

### `motions.pak` — animation timing marks

`198 = 4 + 2 + 48×4` exactly, and slot *n* sits at byte `6 + 4n`. The slot
meanings come from retail's own `MOTIONTAG` parser, whose tag list sits in
`.rodata` immediately before the string `PAK\MOTIONS.PAK`:

| Slot | Tag | Byte |
|---|---|---|
| 16 | `Foot l` | +70 |
| 17 | `Foot r` | +74 |
| 18 | `Foot F_l` (front-left) | +78 |
| 19 | `Foot F_r` | +82 |
| 23 | `Speed` | +98 |
| 24+n | `Hit` | +102… |

The foot slots are footfall phases: 406 of 500 `WALK`/`RUN`/`FLY` clips carry
them and **zero** carry `Hit`, against rotated controls that give 68–91 foot
hits and 135–226 spurious `Hit` hits. Slots 18/19 appear only on quadrupeds.
`Damping`, `RF`, `RG` are not float slots — they set a mode enum, and slots
20/21/22 are zero in all 3 423 records.

The in-memory record is `char name[64]; float slot[48]` at a 256-byte stride —
which is exactly `models.tmp`'s motion record, so the on-disk 198 bytes are
that record's tail from +58.

`texture.tmp` and `models.tmp` are **derived**, not source: `models.tmp`'s
header stamps `models.pak`'s exact byte size (`+8` = 434 340 164) and the pair
size (`+24` = 434 401 916), and its `0x100` preamble is a section table whose
`280` and `1 879 636 = 280 + 1574*1194` are the mesh and motion record starts.
Both are removable.

### The `03` files are a beer advertisement

The `03` suffix is not a version, a patch or the Underworld expansion. The
three overlays together ship **one item**:

| Overlay | Real entries |
|---|---|
| `items03.pak` | record **3999**, name `Krombacher.grn` |
| `models03.pak` | `KROMBACHER.GRN`, `BO02_ACTIVATE.GRN`, + `INVALID_MODEL`/`INVALID_MOTION` sentinels |
| `texture03.pak` | `KROMBACHER.TGA`, `CAB.TGA` |

Krombacher is a German brewery. `items03.pak`'s other 32 767 records are empty
apart from a `u16` slot stamp at `+0x76` equal to the record index; base
`items.pak` slot 3999 is empty (flag 0, all-zero) while 4000+ hold the Seraphim
equipment, so the overlay **adds** into a free slot rather than replacing
anything.

> **Earlier belief.** These were read as an Underworld add-on overlay, on the
> strength of being sparse overlays at all. Listing the entry names refutes it:
> nothing in them is expansion content.

## `world/` — terrain and placements

| File | Bytes | What | Reader |
|---|---|---|---|
| `sectors.wldx` | 63 854 322 | `WLD` v5 — 6 050 zlib blobs, one per sector | `engine/formats/world.gd` |
| `sectors.keyx` | 4 646 656 | `WLK` v5 — directory: 256-byte header + 6 050 × 768 B | `engine/formats/world.gd` |
| `static.pak` | 78 889 320 | 1 038 014 × 64 B object placements | `engine/formats/statics.gd` |
| `triggers.pak` | 36 556 | `TRG` v1 — **not a pak**; 1 246 flat trigger records | `tools/formats/parse_trg.py` |

`triggers.pak`'s record is **unaligned-packed**, which is what makes it look
like four `u32` and read wrong:

```
+0x00 u32  trigger id (== record index)
+0x04 u16  kind (16 = live, 0 = dead)
+0x06 u32  placing static.pak record index   <- unaligned
+0x0a u16  always 1
+0x0c u32  always 0
```

The back-link is exact: all 1 246 resolve to a unique static, and the reverse
link (`static.pak +0x17`, an unaligned `u32` equal to self index + 1) resolves
1 246/1 246. Read as a `u16` it appears lossy — 13 apparent collisions — which
is an artifact of truncating a 20-bit field, not a property of the data.

`static.pak +0x1b`: its 1 870 nonzero values all sit on type-0/flags-0 empty
records and target a set of exactly the 1 246 trigger-carrying statics (set
equality), so it is not an unpinned free list.

## `bin/` — rules tables and script trees

### The eleven loose tables

Retail compiled these from German-named `.txt` sources it does not ship; the
pairing is in the executable's `.rodata` (`scripts\Rustungenswitch.txt` →
`bin\Rust.bin`, `scripts\sets.txt` → `bin\sets.bin`, `scripts\waffenmod.txt` →
`bin\wpmod.bin`, `scripts\treppe.txt` → `bin\treppe.bin`, beside a
`compiling %s`).

| File | Bytes | What it is | State |
|---|---|---|---|
| `wpmod.bin` | 147 796 | item-modifier table, 572 records; **length = 54 + 6×int[53]** | Solved — `engine/formats/wpmod.gd` |
| `world.bin` | 46 264 | sector directory: `u32 3855`, then 3 855 × `(idx, X, Y)`; X,Y all multiples of 64 | Solved |
| `static10_18.bin` | 41 440 | generated remap, magic `map` v0, 2 574 × 16 B; consumed by `cWorld::remapTrigger_load` | Partial |
| `world2.bin` | 32 772 | **not a byte table** — `u32` byte-count + 16 384 `u16` sector-presence grid | Solved |
| `balance.bin` | 24 328 | the tunables — flat int32/float32 at fixed absolute offsets | [balance-bin.md](balance-bin.md) |
| `treppe.bin` | 19 952 | staircase footprint → anchor map, 2 494 pairs, 661 staircases | Solved |
| `sets.bin` | 7 396 | 65 item sets, `u32 66` + 66 × 28 int32 | Solved — `engine/formats/sets.gd` |
| `wea.bin` | 4 648 | 256 equipment pools; 906 members, **906/906** are items.pak ids naming a `.GRN` | Solved — `engine/formats/equipment.gd` |
| `rust.bin` | 4 392 | armour switch — which mesh an armour becomes per wearer | `engine/formats/armour.gd` |
| `merc.bin` | 1 872 | 117 **merchant** map icons `(cache, x, y, class)` | Solved |
| `multistart.bin` | 768 | 48 multiplayer start world-positions | Solved |

### `sets.bin`, fully read

```
u32 count = 66, then 66 records of 28 int32 (112 B)
  [0..9]   up to 10 items.pak member ids, zero-padded
  [10]     the SET NAME, as a PRE-HASHED global.res key
  [11]     (record_index << 8) | member_count
  [12..27] always zero
```

Record 0 is entirely `0xcccccccc` (MSVC uninitialised filler), so 65 records
carry data. Field 11 holds for 65 of 65. Field 10 was the last unknown and is
now closed: `global.res` treats a **negative** id as a key that is already
hashed, and read that way all **65 of 65** resolve to English set names —
"Dark Side of Feac", "Uriel's Legacy", "Astrala's Powermonger", "Dream Netting
of the Gods". Meaningful names at 65/65 are not a coincidence a wrong reading
produces. Members are coherent suites: record 6 is the seven Seraphim pieces,
records 33–38 the Christmas set.

### `treppe.bin` — staircase footprints

Each record is a `(key, value)` pair of `u32`, 2 494 of them in ascending key
order, no header. **Both halves are the same 29-bit packed world position**:

```
poskey = (level << 26) | (y << 13) | x        level 0..4, x/y on the 6400x6400 tile grid
```

Every one of the 2 494 keys lands in a sector `sectors.keyx` actually holds —
**2494/2494 = 100.0000%**, against a 61.3% random-uniform control. The table
maps each cell a staircase covers to that staircase's anchor cell; 661 pairs
are identity, so there are 661 staircases. Footprints run 2–27 cells, and 562
of 661 are full rectangles.

> **Earlier belief.** A packed-position decode was tried and refuted at 62.4%
> against a 60.5% baseline — chance. That refutation was right about the
> *divisor*, not the idea: it assumed a 6400 stride, and the engine shifts by
> 13, i.e. a stride of 8192. With the correct shift the same test returns
> 100.0000%. A refuted decode is not the same as a refuted hypothesis.

Caveat worth keeping: no site in `install/sacred` was found that ever *queries*
this map — it is loaded, constructed and destructed, and `sacredserver` has no
`treppe` string at all. Stated as "no lookup site found", not as proof of
absence, so the port impact is unknown.

### `world2.bin` — the sector-presence grid

```
u32  32768        <- a BYTE count, not an entry count
u16  grid[16384]  <- value = grid[(y << 7) + x], a 128-wide power-of-two stride
```

0 means no sector at that grid cell; nonzero is a 1-based ordinal in row-major
scan order. Exactly **6 050** cells are nonzero and their values are the dense
set 1…6050 with no gap and no repeat. Those 6 050 cells are **set-identical to
the 6 050 sector coordinates in `sectors.keyx` — symmetric difference zero**.

The ordinal is *not* a `keyx` row: `world2` is row-major and `keyx` is stored
column-major, so only 3 of 6 050 coincide. Anything treating one as the other
is wrong. For the port the file is redundant — it says nothing that is not
derivable from `keyx` — but it is a free startup oracle.

> **Earlier belief.** Log row 744 read this as 32 768 single bytes, found "255
> distinct values in ascending order", and discarded it as a generated pattern.
> The byte reading was the error: those 255 values are the low/high halves of
> `u16` counters. Row 805's "world2.bin likewise holds only a byte ramp" fails
> for the same reason.

### `wpmod.bin` — the length rule

```
u32 count = 572, then records from int index 1:
  54 int32 fixed part, whose LAST int (int[53]) is a block count 0..5
  then int[53] blocks of 6 int32
  length = 54 + 6 * int[53]
```

That consumes the file exactly, 572 records ending on the last byte. Length
histogram 54×3, 60×241, 66×240, 72×70, 78×11, 84×7 — the earlier heuristic
segmenter reported 591 starts because it missed both the zero-block and
five-block records.

`int[0..4]` are five items.pak ids and `int[5]` is how many are live; slots at
or above the count hold **stale values from the previously written record**,
which is a compiler artifact that corroborates the field rather than
undermining it. The three zero-block records are cosmetics that grant nothing
(`dwarf_goggles.grn`, the two `vlady` hairs, `magician_cowl.grn`) — exactly
what a zero block count should mean.

#### The fields, bound to columns

Read off the compiler, which parses `scripts\waffenmod.txt` and writes this
file. Every tag `strstr`s the source line and `strtol`s what follows into a
fixed stack slot, and the record buffer's base is visible in the code, so the
tag→column map is transcribed rather than fitted.

| Column | Tag | Meaning | Measured |
|---|---|---|---|
| `int[0..4]` | — | five `items.pak` ids | 1508 distinct, all naming a `.GRN` |
| `int[5]` | — | how many of the five are live | 1…5 |
| `int[6]` | `mod:` | modifier magnitude | 8…180, never 0 |
| `int[7]` | `var:` | variance around it | 0…35 |
| `int[8..37]` | `ph: fe: ma: gi: rp: rf: rm: rg: aw: vw:` | **ten channels × 3 values** | see below |
| `int[38]` | `EWT_*` | item-type enum | −1…32 |
| `int[39]` | `MinLev:` | minimum level | 0…90 |
| `int[40]` | `MinRare:` | minimum rarity | 0…15 |
| `int[53]` | — | block count | 0…5 |

The ten triples are the damage/resistance channels in source order:
`ph` physisch, `fe` feuer, `ma` magie, `gi` gift, then `rp rf rm rg` their
resistances, then `aw` and `vw` (the `AW,`/`VW,` pair). Each parses as three
comma-separated integers.

`int[38]`'s 33 values are the `EWT_` equipment-type enum, listed in the
executable: `0 Schwert, 1 Dolch, 2 Degen, 3 Säbel, 4 2HSchwert, 5 Axt,
6 2HAxt, 7 Schild, 8 Bogen, 9 Armbrust, 10 Klingenwaffe, 11 Kettenwaffe,
12 Peitsche, 13 Rüstung, 14 Ring, 15 Amulett, 16 Helm, 17 Armschiene,
18 Beinschiene, 19 Gürtel, 20 Schulter, 21 Speer, 22 Keule, 23 Stab,
24 Magierstab, 25 Zaumzeug, 26 Schuhe, 27 Handschuhe, 28 Flügel, 29 Item,
30 Pistole, 31 Muskete, 32 Rucksack`, with `−1` = `EWT_NichtGut` and the
field left untouched when no `EWT_` tag appears.

#### The 6-int block

| Field | Meaning |
|---|---|
| `blk[0]` | low 16 = chance percent; **bit 31 = RESISTANCE** |
| `blk[1]` | low 16 = id (three namespaces); bits 16–18 = conditioning attribute |
| `blk[2]` | magnitude **range**: `u16 min`, `u16 max` |
| `blk[3]` | group id — **1…11** skill groups, **14…34** class-spell groups |
| `blk[4]` | the `Spell:` section's magnitude, default **20** |
| `blk[5]` | the `Skill:` section's magnitude, default **10** |

`blk[0]`'s low half is a clean percentage ladder — 10, 15, 17, 20, 25, 30, 33,
40, 50, 60, 70, 80, 100 — with 572 of 1010 blocks at 100.

**The block is a tagged union: which slot carries the magnitude depends on
which source section produced it.** That is not inferred from the shape; it is
what the parser does, and the file agrees to within three records:

| Section | Magnitude lives in | `blk[2]` | `blk[4]` | `blk[5]` |
|---|---|---|---|---|
| `Bonus:` (773 blocks) | `blk[2]` | non-zero in **770/773** | all 20 | all 10 |
| `Spell:` (100) | `blk[4]` | **zero in 100/100** | overridden 68× | all 10 |
| `Skill:` (129) | `blk[5]` | **zero in 129/129** | all 20 | overridden 114× |

`blk[4] != 20` occurs in 68 blocks and **all 68 are `Spell:`**; `blk[5] != 10`
occurs in 114 and **all 114 are `Skill:`**. A `Bonus:` block never touches
either.

#### `blk[2]` is a hyphen range, not two fields

The parser `strchr`s a **`'-'`**, `strtol`s the part before it into *both*
halves, and only then overwrites the high half from the part after. So the
source writes `min-max`, and a bare number yields `min == max`.

The file bears that out: `min <= max` in **1010/1010** blocks with zero
violations, and `min == max` in the 242 bare-number cases. Ranges run
`min` 0…60 and `max` 0…90, scaled per bonus — `VW` spans 3…60 / 9…90, `PD`
physical damage 2…30 / 5…50, `GS` 2…9 / 5…19.

The unit is therefore whatever the bonus itself is denominated in; the source
text carries no unit and `scripts\waffenmod.txt` is not shipped.

#### `blk[4]` and `blk[5]` are positional, not tagged

Neither has a tag name. `blk[4]` is the number following the class-spell group
in a `Spell:` line; `blk[5]` is the number following the skill name in a
`Skill:` line. Both fall back to their default when the number is absent, which
is why 942 and 896 blocks carry 20 and 10.

Their observed values are `blk[4]` ∈ {12, 15, 17, 18, 20, 25, 30} and `blk[5]`
∈ {10, 11, 12, 14, 15, 20, 22, 25, 27, 30}. Since the block that uses them
never carries a `blk[2]` range, they occupy the magnitude role for their
section — but what they are denominated in (a level, a rank, a percentage) is
named nowhere in the shipped data.

`blk[1]`'s low 16 bits carry an id from **three disjoint namespaces**, and
every value in the file falls inside them with nothing left over:

| Range | Meaning | Blocks |
|---|---|---|
| `0` | a `Spell:` block — the payload is `blk[3]` | 148 |
| `599 + skill` | a SKILL bonus | 89 |
| `801…820` | a `Bonus:` id | 773 |

**The 6xx band is `599 + skill id`**, written by the `Skill_` tag: it parses
the trailing name, looks it up in the executable's 34-entry `SKILL_*` table
(ids 0…33, index == id), and the helper it passes the result through adds
**599**. The observed 601, 606–611, 620, 624 are therefore
`Waffenkunde, Fernkampf, Wendigkeit, Parade, Konstitution, Rüstung,
Meditation, Handel, Konzentration`.

That offset is confirmed semantically, which is the part a wrong constant could
not survive — each skill lands on exactly the gear that skill governs:

| Skill | The items it modifies |
|---|---|
| `Fernkampf` (ranged) | `Pistol_simple`, `Pistol_multi`, `Pistol_nice`, `muskette_1` |
| `Parade` (parry) | `daem_shield01`, `daem_shield02`, `daem_shield03` |
| `Waffenkunde` (weapon lore) | `Dark_Sword`, `D_Sword_02` |
| `Rüstung` (armour) | `Dwarf_metal_body`, `Dwarf_curious_body` |
| `Handel` (trade) | `Dwarf_AM_Brosche`, `Seraphim_AM_Brosche` |

The `Bonus:` ids are literal constants in the compiler: `PD,`→801, `FD,`→802,
`MD,`→803, `GD,`→804, `BonusP,`→805, `BonusF,`→806, `BonusM,`→807,
`BonusG,`→808, `AW,`→809, `VW,`→810, `ASpeed,`→811, `WSpeed,`→812,
`RegSpell,`→813, `RegMove,`→814.

#### `blk[1]`'s high bits name a conditioning attribute

The six `Bedingung:` tags each write a standalone id **or**, when the block
already carries one, `or` a small selector into bits 16–18:

| Tag | Standalone id | Selector |
|---|---|---|
| `ST,` Stärke | 815 | 1 |
| `GS,` Geschicklichkeit | 816 | 2 |
| `WI,` Wissen | 817 | 3 |
| `RP,` | 818 | 4 |
| `RM,` | 819 | 5 |
| `CH,` Charisma | 820 | 6 |

220 blocks carry a selector, and in **every one** the low half is a `Bonus:` id
in 801…810 — so the pairing reads as "this bonus is conditioned on that
attribute": `ST + 801` is physical damage scaling with Strength, `WI + 802`
fire damage with Wissen.

> **Earlier belief, mine.** I first read the selector as stacking on a *skill*
> id, because the compiler's test is only "something is already set" and the
> `Skill_` tag is parsed just before these. The control refutes it: of 220
> flagged blocks, **220** carry a low half outside the skill range. A skill and
> an attribute never co-occur in the shipped data.

**The resistance tags reuse the damage ids.** `PR, FR, MR, GR` are assigned
801–804 exactly as `PD, FD, MD, GD` are, and then `or`ed with `0x8000` in the
high half — so the same channel id means damage or resistance according to
bit 31 alone. Measured: 151 of 1010 blocks carry bit 31, and their ids are
**only** 801–808, which is precisely the set of tags that come in
damage/resistance pairs. Nothing outside that set ever carries the bit.

`blk[4] = 20` and `blk[5] = 10` are not near-constant by accident: they are the
values the block initialiser writes before any tag is parsed, so they are
defaults and the 942/1010 and 896/1010 majorities are records that never
override them.

> **Earlier belief.** `blk[1]`'s high half was read as "a level or rank 0…6".
> It is a flag field: an attribute tag `or`s `0x10000` into it when a skill id
> is already present, and writes a standalone id otherwise.

> **Earlier belief.** `int[38]` was ruled out as `EWT_` on the grounds that
> `EWT_` has 21 members. It has 33, and they match the measured range exactly.

### `world.bin` is a legacy-savegame remap, not a live index

`world.bin`'s 3 854 sector coordinates are a **strict subset** of the 6 050 in
`sectors.keyx` — 0 world.bin-only, 2 196 keyx-only. The reason is not
statistical: `cWorld::load()` reads the table only when `floor.pak` is larger
than **139 999 999 bytes** (ours is 187 968 064, so it loads), and the
serializer uses it to translate a *saved* sector ordinal into a current one.
`col0` is the sector's id in the **older, smaller** world; `(X, Y)` is where it
sat. The 3 854 are the pre-expansion sectors, a subset of today's 6 050 because
the add-on appended sectors and never moved the old ones — which is also why
the records are in the same raster order `keyx` assigns slots today.

So the file matters only for loading pre-expansion saves.

> **Earlier belief.** Log row 805 called `world.bin` a dead end. It is right
> that the file contains no text; it is wrong that the file is unstructured.
> Log row 744 discarded `world2.bin` because "every nonzero byte value occurs
> exactly 280 times" — measured, 22 values occur 280×, 139 occur 24×, 93 occur
> 23× and one occurs 187× (summing to 11 822). Both are dealt with above.

### `static10_18.bin` — a save-version trigger remap the port must not implement

The name reads as "static 10 → 18", and that is exactly what it is: a
migration table between save version 10 and 18. Its four columns are
`(old static index, new static index, old trigger id, new trigger id)`, and
`cWorld`'s load path uses `col3` as an index into the live trigger array and
writes `col1` into the relocated trigger's placing-static field. The first 8
records carry `0xCCCCCCCC` filler in the last two columns only.

An open reimplementation starting from current saves never needs it. Recorded
so nobody decodes it twice.

> **Earlier belief.** Its `col0`/`col1` were read as packed world positions and
> refuted at 45.5% against a 60.5% baseline. That refutation stands — they are
> `static.pak` indices, not positions.

### `merc.bin` — merchant map icons, not mercenaries

117 records × 16 bytes, and the German gloss was wrong: `merc` is *Merchant*.

```
+0x00 u32  ALWAYS 0 on disk — a runtime cache slot the loader fills in
+0x04 u32  world cell x
+0x08 u32  world cell y
+0x0C u32  merchant class 0..3
```

The class is not a layer. The world-map renderer switches on it to pick an
icon: `0 → MOUSE_TRADER.TGA`, `1 → MOUSE_BLACKSMITH.TGA`, `2 → MOUSE_COMBO.TGA`,
`3 → mouse_horsetrader.tga`. So the measured histogram `{0:46, 1:28, 2:23,
3:20}` reads as 46 traders, 28 blacksmiths, 23 combined shops, 20 horse
traders. The 0…3 range coinciding with startcode's layer field was chance.

The record carries no entity id because it does not need one: after the read,
the loader resolves the object standing at each cell **by position** and caches
it into `+0x00`.

### `multistart.bin` — 48 multiplayer starts, and the unit is pinned

A flat 768-byte blob, `48 × {u16, pad, s32 wx, s32 wy, u8, pad}`. The values
are **world positions**, not cells, and the conversion is
`cell = (int)(wp * 0.0186339)` — truncation, and `1/0.0186339 = 53.665 630 92`,
the project's known tile size.

Both earlier candidate units are refuted. All 96 coordinates are the truncated
*centre* of their cell: `wp − floor((cell+0.5) × 53.66563)` lands in `(−1, 0]`
for 96 of 96, a window 0.0161 wide, against 0.9375 under /32, 0.9219 under /64,
0.9465 for a byte-rotated control and 0.9761 for random-in-range. The earlier
"48/48 land in a real sector" for both /32 and /64 was chance against the
60.5% baseline — the fractional test separates them where the sector test
cannot.

### The twenty script trees

`bin/` holds ten trees plus ten more under `bin/addon/`, each with the same six
files. Grouped by md5 — the table that says what is per-class and what is
shared:

| File | Structure across the 20 trees |
|---|---|
| `questpoolcode.bin` | **0 bytes in all 20** |
| `questcode.bin` | 4 classes: empty (9 addon trees), 15 B (`netscript` ×2), 40 B (7 base classes), 53 B (`gladiator` = `netscriptcamp`) |
| `startcode.bin` | 9 addon trees **byte-identical**; each base class distinct; `base/gladiator` == `base/netscriptcamp` |
| `funkcode.bin` | same shape; `addon/netscript` is 2 774 679 B against `base/netscript`'s 2 774 680 — one byte shorter |
| `vectoren.bin` | 9 addon trees identical; `base/netscript` and `addon/netscript` share a size (1 660 120) but differ |
| `defpos.bin` | **all 20 distinct** except `base/gladiator` == `base/netscriptcamp` and `addon/gladiator` == `addon/netscriptcamp` |

Two facts worth keeping: the expansion **collapses the per-class split** — nine
of ten `addon/` trees are the same bytes — and the single-player campaign tree
`netscriptcamp` **is** the Gladiator tree.

### `questcode.bin` is the initial quest state, not a quest database

Fully decoded; the parse consumes all ten non-empty files exactly.

```
record := u16 tag; u16 total_len; u8 0x01; cstring name; u8 0x0b; u32 0
  tag 0x44  numeric quest id, stored as a STRING ("1101", "31")
  tag 0x43  named quest flag ("DaemonTotFranz")
```

Every tree seeds quest `1101` and flag `DaemonTotFranz`; the Gladiator/campaign
tree adds `31`. **Quests do not live here.** Quest *titles* are in
`vectoren.bin`'s second section; quest *logic* is in `funkcode.bin`.

### `vectoren.bin` — three sections, and this is where quests live

```
SECTION 1  procedure table for funkcode.bin
  u32 count, then count x 84 B from offset 4:
    +0x00 char name[64]
    +0x40 i32 byte offset into funkcode.bin
    +0x44 i32 length
    +0x48 i32 quest id, -1 = none
    +0x4c i32, +0x50 i32
  Record 0 is an ALL-ZERO SENTINEL (offset 0, length 0).

SECTION 2  quest table, at 4 + count*84
  u32 count, then count x 292 B:
    +0x000 u32  quest id
    +0x004 char title[256]   German quest-log title
    +0x104 u32  enum {0,15,28,53}   unknown
    +0x108 u32  enum {0,13,15,25}   unknown
    +0x10c u32  section-1 index -> QIS_Trigger<id>
    +0x110 u32  ...              -> QIS_OnEnter<id>
    +0x114 u32  ...              -> QIS_OnSetUp<id>   (0 = absent)
    +0x118 u32  ...              -> QIS_OnExit<id>
    +0x11c u32  ...              -> QIS_OnLose<id>    (0 = absent)
    +0x120 u32  always 0

SECTION 3  dynamic-quest ("Zufallsquest") region table, VARIABLE length
  Present only in the base type_npc_* trees and netscriptcamp; absent from
  every addon/ tree and from base netscript/. Regions 1..13 and 15..23 —
  Region14 does not exist. In the base trees every symbol it references is
  named `ToDo:-1.<slot>`: the content is placeholder.
```

Section 1 chains exactly: `offset[i] + length[i] == offset[i+1]` for
18 768/18 768 entries in `netscript`, and `sum(len)` = 2 774 680 = `funkcode.bin`'s
exact size. See [script-bytecode.md](script-bytecode.md).

**Section-1 indices are based at offset 4, counting the sentinel.** This is the
one thing to get right, and it is checkable: resolving each quest's
`+0x10c` and asking whether the symbol is literally `QIS_Trigger<that quest id>`
gives **285/285** at base 4 and **0/285** at base 88, where it lands one record
early on `SelfTriggerQuest<id>` every time.

> `engine/formats/funk.gd` uses `VEC_HDR := 88`, and that is **not** a bug for
> what it does — it looks symbols up by funkcode *offset*, and 88 simply skips
> the zero sentinel, which a zero-length record cannot contribute to. But its
> comment calls 88 "u32 count, then padding to the first record", and there is
> no padding: 88 = 4 + 84 is one whole record. Any future consumer that treats
> a vectoren value as an **index** must use base 4.

### `defpos.bin` is a regenerable cache, and 19 of 20 copies are stale

The loader checks a magic first: `u32 == 1234`, then three counted sections of
100-, 80- and 76-byte records. If the magic fails it **closes the file,
rebuilds all three tables from the already-open `startcode.bin` and
`funkcode.bin`, and writes the file back**.

Of the 20 shipped copies, exactly **one** carries the magic —
`bin/type_npc_seraphim/defpos.bin`. The other 19 begin with their own first
section count (975 / 1485 / 1908 / 1910 / 1921 / 1945) and are therefore
rejected on sight and regenerated at load. The same check sits at a
byte-identical instruction in all six shipped clients; `sacredserver` has no
`defpos` loader at all.

That reframes the file: it is **derived**, not a source of truth, so a port has
no obligation to read it — and the one file that differs most likely differs
because this install has only ever been played as a Seraphim, which is an
inference and is flagged as one.

## Audio — decoded end to end, previously undocumented

`sound.pak`'s index flag maps 1:1 to codec across all 6 598 payloads with zero
exceptions: `0x20` ↔ `RIFF` (3 194 — 1 825 plain PCM, 1 369 IMA-ADPCM),
`0x21` ↔ `OggS` (3 404). `max(offset+size)` equals the file size exactly, so
the index is complete and there is no trailer. No proprietary codec appears
anywhere.

The naming is not on disk. The retail ELF carries a **68-byte-record symbol
table** at `0x751c80`–`0x7c3d14`: 6 869 records of `{u32 id; char name[64]}`,
every name prefixed `SOUND_FX_`. It names **all 6 598** blobs with zero
unnamed, and maps 171 of the 173 `mp3/*.ogg` files onto a contiguous id band
6500–6717 — one id space spanning pak blobs and disk streams.

`mp3/` is misnamed: 173 files, all Ogg Vorbis (`OggS` 173/173), stereo
44 100 Hz, nominal 224 kbps, 14 810.96 s total (4 h 07 m).

`sndprofiles.pak` is the selection layer, not a mixer:

```
+0x00 char name[32]
+0x20 u32 present (1 in every populated record)
+0x24 20 bytes, always zero
+0x38 u16 ids[8][8]      8 slots x 8 variant sound ids -> 184
```

All 2 355 distinct ids it references resolve against the `SOUND_FX_` table with
zero unresolvable. Slot meaning is per-profile and legible: profile 102
`footsteps_hero_male` uses slots 1–7 as ground surfaces, profile 500 `sword`
uses slot 0 miss / 1 hit / 2 parade.

The slot enum is also in the ELF — a second 68-byte table at `0x7c3d20`,
immediately after the `SOUND_FX` table plus 8 bytes of padding, holding 49
records: six slot families of 8 restarting at id 0, plus `SND_MAX=8`.

| Family | Slots |
|---|---|
| Creature | APPROACH, ATTACK, NORMALHIT, CRITICALHIT, FOLLOW, FOLLOW_HEROFLEE, FLEE, DEATH |
| Weapon | MISS, HIT, PARADE, SHOOT, DROP, WOOSH, ×2 RESERVED |
| Locomotion | ENVIRONMENT, FOOTSTEP_GRASS, WATER, SNOW, SWAMP, STONE, SAND, WOOD |
| Object | USE, OPEN, KNOCK, CLOSE, ×4 RESERVED |
| Music | PEACE, NEARBY, FIGHT, JINGLE_DANGER, JINGLE_FIGHT_NORMAL/HARD/OVER/EASY |
| Generic | SOUND00…SOUND07 |

> **Correction to `tools/formats/pak.py`.** Its docblock lists `sndprofiles`
> under FIXED with a 196-byte stride. The file has a 12-byte blob index and the
> stride is 184. `pak-containers.md` already had it right.

Two `.ogg` files have **no id in any shipped binary**:
`atmo_village_siege_military.ogg` and `atmospot_waterfall.ogg`. The profile
literally named `atmospot_waterfall` fills all eight entries of slot 0 with id
6507, which resolves to `atmo_village_siege_military_summer`. Retail behaviour
to reproduce is the id, not the name.

## `scripts/`, `templates/`, `save/`

| File | Bytes | What |
|---|---|---|
| `scripts/us/global.res` | 2 760 574 | `SZ` — 23 123 text slots, payloads **UTF-16-LE**, keyed by a **hash** of the name |
| `templates/hero00–07.ptx` | ~19 000 each | new-game starting characters, one per class — PAX format in all but extension |
| `save/game01.pak` | 2 687 832 | the shipped world savegame |
| `save/hero06.pax`, `hero07.pax` | 22 417 / 23 218 | shipped level-29 Dwarf and Daemon heroes |
| `lgp-state/saveindex` | 2 048 | LGP launcher state, encrypted — not game data |
| `scripts/us/.cvsignore` | 10 | shipped by accident |

`global.res` names hash as `h = h*0x71 + toupper(c)` — which is why grepping
the file for a string finds nothing even when the string is present. In
practice only numeric names resolve: 6 247 ids in 0…99 999.

**The `0xC8` record ids in `.ptx`/`.pax` are `items.pak` record indices.** Over
the union of 93 distinct ids from all eight templates and both saves,
`items.pak` resolves 79/93 = 85% against a same-range random control of 34%,
while `global.res` resolves 60/93 = 65% against a control of 63% — chance.
Where they disagree, `items.pak` is right: ids 1224/1227/1230 are
`Daemonia_Armor01_Shoes.grn`, `..02_Belt.grn`, `..02_Helm.grn`, three distinct
items that the `global.res` reading collapses into three copies of the word
"Shield". This closes the open question in [pax-saves.md](pax-saves.md).

### `.pax` section `0xCD` is the explored-map bitmap

The largest section in every hero file by a factor of 35 — 822 824 bytes
inflated, identical in size across both saves and all eight templates, so it is
structural rather than per-hero.

```
u32 0xB00BB00B ; u32 6051 ; 12 zero bytes ; u32 0xDEADB00B
then 6050 records of 136 B:
  +0x00 u32 k+1        (holds for 6050/6050)
  +0x04 u32 0
  +0x08 128 B          32x32 reveal bitmap
24 + 136 * 6050 = 822 824 exactly
```

6 050 is `sectors.keyx`'s exact record count, and record *k* corresponds to
`keyx` record *k*. Sectors are 64×64 tiles and the bitmap is 32×32, so **one
bit per 2×2 tile block**.

Two independent confirmations. Spatially, the shared edge between two
*geometrically adjacent* sectors differs by 1 bit while non-adjacent controls
differ by 21 — revealed blobs cut off at a sector edge and resume in the
neighbour. Semantically, `templates/hero07.ptx`, a character who has never been
played, has exactly **one** revealed sector, against eight in `save/hero07.pax`
and five in `hero06.pax`.

## Media and runtime — closed out

| Path | Count / size | What |
|---|---|---|
| `movie/` | 10 files, 173 MB | MPEG-1/2, all 640×480 @ 25 fps, mp2 audio, 472.8 s total |
| `mp3/` | 173 files, 357 MB | Ogg Vorbis (see Audio) |
| `xfonts/` | 413 files, 7.2 MB | 32-bit X core fonts + `xset`, so the gtk-1.2 security dialog is legible |
| `lib/lib1/` | 53 entries, 39 MB | the bundled runtime — **on the RPATH** |
| `lib/lib2/` | 2 files | `libasound.so.2` — **not reachable**, see below |
| `share/alsa/` | 67 files | ALSA configuration |
| `font/`, `resource/` | 7 files | two TrueType faces + progress-bar bitmaps |
| `map/map.pdf` | 10.3 MB | the printed game map, PDF 1.4 |
| `credits.txt`, `credits2.txt` | 11 489 / 11 463 B | credit-roll scripts, `TYPE=`/`SUB=`/`NEWLINE` directives |
| `settings.cfg`, `gameserver.cfg` | 870 / 69 B | engine tunables |
| `global.txt`, `explog.log` | 0 / 12 B | empty; a closed log stub |

The binary's `RPATH` is `$ORIGIN/lib/lib1:/usr/local/games/sacred/lib/lib1` —
**`lib2/` is on no search path**, yet `libasound.so.2` is a direct `NEEDED` and
exists only there. ALSA therefore resolves from the system and `lib/lib2/` is
an inert staging directory.

`settings.cfg` selects `GFXSTARTUP : LoadingUW00.bmp` and `GFXLOADING :
LoadingUW01.bmp`, so the `uw` asset variants are not alternates — they are what
this Gold install loads.

**The credits keys resolve nowhere.** `credits.txt`/`credits2.txt` reference
389 distinct symbolic keys; 0 of 389 resolve in `global.res`, against a control
where the same loader resolves 23 123 slots, 6 247 numeric ids, and
`9400 → 'Heavenly Magic'`. The keys also appear nowhere in the executable. The
strings are in no shipped file.

## Executables

Six `sacred` builds ship, in two size classes — 9 164 968 (the 1.0.02 family)
and 9 180 988 (1.0.1). All are 32-bit i386 ELF, stripped. Which build answers
which question: [../builds/build-survey.md](../builds/build-survey.md).

`sacred.i64` (101 MB) is an IDA database — **our own artifact**, not shipped
game data. `mssds3d.m3d` and `mssmp3.asi` are Windows **PE32** DLLs (Miles
Sound System) inert in a Linux install.

## `Settings.cfg` — the key set, from three sources, 2026-08-25

Documented nowhere here until now. The union across our install, three VK
copies and the Sacred NL distribution is **156 keys**; our own `Settings.cfg`
carries 46.

**Sixteen keys are named by retail's binary and absent from our config file**,
so they are engine-recognised and merely unset here: `ACCEPT_LICENSE`,
`COMBINE_SLOTS`, `FIRST_LOGIN`, `FONT`, `LADDER_EXPORT`, `NETWORK_CDKEY`,
`NETWORK_CDKEY2`, `NETWORK_LOBBYLOGIN`, `NETWORK_PASSWORD`, `NETWORK_PLAYER`,
`NETWORK_SESSION`, `SCREEN_QUAKE`, `SHOWEXTRO`, `SHOWEXTRO_UW`,
`SHOW_HEROINFO`, `WAITRETRACE`.

`FONT` is the one with structure: it repeats, and takes three arguments —
`FONT : 5, "AntiquaSSK", 16` — so the seven UI font slots are configurable by
face and size, which bears on [ui-taskbar.md](ui-taskbar.md).

The remaining 94 unmatched keys are **mod namespaces, not engine keys**: 90
`NL_*` (Sacred NL — GUI zoom, object lists, keyboard layout, resolution) and
`SR_*` / `WINDOW_*` (the Sacred Resolution mod). None appears in retail's
binary, which is the expected result and is what makes the 16 above credible.

> Presence in the binary is a filter, not a proof — the test is that an
> all-caps token appears both in a real `Settings.cfg` and in the executable.

### What the keys mean — from `sacredtools 3.3`

The config editor `sacredtools.exe` carries an alphabetical Delphi string
table of **56 key names**, and its Delphi control names sit beside them in
form order, so each key can be placed on the tab that edits it. Its bundled
CHM then describes what that tab's controls do. Together they give the first
semantic account of the key set — checked in as
[`generated/settings-cfg-keys.tsv`](generated/settings-cfg-keys.tsv).

| Tab | Keys |
|---|---|
| Graphics | 15, incl. `DETAILLEVEL`, `FSAA_FILTER`, `GFX32`, `NIGHT_DARKNESS`, `FORCE_BLACK_SHADOW` |
| Sound | 6 |
| Gameplay | 12, incl. `WARNING_LEVEL`, `COMBINE_SLOTS`, `UNIQUE_COLOR` |
| Network | 12 |
| Chat | 3 |
| Fonts | `FONT` |
| Underworld | 7 — addon-only: `DEFAULT_SKILLS`, `TASKBAR_ICONS`, `SCREEN_QUAKE`, `FIRST_LOGIN`, `COMPAT_VIDEO`, `WAITRETRACE`, `ACCEPT_LICENSE` |

Three that bear on work in this repository:

- **`FORCE_BLACK_SHADOW` — "disables shadow transparency, giving less
  realistic black shadows; affects performance."** So retail's shadows are
  **alpha-blended by default**, and this key forces them opaque. That answers
  the last of the three questions row 1063 left open about the shadow path.
  See [../engine/game-wiring.md](../engine/game-wiring.md).
- **`TASKBAR_ICONS`** — "show damage-type icons for the weapon in the active
  slot". An addon-era taskbar element; see [ui-taskbar.md](ui-taskbar.md).
- **`LANGUAGE`** — an unsupported value makes **all** in-game text vanish, so
  the value must match a directory under `scripts/`.

`FIRST_LOGIN` is documented as unknown by the tool's own author — its help
page says, verbatim, "Первый вход (Действие не выяснено)", *action not
established*. It is not an omission on our side.

> The tool is third-party and its glosses are its author's, not Ascaron's.
> Only `SOUND3D` is new — every other key it names was already in a config
> file we hold, which makes the 56 a corroboration and not a discovery.

## Two external tables that check out against our own install, 2026-08-25

Both came out of the VK file set. Neither is first-party; both are verified
against data we already hold, and both are checked into
[`generated/`](generated/).

### `combat-art-ids.tsv` — 152 combat arts across nine class sheets

Joins the engine's combat-art id to its symbol, its four `global.res` string
ids and its three UI texture ids.

| Check | Result |
|---|---|
| `nameID` resolves in our `global.res` | **145 / 145** |
| `descShortID` resolves | **145 / 145** |
| `descLongID` resolves | **145 / 145** |
| symbol present in retail's binary | **150 / 152** |

The two symbol misses are the sheet's own `ECM_RESERVED1` / `ECM_RESERVED2`
placeholders for empty slots. **Match the symbol case-insensitively** — the
table uppercases, the binary does not (`ECS_HoelleDisk`, `ECS_Tod`,
`ECS_Tentakel`), and some binary entries carry a trailing comma. The binary
holds 172 `EC[SM]_` symbols in total, so the table covers 150 of them and 22
are uncovered.

Worked row: art 21 `ECS_CRUSADERSTRENGTH`, Heavenly Magic, `nameID` 818 →
"Strength of Faith", `descShortID` 9849 → "Aura which increases the attacking
capabilities of the Seraphim and her comrades.", `uitex` 561/562/563.

### `dialogue-functions.tsv` — 1026 dialogue entry points

Maps a script function name to its function id, its in-head portrait index,
an object id and a quest. **1022 of 1025 names (99.7%) are present in our own
`vectoren.bin`**, taken across all script trees.

Two traps, both of which cost me a wrong answer first:

- `vectoren.bin` stores these **prefixed** — `Dialog:wegweiser_MPStart2` — and
  also carries an `F_` sibling (`wegweiserF_MPStart2`). An exact-match join on
  the bare name returns 3 of 1025 and reads as a total mismatch.
- The three genuine misses are **cp1251 mojibake in the spreadsheet**, not
  missing functions: `DlgBrьckenwдchter51` is `DlgBrückenwächter51`.

The four columns beside the name — `funcID`, `inHeadImage`, `objID`, `quest` —
are not derivable from anything we hold, and are unverified. The same
workbook's first sheet, an `eInHeadImage` enum of `HI_nnn_NPC_DIALOG_nn`
names with texture ids, resolves against **neither** the binary nor
`global.res`; treat that sheet as the author's own naming until something
confirms it.

## Open

- What `wpmod.bin`'s magnitudes are *denominated* in. The structure is closed —
  `blk[2]` is a `min-max` range, `blk[4]`/`blk[5]` are the positional magnitudes
  of the `Spell:`/`Skill:` sections — but no tag names a unit, and
  `scripts\waffenmod.txt` is not shipped.
- Whether `treppe.bin` is live at all: no lookup site was found by
  member-offset search, and `sacredserver` has no `treppe` string.
- `vectoren.bin` section 2's two enums at `+0x104` `{0,15,28,53}` and `+0x108`
  `{0,13,15,25}`.
- Section 3's region content is placeholder (`ToDo:-1.<slot>`) in the base
  trees, so whether the dynamic-quest system shipped functional is unresolved.
- What references a `wea.bin` pool (0…255) or a `sndprofiles` index (0…8 191).
  Neither `items.pak` nor `creature.pak` carries a column that agrees; both are
  most likely script arguments.
- What selects the current music/atmo profile as the player moves. Not in
  `global.res` and not in `bin/*.bin`.
- Where the 389 credits keys resolve.
- `merc.bin` places 117 points with a layer and no id, so what stands there is
  unknown. `multistart.bin`'s sub-cell unit is not pinned: /32 and /64 both
  score 48/48 against the sector test.
- `sets.bin` field 10, and where a set's *bonus* lives — `sets.bin` lists
  members only.
- `static10_18.bin`'s `col0`/`col1` key space. Decoded as a packed position it
  scores 45.5%, **below** the 60.5% baseline, so that reading is refuted.
- The two duplicated sound ids 13024 and 13402, each carrying two names.

---
Provenance: a 12-family audit of 2026-08-16, each family measured by one agent
and then adversarially re-checked by a second; 39 claims were refuted or
corrected at that step. Container framing from `tools/formats/pak.py` and
`engine/formats/pak.gd`; per-format detail from the documents linked above;
log rows 909–916.
