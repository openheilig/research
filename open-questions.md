# Open questions

**Status:** Standing
**Purpose:** Every open item from every document in this repository, in one
list, so the state of the project is one read rather than nineteen.

Each entry is the `## Open` section of the document it names — that document
is the authority, this is the index. If you close one, close it there and
strike it here.

Nineteen documents carry an `## Open` section. Three of them — `balance-bin`,
`tech-stack`, `build-survey` — say "nothing open" and are deliberately absent
below. The rest are indexed. An audit on 2026-08-16 found five items that were
open in their own document and missing here; they are the five marked
**(indexed 2026-08-16)**. An index that under-reports is worse than no index,
so check this list against `grep -l '^## Open' **/*.md` when you add a
document. Count the HEADINGS, not the files: as of 2026-08-17 `ui-taskbar`
carries two, so nineteen documents hold twenty `## Open` sections.

Three items were opened on 2026-08-17 by measuring the start scene against
retail rather than by reading the binary — the overlay tile selection, the
authored draw order for building parts, and how a filled art slot is
composited. Each is marked **(opened 2026-08-17, row N)** and each carries the
hypotheses already REFUTED for it, because that is the expensive half.

## Formats

| Question | Where it lives |
|---|---|
| Eight of 3421 animation clips do not decode. | [formats/granny-grn.md](formats/granny-grn.md) |
| ~~One mesh's vertex count disagrees with an outside reading, 279 against 280.~~ Struck 2026-08-15: retail's own index array for that batch resolves to 279, so the outside reading is the wrong one. | [formats/granny-grn.md](formats/granny-grn.md) |
| Bink `.bik` and Miles `.mss` are third-party formats we do not decode. (`mixed.pak` was listed here in error — the engine had read it; struck 2026-08-15 and confirmed in a second build.) | [formats/pak-containers.md](formats/pak-containers.md) |
| ~~Whether the `0xC8` record type ids share the `items.pak` id space.~~ Struck 2026-08-16: they are `items.pak` record indices — 85% against a 34% control, while the rival `global.res` reading scores 65% against a 63% control, i.e. chance. | [formats/pax-saves.md](formats/pax-saves.md) |
| ~~**96** script opcodes have no verified meaning beyond what their string payloads suggest.~~ Struck 2026-08-16: the binary ships the script compiler's own keyword tables (name strings, pointer array, parallel u32 opcode array) and they name **120 of 141**, agreeing with all seven opcodes named earlier by unrelated routes and refuting three (67/75/76 are `SetVar`/`IncVar`/`DecVar`, not `atmo_rg`). **21 remain unnamed** — 0, 27–34, 39–44, 47, 101, 102, 111, 122, 123 — and a NAME IS NOT A BEHAVIOUR: what each opcode does to the world is still unmeasured. The 66 zero-width tags are presumably operators whose identity sits in handlers already located. | [formats/script-bytecode.md](formats/script-bytecode.md) |
| `world/static.pak`'s `WldxEntry +0x08` slot is empty in every retail cell but populated in 73 prerelease cells — a dropped layer whose target table is unidentified. | [formats/world-sectors.md](formats/world-sectors.md) |
| **(opened 2026-08-17, rows 1019/1020)** **The per-cell `floor.pak` overlay tile SELECTION.** The largest remaining difference in the start scene (~2% of the frame, the cobbles at the screen's bottom corners) is this layer drawing the wrong tiles: nulling `floor_pak` repaints 91.7% of that corner while `--noobjects` moves 5.3%, and disabling overlays outright leaves the whole-frame delta unchanged — the port draws overlays there as wrong as drawing none. Three hypotheses are already refuted: the `next == self+1` chain-walk rule truncates nothing here, the composite order is correct (reversing it makes the corner markedly worse), and it is NOT a placement offset (edge correlation over ±40px returns (0,0) at 0.45, against 0.9965 sixty pixels away in the same sector). Everything downstream of the tile id is correct. | [formats/world-sectors.md](formats/world-sectors.md) |
| **(opened 2026-08-17, row 1016)** **The authored draw order for building parts.** Object placements now sort at the sprite's foot, gated on sprite width so a room shell is not ordered in front of its own contents. One case remains wrong: a candle standing on a wine rack's shelf is covered by the rack, because a single painter key cannot order a sprite that CONTAINS another. Retail draws building parts in an authored order — the part index in `<BUILDING>_<level>_<part>` is the obvious candidate — and it has not been recovered. An aspect-ratio gate was measured and refuted at every ratio 1.5–4.0. | [formats/world-sectors.md](formats/world-sectors.md) |
| **(opened 2026-08-17, row 1021)** **How a filled combat-art slot is composited.** The art is not in `GUI_MAIN_01`…`06` (best error 56, where genuine finds score under 1.0) — each art has its own texture, 440 `GUI_MOVE_*`/`GUI_MOVES_*`/`GUI_SPELL_*`/`GUI_SPELLS_*` entries, each a 256×256 with the picture in the top-left ~64×64. But a slot is at least three layers: the icon is a dark greyscale disc while retail shows those swords blue over a green orb inside a gold ring. Shape correlation across all 440 tops out at 0.40 with no separation, so search is exhausted from both directions; what is missing is the compositing rule (the backing colour's source and the framing ring). | [formats/ui-taskbar.md](formats/ui-taskbar.md) |
| ~~`wpmod.bin`'s record length rule, and its field names.~~ Struck 2026-08-16: length = 54 + 6×int[53]; and the tag→column map is transcribed from the compiler that writes the file — `mod:`=int[6], `var:`=int[7], ten channel triples at int[8..37], `EWT_` type at int[38], `MinLev:`=int[39], `MinRare:`=int[40]. Its `blk[1]` id namespaces are closed too (0 = `Spell:`, 599+skill, 801–820 = `Bonus:`), as is the block's shape: it is a tagged union whose magnitude slot depends on the source section. What the magnitudes are *denominated* in stays unknown — no tag names a unit. | [formats/install-inventory.md](formats/install-inventory.md) |
| ~~`treppe.bin`'s key encoding and `world2.bin`'s index space.~~ Struck 2026-08-16: treppe packs `(level<<26)｜(y<<13)｜x` (2494/2494 = 100.0000% vs a 61.3% control — the earlier refutation had the wrong divisor), and world2 is a u16 sector-presence grid set-identical to keyx. `static10_18.bin`'s key space is still refuted as a packed position (45.5% against a 60.5% baseline). | [formats/install-inventory.md](formats/install-inventory.md) |
| ~~Why `world.bin` lists only 3854 of the 6050 sectors.~~ Struck 2026-08-16: it is a legacy-savegame remap listing the pre-expansion sectors, read only when `floor.pak` exceeds 139,999,999 bytes. `static10_18.bin` is the same idea for triggers (save v10→v18); neither is needed by a port starting from current saves. | [formats/install-inventory.md](formats/install-inventory.md) |
| Whether `treppe.bin` is queried at all — no lookup site was found by member-offset search, and `sacredserver` has no `treppe` string. | [formats/install-inventory.md](formats/install-inventory.md) |
| `vectoren.bin` section 2's two enums at `+0x104` and `+0x108`, and whether the dynamic-quest region system shipped functional — its content is placeholder (`ToDo:-1.<slot>`) in every base tree. | [formats/install-inventory.md](formats/install-inventory.md) |
| What references a `wea.bin` equipment pool (0…255) or a `sndprofiles.pak` profile index (0…8191). Neither `items.pak` nor `creature.pak` carries a column that agrees. | [formats/install-inventory.md](formats/install-inventory.md) |
| ~~What selects the current music/atmosphere profile as the player moves.~~ **Struck 2026-08-16 (row 960).** It is in neither of those because it is a per-sector field in `world/sectors.keyx`: 6050 records of 768 bytes (and `256 + 6050*768` is the file length exactly), each carrying a climate byte and a 256-byte `cSectorEnvironment` with a region id, a `SOUND_FX_*` music id and a secondary atmosphere id. `sub_80DB27C` reads them on sector CHANGE. `sndprofiles.pak` is ruled out — it is the per-creature combat sound variation sets. | [formats/install-inventory.md](formats/install-inventory.md) |
| **(indexed 2026-08-16)** The `.acs` argument encoding: a call with an inline string emits an extra word before its arguments where a call without one does not. | [formats/armalion-acs.md](formats/armalion-acs.md) |
| **(indexed 2026-08-16)** HP is in no table read so far, and neither are the attack and defence ratings — `creature.pak` carries base *attributes* and retail derives the combat numbers from them. | [formats/creature-pak.md](formats/creature-pak.md) |
| **(indexed 2026-08-16, advanced 2026-08-17 row 1015)** The life and mana gauges. All 46 `cUI_Taskbar2` functions were enumerated and the class references no orb, globe or fill-bar art and computes no fraction or scissor rect, so whatever draws them is elsewhere. **That "elsewhere" now has a name and a lead.** The claim that `GUI_main_02`'s 95x107 elements belong to the mercenary window is RETRACTED: the block at that sheet's origin is the PLAYER PORTRAIT frame, drawn by retail at (932,15) and now drawn by the port, matched to the pixel. So the portrait is a separate window `cUI_Taskbar2` correctly does not reference -- and the same sheet carries three beaded segmented BARS (yellow/green/red, plus framed variants), one of which retail draws green immediately under that portrait at ~(942,124), 82x11. Find the window; the gauges are likely in it. | [formats/ui-taskbar.md](formats/ui-taskbar.md), [engine/game-wiring.md](engine/game-wiring.md) |
| **(indexed 2026-08-16)** The Armalion script API id space is not joined to retail's opcodes: ~85 API names, 62 recorded with handler addresses, against a 141-entry dispatcher whose ids do not line up. | [builds/armalion-source-tree.md](builds/armalion-source-tree.md) |
| ~~**The whole symbolic `Res:` namespace resolves in no shipped file**~~ **Struck 2026-08-16 — the claim was false and the cause was our own arithmetic.** The name hash was transcribed correctly but reimplemented in 64-bit GDScript, while retail runs it in int32 where `113*v` wraps from the fifth character on. The two agree for exactly four characters, which is every numeric key in the shipped files and no symbolic one. With the wrap in place NPC names resolve 1521 of 1521 (was 1377), 319 of 1242 static `QuestBook` keys come back as English quest-log prose, and the rest are runtime-composed rather than absent. See row 954. | [formats/global-res.md](formats/global-res.md) |

Nothing is open on `.pak` framing, `tiles.pak`, `global.res`, `mixed.pak`, or
the `balance.bin` layout and key names. The world cell record is fully
accounted for; the one loose end is the dropped layer noted above.

Audio is no longer open: `sound.pak`, `sndprofiles.pak` and `mp3/` were
decoded end to end on 2026-08-16, including the `SOUND_FX_` symbol table in
the executable that names every blob.

## Engine behaviour

| Question | Where it lives |
|---|---|
| The combat RESOLUTION step is undecoded: how damage meets resistance, criticals, and what the weapon-slot flag selects. To-hit, the derived-stat kernel, and **both attack and defence ratings end to end** (row 959) are recovered; hit points are in no table read so far. | [engine/combat-formulas.md](engine/combat-formulas.md) |
| No recovered name has been confirmed by behaviour. Cross-source agreement covers 34 of 131 against the Armalion name catalogue, plus 18 of 130 against the RTTI vtable walk at the same address (0 disagreements). The behavioural arm still covers none. | [engine/decompilation-coverage.md](engine/decompilation-coverage.md) |
| 11,303 of 13,716 callables in the retail binary have no identity, and the vtable route that produced the other 2,413 is saturated. Widening the assert-string pattern past `c[A-Z]` adds ~9 names, so that route is close to saturated too. | [engine/decompilation-coverage.md](engine/decompilation-coverage.md) |
| ~~Which creature-struct offset is which named attribute.~~ **Struck 2026-08-16 (row 959).** `sub_81FA5AA` and `sub_81FA622` read `AT = f32[+0x5A] * f32[+0xE6]` and `PA = f32[+0x5E] * f32[+0xEA]`; `CalcResults` builds the bases as `0.5*(STR+DEX)` and `0.2*STR + 0.8*DEX` over the first and third of the six u16 at `+0x10`, plus a flat gear term, from a per-class coefficient table that is identical for every class in retail. `ProzAW` scales a non-hero's ratings by difficulty. The `+0x3C/+0x3E/+0x40` triple is NOT AT/PA — it is three speed percentages. | [engine/combat-formulas.md](engine/combat-formulas.md) |
| The Granny converter cannot be driven past a null dereference on any real Sacred `.GRN`. Reopening means a different converter build, not a different way of calling this one. | [engine/granny-runtime-oracle.md](engine/granny-runtime-oracle.md) |
| **(opened 2026-08-17, row 990)** **What decides OUTDOOR walkability.** The region grids are the shipped navmesh (row 308) but cover only 26.55% of the start area. Everything else falls through to `byte26 != 1 && byte26 != 4`, taken from Armalion's `sub_4B0220` — and byte 26 is a **terrain height corner** in retail data (`+0x18`/`+0x19`/`+0x1a`/`+0x1b` statistically identical, `r=+0.86` between neighbours), while the retail binary contains **no** `cmp byte ptr [reg+0x1A], 1` or `, 4` at all. So 96.06% of the port's walkable ground rests on a comparison against terrain height. No cell field tested (`+0x1f` low/high nibble, `+0x1e`, `+0x1a`, static chain, floor chain) predicts the navmesh's own answer better than its 91.01% base rate. Two untried leads: Armalion's second branch (`*(DWORD*)cell & 4` → a static-object chain lookup, never ported), and whatever retail's own movement code reads instead. | [formats/world-sectors.md](formats/world-sectors.md) |
| ~~**Retail's player walk speed.**~~ **Struck 2026-08-17 (row 1013).** Measured twice from retail itself via the autopilot shim: click-walk along one iso axis, framebuffer captures every 500 ms, phase-correlate — the camera tracks the player, so its total scroll is the walk. Both runs: 5.77 cells net over ~4.1 s = **1.4–1.6 cells/s**; zoom pinned by sprite height from the same frames. `Movement.CELLS_PER_TICK` set to 1.5 (was 4.0 — the port ran ~2.7× too fast). Residual ±0.15 cells/s from the 500 ms bracketing; a tighter number wants an in-process position re-scan in the same run. | [engine/game-wiring.md](engine/game-wiring.md) |
| ~~**Water is not rendered at all.**~~ **Struck 2026-08-17 (rows 1008/1011).** The port draws the animated liquid pass: 14-record material table read off the Linux binary's unrolled initialiser and rendered as extra sector surfaces above the overlays; controlled A/B, `b>r` on 0% of sea pixels before, 100% after. Row 1008's first table reading was one slot off — it missed the second of two adjacent `B_WATER` blocks — and its claim that row 669's names were shifted is retracted: **row 669's names were correct**. The frame counts pin the true order (the pak's only 20-frame sets land on records 6/7 only under it) and `liquid_check` now asserts exactly that. | [formats/world-sectors.md](formats/world-sectors.md) |
| ~~**Which file holds the per-sector liquid material id.**~~ **Struck 2026-08-17 (row 1010).** It is in the keyx record itself: the 768-byte-record loader (`sub_80EF4EE`) memcpy's record bytes `0x1E9..0x2E8` (a 0x100-byte environment block) into a per-sector slice hung off the runtime sector at `+0x17C`, so the two ids are keyx record bytes **736** and **737**. Validated over all 1360 liquid sectors — every value in 0..13, sea = `B_WATER`, 22 sectors carry two different liquids at once. Wired as `World.liquid_id`; lava maps and mixed sectors now draw their own materials. | [formats/world-sectors.md](formats/world-sectors.md) |
| ~~**Liquid animation cadence**~~ **Struck 2026-08-17 (row 1011)** — measured, not a field: the draw picks `frame = ((ms >> 1) & 0x3FF) · count / 1024`, a fixed 2048 ms loop for every set (≈24.4 fps at 50 frames, ≈9.8 at 20); wired. Also recovered and wired: record `+0xD0` is a signed alpha multiplier over the cell's corner DEPTH bytes — retail's shoreline fade. **Still open: the reflection pass.** `+0xCC` gates a pass-1 vertically-mirrored untextured quad modulated by ambient·(−8·depth); its geometry is read but its blend state goes through untraced render-state calls (`sub_83B41F2`), and a wrong blend looks worse than the pass's absence. | [formats/world-sectors.md](formats/world-sectors.md) |
| ~~**DUNKELELVE (models.pak entry 402) builds no rig.**~~ **Struck 2026-08-17 (row 1009).** The pairing was never a matching problem — the file states it: each `FormMesh` (0xCA5E0C03) carries a payload int32, the 1-based index over ALL Mesh nodes of the mesh its bone section belongs to. Validated corpus-wide (unique, in range, satisfies every size on all 971 declaring entries; agrees with the strict rule on all 807 it decides) and landed as `Models._pair_by_reference`, primary over both fallbacks. All seven class bodies build; decode count 920 → 955; the Dark Elf renders as a coherent figure. | [formats/granny-grn.md](formats/granny-grn.md) |

## Deliberately not open

These are settled decisions, not gaps, and re-raising them costs time:

- **Building construction and the interior/exterior swap** are not documented
  in this repository. Their settled half is entangled with render-architecture
  choices about the port, which are decisions rather than facts about a
  format.
- **What the individual balance tunables do to play** is not a format
  question. It needs the engine to consume them first.

## Not questions — integration debt

Decoded, written up, and simply not wired into the engine yet. Named here so
they are not mistaken for research:

- `triggers.pak` has a reader and no `formats/` class, though
  `checks/trigger_check.gd` and six probes do read it.
- `formats/equipment.gd` and `formats/wpmod.gd` each have a passing gate and
  **no production caller** — `equipment_check.gd` and `wpmod_check.gd` are the
  only things that construct them. Decoded and unwired, which is this
  section's definition. (`formats/sectors.gd` was the third of these until
  2026-08-16, when `main.gd` began firing the environment lookup on a change
  of the player's sector; link 6c in `engine/game-wiring.md` claimed "yes"
  for the whole of the interval it was gate-only.)
- `DefPos.bin` is **no longer integration debt**: it is a regenerable cache of
  `startcode.bin` + `funkcode.bin` that retail rebuilds whenever its `1234`
  magic is absent, which is true of 19 of the 20 shipped copies. A port has no
  obligation to read it.

---
Provenance: the `## Open` section of each document named above; the findings
log for the integration items.
