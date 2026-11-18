# The decompilation corpus — availability and identity admission

**Status:** Standing
**Purpose:** The machine-readable RE layer under `analysis/decomp/` (private,
never published): per-function exports and manifests, identity joins and a
cross-binary class matrix. This document says what exists, how it was made,
and how to regenerate it. Companion: [decompilation-coverage.md](decompilation-coverage.md)
owns the *naming* question; this one owns the *corpus*.

Historical2026-08-29 claim: ~~every function of every build decompiled:
74,592 functions,0 failures,0 timeouts~~. **Superseded2026-10-12:** the exporter
counted null Hex-Rays returns as successful `"None"` text. All74,592 manifest
records join their headers, but1,044 bodies are null,38 nonempty bodies carry
explicit decompiler warnings, and the current admission gate correctly fails.
Only73,548 records contain nonempty text;73,510 have no measured warning marker.
Neither count establishes semantic correctness or complete mapping.

The new `tools/binary/decompile_audit.py` reports private binary identity,
manifest/header/summary joins, availability, warnings and a targeted repair
queue. `decompile_dump.py` now rejects null/empty returns and records truthful
statuses/exit codes. Original corpora are preserved, not rewritten as green.

## The builds

| label | binary | role | funcs | historical header-ok | vftables (slots) | structurally best-named |
|---|---|---|---|---|---|---|
| `linux1002` | staged LGP 1.0.02 patched copy | current live oracle has identical `.text`, different VRAM-cache data; see caveat | 16,004 | 16,004 | 2,413 vmethods via `extract_vtables_elf.py` | 3,794 (23.7%) |
| `win228eng` | Gold 2.28 ENG (`builds/windows-disc/eng/extracted`) | shipping-truth pixel oracle | 9,642 | 9,642 | 328 (1,988) | 2,246 (23.3%) |
| `demo-usa` | 2004 USA demo (`analysis/builds/demo-usa`) | structure oracle (202 vftables, §10) | 8,354 | 8,354 | 200 (1,341) | 1,889 (22.6%) |
| `armalion-sep01` | 2001-09-20 `armalion.exe` | the narrating debug build | 7,388 | 7,388 | 127 (736) | 786 (10.6%) |
| `armalion-us01` | 2001-09-20 `armalion_us.exe` | rules oracle (combat maths named in debug strings) | 7,401 | 7,401 | 127 (736) | 787 (10.6%) |
| `armalion-dec01` | 2001-12-11 `armalion.exe` | source-layout oracle (`_assert` paths) | 8,514 | 8,514 | 145 (935) | 957 (11.2%) |
| `gold228-rus` | Gold 2.28 RUS (`analysis/builds/gold228-rus`) | localization rebuild of the same source tree | 9,636 | 9,636 | 328 (2,072) | 2,277 (23.6%) |
| `sacredserver` | staged LGP server | private server reference; multiplayer outside current release | 7,653 | 7,653 | not audited here | 896 |

`win228rus` is an old alias of the actual `gold228-rus` corpus, not a ninth
dump. The former table duplicated it and omitted sacredserver. Naming above
includes imports/RTTI-slot identities; it is not live method-behaviour coverage.

| build | nonempty text | null text | null size >6 | warning text |
|---|---:|---:|---:|---:|
| linux1002 | 15,453 | 551 | 0 | 5 |
| win228eng | 9,621 | 21 | 21 | 12 |
| gold228-rus | 9,615 | 21 | 20 | 15 |
| demo-usa | 8,335 | 19 | 11 | 5 |
| armalion-sep01 | 7,384 | 4 | 0 | 0 |
| armalion-us01 | 7,395 | 6 | 1 | 0 |
| armalion-dec01 | 8,509 | 5 | 0 | 0 |
| sacredserver | 7,236 | 417 | 1 | 1 |

