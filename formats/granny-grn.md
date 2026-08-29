# Granny 1.x `.GRN` — models, skeletons, animation

**Status:** Read
**Purpose:** How a `.GRN` is laid out -- meshes, skeletons, per-bone animation
-- and which parts of it the engine consumes.

3413 of 3421 kind-65 entries decode under the current reader; row 1159 shows
only **two** are genuine decoder gaps, while six are correct refusals.

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

A `u32` node count sits at `+0`, and the `{tag, rel, children}` triples of
stride 12 begin at `+16`; the 12 bytes between the two are not read. (This
paragraph previously put the count itself at `+16`. `formats/models.gd` has
always read it at `+0` — `DIR_OFF` is the offset of the triples, not of the
count — so the code was right and the sentence was wrong.)

`376` is not a new magic number — it reconciles the constants the engine
already shipped for kind 64 (`1634 − 1258 = 376`) and extends them to kind 65
(`320 + 376 = 696`). Verified corpus-wide across all 4993 entries, in both
kinds.

Node tags in use: `0xCA5E0506` bone, `0xCA5E1200` animation, `0xCA5E1203`
transform-track section, `0xCA5E1204` transform-track keys, `0xCA5E1205`
animation section.

## Per-bone animation record (`0xCA5E1204`)

An ordinary record has a 52-byte header carrying three declared counts, then
three flat time tracks and three flat payload arrays. There is **no fixed
48-byte trailer**; row 1168 corrects that earlier decoder error. Exact size:

`52 + 16*numTranslates + 20*numQuaternions + 40*numUnknowns`

Each unknown key is one time float plus a 9-float `3x3` scale/shear matrix.
Sampled records instead carry a 12-byte header followed by `N` 68-byte poses:
time, vec3 translation, quaternion and mat3 scale/shear.

| Offset | Field |
|---|---|
| 24 | `numTranslates` |
| 28 | `numQuaternions` |
| 32 | `numUnknowns` |

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

## Two apparent defects that are not defects

Both were reported by me from renders and both are refuted by measurement.
Recorded because the renders are genuinely misleading and a later reader will
see the same two things.

**The dwarf is not distorted.** `DWARF.GRN` at the figure viewer's default yaw
of 270 reads as a formless blob — he is simply very broad seen edge-on; at
yaw 0 he is a dwarf with arms, legs, feet and a beard. His *skeleton* view is a
starburst of spokes from the waist, which looks like a broken parent chain and
is not: they are eight 4-bone chains named `Bone01`, `Bone05` … `Bone29`,
authored beard-braid/cloth bones, 32 of his 117 bones against `GLADIATOR`'s 68.
The one odd link — `Bip01 L/R Thigh` parented to **`Bip01 Spine`**, not
`Bip01 Pelvis` — is shared by `GLADIATOR` and `SERAPHIM`, both of which render
correctly, so it is how Sacred's rigs are authored. Retail's character-select
screen draws the dwarf large and his proportions and stance match the port's.

**The muddy texture regions are authored.** `Sera_legs.tga`'s lower ~60% and a
panel of `Gladiator_body.tga` are mottled dark brown against clean art
elsewhere, and they look like decode corruption. They are not — the decode
asserts an exact inflate to `w*h*2`, retail's own GL uploads correlate **1.000**
with our decode of both images, and the UV box matches retail's own
`glTexCoordPointer` array. The direct test: retail equips a new Seraphim with
**nothing** (row 1101), so the start-scene capture shows her bare, and retail's
own frame draws the same dark brown thighs between a skin-toned midriff and
white knee boots that the port draws. They look unfinished because they are the
parts equipment covers — retail's character-select Gladiator hides exactly that
midriff and thigh band behind a belt and kilt.

**Capturing retail's character select**, which is the right ground truth for
character rendering and much better than the 125 px start-scene hero:

```sh
sh analysis/tools/drive/shot.sh charsel "6000 click 512 287" 10000 16
```

That fires only the Ancaria Campaign click from `menu.sh`'s `new` route and
captures while the hero picker is still up — every playable hero as a live
Granny model, several hundred pixels each, no world load. The figures are
**equipped** there, so it settles proportion, stance and silhouette but not
base-rig skin; the unequipped start-scene Seraphim is the frame for that.

## Ground truth

