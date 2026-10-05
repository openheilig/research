# How retail wires itself together, and where the port stands

**Status:** Standing
**Purpose:** The one document that answers "what is linked to what" — retail's
own call order, each link's file, and whether this port has it. Written because
the project had eleven format documents and no map of how the formats meet.

Provenance: the new-game trace through `install/sacred` (findings log rows
954–963), plus the readers and gates named per row. Combat and regeneration
were added from rows 1036–1051; the gap list is row 1052. Function addresses are
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

          IT BUILDS EVERYTHING DERIVED, not just AT and PA (rows 1045/1046).
          354 float stores land in the block at +0x5A..+0xF2, and among them:
            +0xEE  regeneration rate for SPELLS      *= 1 + [+0x18]*0.01
            +0xF2  regeneration rate for ARTS        *= 1 + [+0x16]*0.01
          and it calls sub_82047D4 to rewrite every art's own clock.
          So this is the function to read for any "where does stat X come
          from" question -- MAX HIT POINTS included, which is still unfound.

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
| 6b | placements → world | `startcode.bin` | NPCs via `--npcs`; opcode-8 model objects stream with sectors (finding 1253) | `spawn_check`; live chest draw and sector unload/revisit smoke |
| 6c | sector → music | `world/sectors.keyx` | **yes** `formats/sectors.gd` | `sectorenv_check` |
| 7 | band + hero level → level | `sub_81806DC` | **yes** | `spawnlevel_check` |
| 7b | body id → creature | `creature.pak` | **yes**, all 86 bytes | `creature_check` |
| 7c | class pair → hostility | faction matrix | **yes** | `factions_check` |
| 8 | attributes+skills → AT/PA | `sub_81FA5AA` / `sub_81FA622` | **yes** — base and multiplier | `combat_check` |
| 8b | AT/PA → to-hit | `sub_428790` | **yes** | `combat_check` |
| 8c | damage vs resistance | `sub_81FAC30` | **yes** (rows 1036/1037) — four channels through the same curve to-hit uses, `dmg = raw·(1 − 1/(raw/armour + 1)^k)·(100 − resist)/100`, with the LEVEL DIFFERENCE as the exponent. `world/combat.gd`. Armour and resist go in as zero because nothing equips yet — see 3b. | `combat_check` |
| 8d | attributes → regeneration | `sub_820E04C` → `sub_82047D4` | **yes** (rows 1043–1048). An art costs TIME, not mana: `total = base + level·step` from the table at `0x8793D00`, `rate = bonus·(1 + attribute/100)` per kind. `world/regen.gd`, `formats/combat_arts.gd`. | `regen_check` |
| 8e | template → combat arts | PAX `0xC7` `+0x4CD` | **yes** (row 1049) — the saved records are the live 22-byte structs; `encounter.strike(rng, art_id)` spends one and refuses a cold art. | `regen_check` |
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
| facing / heading | **done (row 965)** | derived per model; the hero turns to its last heading |
| idle vs walk vs attack | **done (rows 1053, 1056)** | `Rigs` resolves a clip per ACTION, `PlayerView.play_action` switches between them, and `main.gd` drives it per frame from whether the hero's cell moved. WALK and IDLE only — one measured walk speed means no threshold to switch RUN on. All seven class bodies resolve a walk. |
| **HUD** | **done (row 962)** | retail's own rects and coordinates. The LIFE gauge is drawn too (rows 1039–1044): it is the PORTRAIT RING, `UI_CHR_HEALTH_01/02`, sliced at a waterline. There is no mana gauge because there is no mana (row 1042). Nothing drives the ring yet — the hero has no hit points. |
| sound, particles, water, weather | **none** | no screenshot impact |

---

## What is actually left for a 1:1 small-scale MVP

Seven items were listed here on 2026-08-16. **All seven are now settled** —
six closed by rows 954–965, the seventh a capture-runbook decision rather than
a research gap — so this list no longer describes what is left.

**The current gap list, re-ranked 2026-08-20 (row 1052)**, is below it.

| # | Item | State |
|---|---|---|
| 1 | HUD | **Closed.** The layout is a static 1887-entry sub-rect table at `0x880DC68` placed by `cUI_Taskbar2` onto a fixed 1024×768 canvas. `view/hud.gd` draws the console, wings, buttons, combat-art arc and both slot wings from retail's own art. **The life gauge is drawn too** (rows 1039–1044) — it is the portrait ring, not an orb, and there is no mana gauge to draw. See Open. |
| 2 | Facing | **Closed** (row 965). The alignment bone is the net rotation above `Bip01`; `set_yaw` is wired and gated by `facing_check`. |
| 3 | NPCs by default | **Decided, not open.** `--npcs` already places the scripted cast at real retail cells. It stays opt-in because `tools/parity/follow_parity.sh` and `loggia_sweep.sh` photograph the world and rely on the current default rather than passing a flag; flipping it would silently change frames they compare. Closing this properly means adding `--nonpcs` to those runbooks first, which is a capture decision and not a research gap. |
| 4 | Quest text on screen | **Closed.** The console shows the quest's own line; quest 74 reads *"The Soul of the Demon"* / *"Kill the demon, after Shareefa has summoned it."* |
| 5 | Composed keys in the VM | **Closed.** `QuestLog.resolve_with` substitutes from its own variables, and `SetVarBit` is now understood as a bit index, so the variables it reads are right. |
| 6 | The AT/PA base | **Closed.** `0.5·(STR+DEX)` and `0.2·STR + 0.8·DEX`. The MVP fight is 31%, entirely derived. |
| 7 | Clip selection | **Closed.** The action is readable from the clip name even though the character is not; all five buildable bodies now play their IDLE instead of whatever scored highest. |

### The current gap list (row 1052)

