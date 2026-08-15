# `.pak` containers

**Status:** Solved
**Purpose:** The two `.pak` layouts, and which files are and are not this
format.

Both shapes read, `--list` and `--extract` work, `--self-check` passes.
Reader: `tools/formats/pak.py`.

The 256-byte header was derived independently here, then found to agree
verbatim with Resacred's `rs_file.h:91-109` and the Delphi `PakExtractor`
spec — two outside descriptions that were consulted only after the fact.

A fourth source has since turned up, and it is the only one that gives the
**field names**: the Armalion debug build asserts on `hdr.tag[0]`,
`hdr.tag[1]`, `hdr.tag[2]`, `hdr.ver` and `hdr.numEntries` — the engine's own
names for the three magic bytes, the version and the count. Its `DEBUG.LOG`
also logs each container's version, so `TEX` v3, `MDL` v3, `ISO` v3, `SND` v1,
`OBJ` v1 and `TRG` v1 are unchanged across the three years to retail, while
`ITM` went v2→v5 and the world containers v4→v5. See
[../builds/armalion-source-tree.md](../builds/armalion-source-tree.md).

**`OBJ` v1 is frozen, and the debug build names its record.** Checked against
the prerelease's own `World/` directory:

| file | Armalion 2001-09 | retail | payload | flags |
|---|---|---|---|---|
| `Floor.PAK` | `OBJ` v1, 2,591,136 | `OBJ` v1, 6,713,136 | **16 B in both** | `0x87` |
| `Static.PAK` | `OBJ` v1, 393,765 | `OBJ` v1, 1,038,014 | **64 B in both** | `0x86` |
| `Triggers.PAK` | `TRG` v1, 1,218 | `TRG` v1, 2,268 | 16 B in both | — |
| `Sectors.key` | `WLK` **v4**, 1,201 | `WLK` **v5**, 6,050 | 512 → **768** | — |

Same version, same payload and same flags byte, three times; the one container
whose version moved is the one whose record moved. **The version byte covers
the record's size and field offsets, not the bit packing inside a field** —
`floor.pak` is `OBJ` v1 in both builds and its `+0x04` word is split 16/16 in
the prerelease and 17/15 in retail, because the tile table outgrew 16 bits.
See [world-sectors.md](world-sectors.md). The world grew from 1,201
sectors to 6,050 and from 2.6M to 6.7M floor records without the layout
changing, so **the prerelease world data is a second corpus for the retail
readers** — `pak.py` reads it unmodified.

`Static.PAK`'s 64-byte payload is `sObjectStatic`, which the debug build
prints as 64 bytes. That is the engine's own name for the record.

> **Count the index separately from the payload.** Dividing file size by entry
> count gives 76 for `static.pak` — 12 bytes of blob index plus the 64-byte
> record — and 76 matches no class. Read twice here: first as evidence that a
> frozen version can still change its record, then as evidence that an on-disk
> stride is not comparable to a `sizeof` at all. Both were the same arithmetic
> slip, and the payload does equal the `sizeof`.

`Scripts/SCRIPT.PAK` from the Armalion prerelease is this same format under a
different magic — `ACS` v2, blob layout, read by `pak.py` unmodified. See
[armalion-acs.md](armalion-acs.md).

## Header — common to every `.pak`

```
0x000  char[3] magic + u8 version
0x004  u32     entry count
0x008  u8[8]; i32 worldX; i32 worldY; u8[232]   -> 0x100
```

## Two layouts

**Blob** (the majority). An index at `0x100`:

```c
struct { u32 flags; u32 offset; u32 size; }[count]
```

Offsets are absolute and the entries are contiguous. Named payloads begin
with a NUL-padded filename. Known flags: `0x04` TGA, `0x40` Granny `.GRN`,
`0x20` raw RIFF/WAVE.

> **`size` is the PAYLOAD, not the entry.** In `texture.pak` an entry is a
> 32-byte name, then `u16 width; u16 height; u32 format; u32 payload_size`,
> zeros to `+80`, then the zlib stream — and the index's `size` field is that
> zlib stream's length alone. `ELVE_SORCERESS_HANDS.TGA` declares **15**, and
> is a real 16×16 texture the game draws. Anything that treats `size` as the
> entry length, or gates on it to decide whether a name is there, loses the
> small entries: 28 of `texture.pak`'s 25535 are under 32 bytes, all of them
> solid-colour placeholders (`DUMMY*`, `FX_OPAQUE`, and eight `*_HANDS`
> referenced 13 times from `models.pak`).

**Fixed-record** (three files only). Records tile the file directly with
`stride = (filesize - 256) / count`; there is no index.

## The corpus

| File | Magic | v | Entries | Layout | Payload |
|---|---|---|---|---|---|
| `creature.pak` | CIF | 0 | 474 | fixed | 86 B (`0x56`) |
| `weapon.pak` | WPN | 8 | 4883 | fixed | **two** sections: 258 B + 64 B |
| `motions.pak` | MHP | 1 | 3423 | fixed | 198 B |
| `items.pak` | ITM | 5 | 32768 | blob | uniform 128 B (`0x80`) |
| `tiles.pak` | ISO | 3 | 90132 | blob | uniform 64 B |
| `sndprofiles.pak` | SPF | 1 | 8192 | blob | uniform 184 B |
| `models.pak` | MDL | 3 | 4993 | blob | `.GRN` |
| `texture.pak` | TEX | 3 | 25535 | blob | `.TGA` |
| `sound.pak` | SND | 1 | 50000 | blob | RIFF/WAVE **and** Ogg Vorbis |
| `mixed.pak` | MIX | 0 | 32096 | blob | heterogeneous |
| `items03` / `models03` / `texture03` | — | — | 32768 / 4 / 3 | blob | one promo item |

