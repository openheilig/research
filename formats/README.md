# formats — one document per file format

**Purpose:** One document per retail file format, field by field, with the state of each
decode stated up front.

What each retail file format is, field by field, with the status of the
decode stated up front. Read the document before re-deriving anything: most
questions here are already settled, and several were settled twice because
nobody looked first.

| Document | Format | Status |
|---|---|---|
| [pak-containers.md](pak-containers.md) | `.pak` archives, both shapes | Solved |
| [world-sectors.md](world-sectors.md) | `sectors.keyx` / `sectors.wldx` | Solved |
| [granny-grn.md](granny-grn.md) | Granny 1.x `.GRN` models, skeletons, animation | Read — 3413 of 3421 clips |
| [pax-saves.md](pax-saves.md) | `.pax` hero saves | Read |
| [script-bytecode.md](script-bytecode.md) | `StartCode.bin` / `FunkCode.bin` | Read — format closed, semantics open |
| [balance-bin.md](balance-bin.md) | `balance.bin`, the rules dataset | Read |
| [global-res.md](global-res.md) | `global.res`, the text resource tree and its two namespaces | Read |
| [armalion-acs.md](armalion-acs.md) | `ACS` — Armalion's compiled scripts, checked against the source shipped beside them | Read — a vocabulary for retail's opcodes, not a key to them |

Status words mean what [../README.md](../README.md) says they mean, and every
document ends in a `## Open` section — read that before assuming a format is
finished.

## Generated tables — `generated/`

These live in their own directory because they are **machine output**: the
generator rewrites the whole file, header included, so an edit here survives
until the next run and no longer. Change the generator instead.

| Table | From |
|---|---|
| [script-opcodes.md](generated/script-opcodes.md) / `.json` | `tools/binary/opcodes.py` — handler addresses, strings, RTTI classes |
| [script-opcodes-behaviour.md](generated/script-opcodes-behaviour.md) | `tools/binary/opsem.py` — what the shipped scripts do with each opcode |
| [balance-keymap.tsv](generated/balance-keymap.tsv) / `.json` | `tools/binary/balance_keymap.py` — 361 of the 379 balance fields, resolved to offsets with type and shipped value |

## Related

Implementations: Python in [tools/formats/](../../tools/formats/), GDScript in
[engine/formats/](../../engine/formats/). The two are diffed against each
other — that agreement, not either one alone, is why these documents claim
what they claim. The rule is in [../method/discipline.md](../method/discipline.md).
