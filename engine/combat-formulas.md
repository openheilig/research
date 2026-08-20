# Combat formulas

**Status:** To-hit, both ratings and the damage resolution recovered and
confirmed live on retail; resistances and elemental channels still unexercised
**Purpose:** The combat arithmetic: to-hit end to end, the derived-stat kernel
that feeds damage and resistance, and the resolution step that consumes them.

Two things are recovered. **To-hit** is read end to end in two binaries —
`armalion.exe` and `armalion_us.exe`, **both 2001 prerelease**, so "two
binaries" is not "two generations", and **retail has never been checked**.
Its call site in the prerelease also drops one of the four inputs; see the
warning under *To-hit*. The **derived-stat pass** -- how a creature's damage
and resistance numbers are built from its attributes and the balance table --
is read off retail Linux and is confirmed only there.

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

> ⚠️ **This is the function's body, and Armalion never calls it that way**
> (rows 1032–1033, 2026-08-20). The decompilation above is correct —
> `sub_428790`, reached through thunk `sub_40286F` — but the **call site drops
> `AT`**. At `0x4221ef` the caller pushes
> `(event+0x18, this+0x134, event+0x18, var_20)` = `(ALVL, DLVL, ALVL, PA)`.
> The third argument repeats `ALVL`; `AT` lives at `event+0x1C` and is passed
> to the trace `printf` but **never to the formula**. What the prerelease
> actually computes is therefore
>
> ```
> hit% = clamp( 200·ALVL/(ALVL+DLVL) · ALVL/(ALVL+PA), 5, 95 )
> ```
>
> Measured against a controlled sweep of the hero's level in the running game
> — levels 2, 3, 5, 8, 12, 20, both attack directions, 214 samples in 12
> distinct pairings — the call-site model is **exact on 12 of 12**. The formula
> as written above matches only the 4 that sit on a clamp boundary.
>
> **This is prerelease behaviour, not Sacred's.** The "confirmed in two
> binaries" above means `armalion.exe` and `armalion_us.exe`, both 2001. See
> *Retail's own shape* below — it is a different formula.

## Retail's own shape — one ratio, no level term

Located structurally in the Linux LGP `sacred` (row 1034), because retail
strips the combat trace strings and contains no `cmp …, 95` anywhere.

The anchors are the two rating getters — `sub_81FA5AA` reads attack at
`creature+0xE6`, `sub_81FA622` reads defence at `+0xEA` — and exactly four
functions call **both**:

| function | size | what it is |
|---|---|---|
| `sub_80DC90C` | 0x1bbf | character-sheet **text** builder (`std::allocator<wchar_t>` throughout) |
| `sub_816C244` | 0x14287 | the **attack construction**: reads AT/PA, applies buff/curse multipliers `flt_8B89CF0` / `flt_8B89CE8` / `flt_8B89CF4` gated on `+0x1F6` bit 8 and `+0xB1` bit 4, then assembles the four damage channels (`+0xD6/+0xDA/+0xDE/+0xE2` × `+0x66/+0x6A/+0x6E/+0x72`) |
| `sub_83A37CA` | 0x106 | computes a **pair of hit percentages** |
| `sub_854AF5E` | 0x32a9 | UI caller of the above |

`sub_83A37CA` computes both directions and clamps each at 100:

```c
*a3 = 100 * curve(AT_self,      other[+30]);   if (*a3 > 100) *a3 = 100;
*a4 = 100 * curve(other[+28],   PA_self);      if (*a4 > 100) *a4 = 100;
```

and the curve `sub_815D44C` is **parameterised**:

```
k   = -ln(1 - a5) / ln(a4 + 1)
out = 1 - 1 / ((a2/a3 + 1)^k)          (returned in *a6; the return value is a2*out)
```

`sub_83A37CA` passes `a4 = 1.0, a5 = 0.5`, which makes `k = 1` exactly, and
then

```
hit% = 100 · AT/(AT + PA)
```

**One ratio, clamped at 100, and no level term at all** — structurally unlike
the prerelease's `clamp(200·…·…, 5, 95)`, and consistent with retail having no
`cmp …, 95`.

### Confirmed live, in combat (row 1035)

