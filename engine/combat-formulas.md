# Combat formulas

**Status:** Partial
**Purpose:** The combat arithmetic recovered so far: to-hit end to end, and the
derived-stat kernel that feeds damage and resistance. The resolution step that
consumes them is not decoded.

Two things are recovered. **To-hit** is complete and confirmed in two binaries.
The **derived-stat pass** -- how a creature's damage and resistance numbers are
built from its attributes and the balance table -- is read off retail Linux and
is confirmed only there.

## To-hit

```
hit% = clamp( 200·AT/(AT+PA) · ALVL/(ALVL+DLVL), 5, 95 )
```

where `AT` = *Angriffswert*, attack rating; `PA` = *Verteidigungswert*,
defence rating; `ALVL` / `DLVL` = attacker and defender level. The names come
from the balance key table — see
[../formats/balance-bin.md](../formats/balance-bin.md).

As the programmer wrote it: `2·AT/(AT+PA)` scaled to percent, times the level
ratio, clamped to `[5, 95]`.

```c
int __stdcall to_hit(uint16 AT, uint16 PA, uint16 ALVL, uint16 DLVL)
{
  int16 v5 = (int64)(((double)AT + (double)AT) / (double)(PA + AT)
                     * 100.0 * ((double)ALVL * 1.0 / (double)(DLVL + ALVL)));
  if (v5 < 5)  v5 = 5;
  if (v5 > 95) return 95;
  return v5;
}
```

## Roll semantics

From the call site in `cCreature::receive_event`:

```c
v40 = to_hit(AT, PA, ALVL, DLVL);
result = rand(0, 100);
if (result >= v40) { /* miss */ } else { /* damage */ }
```

So the roll is `rand(0,100)` and the test is strict: `roll < hit%` hits.

## Why it is trustworthy

Two independent binaries agree.

- In `armalion.exe` it is **inlined** into `cCreature::receive_event`
  (`sub_41FA30`), reached from the DAMAGE trace string in one xref, then one
  mandatory thunk hop to `0x4251D0`.
- In `armalion_us.exe` it is **its own function** (`sub_428790`), reached in
  two hops from the same trace string. Algebraically identical.

The event-ID enum falls out of the same dispatch switch: 1, 3, 4, **7 =
DAMAGE**, 8 = heal, 10 = move/teleport, with `0x101` a special case and
`> 0x101` re-based by `−259` into a second switch.

## The clamp is absent from retail — and that is the interesting part

> Hunting the same `cmp …, 95` shape in retail Gold 2.28 found four candidate
> sites in 4.7 MB, and **all four were eliminated**: two are the script
> lexer's identifier test (95 decimal is `0x5F`, `'_'`), one is an adjacent
> parser with zero string refs, and the largest — `sub_5EB010`, 0x738b bytes,
> **393 string refs, all balance-table keys** — is the balance key parser.
>
> **The hardcoded clamp does not exist in retail.** The elimination was worth
> more than a hit would have been: it showed retail moved the tunables out of
> code and into a named data table. `sub_5EB010` → `balancing_parseKeys`,
> `sub_65CDE0` → `script_lexer_isIdentChar`.

⚠️ A method warning attaches to that search. The instruction-query tool
**silently caps at 200,000 instructions**, reporting `truncated: true` with a
`next_start`. A `count: 0` from a truncated scan is indistinguishable from a
real negative. Scans must be chunked until *every* chunk reports
`truncated: false` — otherwise the negative is worthless. See
[../method/discipline.md](../method/discipline.md).

## Where the tunables actually land

That elimination said retail reads its tunables from a named table. This is
where they land. `FUN_0812d25c` in `sacred_orig` is the Linux
`balancing_parseKeys`, and each key is parsed by one flat, repeating shape:

```
PUSH  "BalanceDmg"            ; the key
CALL  FUN_0812804c            ; locate it in the loaded text
CALL  strrchr(tok, '=')       ; step past the '='
CALL  __strtod_internal
FSTP  float ptr [0x08b8d2a0]  ; <- the destination global
```

so a key's name and its runtime address fall out together. 348 of them are
tabulated in
[../formats/generated/balance-globals.tsv](../formats/generated/balance-globals.tsv),
with the functions that read each one.

**Every key sits at `global = 0x08b8d29c + its balance.bin file offset`, 343 of
343 with no exception.** Those two numbers were recovered from *different
binaries by different means* -- the offsets from `Sacred.exe` by
`tools/binary/balance_keymap.py`, the globals from the Linux parser's
disassembly -- and neither knew about the other. `balance.bin` is loaded
verbatim as one struct at `0x08b8d29c`, and reading a balance field at runtime
is a single absolute load.

That is what makes the rest of this section reachable: name a tunable, get its
address, and the functions that read it are the formulas that use it.

## The derived-stat kernel

Every attribute-to-combat-number conversion in `FUN_0820ef48` goes through one
expression. Written as it decompiles, with `0.0064102565` being `1/156`:

```
K(S) = (156 - BalStatOff)*S/156 + BalStatOff + 9
```

A straight line in the attribute `S`, pinned so that `K(0) = BalStatOff + 9`
and `K(156) = 165` whatever `BalStatOff` is. Retail ships `BalStatOff = 20`,
making it `K(S) = 0.8718*S + 29`.

Damage accumulates per channel, each channel dividing by its own tunable:

```
damage_channel += ( K(attr) * weapon_term * 0.1 + flat_term ) / BalChar<C>D
```