Ranked by effort, then by how much each unblocks. Each row cites what shows the
gap is real, because the previous list stayed on this page for four days after
it stopped being true.

| # | Gap | Effort | Unblocks | What shows it |
|---|---|---|---|---|
| ~~1~~ | ~~**Runtime clip switching**~~ **Landed 2026-08-20 (row 1053), with one body left out.** `PlayerView.play_action` switches a body to its clip for a named ACTION; `main.gd::_drive_hero_action` picks WALK or IDLE once per frame from whether the cell actually moved — the only movement signal the sim has, since `heading` is never written by click-to-move and `facing` is held. WALK and IDLE only: the port has one measured walk speed (row 1013), so there is no threshold to switch RUN on. **But SERAPHIM.GRN resolves no WALK**, and she is who a retail start spawns — see the new #1 below. | — | — | `checks/hero_anim_check.gd` switches a body, switches it back, and refuses an action it has no clip for. |
| ~~1~~ | ~~**The Seraphim's two rig revisions**~~ **Closed 2026-08-21 (row 1056).** `Rigs._admit_by_prefix` learns a character's clip prefix from the one clip geometry proved and fills the actions geometry left empty. `hero_anim_check` reports `no_walk=[]` where it reported `[SERAPHIM.GRN]`; `rigs_check`'s permutation control still separates at real ≥0.844 vs permuted ≤0.489. **Every class body can walk, and the hero switches clip at runtime.** | — | — | — |
| 2 | **The hero as a creature** — hit points, taking damage, a two-sided fight | Med-High | The ring gauge's driver, somewhere for experience to live, a fight that can be lost | `view/hud.gd::set_health` is called only by `hud_check`. The foe is a registry actor with `hp`; the hero is scalars on `Encounter`. Blocked on the max-HP derivation — likely inside `CalcResults`, which builds every other derived stat. |
| 3 | **Equipment actually equipped** | Medium | Armour and resist for 8c, the `bonus` term for 8d, the weapon base for the recovery clock | Row 3b: *"ids only, not equipped"*. Three formulas currently take a placeholder: `Combat.damage` gets zero armour, `Regen.rates` gets bonus 1.0, `sub_81A8636` falls back to 20.0. **They are one gap, not three.** |
| 4 | **What an art does to damage** | Med, uncertain | Makes a spent art matter | `+0x50`/`+0x54` are measured and read like a multiplier for attack moves, but the same field is a duration on a shapeshift art. Row 1036 notes the weapon-slot flag appears nowhere in `sub_81FAC30`. |
| 5 | **The experience value** | Low-Med | Fills the XP bar that is already identified and drawable | Dumping the hero creature's `+0x420`…`+0x620` across a kill moved only two noise fields. |
| 6 | **Actor shadow raster parity** — native blob geometry now renders | Open | Matching actor footing | Finding 1232: texture and five float32 strip sizes match native captures; pose, anchor, encoded blending, native depth/order, and projected/stencil branch remain incomplete. |

**Why #2 is not first.** It reads like the obvious next step and it is not the
biggest one. The list is ordered by effort and by what each unblocks, never by
what is interesting — ranking by interest is how the old list survived four days
after it stopped being true. #1 stays at the top on a technicality worth being
explicit about: the *work* of clip switching is done and gated, and what is left
is one body's rig, which is cheap to finish and finishes a visible feature for
the only character a default run actually spawns. Hit points still need a
derivation found before any of #2 can start.

## Open

- ~~**The life and mana gauges.**~~ **Closed 2026-08-20 (rows 1039–1042), and
  this entry was WRONG about where to look.** It said the orb-looking elements
  of `GUI_main_02` "belong to the mercenary window". They do not: they are
  `UI_CHR_HEALTH_01` and `_02`, retail's own name for the **portrait ring**,
  which is the life gauge. The enumeration above is still correct and was never
  the problem — `cUI_Taskbar2` genuinely references no orb art, because the
  gauge was never in the taskbar. It is the portrait window.

  **And there is no mana gauge, because Sacred has no mana.** Its six
  attributes are Strength, Endurance, Dexterity, Physical Regeneration, Mental
  Regeneration and Charisma; not one of the 1446 named interface elements is a
  mana anything; the mana potion types are dead; and the `+0x4C8` pool triple
  is entirely hit points. What a combat art costs is TIME, shown per art on the
  slot as `UI_ACTION_GRAYED` and a `_LOAD` state — see 8d.

  `view/hud.gd` draws the ring and fills it. **What is missing is the number,
  not the gauge:** nothing calls `set_health()` because the hero has no hit
  points. See the MVP list below.
- ~~**Character facing.**~~ **Closed 2026-08-16 (row 965).** The split was the
  chain above `Bip01`, quantised to 0° or −90°: three bodies carry a `Root` bone
  that cancels `Bip01`'s −90°, three do not. Measured in `Bip01`'s own frame all
  seven agree (worst dot 0.9556), so the mesh-space angle is a sound per-model
  constant — it already contains the alignment. `PlayerView.face()` derives the
  target from the node's basis rather than the world displacement, because this
  world is pre-projected and its depth axis vanishes under normalisation.
- **The damage/resolution step** — how damage meets resistance, criticals, and
  what the weapon-slot flag selects. To-hit and both ratings are recovered;
  this is what is left of a fight.
- **Hit points.** In no table read so far.
- **The 200× factor and the 5..95 clamp** are in the Windows binary's
  `sub_428790` and are *not* in the Linux build, whose display path computes
  `100·AT/(AT+PA)` through `sub_815D44C` with no level term. Recorded as a
  discrepancy between two binaries rather than resolved.