`tools/granny_oracle/` checks our decode against the vendor's own. That is the
oracle; agreement between our two readers is the gate
(`tools/parity/grn_parity.sh`). **It reads a converted file, not ours:** the
chain is `.GRN` → `grn2gr2.dll` → `.GR2` → `granny2.dll`, so a 2.x runtime is
answering questions about a 1.x file that a converter has already rewritten.

### The native 1.x runtime exists, 2026-08-25

`Granny.dll` — the actual library Sacred links — was recovered from VK. Built
**2003-11-27**, copyright *1998-1999 RAD Game Tools*, 101 named exports, and
its identity is not in doubt:

| Binary | Granny symbols | Against the DLL's 101 exports |
|---|---|---|
| Windows `Sacred.exe` | 55 | imports **54**, the subset it calls |
| Linux `sacred` | 122 | carries **all 101**, plus 21 `Granny*` error-enum names — statically linked |
| DLL exports unused by either | — | **none** |

So the export surface is exactly Sacred's Granny API. A harness against this
DLL would read `.GRN` **natively** and remove the `grn2gr2` conversion from
between our reader and the vendor's, which is the one step in the present
chain nobody has audited.

Two open items above are the obvious first questions for it: the **eight of
3421 animation clips that do not decode**, and **`DUNKELELVE.GRN` and
`MAGICIAN.GRN`, the two of seven class body meshes that do not build**. Both
are our decoder being short, so the vendor's answer settles them.

> **Licence, unchanged and binding.** The DLL is proprietary RAD code. It stays
> in the unpublished workspace, is never committed, and is read only by
> observing the OUTPUTS of its exported API — the same terms
> `tools/granny_oracle/README.md` already sets, which is why no third-party
> binary lives in that directory. Nothing here has been run yet; this records
> that the oracle is available, not that it has spoken.

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

### All eight heroes at once — 44 of 45 batches, no mismatch

The character-select screen draws every playable hero simultaneously, so one
capture tests the whole rule. Predictions were written out of our reader
*first* (53 batches over nine models), then matched against the frame:

| model | batches | result |
|---|---|---|
| `SERAPHIM` | 6 | all 1.000 |
| `GLADIATOR` | 6 | all 1.000 |
| `MAGICIAN` | 6 | all 1.000 |
| `DARKELVE` | 8 | all 1.000 |
| `ELVE_SORCERESS` | 7 | 6 at 1.000, one special (below) |
| `VLADY_D` | 8 | all 1.000 |
| `dwarf` | 3 | all 1.000 (one material, three groups) |
| `Daemonia` | 1 | 1.000 |
| `VLADY_N` | 8 | **not drawn** — negative control |

Each model's batches were located by matching its *whole ordered* triangle-count
sequence, so a coincidence on one count cannot produce a hit. `VLADY_N` is the
vampiress's night form: our reader predicts an 8-batch sequence for it and that
sequence appears **zero** times in the trace, which is what a prediction about a
model that is not on screen should do. Every other run appears 330 times — once
per frame.

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

### Equipment is skinned by the ITEM, not by the mesh

A body's textures come from its own material table, above. **A weapon's do
not.** `cGranny::bindTextures` (`0x80f7866`) reads an override at `this+0x34`
and uses it *instead of* the by-name pak lookup whenever it is non-zero. The
setter is the 14-byte `0x80f7858`, and every caller follows one shape:

```
setOverride(granny, itemTexture(items, id));   // 0x8136692
draw;
setOverride(granny, 0);
```

`0x8136692(items, id)` is `*(u32*)(items + id*128 + 24)`, and its sibling
`0x81365f8` returns `items + id*128 + 71` — the model name our reader already
reads at record `+0x37`. So the engine's table base sits **16 bytes below record
0**, and the override field is **`items.pak` record `+0x08`**: a direct
`texture.pak` **entry index**, not a name and not a hash.

Proof by bijection — the twelve items naming `SHIELD_KITE.GRN`:

| item | `+0x08` | texture.pak entry |
|---|---|---|
| 1200 | 8391 | `SHIELD_KITE01.TGA` |
| 1201 | 8392 | `SHIELD_KITE02.TGA` |
| 1208–1217 | 8393–8402 | `_DARKELF`, `_IVORY`, `_IVORY1`, `_KING`, `_MASCARELL`, `_MORDREY`, `_ORGANIC`, `_VALOR`, `_VAMPIRE`, `_VAMPIRE1` |

