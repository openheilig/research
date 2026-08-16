# Open questions

**Status:** Standing
**Purpose:** Every open item from every document in this repository, in one
list, so the state of the project is one read rather than eleven.

Each entry is the `## Open` section of the document it names — that document
is the authority, this is the index. If you close one, close it there and
strike it here.

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
| What selects the current music/atmosphere profile as the player moves — it is in neither `global.res` nor `bin/*.bin`. | [formats/install-inventory.md](formats/install-inventory.md) |
| **The whole symbolic `Res:` namespace resolves in no shipped file** — 0 of 2435 in one script tree, including every one of the 3676 `QuestBook` and 3367 `Text` operands, so no quest-log line and no line of dialogue can be displayed from this install. The 389 `credits.txt` keys are one corner of it. Struck as a *format* question and restated as a *data* one: the mapping is not present to be found. | [formats/global-res.md](formats/global-res.md) |

Nothing is open on `.pak` framing, `tiles.pak`, `global.res`, `mixed.pak`, or
the `balance.bin` layout and key names. The world cell record is fully
accounted for; the one loose end is the dropped layer noted above.

Audio is no longer open: `sound.pak`, `sndprofiles.pak` and `mp3/` were
decoded end to end on 2026-08-16, including the `SOUND_FX_` symbol table in
the executable that names every blob.

## Engine behaviour

| Question | Where it lives |
|---|---|
| The combat RESOLUTION step is undecoded: how damage meets resistance, criticals, and what the weapon-slot flag selects. To-hit and the derived-stat kernel that builds the damage and resistance numbers are recovered. | [engine/combat-formulas.md](engine/combat-formulas.md) |
| No recovered name has been confirmed by behaviour. Cross-source agreement covers 34 of 131 against the Armalion name catalogue, plus 18 of 130 against the RTTI vtable walk at the same address (0 disagreements). The behavioural arm still covers none. | [engine/decompilation-coverage.md](engine/decompilation-coverage.md) |
| 11,303 of 13,716 callables in the retail binary have no identity, and the vtable route that produced the other 2,413 is saturated. Widening the assert-string pattern past `c[A-Z]` adds ~9 names, so that route is close to saturated too. | [engine/decompilation-coverage.md](engine/decompilation-coverage.md) |
| Which creature-struct offset is which named attribute: `+0x56` feeds every damage channel, `+0x4a` and `+0x4e` are two weapon slots. The skill slots at `+0x24` are now read (8 bytes, one skill type each), so the same route — a UI printer that pairs an offset with a resource id — should reach the rest. | [engine/combat-formulas.md](engine/combat-formulas.md) |
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
- `DefPos.bin` is **no longer integration debt**: it is a regenerable cache of
  `startcode.bin` + `funkcode.bin` that retail rebuilds whenever its `1234`
  magic is absent, which is true of 19 of the 20 shipped copies. A port has no
  obligation to read it.

---
Provenance: the `## Open` section of each document named above; the findings
log for the integration items.
