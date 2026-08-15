# Open questions

**Status:** Standing
**Purpose:** Every open item from every document in this repository, in one
list, so the state of the project is one read rather than eleven.

Each entry is the `## Open` section of the document it names — that document
is the authority, this is the index. If you close one, close it there and
strike it here.

## Formats

| Question | Where it lives |
|---|---|
| Eight of 3421 animation clips do not decode. | [formats/granny-grn.md](formats/granny-grn.md) |
| One mesh's vertex count disagrees with an outside reading, 279 against 280. | [formats/granny-grn.md](formats/granny-grn.md) |
| `mixed.pak`'s heterogeneous payloads are unread; Bink `.bik` and Miles `.mss` are third-party formats we do not decode. | [formats/pak-containers.md](formats/pak-containers.md) |
| `global.res`'s by-name namespace appears to hold only numeric names; no plain-word name resolves, so whether the Armalion build's named resources were carried over at all is unread. | [formats/global-res.md](formats/global-res.md) |
| Whether the `0xC8` record type ids share the `items.pak` id space. A probe with a control arm exists; it has not returned a verdict. | [formats/pax-saves.md](formats/pax-saves.md) |
| 102 script opcodes have no verified meaning beyond what their string payloads suggest, and 66 zero-width tags are presumably operators whose identity sits in handlers already located. | [formats/script-bytecode.md](formats/script-bytecode.md) |

Nothing is open on `.pak` framing, the world cell record, or the `balance.bin`
layout and key names.

## Engine behaviour

| Question | Where it lives |
|---|---|
| The combat RESOLUTION step is undecoded: how damage meets resistance, criticals, and what the weapon-slot flag selects. To-hit and the derived-stat kernel that builds the damage and resistance numbers are recovered. | [engine/combat-formulas.md](engine/combat-formulas.md) |
| No recovered name has been confirmed by behaviour: cross-source agreement covers 34 of 131, the behavioural arm covers none. | [engine/decompilation-coverage.md](engine/decompilation-coverage.md) |
| 11,303 of 13,716 callables in the retail binary have no identity, and the vtable route that produced the other 2,413 is saturated. | [engine/decompilation-coverage.md](engine/decompilation-coverage.md) |
| Which creature-struct offset is which named attribute: `+0x56` feeds every damage channel, `+0x4a` and `+0x4e` are two weapon slots. `FUN_081f686a` and its 117 balance keys is where to settle it. | [engine/combat-formulas.md](engine/combat-formulas.md) |
| The Granny converter cannot be driven past a null dereference on any real Sacred `.GRN`. Reopening means a different converter build, not a different way of calling this one. | [engine/granny-runtime-oracle.md](engine/granny-runtime-oracle.md) |

## Deliberately not open

These are settled decisions, not gaps, and re-raising them costs time:

- **Building construction and the interior/exterior swap** are not documented
  in this repository. Their settled half is entangled with render-architecture
  choices about the port, which are decisions rather than facts about a
  format.
- **What the individual balance tunables do to play** is not a format
  question. It needs the engine to consume them first.

## Not questions — integration debt

Decoded, written up, and simply not wired into the engine yet. Named here so
they are not mistaken for research:

- `triggers.pak` and `DefPos.bin` have readers and appear in no engine
  code path.

---
Provenance: the `## Open` section of each document named above; the findings
log for the integration items.