- **Actor blob shadows, current 2026-09-07 (finding 1232).** The observed
  start actor uses LGP `0x080FE8DA` / Windows `0x00407960`, not a projected
  body mesh. Five strips use root, root/left-foot midpoint, root/right-foot
  midpoint, left foot, and right foot; all flatten to sampled support height.
  Half-size starts at float32 `radius * 0.6000000238418579`, then shrinks by
  float32 `0.6666666865348816` for the midpoint and foot pairs. Captured radius
  50 gives 30.0000019073, 20.0000019073, and 13.3333349228.

  The production renderer's five sizes and 64×64 SHADOWDOT texture match the
  live buffers exactly. Native diffuse is black with alpha 80/255; filtering
  is linear/repeat, with source-alpha blending and no alpha test. This branch
  does not inherit the body's calibrated scale. Radius comes from item `+20`,
  model scale/reflection from kind-64 metadata `+1136/+1140/+1144`, and
  initial heading from item `+87` through event 64. Actor `+112/+116` are
  heading, not scale. Support height uses the authored child grid and native
  four-triangle fan, not terrain sorting depth.

  **Not 1:1.** ~~The captured root and port root differ by about 0.57/0.54
  px.~~ Finding **1234** corrects the shadow origin to the integer native
  lattice point already used for heading/support. A frozen-pose experiment
  reduces root-quad mean error to **0.045851/0.066250 px**, confirmed by the
  production packet. All origin writers now share that point; the body's
  calibrated placement is unchanged. No fitted pixel offset was introduced.
  The frozen control changes only 2872 shadow pixels, with exact floor control.
  Evidence: `donotpublish/tmp/shadow-contract-20260907/anchor/` and
  `anchor-integrated/`.

  Foot positions still differ at unmatched animation times.
  ~~Forward+ blends the production blobs in linear color; global insertion
  is unwired.~~ **Updated 2026-09-21, finding 1263:** blob packets now draw
  directly in encoded RGB immediately before their actor in the shared static/
  model FIFO. Cropped model targets isolate self-depth; native cross-model
  shared-depth equivalence and the projected/stencil branch remain open.
  The pre-static-compositor world-band result was **7.642253%** exact-difference pixels,
  RGB MAE **2.446650**, with an exact floor control. Actor animation times
  differ between full-scene captures; this is not an isolated score-improvement
  claim.
  Evidence: `donotpublish/tmp/shadow-contract-20260907/run02/` and
  `port-native/`. A clean render and correct geometry do not close these gaps.
  Finding **1237** subsequently reduces the world band to approximately
  **5.230% / MAE 1.713** through static sprite blending and authored static
  shadows; that does not close the actor-specific gaps above.

- **Native actor input observation, 2026-09-07 (finding 1238).**
  The running start Seraphim uses effective model `SERAPHIM.GRN`, native
  model row 661, and motion enum 2. The shipped model table at `+112 + 4*2`
  selects native motion row 1467, `SERA_IDLE_BH.GRN` (pak entry 3039).
  Its entire 1,024-byte motion table agrees with the live model row.
  The active control's rate is **1.25**; its last evaluated local time in
  the retained frame is **0.22374999523162842**. Body and shadow observation
  share clock, sequence, and actor bytes. This is not a phase fitted to pixels.

  The body draw enables fixed-function lighting with one directional light;
  light/material state is captured. The observer still marks its raw raster
  path incomplete for lit draws: these inputs are observed, not a successful
  raster replay. Production body pose, calibrated transform, and lighting are
  unchanged by this finding. Evidence: `donotpublish/tmp/actor-pose-20260907/run02/`.

