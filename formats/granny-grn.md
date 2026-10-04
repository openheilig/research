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

### Sacred's authored motion references (finding 1244)

Before the kind-64 root tag, Sacred's model wrapper carries 256 little-endian
u32 motion references starting at byte **112**. A native motion enum selects
`112 + 4*enum`. The value is an ordinal among **kind-65 entries in archive
order**, not an index in the mixed-kind pak table. Zero denotes no motion;
kind-65 ordinal zero is the invalid-motion entry and still occupies its slot.
These are Sacred wrapper fields, not generic Granny node-directory fields.

LGP `sub_810D0FC` checks this reference and `sub_810D1F2` consumes it. The
running Seraphim's motion enum 2 selects ordinal 1467, `SERA_IDLE_BH.GRN`,
mixed pak entry 3039; her whole 1,024-byte motion table matches the shipped
header (finding 1238). All 5,110 nonzero references in the primary archive
resolve within its kind-65 table.

`Models.native_motion_entry(model_entry, motion)` implements this lookup
without geometric matching, returning -1 for absent, invalid or out-of-range
references. It resolves the supplied Pak only, not runtime overlay/model
replacement. The viewer consumes it through `--grn=NAME --motion=ENUM`.
Choosing the enum from actor state and equipment is a separate engine rule;
neither a filename suffix nor the viewer's explicit enum supplies that rule.

Verification: corpus-wide reader comparison and live model-viewer playback;
enum 2 binds 69 of 70 named tracks on the Seraphim, with `Bip01 Footsteps`
unbound. This is not a claim of native pose parity. Evidence:
`donotpublish/tmp/actor-pose-20260907/native_motion_reader_check.gd` and
`native-motion-viewer.png`.

### Action and weapon-mode selection table (finding 1248)

The ordinary selector indexes a six-row, twenty-column u32 table by
`20 * action_row + weapon_mode`. The complete 480 bytes agree in Linux
LGP (`0x87938E0`), Gold 2.28 ENG (`0x956E40`) and RUS (`0x955E48`).
Consumers are respectively `sub_8197C8A`, `sub_5467A0` and `sub_546BA0`.
The USA demo's table is at `0x7B80E8`: all six rows agree in columns 0–12,
but columns 13–19 are zero. Gold adds values in columns 13 and 14 only.
Each binary has one matching candidate; a one-dword-shifted control fails.

For weapon modes 0–12, the table is described by these formulas:

| Action row | Authored clip family | Motion enum |
|---|---|---|
| 0 | IDLE | `1 + mode` |
| 1 | WALK | `27 + mode` |
| 2 | RUN | `40 + mode` |
| 3 | FIDLE | `14 + mode` |
| 4 | ATTACK variants | `79 + 5 * mode` |
| 5 | DEFEND | `53 + mode` |

Gold column 13 is respectively 235, 237, 238, 236, 241, 239; column 14
repeats column 12. This table is **not the whole selector**: actor flags,
mounts, type-specific branches, available-motion checks and attack variation
also participate. For missing walking/running motions, the Linux and ENG
consumers try their own family's first three enums, then the other family's
first three; successful fallback writes locomotion mode 1 or 2 at actor
`+504`. Exhausting those candidates returns enum 2 if available, otherwise 1.
Other nonzero, non-attack rows can fall back to row 0. Do not replace these
rules with geometric similarity.

The Seraphim's authored entries explicitly include `SERA_WALK_1H.GRN`
(enums 27–29), `SERA_RUN_1H.GRN` (40–42), and `SERA_IDLE_BH.GRN` (1–3).
Thus the port's earlier claim that she has no walk is a resolver failure,
not absent content. `NOVIZIN02.GRN` names `NOVI_IDLE_BH.GRN` at enum 1,
not the geometrically selected `PRSS_IDLE_BH.GRN`.

