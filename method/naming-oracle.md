# The naming oracle

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

## Worked result

[../engine/combat-formulas.md](../engine/combat-formulas.md) is the to-hit
formula recovered end to end through this oracle, then independently
confirmed in a second binary.

---
Provenance: validated on an 8/8 sample in this project's private analysis
workspace.

