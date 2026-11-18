# `balance.bin` — the rules dataset

**Status:** Read
**Purpose:** The layout of the rules dataset and the names of its keys.

The most reimplementation-relevant result in the project: **the rules layer no
longer needs decompiling, it needs reading.**

`install/bin/balance.bin`, 24,328 bytes, md5 `15ab686284b1ec1c386beb4ac42af5ff`.

## The layout

> **The fixed tuning region has little-endian `int32`/`float32` slots at
> recovered absolute file offsets; their key names are not serialized.**
> The earlier claim that the *entire file* was an unframed flat blob is
> incomplete: the shipping loader consumes a length-prefixed fixed region
> followed by counted collections. See the 2026-10-07 correction below.
>
> The earlier structural hypotheses refuted in log rows 191–195 remain
> refuted. They do not establish that the trailing collections lack framing.

The difficulty grid is five `float[6]` arrays at stride 24, slot order
`[silver, gold, platinum, niobium, global, bronze]`, at offsets 1812 / 1836 /
1860 / 1884 / 1908.

Offsets were cross-read from `SacredMagician` (a community Kotlin editor
using ~50 hardcoded absolute offsets, "compatible from Sacred 1.0 to
2.29.14") and then **verified against the real file**, not trusted.

## The key names

Contiguous in the executable at VA `0x95b790`–`0x95ca78`, parsed by
`balancing_parseKeys`. **380 unique keys**, German-named. **361 of the 379
fields are now named** — see [balance-keymap.tsv](generated/balance-keymap.tsv) and
[balance-keymap.json](generated/balance-keymap.json).

The two that unlock combat: **`AW` = *Angriffswert*** (attack rating) and
**`VW` = *Verteidigungswert*** (defence rating). Those are exactly the to-hit
inputs — see [../engine/combat-formulas.md](../engine/combat-formulas.md).

Families present: `ProzAW` / `ProzHP` / `ProzDmg` / `ProzRes`; `FernAW*`
(ranged); `AWFakChamp` / `AWFakBoss` / `VWFakChamp` / `VWFakBoss`;
`RunenProLevel_*`; `BalChar*R` / `BalChar*D`; `BalanceMag` / `BalanceRes` /
`BalanceDmg`; `regionkill`; `ITBonus_MEDUSA` / `ITBonus_DRAGON_*`;
`PriceAwVw:` / `PriceLevel:`; `RareSlotGold` / `Silber` / `Bronze`;
`SpawnEnemysMP` / `SP`; `skilllearn1..6`; class codes `DAEM DWAR VAMP WELF
DELF MAGE SERA`; and `_w` / `_s` / `off` triples for ~30 skills.

## Cross-build identity

- **380/380** retail balance keys are present in `install/sacred`, the Linux
  binary this project actually runs. Zero absent.
- **8/8 shipped `.bin` data files are byte-identical** between the Windows
  disc and the Linux install — `Balance.bin`, `Rust.bin`, `MultiStart.bin`,
  `merc.bin`, `sets.bin`, `wea.bin`, `treppe.bin`, `static10_18.bin`.

⇒ The same data file serves both platforms.

> A first pass reported three of these **missing**. The comparison loop used
> lowercase names; Windows ships them capitalised. Re-running with correct
> casing showed all eight identical. Had it not been re-checked, a false
> "Linux-only data files" claim would have entered the record.

## A second route to the same data

Retail Windows ships a **live Excel→C-header balance exporter**. `CTRL+K` in
the shipping game writes `defaults.h`, `xls_sheroconst.h` and
`xls_smovetypes.h` into the game folder, from `sacred.xls`. That is an
independent route to the dataset with no decompilation involved.

The Linux port does **not** carry it — `sacred.xls`, `defaults.h`,
`BALANCING:` and `CTRL+K` are all absent. The facility is Windows-only.

## The balance headers are generated from `sacred.xls`, 2026-08-25

Unpacking Sacred Plus's `Sacred.exe` — a repack whose code section is on disk
where retail's StarForce-wrapped one is not — exposes the build pipeline's own
strings. They are present in every Windows build we hold (`gold228-rus`, the
Windows disc, Sacred Plus) and in **none** of the Linux `sacred`, which dropped
the exporter:

    BALANCING: Getting active Excel object..     BALANCING: No Excel is running!
    BALANCING: Writing defaults.h..
    BALANCING: Writing xls_magictypes.h..
    BALANCING: Writing xls_sheroconst.h..
    BALANCING: Writing xls_smovetypes.h..
    // generated from sacred.xls - do not edit manually!

So the game drives a live Excel instance over a workbook called **`sacred.xls`**
and writes four C headers from it. Two of the three `xls_*` names line up with
the spreadsheets recovered from VK and verified in row 1079 —
`SpellType.xls` is a **magic-type** table and `xls_magictypes.h` a magic-type
header; `SpellMove.xls` is a **move** table and `xls_smovetypes.h` a move-type
header.

> That correspondence is naming plus the 145-of-145 `global.res` resolution
> those sheets already passed. It is **not** proof that the VK files are sheets
> of Ascaron's `sacred.xls`; nobody has seen that workbook. `xls_sheroconst.h`
> has no counterpart among the recovered files.

## Starting equipment — two tables, a text format, and no data, 2026-08-25

`cEngine::initGame` names the step in its own trace: `cEngine::cEngine() equipe`,
at `0x80b7692` in the Linux binary, right after `initHero(e)`. It fills the new
hero from **two arrays**, and the indexing is read off the code, not guessed:

    dword_8B89740[class * 20 + slot]   worn      — 8 classes x 20 slots, slots 0..18 used
    dword_8B899C0[class *  8 + i]      carried   — 8 classes x 8 entries

Each entry is an item id; **an entry of 0 is skipped**, so a zeroed table equips
nothing. Class index comes from the hero id with `8` and `9` folded down by one.

**Exactly one function writes either array**, and it is a **text parser** — the
same one that reads `BalanceDmg`, `BalanceRes`, `regionkill` and 390 other keys.
The two keys are `equip` and `inventory`, and the grammar falls out of the
`strchr('=')` / `strchr(',')` / `strtol` sequence:

    equip=<CLASS>,<slot>,<itemid>
    inventory=<CLASS>,<itemid>

`<CLASS>` is matched with `strncasecmp(...,4)` against a fixed list:

| SERA | GLAD | MAGE | DELF | WELF | VAMP | DWAR | DAEM |
|---|---|---|---|---|---|---|---|
| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |

### No shipped file carries such a line

Searched: the whole retail install, `bin/balance.bin` itself, the Armalion
prerelease, Sacred Plus, every community mod and tool, and the 88,914-item VK
corpus. **Zero occurrences of `equip=`.** `balance.bin` contains none of the
strings `equip`, `inventory` or `SERA`.

> **The reading this supports, with its risk stated.** On a stock install both
> tables stay zero, every slot is skipped, and **retail equips a new hero with
> nothing** — what the start capture shows is her base rig and its own texture,
> not worn items. What is PROVEN is the layout, the grammar, the class map, the
> zero-skip and the absence of any input. What is NOT proven is that no other
> path fills them: the parser is reached through a pointer rather than a direct
> call, so its call site was not traced, and a file we do not hold cannot be
> ruled out by searching the ones we do.

**Consequence for the port, MEASURED 2026-08-25 (row 1102).** `engine/main.gd`
dresses the Seraphim from `sets.bin` **set 6**, which
[install-inventory.md](install-inventory.md#setsbin-fully-read) records as *the
seven Seraphim pieces* — a magic item SET like "Uriel's Legacy", not a starting
kit. `main.gd:47` already said so. A `--nodress` flag was added to test it:

| port config | world band | hero box |
|---|---|---|
| dressed, `START_SET` 6 | 10.95% | 31.10% |
| **bare rig, `--nodress`** | **10.92%** | 29.94% |
| mesh hidden, `--hideplayer` | **10.88%** | 28.81% |

**Undressing her recovers 0.03pp of her 0.78pp — about 4% of her cost — and
hiding her entirely is still better than either.** The two port renders differ
from each other by 2,324 px, so the outfit is not a no-op on screen; it simply
does not move the score.

**So the port's Seraphim is wrong in a way equipment cannot fix.** The remaining
~96% is the base rig itself — mesh, texture, pose or scale — and that is where
the 0.34pp surface gap of row 1100 actually lives. Dropping set 6 is correct on
the evidence of this document, but it is worth 0.03pp and not a fix.

> **Not changed:** `START_SET` is still the default. Retail equipping nothing
> argues for making `--nodress` the default, but `main.gd:47` keeps set 6 so the
> composition path is exercised by the default run, and that trade is a human
> call rather than a measurement.

## Open

The scalar key map does not name or decode the trailing keyed collections.
The HorseData framing is source-established below, but a selected profile's
numeric contents still require an owned-media lookup and an independent
runtime map witness. What individual tunables *do* remains a separate question.

## Counted tail and HorseData — correction, 2026-10-07

The binary initializer, independently corroborated in Linux and Windows
builds, consumes this sequence from the file start:

| Region | Framing |
|---|---|
| Fixed tuning data | `u32 byte_length`, then exactly `byte_length` bytes |
| Collection 1 | `u32 count`; each entry has a four-byte key, `u32 row_count`, then `row_count` twelve-byte rows |
| Collection 2 | `u32 count`, then `count` eight-byte key/value pairs |
| Collection 3 | `u32 count`, then `count` eight-byte key/value pairs |
| Collection 4 / HorseData | `u32 count`; each entry is a signed `i32` key followed by seven signed `i32` values |

The HorseData record's first integer supplies the baseline movement
percentage for creature **class 4 (horse)** before conversion to signed16.
The class predicate is not the class-1 hero predicate. Ordinary heroes take
the separate source baseline of 130, except type 5's branch of 120.
HorseData is a keyed record map, not an attribute-derived interpolation
curve; its reader must not invent a profile for missing media.

The native container location at loaded-base plus 4968 is **not a file
offset**. The reader must advance through the variable-sized preceding
regions. None of those preceding counts may be skipped by assuming a stock
size. Unnamed scalar slots and framed trailing collections coexist.

Collection 5 is the ordered `UniqueThreshold` vector: `u32 count`, then
`count` records containing an `i32` key and three `f32` parameters
(`86607DC` collection read; `8134A18` cache lookup). Its in-memory
container at loaded-base plus 4992 is not a file offset either.
`Balance.unique_thresholds()` retains the **first serialized match** for
each key, matching the original lookup rather than the last-write policy
of other maps. The duplicate-key correction leaves all five distinct
shipped rows unchanged; absent-key cache defaults remain separate from
present data. Source/original-byte evidence and failing-before/passing-after
reader/source-consumer proof are retained in
`tmp/workflow-reference-20261016/w1-existing-threshold-duplicate-precedence/`.
This qualifies the reader contract, not a current actor or loaded cache.

The collection-2 map is not itself the quality-selection descriptor vector.
`8369CE6` rebuilds that vector in fixed key order **0 through 17**, narrows
each map value to a word, and supplies a zero candidate for a missing key.
`86932DC` preserves the word; the selected `81B0BC0` consumer reads it as
unsigned16. `Balance.item_quality_weights()` now returns that descriptor
view, not arbitrary raw signed32 map pairs. Extra map keys do not enter the
view. Raw indexing, rarity-map guards and the other map policies are unchanged.
Full original rebuilds with real erase/lookup/pair/spare-append callees and
actual reader comparisons are retained at
`tmp/workflow-reference-20261016/w1-balance-scalar-map-integer-domain/`.
The owned table is unchanged. Missing-key insertion and full-capacity
allocation remain explicitly unexecuted; no RNG or current-owner result is
claimed by the descriptor correction.

The selected `81B0BC0` quality loop also zero-extends the boost word
**before** applying its cap of 700. `LootRoll.Producer.quality()` now
preserves that ordering; masking only the cap predicate incorrectly allowed
high-word bits into the multiplier. Original deterministic arithmetic under
explicit software precision controls and actual-selector counterexamples
are retained at
`tmp/workflow-reference-20261016/w1-quality-weight-selection-arithmetic/`.
These use declared mathematical thresholds, not observed RNG outcomes.
Probability staging, draw chronology, physical FPU mode and actor admission
are unchanged and are not qualified by this integer-width correction.

The variable-probability path spills its probability to f32 but retains the
subsequent scale on the FPU stack (`81B1379`, `81B13C0..13D4`). The port
removed only an extra forced f32 conversion of that scale, retaining its
existing host scalar and source probability conversion. An actual-helper
half-probability frame exposed the defect at weight 200, boost 1 and numeric
threshold 1161: all three explicit original software precision controls
select quality 0, while the extra conversion selected quality 1.
The actual consumer now passes that discriminator and the retained nominal
unit/half-probability cases. Evidence:
`tmp/workflow-reference-20261016/w1-quality-variable-probability-surface/`.
This is not a physical FPU-mode selection or a global floating-point parity
claim; draw chronology and actor/source-R0 gates remain unchanged.

`engine/formats/balance.gd` now exposes `horse_profile(profile: int)`.
Success returns `{profile, values}`, where `values` is a seven-element
`PackedInt32Array`; a zero first integer is preserved. Failure returns
`{error, kind}` with `missing_data`, `missing_collection`, `malformed_data`
or `absent_profile`. A valid empty collection means absent profile, not
missing collection. Framing and the key-to-record index are validated and
cached only on first lookup, against the bytes opened through `Pak.resolve`.
Unsigned lengths/counts are bounded by remaining bytes before stride
multiplication or iteration; there is no arbitrary record-count limit.

`engine/checks/balance_horse_check.gd` contains authored, non-retail
fixtures for variable fixed lengths, preceding nested rows/pair maps,
signed fields, zero-versus-absent profiles, absent collection counts,
truncation of every byte prefix, oversized unsigned counts and incorrect
preceding framing. Its optional `--horse-profile=<key>` lookup reads owned
media; `--horse-values=<seven comma-separated signed integers>` compares
against a separately guarded runtime map record. The explicit key and
witness must be provided by measurement, not chosen to make the check pass.
Runtime captures and lookup output remain private; no raw table is embedded
in source. No selected-profile numeric facts or passing-run claims are
established by this correction.

Source provenance: Linux binary initialization in `LGP00014.c:1423–1467`
and deserialization in `linux1002/00055.c:1070–1176`; Windows corroboration
in `ENG00014.c:8750–8893` with prefix helper at `18629–18641`, and
`RUS00014.c:8921–8933`. The text HorseData parser independently establishes
one key plus seven integers (`LGP00014.c:6730–6764`,
`ENG00014.c:17141–17202`, `RUS00014.c:17159–17218`). The first-integer
consumer is established at `LGP00017.c:648–652` and corroborated at
`ENG00011.c:26220–26240` / `RUS00011.c:26132–26149`. These are source
locations for private research, not copied implementations.

The historical starting-equipment conclusion above is also superseded:
the guarded route witness records real worn starter instances, as documented
in [seraphim-native-route.md](../engine/seraphim-native-route.md).
Absence of text key names in the binary does not prove that runtime equipment
tables remain zero. HorseData alone likewise does not establish the final
movement percentage: actual equipment, modifiers, effects and actor state
still belong to the separate runtime calculation.

---
Provenance: `tools/binary/balance_keymap.py`; findings log rows 191-195; the key table
in `balance-keymap.tsv`.