**Live control:** in `run14`, an open-floor click produces sampled enum
`2 → 41 → 2`, with enum 41 at 48,055 ms and a visibly running figure in
the 48,000-ms frame. The same equipped references `[0,18]` remain throughout.
The bench-target control `run13` samples only enum 2. Both record 30 samples
with no identity failures and a proper timeline end. In both, actor `+504`
changes from 0 to 2 and remains 2 after the figure is idle; it is therefore
not interchangeable with the currently playing motion enum.

Evidence: `donotpublish/tmp/actor-pose-20260907/measure_motion_table.py`,
`motion-table-measurements.json`, `motion_table_clips.gd`,
`measure_locomotion.py`, `locomotion-measurements.json`, and `run13/` /
`run14/`. Gameplay integration remains open.

### Equipped weapon input (finding 1245)

LGP `sub_8368E26` reads the weapon's item type at object `+16`, resolves its
weapon-table index through the item's u16 at `+40` (`sub_814D1CC`), and
returns byte **30** of the corresponding **258-byte** `weapon.pak` row.
This is a weapon-mode input, not a motion enum.

Two independent start-scene observations, at 42 and 60 seconds, identify the
held reference at actor `+472` as reference 18, type **7901**, with the
`cWeapon3D` vtable. Actor `+468` is empty. Its weapon index is **4748** and
mode byte is **1**. All 258 bytes match the shipped row in both body and
shadow snapshots; the adjacent row does not match, and byte 31 is zero.

The later capture also observes motion enum **2**, active playback of
`SERA_IDLE_BH.GRN`, and rate **1.25**. The earlier capture instead has enum
zero and no active control; its motion-identity guard fails explicitly.
Thus the equipment input is stable across these samples, but equipment
alone does not establish current playback. This does not prove the
transition between the samples, which came from separate processes.

The native hand-precedence rule is in `sub_81988EC`: resolve both references
as `cWeapon3D`; with two eligible weapons and the first not satisfying
`sub_813813A`, any nonzero weapon mode produces mode 8. Otherwise the second
resolved hand overrides the first; mode 14 becomes 12 when its argument is
false. These branches are statically read, not all behaviourally exercised
by the one-equipped-hand observation.

Evidence and reproducible byte comparison:
`donotpublish/tmp/actor-pose-20260907/{run07,run08}/`,
`measure_equipped_motion.py`, and `equipped-motion-measurements.json`.

**Same-process control (finding 1246):** a bounded, read-only timeline in
`run09` re-resolves the actor through the object manager and verifies
reference, address and type on each sample. All 28 samples between 42,026
and 69,118 ms retain enum 2, one unchanged active-control tuple and equipped
references `[0,18]`; no identity guard fails. Sampling is once per second,
not a claim that no shorter transition occurred. This run already has active
playback at 42 seconds, so `run07` must not be generalized into a fixed
startup delay. Raw events and `run09/timeline-summary.json` preserve the
control. The cause of the earlier inactive sample is not established.

### Authored local animation poses (finding 1247)

**Correction, 2026-09-07:** the port's rest-relative retargeting rule from
finding 766 was wrong. A clip key is an absolute local pose, not a delta to
be multiplied by `model_rest * inverse(clip_rest)`. Dropping position tracks
was also wrong. Production `ModelView.build_animation` now binds both
authored local channels directly, retaining the existing curve interpolation
and parentless world-placement exclusion.

The discriminator is a frozen pose comparison against native runtime bones,
not agreement with the model's bind pose. Native model and motion identity,
last evaluated animation time, every bone name and parent, and all 300-byte
bone records were captured from two actors. Coordinates are reconciled by
the captured animation transform's Y reflection, not a fitted rotation.

| Model | Compared model-space bones | Old maximum position error | Corrected maximum position error | Old maximum rotation error | Corrected maximum rotation error |
|---|---:|---:|---:|---:|---:|
| `SERAPHIM.GRN` | 71 | 2.23904 | 0.000929 | 78.0417° | 0.004283° |
| `NOVIZIN02.GRN` | 72 | 0.758285 | 0.000441 | 90.0000° | 0.001935° |

