# OpenHeilig — research

Reverse-engineering findings on *Sacred Gold*: its file formats, script
bytecode, balance data and engine behaviour.

Everything here is prose and tables we wrote. There are no bytes from the game
in this repository — no extracted assets, no decompiler output, no symbol
tables, no memory dumps. Those stay in a private workspace by design, and the
`.gitignore` is an allowlist so they cannot arrive by accident.

## Start here

| I want… | Read |
|---|---|
| to read a `.pak` | [formats/pak-containers.md](formats/pak-containers.md) |
| to load the world | [formats/world-sectors.md](formats/world-sectors.md) |
| models, skeletons, animation | [formats/granny-grn.md](formats/granny-grn.md) |
| hero saves | [formats/pax-saves.md](formats/pax-saves.md) |
| script bytecode and opcodes | [formats/script-bytecode.md](formats/script-bytecode.md) |
| the rules and tunables | [formats/balance-bin.md](formats/balance-bin.md) |
| how combat resolves | [engine/combat-formulas.md](engine/combat-formulas.md) |
| what the Linux port is built on | [engine/tech-stack.md](engine/tech-stack.md) |
| which binary to open | [builds/build-survey.md](builds/build-survey.md) |
| how to name a function | [method/naming-oracle.md](method/naming-oracle.md) |
| **how not to fool yourself** | [method/discipline.md](method/discipline.md) |

Read `method/discipline.md` before doing any measurement. Every rule in it
was learned by getting it wrong first.

## Layout

```
formats/    one document per file format, with the generated tables beside it
engine/     engine behaviour: tech stack, recovered formulas
builds/     which of the seven Sacred builds answers which question
method/     the techniques, and the mistakes that shaped them
log/        the append-only findings log
```

## The findings log

`log/autoresearch-results.tsv` — **830 rows**, one per investigated question,
append-only. Each row records what was asked, what was measured, and the
verdict, including the refutations. It stands in for the missing git history
of the analysis workspace.

**If you are about to test a hypothesis, grep it first.** It may already be
settled, or already dead.

## How to read the distilled documents

These are distilled from a much larger body of dated working notes that is
not published. Each document states what is true now. Where a claim was later
corrected, the correction follows as a blockquote:

> **Earlier belief.** Why it was wrong, and what measurement refuted it.

Those are kept deliberately. In this project the refutation is often the more
useful half — "do not retry this" saves more time than the original claim
would have.

Anything not stated is not claimed. Open questions are marked as open.

## Related

This is one of three repositories:

- [engine](../engine) — the reimplementation: an open Sacred Gold engine on
  Godot 4.7 that reads your own retail install.
- [tools](../tools) — the analysis and extraction tools, and the Python half
  of every parity gate.

## Licence

CC BY-SA 4.0 — see [LICENSE](LICENSE). Covers our prose and tables only; it
grants no rights in Sacred or Sacred Gold.
