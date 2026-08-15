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

## Why the clip-to-mesh match is believed

The shipped names do not line up, so a clip is matched to a mesh by bone
geometry: shared bone names, and rest origins agreeing within a tolerance.
That resolves 121 of 124 distinct creature meshes — and a resolution rate,
on its own, is not evidence. It could equally mean the clip corpus is dense
enough that anything matches something.

So the gate now measures what the score would be **if the hypothesis were
false**. It permutes each mesh's bone origins among that mesh's own bone names
— same bones, same names, same positions, same counts, so the matched-bone
denominator cannot shrink — destroying only the name-to-position
correspondence. Real pairs score **0.844 and above; permuted pairs 0.489 and
below**, and the gate fails if those distributions come within 0.20 of each
other.

A cross-pairing control was considered and rejected: creatures genuinely share
rigs, which is the reason this machinery exists, so pairing one mesh with
another's clip can be *correct*. A control that can accidentally be right is
not a control.

## The material chain — which image a draw batch samples

A mesh entry is drawn as **one batch per group**, and the group's material
number is *not* a texture number. Two links, both measured on `GLADIATOR.GRN`:

| Tag | Record | The field that matters |
|---|---|---|
| `0xCA5E0E02` | group | `{u32 mesh, u32 material (one-based), f32, f32}` |
| `0xCA5E0E04` | group count | triangle count is the second `u32` |
| `0xCA5E0E06` | group triangles | `u32 count`, then 16 bytes per triangle whose first `u32` indexes the **submesh's** triangle array |
| `0xCA5E0D01` | MaterialSection | container |
| `0xCA5E0D00` | Material | 16 bytes; `+4` is a **one-based texture reference** |

**The material is not the texture.** `GLADIATOR`'s six materials reference
textures `2,6,1,3,4,5` — a permutation, so reading the material number as a
texture number put the head batch on the body image, an arm on the boots image
and a leg on the head image, while every batch still carried plausible leather.
That is why a corpus census passed while the character rendered scrambled. The
structural check is a permutation test: where an entry has as many materials as
textures the references must be distinct and in range, and they are on **1546 of
1546** equinumerous entries, **131** of them non-identity.

**A group names its own triangles, and they interleave.** Slicing the submesh's
triangle range in group order assumes a contiguous layout. `GLADIATOR`'s
442-triangle leg submesh splits 212 skin / 230 boot with the boot's indices
running `28..441` *through* the skin's, so the slice put half a boot on a thigh.
The invariant is exact: across a submesh's groups the indices cover `0..n-1`
once — measured on 442/868/385 triangles, no gaps, no repeats.

Both are gated by `engine/checks/skin_check.gd` (joins 6 and 7), and the
interleave gate additionally requires at least one submesh to be non-contiguous,
so it cannot be passed by the slice it replaced.

### Confirmed against retail, at pixel identity

The permutation is not our reader's inference — **retail's own GL calls agree
with it batch by batch.** An `apitrace` capture of a save-load run (hero
`SERAPHIM.GRN`) draws the hero twice per frame: a shadow pass on a 16×16 dummy
texture, then the lit pass. The lit pass issues exactly **six**
`glDrawElements` in the file's own group order —

| group | triangles | our link says | retail uploaded | correlation | next best |
|---|---|---|---|---|---|
| 0 | 144 | `Sera_hair` 128×128 | 128×128 | **1.000** | boots 0.26 |
| 1 | 358 | `Sera_body` 256×256 | 256×256 | **1.000** | legs 0.32 |
| 2 | 612 | `Sera_arms` 256×256 | 256×256 | **1.000** | legs 0.29 |
| 3 | 390 | `Sera_legs` 256×256 | 256×256 | **1.000** | body 0.32 |
| 4 | 280 | `Sera_head` 256×128 | 256×128 | **1.000** | — |
| 5 | 270 | `Sera_boots` 128×128 | 128×128 | **1.000** | hair 0.26 |

`GLADIATOR` confirms it a second time, on a *different* permutation and from a
cheaper capture — the character-select screen renders all eight heroes as live
Granny models, so no world load is needed:

| group | triangles | material | our link says | retail uploaded | corr |
|---|---|---|---|---|---|
| 0 | 212 | 0 | `Gladiator_body` 512×512 | 512×512 | **1.000** |
| 1 | 230 | 3 | `Gladiator_boots` 256×256 | 256×256 | **1.000** |
| 2 | 260 | 1 | `Gladiator_body` 512×512 | 512×512 | **1.000** |
| 3 | 258 | 5 | `Gladiator_body` | *reused, no upload* | **1.000** |
| 4 | 350 | 2 | `Gladiator_body` | *reused, no upload* | **1.000** |
| 5 | 385 | 4 | `Gladiator_Head` 512×256 | 512×256 | **1.000** |

The two *skipped* uploads are themselves evidence: retail re-binds nothing
between the 260, 258 and 350 batches, which is the engine saying materials 1, 5
and 2 name the same image — exactly what the permutation `2,6,1,3,4,5` says.
Identity is refuted on three of the six here (230 would take head not boots, 350
boots not body, 385 body not head).

Retail re-uploads the skin immediately **before each batch** as `GL_BGRA` +
`GL_UNSIGNED_SHORT_4_4_4_4_REV` — so the pixels are in the trace, and the match
above is against the actual image, not merely its dimensions. (That upload
format is also retail's own statement that the payload is ARGB4444.)

Reading the material index as a texture index would have put a *different* image
on four of the six batches, so the identity mapping is refuted by measurement.

The Granny runtime is statically linked into the retail ELF — the `.GRN` tag
constants sit in its `.text` (`0xCA5E0D00` at `0x8064e5c`, `0xCA5E0D01` at
`0x8066975`, `0xCA5E0E06` at `0x8065f06`). That loader is **deliberately not
read**: doing so would settle the format faster and destroy the clean-room
posture the whole decoder rests on. The trace above is retail's *output*, which
carries no such problem.

### From the texture name to the pak entry

The join is retail's own code, and it is an **exact name, not a stem**.
`cGranny::bindTextures` (`0x80f7866`) walks the runtime's new-texture list and
for each one does `sprintf("%s.TGA", name)`, uppercases it, and looks up that
exact string — logging `Texture [%s] not in PAK! IGNORED!!!` on a miss.

The lookup (`0x83c3fde` → `0x83cbd9c`) is a linear-probed hash table keyed on
two further hashes of the name, **with no string compare**; its inserter
(`0x83cbc0a`) probes to the first free slot and never overwrites. So where a
name is duplicated the **first entry in pak order wins** — which is what the
port's stem index already does, now confirmed rather than assumed.
`texture.pak` duplicates 25 names, six of them with differing payloads
(`DAEMONIA_1024`, `DRYADE_SCOUT_ARMS/HEAD/LEGS`, `DWARF`, `DWARF_WARHAMMER`).

### Which side is the front

Orientation is read off the retail bitmap, not off the render.
`Gladiator_body.tga` (512×512) lays out the torso **front** bottom-left
(shoulder strap into a sternum plate), the torso **back** bottom-right (X-lacing
between the shoulder blades), and the leg/kilt panel top-right. The port's
staged viewer at yaw 270 shows the sternum plate and a face; at yaw 90 the
X-lacing and the back of the skull with the braid behind it. Front and back
agree with the atlas, so the mesh is not mirrored.

Ten `.tga` path strings live in the entry and only **three** are referenced —
`Temporary_Gladiator_Mapping/Gladiator_body`, `…/Gladiator_boots` and
`Gladiator_new_exports_2/Gladiator_Head`. The `Models/Maps/` set (`arms`,
`hands`, `legs`, `belt_skirt`) is a superseded export left in the file; four of
the six materials share the one combined body atlas.

## Equipment sockets

A weapon is attached by a **named socket that exists on both sides of the
join**, and the two sides are told apart by the socket's *parent*. Census over
all 1571 rigged `models.pak` entries:

| socket | parent `Bip01 R/L Hand` | parent `__Root` |
|---|---|---|
| `Bone_weapon_01` | 235 — the right-hand socket | 206 — the grip, in the weapon's own space |
| `Bone_weapon_02` | 192 — the left-hand socket | 176 |

**Zero entries cross**: `_01` never hangs off a left hand, `_02` never off a
right one. The two populations are disjoint by parent, so one pair of names
carries two meanings — *where I hold it* on a wearer, *where it is held* on a
weapon — and attaching is aligning the second to the first.

`_01` is the **main hand**, measured rather than read off the number.
`startcode.bin`'s tag-0x02 occurrences 1 and 2 are the two hand slots, and over
the 1111 armed slots the eight classes declare:

- shields are **58.8%** of slot 2 against **10.5%** of slot 1 — a rate, 5.6×;
- all 272 slot-2 meshes carry `Bone_weapon_02`, while **97 of 839** slot-1
  meshes do not — the polearms (`PIKE`, `SPEAR`, `HELLEBARDE`, `STAFF_FIGHT`),
  which have only a main-hand grip.

The second is the load-bearing one: **zero** items sit in the off hand without
an off-hand grip, which the opposite slot assignment could not produce.

Only 192 entries carry the off-hand socket at all, so a body lacking it falls
back to `Bip01 L Hand`. That is measured coincident with the socket (~1e-6) on
the two NPC bodies carrying both — but **3.86 units away** on the DAEMONIA set,
so the fallback is counted and reported, never treated as equivalent.

### A weapon is a rigid prop, not a second garment

Armour shares its wearer's skeleton (R1.4, `checks/equip_check.gd`). A weapon
does **not**. Pointing that same local-rest instrument at the hand meshes
returned median own-agreement `1.0000` **and** median cross-control `1.0000` —
the control scoring exactly as well as the subject, i.e. measuring nothing. The
bone *names* say why: a weapon carries 3–16 bones of its own and shares only
`__Root` and the socket names with a body, so the matched names were the
sockets agreeing with themselves.

199 of the 221 entries carrying a weapon-side grip declare **no vertex weights
at all** — `SWORD.GRN`'s `MeshWeights` node has a span of exactly the 12-byte
header. A weapon is an unskinned prop on a moving socket, and its bones are
locators. "Declares no weights" and "weights did not decode" must therefore be
distinguished, or every weapon reads as a decode failure and every genuinely
broken skin reads as a prop.

## Open

Five of `texture.pak`'s 25535 names are malformed — `TEX\x03\xbfC`,
`AMAZONE_ARMOUR_KURZKHEMD_CELAL.T`, `PHEX_SHE_THIEF_HELMET_LEATHER.TG@`,
`INSTRUMENT_HARFE_128X128_ALPHA.T\x80`, `GIGANT_SPIDER_HAIR_RED_DEMON.TGA@` —
and none has a correctly-named sibling. Retail can never reach them (its key is
built by appending `.TGA`); the port's stem index can. A five-entry divergence,
recorded rather than fixed.

Eight of the 3421 animation clips do not decode.

~~One mesh disagrees on vertex count with an outside reading (279 against
280).~~ **Closed 2026-08-15 against retail's own index array**, not against the
blocked converter: `GLADIATOR`'s 385-triangle head submesh references **279**
distinct vertices in our reader, and retail's `glDrawElements` for that same
batch — 1155 `u16` indices, captured off the character-select screen — resolves
to exactly 279. The outside 280 is the reading that is wrong.

~~The same submesh's UVs run outside `0..1`.~~ **Not a defect.** Retail's own
`glTexCoordPointer` array for that batch has the identical box —
`u -3.194..3.194`, `v -11.277..0.961`, 6 of the 279 vertices outside the unit
square — and retail draws it with `GL_TEXTURE_WRAP_S`/`_T` set to `GL_REPEAT`.
Authored data consumed under repeat; the reader needs no change.

---
Provenance: `tools/formats/grn_tagwalk.py`, `grn_motion.py`, `grn_bonenames.py` module
docstrings, which carry the measurement detail; findings log rows tagged
`granny-grn`.