Position errors are model units. Rotation errors use normalized
double-precision quaternion dot products; Godot's float32 `angle_to` is too
coarse for the corrected residuals. The native `__Root` contains world
placement and is explicitly outside this standalone model-space comparison.
No other bone is excluded or missing. Samples are at native local times
0.8300000131 and 0.8575021327 seconds, respectively.

The old `anim_check` enforced the disproven rest-relative formula and was
removed. `hero_anim_check` now checks actual pose changes over time rather
than a fitted distance from model rest. This establishes local pose binding,
not full actor rendering parity: production clip selection, world placement,
lighting, playback timing and scene insertion remain separate contracts.

Evidence: `donotpublish/tmp/actor-pose-20260907/{run10,run12}/`,
`prepare_pose_comparison.py`, `compare_native_pose.gd`, and
`pose-binding-measurements.json`. The comparison's `legacy` arm restores the
old retargeting and dropped positions as an explicit counterfactual.

**Running control, finding 1249:** a state-triggered capture (`run17`) waits
for object reference 1 to select enum 41, rather than assuming a wall-clock
click has started movement. `SERA_RUN_1H.GRN` at native local time
0.16829998834133164 compares all 71 non-placement bones. The legacy binder's
maximum rotation/position errors are 93.420217° / 2.146717 model units;
production gives 0.216043° / 0.00009156. The active control has no secondary
sequence and the actor's blend-end field is zero.

The remaining angle is **not closed**. Inserting a directly evaluated
quadratic quaternion key at the exact captured time gives 0.214369° maximum
error (`Bip01 L Calf`), so ordinary eight-way track subdivision does not
explain most of it. This isolated `exact-time` arm is not production code,
and no curve change follows from it. Evidence: `run17/godot-pose-*.json`
and `compare_native_pose.gd`; normalized errors use the same metric above.

**Resolved cause, finding 1251:** the preceding exact-time result normalized
each quaternion control point before the blend. That changes its effective
weight. Retaining the stored control magnitudes and normalizing only the
evaluated quaternion reduces the running exact-time maximum error from
0.214369° to **0.00003931°**. The independent Seraphim-idle and novice-idle
captures give **0.00003432°** and **0.00002888°**, respectively.

`Models._clip_ordinary_record` now preserves mode-2 quaternion controls;
`ModelView` already normalizes evaluated poses before inserting Godot keys.
Production still uses eight subdivisions per span: running maximum error
falls from 0.216043° to **0.023341°**, while idle residuals remain about
0.004277° / 0.001937°. Thus premature normalization is corrected, but exact
curve evaluation is not implemented. The synthetic
`quaternion_control_check` fails before the correction and passes afterward
by checking a sampled orientation, not a quaternion-length convention.
Evidence: `run{10,12,17}/godot-pose-{raw-exact-time,production-raw-controls}.json`
and `raw-quaternion-running.png` in the same evidence directory.

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

## The container, reconciled (2026-09-01)

The open-grn project (see Licence boundary) publishes a container model
that looked nothing like ours — signature at 0x00, header at 0x40, section
table at 0x60 — yet every one of its *fields* checks out on our bytes once
it is anchored at the right base. Measured with `tmp/opengrn_check.py`
against `install/pak/models.pak` (GLADIATOR.GRN, SERAPHIM.GRN, BAT.GRN as
kind 64; BATX_ATTACK_BH_A.GRN and GLADIATOR.GRN as kind 65):

- A fixed 64-byte **signature blob** ends exactly at the root tag, i.e. it
  occupies `magic_offset − 0x40` (`0x100` kind 65, `0x4AA` kind 64). Its
  bytes are constant across entries and kinds, so `magic_offset` is not a
  property of the entry kind at all — it is `signature_offset + 0x40`.
- The 32-byte root chunk is a real **header**: `+0` magic, `+4` section
  count = 3, `+8` CRC, `+0xC` 0, `+0x10` header size = `entry_len −
  magic_offset` exactly in all five files, `+0x14` format = 0, tail zeros.
