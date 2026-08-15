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
which are `.pax` files in all but extension and hold the new-game starting
character for each class. The six section types this document lists are scoped
to the external corpus; the retail files carry eight, so they extend the list
rather than contradict it.

## Open

Nothing open.

---
Provenance: findings log rows tagged `pax-hero`; `tools/parity/pax_diff.py`;
`engine/checks/pax_check.gd`.

