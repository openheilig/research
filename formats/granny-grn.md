# Granny 1.x `.GRN` — models, skeletons, animation

**Status:** Read
**Purpose:** How a `.GRN` is laid out -- meshes, skeletons, per-bone animation
-- and which parts of it the engine consumes.

3413 of 3421 animation clips decode; meshes, skeletons and per-bone transform
tracks all resolve, and skeletal animation retargets across characters.

The entries live inside `models.pak` (4993 of them). Two independent walkers
exist and agree — `tools/formats/grn_tagwalk.py` in Python and the reader in
`engine/formats/models.gd` — which is what makes the decode trustworthy. They were
deliberately written from a specification and from direct byte reads, never
by translating one another; a transliterated second implementation would make
the parity diff structurally incapable of failing.

## Two entry kinds

| Kind | What | Root tag at |
|---|---|---|
| 64 | mesh | `0x4EA` |
| 65 | motion | `0x140` |

Chain root is `0xCA5E0000`; the walk ends at `0xCA5EFFFF`.

## Header chunks — measured sizes, tag inclusive

| Tag | Chunk | Size |
|---|---|---|
| `0xCA5E0000` | root | 32 |
| `0xCA5E0102` | copyright | 20 |
| `0xCA5E0103` | object | **20** |
| `0xCA5E0101` | final | **36** |

> The written specification this was built from said 16 for `object` and 32
> for `final`. Both are wrong on real data: using them desyncs the walk after
> the second header tag and it halts on an unrecognised tag four bytes early
> instead of reaching the terminator. 20 and 36 were measured directly on four
> independently sampled entries across both kinds — `BAT.GRN`,
> `GLADIATOR.GRN`, `GLAD_SA5_SHOULDER.GRN` (kind 64) and
> `BATX_ATTACK_BH_A.GRN` (kind 65) — identical shape and identical sizes
> regardless of kind. Only the starting magic offset differs.

Fixed 12-byte leaf chunks (4-byte tag + 8-byte payload) chain by a constant
`+12`: `0xCA5E0200`, and the ranges `0xCA5E1000`–`0xCA5E1003` and
`0xCA5E0F00`–`0xCA5E0F06`.

## The flat node directory

A structure separate from the header chain, reached at

```
section_offset(entry) = magic_offset(entry) + 376
```

then `+16` to a `u32` count followed by `{tag, rel, children}` triples of
stride 12.

`376` is not a new magic number — it reconciles the constants the engine
already shipped for kind 64 (`1634 − 1258 = 376`) and extends them to kind 65
(`320 + 376 = 696`). Verified corpus-wide across all 4993 entries, in both
kinds.

Node tags in use: `0xCA5E0506` bone, `0xCA5E1200` animation, `0xCA5E1203`
transform-track section, `0xCA5E1204` transform-track keys, `0xCA5E1205`
animation section.

## Per-bone animation record (`0xCA5E1204`)

A 52-byte header carrying three declared counts, then three flat time tracks,
then three flat payload arrays, then a real but never-decoded 48-byte trailer
that does not depend on any count.

| Offset | Field |
|---|---|
| 24 | `numTranslates` |
| 28 | `numQuaternions` |
| 32 | `numUnknowns` |

Verified zero-slack across every sampled record: the declared counts account
for the whole body exactly.

## A retracted conclusion worth keeping

> **`0xCA5E1204` is not "genuinely absent" from Sacred's motion data.** An
> earlier session concluded it was, having measured `models.pak` entry 2845 —
> which is `GLADIATOR.GRN` opened as a kind-65 entry: five meshes, 101 bones,
> and correctly *zero* per-bone animation records, because it is not a clip.
> There was never an absence to explain, only the wrong file compared against
> the right table. (Findings row 585; corrected by `grn_motion.py`.)

## Licence boundary

Two outside projects are cited in the tooling, on identical terms, and both
are read **only as documentation of the container format** — tag numbers,
node-type names and record layouts, which are facts about a file format:

- `github.com/SiENcE/Iris1`, `src/granny` — **GPL v2.** Documents the per-bone
  record layout and that clip length is the last translate time.
- `github.com/ptasev/Age-of-Mythology`, AoM Model Plugin — **no licence file
  at all**, which is all-rights-reserved, not permissive. Source of the
  authoritative `0xCA5E12__` node-type names.

Neither project's code was opened, translated or adopted. The flat node
directory shape above is documented by neither of them and was measured here.
Iris1's reading of `0x0C02..0x0C05` as a start/end bracket is a misread —
they are distinct node types.

## Ground truth

`tools/granny_oracle/` calls the **retail Granny 2.x runtime** directly to
check our decode against the vendor's own. That is the oracle; agreement
between our two readers is the gate (`tools/parity/grn_parity.sh`).

## Open

Eight of the 3421 animation clips do not decode. Separately, one mesh
disagrees on vertex count with an outside reading (279 against 280); the
attempt to settle it against RAD's own runtime is blocked, see
[../engine/granny-runtime-oracle.md](../engine/granny-runtime-oracle.md).

---
Provenance: `tools/formats/grn_tagwalk.py`, `grn_motion.py`, `grn_bonenames.py` module
docstrings, which carry the measurement detail; findings log rows tagged
`granny-grn`.