- **Godot lit-draw replay, 2026-09-07 (finding 1240).** The earlier lack
  of a lit replay is now partially closed: a scratch Forward+/Vulkan
  SubViewport replays captured head draw 6 from `run03`, including its
  positions, normals, UVs, texture, matrices, and pre-draw color/depth.
  Per-vertex ambient/diffuse/specular lighting reproduces exactly the
  native draw's **194 changed pixels**, with RGB-byte MAE **0.163230**
  and maximum error **3**. Replacing only the lighting with the port's
  flat ramp raises MAE to **48.905498**. Both arms leave every pixel
  outside the native changed set byte-identical to the supplied background.
  Explicit Vulkan clip output needs Y inversion and reversed depth.
  RGB half-float readback is rounded to UNORM8; Godot's Image.convert
  truncation otherwise adds an artificial approximately one-byte error.
  This validates one captured head draw, not all body surfaces, animation,
  production transforms, or actor insertion. Production remains unchanged.
  Evidence and repeatable measurement: `donotpublish/tmp/actor-pose-20260907/`
  `godot_lighting_replay.gd`, `measure_godot_lighting.py`, and `godot-lit/`.

  **Extended to all six body draws (finding 1241).** Fresh `run04` captures
  before/after color and depth for each body draw; the clock/sequence/actor
  consistency checks remain true. Replaying each independently in Godot gives
  weighted RGB-byte MAE **0.082711**, versus **33.337444** for the flat
  control, over **2,676 draw-pixel observations** (not distinct final pixels).
  Every outside pixel stays exact in both arms. Draws 6–10 have exact
  changed-pixel coverage; draw 11 misses one native pixel at (505,384):
  retail (35,30,30), unchanged/replay (21,21,21). Maximum channel error is
  14. That discrepancy remains unexplained; this is not exact raster parity.
  The observer's inherited `incomplete` marker still reports its unsupported
  lit path; these independent GPU comparisons validate the retained draws,
  not every path in that observer. No production change. Measurements:
  `godot-lit-all/body-measurements.json` and `measure_godot_body.py` under
  the same evidence directory.

  **Raster miss follow-up (finding 1242).** Evaluating modelview and
  projection separately does not recover the draw-11 pixel; it raises MAE
  to 0.176979 with unchanged outside pixels. A fresh native `run05` reports
  `GL_SUBPIXEL_BITS = 8` for all eleven draws, refuting a four-bit
  coverage-grid explanation. The missed pixel's original depth is 1.0
  (after the retail draw, 0.8129918), so this is not an existing-depth tie.
  The boundary discrepancy remains open; no fitted snapping or offsets
  were adopted. Control: `godot-lit-all/separate-011/measurements.json`.

  **Native feedback control (finding 1243).** A separate `GL_FEEDBACK`
  diagnostic on `run06` returns 130 culled triangles with 390 window-space
  vertices, all on a 1/256-pixel grid. That driver path returns constant
  texture coordinates, so direct textured feedback replay is not valid.
  Joining the original UV array through the exact surviving triangle sequence
  (not nearest seam vertices) gives replay MAE 0.162037 versus 0.164021
  from reconstructed raw inputs, with the same missing pixel at (525,391).
  Outside pixels remain exact. Feedback is not evidence that real raster
  arithmetic is identical, and it does not close this discrepancy.
  Evidence: `run06/`, `measure_feedback.py`, and
  `godot-lit-all/feedback-comparison.json` in the actor-pose directory.

  **Source-derived chest lighting (finding 1254).** The light is not a
  universal character preset. LGP `sub_80E7416` and ENG's scene consumers
  select ambient 0.3 when item definition byte `+0x2e` is 3, otherwise 0.8;
  actor flag `0x20000000` replaces it with a clock-driven pulse. The existing
  item reader gives Seraphim/novice category 3 and chest type 5201 category 4.
  LGP scene initialization normalizes direction `(1,-1,-3)`; its GL adapter
  negates the direction and transforms it through the view matrix. Light
  manager `sub_83B3B44` / ENG `sub_642FD0` multiplies diffuse/specular by
  environmental RGB and ambient by the selected scalar. The LGP material
  adapter maps power 30 to GL shininess `30*1.28 = 38.4`.

  Fresh `chest-light-native/` retains before/after buffers for both chest
  draws. Source direction, item-selected ambient and source shininess produce
  the same rounded replay images as captured GL light inputs. Their changed
  coverage is exact (2785/1914 pixels), with no outside changes; RGB-byte
  MAE is 0.016038/0.032393. Flat-ramp controls score 19.445242/25.014629;
  deliberately using the character ambient scores 30.156553/9.548067.
  The source reconstruction explicitly requires the observed white
  diffuse/specular input; it does not substitute white for environmental
  lighting elsewhere. Maximum channel errors are 1/36, with the second
  draw's large residual at (241,285), so exact raster parity remains open.
  The observer still rejects full chest pose/shadow completeness; only its
  guarded draw packets and independent GPU replay are used here.
  Production materials remain unchanged. Evidence under
  `donotpublish/tmp/actor-pose-20260907/`: `lighting-categories.json`,
  `chest-lighting-measurements.json`, `chest-light-replay/`,
  `chest-light-replay-002/`, and `measure_chest_lighting.py`.

  **Unprojection precision corrected (finding 1255).** Integrating normal
  transforms exposed a 0.74-degree chest heading error. The port used double
  division by 768; LGP `sub_80D78BA` multiplies by the float32 reciprocal
  (`0x3aaaaaab` at `0x086d7230`, read by `fmul DWORD` at `0x080d78fe`).
  ENG `sub_623940` independently uses the same reciprocal expression.
  At lattice point (173474,134834), the old conversion produced Y
  160577.078125 rather than native 160577.09375. Subtracting two nearby
  converted points amplified the rounding into the heading error.
  `NativeActorShadow.lattice_to_native` now preserves the shipped reciprocal.
  `native_unprojection_check` fails on both position and heading before the
  fix and passes afterward. Against the fresh chest capture, all 206
  position/UV vertices have a counterpart within 0.007526 native units;
  matching hard-edge seam candidates leaves maximum normal-vector error
  0.000089124. This does not resolve the remaining vertex/readback precision.
  All 53 checks pass. Fresh inspected start-scene gate: world 5.17%/MAE 2.00,
  full 6.44%/3.09; PASS, baseline unchanged. Production lighting is still
  the legacy ramp. Evidence: `chest-port-normals.json`,
  `chest-normal-measurements.json`, and `startgate-after1255/` in the same
  private actor-pose directory.

  **Production object lighting integrated (finding 1256).** Authored model
  objects now use `native_object.gdshader`: source-derived directional
  diffuse/specular vertex lighting, item-category ambient, native alpha test,
  and encoded-RGB texture modulation before conversion to Godot's linear
  target. It consumes the ordinary RGBA8 skin that `ModelView` already
  decodes; terrain's packed-channel convention does not apply here.
  ~~Creature materials remain on the legacy ramp pending posed-normal
  validation.~~ **Closed for initial daylight in finding 1266:** body
  projection and post-skin normal conversion are now validated and wired.
  The simulation still has no native solar clock, so this does not establish
  time-dependent RGB/pulse behavior.
  In the fixed chest rectangle (230,250)-(335,335), production RGB MAE
  falls 12.43746 → 4.033875 against the same retail frame; adjacent floor
  pixels are byte-identical. The projected chest shadow is still absent.
  All 53 checks pass. Fresh inspected gate: world 5.05%/MAE 1.86,
  taskbar 11.01%/6.96, full 6.36%/2.97. The existing ratchet advances its
  world baseline from 5.16 to 5.05; this is not full 1:1 parity.
  Private evidence: `object-light-production.png`,
  `object-light-comparison.png` (old ramp / native lighting / retail),
  `object-light-production-measurements.json`, and `startgate-after1256/`
  in the same actor-pose directory.

  **Actor volume repair (2026-09-21, findings 1265–1266).** Full affine
  scale/shear transport fixes the wolf's distorted neck/limbs; native body
  camera projection and shared directional Gouraud materials remove the
  flat/front-on actor path. Same-pose Seraphim/novice topology and normal
  comparisons, actual GPU normal probes, a guarded native wolf-matrix oracle,
  and twelve rendered wolf idle/walk/run samples distinguish these fixes
  from a brightness adjustment. Actor skin overrides are also consumed;
  forced wolf type 588 uses its authored red/black WOLF_VAMP01 texture.
  See [the affine/volume contract](../formats/granny-grn.md#full-affine-deformation-and-actor-volume--2026-09-21).
  Current fixed benchmark: world **4.58% / MAE 1.66**, full **5.99% / 2.82**,
  repeat difference **0.00%**. The default benchmark contains no wolf and
  therefore cannot establish wolf correctness. Private visible comparisons
  and motion frames are in `donotpublish/tmp/actor-shape-20260921/`.

- **Local pose binding corrected (finding 1247).** Same-clip, same-time
  comparisons against captured native bone arrays disproved the port's
  rest-pose retargeting and omission of position tracks. `ModelView` now
  applies authored local quaternion and position keys directly. Across all
  non-placement bones of the Seraphim and novice nun (71 and 72), maximum
  local angle error falls from 78.041684/89.999983 degrees to
  0.004283/0.001935 degrees; position error falls from 2.239036/0.758285
  to 0.000929/0.000441 model units. The native world-placement `__Root`
  is excluded because the standalone viewer has no world placement.
  Native body/shadow bone arrays are identical within each captured frame,
  and both snapshots retain their clock, sequence and actor bytes.
  This corrects the shared production binder, not gameplay clip selection,
  playback rate/phase, world placement, lighting, or actor depth insertion.
  Evidence and the retained old-binder control:
  `donotpublish/tmp/actor-pose-20260907/measure_pose_binding.py`,
  `pose-binding-measurements.json`, and `run10/` / `run12/`.

- **Scripted model objects now render (2026-09-07, finding 1253).**
  Production previously ignored `Startcode.objects`. The active class's
  opcode-8 records are now indexed by sector; `SectorView.sector_built`
  attaches their model nodes to the sector, including fixed-region and
  record/replay/crowd view construction. Sector retirement frees the nodes.
  This renders authored initial objects; it does not implement chest interaction,
  loot handling, or subsequent script-driven object mutations.

  The start chest is type 5201, `chest.grn`, at authored cell (3232,2512),
  layer 1. Native root position selects the **cell centre**: the corner
  control misses by 27.6133 native units, the centre reconstruction by
  0.015625. That remaining float32-coordinate discrepancy is not claimed closed.
  **Correction:** finding 1255 above closes that origin discrepancy; it
  came from replacing the shipped float32 reciprocal with double division.
  Its item heading is -30 degrees and its model-header scale is (1,1,1).
  Item texture 7442 decodes byte-identically to both native draw textures.

  All 206 native position/UV vertices match the parsed geometry (maximum
  reconstructed position error 0.007528 native units); all 168 triangles
  match with reversed stored winding, versus 128 with unchanged winding.
  **Correction to the initial experiment:** enabling self-depth alone did not
  fix the inside surfaces. The winding/culling control did: chest-window
  RGB MAE falls 16.1251 → 12.6037 with only that change. Production retains
  depth testing and the camera's depth ratios. The old humanoid calibration
  and skeleton-only placement are not used for rigid objects.

  Against the fresh retail frame, the chest window (230,250)..(335,335)
  improves from MAE 18.1046 without the object to 12.6037 with it.
  Adjacent floor control pixels are unchanged. All 52 checks pass; the live
  streaming smoke observes one chest, its destruction on sector unload,
  and exactly one rebuilt chest on return.
  Fresh gate: world 5.17% / MAE 2.00, taskbar 11.01% / 6.96,
  full frame 6.45% / 3.08. PASS under the existing tolerance; the 5.16%
  world baseline is unchanged. This is **not** 1:1 lighting/shadow or
  native actor/static-depth insertion parity.

  Evidence: `donotpublish/tmp/actor-pose-20260907/run19/`,
  `chest-measurements.json`, `measure_chest.py`,
  `chest-winding-{control,native}.png`, `scripted-object-revisit.png`,
  and `startgate-scripted-objects/`. The general actor-pose probe marks
  this capture incomplete because creature-only motion/bone/shadow queries
  do not apply to the chest. Only the separately guarded chest identity,
  geometry, texture and root-transform witnesses above are used.

- **Historical shadow investigation (2026-08-24; superseded above).**
  The then-missing shadow was recorded in `view/player_view.gd:96` as a
  measurement nuisance ("contaminates the bottom") while calibrating character
  scale, and that code comment is the only place in the project the fact is
  written down. **Registered here 2026-08-24 because it was named as a defect
  and then lost.** `AGENTS.md` § STATUS 2026-08-17 (evening) decomposed the
  then-29.3% two-engine delta and made its FIRST bucket "object-sprite shading +
  retail's drop shadows under every object and character (halo blobs in the diff,
  the port draws none)". That bucket does not appear in row 1018's enumeration of
  the 13.55% the same night, was never closed by any row, and the word "shadow"
  appears in neither `open-questions.md` nor `world-sectors.md` nor this file
  until now.

  **Historical, now superseded:** ~~The port draws no character drop shadow.
  Nothing in `view/` emits a shadow pass.~~ The old
  `shaders/object.gdshader:78` reference described Sacred's 4-bit alpha
  gradient rather than a separate shadow pass.

  **What is NOT established, and must not be assumed.** (1) Its pixel cost —
  never measured, in any frame. It is a *candidate* for the roughly 8.4pp of
  row 1018's 13.55% that its four named items do not account for, and a candidate
  is not a finding. (2) Whether the "under every object" half is a separate pass
  at all: `object.gdshader:78` states Sacred's sprites carry shadows, glass and
  smoke at partial alpha, so a static object's shadow may already be baked into
  its art and already drawn. The hero's is the only one proven missing. (3) How
  retail draws the character's — projected quad, sprite, or blob — is untraced.

  **UPDATE 2026-08-24, same day: the mechanism is now named, from our own
  binary.** A lead out of the VK corpus (`tools/vk/`) pointed at a Sacred NL
  config key; the key was then verified first-hand in `install/sacred`, which is
  where the evidence below comes from — the forum was the pointer, not the
  source.

  Historical string-based inferences below are not a substitute for the live
  branch/geometry observation above:

  | String in `install/sacred` | Historical interpretation |
  |---|---|
  | `cGranny::renderShadow()` | retail has a **named shadow render path in the Granny layer** |
  | `cGranny::renderShadowFake()` | and a second, cheaper one — two modes, not one |
  | `SHADOWDOT.TGA` | ~~texture, not geometry~~ — textured geometry; five strips in the observed branch |
  | `SHADOW_TREE00.TGA` | objects get their own shadow art, so the "under every object" half **is** a real pass |
  | `NOSHADOW`, `FLAGS:NOSHADOW` | a **per-object opt-out flag**, so the pass is selective |
  | `FORCE_BLACK_SHADOW` | a `Settings.cfg` key that selects between the modes |

  The art ships in data the port already reads: `texture.pak` carries
  `SHADOWDOT.TGA`, `FX_SHADOWDOT01.TGA` and `SHADOW_TREE00.TGA`. None of these
  seven strings is named anywhere in `research/`, `engine/` or `AGENTS.md`.

  **`FORCE_BLACK_SHADOW` is answered, 2026-08-25.** `sacredtools 3.3`'s bundled
  help documents it on the Graphics tab as *"disables shadow transparency, giving
  less realistic black shadows; affects performance"* — so **retail's shadows are
  alpha-blended by default** and the key forces them opaque. A port that draws a
  solid black blob would be reproducing the non-default setting. The gloss is a
  third-party tool author's, not Ascaron's, but it is specific and it is testable
  against a capture with the key flipped.

  **Scope correction, 2026-09-07:** that tool gloss is not proof that
  FORCE_BLACK_SHADOW changes every branch. The observed five-strip path writes
  alpha 80 unconditionally. Branch-specific behavior must be traced or measured,
  not generalized from the setting's label.

  **THE PIXEL COST IS MEASURED, 2026-08-25 (row 1099), and it is small.** The
  shadow is plainly there in a retail capture and plainly absent from the port's.
  Isolating it — pixels under her feet where retail is more than 8 levels darker
  than the port, with her boot columns excluded — gives **1,826 pixels, 100% of
  them differing, at MAE 32.64**. In the gate's units that is:

  | | of the full frame | of the world band |
  |---|---|---|
  | the shadow | **0.23pp** | **0.30pp** |
  | her body, for scale | 0.59pp | 0.76pp |

  A same-sized control crop of open floor beside her differs by **0.60% at MAE
  0.28** — so the floor is exact and the footing is not, which is what makes the
  0.30pp attributable rather than ambient.

  **So the shadow is NOT the ~8.4pp candidate.** Rows 1061 and 1063 both listed
  it as a candidate for the share of row 1018's delta the four named items do not
  account for. It is worth about a thirtieth of that. Where it differs it differs
  hard — MAE 32.6 — but there are only 1,826 such pixels. **Her own body costs
  more than twice the shadow**, and that is the lead this measurement actually
  hands on.

  **What this still does NOT settle.** What `renderShadow` computes, what
  distinguishes it from `renderShadowFake`, and what `NOSHADOW` is actually set on
  are all untraced — string names are a map, not a mechanism. The measurement
  above is of the HERO's shadow only; the "under every object" half is still
  unmeasured and may be baked into sprite art.

  **Correction to the lead this handed on, 2026-08-25 (row 1100).** Row 1099
  closed by calling the hero's cost a shading difference — "the port draws her
  markedly darker". **That was an eyeball impression and it is wrong.** Over the
  differing pixels the port is on average **+8.09 BRIGHTER**, and darker only
  36.6% of the time. What the capture actually shows is **different equipment**:
  retail's Seraphim wears light blue armour with white boots and a horned helm,
  the port's wears heavy dark segmented plate on shoulders, forearms and legs and
  carries a different weapon.

  **The port says so itself.** `main.gd:47` — *"a full starting kit is not what a
  new retail character has. It is here so the composition path is exercised by
  the default run; a real inventory replaces it."* `START_SET := 6` is an
  acknowledged placeholder, and nobody had connected it to the milestone metric.

