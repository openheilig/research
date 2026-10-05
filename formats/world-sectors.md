# World — `sectors.keyx` / `sectors.wldx`

**Status:** Solved
**Purpose:** How the world grid is stored and what every byte of a cell record
means.

The engine loads the grid straight out of the retail install at runtime, and
the 32-byte cell record is fully accounted for.

## Geometry

| Quantity | Value |
|---|---|
| Sector grid | 100 × 100 |
| Tiles per sector | 64 × 64 |
| One tile | 53.66563 world units |

## The two files

`world/sectors.keyx` — magic `WLK` v5, a directory of **6050** `KeyxSector`
records at 768 bytes each. It is the same container family as the `.pak`
files: `(4646656 − 256) / 768 = 6050` exactly.

`world/sectors.wldx` — zlib blobs, each inflating to `WldxEntry[4096]` at
**32 bytes per cell** (4096 = 64 × 64, one sector's tiles).

Directory fields located in the `keyx` record:

| Offset | Field |
|---|---|
| 236 | `u32` byte offset into `sectors.wldx` |
| 240 | `u32` compressed size |
| 264 | `u32` decompressed size |

## `WldxEntry` — all 32 bytes

Every field is accounted for. Where a reading was settled by finding its
consumer in the interpreter rather than by statistics, that is said so.

| Offset | Field |
|---|---|
| `+0x00` | tile id |
| `+0x04` | `static.pak` chain head |
| `+0x08` | runtime mobile-object list head — **zero in all 24,780,800 cells** |
| `+0x0c` | `floor.pak` overlay chain head |
| `+0x10..13` | signed per-corner render heights |
| `+0x14..17` | per-corner light bytes |
| `+0x18..1b` | signed per-corner second height, ×2.5, bilinearly sampled |
| `+0x1c`, `+0x1d` | signed parent-object deltas |
| `+0x1e` | bit 0 structure, bit 1 room interior, bit 2 door; bits 3–7 never set |
| `+0x1f` | low nibble = class; high nibble = a 16-class ground-type tag — values 9 and 10 mark **animated liquid** (see below) |

Three of these are worth stating as negatives, because each cost a round of
hypotheses:

> `+0x08` is the **head reference of an intrusive singly linked NonStatic
> object list** (row 1157), not a pointer, cache or archive ordinal. Armalion
> v4 stores the equivalent field at cell `+0x0c`. `NonStatic.PAK` loader
> `sub_44C090` registers each object's ref in the global `cObjectManager`
> ref-to-pointer table, then inserter `sub_449450` stores the old cell head in
> the new object's `+0x30` next-ref and the new ref in the cell head.
> `sub_449220` removes a node by replacing the head or resolving and splicing
> predecessor `+0x30` links, then clears the removed next-ref. Disk seeds the
> list; runtime load, movement and removal maintain it. Retail v5 copies the
> cell bytes verbatim (`sub_80EF4EE`) but has zero heads in all 24,780,800
> cells and ships no `NonStatic.PAK`, so the complete layer was removed before
> release. Armalion carries 73 non-zero heads, maximum ref 170, within the
> 768-object space implied by `106 * 384 = 40,704` payload bytes divided by
> the debug-reported 53-byte `sObjectNonstatic` record.
>
> ~~`+0x1e` bit 1 marks "covered by a region sub-grid".~~ Refuted.
>
> ~~`+0x18..0x1b` is a per-corner overlay blend.~~ Refuted; it is a second
> height, sampled the same way as `+0x10`.

`+0x1e` and `+0x1f` were both settled by **drawing the field** as a per-cell
colour overlay after statistics stalled on them. A bit whose meaning will not
separate in a census often separates instantly when you can see where in the
world it is set.

## Animated liquid — the `+0x1f` high nibble, rows 669 / 1008–1011

High-nibble values **9** and **10** mark a cell as liquid. This is the same
byte the engine's walkability path blocks movement on, and it forms connected
regions, not speckle — 4096/4096 in the open-sea sectors, coast-shaped
fringes elsewhere. On such a cell retail draws an animated liquid surface
*over* the ordinary ground tile; on the open sea that ground tile is a single
flat image (`ISO00`) repeated across the whole sector, so a port that skips
the liquid pass shows a uniform grey-tan plane where the sea should be.

The engine's material table has **14 records** of 0xD8 bytes each, built by
an unrolled initialiser (`sub_83A6572` in the Linux binary, `FUN_00417f00`
in the Windows one — row 669). Record layout: `+0x00..0xC7` fifty frame
slots, `+0xC8` frame count, `+0xCC` reflective flag, `+0xD0` signed alpha
multiplier, `+0xD4` "hot" flag. The record order is the initialiser's push
order — **the sequence opens with two adjacent `B_WATER` blocks**, which a
first reading (row 1008) missed, sliding every later name one slot; the
frame counts pin the truth, because the pak's only 20-frame sets land on the
records whose `+0xC8` says 20 only under this order (row 1011 retraction):

| idx | material | frames | reflective | alpha | hot |
|---|---|---|---|---|---|
| 0 | `B_WATER` | 50 | 1 | −12 | |
| 1 | `B_WATER` | 50 | 1 | −12 | |
| 2 | `C_WATER` | 50 | 1 | −12 | |
| 3 | `D_WATER` | 50 | 1 | −12 | |
| 4 | `A_LAVA` | 50 | | −255 | 1 |
| 5 | `B_LAVA` | 50 | | −255 | 1 |
| 6 | `C_LAVA` | **20** | | −255 | 1 |
| 7 | `A_SCHWEFEL` | **20** | | −255 | 1 |
| 8 | `D_LAVA` | 50 | | −255 | 1 |
| 9 | `E_WATER` | 50 | | −255 | |
| 10 | `F_WATER` | 50 | 1 | −24 | |
| 11 | `G_WATER` | 50 | 1 | −12 | |
| 12 | `E_LAVA` | 50 | | −255 | 1 |
| 13 | `B_WATER` | 50 | | −12 | |

This is **row 669's original name order restored** — its reflective set
{0,1,2,3,10,11} = "all waters bar E_WATER(9) and B_WATER(13)" reads true
verbatim under it. `B_WATER` fills three slots (0, 1, 13), matching its
three xrefs into the initialiser. Frames live in `texture.pak` as
`<stem>%02d.TGA` from 00, all 128×128 opaque squares, tiling in screen
space.

**Cadence** (row 1011): the draw picks
`frame = ((ms >> 1) & 0x3FF) · count / 1024` — the whole set loops once
every **2048 ms** whatever its length (≈24.4 fps at 50 frames, ≈9.8 at 20).
There is no per-record delay.

**Depth alpha** (row 1011): on liquid cells the signed corner bytes
`+0x10..13` hold **depth**, not render height — the open sea sits at −20,
shallows rise toward 0. Per corner, liquid alpha =
`clamp(alpha_mult · depth, 0, 255)`: negative times negative reads opaque
over deep water and fades to nothing at the shoreline; −255 is opaque at any
depth. The water surface itself is drawn flat — the depth shapes the bed
beneath it.

**Reflection** (row 1012, traced): `+0xCC` (the `REFLECTIVE` flag, true for
ids 0,1,2,3,10,11) gates a **pass-1** quad drawn *before* the bed. Geometry:
the iso diamond vertically mirrored (N↔S screen flip) — a no-op for the
port's flat water surface, so the reflection reuses the bed quad — textured
with the **same animated frame** as the bed (texcoord = the cell's own
screen position over 128 px). Color: a grey tint carried in all three
channels, modulated by the light field (`imul` by `sub_83AD576`'s return),
with **alpha = `clamp(−8 · depth, 0, 255)`** — a *fixed* factor 8,
independent of the record's `+0xD0` alpha multiplier (that multiplier is
only read on the bed path, loop 2). So at equal depth the reflection is
fainter than the bed (−8 vs −12 for water), and where the bed is opaque
deep water it covers the reflection; the reflection only shows through where
the bed's own alpha has faded toward the shoreline. Blend is the ordinary
`GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA`, drawn into the same transparent
pass the bed uses. The port draws it as a second surface per reflective cell
with `render_priority = −1`, so it sorts behind the bed exactly as retail's
pass order dictates.

Which record a cell uses is **per sector**, not per cell, and it lives in
the **keyx record itself** (row 1010). The 768-byte-record loader
(`sub_80EF4EE`) memcpy's record bytes `0x1E9..0x2E8` — a 0x100-byte
"environment" block — into a per-sector arena slice hung off the runtime
sector at `+0x17C`; the draw (`sub_80E3EB2`) then reads block `+0xF7` for
nibble-9 cells and `+0xF8` for nibble-10 cells. So the two ids are keyx
record bytes **736** and **737**. Measured over all 1360 liquid sectors:
every value is in 0..13, the sea reads `B_WATER`, and 22 sectors carry two
*different* liquids at once — which is why the format keeps two bytes. (An
older 512-byte-record loader, `sub_80EF028`, keeps the same block out of
line and seeks it by a stored file offset; the shipped `sectors.keyx` uses
the embedded form.)

## `floor.pak` — the overlay layer

`+0x04` of a `floor.pak` record holds **two** `tiles.pak` indices: the low 17
bits are the art tile, the top 15 the mask tile, and 0 means no mask. Retail
blends them in a single quad across two texture units — confirmed by capturing
the retail process's own GL calls, not inferred from how it looks:

```
unit 1 (mask)  COMBINE_RGB=REPLACE    SRC0_RGB=PREVIOUS
               COMBINE_ALPHA=REPLACE  SRC0_ALPHA=TEXTURE
unit 0 (art)   ENV_MODE=COMBINE       COMBINE_RGB=MODULATE
                                      SRC0_ALPHA=PRIMARY_COLOR
framebuffer    GL_BLEND, glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)
```

So colour is the art tile modulated by the per-corner light, and alpha comes
from the mask tile's texels.

The 17/15 boundary is right for retail. **The argument previously recorded for
it was not.** The gate first asserted only that the top field resolves to a
valid tile whose orientation equals its index mod 18 — neither can fail,
because that identity holds for all 90,132 tiles.pak indices and a doubled
small id stays inside the count. Reading from bit 16 passed. The replacement
argument — that the two fields must partition the word with no shared bit, so
"only 17/15 reconstructs the original u32" — is **a tautology**:
`(v & mask) | ((v >> s) << s) == v` for every `s`, measured 6872 of 6872 at
splits 13, 16, 17, 18 and 20 alike. The companion `over16` test is circular in
the same way: it counts values of the *17-bit read* that exceed 65535, which
is just "bit 16 is sometimes set" and is equally consistent with bit 16
belonging to the mask.

What actually settles it is **which read fills the tile table**:

| read | max art index | leaves unused |
|---|---|---|
| **17 bits** | **90,131** = exactly the last tile | nothing |
| 16 bits | 65,420 — its own ceiling is 65,536 | tiles 65,421–90,131, 27% of the table |

A field that runs to the final entry of its index space and stops is that
index. `floor_check` now asserts the fill ratio, and that is the assertion
that fires when the split is set to 16 — the other three do not.

**The split is build-dependent.** The Armalion prerelease's `tiles.pak` holds
13,402 tiles, and there the 17-bit read reaches 67,071 — impossible — while
the 16-bit read tops out at 13,400. So Armalion packs **16/16** and retail
packs **17/15**: the art field widened by a bit, taken from the mask, when the
tile table outgrew 16 bits. Both files are `OBJ` v1 with the same 16-byte
record, so **the version byte covers the record's size and field offsets, not
the bit packing inside a field.**

Drawing this layer is what moved the port's load-path invariant from 28,672
quads to 35,340.

#### A third corpus settles it: the split follows the tile count, 2026-08-25

`Sacred Plus Internacional 1.8` (a 2023 Raven Rock repack, installed from
`builds/windows-disc/sacred-plus/`) carries a complete **pre-Underworld
`World/`** — 3,982 sectors against retail's 6,050, `(3058432 − 256) / 768`
exactly — and a `tiles.pak` of **47,099** tiles against retail's 90,132. Run
the fill test on all three:

| corpus | tiles.pak | 16/16 max low | 17/15 max low | verdict |
|---|---|---|---|---|
| Armalion 2001 | 13,402 | 13,400 | 67,071 — impossible | **16/16** |
| **Sacred Plus** | **47,099** | **47,098 = the last tile** | 67,071, and 1,628,732 records exceed the table | **16/16** |
| retail | 90,132 | 65,420, leaves 24,711 unused | **90,131 = the last tile** | **17/15** |

Each build's winning read lands on its final tile and stops. So the split is
**not** an era or a version marker: it tracks the tile table crossing 2^16,
which happened when Underworld grew it from 47k to 90k. A reader must take the
split from `tiles.pak`'s count, not from the file version — all three are
`OBJ` v1 with the same 16-byte record.

> `checks/floor_check.gd` hardcodes `LOW17` and `TILE_COUNT := 90132`. That is
> correct for the install this project targets and wrong for any pre-Underworld
> data; it is a pinned assumption, not a general reader.

#### `+0x08` was empty before Underworld too

The same scan over Sacred Plus reads **0 non-zero `+0x08` in all 16,310,272
cells**, matching retail's 0 in 24,780,800. With the Armalion prerelease's 73
populated cells, that dates the drop: the layer was gone by the shipped game
and did not come back. Two further readings hold unchanged on the new corpus —
`+0x1e` never sets a bit above 2 (values 0–7 only), and `+0x1f`'s high nibble
uses all 16 classes. All 3,982 sectors inflate to their declared size, 0 bad.

## The prerelease is not a second corpus for this record

`OBJ v1` is frozen, so the Armalion `Floor.PAK` and `Static.PAK` records are
directly comparable (above). The **sector data is not**. `Sectors.key` is
`WLK` **v4** with a 512-byte directory record against retail's 768, and its
cell layout differs: the tile id sits at `+0x04`, not `+0x00`, with a flag
word at `+0x00`. That is exactly what the version bump predicts.

For anyone who does want to read it: 1,201 sectors, the block offset is the
`u32` at directory-record `+92`, blocks are 131,360 bytes = a 288-byte prefix
followed by 4,096 × 32-byte cells, and the cells are stored **uncompressed** —
retail's zlib came later.

The v4 cell is retail's record **shifted by exactly one word**, with an extra
flag word in front. Three of the four fields are pinned independently, each by
filling its own table exactly and carrying no duplicate link:

| v4 | retail | field | evidence in the prerelease |
|---|---|---|---|
| `+0x00` | — | flag word | only `0` or `0x20000000` |
| `+0x04` | `+0x00` | tile id | max 13,395, and `Tiles.pak` holds 13,402 |
| `+0x08` | `+0x04` | static chain head | 116,832 links, **0 excess**, max 393,764 = `Static.PAK` count − 1 |
| `+0x0c` | `+0x08` | the slot retail zeroes | 73 links, 0 excess, max 170 |
| `+0x10` | `+0x0c` | floor chain head | 1,969,048 links, **0 excess**, max 2,591,135 = `Floor.PAK` count − 1 |

Two fields landing exactly on their table's last record, with not one record
linked twice between them, is what makes the alignment safe — and it is what
makes the `+0x0c` ↔ `+0x08` row an inference worth stating rather than a
guess.

> Two hazards. The prerelease is a debug build and leaked **uninitialised heap
> into its shipped data**: 4,211 of 4.92M cells contain `0xcdcdcdcd`, MSVC's
> debug fill. Anything measured against this corpus has to exclude them or it
> will report nonsense — an unfiltered pass gave a "tile id" of 543,162,368.

## `mixed.pak` — the header anchor must NOT be applied

A placement in `static.pak` stores the sprite's **top-left corner**, and the
sprite's tiles carry `dst` rects in the sprite's own pixel frame. Those two
are complete: the tiles go down at the stored position and nowhere else.

The 16-byte `mixed.pak` header's `i16 dx, dy` at `+0x08` looks like a hotspot
to add, and adding it is wrong. Measured across the corpus it is exactly the
**negation of the sprite's minimum tile `dst` corner**:

| sprite | header `dx,dy` | first tile `dst` |
|---|---|---|
| `Bench 2` | `(0, -7)` | starts at y **7** |
| `Bench 1` | `(0, -36)` | starts at y **36** |
| `MINI_BLUE_4` | `(-4, 0)` | starts at x **4** |
| `KLOSTER_KAPELLE01_*`, `CW_Tree 34(B)` | `(0, 0)` | starts at **0,0** |

So applying it *cancels* the offset the `dst` rects already encode, re-seating
each sprite on its bounding box instead of on the frame it was authored in.
Dropping it moved the port's whole-frame delta against a retail spawn capture
from 23.97% to 14.24% — the largest single correction in that scene.

> **Why it survived.** Every `KLOSTER_KAPELLE` structure piece has a zero
> anchor, so the chapel's walls, floor and pillars lined up perfectly while
> the benches sat 7 pixels high. The error was invisible exactly where it was
> zero, and isolated high-contrast props on open floor were the only place it
> could be seen at all. It also *masked* a second measurement: a sweep of the
> object-sort width threshold showed a flat plateau before the fix and a clear
> minimum after it.

What `dx, dy` is for is unrecovered. Nothing needs it to place a sprite.

## `static.pak` — the chain link is pinned, not cited

A cell's `+0x04` names only the head; the rest hang off `nextStaticId` at
record `+0x1f`, which came from Resacred's `rs_file.h`. Being able to read the
value as an index proves nothing — **six** offsets in the 64-byte record are a
valid index for every record. Two structural properties do settle it, and
`tools/parity/static_next_check.py` measures both:

| | retail | prerelease |
|---|---|---|
| links at `+0x1f`, minus distinct targets | **0** of 174,422 | **0** of 51,224 |
| best rival carrying comparable traffic | `+0x2c`, 30,973 excess of 31,031 | `+0x24`, 7,030 of 7,054 |
| targets that are also cell heads | **0** of 31,752 heads | (v4 cells, not comparable) |

A chain is a linked list, so no record may be the target of two links; and a
linked record is never a head, which the *world cells* say from a different
file entirely. The 64-byte record otherwise decodes identically in both
builds — self-index 393,765/393,765 and 1,038,014/1,038,014, and eight of the
`+0x08` flag values are shared.

> **Count excess, not colliding targets.** "How many targets are hit more than
> once" scores a field that points 418,026 times at a *single* record as one
> collision. `+0x1a` does exactly that and read as the closest rival until the
> measure was fixed to `links − distinct targets`.

## `tiles.pak` — the tile table is a product, and it names its own art

Read fully, and confirmed against the Armalion prerelease's own `tiles.pak`
(`ISO` v3 there too, 13,402 tiles against retail's 90,132):

```
+0x00  char[32]  SOURCE TGA FILENAME, NUL-padded -- "iso00.tga".."iso999.tga"
+0x20  u32       texture.pak id
+0x24  u32       orientation, and it is EXACTLY tile_id % 18
+0x28  u32       0
+0x2c  u32       65536, constant in every record of both builds
+0x30  u32[4]    0
```

**Eighteen consecutive tile ids share one filename and one texture id** —
5008 of 5008 groups in retail, 745 of 745 in the prerelease. So
`tile_id = group * 18 + orientation`, and name, group and texture id are in
bijection: 5008 names, 5008 groups, and not one name on two texture ids. The
only content in a 5.8 MB file is 5008 texture ids and 5008 names.

Two consequences.

The filename was not documented before and is a **naming oracle for terrain
art** — every tile now says which `.tga` it was cut from. `Tiles.source_name()`
returns it.

And `orientation(i) == i % 18` is an **identity by construction**, not a
property of the data. `floor_check` had already measured that it cannot fail
and treated it as an awkward coincidence; it is not a coincidence, and any
test resting on it is permanently vacuous.

`+0x20` and `+0x24` are each the *only* offset in the record that could be
what they are: no other word is ever a plausible texture index, and no other
word ever lies in 0..17. That holds in both builds.

## Walkability

Cell-space walkability is a lookup over Sacred's own region grids, not a
derived collision mesh. The engine implements it in `world/walkable.gd`; the
reference decode is in `tools/parity/verify_ref.py`, and the two are diffed.

Sectors used as the parity sample are deliberately *not* the same list as the
general sector sample — the general list yields zero regions in every entry
(measured, not assumed), which would make the gate unfalsifiable.

## Related

`vectoren.bin` is `FunkCode`'s symbol table — 5684 sectors, 0 orphans. See
[script-bytecode.md](script-bytecode.md).

`Triggers.PAK` is `TRG v1` and not a `.pak`; see
[pak-containers.md](pak-containers.md). `parse_trg.py` reads the Armalion
prerelease's copy unmodified, and the 16-byte record is Armalion's own
`cTrigger`, which its debug build prints as 16 bytes. Across both builds the
first word is the record's **own index** in every populated record
(209/209 and 1246/1246), the fourth is always zero, the second is a varying
high half over a low-half flag that retail fixed at `0x0010`, and the third is
bit 16 over a small code taking 16 distinct values in retail and 6 in the
prerelease. That code is *not* named here: `trigger_setType` exists in the
script API, but the prerelease's one `trigger_setType (1,1)` call refers to a
trigger whose record is **all zeros**, which shows the file holds only
authored triggers and scripts create the rest at runtime — useful in itself,
and not a confirmation of anything.

Building construction — how pieces are placed and linked, and how the
interior/exterior swap works — is deliberately **not** documented here. Its
settled half is entangled with render-architecture decisions about the port,
which are choices rather than facts about the format.

## Open

Nothing open in the 32-byte cell record. Every disk field is accounted for;
retail `+0x08` is the removed NonStatic object-ref chain documented above.

### Corner-floor rendering — selection diagnosis withdrawn

**Correction, 2026-09-06:** the earlier conclusion that the port selects the
wrong overlays is withdrawn. Finding 1022 already matched chain order, art
and mask ids, and uploaded texture hashes against a live retail trace.
Finding 1138 recovered the bit split but did not demonstrate pixel parity;
its upstream-selection explanation contradicts that trace. The discrepancy
remains open. The measurements below are historical layer-localisation
experiments, not proof that texture selection is wrong.

What is bracketed:

| measurement | result |
|---|---|
| null `floor_pak` | repaints **91.7%** of that corner → the pixels ARE overlays |
| `--noobjects` | moves **5.3%** of it → not object sprites |
| `OVERLAY_MAX` 8 → 32 | no change → no chain is truncated |
| overlays off entirely | whole-frame delta **unchanged** at 13.59% |

That last row is the informative one: the port is not failing to draw overlays
there, it is drawing overlays as wrong as drawing none.

Earlier hypothesis tests, retained with their subsequent corrections:

1. **Not the chain-walk restriction.** `_tile_stack` follows the chain only
   while the next link is `self + 1`. Following any in-range link instead
   changes the frame not at all — no chain in this sector jumps.
2. **Reversing all overlay pairs worsened that capture:** the corner went
   from 74.4% differing to 92.2%, MAE 34.2 to 43.4. This does not establish
   equivalence of retail's alternating masked/unmasked passes and the port's
   separate opaque/blended surfaces.
3. ~~**Not a placement offset.**~~ **Withdrawn 2026-09-06 (row 1227):**
   the old whole-crop edge correlation preferred `(0,0)` at only 0.45
   (versus 0.9965 on the control), but did not isolate terrain from the
   other layers. Direct draw inputs prove a 24-pixel cell-origin error.
   Moving terrain alone improves both corners; moving an entire composed
   crop was not the same experiment.

The per-draw framebuffer comparison is now complete (rows 1226–1228);
its geometry correction and remaining compositing differences are below.
Retail `0x080E3382` walks the chains while drawing and alternates masked and
unmasked runs; preparation `0x080E0E96` stores an unchanged head plus a quad.
The former port separated those categories into spatial surfaces. Retail's
first overlay pass is masked (row 1223). Finding 1230 replaces the old depth
approximation with global encoded-space canvas composition.


**Pass order is now static, cross-build fact (2026-09-06, row 1223):** LGP
`0x080E3382` and Windows 2.28 ENG `0x62D3C0` are the same algorithm — whole-scene
alternating passes, **masked pass first** (flag 0 routes every entry to the
two-texture draw `sub_80DF8AA` / `sub_629300`), then an unmasked pass that skips
masked entries (single-texture `sub_80DF868` / `sub_6292C0`), a flush between
passes, repeating until a pass draws nothing; texture state switches lazily on id
change. ~~The port always composites the masked tile on top, so flipping the
two surfaces should correct the corner.~~ **Withdrawn 2026-09-06 (row 1224):**
this ignored the depth buffer. Unmasked quads write depth at their stack index;
masked quads test against it. A later opaque tile can therefore reject an
earlier masked fragment even though the masked surface is submitted last.

**Controlled rendering experiment (row 1224):** using the same stored retail
frame (`tmp/render-order-20260906/retail-38000.png`), fresh Forward+ renders
compared the unchanged shader, a masked-first arm (unmasked moved to the
transparent queue with depth writes retained), and a separate mask-depth-test
disabled arm. Right-corner crop `(830,650,194,118)`:

| Arm | Different pixels | Mean absolute RGB-channel error |
|---|---:|---:|
| Unchanged | 70.4307% | 19.6576 |
| Masked first | 70.4307% | 20.0274 |
| Mask depth test disabled | 70.4875% | 19.6676 |

The matching floor control `(170,290,40,20)` remains byte-exact in all arms;
the cabinet crop is unchanged too. Disabling depth testing changes 46.086%
of the corner, confirming that depth rejection is active, not hypothetical.
Neither experimental change was landed. The global pass flip is not the
corner fix; this does not establish whole-world equivalence to retail's
alternating per-chain runs. The subsequent per-draw comparison below used
the already trace-matched cells `(3246,2514)` through `(3248,2514)`.
Evidence for the earlier pass-flip experiment:
`donotpublish/tmp/floor-order-20260906/measurements.json`.

**Terrain alpha boundary corrected (row 1225):** LGP `sub_83B3E00` calls
`glAlphaFunc(GL_GREATER, 64/255)` (float bits `1048608897`); Windows 2.28 ENG
`sub_643190` sets render states `ALPHAREF=64`, `ALPHAFUNC=GREATER`.
The port's `c.r < 0.25` discard incorrectly accepted both 0.25 and equality
at 64/255. It now discards `c.r <= 64.0/255.0`. A Forward+ GPU boundary probe
fails before and passes after, rejecting both values and retaining 65/255.
Only 26 static pixels change in the start frame after excluding the two actor
rectangles; their summed absolute RGB error falls from 2358 to 2211.
This is a small correctness fix, **not** a solution to the cobble mismatch.

**Scope correction (row 1226):** the configured alpha reference above is
real, but does not prove the test is enabled. The captured LGP floor draws
have alpha testing **disabled**, standard source-alpha blending enabled,
and depth testing/writes disabled. The port still uses an opaque-depth
approximation; row 1225 is not proof of floor-pass equivalence.

#### Per-draw parity — geometry corrected, 2026-09-06

The live GL dispatch was hooked only after pointer equality verified the
entrypoints. Indexed batches were split into their existing two-triangle
quads so each contribution could be captured before and after. The final
split and unsplit retail corner/control crops are byte-identical. Of 135
captured quads, 19 belong to the three failing cells and control cell
`(3231,2513)`. Their actual texture bytes, UVs and blend state reproduce
every measured interior RGB contribution with maximum error below **0.84
byte** (rounding/filter precision), including masked draws.

Three geometry differences are now corrected:

- Cell coordinates identify the north corner, not the tile centre.
  Terrain was 24 screen pixels too high at 1x. The sector builder centres
  the tile half a cell south; objects and the camera are unchanged.
  Water uses the same corrected cell centre to stay aligned with its bed.
- Retail draws half-extents **48.2 / 24.2**, not 48 / 24. Both Linux and
  Windows initialize those constants. The 0.2-pixel overdraw covers seams.
- The UV table is four asymmetric tips, not a rectangle. With atlas-slot
  origin `(x,y) = (104*(n%2)+52*((n/2)%2),25*(n/2))`, integer division,
  retail's N/E/S/W coordinates are:
  `((x+50)/256+.002,(y+.5)/256+.002)`,
  `((x+97.5)/256+.002,(y+23.5)/256)`,
  `((x+50)/256,(y+48)/256+.002)`,
  `((x+2)/256+.002,(y+23.5)/256+.002)`.
  Both retail constructors populate this table at object offset +2096.
  `TextureFormat.slot_uv()` and its facade now return the four points;
  both art and mask consumers use them directly.

The production geometry matches all 19 captured quads within **0.0032 px**;
art and mask UVs match within **1e-7**. Controlled right-corner results,
same retail frame and crop `(830,650,194,118)`:

| Port arm | Different pixels | Mean absolute RGB-channel error |
|---|---:|---:|
| Before | 70.4307% | 19.6616 |
| Correct cell centre only | 69.3911% | 9.7404 |
| Centre plus retail UVs | 65.6954% | 6.7447 |
| Centre, UVs and retail extents — production | 33.6187% | 5.9615 |

The left corner changes from 13.5897% / 5.4064 to 10.0991% / 1.4607.
The matching floor control remains byte-exact. A fresh menu-to-game gate
passes at **world 7.80%, full frame 9.52%** (prior baselines 9.58% and
12.52%); these gate MAEs use the max-channel instrument, not the
mean-channel instrument in the table above. Floor-reader checks pass;
the liquid check passes its assertions but emits resource-leak diagnostics
at teardown. A separate no-player sea render verifies the water surface.

**Binding discrepancy resolved — oracle correction (row 1229):** the
captured masked LGP draws really did bind the mask page to both units,
but this was caused by our compatibility patch, not canonical composition.
It preseeded the cached MiB value at `0x8835E80` with 262144 (KiB).
The cached path bypasses the Xorg parser's division by 1024; startup's
32-bit `value << 20` therefore produced a **zero-byte texture budget**.
Live A/B confirms zero before and 268435456 bytes after changing only the
cached value to 256. Constant eviction explains texture-name reuse; a
once-per-name texture dump was invalid evidence.

The patch generator and that four-byte field in the live binary are now
corrected, with all other binary bytes and pristine originals preserved.
A fresh run observes **270 masked batches, zero aliased bindings**, with
17698304 texture bytes resident. Windows normal startup independently
clamps its texture budget to at least 32 MiB. The old binary, captures,
baseline and both hashes are retained in
`donotpublish/tmp/floor-cache-20260906/`. Scores across this reference
correction are not the same instrument. Do not emulate the broken zero-budget
reference.

**Ordered composition implemented (2026-09-07, row 1230):** `FloorView` draws
ground, then alternating masked/unmasked whole-scene chain passes in a non-HDR
canvas. Art RGB is multiplied by corner light before blending; masks retain
independent UVs and alpha. A spatial display plane decodes the completed
encoded-space image once for Forward+. Originally all camera and sector changes
rebuilt commands; the 2026-09-22 pan cache described below supersedes that
invalidation rule. Old per-tile depth and alpha-test shaders were removed.

The production render executes without script errors. Against the retained
corrected-budget reference, before adding miniature objects, the right corner
is 24.462694% different / RGB-channel MAE 4.404974; the matching floor control
is byte-exact. These are crop measurements, not a fresh parity gate.

The remaining corner error is not all floor: a fresh native capture draws two
later 64x64 textured objects at `(881,664)` and `(888,678)`, sampling
`MINIOBJ4_35.TGA` at UV 0..0.25. Their vertices are white, blending is enabled,
and alpha/depth tests are disabled. They resolve to static records 758596 and
758619, item 1062, whose direct texture field is populated while its MIX
sprite field is zero. Evidence: `tmp/floor-cache-20260906/miniature-state/`
and `production-floor-measurements.json`. Global object blending, lighting,
ordering and actor insertion still need parity work.

Evidence: `donotpublish/tmp/floor-draw-20260906/`, including
`draws.jsonl`, per-draw framebuffers/textures, `measurements.json`,
`production/port-inputs.jsonl`, and `final/` fresh gate captures.

**Banding null (row 1223):** default vs `--bands=4096` (4155 meshes vs 16) on
the same start-scene view differs only at the two animated actors — 0.0015% of
the world band, MAE 0.0005 — and both score 9.54/9.45 against that day's retail
reference (default repeat: 9.55). Per-sector mesh-band granularity is not a
live ordering variable for statics in this scene. Evidence:
`donotpublish/tmp/render-order-20260906/`.

### Miniature art and authored state admission — 2026-09-07

Finding 1231 restores the two direct-texture cobbles described above.
The same corrected-budget corner crop improves from **24.462694% / MAE
4.404974** to **13.611742% / MAE 1.049581**. Miniatures use the item
definition's texture and the placement's atlas parameters, not a nonexistent
MIX sprite. Their captured white vertex colors are not evidence that every
miniature is unlit.

Finding **1236** observes the sampler on both live miniature draws:
minification and magnification are **GL_NEAREST (9728)**, with
**GL_REPEAT (10497)** on both axes. The native texture bytes and an independent
floor control remain byte-identical to the retained reference. Thus switching
these cobbles to linear filtering is not a source-correct remedy for their
remaining difference. This observation is scoped to those two draws, not
every MIX texture or animation. Evidence:
`donotpublish/tmp/floor-cache-20260906/miniature-sampler/`.

A third apparent cobble is excluded by retail's building-state predicate.
The owning base cell redirects once through signed bytes `+28/+29`; its
target static chain supplies the first parent with flags `0x10`. Parent
`+39` selects a raw `cTrigger` record. The record is 16 bytes, and its
**u16 at +10 is state**, including legitimate zero. The loader copies the
authored table unchanged; state is not reconstructed from an interior enum.

In the main art list, after its other exclusions, gated records
`(definition_flags & 0x800004) || (placement_flags & 8)` are admitted when
there is no parent, or when **placement.u16(+43) equals parent trigger state**.
LGP `0x080E0E96` and Windows `0x0062AE90` agree. Parent `+23` and child
`+31` define storey order; child `+12/+45` identifies the native sector and
original height-record ordinal. Family names, rectangle overlap, and compact
region-list ordering are not substitutes for this identity.

The discriminating placement is static 758494, owner `(3244,2509)`, mask 0.
It resolves to parent 758529 at `(3219,2511)`, trigger 2034: authored state 1,
**live state 2** after the start transition. Both valid cobbles have no
parent gate. A nonmutating render-thread snapshot records 4222 live triggers;
the authored table contains 2268. Do not fabricate the script-created slots.

The port now shares raw trigger state between simulation and rendering.
Its `2 → 0 → 2` production signal path hides, shows, and restores the
previously extra cobble while both ungated controls remain byte-identical.
The corrected crop `(978,512,46,44)` has MAE **0.053524**, maximum error
**one RGB byte**: admission is corrected, but raster rounding is not exact.
Complete retained geometry permits later state changes; no permanently
discarded mask-zero sprite or fixed visibility-bucket ceiling remains.

Still open: native interaction prerequisites, script-created/save-restored
trigger routing, dynamic object/actor insertion and lighting. Finding 1237
implements global static ordering and encoded blending. Evidence: `donotpublish/tmp/floor-cache-20260906/extra-miniature-state/`
and `admission/`. The latter exercises the real renderer, not a copied predicate.



### Shelf-candle ordering — recovered and rendered, 2026-09-06

**Correction to rows 1016 and 1139:** the claim that this failure is
irreducible from shipped data is false. Retail visits cells diagonally
(`x+y`, then `x`), traverses each `static.pak` chain, and appends records
without a sprite-height re-sort to the selected draw list. In LGP 1.0.02,
preparation is `0x080E0E96`, list append `0x0864F8F0`; the definition flags
come from `items.pak +0x00` (accessor `0x08136218`). Bit 4 routes to list
0/2 using placement flags/mask; bit `0x800000` routes to list 4; ordinary
sprites go to list 3.

At cell `(3237,2507)`, the shipped chain places a wall, Cabinet 3, six plates,
then six CANDLE11 records. Sorting these by their sprite feet hid the candles.
The port now preserves pass/cell/chain order, without changing sprite
coordinates, art ids, or tile rectangles. In a controlled start-scene capture,
the cabinet crop `(710,185,140,185)` improves from **7.633% differing pixels,
MAE 5.822**, to **0.803%, MAE 0.023**. The six candles are visibly present.
World band `(0,0,1024,600)` improves **11.084% → 9.588%**, MAE
**4.108 → 3.234**. MAE is mean absolute RGB byte error. These are LGP
reference measurements, not a claim of cross-build or whole-world parity.

The shader also preserves authored alpha instead of forcing every surviving
texel opaque. Alpha-only control: world **11.072%**, MAE **4.086**; this small
change does not explain the much larger ordering improvement.

**Superseded by finding 1237:** mesh bands have been removed. Static objects
now share the encoded compositor in global native pass/cell/chain order.
Dynamic object/actor insertion remains open; neither result is whole-world parity.
Evidence: `donotpublish/tmp/placement-20260906/`, finding 1222.

### Global static composition and authored shadows — 2026-09-07

Finding **1237** replaces per-sector transparent mesh ordering with the
existing encoded-space canvas. Static sprites preserve the native global
pass/cell/chain sequence; texture switches do not regroup the sequence.
The obsolete `--bands` option and static 3D mesh construction are removed.
Actor and liquid draws remain separate; they are not silently claimed to
have native dynamic-list placement.

LGP `0x080EA104` reads the static shadow from item offsets **91** (u16 atlas
tile), **93/95** (s16 offsets, doubled), **99** (skew selector), and **100**
(u16 radius). This is separate from the actor radius at item +20.
The atlas is `SHADOW_TREE00.TGA`, split into 16×16 UV cells. Live native
draws confirm linear minification/magnification, repeat on both axes,
blending, and no depth or alpha test.

The definition shadow bit is `0x10000`; placement `0x800` suppresses it.
Mask-one placements enter dedicated list 1. Eligible gated ordinary sprites
with a higher mask carry flag 17 and draw their shadow immediately before
their sprite. Miniatures and higher-priority definition routes do not
acquire this inline branch. The captured start scene emits **97 dedicated
and 28 inline shadows**. An independently reconstructed experiment matches
all **125 × 4 native corner positions exactly**.

The native shadow colour uses `(255 - environment[77188])` alpha. The
captured daylight value is zero transparency. The port has no solar-clock
input yet: daytime is supported, night-dependent transparency is explicitly
unimplemented, not replaced with a tuned opacity.

Against the retained same reference, production world-band differing pixels
fall from approximately **7.642% to 5.230%**, RGB MAE **2.446 to 1.713**.
The corner crop falls from **13.611742% / MAE 1.049581** to
**10.588852% / MAE 0.047833**; the latter still has maximum error **7 bytes**,
so a small MAE is not pixel equality. The cabinet reaches **0.131868% /
MAE 0.005348**. The independent floor control remains byte-identical.
A live production `2 → 0 → 2` trigger transition changes 1,438 pixels in
the extra-cobble rectangle, restores it exactly, and changes neither of the
two ungated miniature controls.

Evidence: `donotpublish/tmp/global-order-20260907/run03/` (complete queue and
consumer witness), `control01/` (unchanged floor control), `run04/` (actual
shadow draws), `shadow-composition/` (frozen experiment), `production/`,
and `production-admission/`. Checks: **51 pass, 0 fail**; layer scan:
**26 files, 0 violations**. The then-missing chest was subsequently rendered
(findings 1253–1256). Finding 1260 maps ordinary actor insertion; finding 1263
below implements shared composition. Projected/stencil branches remain separate.

### Dynamic/static interleave — recovered contract, 2026-09-21

Finding **1260** recovered the ordering algorithm. **Finding 1263 implements
ordinary model insertion below.** Native does not compare model feet against sprite bounds.
It emits statics and dynamics into the same five FIFO vectors while traversing
cells. The consumer preserves that sequence.

Producer: LGP `0x80E0E96`, ENG `0x62AE90`, RUS `0x62B310`.
LGP append `0x864F8F0` copies a 24-byte record to vector.end; ordinary consumer
`0x80E7416` visits passes 0..4 with +24-byte iteration. The frame owner
`0x80EA3B8` completes preparation before special/ordinary consumption.
ENG/RUS producer agreement is independent; their ordinary consumer dumps are
split functions, not evidence for a clean full consumer decompilation.

#### Record and ownership

| Queue byte offset | Meaning |
|---|---|
| +0 | Tag; zero identifies a dynamic object |
| +4 | Object-manager reference for dynamics, static reference otherwise |
| +8/+12 | Float payload; elevated dynamics legitimately contain zero |
| +16/+20 | Integer screen-space anchor of the visited cell, not world coordinates |

Five ordinary `(begin,end,capacity)` headers begin at LGP worldView +525144,
stride 12; ENG/RUS +617168. The separate special/liquid vector is LGP
`0x8908A54`, ENG `0xAD7848`, RUS `0xAD67C8`.

Dynamic cell head is cell u32 +8; next is object u32 +48. Object +44 is a
**discriminated reference**: object flag 0x400 means dynamic owner, otherwise
static support (`0x815FA38/5C/72`). Do not treat every +44 as a static index.
Support lookup uses the actual static's sector u16 +12 and height-grid byte
+45; a numerical layer alone is not its identity.

#### Emission order for each visited base cell

1. Emit its static chain under the established static routing predicates.
2. Select support grids. **Correction, finding 1262:** the gate is the
   **lowest set-bit index** returned by `cTrigger::getState`, not simply raw
   state nonzero. Raw 0, 1 and 3 therefore select no support phase unless
   parent placement flag 0x400 selects **all authored children**, following
   parent +23 and child +31. Otherwise that index selects one authored child.
   Look up the current world cell inside the child's exact grid.
3. For each selected support cell, walk its dynamic chain twice:
   non-category-12 objects to pass 3 first, category-12 objects second.
   Category 12 routes to pass 4 for definition flag 0x800000; otherwise
   flag 4 **and object layer byte +36 == 0** routes to pass 0; else pass 3.
4. Then walk the original base cell's dynamic chain, again non-category-12
   before category 12. Base category-12 routing is the same **without the
   layer-zero condition**. Ordinary non-category-12 goes to pass 3, except
   the special/liquid route: cell special condition, type eligibility
   `0x814D216`, no object flag 0x100000, virtual query `(42,0)` returns zero,
   and item category is not 4 must all hold.

Definition category is record byte +46; flags are record u32 +0.
Within each category partition, preserve linked-list order.
Placement **prepends**, rather than sorting: LGP `0x811577A` / `0x8116B46`,
ENG `0x5FCCB0` / `0x5FDE80`, RUS `0x5FD200` / `0x5FE3D0`.
An ascending object-ID tie-break would invent behavior.
The dynamic consumer also has draw admission: definition flag 2 enables its
model/FX branch; flag 0x1000 adds a parent/trigger mask test. Being queued
does not prove a draw was submitted.

#### Fresh native witnesses and falsifying controls

The guarded 38-second start-scene snapshot contains ordinary list counts
**180,97,4,256,0**, with **14 dynamic entries**, all in pass 3. It has zero
missing owner chains and zero concurrent changes; all six vectors remain
unchanged when the same frame reaches the consumer.

| Pass-3 ordinal | Identity | Owning cell / captured grid | Neighboring static refs |
|---:|---|---|---|
| 82 | Chest type 5201 | (3232,2512), grid 1 | 758640 before, 758480 after |
| 108 | Seraphim type 1 | (3236,2511), grid 1 | next static 758436 |
| 152 | Novice type 679 | (3238,2518), grid 1 | 758700 before, 758645 after |

Four Coalpot/FX_FIRE_L pairs directly witness same-cell static-before-dynamic
emission. The captured hero and chest cells have **no static head**: a global
walker that skips static-empty cells loses them.
Moving all dynamics after all statics reverses **1,281 dynamic/static queue
relations** in this snapshot. This is an ordering counterexample, **not a
count of visible pixel errors**; many entries are offscreen.
The no-snapshot control preserves the independent floor crop byte-for-byte.
Both fresh gameplay frames were inspected.

**Original implementation gap:** the port flattened static commands into a
background plane and drew models separately. Finding 1263 replaces that split.
The ordering key preserves `(pass, cell visit, emission phase/ordinal)`,
including selected support grids and cells without statics.

**Still unverified live in retail:** multi-object same-cell ties/repositioning,
category-12 alternate passes, and a nonempty special vector. Those branches
are source-proven across builds, not exercised by the captured native scene.
Private reproducible evidence:
`donotpublish/tmp/global-order-20260907/mapping-20260921/verify_mapping.py`,
`measurements.json`, `scene-assets.json`, native queue/cell snapshots, and
`mapping-control-20260921/`. The asset manifest/dumps are not publishable.

### Shared ordinary model composition — implemented, finding 1263

The production renderer now inserts model images and their native five-quad
blob shadows into `FloorView`'s static FIFO. `SectorView.place_actor` receives
the actual definition, cell, support and layer; placement serials transcribe
head insertion, rather than sorting object IDs. Authored objects retain sector
lifetime; hero and creature views retain their existing simulation/main owner.
The forced-creature diagnostic now selects and reports an actual creature
definition instead of bypassing composition with type -1.

`ModelCanvas` preserves the model/skeleton/socket subtree inside an isolated
3D world. It caches per-bind geometry bounds and final modified bone poses,
then rasterizes a pixel-aligned orthographic crop. Hidden/offscreen targets
shrink to 2×2; no per-frame vertex extraction or CPU image readback occurs.
`frame_pre_draw` synchronizes animation, camera and FIFO commands.
**Correction, 2026-09-22:** finding 1271's actor-only split did not eliminate
camera-follow rebuilds and was not verified with frame timings. Orthographic
pans now translate cached canvas commands within a 256-pixel padded border.
Actor crops and shadows cancel that translation because they already use the
current camera. Crossing the border, zoom, rotation, depth motion, resize,
sector changes and admission changes rebuild the commands. Freeing commands
also invalidates both ordering streams; the previous implementation could
rebuild ground while silently dropping statics after a camera change.

The asset-free `probes/canvas_pan_probe.gd` compares cached and freshly rebuilt
frames byte-for-byte across integer/fractional pans, cache-boundary crossing,
zoom and resize, and checks that an initially offscreen static enters the
frame. Run with Forward+/Vulkan and a display (or `xvfb-run`), not headless.
At 2560×1440 under Xvfb, a 60-frame camera-pan probe measured median
`FloorView.sync` CPU time **129.005 ms before → 1.649 ms after**; this is an
isolated renderer measurement, not a whole-game FPS claim.

A separate native-desktop run used a maximized **3456×1926** window, the same
new-game route and movement goal `(3240,2514)`, with the pre-change script
loaded as the control. Movement-only median/p95 frame times changed from
**255.713/545.819 ms** (36 moving frames) to **28.047/59.227 ms** (119 moving
frames). The hero reached the same destination. That pan-only build's full
sample still contained a **760.153 ms** worst frame; it was not a fix for
streaming stalls. Evidence: `donotpublish/tmp/walk_perf.gd`,
`movement_perf.gd`, `floor_view-before-pan.gd`, and `walk-perf.png`.

**Progressive loading, 2026-09-22:** the user's overlay showed 23 resident
sectors and seven queued while standing still. A matched-view profile found
30 synchronous sector builds averaging **349 ms**, each followed by a canvas
rebuild averaging **303 ms**. Earlier warmed-frame benchmarks excluded this
work and therefore could not substantiate a responsive ordinary launch.

Sector construction and canvas command generation now have separate **3 ms
cooperative work slices per process frame**. Sector data stays job-owned until
complete; camera changes cancel unwanted work. Canvas generations build under
hidden roots and publish atomically, retaining the prior complete image and
its resources during construction. Additive arrivals coalesce rather than
perpetually restarting the build; invalidating changes cancel obsolete jobs.
Retired raw RIDs release incrementally. Display-plane transforms are flushed
at publication, avoiding the one-frame stale transform exposed by the pan
pixel-equality probe.

Liquid frame decoding cooperates with the same sector checkpoint, and a
cancelled decode is never cached as failure. Authored 3D objects use a shared
budgeted queue and hidden sector-owned staging nodes; the sector's object
group is registered and shown only when complete. Scene readiness includes
these jobs and rendered canvas publication. Capture/probe waits now use a
120-second completion deadline and fail rather than photograph incomplete
loading after an arbitrary 600 frames.

Cold-start measurements include loading, not just the settled scene:

| Client size | Loading median / p95 / max | Settled median / p95 | Loading duration |
|---|---|---|---|
| 1024×768 | 9.219 / 21.292 / 210.938 ms | 8.315 / 9.917 ms | 12.942 s |
| 3456×1926, matched view | 21.274 / 28.774 / 355.581 ms | 19.901 / 21.341 ms | 33.177 s |

F3 input was processed before loading completed in both runs. These budgets
are cooperative, **not hard upper bounds**: individual native resource/model
operations and initial navigation setup are still atomic. The default run
logged a 205 ms initial path-window fill. Progressive loading trades longer
large-view fill time for responsive intervening frames; no hitch-free or
universal 60 FPS claim is made. Evidence remains private:
`donotpublish/tmp/progressive_startup_probe.gd`, `progressive-default.json`,
`progressive-large.json`, and their screenshots.

The old warmed-shot benchmark correctly **refused to emit a score** after this
change: its two captures differ in 5,123 pixels, localized to the animated hero
and novice. Wall-time-budgeted loading finishes on different simulation frames,
so “four seconds after loading” no longer pins their animation phase. Its
repeat-equality safeguard remains intact; no new pixel-parity score or baseline
was asserted. Frozen production-scene admission and unload/revisit comparisons
still restore the entire image byte-for-byte. A future benchmark comparison
needs an explicit common animation timestamp, not a relaxed equality check.

Opaque/scissored body surfaces now use self-depth, replacing the old
`no_depth_test` workaround. The existing calibrated actor lighting is unchanged.

Blob shadows are excluded from the 3D capture and submitted directly in
encoded RGB before their actor. This removes their previous linear-framebuffer
blend discrepancy without fitting a new opacity. Cropped model images use
premultiplied-alpha composition; the final scene is decoded once for Forward+.

**Admission correction, finding 1262:** IDA MCP decompilation of `0x83A6312`
confirms lowest-bit indexing. Real chapel raw-state tests verify 2/6 admit
child 1, 0/3 reject it, and base dynamics remain independent. Exact grid
identity uses sector + height-grid ordinal. Zero-size aliases preserve every
native phase occurrence, including the subsequent base phase; they are not
deduplicated. The unflagged selector retains its second signed-cell redirect.

Verification exercises actual Forward+/Vulkan rendering:

- A model behind a static is covered; the same geometry later in the FIFO
  covers that static. This reproduction failed before integration.
- Two same-cell models honor prepend/reposition order and category partition.
- Hide/restore, camera pan/zoom, target freeing and automatic tree re-entry pass.
- In the production scene, raw **2 → 0 → 2** and a distant sector
  **unload → revisit** each restore the entire frozen frame byte-for-byte.
  The chest is freed/rebuilt, the hero survives, and actor counts do not grow.
- Forced `WOLF.GRN` resolves type 588 and passes the same lifecycle route.
  The ordinary route has 37 registered models but only 3 active targets,
  totaling 19,307 pixels at the measured bench position; forced wolf adds a
  fourth active target (65,363 total pixels). These are allocations at those
  views, not an FPS claim.

Fixed benchmark: world delta **5.05% → 4.65%**, full delta **6.35% → 6.04%**,
repeat difference **0.00%**. World MAE **1.85 → 1.89** and full MAE
**2.97 → 3.00** worsen slightly: fewer differing pixels does not mean every
remaining pixel is nearer retail.

**Boundary:** this implements ordinary model/static FIFO composition, not
full renderer parity. Native cross-model shared-depth equivalence is not
established by independent model targets. Liquid/special-vector composition,
projected object shadows, actor lighting and existing pose/placement errors
remain separate gaps.
Private screenshots and runnable smoke harnesses:
`donotpublish/tmp/shared-composition-20260921/`; fixed benchmark artifacts:
`donotpublish/tmp/autoresearch-start-scene/`.

**Historical reading below — superseded, not current guidance:**

Object placements are sorted at the sprite's foot (see `mixed.pak`, above),
gated on sprite width so that a room shell is not sorted in front of the
furniture standing inside it. The gate leaves one case wrong: a candle
standing on a wine rack's **shelf** is covered by the rack, because the rack's
foot is on the floor and the candle's is up in the air.

A single painter key cannot order a sprite that contains another. Retail draws
building parts in an authored order — the part index in
`<BUILDING>_<level>_<part>` is the obvious candidate — and that order has not
been recovered. An aspect-ratio gate ("only give the height to something far
taller than it is wide") was measured and **refuted** at every ratio from 1.5
to 4.0; it loses more on squat furniture than it wins back on candles.

**UNVERIFIED EXTERNAL CLAIM, recorded 2026-08-25 — it would explain the failure
to recover this, so test it before searching further.** Two Russian modders who
have each done their own reverse engineering state, in a
[2023-03-12 thread](https://vk.ru/wall-191594029_1960) recovered by `tools/vk/`,
that the composition is **not in the shipped files at all**:

> «Какие именно составные объекты входят в статический объект в файлах не
> описано. Эти описания были в редакторе уровней (нам не доступном).»
> — *which composite objects go into a static object is not described in the
> files; those descriptions were in the level editor, which is not available to
> us.*

Their model of the hierarchy is **tiles -> MIX-objects (`mixed.pak`) -> static
objects (`static.pak`), and static objects carry STATES**; the level editor
emitted `Static.PAK` and `Floor.PAK` from a source map that never shipped. One
of them wrote a script that assembles MIX-objects out of `mixed.pak` and reports
that **furniture assembles whole while houses come out only as "MIX-parts"** — a
floor, a piece of front wall — which is the same boundary this project hits from
the other side.

WARNING: this is a forum claim, and it is documentation of behaviour rather than
a citation — the same clean-room rule as `community/unpack-tools/`. It is
recorded because it is *falsifiable and cheap to test*: if the per-static-object
part list really is absent from the shipped data, recovering "the authored
order" from files is impossible by construction, and this item should be
re-scoped to a measured per-building table rather than a search. Nobody has
checked it. **Do not cite it as a finding and do not close this item on it.**

The authored building-state links and admission predicate are now documented
above. Full native building interaction and actor displacement remain separate
from the renderer's recovered visibility contract.

---
Provenance: `tools/parity/verify_ref.py` and `engine/world/walkable.gd`, which are the
two independent decoders; findings log rows tagged `sectors.wldx` and `world`.

