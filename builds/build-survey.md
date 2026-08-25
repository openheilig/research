# The build survey — which binary to ask which question

**Status:** Solved
**Purpose:** Which of the seven Sacred builds to open for a given question, and what
does and does not transfer between them.

Seven builds of Sacred were surveyed with the same battery. They are not
interchangeable: each lost or kept different things, and picking the wrong
one costs days.

**The reimplementation target is the *game*, not any one binary.** So the
method is: pick the most legible binary for the question, recover the
behaviour there, then confirm it against the binary that actually ships.

## The builds

| Build | Why you would open it |
|---|---|
| **`armalion_us.exe`** (2001) | **The best binary in the workspace.** Debug build, full MSVC RTTI, working Hex-Rays, and the combat traces intact. Default target for behaviour recovery. |
| `armalion.exe` (2001-09-20) | The original naming-oracle target: 127 vftables, 108 `cClass::method()` trace strings, 7388 functions. Superseded by `armalion_us.exe` for combat work. |
| `armalion_de.exe` | Solved by byte-diff against its sibling. No IDA session needed. |
| Armalion 2001-12-11 | **Wider but less precise** — 113 vs 99 methods, 55 vs 48 classes, 145 vs 127 vftables, 8514 vs ~8400 functions, but 6/8 vs 8/8 clean renames, **and it has lost the combat traces.** Not a replacement for `armalion_us.exe`. Its `_assert` source paths are its own contribution. |
| USA demo (`Sacred.exe`) | **Richer RTTI than armalion** — 202 vftables, 271 type descriptors — and broader subsystem coverage: items, inventory, object manager, Granny, textures, DX7, sound, movies, script compiler, full UI. **No thunk table**, so call resolution is one hop shorter. Debug-string oracle transfers but is noisier. |
| Gold 2.28 retail (2006-10-13) | What the rules layer actually looks like when shipped: tunables moved out of code into `balance.bin`, 380 named keys, the Excel exporter. |
| LGP Linux `sacred` 1.0.02 (2010) | **The binary this project runs.** Stripped, but Itanium RTTI intact — 318 classes recovered. Confirm here; recover elsewhere. |

Two builds were closed on cheap external measurement without an IDA session
at all — Gold 2.28 Russian retail among them.

## A third debug build, from its `DEBUG.LOG` alone, 2026-08-25

A `DEBUG.LOG` recovered from VK is a **fourth data point and a third build** —
addon-era retail lineage, not Armalion. It identifies itself by content, not by
a banner: it plays `SOUND_FX_MENU_ADDON` and loads `ITEMS03/MODELS03/TEXTURE03`,
the Krombacher promo paks that only the shipped game carries.

Every Sacred debug build opens by printing its own class sizes. Three builds
side by side, and the split is the point:

| class | 2001-09-20 | 2001-12-11 | VK log (addon-era) |
|---|---|---|---|
| `cSectorChunk` | 512 | 512 | **512** |
| `sObjectStatic` | 64 | 64 | **64** |
| `cTrigger` | 16 | 16 | **16** |
| `cPatchIso` | 32 | 32 | **32** |
| `cPatchSharedIso` | 64 | 64 | **64** |
| `cGrnMtnChunk` | 256 | 256 | **256** |
| `cSoundChunk` | 128 | 128 | **128** |
| `cTimerListener` | 12 | 12 | **12** |
| `cSectorEnvironment` | 256 | 256 | **256** |
| `granny_transform_state` | 220 | 220 | **220** |
| `cWorld` | 132448 | 132708 | 56936 |
| `cSector` | 384 | 384 | 1452 |
| `cObjectShared` | 384 | 256 | 128 |
| `cGrnMdlChunk` | 1200 | 1200 | 1194 |
| `cEvent_creature` | 52 | 52 | 68 |
| `cCritical` | — | — | 24 |
| `sObjectNonstatic` / `…3D` | 53 / 86 | 53 / 86 | absent |

**The classes that own an on-disk record never move; the runtime classes move a
lot.** `sObjectStatic` is 64 bytes across five years and three builds, and so
are `cTrigger`, `cPatchIso`, `cPatchSharedIso` and `cSectorChunk` — an
independent third confirmation of the freeze
[`pak-containers.md`](../formats/pak-containers.md) argues from the file data.
Meanwhile `cWorld` more than halved, `cSector` nearly quadrupled and
`cObjectShared` fell by two thirds. A size in this table is evidence about a
FORMAT only for the classes in the frozen group; for the rest it is evidence
about a build.

**A fourth build, from Sacred Plus, 2026-08-25.** Its `DEBUG.LOG` prints a
fourth distinct table: `cWorld` **1023592**, `cMutex` **28** where the VK log
has `cCritical` 24, and an `Evaluierung 124124` line no other build prints. Its
`cSector` 1452, `cObjectShared` 128, `cGrnMdlChunk` 1194 and `cEvent_creature`
68 are **identical to the VK addon-era log**, so the two are close siblings.
Every member of the frozen group above is unchanged again — `sObjectStatic` 64,
`cTrigger` 16, `cPatchIso` 32, `cPatchSharedIso` 64, `cSectorChunk` 512,
`cGrnMtnChunk` 256, `cSoundChunk` 128, `cTimerListener` 12,
`cSectorEnvironment` 256, `granny_transform_state` 220 — now across **four**
builds and roughly five years. `cWorld` has taken four different values.

`cCritical` (24) appears in no earlier build, and `sObjectNonstatic` /
`sObjectNonstatic3D` are printed by both Armalion builds and by neither the VK
log — the print list itself changed, so absence here is not absence from the
engine.

The same log independently confirms seven container headers we had read from
the files themselves — `texture.pak` v3/25535, `texture03` 3, `items.pak`
v5/32768, `items03` 32768, `models.pak` v3/4993, `models03` 4, and
`sndprofiles.pak` v1/8192. Seven for seven. The log misspells that last one
`SNDPORFILES.PAK`; the file on disk is `sndprofiles.pak`, and the typo is
retail's, not ours.

## The rule that follows

Recover on a debug build, **confirm on the shipping one**. The to-hit formula
is the worked example: recovered from `armalion.exe`, independently confirmed
in `armalion_us.exe`, and then found *absent* from retail — which is how we
learned retail had moved its tunables into a data file. See
[../engine/combat-formulas.md](../engine/combat-formulas.md).

## What transfers, and what doesn't

**Transfers:** the naming oracle (with build-specific precision), class
hierarchies, formula shapes, the data files themselves — 8/8 shipped `.bin`
files are byte-identical between Windows and Linux, and 380/380 balance keys
are present in the Linux binary.

**Does not transfer:** the thunk-table hop (`/INCREMENTAL` builds only), the
hardcoded to-hit clamp (absent from retail), the Excel balance exporter
(Windows-only).

## Data in this directory

- `armalion-script-api.tsv` — script API surface extracted from the Armalion
  debug build.
- `gold-linux-resource-comparison.tsv` — resource deltas between builds.

## Open

Nothing open. The survey answers which binary to open; it makes no claim
about any finding recovered from one.

---
Provenance: seven IDA sessions run in this project's private analysis
workspace, same battery each time.