Twelve items onto twelve textures, one to one. Corpus control over the 2652
item records that name a `.GRN` and index `texture.pak`: token agreement
between model name and texture name is **mean 0.630, median 0.667** against a
permuted control at **0.012 / 0.000** — a 50× separation. It is a general skin
system, creatures included: `WOLF` → `WOLF_VAMPDAY025`, `BEAR` → `BEAR_MAGIC`,
`NOBLE_MAL` → `NOBLE_MAL_GEIST`.

This is why `SHIELD_KITE.GRN`'s own `Shield_kite_cross.tga` is not in the pak:
the mesh's texture name is only the fallback, and no shipped item uses it.

**Watched, but not yet isolated.** In the world capture the hero's weapon is a
64-triangle batch on a 32×128 texture whose pixels correlate **1.000** with
`SWORD_BASTARD.TGA` (entry 8654) and ≤0.322 with any other 32×128 entry;
`SWORD_BASTARD.GRN` is 64 triangles and `items.pak` record 1724 names it with
`+0x08 = 8654`. Consistent — but *not* discriminating, because that mesh names
`sword_bastard.bmp`, whose stem resolves to the same texture. Isolating the
override needs a case where the two routes disagree (a kite shield, or a
variant-skinned creature), and neither was on screen in the scenes captured.

### The one exception, and it is a bug on our side

`ELVE_SORCERESS`'s 372-triangle hands batch is the only one of the 45 our reader
could not match, because it resolves the name to **nothing**: `find_model_texture`
returns -1 for `elve_sorceress_hands.bmp`. Retail binds a real 16×16 texture
there — a flat skin block, ARGB4444 `0xFDA8` — and never logs a miss.

`ELVE_SORCERESS_HANDS.TGA` *is* in `texture.pak`, entry 7189, with a declared
size of **15** — the zlib payload length, not the entry length (see
[pak-containers.md](pak-containers.md)). The port's stem index skips any entry
whose declared size is under 32 bytes before it ever reads the name, so 28
entries are invisible to it, 8 of them referenced 13 times from `models.pak`
(`*_HANDS` placeholders for the wood elf, novice, witch, amazon, Boron priest
and priestess, elf ghost). Those batches render untextured in the port and
skin-coloured in retail. One gate short of a one-line fix; recorded here first.

Ten `.tga` path strings live in the entry and only **three** are referenced —
`Temporary_Gladiator_Mapping/Gladiator_body`, `…/Gladiator_boots` and
`Gladiator_new_exports_2/Gladiator_Head`. The `Models/Maps/` set (`arms`,
`hands`, `legs`, `belt_skirt`) is a superseded export left in the file; four of
the six materials share the one combined body atlas.

### Which SUBMESH a draw batch paints — the file says so

A group's `mesh` field is **not** the reader's submesh number, and the two
orders differ per file with no single rule:

| entry | reader's submeshes (triangles) | group order |
|---|---|---|
| `GLADIATOR.GRN` | 868, 385, 442 | 442, 868, 385 — a 3-cycle |
| `DWARF.GRN` | 824, 136, 1384 | 1384, 824, 136 — the same 3-cycle |
| `WALDELFE_DARK.GRN` | 1282, 116, 394 | 116, 1282, 394 — a transposition |
| `SERAPHIM.GRN` | 144, 1910 | 144, 1910 — the identity |

**`group.mesh` is a 0-based index into the FormMesh list, and each FormMesh
(`0xCA5E0C03`) node's int32 payload is the 1-BASED index of its Mesh node
counted over ALL Mesh nodes in directory order.** That is the same reference
the bone-list pairing above already trusts, read for a second purpose rather
than re-derived.

Measured over every entry carrying groups: on **1563 of 1565** the submesh this
resolves to has exactly the triangle total its groups declare. The two that
disagree — `SERABFG.GRN` and `EDLST_RUND_GESCHL_KLEIN.GRN` — resolve to the
same submesh a triangle-count rule picks and disagree only on the count, each
declaring a single triangle against a 114- and an 80-triangle mesh. So the join
is unanimous and the residue is those two files' own group data.