- The header chain's `0102`/`0103`/`0101` chunks are a **section table**,
  three 0x14-byte entries `{0, offset, crc, 0}` whose offsets are
  **relative to the signature start**: `0102 → 0x9C`, `0103 → 0x1B8`
  (constant in every entry), `0101 → a small region at the file tail`.
- `0103`'s target is the flat node directory: `sig + 0x1B8 = magic + 0x178`
  — the mysterious `376` is `0x1B8 − 0x40`. The directory's shape (count at
  +0, 12 skipped bytes, 12-byte `{tag, rel, children}` triples at +16) is
  exactly open-grn's "old-format chunk stream" (`{count@0, table@+0x10}`,
  advance = skip the declared subtree), derived independently by both
  projects.
- `0102`'s target is the byte after `0101`'s own 0x14 bytes: what our walk
  measured as "`0101` = 36" is table entry (20) plus the next section's
  prologue (16) — a count word = 6, then at `+0x10` the four 12-byte leaf
  chunks (`0200`, `1003`, `1001`, `1002`) plus two terminators: six table
  slots, verified on both kinds.

So the three "header chunks" were never three of a kind: the walker has
been stepping through header + section table + section 0's table region.
The measured strides (32/20/20/36, leaves 12) were correct because each
region it crosses really has that stride. `engine/formats/models.gd` needs
no change; this section records why its constants are what they are.

open-grn's claims that do NOT survive contact with the retail data: the
signature is not at file offset 0 (their `grn_dump.py` reports MISMATCH on
every real entry); the models.pak index entry is `{kind, offset, size}`,
not `{kind, 0, 0}`; and their export list of 105 functions including 4
Bink exports — the retail disc `granny.dll` has exactly 101, none Bink,
and `Sacred.exe` imports 54 Granny functions and no Bink at all (export
table length `0x65`, re-measured 2026-09-01). Their per-record layouts —
68-byte bone, 52-byte transform-track header with counts at +24/+28/+32,
the `+8` format flag distinguishing interleaved (12-byte header + 68-byte
frames) from split (times at +52) — agree with our decode and name the two
track variants we had measured but never cross-referenced. The rest of
their container tree is **verified on our bytes** (`tmp/opengrn_tags.py`,
2026-09-01): `0602` appears exactly once with only `0601` children; every
`0601` carries `0604`/`0901`/`0702`; every `0901` span is a multiple of 24
(6× int32 per face — what the reader already assumed); GLADIATOR's six
`0D00` materials expose their texture reference at a `0D03` sub-chunk's
`+4`, one-based, and the values are **[2, 6, 1, 3, 4, 5] — exactly the
permutation retail's own GL uploads confirmed**; GLADIATOR's `0B01` holds
68 `0B00` objects against 68 bones, and BATX's clip holds 42 whose `1204`
ChannelIds are precisely 1..42, each once — the "ChannelId is a 1-based
OBJECTS index" claim, bijectively. Both files carry the `0301`/`0304`
nodes their table calls the texture section. Our reader already reaches
the same names a different way — the DataExtension `__ObjectName` chain
(`object_names()`, `bone_names()`, `clip_bone_names()`) — so nothing here
changes the port; it is a second, independent derivation of the same
file. Still un-named by anyone: `0303`, `0305`, `0C04`, `0C05`, `0E03`,
`0E05`, `0E07` — their Python tool's enum proposes names for all but
`0C04` (texture-map image, texture-image section, FormPoseWeights,
render-pass field section/constant/section), none yet verified beyond the
structure checks above.

### Differential against their unpacker, 111/111 (2026-09-01)