`world/sectors.keyx` is the same family: magic `WLK` v5, and
`(4646656 − 256) / 768 = 6050` sector records exactly.

`sound.pak`'s index flag selects the codec, 1:1 across all 6598 payloads with
zero exceptions: `0x20` ↔ `RIFF` (3194), `0x21` ↔ `OggS` (3404). Naming and
the `sndprofiles` slot enum live in the executable, not on disk — see
[install-inventory.md](install-inventory.md).

## Three corrections worth keeping

> **`weapon.pak` is not a 322-byte record.** 322 divides the body exactly
> (`256 + 4883×322` = the file size), which is what made it convincing, but it
> is the SUM OF TWO PARALLEL TABLES: 4883 records of 258 B from `0x100`, then
> 4883 records of 64 B from `0x133966`. The discriminator is a name field —
> at stride 258 an ASCII run sits at `+0x28` in 81.3% of records, at stride
> 322 the best offset reaches 5.0%, which is noise. Measured 2026-08-16.

> **The `03` overlays are not Underworld content.** They ship exactly one
> item between them: `items03.pak` carries a single populated record (3999,
> `Krombacher.grn`), `models03.pak` holds `KROMBACHER.GRN` plus a motion and
> two `INVALID_` sentinels, `texture03.pak` holds `KROMBACHER.TGA` and
> `CAB.TGA`. Krombacher is a brewery: it is product placement. Base slot 3999
> is empty, so the overlay adds rather than replaces.

> **`items.pak` is not 140-byte stride.** That figure was arithmetic
> coincidence. It is a blob container of **128-byte** records — so the
> engine's `type*0x80` table does have an on-disk counterpart after all,
> though the in-memory struct differs: the `+0x1a` creature-row field is zero
> in every on-disk record, meaning that indirection is built at load time.

> **Small entry counts tile by accident.** The `03` overlays were misdetected
> as fixed-record for that reason. `pak.py` now prefers blob whenever entry 0
> parses as a valid index slot.

## Not this format

`Triggers.PAK` is **not** a `.pak` at all — it is `TRG v1`, a different
container that happens to share the extension. Reader: `tools/formats/parse_trg.py`.
That raised the obvious follow-up, audited separately: which other files can
the community toolchain misread the same way.

## `mixed.pak` is not heterogeneous, and was not open

This document listed `mixed.pak`'s payloads as open. They were not: the port's
`engine/formats/mixed.gd` had already read them, and the entry here was simply
never struck. Recorded rather than quietly fixed, because a research doc
lagging the engine is the failure mode this split is supposed to prevent.

Every slot is populated and every one has the same flags byte (`0x63`). A
record is one **sprite assembled from pieces**:

```
u32  count            // 0 for the 15,840 records that are invisible markers
u16  w, u16 h
i16  dx, i16 dy
u32  (uninitialised 0xcccccccc, in BOTH builds)
{ char[32] name; u32 texture_id; u16 x1,y1,x0,y0; u32; f32 u0,v0,u1,v1 } × count
```

Confirmed against the Armalion prerelease's own `MIXED.PAK`, which is `MIX` v0
there too (8,192 slots against retail's 32,096):

| | retail | prerelease |
|---|---|---|
| `size == 16 + count*64` | 32,095/32,095 | 8,191/8,191 |
| element names ending `.444` | 209,956/209,956 | 24,297/24,297 |
| `texture_id` below `texture.pak`'s count | max 25,534 of 25,535 | max 17,138 of 17,146 |
| the four floats inside [0,1] | 839,824/839,824 | 97,188/97,188 |
| destination rect ordered `x1>=x0, y1>=y0` | 209,956/209,956 | 24,297/24,297 |

`.444` is **Sacred's own image format** — `armaSource/library/formats/444.cpp`
in the [source tree](../builds/armalion-source-tree.md), and the debug build
asserts `hdr.tag[0]=='4'` through `hdr.tag[2]=='4'`. Retail ships no `.444`
files; the names are provenance for art that became `texture.pak` TGAs.

## A header variant, so nobody assumes the count is always at +4

The Armalion prerelease's `PAK/GFX.PAK` is `GFX` **v5**, and its entry count is
at **+0x08**, not +0x04 where every retail container puts it: the header reads
`"GFX", 5, 0, 65536, 0, 0`. The count is confirmed by arithmetic rather than
by reading it — `0x100 + 65536*12` is exactly entry 1's offset. 2,454 of the
65,536 slots are populated, all with flags `1`, and each payload begins
`"GraphiMix\0"` followed by its own length.

Retail ships no `GFX.PAK`; the 2D sprite archive became `mixed.pak` plus
`texture.pak`. It is recorded only so that "the count is the `u32` at +4" is
not treated as a rule of the family.

## Open

Bink `.bik` (ffmpeg decodes it), Miles `.mss`.

---
Provenance: measured against the retail install in this project's private
analysis workspace. Tools: `pak.py`, `parse_trg.py`, `verify_ref.py`.

