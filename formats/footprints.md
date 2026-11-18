# Footprints / collision bitmask

**Status:** Partial; Gold transfer reopened 2026-09-29.

**Correction:** the earlier “Solved” verdict transferred an Armalion predicate
to Gold without establishing its shipping consumer. The historical sections
below describe that investigation, not an accepted Gold implementation spec.
In particular, the claim that `byte26 != 1 && byte26 != 4` is correct for every
non-door Gold cell is withdrawn.

Gold LGP `0x080EE194`, ENG `0x00636C10`, and RUS `0x00637040` agree on a
different base predicate: resolve the support-aware cell; reject missing cells,
low-nibble `cell[31]` classes 1/2, or `cell[30] & 8`; otherwise allow. With
`cell[30] & 4`, a coordinate-keyed runtime trigger lookup can override that
answer using bit 0 of the resolved sixteen-byte record's byte 10. This is not
the Armalion static-mask intersection described below. The companion wrapper
LGP `0x080EE244` / ENG `0x00636D20` / RUS `0x00637150` adds resolved
class2 when the base predicate returned false. This is an independent extra
acceptance, including base bit8 rejection; it is not a universal bit8 blocker.
Caller choice still matters.

**2026-10-06 actual actor-consumer correction:** the independent direct chains
LGP`0x08249872→0x082454EA→0x0824565C→0x08230C90`,
ENG`0x004FB7E0→0x004F7CE0→0x004F7E10→0x004EBCA0`, and
RUS`0x004FBBF0→0x004F80F0→0x004F8220→0x004EC0B0` use an
actor-specific gate, not the unconditional class2 world companion. Class2
exceptions depend on actor+500 bit0x200000, active effect1094/corner conditions
or a special actor category; object+20 flags are a different field.
A guarded six-frame native opening/talk/held-travel observation retains
movement flags0 and a valid empty effect vector for both hero1 and novice679.
The bounded engine baseline therefore rejects class1 **and class2**; the
previous blanket class2 acceptance is withdrawn. A real movement consumer
crossed a class2 blocker before and refuses it after, retaining class0 travel.
Native privileges, coordinate overrides, occupancy and local alternative
candidate steering remain separate unqualified contracts.

The fresh-spawn port hold and continuous08 native hold are not state-aligned:
the latter follows authored conversation/acknowledgement at precise world
173766,135052, not the starting cell centre. A private derived-position
terrain control exits through the eastern lane, whereas the fresh-spawn
control meets class1. Its float32 position reconstructs173765,135051, so it is
not an exact native-state replay. Do not open those walls or substitute this
diagnostic for C1/C2 acceptance.

A bounded live LGP observation captured 24 entry/return pairs. Same height byte
0 produced both true and false; class-0 cells with height byte 1 returned true,
while class-1/2 cells returned false. All observed samples had flags 0 and
layer 0: trigger overrides, bit-8 rejection, support layers, and actor-specific
permissions were not behaviorally tested in that sampling pass.
[The revision audit](../engine/engine-revision-2026-09-29.md) preserves its
evidence and remaining gates; no engine implementation changed in that pass.

**2026-10-06 support integration:** ordinary creature movement's unowned
actor+44 relation selects the actual static/world support, not the focus
actor's grid or a dynamic inventory owner (tag0x400). Actor+36 is a byte-sized
support ordinal, not physical height. Prospective selection
LGP`0x080EE302` adjusts the support to an authored parent/ordinal or base;
missing/outside child records are not replaced by a matching rectangle.
The actual chapel probe resolves child758530 over base-class1. It keeps
3242,2509 and3243,2508 blocked, but admits the authored exit beyond the
overlapping zero-filled rectangle at3226,2520/2521. This is a bounded
engine/source result; native whole-route chronology and all override states
remain separate qualification gates.