`tmp/opengrn/` holds a one-shot cross-check: their MIT `grn_unpack.py`,
fed signature-anchored slices (entry bytes from `magic_offset − 0x40`) of
twelve mesh entries — all nine character-select heroes plus `BAT`,
`DUNKELELVE` and both elve-sorceress spellings — and three clips, against
the same entries read through `Sacred.Models`. **111/111 checks agree**:
bone counts, the exact bone-name lists in bone order, mesh counts, source
position and normal counts, triangle totals, every material→texture
permutation (GLADIATOR `[1,5,0,2,3,4]` on both sides), texture counts,
track counts, clip durations, and the ChannelId bijection. The only
representation difference: a material naming no texture is `null` in
their JSON, `-1` in ours. Track-name coverage: every non-helper track
name in all three clips is a bone name of the matched mesh — the residue
is authored `Cam_*.Target` / `Spot*.Target` / `Bip01 Footsteps`
cinematic helpers, the trap a naive name-set clip equality would trip
over; a helper-filtered coverage term is the shape that works.

**Corpus sweep, 2026-09-01.** The same differential in-process over every
walkable entry (`tmp/opengrn/corpus_sweep.py`, ~20 s): 4991 of 4993 (two
refused, both by the standing rule), their decoder zero failures, and
**zero divergences in any field** — bone counts, mesh counts,
source-position/normal counts and triangle totals (1571/1571 each), the
full material→texture tuples (1571/1571), texture counts (1571/1571), and
track counts (3420/3420). The two decoders agree on the whole corpus, not
just the sample; the third decoder the two-reader rule wanted already
existed.

**The sweep's one actionable find for the port: interpolation modes.**
Every ordinary `0xCA5E1204` track declares position and quaternion mode
**2 (quadratic)** — 255,461 of 255,461; scale-shear modes split
1:171,384 / 2:82,815 / 0:1,262; 3,013 sampled records carry none. The
records also carry three knot-time selectors at `+0x24/+0x28/+0x2C`.
~~`models.gd` reads none of these fields, so the port cannot be honoring
quad-authored keys.~~ **IMPLEMENTED 2026-09-01.** `models.gd` now decodes
the three modes and three selectors into every record (sampled records
declare none and are marked linear); `model_view.gd` honours mode 2 by
subdividing each key span into 8 linear keys — Godot Animation has no
custom basis. ~~The retarget is applied per generated key.~~ **Corrected
2026-09-07 (finding 1247):** local keys are bound directly. Mode 0 plays
nearest; unknown modes play linear and log.

Mode 2's SEMANTICS needed settling first, and the corpus settled it: it
is a **corner-cutting quadratic B-spline** — keys are control points, the
curve at each knot equals the time-weighted blend of that knot's two
neighbours (uniform case: their average), so the curve does not pass
through the keys at all. Two facts decide between the readings: 9,956
position records carry EVEN key counts, which a Bézier-triples reading
cannot parse, and all 255,461 records have strictly ascending key times,
which is all a B-spline needs. The vendor's pad structure (leading time
pads, trailing duplicates) pins curve(t0)=k0 and curve(tLast)=kLast —
~~This preserves anim_check's t=0-is-bind-pose property.~~ **Withdrawn
2026-09-07 (finding 1247):** endpoint interpolation does not imply agreement
with the model's bind pose. Historical verification on 2026-09-01:
anim_check and anim_freeze_check green,
verify_parity 38/38, start-scene gate **PASS** (world 11.09 vs best
11.00, inside TOL 0.10 and the port's own 0.60pp band noise — the gate
refuses a regression and resolves no improvement at this scene's
distance).

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
| 0 | record id (the 1-based transform-channel index) |
| 4 | unused |
| 8 | format flag: 0 interleaved, 1 split |
| 12 | position interpolation mode |
| 16 | quaternion interpolation mode |
| 20 | scale/shear interpolation mode |
| 24 | `numTranslates` |
| 28 | `numQuaternions` |
| 32 | `numUnknowns` |
| 36/40/44 | knot-time selector per channel |
| 48 | padding |