**What it was worth.** The port reconciled the two orders by triangle count and
refused the split whenever two submeshes had the same face count. That refused
**143 of 1567** entries, and the twelve of them naming more than one texture
rendered as flat untextured clay — including `THIEF2_FEM.GRN` (8 materials) and
`ELVE_SORCESS.GRN` (7), both character bodies, drawn as featureless
silhouettes. The count rule agreed with the reference wherever it decided at
all, so this is the same answer without the refusals.

**Triangles no group claims are never drawn by retail** (row 1167). The
character-select apitrace identifies `ELVE_SORCESS` by its full ordered group
sequence and observes exactly seven `glDrawElements` batches — its seven
declared groups — with no six extra 14-triangle calls for the six wholly
unclaimed submeshes. Fully grouped `SERAPHIM` and `GLADIATOR` controls emit
exactly their six declared batches. The binary has one real indexed-model draw
callsite, `sub_8053772:0x8053935`, so the trace is exhaustive: there is no
default-material or separate leftover pass. Port rule: emit only explicit
group index lists; count but skip every unclaimed triangle, even for
single-texture entries.

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

### The FormMeshBone pairing rule

A mesh's weight stream stores **local** bone indices; a `FormMeshBoneSection`
turns them into global bone ids. Which section belongs to which mesh **is
stated in the file** (row 1009), though it took the one undecidable entry to
find it: each `FormMeshBoneSection` (0xCA5E0C09) hangs under a `FormMesh`
node (0xCA5E0C03), and the FormMesh's own payload int32 is the **1-based
index of the Mesh node its bone list belongs to**, counted over *all* Mesh
nodes in directory order — non-drawable ones included (six retail entries
carry those, which is what fixed the index space). Validated corpus-wide
before being trusted: on all 971 weight-declaring entries the payloads are
unique, in range, and hand every drawable mesh a list satisfying
`len(list) >= highest + 1`; everywhere the matching below decides on its
own, the two agree by content — zero disagreements. The entry that forced
the find: `DUNKELELVE.GRN` (402), whose two size-10 lists compete for the
meshes needing 9 and 10, spatially inseparable, so no counting or distance
rule could break the tie.

The earlier negatives stand, reframed:

- **Containment under `Mesh` nodes** — still false; the sections were never
  children of the *Mesh*. They are children of the *FormMesh*, and the
  FormMesh names the Mesh.
- **Position** — still refuted as a rule. `DWARF_BLACK_BODY.GRN`'s three
  meshes need 27, 3 and 12 bones while its sections run 12, 27, 3; the
  FormMesh payloads are exactly the permutation that reorders them.

The constraint `len(list) >= highest + 1` — a mesh's local indices must fit
inside its list, prefix use allowed (`SERABOOTS01.GRN`) — remains the
validity check on the reference, and the **perfect matching** under it,
accepted only when every valid matching hands each mesh the same list,
survives as the fallback for a malformed reference, which no retail entry
has.

| rule | decodes |
|---|---|
| equality search (previous) | 736 |
| perfect matching (current) | 802 |
| both succeed | 736 — **agree on all 736, 0 disagreements** |

Zero regressions, so this is a strict generalisation rather than a different
answer.

**The residue is genuinely ambiguous, and refusing it is correct.** The 169
that still fail are *left/right symmetric* pieces whose two lists are the same
size but different bones — `SERABOOTS01`'s sections are `[6,7,8,5,12]` and
`[10,11,12,9,8]`, one leg each; `SERASHOULDER01`'s are `[6,8,7,9]` and
`[6,10,7,11]`. A coin flip would bind one boot to the opposite leg, which is
worse than leaving it off. The next lever is **geometric** — score each
candidate matching by the distance from a mesh's vertices to its assigned
bones' rest positions — with the 736 already-decided entries as the control.

### Wearing a garment: the join, and the control that nearly passed

Armour is put on by remapping the garment's skin binds onto the wearer's
skeleton **by bone name**, keeping bind order (which is what the mesh's
`ARRAY_BONES` indexes) and keeping each bind's own pose (a fact about the
garment's geometry, not the wearer's).

The naming gap that looks fatal is not. Uriel's Legacy pieces carry 75-77 bones
against `SERAPHIM.GRN`'s 72 — the extras are `Angel_armor_Breast`,
`Bip01 Ponytail1`, `Bip01 Footsteps`, two `Spot*.Target` light aims — but every
one is an **unweighted locator**. Bones that are weighted *and* absent from the
body: **zero, across all seven pieces**.

