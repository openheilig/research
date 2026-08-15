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
| `+0x1f` | low nibble = class; high nibble = a 16-class ground-type tag |

Three of these are worth stating as negatives, because each cost a round of
hypotheses:

> `+0x08` carries no meaning to recover. It is empty in every cell of the
> world — a slot the running engine fills, not data the map ships.
>
> ~~`+0x1e` bit 1 marks "covered by a region sub-grid".~~ Refuted.
>
> ~~`+0x18..0x1b` is a per-corner overlay blend.~~ Refuted; it is a second
> height, sampled the same way as `+0x10`.

`+0x1e` and `+0x1f` were both settled by **drawing the field** as a per-cell
colour overlay after statistics stalled on them. A bit whose meaning will not
separate in a census often separates instantly when you can see where in the
world it is set.

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
[pak-containers.md](pak-containers.md).

Building construction — how pieces are placed and linked, and how the
interior/exterior swap works — is deliberately **not** documented here. Its
settled half is entangled with render-architecture decisions about the port,
which are choices rather than facts about the format.

## Open

Nothing open on the cell record: all 32 bytes are accounted for. Building
construction and the interior/exterior swap are a separate matter and are
deliberately not documented here; the reason is under Related, above.

---
Provenance: `tools/parity/verify_ref.py` and `engine/world/walkable.gd`, which are the
two independent decoders; findings log rows tagged `sectors.wldx` and `world`.

