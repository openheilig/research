# formats — one document per file format

What each retail file format is, field by field, with the status of the
decode stated up front. Read the document before re-deriving anything: most
questions here are already settled, and several were settled twice because
nobody looked first.

| Document | Format | Status |
|---|---|---|
| [pak-containers.md](pak-containers.md) | `.pak` archives, both shapes | Solved |
| [world-sectors.md](world-sectors.md) | `sectors.keyx` / `sectors.wldx` | Read and streaming |
| [granny-grn.md](granny-grn.md) | Granny 1.x `.GRN` models, skeletons, animation | Read end to end — 3413 of 3421 clips |
| [pax-saves.md](pax-saves.md) | `.pax` hero saves | Decoded |
| [script-bytecode.md](script-bytecode.md) | `StartCode.bin` / `FunkCode.bin` | Parsed from the interpreter |
| [balance-bin.md](balance-bin.md) | `balance.bin`, the rules dataset | Layout determined, keys named |

## Generated tables

These are produced by the tools, not written by hand, and are regenerated
rather than edited:

| Table | From |
|---|---|
| [script-opcodes.md](script-opcodes.md) / `.json` | `tools/binary/opcodes.py` — handler addresses, strings, RTTI classes |
| [script-opcodes-behaviour.md](script-opcodes-behaviour.md) | `tools/binary/opsem.py` — what the shipped scripts do with each opcode |
| [balance-keymap.tsv](balance-keymap.tsv) / `.json` | `tools/binary/balance_keymap.py` — the 380 named balance keys |

## Related

Implementations: Python in [tools/formats/](../../tools/formats/), GDScript in
[engine/formats/](../../engine/formats/). The two are diffed against each
other — that agreement, not either one alone, is why these documents claim
what they claim. The rule is in [../method/discipline.md](../method/discipline.md).