**The name test discriminates nothing, and only the control says so.** Offered
those seven garments, `GLADIATOR.GRN` binds 5 and refuses 2 — *exactly*
`SERAPHIM.GRN`'s own score. Every humanoid shares the `Bip01 *` biped names.
This is the same degenerate comparison the weapon-socket instrument produced
below, and it would have shipped as "armour composition works".

What separates is **R1.4's local instrument**: the bind bone's own rest in the
garment against the same-named bone's rest in the body, chain never composed.

| wearer | HELMET01 | GLOVES01 | ARMOR01 | BELT01 | WINGS01 |
|---|---|---|---|---|---|
| `SERAPHIM` | 1/1 | 1/15 | 5/7 | 3/5 | 2/7 |
| `GLADIATOR` | 0/1 | 0/15 | 0/7 | 0/5 | 0/7 |
| `DWARF` | 0/1 | 0/13 | 0/7 | 0/5 | 0/7 |

So admission is *at least one bind bone agrees* — weak-looking, totally
separating on what has been measured. The own-rates are low and uneven because
a garment poses fingers and extremities freely; a per-bone fit score would be a
better rule and needs more than five pieces to set a cut on.

This matters rather than being a nicety: `rust.bin` exists to say which mesh an
armour *becomes* for a different wearer, so binding one to the wrong body is a
real error.

### External prior art, and where it disagrees with us

The only public implementation of Sacred's attach maths is the community **GRN
Model Viewer** (`https://sacred-tribute.com/3d/`), a self-contained HTML/JS
reconstruction derived from ptasev's Age of Mythology Granny work. Every other
Sacred tool — SacredMagician, SacredUtils, SacredGameTools, bssth/sacred-sdk —
is balance/save/data work and says nothing about mesh attachment. Documentation
only; no code was taken.

