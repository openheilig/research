# How retail wires itself together, and where the port stands

**Status:** Standing
**Purpose:** The one document that answers "what is linked to what" — retail's
own call order, each link's file, and whether this port has it. Written because
the project had eleven format documents and no map of how the formats meet.

Provenance: the new-game trace through `install/sacred` (findings log rows
954–963), plus the readers and gates named per row. Function addresses are
virtual (`vaddr = 0x08048000 + file_offset`).

---

## The chain, in retail's order

```
 [1] class chosen in the UI                 cUI_Character::executeAction  0x853EF6A
      |                                     byte [uiObj+0x16E] = slot 0..7
      v
 [2] templates/hero0N.ptx  ---copied verbatim, 512B at a time--->  Save/HeroNN.pax
      |                                     the fread/fwrite loop at 0x853FA45
      |                                     THE TEMPLATE IS NOT PARSED HERE
      v
 [3] savegame load                          sub_80AFCDE / chunk reader sub_80CCA20
      |                                     -> CharacterType, level, skills, items
      v
 [4] script tree load                       sub_8265BF6
      |   GetTypeName(CharacterType) -> "bin\TYPE_NPC_SERAPHIM"      sub_815B3A2
      |   StartCode -> FunkCode -> QuestCode -> QuestPoolCode
      |            -> DefPos -> Vectoren -> merc.bin -> treppe
      |   then every Vectoren name starting "Sector"/"Region" is registered
      |   then StartCode is executed once by the VM                  sub_826DA00
      v
 [5] sector Init                            sub_80DAFF4 -> sub_829F9D4
      |   runs the procedure named "Sector<cx><cyyy>Init"
      |   opcode 100 SpawnValues(50, lo, hi) -> marker object {+211=lo, +213=hi}
      v
 [6] sector Enter                           sub_80DB06C -> sub_829FAF4
      |   runs "Sector<cx><cyyy>Enter"; walks placements; fires triggers
      |   (music is NOT selected here -- see the sector-change path below)
      v
 [7] creature spawn                         sub_8180B22
      |   level = clamp(hero level, band) + rand()%2                 sub_81806DC
      v
 [8] stat build                             cCreatureHero::CalcResults  sub_820E04C
          base AT (+0x5A) = 0.5*(STR+DEX), base PA (+0x5E) = 0.2*STR + 0.8*DEX
          multipliers (+0xE6, +0xEA) start at 1.0f and every one of the eight
          skill slots folds in through the curve at sub_81F55B0
          AT = base * mult * ProzAW[difficulty]        sub_81FA5AA / sub_81FA622

 [9] sector CHANGE (not entry)              sub_80DB27C
          world/sectors.keyx env -> music id, climate, region
          -> cMSS::receive_event, then the chooser sub_84EADBA
```

The **text layer** hangs off every one of [4]–[8]: quest titles are plain text
in `vectoren.bin`, everything else is a key into `scripts/<lang>/global.res`,
reached through the name hash at `sub_80ACC3E`.

---

## Link by link, against this port

| # | Link | Retail's file | Ours | Gate |
|---|---|---|---|---|
| 1 | class → template | UI slot 0..7 | **n/a — no class-select UI**; `START_CLASS` is a constant | — |
| 2 | template → save | `templates/hero0N.ptx` | **yes** `formats/hero.gd` | `pax_check` |
| 3 | save → character | PAX `0xC7` | **yes** — level, xp, gold, 6 attributes, 8 skill slots | `pax_check` |
| 3b | save → inventory | PAX `0xC8` | **ids only**, not equipped | `pax_check` |
| 3c | save → quest bits | PAX `0xCE` | **understood, not persisted** — 5 tiers x 160 bits | `quest_check` |
| 4 | CharacterType → tree | `GetTypeName` table | **yes** — confirmed twice | `pax_check` |
| 4b | tree load | `startcode`/`funkcode`/`vectoren` | **yes** | `startcode_check`, `quest_check` |
| 5 | sector → Init proc | `Sector<cx><cyyy>Init` | **yes** | `spawnlevel_check` |
| 5b | `SpawnValues` → band | opcode 100 | **yes** | `spawnlevel_check` |
| 6 | sector → Enter proc | `Sector<cx><cyyy>Enter` | **partly** — hooks run for quests, not per-sector | `quest_check` |
| 6b | placements → world | `startcode.bin` | **yes** (`--npcs`) | `spawn_check` |
| 6c | sector → music | `world/sectors.keyx` | **yes** `formats/sectors.gd` | `sectorenv_check` |
| 7 | band + hero level → level | `sub_81806DC` | **yes** | `spawnlevel_check` |
| 7b | body id → creature | `creature.pak` | **yes**, all 86 bytes | `creature_check` |
| 7c | class pair → hostility | faction matrix | **yes** | `factions_check` |
| 8 | attributes+skills → AT/PA | `sub_81FA5AA` / `sub_81FA622` | **yes** — base and multiplier | `combat_check` |
| 8b | AT/PA → to-hit | `sub_428790` | **yes** | `combat_check` |
| 8c | damage vs resistance | undecoded | **no** — deliberately no formula | — |
| T | key → text | `global.res` | **yes** since row 954 | `resources_check` |
| T2 | composed key → text | VM variable substitution | **yes** `QuestLog.resolve_with` | `quest_check` |