- **The hero costs 0.78pp of the world band, and drawing her is a net loss.**
  Measured 2026-08-25 with an exact silhouette — the port driven twice, once
  normally and once with `--hideplayer`, so her pixels are the difference between
  two port frames rather than a guessed box.

  | | px | of the world band |
  |---|---|---|
  | both engines drew her — surface / outfit | 2114 | **0.344pp** |
  | port drew her where retail did not | 956 | 0.156pp |
  | retail drew her where the port did not | 1722 | 0.280pp |
  | **hero total** | **4792** | **0.780pp** |

  So roughly **44% surface, 56% silhouette** — it is not one problem. (The two
  silhouette rows depend on a threshold estimate of retail's own mask and carry
  method uncertainty; the 0.780pp total and the 0.344pp surface row do not.)

  **Hiding her entirely scores BETTER than drawing her as she is**: the world band
  reads 10.95% with her and **10.88% without**. The port's hero rendering is
  currently worth −0.07pp — retail still draws her either way, so the port's
  version does not remove the disagreement, it relocates it.

  Row 1018's *"unmoved by dressing — START_SET 6 and the bare rig score within
  0.01pp"* does **not** exonerate the outfit. It shows the bare rig and set 6 are
  **equally wrong**, which is what you would expect when neither is what retail
  starts her in.

  **ANSWERED 2026-08-25 (rows 1101, 1102), and the answer removes equipment from
  the list of suspects.** Retail equips a new character from two arrays filled by
  `equip=<CLASS>,<slot>,<itemid>` and `inventory=<CLASS>,<itemid>` lines in the
  balance text, and **no shipped file carries such a line** — not the install, not
  `balance.bin`, not any build we hold, not the 88k VK corpus. A stock retail
  Seraphim therefore starts BARE, and the capture shows her base rig.

  `main.gd` gained a `--nodress` flag so that could be measured rather than
  argued. All three configurations against the same retail frame:

  | port config | world band | MAE | hero box |
  |---|---|---|---|
  | dressed, `sets.bin` set 6 | 10.95% | 5.05 | 31.10% |
  | bare rig, `--nodress` | 10.92% | 5.02 | 29.94% |
  | mesh hidden, `--hideplayer` | 10.88% | 4.96 | 28.81% |

  **Undressing her recovers 0.03pp of her 0.78pp — about 4% — and hiding her
  entirely is still better than either.** The dressed and bare port renders differ
  from EACH OTHER by 2,324 px (0.378% of the world band), so the outfit is plainly
  on screen; it simply does not move the score. This independently reproduces row
  1018's result, which was doubted precisely because the two outfits look nothing
  alike. They do look nothing alike. It still does not matter to the metric.

  **So the remaining ~96% is the base rig itself — mesh, texture, pose or scale —
  and that is where the 0.344pp surface row above actually lives.** That is the
  next thing to attack, and it is a different investigation from equipment.

  `START_SET := 6` was deliberately NOT changed. It is the only thing exercising
  the equipment-composition path in a default run, and the trade is 0.03pp of
  fidelity against that coverage — a human decision, recorded in
  [../open-questions.md](../open-questions.md) § Decisions waiting on a human.

  The footing crop that this entry once named as "the cheap first step" was taken
  on 2026-08-25 and is the 0.30pp shadow measurement above. What is still cheap
  and still undone: decode `SHADOWDOT.TGA` and look at it.

