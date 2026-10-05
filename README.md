# OpenHeilig — research

Reverse-engineering findings on *Sacred Gold*: its file formats, script
bytecode, balance data and engine behaviour.

Everything here is prose and tables we wrote. There are no bytes from the game
in this repository — no extracted assets, no decompiler output, no symbol
tables, no memory dumps. Those stay in a private workspace by design, and the
`.gitignore` is an allowlist so they cannot arrive by accident.

Public development research, not a claim of a finished game. Current engine
behavior and the approved 0.0.1 boundary are recorded in the
[engine status](https://github.com/openheilig/engine) and
[playable Seraphim milestone](https://github.com/openheilig/engine/blob/main/docs/milestone-0.0.1.md).
The September29 audit is historical evidence; later implementation does not
turn its open contracts into completed features.

## Start here — and what state each answer is in

| I want… | Read | Status |
|---|---|---|
| **to know what a file on the disc is** | [formats/install-inventory.md](formats/install-inventory.md) | Read |
| to read a `.pak` | [formats/pak-containers.md](formats/pak-containers.md) | Solved |
| to load the world | [formats/world-sectors.md](formats/world-sectors.md) | Solved |
| sound, music, sound selection | [formats/install-inventory.md](formats/install-inventory.md) | Read |
| models, skeletons, animation | [formats/granny-grn.md](formats/granny-grn.md) | Read |
| hero saves | [formats/pax-saves.md](formats/pax-saves.md) | Read |
| script bytecode and opcodes | [formats/script-bytecode.md](formats/script-bytecode.md) | Read |
| the rules and tunables | [formats/balance-bin.md](formats/balance-bin.md) | Read |
| to draw the interface | [formats/ui-taskbar.md](formats/ui-taskbar.md) | Partial |
| how combat resolves | [engine/combat-formulas.md](engine/combat-formulas.md) | Partial |
| what the Linux port is built on | [engine/tech-stack.md](engine/tech-stack.md) | Solved |
| whether RAD's own runtime can check our `.GRN` decode | [engine/granny-runtime-oracle.md](engine/granny-runtime-oracle.md) | Blocked |
| which binary to open | [builds/build-survey.md](builds/build-survey.md) | Solved |
| how to name a function | [method/naming-oracle.md](method/naming-oracle.md) | Standing |
| which instrument to reach for | [method/instruments.md](method/instruments.md) | Standing |
| **how not to fool yourself** | [method/discipline.md](method/discipline.md) | Standing |
| everything still unanswered | [open-questions.md](open-questions.md) | — |

Read `method/discipline.md` before doing any measurement. Every rule in it
was learned by getting it wrong first.

## Layout

```
formats/            one document per file format
formats/generated/  machine-written tables -- regenerate, never edit
engine/             engine behaviour: tech stack, recovered formulas, oracles
builds/             retail/prerelease build roles and independent references
method/             the techniques, and the mistakes that shaped them
log/                the append-only findings log
open-questions.md   every open item from every document, in one list
```

## The shape of a document

Every document in this repository, and every `README` in
[tools](https://github.com/openheilig/tools), opens and closes the same way, so its state is readable
without reading its body:

```markdown
# <subject>

**Status:** <one word from the table below>
**Purpose:** one sentence — the question this answers.

…body…

## Open        ← what is still unanswered, or the words "Nothing open"

---
Provenance: the decoders, tools and log rows this rests on.
```

| Status | Means |
|---|---|
| **Solved** | Two independent decoders agree. Nothing open. |
| **Read** | The format has been decoded/read with named gaps; production integration is a separate engine claim. |
| **Partial** | The core is understood; the gaps are named under `## Open`. |
| **Blocked** | A stated obstacle, and what would remove it. |
| **Open** | Not started. |
| **Standing** | Not a result — a rule or a technique that stays true. `method/` only, and exempt from `## Open`. |
| **Generated** | Machine-written. Regenerate it; do not edit it. |

`## Open` is mandatory for every document that reports a result. A document
with neither an open item nor the words "Nothing open" is unfinished, and that
is a checkable claim rather than a matter of taste.

## The findings log

`log/autoresearch-results.tsv` is the append-only findings ledger, one row per
result. Cite the monotonic finding **id**, not the physical line number or a
remembered row total. Rows preserve observations, corrections and refutations;
they do not by themselves establish the current engine's behavior.

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

- [engine](https://github.com/openheilig/engine) — the reimplementation: an open Sacred Gold engine on
  Godot 4.7 that reads your own retail install.
- [tools](https://github.com/openheilig/tools) — the analysis and extraction tools, and the Python half
  of every parity gate.

## Licence

CC BY-SA 4.0 — see [LICENSE](LICENSE). Covers our prose and tables only; it
grants no rights in Sacred or Sacred Gold.