Loaded from a save standing next to a hostile and watched under
`tools/live/tohit_bp.sh`. 55 breakpoint hits, **every one of them on the
curve**, and `a4 = 1.0, a5 = 0.5` on every single call — so `k = 1` always and
the curve is always the plain ratio.

The roll sits immediately after the combat call site at `0x81fc6fc`:

```asm
call  815d44c                  ; the curve; its fraction lands in [ebp-0x1f0]
fstp  st(0)                    ; discard the return value (a2 * out)
mov   ebx, 0
call  rand@plt
...                            ; edx = rand % (1000 - 0 + 1) + 0   -> 0..1000
fild  [esp]
fmul  ds:0x86e6ba8             ; = 0.001 exactly, so roll is 0.000..1.000
fcomp [ebp-0x1f0]              ; roll against the chance
fnstsw ax
test  ah, 45h
jne   81fc99d
```

So retail resolves a swing as

```
chance = AT / (AT + PA)
roll   = (rand() % 1001) * 0.001          uniform 0.000 .. 1.000
```

A live sample from that site, 26 identical swings: `a2 = 19.5, a3 = 26.4`
→ **42.5%**.

**Which branch is the hit — settled (row 1037).** `fcomp` sets `C0` when
`st(0) < operand`, so the `jne` is taken on **`roll < chance`**, and its target
`0x81fc99d` is where the damage step continues. Breakpointing the comparison,
the jump target and the fall-through together resolves it without inference:

| roll | branch | reached `0x81fc99d` |
|---|---|---|
| 0.142, 0.345, 0.322, 0.366, 0.320, 0.396, 0.261, 0.074, 0.024, 0.072, 0.055 | taken | **yes**, 11 of 11 |
| 0.939, 0.785, 0.516, 0.531, 0.554, 0.446, 0.676 | not taken | **no**, 7 of 7 |

with `chance = 0.424837` throughout, which is `19.5/(19.5+26.4)` to six
places. So `roll < chance` **hits** and the comparison is **strict**; a miss
leaves through the fall-through, which builds a `std::string` and never
reaches the damage step.

One thing on that fall-through is worth recording so it is not re-investigated:
between it and `0x81fc99d` sits a block at `0x81fc945`–`0x81fc99d` that scales
all four damages by `(x + 10·c) · 0.01` for the difficulty constant `c`. It
looks like it belongs to the miss path and does not — it was reached **zero**
times in this session, so it has another predecessor. Do not read the roll as
gating a damage bonus.

**Why the getters never fire.** `sub_81FA5AA` and `sub_81FA622` were not hit
once during combat. The combat path **inlines** the same `base × skill-product`
multiply — `fld [ebp-0x248]; fmul [ebp-0x48]` right before the push — instead
of calling them. The getters serve the character sheet; the fight computes its
own.

**And the resolution step has an address now.** The other live curve sites are
`0x81fd0d7`, `0x81fd127`, `0x81fd177`, `0x81fd1c7` — four blocks spaced 0x50
apart inside `sub_81FAC30`, which is the **four damage channels** running
through the same ratio curve. That step is now decoded and confirmed live; see
*Damage resolution* below.

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

## Damage resolution — the same curve, four times (row 1036)

`sub_81FAC30` resolves a landed swing into damage. It runs the **same curve**
as to-hit, `0x815D44C`, once per channel, and the curve's full contract is:

```c
float curve(void *self, float a2, float a3, float a4, float a5, float *a6)
{
    a5 = clamp(a5, 0.0f, 0.99f);          // a5 >= 1.0 becomes 0.99
    a4 = max(a4, -1.0f);
    float k = -logf(1.0f - a5) / logf(a4 + 1.0f);
    float r = fabsf(a3) > 0.001f ? a2 / a3 : 10000.0f;
    float f = 1.0f - 1.0f / powf(r + 1.0f, k);
    if (a6) *a6 = f;                      // the fraction, as an out-param
    return a2 * f;                        // the fraction applied to a2
}
```