990 nulls are at most six bytes. Actual import-table/relocation joins now prove
989 external imports or import-bound jumps; RUS00640020 is a truncated SEH
prologue, not a thunk. None is counted as successfully decompiled pseudocode.
The prior supplemental ledger labels21 original nulls as recovered private
text: seven each ENG/RUS, six demo and one server. **Workflow2026-10-16
correction:** three retained demo text values actually end in explicit truncation
markers (0054A690/0054D220/00611640). Do not treat their `body_available` label
as a complete function. A later approved Linux08113DEE repair adds a22nd
supplement; the new navigation index classifies19 as unwarned and3 truncated.
Original exports/ledgers remain unchanged. These labels are not semantics.
All original manifest/header joins still pass. Legacy summaries remain
hash-unbound; current input hashes do not establish historical export provenance.
The 11 patch-series builds are **not** dumped: UPX-packed with a tampered
packer and closed as provenance-only (RESEARCH §16).

## What is on disk

`analysis/decomp/<label>/` (422 MB total; stays in `donotpublish` — Hex-Rays
output is decompiler output, § Publishing):

| file | holds |
|---|---|
| `functions.tsv` | every function: addr, end, size, IDA name, demangled, status, chunk |
| `chunks/NNNN.c` | the pseudocode, 200 functions per file, each introduced by `// ==== addr=... name=... status=...` |
| `summary.json` | totals for the run |
| `vftables.tsv` | MSVC builds: `class, slot, target_addr, target_name` from IDA's RTTI parser |
| `vmethod-names.tsv` | vftables normalized to `addr → class::vfNN` for the join |
| `assert-names.tsv` | linux1002 only: the 130 addresses named by the wider assert-string scan (`tools/binary/assert_names.py`) |
| `doc-roles.tsv` | linux1002 only: 251 (address, role) pairs machine-extracted from all 31 research docs, 159 distinct addresses |
| `names.tsv` | the join: addr, end, size, ida_name, best_name, name_sources, roles |

## Identity layers, joined

`tools/binary/identity_join.py` merges, per function: assert-string names
(130, linux only), RTTI vtable methods (2,413 linux / 1,988–2,072 MSVC
slots), IDA's own demangled names, and doc-derived roles. `best_name`
priority: assert > RTTI > IDA; every source that fired is kept in
`name_sources`, nothing silently dropped.

