# Combat formulas

**Status:** Partial
**Purpose:** One combat formula recovered end to end, as the worked example of what the
method yields. The rest of the combat system is not decoded.

One formula is recovered end to end and independently confirmed in two
binaries. It is offered as the worked example of what the method yields, not
as a complete combat model — the rest of the system is not decoded.

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

## Open

One formula of a combat system. Damage, resistances, criticals, and every
other resolution step are undecoded -- this document is the worked example of
the method, not a model of combat.

---
Provenance: recovered by decompilation in this project's private analysis
workspace and confirmed in a second binary. See
[../method/naming-oracle.md](../method/naming-oracle.md) for the technique.