**One function, two consumers, and that is why to-hit discarded the return.**
To-hit passes `&[ebp-0x1f0]` as `a6` and rolls against the **fraction** `f`;
damage passes `a6 = NULL` and keeps the **return** `a2 * f`. The `fstp st(0)`
at `0x81fc701` is not a thrown-away result — the result it wanted came back
through the pointer.

All five call sites pass `a4 = 1.0` (three push the literal `0x3f800000`; the
first synthesises it from the `fld1` left on the FPU stack, which reads like a
leak and is not one). So `ln(a4+1) = ln 2` throughout, and

```
k = -log2(1 - a5)
```

Per channel `i` of four:

```
dmg[i] = a2[i] * (1 - 1 / (a2[i]/a3[i] + 1)^k) * (100 - resist[i]) / 100
```

`a2[i]` is raw damage, `a3[i]` is the target's armour in that channel. The
`0.01` is the literal at `0x86e6b9c`; `100` is `0x86e6b98`.

### The level term — it lives here, not in to-hit

`a5 = 0.5 * [ebp-0x2c8]`, and `[ebp-0x2c8]` defaults to `1.0` (stored as an
immediate at `0x81fcef2`). It is raised only at `0x81fcf0f`, when the attacker
`cCreature` passes a type gate (`[atk+0x0c] > 0x10`) **and** outranks the
target:

```
Δ         = max(0, attackerLevel - targetLevel)     // u16, one-sided
[ebp-2c8] = 1 + 0.01 * Δ
a5        = 0.5 * (1 + 0.01 * Δ)
k         = 1 - log2(1 - Δ/100)
```

Δ = 0 gives `k = 1`, which collapses the curve to `a2²/(a2+a3)`. Δ = 50 gives
`k = 2`. The `a5 >= 1.0 → 0.99` clamp inside the curve means the bonus
**saturates at Δ = 98** (`k = 6.64`) instead of going undefined — so the
obvious `ln(0)` bug is guarded, deliberately.

So **level asymmetry is a damage effect, not an accuracy effect.** To-hit has
no level term (see above) and this is where the level difference actually went.

### Difficulty and resistance

`ds:0x8906cd0` holds the difficulty, `0..4`, and does two things.

It is scaled into `esi` and used to index the target's **resistance table**:
the resist byte for channel `i` is read at `[info + esi + 0x42 + 5*i]`, so the
table is **4 channels × 5 difficulties of `uint8` percent**, occupying
`+0x42 .. +0x55` of the creature-info record. The record stride is **0x56**,
measured live off the record array, so the table is exactly the record's last
20 bytes — the arithmetic closes.

Separately, when `[atk+0x4eb] & 8`, channel 0's armour is scaled by
`max(0, 1 - 0.4*c)` with `c` selected by difficulty — `1.0, 1.34, 1.67, 2.0,
2.34` at `0x86e6b50/b70/b74/b54/b78`, giving multipliers `0.6, 0.464, 0.332,
0.2, 0.064`. Armour matters less as difficulty rises.

Two smaller adjustments precede the channels: a modifier search for id `0x38`
in the attacker's list at `[atk+0x4ba]` scales channel 0's armour by
`100/[mod+0x14]`, and a target of type `0x47` bypasses the whole step, its
four raw damages copied straight out at `0x81fcf67`.

### `a1` is not the creature — it is the creature's combat block

`a1` and the attacker use different level offsets (`a1+0x56` against
`atk+0x3fe`), which looks like two structs until the live pointers land:
`0xafd9a68 - 0xafd96c0 = 0x3a8`, the same `add edx,0x3a8` the code performs,
and `0x3a8 + 0x56 = 0x3fe`. **`a1 = &creature[0x3A8]`**, and the two level
reads are one field. The attacker is recovered by
`__dynamic_cast(..., "7cObject", "9cCreature")` at `0x81fae61` and is NULL for
a non-creature source.

### Confirmed live

Breakpoints at `0x81FD08D` (all inputs) and `0x81FD1F5` (all four results),
against the staged combat save — `tools/live/bp-damage.gdb`.

