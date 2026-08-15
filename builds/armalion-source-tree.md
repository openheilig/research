# Sacred's original source tree, from the Armalion debug build

**Status:** Read
**Purpose:** The module layout, file names and class/method names of the
engine as its authors wrote it — recovered from assert strings that only a
debug build carries.

Tool: `tools/binary/armasource.py`. Table:
[armalion-source-tree.tsv](armalion-source-tree.tsv).
Binary: `analysis/builds/armalion-2001-12-11/armalion.exe`, a prerelease dated
two years before retail.

## Why this is readable at all

The 2001-12-11 Armalion build is a **debug** build. Its `assert`s survived, and
an MSVC assert emits three literals: the invariant text, the
`C:\Projects\Armalion\armaSource\…` path of the file it sits in, and a
`cClass::method()` naming the function. Retail is a release build and has
none of them.

The association between a method and its file is not inferred. The linker
emits a translation unit's string literals **together**, so walking the string
table in file-offset order and attributing each `cClass::method()` to the
nearest preceding path recovers it. That is an ordering property of the object
file.

**76 source files, 259 methods, 80 classes.**

## The module tree

```
armaSource/
  armalion.cpp
  kernel/            events.cpp  kernel.cpp
  engine/            engine.cpp  engineevt.cpp  world.cpp  sector.cpp
                     view.cpp  view1.cpp  view2.cpp  view3.cpp
    calendar/        calendar.cpp
    granny/          grannymgr.cpp  modelmgr.cpp
    kampaign/        quest.cpp
    scenery/         scenery.cpp
    weather/         weather.cpp
    object/          objectmgr.cpp
      base/          object3d.cpp  itemtype.cpp  objectfactory.cpp
      creature/      creature.cpp  creature_main.cpp  creature_states.cpp
                     creature_moving.cpp  creature_fighting.cpp
                     creature_magic.cpp  creature_collision.cpp
                     creature_generator.cpp  creature_map_scan.cpp
      items/         item_weapon.cpp  item_armor.cpp  item_arrow.cpp
                     item_potion.cpp
      inventory/     inventory.cpp
      sfx/           sfx.cpp
  scripts/           scriptcompiler.cpp  scriptinterpreter.cpp  scriptrpc.cpp
  library/           dx7/ file/ formats/ generic/ math/ resource/ timer/
  ui/                console.cpp  commands/  controls/  windows/
  particle/  sound/  movie/
```

`library/dx7` is the whole renderer and is the one module the port replaces
outright rather than reproduces. `engine/`, `engine/object/` and `scripts/`
are the parts that have to behave identically.

## What carries over to retail

Armalion is a prerelease. A name here is **evidence about Armalion and a
hypothesis about retail** — except where retail's own RTTI still carries the
class, which is the check the tool reports:

> **25 of 80 classes are also in retail's RTTI**: `cCreature`, `cWeapon3D`,
> `cArmor3D`, `cArrow3D`, `cObject3D`, `cObjectFX`, `cSector`, `cWorld`,
> `cWorldView`, `cEngine`, `cResourceMgr`, `cParticleSystem`, `cFsFont`,
> `cFile_cached`, `cFile_memoryMapped`, `cEvent_idle`, `cUI_Character`,
> `cUI_Chest`, `cUI_Control2`, `cUI_Diary`, `cUI_Manager`, `cUI_Megamap`,
> `cUI_OverviewMap`, `cCommand_armaPlay`, `cCommand_armaStop`.

Those 25 span every module that matters, so the tree is retail's tree too, at
the granularity of modules and classes. Individual *method* names are not
covered by that check and must not be transplanted onto retail addresses.

## The creature state machine

`engine/object/creature/` is 53 of the 259 methods and is the simulation the
port has to reproduce. The states name themselves:

| file | what it holds |
|---|---|
| `creature_states.cpp` | `state_Main_Idle`, `state_Dying`, `state_proc_Jumping[_init]`, `state_Main_ItemDo_{container,inventory,useable}`, `getMotionIdle`, `moveDoorPosition`, `mountAt_getParam` |
| `creature_moving.cpp` | `state_proc_Moving_{init,default,target_position}`, `state_proc_AngleMoving_{init,moveEvent}`, `state_Main_ItemDo`, `typeToMotion` |
| `creature_fighting.cpp` | `state_Attacking`, `state_Fighting_init`, `sState::getParam` |
| `creature_collision.cpp` | `collision_detect`, `state_Fighting` |
| `creature_magic.cpp` | `state_proc_Magic_{Interpret,Send}`, `cCreatureQueue::pop` |
| `creature_generator.cpp` | `cCreatureInfo::getWeaponSkill`, `state_proc_Magic` |
| `creature.cpp` | `levelCreate/Enter/Exit/Log`, `subtractLife`, `getStaminaRecovery`, `attachEquipment`, `grnWearEquipment`, `getSlotAttachBoneName`, `equipment_getSlotType`, `inventory_putItem`, `exeScript`, `setScript`, `handle_event`, `render` |

So combat is a **state machine with a separate collision pass**, not a
function; `state_Fighting` appears in `creature_collision.cpp` while
`state_Attacking` is in `creature_fighting.cpp`. The still-open combat
*resolution* step in [../engine/combat-formulas.md](../engine/combat-formulas.md)
is a transition inside that machine, which is why looking for it as one
arithmetic function has not found it.

`cCreatureInfo::getWeaponSkill` is the Armalion-side name for the lookup that
[../engine/combat-formulas.md](../engine/combat-formulas.md) § *The family
prefixes* recovers in retail as skill type → balance family.

## The script VM

`scripts/` gives the interpreter's own vocabulary and its invariants:

| string | what it states |
|---|---|
| `fp->code[ip]<256` | the opcode is a **byte** indexing a byte array; `fp` is a frame, `ip` an instruction pointer |
| `chunk.type==CHUNK_TYPE_ACS` | the compiled-script container is chunked and the script chunk is `ACS` |
| `hdr.ver==2` | that container is at version 2 |
| `serialID<exportFunctions.size()` | the API is dispatched by **index into a vector**, not by a fixed switch |
| `invalid_function`, `Invalid` | the interpreter's own error text for a bad id |

Classes: `cScriptCompiler` (`parseStatement`, `parseArmaStatement`,
`parseResource`, `loadScriptedSequenceR`, `save`, `saveResources`, `load`),
`cScriptVM::load`, `cScriptInterpreter` (`load`, `interpreter`,
`findScriptFunction`), `cScriptLoader::getResource`.

`cScriptCompiler` shipped **inside the game**, so Armalion could compile its
own scripts at runtime. That is why `RESOURCE.PAK` is reachable from
`cScriptLoader::getResource` — see [../formats/global-res.md](../formats/global-res.md).

## Open

- **The script API id space is not joined to retail's opcodes.** The binary's
  name table holds ~85 API names; `armalion-script-api.tsv` records 62 of them
  with handler addresses. Retail's dispatcher has 141 entries and the ids do
  **not** correspond — Armalion 67 is `printTextResource` while retail 67 is a
  variable setter, and the same disagreement holds for every sampled pair. The
  Armalion API is a **vocabulary** for naming retail opcodes, not a key to
  them. A join needs a shared invariant, not a shared index.
- Method names are Armalion's. Only the 25 class names above are checked
  against retail; nothing here licenses renaming a retail address.

---
Provenance: `tools/binary/armasource.py`, which fails if the ordering trick
binds fewer than 40 files or 60 methods; the retail overlap checked against
`analysis/rtti/lgp_linux_vmethods.tsv`.