## The render side

| Layer | Ours | Note |
|---|---|---|
| terrain, heights, per-corner light | **done** | `height_scale` is still an admitted guess of 1.0 |
| `floor.pak` overlay + mask blend | **done** | read off a live capture |
| statics, chains, painter order | **done** | 935 objects in sector 50,39 vs 777 heads |
| camera, projection, zoom steps | **done** | zoom steps recovered by driving retail under Xvfb |
| player mesh, armour, weapons | **done** | refuses rather than approximating |
| **player animation** | **done** | 5 of 7 bodies, each playing its own IDLE (rows 957, 963) |
| NPC / creature animation | **opt-in flags only** | not on in a default run |
| facing / heading | **built and unused** | `set_yaw` is called with `0.0` everywhere |
| idle vs walk vs attack | **selection done, switching not** | `Rigs` resolves a clip per ACTION (row 963); nothing changes clip at runtime yet |
| **HUD** | **done (row 962)** | retail's own rects and coordinates; gauges still open |
| sound, particles, water, weather | **none** | no screenshot impact |

---

## What is actually left for a 1:1 small-scale MVP

Seven items were listed here on 2026-08-16. **Six are closed** (rows 954–963);
what follows is the state after that pass.

| # | Item | State |
|---|---|---|
| 1 | HUD | **Closed.** The layout is a static 1887-entry sub-rect table at `0x880DC68` placed by `cUI_Taskbar2` onto a fixed 1024×768 canvas. `view/hud.gd` draws the console, wings, buttons, combat-art arc and both slot wings from retail's own art. **Except the life/mana gauges** — see Open. |
| 2 | Facing | **Open.** `set_yaw` is still passed `0.0`. See Open. |
| 3 | NPCs by default | **Open, deliberately.** `--npcs` places the scripted cast at real cells; turning it on by default changes frames that other gates md5, so it is a runbook decision rather than a code one. |
| 4 | Quest text on screen | **Closed.** The console shows the quest's own line; quest 74 reads *"The Soul of the Demon"* / *"Kill the demon, after Shareefa has summoned it."* |
| 5 | Composed keys in the VM | **Closed.** `QuestLog.resolve_with` substitutes from its own variables, and `SetVarBit` is now understood as a bit index, so the variables it reads are right. |
| 6 | The AT/PA base | **Closed.** `0.5·(STR+DEX)` and `0.2·STR + 0.8·DEX`. The MVP fight is 31%, entirely derived. |
| 7 | Clip selection | **Closed.** The action is readable from the clip name even though the character is not; all five buildable bodies now play their IDLE instead of whatever scored highest. |

## Open

- **The life and mana gauges.** All 46 functions of `cUI_Taskbar2` were
  enumerated: the class references no orb, globe or fill-bar art and computes
  no fraction or scissor rect. The orb-looking elements in `GUI_main_02` belong
  to the mercenary window. So the most recognisable part of the screen is
  deliberately not drawn rather than guessed.
- **Character facing.** `set_yaw` and `rig_placement.yaw` are built and unused.
  `ActorState.heading` is per-tick movement intent, not a facing, and the
  cell-space→yaw convention is uncalibrated — implementing it means inventing a
  constant that could be 180° wrong. The measurable route is a walk clip's root
  translation direction, which gives the model's own forward axis.
- **The damage/resolution step** — how damage meets resistance, criticals, and
  what the weapon-slot flag selects. To-hit and both ratings are recovered;
  this is what is left of a fight.
- **Hit points.** In no table read so far.
- **The 200× factor and the 5..95 clamp** are in the Windows binary's
  `sub_428790` and are *not* in the Linux build, whose display path computes
  `100·AT/(AT+PA)` through `sub_815D44C` with no level term. Recorded as a
  discrepancy between two binaries rather than resolved.
- **`height_scale`** in `sector_view.gd` is still an admitted guess of 1.0.
- **Eight of 3421 animation clips** do not decode.