Modes: 0 copy, 1 linear, 2 quadratic, 3 cubic. Measured corpus-wide
(2026-09-01): position and quaternion are **2 on all 255,461 ordinary
tracks**, scale/shear splits 1:171,384 / 2:82,815 / 0:1,262, and mode 3
never occurs. Mode 2 is a corner-cutting quadratic B-spline — keys are
control points, the curve at each knot is the time-weighted blend of that
knot's two neighbours — implemented in `model_view.gd` by span
subdivision; see "The container, reconciled" above for the full argument
and the engine commit. The selectors distinguish the three knot-time
vectors; the times are stored per channel regardless, so a reader takes
the arrays and never chases the selector.

## A retracted conclusion worth keeping

> **`0xCA5E1204` is not "genuinely absent" from Sacred's motion data.** An
> earlier session concluded it was, having measured `models.pak` entry 2845 —
> which is `GLADIATOR.GRN` opened as a kind-65 entry: five meshes, 101 bones,
> and correctly *zero* per-bone animation records, because it is not a clip.
> There was never an absence to explain, only the wrong file compared against
> the right table. (Findings row 585; corrected by `grn_motion.py`.)

## Licence boundary

Three outside projects are cited, on identical terms — each is read **only
as documentation of the container format** — tag numbers, node-type names
and record layouts, which are facts about a file format:

- `github.com/SiENcE/Iris1`, `src/granny` — **GPL v2.** Documents the per-bone
  record layout and that clip length is the last translate time.
- `github.com/ptasev/Age-of-Mythology`, AoM Model Plugin — **no licence file
  at all**, which is all-rights-reserved, not permissive. Source of the
  authoritative `0xCA5E12__` node-type names.
- `github.com/karolak6612/open-grn` — **MIT.** A clean-room granny.dll 1.2b
  reimplementation whose container documentation anchored the 2026-09-01
  reconciliation below; every value was then measured on our own bytes. Its
  code may be read and its claims tested; none of it is adopted.

The first two projects' code was never opened, translated or adopted;
open-grn's was read and none of it adopted. The flat node directory shape
above is documented by neither of the first two and was measured here —
open-grn's parser later turned out to contain the same shape, so it is now
derived independently on both sides.

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

~~Two open items above are the obvious first questions for it: the eight of
3421 animation clips that do not decode, and `DUNKELELVE.GRN` and
`MAGICIAN.GRN`, the two of seven class body meshes that do not build.~~
**Correction, 2026-09-07 (finding 1235):** all seven class bodies build
since row 1009 (`Models._pair_by_reference`). The 2026-09-01 assertion that
the remaining clip decoding gaps were closed was premature: its count-zero
discriminator refused valid interleaved records with nonzero pose components.
Header `+8`, already documented above, selects interleaved (`0`) or split (`1`).
Using that flag restores 26 clips; 3415 decode, with the six refusals listed
below. Record sizes remain `12+68*N` and `52+16*nt+20*nq+40*nu`.

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

**2026-10-04, native class-6 startup:** item 6 is `VLADY_D.GRN`
(models entry 680); item 7 is `VLADY_N.GRN` (entry 689), and both creature
definitions are HERO-class records. The class-6 template remains the selected
hero data; type 7 does not become a selectable class. The native motion-1
references resolve to `VMPD_IDLE_BH.GRN` / `VMPN_IDLE_BH.GRN`, respectively.
The port's class map now includes the day body. Production `PlayerView`
construction, eight decoded skin batches and a textured Vulkan capture were
exercised; startup reports class-6 data at its authored `(3500,2477)`, layer 2,
with 129 derived HP. ~~The ordinary world capture still obscures her beneath
the storey surface.~~ **Follow-up:** the live actor is admitted to painter
phase `(3,1,0)`, and its ready capture contains her textured body in a
`(460,0,99,118)` screen rect with the head clipped at the top. Missing admission
is not the cause; a storey-occlusion explanation is not proven. This remains
body support, not proven in-world placement/parity.