- **The base rig: scale and placement are RIGHT, the POSE is wrong.** Measured
  2026-08-25 against the same retail frame, using the port's exact hero mask
  (drawn minus `--hideplayer`) and retail's thresholded one in the same column.

  | | px | height | width | centroid x | IoU vs retail, best-aligned |
  |---|---|---|---|---|---|
  | port, dressed | 3070 | 146 | 43 | 512.4 | **0.460** at dx=0, dy=+5 |
  | port, `--nodress` | 2574 | 135 | 43 | — | **0.441** at dx=−1, dy=+3 |
  | retail | 3837 | 149 | 76 (shadow included) | 511.9 | — |

  **Height agrees to 2%** (146 vs 149) so scale is not the defect. **The best
  alignment is dx=0** and the centroids agree to half a pixel, so horizontal
  placement is not the defect either; the port draws her about **5px high**,
  which is small. What is left is shape: **IoU 0.46 — less than half the
  silhouette overlaps even after the best shift**, and shifting buys only
  0.019 of it, so it is not a translation. Retail's idle holds her arms away
  from the body with a drawn sword; the port's holds them in. **The bare rig
  is WORSE than the dressed one (0.441 vs 0.460)**, which is the second
  independent reason equipment is not the lever here.

- **Retail's own frame is not reproducible, and the variance sits exactly on
  the things being compared.** Two retail runs of the same route at the same
  millisecond differ by **1.31% of the world band** (MAE 0.15–0.38), and a
  difference map puts effectively all of it in one place: the hero, the NPC
  beside her and the `?!` marker, which are animating. That is a noise floor
  larger than the whole 0.78pp hero budget, and no single-frame pose
  comparison can see past it. Row 1075 characterised 0.07pp of port-side
  run-to-run spread; this is the retail side and it is nearly twenty times
  bigger. **Any future pose work must measure against several retail frames,
  not one.**

