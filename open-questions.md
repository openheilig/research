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
document.

## Formats

| Question | Where it lives |
|---|---|
| Eight of 3421 animation clips do not decode. | [formats/granny-grn.md](formats/granny-grn.md) |
| ~~One mesh's vertex count disagrees with an outside reading, 279 against 280.~~ Struck 2026-08-15: retail's own index array for that batch resolves to 279, so the outside reading is the wrong one. | [formats/granny-grn.md](formats/granny-grn.md) |
| Bink `.bik` and Miles `.mss` are third-party formats we do not decode. (`mixed.pak` was listed here in error — the engine had read it; struck 2026-08-15 and confirmed in a second build.) | [formats/pak-containers.md](formats/pak-containers.md) |
| ~~Whether the `0xC8` record type ids share the `items.pak` id space.~~ Struck 2026-08-16: they are `items.pak` record indices — 85% against a 34% control, while the rival `global.res` reading scores 65% against a 63% control, i.e. chance. | [formats/pax-saves.md](formats/pax-saves.md) |
| ~~**96** script opcodes have no verified meaning beyond what their string payloads suggest.~~ Struck 2026-08-16: the binary ships the script compiler's own keyword tables (name strings, pointer array, parallel u32 opcode array) and they name **120 of 141**, agreeing with all seven opcodes named earlier by unrelated routes and refuting three (67/75/76 are `SetVar`/`IncVar`/`DecVar`, not `atmo_rg`). **21 remain unnamed** — 0, 27–34, 39–44, 47, 101, 102, 111, 122, 123 — and a NAME IS NOT A BEHAVIOUR: what each opcode does to the world is still unmeasured. The 66 zero-width tags are presumably operators whose identity sits in handlers already located. | [formats/script-bytecode.md](formats/script-bytecode.md) |
| `world/static.pak`'s `WldxEntry +0x08` slot is empty in every retail cell but populated in 73 prerelease cells — a dropped layer whose target table is unidentified. | [formats/world-sectors.md](formats/world-sectors.md) |
| ~~`wpmod.bin`'s record length rule, and its field names.~~ Struck 2026-08-16: length = 54 + 6×int[53]; and the tag→column map is transcribed from the compiler that writes the file — `mod:`=int[6], `var:`=int[7], ten channel triples at int[8..37], `EWT_` type at int[38], `MinLev:`=int[39], `MinRare:`=int[40]. Its `blk[1]` id namespaces are closed too (0 = `Spell:`, 599+skill, 801–820 = `Bonus:`), as is the block's shape: it is a tagged union whose magnitude slot depends on the source section. What the magnitudes are *denominated* in stays unknown — no tag names a unit. | [formats/install-inventory.md](formats/install-inventory.md) |
| ~~`treppe.bin`'s key encoding and `world2.bin`'s index space.~~ Struck 2026-08-16: treppe packs `(level<<26)｜(y<<13)｜x` (2494/2494 = 100.0000% vs a 61.3% control — the earlier refutation had the wrong divisor), and world2 is a u16 sector-presence grid set-identical to keyx. `static10_18.bin`'s key space is still refuted as a packed position (45.5% against a 60.5% baseline). | [formats/install-inventory.md](formats/install-inventory.md) |
| ~~Why `world.bin` lists only 3854 of the 6050 sectors.~~ Struck 2026-08-16: it is a legacy-savegame remap listing the pre-expansion sectors, read only when `floor.pak` exceeds 139,999,999 bytes. `static10_18.bin` is the same idea for triggers (save v10→v18); neither is needed by a port starting from current saves. | [formats/install-inventory.md](formats/install-inventory.md) |
| Whether `treppe.bin` is queried at all — no lookup site was found by member-offset search, and `sacredserver` has no `treppe` string. | [formats/install-inventory.md](formats/install-inventory.md) |
| `vectoren.bin` section 2's two enums at `+0x104` and `+0x108`, and whether the dynamic-quest region system shipped functional — its content is placeholder (`ToDo:-1.<slot>`) in every base tree. | [formats/install-inventory.md](formats/install-inventory.md) |
| What references a `wea.bin` equipment pool (0…255) or a `sndprofiles.pak` profile index (0…8191). Neither `items.pak` nor `creature.pak` carries a column that agrees. | [formats/install-inventory.md](formats/install-inventory.md) |
| ~~What selects the current music/atmosphere profile as the player moves.~~ **Struck 2026-08-16 (row 960).** It is in neither of those because it is a per-sector field in `world/sectors.keyx`: 6050 records of 768 bytes (and `256 + 6050*768` is the file length exactly), each carrying a climate byte and a 256-byte `cSectorEnvironment` with a region id, a `SOUND_FX_*` music id and a secondary atmosphere id. `sub_80DB27C` reads them on sector CHANGE. `sndprofiles.pak` is ruled out — it is the per-creature combat sound variation sets. | [formats/install-inventory.md](formats/install-inventory.md) |
| **(indexed 2026-08-16)** The `.acs` argument encoding: a call with an inline string emits an extra word before its arguments where a call without one does not. | [formats/armalion-acs.md](formats/armalion-acs.md) |
| **(indexed 2026-08-16)** HP is in no table read so far, and neither are the attack and defence ratings — `creature.pak` carries base *attributes* and retail derives the combat numbers from them. | [formats/creature-pak.md](formats/creature-pak.md) |
| **(indexed 2026-08-16)** The life and mana gauges. All 46 `cUI_Taskbar2` functions were enumerated and the class references no orb, globe or fill-bar art and computes no fraction or scissor rect, so whatever draws them is elsewhere. | [formats/ui-taskbar.md](formats/ui-taskbar.md), [engine/game-wiring.md](engine/game-wiring.md) |
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