The native form setter is independently present in LGP `81AC81E`, Gold ENG
`557040` and Gold RUS `557300`: it changes actor type 6/7, rebuilds the model,
preserves animation parameters and recomputes art regeneration. Recovering
that setter is not recovery or implementation of its activation, duration,
daylight and damage rules; those remain open.

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
(`0x83cbc0a`) probes to the first free slot and never overwrites.
~~Therefore the first entry in pak order wins (finding 897).~~ **Retracted
2026-09-21, finding 1258:** that conclusion omitted insertion order.
The constructor inserts headers **backwards**, so the **last matching texture
entry wins**. LGP `0x83C284E` / `0x83C2CAA`, ENG `0x656420` and RUS `0x656910`
agree. Model-name indexing is different: its forward insertion remains
first-wins. Do not apply one precedence policy to both asset families.

A fresh read-only native snapshot retains the texture-manager header vector,
hash buckets and hash seed table. Replaying lookup against those captured
buckets selects these entries, rejecting the earlier-entry control:

| Name | Earlier, wrong by name | Native selection |
|---|---:|---:|
| DRYADE_SCOUT_ARMS | 6868 | 6890 |
| DRYADE_SCOUT_HEAD | 6873 | 6895 |
| DRYADE_SCOUT_LEGS | 6874 | 6896 |
| DAEMONIA_1024 | 7168 | 7974 |
| DWARF | 7178 | 7986 |
| DWARF_WARHAMMER | 7180 | 8481 |

Five pairs have different decoded pixels. DWARF_WARHAMMER differs as a
compressed payload but decodes identically: it distinguishes lookup identity,
not appearance. CHEST02 still resolves to direct-ID control 7442; an absent
name misses. The snapshot reports zero concurrent changes, all six scene
vectors unchanged at consumption, and restored observation hooks. Its
untouched floor crop is byte-identical to a fresh no-snapshot control.

`TextureFormat._stems` now iterates backwards. The DWARF regression fails
before the fix (7178) and passes afterward (7986); the real Forward+ staged
viewer renders all three Dwarf surfaces textured. This verifies lookup and
render-path integration, **not a new native GPU-upload comparison** or
whole-scene visual parity.

Evidence: `donotpublish/tmp/global-order-20260907/asset-glue-20260921/`,
`asset-control-20260921/`, and `mapping-20260921/verify_mapping.py`.
Direct texture indices are unchanged. Numbered archive composition is mapped
in [install-inventory.md](install-inventory.md), but remains unwired in the port.

Final verification: **53 checks pass, zero fail**. The fixed start-scene
benchmark remains **5.05% world delta / MAE 1.85**, full **6.35% / 2.97**,
with **0.00% repeated-render difference**. This skin-selection fix does not
improve the Seraphim-only benchmark; no broader visual improvement is claimed.

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

## Full affine deformation and actor volume — 2026-09-21

Findings **1265–1266** correct two independent defects that scene-order checks
could not detect: the wolf's stretched geometry and the flat appearance of
otherwise coherent actors.

### Scale/shear is part of the pose, not optional metadata

`WOLF.GRN` entry 16 and its authored idle `WOLF_IDLE_BH.GRN` entry 1632 have
valid weight references and matching animation-channel identities. The defect
was downstream: `ModelView` applied positions and quaternions but discarded
the nine-component scale/shear tracks. `Skeleton3D.reset_bone_poses()` also
decomposes a sheared rest transform into TRS, losing part of the matrix.

Native local composition is **translation × rotation × full scale/shear**,
then parent composition. LGP `0x08074996` multiplies all nine scale/shear
components; `0x08074F9C` composes the parent. Eight actual wolf neck, head and
clavicle inputs were evaluated through the original native function in a
guarded running process. Full-matrix predictions agree within **1.683e-7**
per component; diagonal-only controls differ by **0.177–1.128**. Input/stack
canaries remain unchanged, floating-point state is restored, and the independent
floor-control capture matches the no-call arm exactly. This is a native local
transform oracle, not a claim to have captured a complete native wolf draw.