- **The port draws no scripted cast at all, and in this frame that costs more
  than the hero does.** Retail's start frame contains a robed NPC with a `?!`
  quest marker standing beside the player. The port draws nothing there:
  **5,639 px at MAE 56.8 = 0.918pp of the world band**, against the hero's
  0.780pp at MAE ~17. The mechanism is `cInterpretSQW`, which runs a
  `Sector<x><y>Enter` procedure per sector as the player arrives;
  `Sector50039Enter` alone creates four `NOVIZIN.GRN`, a `CHICKEN`, a `RABBIT`
  and four `FX_FIRE_L`. See
  [../formats/script-bytecode.md](../formats/script-bytecode.md) for the
  decode. **Her own source is now known (row 1106):** she is not from a sector
  script at all but from `QIS_OnEnter1`, the OnEnter hook of quest 1
  (`Tutorial`), which does `CreateNPC res:17095, 679, novizin1, ...` and then
  `NPC_Goto res:17095 -> cell (3237,2514)`. The port's `Sacred.Vectoren`
  already resolves that hook and `world/quest_log.gd` can already run one —
  **nothing fires quest 1's OnEnter when a new game starts**, and that is the
  integration gap.

- **Quest 1's OnEnter is WIRED, and drawing her made the metric WORSE.**
  Implemented 2026-08-25 (row 1107): `world/script.gd` gained `SetVar`,
  `CreateNPC`, `NPC_Goto` and `QuestKompassObj`, `world/quest_cast.gd` collects
  the cast as data, and `main.gd` runs quest 1 on a new game (`--noquests` opts
  out). The hook executes all 10 records and builds `NOVIZIN02.GRN` at cell
  **3237,2514** — exactly where the script says.

  | | world band | MAE | the NPC window |
  |---|---|---|---|
  | before, she is absent | 10.95% | 5.05 | 5,639 px / 0.918pp / MAE 56.8 |
  | drawn, walking on the spot at a cell corner | 11.11% | 5.23 | 6,598 px / 1.074pp |
  | **drawn, idling at the cell centre** (row 1110) | **11.01%** | 5.17 | **6,006 px / 0.978pp** |

  **Drawing her costs 0.16pp more than not drawing her**, stable across two
  runs. This is the same shape as row 1100's result for the hero — retail draws
  the figure either way, so a port figure in nearly the right place does not
  remove the disagreement, it doubles it.

  **What is right:** the model, the cell, and the horizontal placement —
  centroid x **416.9** against retail's **416.5**, agreeing to under half a
  pixel, which is what confirms the whole projection and the handle-matching.

  **What is wrong, in the order it costs:**
  1. ~~**Scale and vertical anchor** — 10–19% too tall.~~ **WRONG, and
     retracted the same day (row 1108). The scale is right and the character
     scale needs no fix.** That figure came from comparing the port's exact
     silhouette against a retail mask that had swallowed the `?!` marker above
     her and her drop shadow below. Segmenting her instead on brightness inside
     her own column — a white robe on a mid-grey floor, which excludes a marker
     that is not in the column and a shadow that is darker rather than brighter
     — gives **retail 122 px against the port's 125**. That is **2.4%**, inside
     the 6% the original two-landmark calibration already disagrees with itself
     by. `RETAIL_HUMANOID_PX / REF_HEIGHT` = 133/73 survives its first
     independent second-model test.

     **Nor is there a per-model rig-origin offset.** Drawing this NPC at the
     hero's own cell as a one-off diagnostic lands her feet at **exactly y=385,
     the hero's own foot row**, and in the hero's own x range. Two different
     models ground identically.

     **What the remaining vertical difference actually is: animation phase.**
     The same rig at the same scale measures **y 330–454, 125 px tall** at cell
     3237,2514 and **y 234–385, 152 px tall** at cell 3236,2511 — a 22% swing in
     drawn extent from nothing but the clip's phase, because `main.gd` seeks
     each NPC by a hash of its cell so a capture is reproducible. So her pose,
     not her size, is what is off, and it is off by a deliberate arbitrary
     amount. Retail's own nun is animating too (row 1104's 1.31% run-to-run
     variance is largely her and the hero).

     **The lesson is the one already in this file twice.** Row 1099 called the
     hero's cost a shading difference from an eyeball impression and row 1100
     had to retract it; this retracts a height from a contaminated mask. A mask
     built by thresholding one image against another carries everything that
     differs, not the thing being measured — segment the subject on its own
     properties instead.
  1b. **NOT a transparency defect — I claimed one and it does not exist**
     (row 1111). Comparing a 4x crop of retail's nun, who is turned AWAY from
     the camera, against the port's, who faces it, I read the garment's own
     gold-filigreed front as limbs showing through a translucent robe. At yaw
     180 the port's rig is plainly solid. Every one of ten character skins
     sampled is opaque on every texel, and switching the material to
     `TRANSPARENCY_ALPHA_DEPTH_PRE_PASS` moves **zero pixels**. What WAS wrong
     was the comment: `DEPTH_DRAW_OPAQUE_ONLY` does not "still write the depth
     buffer", it means only opaque materials do. The behaviour is fine, the
     description was not, and only the description changed.

  2. **Facing — still open and NOT fittable.** Retail has her turned away from
     the camera; the port's `set_yaw(0.0)` faces it. Nothing in the hook says
     which way she looks, and her real facing is presumably her arrival
     direction from a walk the port does not simulate. Picking a yaw that
     matches one capture would be fitting, not transcribing.

  2b. **Two defects that WERE fixable were fixed (row 1110).** The clip: all
     three NPC builders called `Rigs.clip_for()`, the best geometric score,
     where `view/player_view.gd` has always called `rest_clip()` for the hero —
     so she was playing `PRSS_WALK_BH`, walking on the spot. `clip_for` gives
     the Seraphim herself `SERA_SPECIAL_MULTI_2WAFFEN`, which is why the hero
     path already avoided it. The placement: `cell_to_world` maps an integer
     cell to its CORNER, right for a spawned hero (feet y=385 in both engines)
     and wrong for an NPC that WALKED here, because `NPC_Goto` converges on the
     cell's middle — feet corner 456, centre 480, retail 483.
  3. **No drop shadow** (rows 465–495 in retail, ~30 px of solid difference).
  4. **No `?!` marker** — deliberate, the art is unidentified.

  The same run independently re-confirms the 1:1 world-to-pixel scale of row
  1106: `player_view.gd`'s own calibration block states "ortho size 768 over a
  768-px viewport", reached from a different direction entirely.

- **The port has two noise floors and they are not the same size** (row 1109).
  Two runs of the SAME build differ by **3,710 px = 0.604pp of the world band**,
  yet both score **11.11** against the same retail frame — identical to 0.01pp,
  because the pixels that wobble are already inside the set that differs from
  retail. So the gate's `TOL=0.10` sits six times above the score's own noise
  and is sound, while a pixelwise comparison of two captures is six times
  noisier than the tolerance and will call a no-op a regression. It nearly did:
  a change verified as pixel-neutral moved 2,848 px, fewer than the control.
  **Compare scores, never raw pixel counts.** Retail's own floor is 1.31pp of
  pixels (row 1104).

- **`height_scale`** in `sector_view.gd` is still an admitted guess of 1.0.
- **Eight of 3421 animation clips** do not decode.