**64 of 64 channels exact**, over 16 swings and four distinct attacker/target
pairings in both directions. The level term fired on its own: a level-2
attacker against a level-1 target printed `lvlterm = 1.010000` and `k =
1.0145`, and the same pair reversed printed `1.0` — the one-sided gate,
live. Forcing `[ebp-0x2c8]` to `1.5` (`k = 2`) predicted 20.9921 and 23.0310
against observed 20.992071 and 23.030998.

> ⚠️ **Not exercised: resistances, elemental channels, difficulty.** Every
> sample was a level-1 hero with a plain weapon at difficulty 0, so only
> channel 0 ever carried damage, every resist byte read 0, and the
> `|a3| <= 0.001 → r = 10000` guard never ran. The channel formula is
> confirmed; the resist-table layout is **structural** — from the indexing and
> the 0x56 stride — and still wants a resistant target to close.

## Where AT and PA come from — skills, not attributes

**No base attribute becomes either rating.** Both accumulate from **skill
levels**.

`sub_81F596E`, the creature stat builder, dispatches on skill *type* through
the jump table at `0x81F5A94`. For each skill a creature carries it feeds that
skill's **level** — the `u16` at `+2` of the skill entry, the type being at
`+0` — into one shared curve, twice, once per `balance.bin` triplet belonging
to the skill's family.

`sub_81F55B0`, verbatim:

```c
float f(float off, float S, float s, float w)
{
  if (S < 1.0) return 0.0;
  float v = (1.0 - 1.0/((S - 1.0)/s + 1.0)) * (w - off);
  return off + v + v;
}
```

A saturating function of the skill level: `f(1) = off`, `f(1+s) = off+(w−off)`,
and a ceiling of `off + 2(w−off)` it approaches without reaching. **An
untrained skill contributes zero, not `off`** — a floor, not a clamp.

Case 8 is *Agility* (family `W`) and computes **AW** from
`WoffAW/W__sAW/W__wAW` and **VW** from `WoffVW/W__sVW/W__wVW` off the same
level. Case 3 is *Long-handled Weapons* (family `STK`) and computes AW and then
**SP** — so the second output's meaning is per-family, not fixed.

### Which skills

| | families | skills |
|---|---|---|
| **AW** (attack) | STK, SK, AK, KK, FK, BK, FEK, W, HR, *WT* | Long-handled Weapons, Sword Lore, Axe Lore, Blade Combat, Unarmed Combat, Dual Wielding, Ranged Combat, Agility, Constitution (`WT` maps to no skill) |
| **VW** (defence) | **W, HP** | **Agility and Constitution — and nothing else** |

Retail's `W` triplets: AW `7 / 50 / 125`, VW `13 / 50 / 225`.
`HP` VW: `4 / 50 / 130`. `VWFakBoss` = 2.0 and `VWFakChamp` = 1.5 multiply the
defence rating; what marks a creature boss or champion is not recovered.

### Where the level comes from — a per-sector band

The curve consumes a skill *level*, and `creature.pak` carries skill **types**
only. The level comes from the **spawn**, not the creature table.

Opcode 100 `SpawnValues` carries `(50, lo, hi)`. The first is 50 in all 11,498
records; the other two are an ordered band — `lo ≤ hi` in **11,498 of 11,498**,
against **0.00%** for the same test on the first pair.

It is **geographic**, which is what makes it a level band rather than a weight.
Each `Sector<cx><cyyy>Init`/`Enter` procedure owns a funkcode span, and the
`SpawnValues` inside it belong to that sector:

| sector | | band |
|---|---|---|
| 50,39 | Seraphim start | **(1, 4)** |
| 51,39 · 53,42 · 54,43 · 59,5 | magician, elves, dwarf, gladiator starts | **(1, 4)** |
| 5,26 | Daemoness start | (45, 80) |
| 97,60 | Underworld start | (30, 50) |

**Seven of nine class starts land on the game's lowest band.** The Daemoness is
the counter-example that keeps it honest — the claim is not "every start is
low" but "each start sits where its class begins" — and only 22% of banded
sectors are `(1,4)`, so five hits is not chance.

5666 sectors carry a band; **60 declare more than one** and nothing says what
chooses between them.