The installed Godot API independently discriminates the transport problem:
a synthetic sheared matrix loses **0.513743** basis error through either
rest reset or a weight-1 global-pose override. A direct RenderingServer skin
palette retains it with **zero** error.

The port now retains original affine rests, samples decoded SS modes 0/1/2
without decomposition, and uploads complete palettes after animation and
placement. Activation follows matrix/channel content, never the model name.
Worn meshes use their own binds; sockets, crop bounds and blob shadows consume
the same full global pose. Ordinary TRS rigs retain the engine path.
`affine_skin_check` protects the actual wolf's neck/head/clavicle transport;
GPU probes additionally verify animated matrices and inspect four phases each
of its authored idle, walk and run.

### Native projection and lighting replace the flat actor approximation

World actors now use the same recovered camera projection, model-header scale
and native facing transform as authored model objects. The calibrated
`133/73` body scale, front-on world basis and `HERO_LIGHT` flat ramp are removed
from production actor placement/materials. Root placement is not applied a
second time inside the skeleton.

The shared shader computes native directional Gouraud diffuse/specular light,
ambient **0.3 for category 3 / 0.8 otherwise**, shininess **38.4**, and encoded
texture modulation before linear output. Given native camera `C`, body
projection `Q` and mesh world transform `M`, its normal-space conversion is
`inverse_transpose(C * inverse(Q)) * MODEL_NORMAL_MATRIX * NORMAL`.
`NORMAL` is already skinned; applying skin deformation again would be wrong.
The formula also retains each socket mesh's own world transform.

Same-time native Seraphim/novice comparisons match **all 2054 / 2064 triangles**
through positions, UVs and three-corner topology, with no missing or ambiguous
faces and no normal-based correspondence selection. Normalized direct skin
normals differ by at most **0.000225 / 0.000209**; omitting normalization gives
errors up to **0.219 / 0.074**. Wrong Y-reflection controls average over 1.16.
Actual GPU post-skin/light-space probes agree within **0.00324**, measured
through RGB-half readback; that is not exact float32 equality.

Actor constructors now honor definition texture overrides as equipment and
rigid objects already did. Forced wolf type **588** selects texture **8185,
WOLF_VAMP01.TGA**: its red/black appearance is authored skin, not a red lighting
tint. Heroes, scripted NPCs and rolled creatures use the same binding rule.

Initial white-daylight inputs are supported. Solar RGB/pulse behavior,
general nonuniform-native-scale normal parity, exact animation scheduling and
whole-corpus pixel parity are not established by these fixtures.
All raw captures, oracle inputs, meshes and images remain private under
`donotpublish/tmp/actor-shape-20260921/` and
`donotpublish/tmp/global-order-20260907/wolf-affine-20260921/`.

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

`FX_E_IDLE_BH` and `FX_G_IDLE_BH` decode and merge their duplicate bone ids.
**Correction, 2026-09-07 (finding 1235):** the former claim that count dwords
`+24/+28/+32` distinguish sampled records is withdrawn. Those offsets are
translation/quaternion components in the interleaved layout. Zero-valued
components made the heuristic pass the FX examples while rejecting 2885
of 3013 interleaved records, including `HORS_DYING_A.GRN`.

The explicit **u32 format flag at +8** selects the decoder: `0` interleaved,
`1` split. A complete kind-65 record census finds 3013 interleaved and 255461
split records, no other flags, and zero size violations under the corresponding
formulas. Size congruence alone is not classification. The interleaved decoder
checks its complete size rather than flooring a partial sample count.

The production `--clip=HORS_DYING_A.GRN` command now returns 48 bones,
48 records, length 1.1 seconds, and 4896 channel keys. Existing hybrid
regression checks preserve the 24-to-12 duplicate-bone merge and the
ordinary wolf's variable scale/shear counts. The stale total-coverage assertion
was removed instead of repinned to a new corpus count; the checks exercise
the discriminating layouts. Evidence:
`donotpublish/tmp/shadow-contract-20260907/motion-format-census.json`.

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