Retail ships `BalCharPD = 220`, `BalCharMD = 220`, `BalCharFD = 440`,
`BalCharGD = 440`. The doubled divisors go with doubled numerators -- the `FD`
and `GD` channels sum *two* weapon terms and count their flat terms twice -- so
those two are a mean of both contributions rather than a halving.

Resistances accumulate the same way, without the kernel:

```
res_ph += (a + b) / BalanceResPh      ; 22.0 in retail
res_fe += c       / BalanceResFe      ; 17.0
res_ma += d       / BalanceResMa      ; 17.0
res_gi += (a + e) / BalanceResGi      ; 25.0
```

The `Ph/Fe/Ma/Gi` suffixes are the German *physisch / Feuer / Magie / Gift*, so
the four channels are physical, fire, magic and poison. That is read off the
key names, not assigned.

`FUN_0820ef48` accumulates into eight damage floats at `+0xa6 ... +0xc2` of its
first argument and four resistance floats at `+0x66 ... +0x72`, and is called
from six sites clustered around `FUN_081f5014`, in the same neighbourhood as
`FUN_081f686a`, which reads 117 balance keys and is the creature stat builder.

## The level curve, and the two numbers that tune it

`FUN_081f64ac` is four lines long and is the single curve behind every
level-scaled stat in the game. `FUN_081f686a` -- reached both from the
derived-stat pass and from `cUI_StatisticsChar::vf06`, so it is what the
character sheet displays -- calls it 41 times, once per stat family, and does
nothing else of substance.

As decompiled, then reduced:

```c
longdouble stat_curve(float off, float L, float s, float w)
{
  if (L < 1.0) return 0;
  longdouble r = (w - off) * (1 - 1/((1/s)*(L - 1) + 1));
  return r + r + off;
}
```

```
stat(L) = off + 2*(w - off)*(L - 1) / ( (L - 1) + s )
```

The two forms agree to 1.6e-12 relative over 20000 random inputs, which is
float noise. It is a saturating hyperbola, and the three tunables are its
geometry:

| | |
|---|---|
| `stat(1)` | `off` -- the value at level 1 |
| `stat(1 + s)` | `w` exactly -- `s` is the level span to the midpoint |
| `stat(inf)` | `2w - off` -- the asymptote, never reached |

**Every one of the 41 complete `off`/`_s`/`_w` families in retail ships
`_s = 50.0`, with no exception**, while `_w` ranges from 10 to 375. So `_s` is
not really a per-stat tunable at all: the whole game shares one curve shape,
pinned at level 51, and a stat is tuned by moving only its two endpoints. That
also settles the German: `_s` is *Stufe*, the level, and `_w` is *Wert*, the
value there.

Worked example, with retail's numbers:

| family | `off` | `_s` | `_w` | L=1 | L=51 | L=50 | asymptote |
|---|---|---|---|---|---|---|---|
| `HP…VW` | 4 | 50 | 130 | 4.00 | 130.00 | 128.73 | 256 |
| `AK…AW` | 9 | 50 | 150 | 9.00 | 150.00 | 148.58 | 291 |
| `BK…AW` | 12 | 50 | 200 | 12.00 | 200.00 | 198.10 | 388 |
| `R…Ph` | 12 | 50 | 250 | 12.00 | 250.00 | 247.60 | 488 |

### What the `SP` families are

The suffix families divide by what they modify. `AW` is *Angriffswert* and is
read straight. `SP` is applied by `FUN_081f64e8` as a **percentage against a
duration**:

```c
pct = stat_curve(bal_XoffSP, level, bal_X_s_SP, bal_X_w_SP);
record[+0x10] = (1.0 / (pct*0.01 + 1.0)) * base[+0x10];   /* floor 1.0 */
```

and formatted for display as `"%s %s %+d%%"`. Dividing a duration by
`1 + pct/100` shortens it, so the `SP` families are a **speed** modifier, with
a hard floor of `1.0` and a running minimum tracked at `+0x12`. That reading is
off the code, not off the name.

## Open

- **The resolution step is still undecoded.** This section is how a creature's
  damage and resistance *numbers* are built. What consumes them at the moment
  of a hit -- how damage is reduced by resistance, criticals, and the
  `param_3` flag that selects between the `+0x4a` and `+0x4e` weapon terms --
  is not read.
- **The struct offsets are offsets, not names.** Which attribute lives at
  `+0x56`, and which weapon slot at `+0x4a` against `+0x4e`, is not
  established. `FUN_081f686a` turned out to be the level curve rather than the
  field map, so this is still open; the character-sheet strings that
  `cUI_StatisticsChar::vf06` prints beside each value are the next lead.
- **The two- and three-letter family prefixes are unexpanded.** `AK`, `BK`,
  `FEK`, `FK`, `KK`, `HR`, `BA`, `BL`, `EM`, `FM`, `HM`, `HOM`, `KO`, `LM` name
  skills and attributes and are not yet matched to their German words.
- **The kernel is single-source.** Unlike to-hit it has been read only in
  retail Linux `sacred_orig`. The Armalion builds that confirmed to-hit are the
  obvious second arm and have not been checked.
- 28 of the 348 balance keys are parsed and then read by nothing.

---
Provenance: to-hit recovered by decompilation in this project's private
analysis workspace and confirmed in a second binary. The key-to-global map and
the derived-stat kernel from `sacred_orig` via `tools/binary/ghidra/`,
cross-checked against `balance_keymap.py`'s independently recovered file
offsets. See
[../method/naming-oracle.md](../method/naming-oracle.md) for the technique.

