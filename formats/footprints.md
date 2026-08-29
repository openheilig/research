# Footprints / collision bitmask

**Status:** Solved (2026-08-27, rows 1148–1156). Supersedes the "geometric
footprint" hypothesis that drove the original W2 framing — outdoor walkability
is **not** a per-object footprint polygon tested against the player point. It
is a 16-bit collision-class **bitmask intersection**. Closes open-questions
row 89 (opened 2026-08-17, row 990), which measured the old `byte26 != 1 &&
byte26 != 4` heuristic at a 91.01% base rate and found no cell field predicted
the navmesh better. The mask IS the answer the field scan could not find.

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

Nothing open in the collision-bitmask decode. The port implementation remains
integration work, not research.

## Behavioural confirmation

- **Behaviourally confirmed** by existing retail-oracle controls (correction
  row 1155): tent door cell `(3422,1844)` was entered (row 625); three chapel
  exterior-to-interior crossings were archived (row 626); and the controlled
  nine-run campaign reached the doorway 3/3, crossed outward 3/3 and inward
  2/2, while all six runs that never reached it produced zero transitions
  (row 676). These independent observations agree with the corrected assembly
  polarity: a non-zero mask intersection is walkable.