> **Still open:** how a level is *drawn* from the band, and whether that number
> is also the level at which the creature's skills are known. Both are needed
> before the ratings above can be computed.

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
key names, not assigned — and the engine says the same thing independently:
`global.res` resources 1078-1081, the four rows the character sheet prints
under *Damage*, are `Physical, Fire, Magic, Poison` in that order.

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

### The family prefixes

`AK`, `BK`, `FEK`, `FK`, `KK`, `STK`, `SK`, `W`, `P`, `R`, `HR`, `HP`, `WK`,
`MK`, `Med`, `KO` and the magic families are German abbreviations that the
engine never spells out. They are resolved by an identity the binary states
about itself, so **none of it is a guess from the German**:

- A creature holds 8 skill slots at `+0x24`, one byte each; that byte is the
  **skill type**, returned by `FUN_0821b576(creature, slot)`.
- `cUI_StatisticsChar::vf06` turns that same byte into the skill's name with a
  literal `add $0x24b7, %eax` at `0x0854e99b` — **resource id = skill type +
  9399** — and passes the byte *unchanged* to `FUN_081f686a`.
- So `FUN_081f686a`'s jump table at `0x086ed568` is indexed **by skill type**.
  Case *i* reads exactly the balance globals belonging to skill `9399 + i`.

The table is a **permutation**, not the identity — index 1 jumps to the
*tenth* case body by address. Reading the cases in address order therefore
produces a complete, self-consistent and entirely wrong answer, which is what
happened on the first attempt.

The check is not an assertion but a coincidence that cannot survive being
wrong: **thirteen** case bodies push their own skill's resource id as a
literal, and every one equals `9399 +` that case's table index. 13 of 13,
zero mismatches. Shifting the base by one, or the table pointer by one entry,
breaks 12–13 of them.

| abbrev | German | skill | id |
|---|---|---|---|
| `HM` | Himmelsmagie | Heavenly Magic | 9400 |
| `WK` | Waffenkunde | Weapon Lore | 9401 |
| `STK` | Stangenwaffenkunde | Long-handled Weapons | 9402 |
| `SK` | Schwertkunde | Sword Lore | 9403 |
| `AK` | Axtkunde | Axe Lore | 9404 |
| `BK` | **Beidhändiger Kampf** | Dual Wielding | 9405 |
| `FEK` | Fernkampf | Ranged Combat | 9406 |
| `W` | **Wendigkeit** | Agility | 9407 |
| `P` | Parieren | Parrying | 9408 |
| `HR`, `HP` | — | Constitution | 9409 |
| `R` | **Rüstung** | Armor | 9410 |
| `Med` | Meditation | Meditation | 9411 |
| `KK` | Klingenkampf | Blade Combat | 9412 |
| `MK` | Magiekunde | Magic Lore | 9413 |
| `FM`/`WM`/`EM`/`LM`/`MM` | Feuer/Wasser/Erd/Luft/Mond | the five magic schools | 9414-9418 |
| `Reiten` | Reiten | Riding | 9421 |
| `FK` | **Faustkampf** | Unarmed Combat | 9423 |
| `KO` | **Konzentration** | Concentration | 9424, and shared with 9419, 9426 |
| `BA` | Ballistik | Ballistics | 9425 |
| `BL` | Blutdurst | Bloodlust | 9427 |

**Only the last two columns are read from the binary.** The German column is
inference — the expansion that makes the abbreviation fit the skill the engine
assigned it. It is a reading aid and nothing downstream should depend on it;
the *identity* being claimed here is prefix → skill, not prefix → German word.

Four of these are exactly the trap. `FK` reads as *Fernkampf* and is
*Faustkampf*; `BK` reads as *Bogenkampf* and is *Beidhändiger Kampf*; `W`
reads as *Waffe* and is *Wendigkeit*; `R` reads as *Resistenz* and is
*Rüstung*. Guessing from the German would have got **4 of 13 wrong** while
looking entirely reasonable — and three of the four wrong answers would have
put a weapon family on the wrong weapon.

`KO` is the one prefix that is genuinely shared: its triple serves
Concentration, Vampirism and Trap Lore, so an earlier reading of "KO =
Vampirism" from a single call site was wrong twice over — wrong that it was
one skill, and wrong about which.

