# `balance.bin` — the rules dataset

**Status: layout determined, keys named.** This is the most
reimplementation-relevant result in the project: **the rules layer no longer
needs decompiling, it needs reading.**

`install/bin/balance.bin`, 24,328 bytes, md5 `15ab686284b1ec1c386beb4ac42af5ff`.

## The layout

> **`balance.bin` is a flat record of little-endian `int32`/`float32` at
> fixed absolute byte offsets.** No per-key index, no string table, nothing
> self-describing. The key *names* live in the **executable**; the *values*
> live in the file; **nothing but hardcoded offsets binds them.**
>
> Two earlier structural hypotheses were refuted (log rows 191–195). Both
> assumed the file carried its own structure. It does not — which is why no
> self-describing reading could ever have worked. The refutations stand as
> measurements: don't retry them.

The difficulty grid is five `float[6]` arrays at stride 24, slot order
`[silver, gold, platinum, niobium, global, bronze]`, at offsets 1812 / 1836 /
1860 / 1884 / 1908.

Offsets were cross-read from `SacredMagician` (a community Kotlin editor
using ~50 hardcoded absolute offsets, "compatible from Sacred 1.0 to
2.29.14") and then **verified against the real file**, not trusted.

## The key names

Contiguous in the executable at VA `0x95b790`–`0x95ca78`, parsed by
`balancing_parseKeys`. **380 unique keys**, German-named. **361 of the 379
fields are now named** — see [balance-keymap.tsv](balance-keymap.tsv) and
[balance-keymap.json](balance-keymap.json).

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

---
Provenance: `tools/balance_keymap.py`; findings log rows 191-195; the key table
in `balance-keymap.tsv`.

