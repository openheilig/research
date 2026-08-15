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
| `motions.pak` | 678 010 | `MHP` v1 | fixed 198-byte side table, one per merged motion | none | Partial |
| `creature.pak` | 41 020 | `CIF` v0 | **flat** table: 474 × 86 B | `engine/formats/creatures.gd` | Solved |
| `models03.pak` | 61 752 | `MDL` v3 | overlay — 4 entries | generic | Solved |
| `texture03.pak` | 27 361 | `TEX` v3 | overlay — 3 entries | generic | Solved |
| `texture.tmp` | 2 043 320 | `TXL` v1 | **not a pak** — the texture manager's merged name/offset cache | none | Solved |
| `models.tmp` | 2 755 924 | — | derived cache of the merged `models.pak` + `models03.pak` | none | Solved |
| `*.bmp` (6) | 1.5–2.4 M each | — | plain Windows BMP 1024×768, load/save screens | Godot native | Solved |

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
| `wpmod.bin` | 147 796 | item-modifier table; variable-length records, 5 item-id slots + ~53 numeric fields. Header count 572. | Partial — **no length rule** |
| `world.bin` | 46 264 | sector directory: `u32 3855`, then 3 855 × `(idx, X, Y)`; X,Y all multiples of 64 | Solved |
| `static10_18.bin` | 41 440 | generated remap, magic `map` v0, 2 574 × 16 B; consumed by `cWorld::remapTrigger_load` | Partial |
| `world2.bin` | 32 772 | `u32 32768`, then 32 768 `u8`; 11 822 nonzero, 255 distinct values | **Unknown index space** |
| `balance.bin` | 24 328 | the tunables — flat int32/float32 at fixed absolute offsets | [balance-bin.md](balance-bin.md) |
| `treppe.bin` | 19 952 | sorted `u32 → u32` map, 2 494 pairs, binary-searchable | Partial — **key encoding unsolved** |
| `sets.bin` | 7 396 | 65 item sets, `u32 66` + 66 × 28 int32 | Read |
| `wea.bin` | 4 648 | 256 equipment pools; 906 members, **906/906** are items.pak ids naming a `.GRN` | Read |
| `rust.bin` | 4 392 | armour switch — which mesh an armour becomes per wearer | `engine/formats/armour.gd` |
| `merc.bin` | 1 872 | 117 world placements `(0, x, y, layer)` | Partial |
| `multistart.bin` | 768 | 48 records; three blocks of 16, blocks 0 and 2 byte-identical | Partial |

`sets.bin` field 11 is solved: it equals `(record_index << 8) | member_count`
for 65 of 65 records. Record 0 is entirely `0xcccccccc` (MSVC uninitialised
filler), so 65 of the 66 declared records carry data. Members are coherent
suites — record 6 is the seven Seraphim pieces, records 33–38 the Christmas
set.

`world.bin`'s 3 854 sector coordinates are a **strict subset** of the 6 050 in
`sectors.keyx` — 0 world.bin-only, 2 196 keyx-only. A 100% subset relation on
3 854 entries is not what a wrong stride produces.

> **Earlier belief.** Log row 805 called `world.bin` a dead end. It is right
> that the file contains no text; it is wrong that the file is unstructured.
> Log row 744 discarded `world2.bin` because "every nonzero byte value occurs
> exactly 280 times" — measured, 22 values occur 280×, 139 occur 24×, 93 occur
> 23× and one occurs 187× (summing to 11 822). The advice to leave it alone may
> still be sound; the stated reason is not a fact.

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

`vectoren.bin` section 1 is the procedure table for `funkcode.bin`:
`name[64]` + five `i32`, of which the first two are a byte offset and length
that tile `funkcode.bin` exactly (18 768/18 768 chained, `sum(len)` =
2 774 680 = the file's exact size). See [script-bytecode.md](script-bytecode.md).

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

## Open

- `wpmod.bin`'s record length rule. The used-count sits *after* the id slots
  and the tail varies in 6-int steps, so the 572 declared records cannot be
  enumerated; a heuristic segmenter over-segments to 591.
- `treppe.bin`'s key encoding. The packed `layer*6400*6400 + y*6400 + x`
  reading scores 62.4% of decoded cells inside a real sector against a
  **60.5% analytic random baseline** — chance. Every shift 8…21 and every
  hi/lo split was tested; none beat chance.
- `world2.bin`'s index space. 32 768 entries matches `items.pak` exactly and
  nothing else in the install, but only 4 646 of 11 822 nonzero indices land on
  a named record and group membership is incoherent.
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