Skill type **30** (`9429` Two-handed Weapons) jumps to the default: it has no
case and no balance family, and is the only skill in the list that does not.
`9420` Trading, `9422` Disarming and `9432` Forge Lore have cases but touch no
balance global.

Regenerated and re-checked by
[`tools/binary/skillmap.py`](../../tools/binary/skillmap.py) into
[generated/skill-families.tsv](../formats/generated/skill-families.tsv).

## The creature's level is a CLAMP on the hero's, not a draw

`sub_81806DC`, the only consumer of the per-sector band (row 953), called once
from the creature spawn path `sub_8180B22`:

```c
level = hero_level;                       /* party: the highest */
if (marker[211] && marker[213]) {
    lo = DiffLo[difficulty] + marker[211];
    hi = DiffHi[difficulty] + marker[213];
    if      (level <  lo) level = lo;
    else if (level <= hi) level = level + rand() % 2;
    else                  level = hi;
}
```

So the band **clamps the player's level** — it is not a range to sample, and
the only randomness in the whole rule is `+0` or `+1`. Sacred's world is
level-scaled by construction. A uniform draw across `[lo, hi]` produces levels
inside the same interval and so passes any range check while getting the
difficulty curve wrong everywhere, which is why `spawnlevel_check.gd` asserts
each branch separately.

`DiffLo`/`DiffHi` are at `0x8B89BA8` and `0x8B89DAC` in the executable, not in
any shipped file; their values are unrecovered, so `level_for` takes them as
arguments rather than assuming a difficulty.

## Where AT and PA actually live

`cCreatureHero::CalcResults` (`sub_820E04C`) initialises **attack at `+0xE6`
and defence at `+0xEA` to `1.0f`** and calls `sub_81F596E(creature, 0, 0, 0)`.

`sub_81F596E` is a **skill-effect applier, not a level-taking stat builder**:
it loops the creature's eight skill slots (ids at `+0x24`, base `+0x2C`, bonus
`+0x34`), switches on the skill type, and multiplies each through the curve at
`sub_81F55B0`. It is passed **no level at all**.

A rating is therefore `base x product-of-skill-curves`, and it is the **base**
that is unrecovered.

**Correction: `+0x3C/+0x3E/+0x40` are not AT/PA.** `CalcResults` sets them to
140/120/<table>, or 100/120/100, or 140/130/120 by creature class, clamps them
to a floor of 80 or 100 and a ceiling of 220, and subtracts 15 from all three
under a curse flag. A triple of percentages near 100–140 is the three speed
ratings, not two combat ratings.

For the MVP this is sharper than half an answer: a new Seraphim knows Magic
Lore and Weapon Lore, families `MK` and `WK`, and **neither carries an `AW` or
a `VW` triplet**. At level 1 no skill she has feeds either rating, so both are
entirely base.

## Open


- ~~**The resolution step is still undecoded.**~~ **Closed** — it is
  `sub_81FAC30`, four channels through the to-hit curve; see *Damage
  resolution*. Two parts of it remain unexercised and are called out in the
  warning there: **no sample had a nonzero resistance byte or a nonzero
  elemental channel**, so the resist term and the `|a3| <= 0.001` guard are
  structural only. Wanted: a target with resistances, and an elemental damage
  source. What still is not read is which flag selects between the `+0x4a` and
  `+0x4e` weapon terms, and where criticals enter — neither appears in
  `sub_81FAC30`, which is consistent with the Armalion source tree showing
  combat as a **state machine split across two files**, `state_Attacking` in
  `creature_fighting.cpp` and `state_Fighting` in `creature_collision.cpp`.
  See [../builds/armalion-source-tree.md](../builds/armalion-source-tree.md).
- **The struct offsets are offsets, not names.** Which attribute lives at
  `+0x56`, and which weapon slot at `+0x4a` against `+0x4e`, is not
  established. `FUN_081f686a` turned out to be the level curve rather than the
  field map, so this is still open; the character-sheet strings that
  `cUI_StatisticsChar::vf06` prints beside each value are the next lead.
- ~~**The two- and three-letter family prefixes are unexpanded.**~~ **Closed.**
  See *The family prefixes* below.
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