The relation lifecycle is separate from focus trigger mutation: LGP
`00013.c:9385–9434,9520–9535` and independent Gold ENG
`00015.c:7472–7555,7635–7686` resolve actor-owned support, with trigger writes
focus-gated. The relation setter changes actor+44/tag0x400, not actor+36;
the remembered-trigger setter does not reset that ordinal either. The engine
therefore distinguishes a parent-disappearance release (reference0, ordinal
preserved) from the bounded authored STEP/base-output transition (0/0).
Actual NPC walk/contact regressions reach the exterior with base phase and
saved0/0 while focus raw state3 remains unchanged; a relation-only control
preserves ordinal1. The retained native novice694 is independently observed
with758530/1 at56259ms and0/0 at68255ms; the sparse interval does not isolate
the exact transition tick or justify attributing it to either setter.

## The system

A cell is walkable iff `(object_mask & type_mask) != 0`, where:

- `object_mask` is the **16-bit collision mask** carried by the *blocker
  static object* occupying a "door-bit" cell.
- `type_mask` is the **16-bit collision mask** of that object's *type*, from
  the global type table.

At least one shared bit means the object's collision class permits crossing;
zero shared bits blocks the cell. The two masks are independent: one lives on
the static-object record, the other on the type-table entry. Both are 16 bits;
the gate is the bitwise AND, not a magnitude comparison.

## How `cWorld::canWalk` consumes it

Traced in the Armalion debug build (identical layout to retail; RTTI-confirmed
earlier), called by `cCreature::canWalk`:

|function|Armalion non-US|Armalion US|
|---|---|---|
|`cWorld::canWalk`|`0x488320`|`0x411370`|
|`cCreature::canWalk` (caller)|`0x429040`|`0x42B770`|

Decode:

1. **Door-bit test.** If `cell[0] & 4` is set (the door bit = `world-sectors.md`
   cell record `+0x1e` bit 2), the cell is gated by a blocker object:
   - Walk the cell's static-object chain from `cell + 0x04` (the head static
     pointer). For each node, test runtime struct flag `struct + 0x08` for bit
     `0x200` — that flag marks a *blocker*.
   - For the blocker found, read its **16-bit collision mask** at `struct + 0x2b`
     and its **type id** (`u32`) at `struct + 0x27`.
   - Index the global type table by that type id; read the type's **16-bit
     collision mask** at `entry + 0xa` (`sub_433D10` = `*(v5 + 5)`, where `v5`
     is the resolved entry).
   - Walkable iff `(object_mask & type_mask) != 0`.
2. **Non-door cells.** Fall through to the terrain-height heuristic:
   `byte26 != 1 && byte26 != 4 && cell_int >= 0`. This is the rule that covers
   ~91% of ground and was the only branch ported before W2; it is correct for
   every non-door cell, which is why the field scan saw no better predictor —

   the door-bit branch was the missing ~9%, and it is a mask lookup, not a
   geometry test.

## static.pak record layout (relevant fields)

Per `static.pak` 64-byte record (row 227 census layout):

|offset|field|
|---|---|
|`+0x00`|self index|
|`+0x04`|type id (links to the type table)|
|`+0x08`|flags (bit `0x200` = blocker)|
|`+0x0e`|ox (absolute iso screen x)|
|`+0x12`|oy (absolute iso screen y)|
|`+0x27`|type id — *duplicate of +0x04 as read by the door-bit walk* (u32)|
|`+0x2b`|**16-bit collision mask**|

Offsets `+0x27`/`+0x2b` sit in the `0x0c–0x3f` band that Wave-1's byte census
found carrying real, non-`0xCD` data — they are the part of the record the
earlier scan could not name. The mask at `+0x2b` is the "footprint" the W2 name
refers to: not a polygon, but a 16-bit collision class word.

## Type table

- The type id indexes a type table that is a **member of the world object**.
  `cWorld::canWalk` calls `sub_404179` (= `jmp sub_486740`) with `this` = the
  world, so the entry-array base is at `world + 4` (`*(this+1)`, via
  `sub_48BDB0` -> `sub_40394A` -> `sub_40127B` -> `sub_48B4E0`), the entry count
  at `world + 0x388` (`sub_401A3C(this + 0x388)`), and the **stride is 16
  bytes**. The guard `id < table_size` (`sub_486740`) rejects out-of-range ids
  before any mask read.
