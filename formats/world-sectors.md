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

> `+0x08` is empty in every cell of the retail world. It is **not** meaningless:
> the Armalion prerelease populates the same slot in 73 cells, so it is a
> layer the shipped game dropped rather than a field with nothing behind it.
> What it indexes is still unidentified — the 73 values are distinct and run to
> 170, while the prerelease's `World/NonStatic.PAK` (a file retail does not
> ship at all) holds only 106 records, so they do not index it as an array.
> See *The prerelease is not a second corpus* below for how the two records
> line up.
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

**Reflection** (open): `+0xCC` gates a pass-1 vertically-mirrored untextured
quad modulated by ambient·(−8·depth). Its geometry is read; its blend state
goes through untraced render-state calls (`sub_83B41F2`), so the port does
not draw it yet.

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

All 32 bytes of the retail cell are accounted for, and one of them has since
become slightly more open rather than less. `+0x08` is empty in every retail
cell and was written off as a runtime slot; the prerelease populates it in 73
cells, so it is a **dropped layer** and the table those 73 distinct values
index is unidentified — `NonStatic.PAK` is the obvious candidate by name and
does not fit, holding 106 records against a maximum value of 170. Nothing in
the retail port depends on it, which is why this is a curiosity rather than a
blocker.

Building construction and the interior/exterior swap are a separate matter and
are deliberately not documented here; the reason is under Related, above.

---
Provenance: `tools/parity/verify_ref.py` and `engine/world/walkable.gd`, which are the
two independent decoders; findings log rows tagged `sectors.wldx` and `world`.

