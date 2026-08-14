# `.pax` hero saves

**Status: decoded.** Section framing and the character stream both read;
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

## Open

Whether the `0xC8` record type ids share the `items.pak` id space is an open
question, not a settled one — `engine/probes/pax_c8.gd` tests it against a control
arm of random ids in the same numeric range, because without the control a
hit rate means nothing.

## Note on the corpus

The eight-hero `.pax` corpus is third-party sample data, not part of the
retail install and not shipped in any of these repositories. The probes read
it from `$SACRED_CHARS`.

---
Provenance: findings log rows tagged `pax-hero`; `tools/parity/pax_diff.py`;
`engine/checks/pax_check.gd`.

