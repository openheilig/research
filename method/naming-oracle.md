# The naming oracle

**Status:** Standing
**Purpose:** How function naming became a mechanism instead of a heuristic.

The single technique that changed the economics of this project. It turns
function naming from a heuristic into a mechanism.

## The mechanism

The Armalion debug build prints trace lines of the form
`cClass::method(...) fmt`. **108 such strings** match `^c[A-Z]\w*::\w+\(` in
the string table.

**Each is referenced by exactly one function: the one it names.** So

```
string → xref → enclosing function
```

is an *unambiguous automatic naming mechanism*, not a guess.

Validated on a sample of eight. **8/8 had exactly one xref each, each landing
in a distinct, previously unnamed function:**

| String | → | Size |
|---|---|---|
| `cCreature::receive_event(DAMAGE) Ref[%d] ALVL[%d] DLVL[%d] AT[%d] PA[%d]` | `sub_41FA30` | 0x89f |
| `cCreature::subtractLife() mlvl[%d] hlvl[%d] exp[%d]` | `sub_425750` | 0x513 |
| `cCreature::canWalk() stateRef[%d] parentRef[%d,%d] trigger[%d,%d]` | `sub_429040` | 0x3b9 |
| `cCreature::inventory_putItem()` | `sub_423B50` | 0x97 |
| `cEngine::cEngine() Campaign mode hero[%d] class[%d]` | `sub_45EA80` | 0x926 |
| `cEngine::creature_morph(ref:%d, desttype:%d)` | `sub_466520` | 0x104 |
| `cWorld::getParentObject() Invalid State for Ref[%d] State[%d]` | `sub_487010` | 0xcc |
| `cScriptInterpreter::findScriptFunction() Unknown Script function: %s!` | `sub_4C5960` | 0x99 |

## Why 108 of 7122 is enough

It reaches only ~1.5% of the unnamed functions — **but they are the right
1.5%.** The set covers combat resolution, XP and levelling, pathing,
inventory, engine construction, both engine threads (`simThread`,
`renderThread`), world and sector loading, minimap, fog and weather, the
script interpreter *and* its compiler, the object manager, item data, the
Granny model loader, and the Miles sound wrapper.

Those are precisely the subsystems a reimplementation has to get right.

## The one mechanical gotcha

The build used MSVC `/INCREMENTAL`, so `0x401000`–`~0x406000` is a
**jump-thunk table**: each entry is a single `jmp rel32` to the real body,
1:1. Confirmed — `sub_402E05` and `sub_402888` are each one instruction.

Every intra-module call in decompiled output goes through a thunk, so
**thunk → body is a mandatory one-hop resolve before decompiling any
callee.** Mechanical, but skip it and every callee looks like a 5-byte stub.

This gotcha is build-specific: the USA demo build has no thunk table, so it
does not apply there.

## The mechanism has more reach than 108

`^c[A-Z]\w*::\w+\(` was the pattern. Relaxing it to any
`identifier::identifier(` finds **275** distinct names, because the engine
also asserts from classes that break the `c`-prefix convention (`dxDriver7`,
`sState`, `BcCreatureQueue`, `granny_transform_state`) and from destructors.

`tools/binary/armasource.py` binds **259** of them to the source file they sit
in, by walking the string table in file-offset order — a translation unit's
literals are emitted together, so the nearest preceding
`armaSource\…\*.cpp` path names the file. That turns the oracle from a list
of function names into a **module map**: 76 files, 80 classes, 25 of which are
still in retail's RTTI. See
[../builds/armalion-source-tree.md](../builds/armalion-source-tree.md).

## The same rule works on opcodes

The mechanism generalises past function naming. A string names a *script
opcode* under the same uniqueness condition — reachable from exactly one
handler — with one addition, because opcode handlers are reached through a
jump table rather than by name and the string may live in a callee:

> **The behavioural profile must agree.** Opcode 79 uniquely reaches
> `cCreature::equipment_reset() EquipmentRef unknown?!`, which reads as a
> name. Its operands are `(res:TEXT, small id)` carrying quest prose. The
> string is a deeper callee's and 79 stays unnamed.

Six opcodes were named this way and marked `[reached]` to keep them separate
from those named by a handler's own strings. See
[../formats/script-bytecode.md](../formats/script-bytecode.md).

## Worked result

[../engine/combat-formulas.md](../engine/combat-formulas.md) is the to-hit
formula recovered end to end through this oracle, then independently
confirmed in a second binary.

---
Provenance: validated on an 8/8 sample in this project's private analysis
workspace.