Two denominators, both correct, do not mix: **16,004** is IDA functions for
`linux1002` (thunks, chunks and handler clones included); **13,716** in
[decompilation-coverage.md](decompilation-coverage.md) is distinct call
targets. The 17.6%-named figure there counts that census's two oracles; the
23.7% here is the union over four layers including IDA's own names — a
superset, not a contradiction. The honest statement is unchanged: the
majority of the engine has no recovered identity, and **no name has been
confirmed by behaviour** (see the coverage document's Open section).

## Cross-binary class matrix

`analysis/rtti/class-sets/class-matrix.tsv` — 439 distinct class names over
six builds, one column each:

| set | classes | linux1002 ∩ | win228eng ∩ |
|---|---|---|---|
| linux1002 (Itanium, `lgp_linux_vtables.json` — 383 primary, 386 with MI secondaries) | 386 | — | 323 |
| win228eng (MSVC `.?AV` scan) | 325 | 323 | — |
| gold228-rus | 325 | — | 325 (identical sets) |
| demo-usa | 268 | 264 | 265 |
| armalion-sep01 | 126 | 85 | 85 |
| armalion-dec01 | 144 | — | — |

ENG and RUS carry byte-identical class sets — the localization-rebuild
verdict (RESEARCH §15.1), now at class granularity. Linux ∩ 2.28 = 323 of
325: the two independent compilers built the same class graph.

## Regenerating

```
venv=~/.claude/plugins/cache/mrexodia/ida-pro-mcp/0.1.0/.venv/bin/python
$venv analysis/tools/binary/decompile_dump.py <binary> <outdir>        # 40 s … 7.7 min
$venv analysis/tools/binary/extract_vftables_idalib.py <binary> <outdir>/vftables.tsv
/usr/bin/python3 analysis/tools/binary/assert_names.py install/sacred > <outdir>/assert-names.tsv
/usr/bin/python3 analysis/tools/binary/identity_join.py <outdir>/functions.tsv <outdir>/names.tsv \
    --name assert=... --name rtti=... --role docs=...
```

Binaries are staged copies inside each `analysis/decomp/<label>/` directory
(copied out of `install/`, `builds/`, `analysis/builds/` — nothing in those
trees is written). Dumps close the IDA database **without saving**, so the
pseudocode and TSVs are the artifacts; a re-run is cheap and reproducible.

## Caveats

- `linux1002` was dumped from the then-live patched binary, now retained staged
  with SHA25655ede8a88103672d48ecc28c440f59c92ec4365a77b0399ca95a67bd8681c995.
  Current`install/sacred` has SHA2568784ae11f876d1c7aeb2f9afeb4d721c8c6ac0c1788ac71b5d817095812528a4.
  Same length; only two bytes differ in VRAM-cache dword08835E80:
  staged262144/current256. Entire `.text` is unchanged. Retained IDB bytes at
  that address agree with staged, while survey metadata follows the current
  input file. Neither original nor live binary was altered by qualification.
- Research docs mix two Linux binaries' addressing conventions
  (`sacred_orig` vs `install/sacred`; `.rodata` `0x6ca0`, `.text`
  non-uniform — tools/binary/README.md). The dump, vftables and
  assert-names layers are exact for `install/sacred`; **`doc-roles.tsv` is
  mixed-provenance** — a joined role is a lead unless its source doc states
  `install/sacred`-space provenance (`game-wiring.md` and the Phase-07
  findings do).
- vftable slots occasionally point at adjustor thunks; the slot number is the
  class-layout truth, and the target resolves through `functions.tsv`.
- The MSVC builds have no assert-string oracle (retail release builds carry
  no debug strings); their `names.tsv` is RTTI-slot names plus IDA's own
  names only, and no Armalion-catalogue join has been applied to them.

## Open
- Required owner repair/admission precedes source-dependent feature work;
  this is not a mandate to repair every global gap before a bounded task.
  Original1082 repair records and their historical classification are retained.
  The current private reference workspace records original availability,
  supplemental truncation and build identity separately; see the engine
  workflow guide. Source correctness remains unqualified and originals unchanged.
- Gold dispatch44A9D0/44AD80 recovers after independently proven no-argument
  singleton401660 replaces a five-argument type in private copies. Four sector
  accessors recover after independent callee/caller proof establishes variadic
  logging rather than a fixed two-argument prototype.
- Server80553B0 is one94482-byte FDE. Exact Hex-Rays error is`too big function`;
  increasing only that private worker's limit64→128KiB returns unwarned text
  in151.05s, then restores64KiB. Global settings/original databases unchanged.
- Function-boundary repair is stopped, not guessed: Gold19 local-code entries,
  demo5, ArmalionUS4BA080 and truncated RUS640020 remain unavailable. Eight
  other Gold bodies fail call analysis at recorded instructions. Warning
  consumers include movement/idle/retirement, split damage/render ownership
  and save-UI lifetime; library/network roles never waive their warnings.

- **Behavioural confirmation remains at zero** for every build — the corpus
  makes that arm cheap, it does not do it. Owned by
  [decompilation-coverage.md](decompilation-coverage.md).
- MSVC naming beyond RTTI slots: the two routes that worked on Linux (debug
  strings, `eh_frame`) do not exist in the Windows builds; a third route
  (cross-build structural transfer) is unexplored.

---
Provenance: this session's dumps (2026-08-29); tools `decompile_dump.py`,
`extract_vftables_idalib.py`, `identity_join.py` in `tools/binary/`;
per-build totals from each `summary.json`.
