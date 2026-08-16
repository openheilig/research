# How retail wires itself together, and where the port stands

**Status:** Standing
**Purpose:** The one document that answers "what is linked to what" — retail's
own call order, each link's file, and whether this port has it. Written because
the project had eleven format documents and no map of how the formats meet.

Provenance: the new-game trace through `install/sacred` (findings log rows
954–957), plus the readers and gates named per row. Function addresses are
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
      |   runs "Sector<cx><cyyy>Enter"; walks placements; fires triggers;
      |   selects sound/music (sub_83C58C6 group)
      v
 [7] creature spawn                         sub_8180B22
      |   level = clamp(hero level, band) + rand()%2                 sub_81806DC
      v
 [8] stat build                             cCreatureHero::CalcResults  sub_820E04C
          AT (+0xE6) and PA (+0xEA) start at 1.0f, then every one of the
          eight skill slots multiplies in through the curve at sub_81F55B0
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
| 4 | CharacterType → tree | `GetTypeName` table | **yes** — confirmed twice | `pax_check` |
| 4b | tree load | `startcode`/`funkcode`/`vectoren` | **yes** | `startcode_check`, `quest_check` |
| 5 | sector → Init proc | `Sector<cx><cyyy>Init` | **yes** | `spawnlevel_check` |
| 5b | `SpawnValues` → band | opcode 100 | **yes** | `spawnlevel_check` |
| 6 | sector → Enter proc | `Sector<cx><cyyy>Enter` | **partly** — hooks run for quests, not per-sector | `quest_check` |
| 6b | placements → world | `startcode.bin` | **yes** (`--npcs`) | `spawn_check` |
| 6c | sector → music | `sndprofiles.pak` | **no code at all** | — |
| 7 | band + hero level → level | `sub_81806DC` | **yes** | `spawnlevel_check` |
| 7b | body id → creature | `creature.pak` | **yes**, all 86 bytes | `creature_check` |
| 7c | class pair → hostility | faction matrix | **yes** | `factions_check` |
| 8 | skills → AT/PA | `sub_81F596E` + `balance.bin` | **curve yes, BASE unrecovered** | `combat_check` |
| 8b | AT/PA → to-hit | `sub_428790` | **yes** | `combat_check` |
| 8c | damage vs resistance | undecoded | **no** — deliberately no formula | — |
| T | key → text | `global.res` | **yes** since row 954 | `resources_check` |
| T2 | composed key → text | VM variable substitution | **`compose()` exists, VM does not call it** | `resources_check` |

## The render side

| Layer | Ours | Note |
|---|---|---|
| terrain, heights, per-corner light | **done** | `height_scale` is still an admitted guess of 1.0 |
| `floor.pak` overlay + mask blend | **done** | read off a live capture |
| statics, chains, painter order | **done** | 935 objects in sector 50,39 vs 777 heads |
| camera, projection, zoom steps | **done** | zoom steps recovered by driving retail under Xvfb |
| player mesh, armour, weapons | **done** | refuses rather than approximating |
| **player animation** | **done (row 957)** | 5 of 7 class bodies; 2 build no rig |
| NPC / creature animation | **opt-in flags only** | not on in a default run |
| facing / heading | **built and unused** | `set_yaw` is called with `0.0` everywhere |
| idle vs walk vs attack | **no** | one clip per mesh, looped forever |
| **HUD** | **none — zero lines** | the single largest screenshot delta |
| sound, particles, water, weather | **none** | no screenshot impact |

---

## What is actually left for a 1:1 small-scale MVP

Ranked by screenshot-and-behaviour delta per unit of cost. The first two are
not research.

1. **HUD.** ~25–30% of a Sacred screenshot by area, and there is no code. The
   art is in `texture.pak`, which is already decoded — `view/cursor.gd` shows
   the name-based lookup. No new reverse-engineering required.
2. **Facing.** `set_yaw` exists and is passed `0.0` at every call site. A crowd
   all facing one way reads as broken instantly.
3. **NPCs on by default.** `--npcs` places the scripted cast at real retail
   cells. An empty village is not what retail looks like. This is a flag.
4. **Quest text on screen.** The prose now resolves; nothing displays it.
   Depends on (1).
5. **Composed keys in the VM.** `Resources.compose()` is written; `ScriptVM`
   must substitute the variable before looking a key up.
6. **The AT/PA base.** The multiplier chain is recovered and the base is not.
   Until then a fight's *numbers* are invented even though its levels are real.
7. **Clip selection** (idle/walk/attack). Needs a clip-naming decode; this is
   the one research item in the list.

## Open

- The PAX section → subsystem map. Sections `0xC3 0xC4 0xCA 0xCB 0xCD 0xCE` are
  framed and inflate correctly, and what consumes each is not recovered. Leads:
  `sub_80AFCDE`, `sub_80B33FC`, `sub_80CCA20`.
- The two difficulty tables at `0x8B89BA8` / `0x8B89DAC` that shift the level
  band. Not in any shipped file.
- The base attack and defence ratings — see
  [combat-formulas.md](combat-formulas.md).
- What selects music on sector entry; the `sub_83C58C6` group is located and
  its input is not.
