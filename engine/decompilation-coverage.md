# What is not decompiled, and what it would take

**Status:** Partial
**Purpose:** How much of the retail engine has a recovered identity, which
routes to more of it are open and which are exhausted, and what a claim has to
survive before it counts.

Measured against `install/sacred`, the LGP Linux binary the port is verified
against. Numbers are from 2026-08-15; the method is repeatable and stated so
the numbers can be re-derived rather than believed.

## Coverage

| | Count |
|---|---|
| Direct-call targets in `.text`, excluding PLT | 11,406 |
| Virtual targets (vtable slots) | 2,413 |
| **Callables (union)** | **13,716** |
| **With any recovered identity** | **2,413 — 17.6%** |
| With none | 11,303 |

By code volume the picture is worse: the unidentified region is **84.1% of the
bytes**, because unidentified frames are larger on average than identified
ones.

Even 17.6% flatters us. A vtable-derived name is `class` plus `slot number`;
it says which class owns the method and where it sits in the table, not what
the method does.

> ~~1901 of 8434 functions carry an identity — 22.5%.~~ **The denominator was
> not functions.** `.eh_frame` FDEs are *frames*, and one frame can cover a
> whole switch-driven interpreter. Five addresses recovered by hand in earlier
> work — the script interpreter's tag handlers and its dispatcher — all turned
> out to sit *inside* other entries, three of them within a single 14,827-byte
> frame. The 88,666-byte "largest function" is a frame too. Call targets are
> the honest denominator.

## The vtable route is saturated, not incomplete

`cCreature` exposes **13 virtual slots**. The Armalion symbol catalogue names
**95 `cCreature` methods**. The other 82 are non-virtual: no vtable will ever
contain them. `cQuestMgr`, `cInventory` and `cTrigger` have no vtable at all.

So the 2,413 figure is not a work-in-progress number that patience will grow.
It is approximately everything this technique can produce, and reaching the
rest requires a different one.

## The route that is open

The retail Linux binary carries its own debug strings, and they had never been
measured — the technique was developed on a debug build of a 2001 prerelease
and assumed to need a cross-build transfer to be useful here.

| In `install/sacred` | Count |
|---|---|
| `cClass::method` strings in `.rodata` | 248 |
| **Referenced exactly once from `.text`** | **207** |
| Referenced zero times | 20 |
| Referenced more than once | 21 |

The 207 resolve to **131 distinct `class::method` names across 53 classes**,
and they bind straight to addresses in the binary the port is checked against.
All 207 use sites fall inside a known frame. They reach 133 frames — 436 KB,
7.2% of the code — and it is *non-virtual* code, precisely what the vtable
route cannot see.

The scan is: find `c[A-Z]\w+::\w+` inside `.rodata`, compute each string's
virtual address, and count occurrences of its 4-byte little-endian address in
`.text`. One occurrence means one use site.

## What a claim has to survive

**A string's frame is a lead, not a result.** Where a method is named by more
than one string, the strings must agree on a frame. 35 methods qualify:

- 31 agree.
- 4 disagree — `cEngine::cEngine`, `cObjectManager::create` (three frames),
  `cWeapon3D::toggleVisuals`, `cArrow3D::toggleVisuals`.

**11% disagreement**, from inlining and adjacent frames. That is the measured
error rate of the inference, and it is why a name needs a second arm:

1. **A second source.** 34 of the 131 also appear in the Armalion symbol
   catalogue of 789 game-class names.
2. **Behaviour.** The named function must be observed doing what the name
   says — reached under a run that should reach it, absent from a run that
   should not.

The 97 names present only in the Linux binary are **not refuted** by their
absence from the 2001 catalogue: that build predates them, and a source that
could not contain the answer cannot argue against it.

## Where the value is

Ranked by what the engine still lacks, against how much the oracle already
hands over with an address attached:

| Subsystem | Named methods with an address | Notable |
|---|---|---|
| Objects and placement | 19 | `create`, `allocate`, `load`, `loadHero`, `ParkCreature`, `getHeightInfo` |
| Models and animation | 18 | `getModel`, `getMotion`, `grnUsePak`, `renderShadow`, `bindTextures` |
| Engine core and save | 13 | `simThread`, `initGame`, `save`, `load`, `dropItem` |
| Quests and scripts | 9 | `parseStatement`, `parseResource`, `loadScriptedSequenceR`, `enterRegion` |
| Creatures and combat | 8 | `equipment_equip`, `pickupItem`, `receive_event`, `WakeUp` |
| Inventory | 8 | `moveItem`, `autoSortItems`, `fixInventory`, `getPotionRef` |
| UI | 6 | `runThread`, `executeAction`, `tooltipCreate` |
| Sound | 4 | `playMusic`, `playSFX`, `advanceTime` |

> ~~The script group is the one that unblocks written-down work: the bytecode's
> format is closed and its semantics are not.~~ **Refuted the same day, row
> 844.** Counting the `name` column of the static opcode table alone gives 34
> of 141 and looks like a large gap. Counting the behaviour table with it gives
> **116 opcodes with a recovered meaning and 25 without — and all 25 have zero
> records in the shipped scripts.** Every one of the 1,360,204 records the game
> actually runs is covered. Script semantics is not a bottleneck.

What the same recount exposes instead: those 116 readings are **inferences**
from operand shapes and handler statics. None has been confirmed by observing
the running game. The gap is not meaning, it is verification.

## Open

No name and no opcode reading has been confirmed by **behaviour**. Cross-source
agreement covers 34 of 131 names; the behavioural arm covers nothing at all.
Until a claim survives both, this document reports a method and a ranking, not
a set of established facts.

The cheap behavioural oracles were checked and do not exist: retail's
`debug.log` emits `makeSpawnInfo` once, as a `TypeManager` startup phase
marker, not per spawn. What remains is live-process observation, which needs
the game driven to a chosen sector — and that route is already recorded as
abandoned, so establishing the behavioural arm means first solving navigation
or finding a different observation point.

The first experiment to run, stated so it can be argued with before it costs
anything: opcode 115 declares spawn groups that only opcode 51 reads, and every
one of its 184 distinct ids is a `creature.pak` id. Predict the creature set
for a specific sector from the script data, observe what the running game
actually instantiates there, and — this is the half that makes it a test —
predict a *different* set for a second sector and check the observation
changes with it. Without that control the first sector proves nothing, because
common creatures appear nearly everywhere.

---
Provenance: `tools/binary/ehframe_funcs.py` and `vtables.py` for the
inventories; the call-target scan and the string-reference scan are described
above in full so they can be re-run; findings log row 843.