- Per-entry **16-bit collision mask** at `entry + 0xa` (`sub_433D10` =
  `*(entry + 5)` read as a WORD).

## Port implementation (proposed, not yet committed)

`godot-port/world/walkable.gd` `_terrain_open` currently reads cell `+0x1a`
(the disproven second-height corner) for door-bit cells and has an unported
static-object chain. Row 1151 specifies the fix: replace the `+0x1a` read with
the door-bit (`cell[0] & 4`) walk of the `+0x04` static chain for flag `+0x08`
bit `0x200`, read mask `+0x2b` / type `+0x27`, look up the type-table mask
`+0xa`, and allow iff `(object_mask & type_mask) != 0` (block iff it is zero).
This is an engine edit and is **not** made in this research-only pass; it is
queued behind a user go-ahead (the phase is analysis-only).

## Source and serialization

- **New-world source:** `.\\WORLD\\TRIGGERS.PAK` (row 1156). World init
  `sub_4864D0` calls `sub_402347(this, ".\\WORLD\\TRIGGERS.PAK")`, resolving
  to loader `sub_489A60`. It opens `rb`, reads the header and type count,
  reserves the table at `world + 0x388`, then loops `fread(entry, 0x10, 1)`
  and copies all four dwords unchanged. File entry `+0xa` is therefore the
  runtime type-mask `+0xa`; no derivation occurs. This corrects row 1154.
- **Save mirror:** `sub_48A820(this, FILE*)` writes a `0x300`-byte header plus
  the same `size` 16-byte entries. Savegame tag 129 reads that serialized
  mirror; new games read `TRIGGERS.PAK`.

## Open

The Gold world-admission helper is cross-build recovered and its base branch
has live positive/negative witnesses. Still open: support selection,
caller-specific class-2 handling, actor-specific traversal, and complete
path/interaction behavior. The historical Armalion-only closure below does
not close these Gold requirements.

**Coordinate-trigger index population — answered 2026-09-29 (E2/NAV).**
A corpus-wide call census finds exactly ONE writer pair for the lookup map
at `0x8BB0148`: `sub_867444A` (clear) then `sub_86796DA` (insert), both in
the world-state loader (`linux1002/chunks/00025.c:13946-13952`), which reads
`8*v47` stream bytes as dword pairs `(coordinate_key, record_index)` and
inserts each. Consequences, both testable:

- On a NEW game the map is empty; `sub_8650744` misses and `0x80EE194`
  preserves its base answer. The bit-4 override is therefore SAVE-DRIVEN
  state, not sector-authored data — consistent with the live observation
  where all 24 samples had `cell[30] & 4 == 0`.
- A save that carries override pairs restores them verbatim on load; the
  port's equivalent belongs in the P1 save schema (trigger-state block),
  keyed by the same `(x<<18)|(y<<4)|(layer>1?layer:0)` packing.

The same loader block also rebuilds `0x8BB0110` (key → dword vector) and
`0x8BB0168` (88-byte records); `chunk 00017.c:32524` inserts the treppe
`(level<<26)|(y<<13)|x` packing into a DIFFERENT map (`v122+7494`), so the
packed-coordinate key scheme is an engine convention, not unique to this
lookup.

Still open for W1 implementation: which caller predicate each movement kind
uses (base `0x80EE194` vs companion `0x80EE244`), support resolution, and
per-actor permissions — with allowed/denied runtime witnesses per the
implementation plan's NAV ticket.

## Behavioural confirmation

- **Behaviourally confirmed** by existing retail-oracle controls (correction
  row 1155): tent door cell `(3422,1844)` was entered (row 625); three chapel
  exterior-to-interior crossings were archived (row 626); and the controlled
  nine-run campaign reached the doorway 3/3, crossed outward 3/3 and inward
  2/2, while all six runs that never reached it produced zero transitions
  (row 676). These independent observations agree with the corrected assembly
  polarity: a non-zero mask intersection is walkable.
