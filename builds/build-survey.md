# The build survey — which binary to ask which question

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

---
Provenance: seven IDA sessions run in this project's private analysis
workspace, same battery each time.

