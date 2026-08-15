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

The 17/15 boundary is **proven, and was not until 2026-08-15**. The gate used
to assert only that the top field resolves to a valid tile whose orientation
equals its index mod 18 — and neither can fail, because that identity holds
for **all 90,132** tiles.pak indices and a doubled small id stays inside the
count. Reading the field from bit 16 instead of 17 passed. What settles it is
that the two fields must partition the word with no shared bit: only 17/15
reconstructs the original u32, and `floor_check` now asserts that. Drawing this layer is what moved the port's
load-path invariant from 28,672 quads to 35,340.

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