Its model, in prose: the wearer's hand-bone rest **world** matrix rebuilt with
**uniform** scale (cube root of the absolute product of its three components —
Sacred's Biped bones carry non-uniform scale that otherwise shears the weapon),
the weapon's grip world matrix rebuilt with scale forced to exactly `(1,1,1)`,
attach = `hand × grip⁻¹`, baked into the vertices and rebound to that one hand
bone at weight 1.0. Armour is the same invert-and-compose shape but **per bone**,
matched by name never by index, unmatched armour bones skipped. Handedness: an
item sent to the left hand with no `Bone_weapon_02` falls back to `_01` with a
`scale(-1,1,1)` mirror; where `_02` exists, **no mirror** — mirroring anyway
puts the weapon in the right place backwards.

**Where it disagrees with this project, unresolved.** The viewer attaches to
`Bip01 R/L Hand` (shields to `Bip01 L Forearm`) and **ignores the wearer's
`Bone_weapon_*` entirely**; its author flags as an open unknown what that bone
is then for. Our census says substituting the hand is invention with a known
failure rate — the socket-to-parent-hand rotation is within 15° on 299 entries
but **90–180° on 107**, and on `SOLDIER` it laid a kite shield flat across the
chest. Our refusal has more measurement behind it; the viewer renders equipped
characters correctly. Neither is settled. For `SERAPHIM` the question is moot on
*position*: her `Bone_weapon_01` sits at local origin `(-1e-6, 0, -1e-6)` on
`Bip01 R Hand`, exactly at the hand, so the two models differ only in rotation.

### The base body already wears boots

`SERAPHIM.GRN`'s own sub-meshes include **`legs` and `shoes`**, and its own
texture list includes **`Sera_boots.tga`** — measured here and independently
read off the mesh name table by the outside sweep. So equipping `SeraBoots01.grn`
on a body that still draws `shoes` puts **two boots in the same place**, which is
exactly the stack the port renders. It generalises: `Gladiator_boots.tga`,
`magician_boots.tga`; Dwarf and Wood Elf use one whole-body texture instead.

Retail must therefore **hide the base sub-mesh an armour piece covers**, and how
it chooses is documented nowhere — the viewer does not implement hiding at all,
it stacks and offers manual toggles. The likely vocabulary is the 18-slot
equipment array at `cCreature + 0x1A4` (main hand `0x0D`, off hand `0x0C`, mount
`0x12`, slots `0x00`–`0x06` helmet/body/belt/arms/legs/**shoes**/gauntlets),
whose slot names line up one-to-one with the base-body sub-mesh names.

**Open:** there is no starting-equipment table in any Sacred 1 data file, and
none is documented anywhere — not the manual, not `balance.bin`, whose decoded
fields are skill unlocks, experience values and spawn counts only. The
`EquipNPC` opcode is real and used, but the Vampiress's 76 records all equip
**horses**, none the hero. Where retail's two Seraphim blades come from is
unanswered.

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

~~Five of `texture.pak`'s 25535 names are malformed.~~ **STRUCK 2026-08-25,
row 1090. They are not malformed — the name field is 32 bytes and they fill
it.**

`SHADOWDOT.TGA` is 13 characters, then NUL padding to byte 32, then the u16
pair `w=64 h=64`. So the field is **32 bytes, NUL-padded**, and a name that is
exactly 32 characters long carries no terminator at all. A reader that scans
for a NUL then runs straight into the width field, which is where every
"malformed" byte came from:

| name (32 chars) | next u16 pair | the stray byte |
|---|---|---|
| `AMAZONE_ARMOUR_KURZKHEMD_CELAL.T` | 256 × 256 | — the `\x00` is the low byte of 256 |
| `GIGANT_SPIDER_HAIR_RED_DEMON.TGA` | 64 × 16 | `@` = 0x40 = 64 |
| `INSTRUMENT_HARFE_128X128_ALPHA.T` | **128 × 128** | `\x80` = 128 |
| `PHEX_SHE_THIEF_HELMET_LEATHER.TG` | 64 × 64 | `@` = 0x40 = 64 |

`INSTRUMENT_HARFE_128X128_ALPHA` self-confirms: the name says 128X128 and the
bytes read 128 × 128.

**Seven, not five.** Scanning all 25,535 entries, 25,528 are NUL-terminated and
**7** fill the field: the four above plus `ELVE_SORVERESS_LEDERHARNISCH.TGA`,
`HORSE_BRIDLE_LEATHER_METAL01.TGA` and `HORSE_BRIDLE_LEATHER_METAL02.TGA` —
those three are exactly 32 characters *with `.TGA` intact*, so they read
correctly by luck and were never flagged. The previously-listed fifth entry
`TEX\x03\xbfC` does **not** reproduce as a literal byte search and is
unaccounted for.

Two of the four have a truncated extension (`.T`, `.TG`) because the full name
would exceed 32 characters. Retail cannot reach those two — its key is built by
appending `.TGA` to a stem, and the stored name is already cut — so that half
of the original observation stands. The other two are reachable and were only
ever a decoder artefact.

Of the eight historically refused kind-65 entries, six are intentional:
`INVALID_MOTION` is a 256-byte stub; `GLADIATOR`, `SD01_ACTIVATE`,
`WILB_DYING_C` and `WIZARD` carry no transform-key records; and
`ANDD_ATTACK_SPECIAL01` contains one non-normalizable quaternion.

`FX_E_IDLE_BH` and `FX_G_IDLE_BH` are fully decoded (row 1168), as are 26
other hybrid entries found by the corpus sweep. Per-record classification is
content-defined: SAMPLED iff count dwords `+24/+28/+32` are all zero and the
span is `12 + 68*N`; otherwise ORDINARY with the size formula above. Across
258,534 records, all 128 count-zero records are valid sampled streams with zero
false positives. Duplicate bone ids retain first/ordinary ordering, and their
sampled record replaces the sparse ordinary output because it is the dense
evaluated form of the same complete transform. No FX decoder research remains.

**169 of the 971 weight-declaring meshes — 17.40% — still have no
unambiguous FormMeshBone pairing**, down from 240 (24.72%). See "The pairing
rule" above for what changed and why the residue is refused rather than
guessed.

~~Two of the seven class body meshes do not build at all: `DUNKELELVE.GRN` and
`MAGICIAN.GRN`.~~ **STRUCK 2026-08-26. Both build fully**, and textured on
every surface — DUNKELELVE 1856 verts / 2115 tris / 8 of 8 surfaces textured,
MAGICIAN 1316 / 1516 / 6 of 6. The claim outlived whatever decoder shortfall
produced it; it was re-tested directly, not argued away.

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

