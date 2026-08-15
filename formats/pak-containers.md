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

> **Version equality does not imply record equality.** `static.pak` is `OBJ`
> v1 in both builds and its stride is 76 bytes, against Armalion's
> `sObjectStatic` of 64. The version byte covers the container, not the
> payload. (`cTrigger` at 16 bytes against `triggers.pak`'s 16-byte stride at
> `TRG` v1 in both is the one case that does line up.)

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

**Fixed-record** (three files only). Records tile the file directly with
`stride = (filesize - 256) / count`; there is no index.

## The corpus

| File | Magic | v | Entries | Layout | Payload |
|---|---|---|---|---|---|
| `creature.pak` | CIF | 0 | 474 | fixed | 86 B (`0x56`) |
| `weapon.pak` | WPN | 8 | 4883 | fixed | 322 B |
| `motions.pak` | MHP | 1 | 3423 | fixed | 198 B |
| `items.pak` | ITM | 5 | 32768 | blob | uniform 128 B (`0x80`) |
| `tiles.pak` | ISO | 3 | 90132 | blob | uniform 64 B |
| `sndprofiles.pak` | SPF | 1 | 8192 | blob | uniform 184 B |
| `models.pak` | MDL | 3 | 4993 | blob | `.GRN` |
| `texture.pak` | TEX | 3 | 25535 | blob | `.TGA` |
| `sound.pak` | SND | 1 | 50000 | blob | RIFF/WAVE |
| `mixed.pak` | MIX | 0 | 32096 | blob | heterogeneous |
| `items03` / `models03` / `texture03` | — | — | 32768 / 4 / 3 | blob | Underworld overlays |

`world/sectors.keyx` is the same family: magic `WLK` v5, and
`(4646656 − 256) / 768 = 6050` sector records exactly.

## Two corrections worth keeping

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

## Open

`mixed.pak`'s heterogeneous payloads, Bink `.bik` (ffmpeg decodes it), Miles
`.mss`.

---
Provenance: measured against the retail install in this project's private
analysis workspace. Tools: `pak.py`, `parse_trg.py`, `verify_ref.py`.

