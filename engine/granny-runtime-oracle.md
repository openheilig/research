# Driving RAD's Granny runtime as an oracle

**Status:** Native1 exported API exercised2026-10-12; historical Granny2
converter remains blocked.
**Purpose:** Record the failed conversion route separately from bounded native1
control creation, query and installation observations. Neither is full pose/
render parity or semantic qualification of the entire proprietary runtime.

The idea was to avoid arguing about our `.GRN` decode by asking RAD's own code
what a file contains. The harness only **calls exported functions and reads
their output** — nothing decompiled, nothing derived from RAD's SDK source.
The binaries it drives come from a community archive and live in a scratch
directory; none is stored in any repository here. The harness itself is
[`tools/granny_oracle/`](https://github.com/openheilig/tools/tree/main/granny_oracle/).

## Native1 control observation — 2026-10-12

The native DLL's SHA256 is
`9ac73f4aa5cc5f0096bac3bc79b299bcae0e839d08a470fc14b924142c643d68`
and MD5`c1ffdac411b506382f2acb7550b5ccf4`, matching the recovered native1
identity. A private freestanding PE calls exports only; no DLL implementation
was disassembled/decompiled. Body entry661 and idle entry3039 are extracted
through the existing identity-checked reader, then opened from their documented
64-byte native signature starts. No converter or redistributed asset is used.

Legitimate game initialization arguments, model/animation/sequence opening and
valid control creation return0. Invalid sequence/animation creation return6/10
and preserve all three sentinel words; invalid control/parameter queries return
6/7 and preserve the double sentinel. All exercised removal/close calls return0.
One initialized private WOW64 prefix is required on this host: the first fresh
win64 prefix timed out during initialization and then lacked32-bit kernel32.
Those failures are retained, not mislabeled native API refusals. Global Wine
configuration and existing user profiles are untouched.

For this control, parameter0/1 are observed timeline endpoints; parameter2
changes rate and inverse lifetime, parameter3 doubles lifetime when changed
1→2. Base lifetime1.3799999952316284 becomes1.1039999961853026 at rate1.25.
Expiry is observed just above the end; the attempted exact-end sample
overshoots by2.9802322831784522e-9, so exact-equality expiry is not qualified.
Parameters4 and5/6/7 accept set/get, but getters alone do not prove dynamic fade.

`GrannyGetBoneIndexFromName` resolves`Bip01 L UpperArm` to10. Its transform
output after valid control creation differs from the no-control bind output.
Pointer/handle bytes are excluded from that comparison. Advancing time alone
leaves the next sampled transform unchanged: this bounds control installation
effect, not per-frame evaluation, fade completion or finished idle pose.
Private evidence:`donotpublish/plans/workflow/animation-qualification/`.

## Clock is not evaluated pose — 2026-10-12

**Correction to the control-only probe above:** its first unchanged bone sample
does not establish a frozen animation runtime. The sampled game-owned
presentation path includes a zero-time clock-wrapper call before rendering:
Linux `8103778 → 80F7664(context,0.0) → 80FC268`; Gold RUS `40A7D0` and ENG
`40A690` call `401920(context,0.0)` before their corresponding renderer.
Exact source-byte joins verify those callsites without entering RAD internals.

Locking for rendering, enumerating all six states, and unlocking leave the
selected pre-lock bone output unchanged. Exported `GrannyAdvanceTime(context,
0.0)` instead changes bone output at an unchanged queried clock. All72 bones
are observed, not just the upper arm; default and full-weight sequences are
stable on a second zero-time refresh. The clock and current evaluated output
must therefore be recorded separately. Exact internal update ordering is not
inspected or inferred as a library defect.

An incoming single-control envelope `5=0/6=0.2/7=1` differs from constant-full
`5=1/6=0/7=1` at time0, then matches its output exactly at sampled0.25/0.5/0.75.
Parameter4 getter stays1. At0.125 the incoming output changes on a repeated
same-clock refresh, unlike full/default controls. This is an observed feedback
limit of the isolated incoming-control setup, not proof of the game's actual
two-control fade formula or handoff completion.

The combined gate verifies4,320 bone outputs and66 rendering-state consumptions,
1,965 Linux/1,797+1,797 Gold instruction encodings; exit0 in0.12s. This qualifies
the bounded source/API observation, not a GPU draw, rendered idle, whole corpus
or release. Evidence remains private under the directory cited above.

## Paired-control exported handoff — 2026-10-12

Selected Seraphim walk3050 and idle3039 run on six same-clock sequences:
independent outgoing/incoming references, normal and early paired promotion,
failed incoming creation, and the isolated incoming control. The outgoing
fixture is52560 bytes, MD5`c45cfe1158a06aed30718b87cf24a21d`; its native signature
starts at256. Extraction reuses the existing checked reader. No fixture or DLL
ships with the harness, and no proprietary implementation is inspected.

Source ordering is pending creation, outgoing5=1/6=duration/7=0/rate, incoming
3=caller count/5=0/6=duration/7=1/rate. The observed duration is the source's
promoted float upper clamp0.20000000298023224, with chosen API rate1 and loop
count10000; these are probe inputs, not a newly measured retail GameSpeed.
Promotion removes outgoing, sets incoming4=1, then the game copies pending
tuple/ID and clears its24-byte pending region. The restore helper itself is
nonzero-deadline gated, not an expiry test; another request may promote early.

At the observed exact transition deadline, all72 normal paired outputs equal
the independent incoming reference before/after promotion. Early promotion
at the midpoint does not produce the full incoming pose. Failed incoming
creation returns10 and preserves its triple and the outgoing-reference output.
Removed-control queries return6 and preserve the scalar sentinel.

Repeated paired midpoint output is not bitwise identical: maximum position
delta is3.814697265625e-6 and matrix-component delta1.4901161193847656e-8.
The isolated incoming midpoint position delta is0.8298524618148804; removing
outgoing early again produces substantial feedback. Preserve those bounds
rather than treating base getter4=1 as full envelope weight or pose completion.

The gate joins exact8,625 Linux/6,103+6,103 Gold instructions,30 actual-byte
promotion guards with a refused mutation, and6,048 native bone observations;
exit0 in0.20s. Ordered finite deadline behavior agrees in the exercised paths,
while raw unordered lower-gate decisions differ across builds. These source/API
witnesses do not execute a canonical retail actor, prove full renderer equality
or waive remaining source blockers. Evidence remains in the private directory.

## Real retail actor witness — 2026-10-12

The owned LGP process now supplies a getter-instrumented, same-frame actor
witness, not just a standalone native1 DLL experiment. All24 canonical native13
inputs and22 screenshot requests are preserved. A private callback on the
existing presented-frame counter records actor1/type1, clock, clip identity,
active/pending parameters and a named bone output. No setters, forced evaluation,
model loads or SDK implementation inspection are invoked.

Native14 contains171 complete actor frames and105 first-window samples with
motion0/no active control, explicitly unqualified. The dialogue-return handoff
is `SERA_TALK_A.GRN`3049 → `SERA_IDLE_BH.GRN`3039, rate1.25 on both controls,
duration0.15000000596046448. Pending frame3602/clock56.68900012969971 precedes
deadline56.70300009846687; promoted frame3603/clock56.706000089645386 follows.
All three pending words become active, and pending selector/deadline clear.

After release, a second handoff keeps enum2/idle filename but installs a new
control tuple at frames5087→5088. Its deadline is81.94800010323524, bracketed
by clocks81.93400013446808/81.95100009441376. Both transitions reach logical
state0 before the tuple transfer. A motion enum or logical idle cannot replace
the native handle/deadline/evaluation observations.

Actual control start equals the previous presented frame's clock, and start
plus duration equals the actor deadline exactly. This is an observed frame
bracket, not the exact internal promotion call time. Observed GameSpeed1.25 and
actor multiplier1.0 are this profile's witnesses, not universal constants.
Native getters can do cache housekeeping; unchanged clock/actor bytes within
each observation do not prove all later state is unaffected.

Owned runtime exits0 in97.248s;24-input/22-frame/171-complete-frame gate exits0
in0.13s and rejects corrupt handles and premature promotion clocks. Initial
105 samples, first C1 locomotion, engine/GPU/audio equality and72 broader source
blockers remain open. Private evidence:
`donotpublish/plans/workflow/actor-transition-qualification/`.

## Inactive startup is not a getter prerequisite — 2026-10-12

Raw-only native15 preserves the same input driver code,24 canonical inputs and
22 screenshots, with no added game/SDK calls or runtime patch. All1,673 passive
actor frames are identity-bound and unchanged across observation. State7/2,
motion0 and zero active controls occur from39701ms through captured50500ms.
This refutes getters as a necessary cause of that pattern; it does not establish
all future-state noninterference or explain the active/inactive cross-run difference.

The actual primary-state table dispatches7 to game-owned82EF858, not ordinary
idle8195B34. Linux and independent Gold state7 substates1/2 do not initialize
animation; actual no-event case2 bytes write338=6 before returning. Six absent/
present-control cases and copied-machine mutations qualify that narrow behavior.
The initial absence can persist along that state path, but continued7/2 after
normal completion requires missing dispatch, a later writer or another unobserved
execution boundary. The raw trace alone does not choose among them.

Actor thinking accumulator520 stays0.09899955987930298 against interval516
0.30000001192092896 while update stamps advance. Later7/6 and active talk/idle
are observed. Do not insert an idle request to hide the remaining scheduling/
rewrite question. The source/raw gate exits0 in0.15s; owned retail exit0 in97.219s.
Evidence:`donotpublish/plans/workflow/startup-qualification/`. No full startup,
pose/GPU/engine equality or broader source admission is claimed.

## Actual startup dispatch boundary — 2026-10-12

Hardware-only native17 observes the event0x10c writer that sets7/2, then the
missing state7-call interval and a later normally completed2→6. All268 hero
calls pair with268 returns; there are0 capture errors. Writer817CC05 is observed
at next PC817CC0E after presented39668ms; the later call is after presented67709ms.
Debugger stops perturb timing, so these are not ordinary trajectory or exact
function-call timestamp claims.

The selected scheduler call is skipped when the game-owned clock object's byte0
is nonzero. Exact Linux/Gold gates and12 branch-reversal controls qualify this
predicate; Gold uses virtual+4 where Linux uses+8. Passive native18/19 provide
1,787/1,786 valid clock/raw pairs with0 invalid samples, but both take the active
startup path. Their nonzero-byte7/1 intervals freeze the thinking accumulator
while global time and update stamps advance. This narrows admission evidence;
it does not observe the gate value during native17's earlier stuck7/2 interval.
No recovered field name, forced animation, complete C1 or game fix is asserted.
Evidence:`donotpublish/plans/workflow/startup-dispatch-qualification/`.
All72 broader source blockers remain. Next: inactive-run timer admission state.
Final source/runtime gate exits0 in1.04s;30,307 Linux/491+491 Gold instruction
joins and six corrupted-evidence negatives pass. Finding1480 retains the cause gap.

## Retail timer cause observed — 2026-10-12

Complete passive timer capture now reproduces the inactive7/2 Seraphim path
without getters, debugger or time/input override. Native21 selects a natural
calendar phase; native20 is the active control. Each provides2,867 valid samples;
3/2 undrained-input samples remain refused. All22 canonical captures complete.

The selected game timer is registered, not retired, priority-eligible and outside
the clock gate. Float32 pause-baseline loss shifts last-update and deadline
31.9689998626709s for clock references8.057999968528748s apart, overcompensating
23.91099989414215s. The actual old/new record differences exactly equal
float32(572530847.969−572530816.0). Count149 remains across1,410 inactive frames;
count150/state7/6 follows deadline catch-up at63703ms.

Source83D03FC/83D03D0/83CF78C binds the clock producer, returned float duration
and timer adjustment. This is a retail Linux oracle-clock cause, not an SDK
animation wait or an implemented engine fix. Fifteen copied-byte controls and
source/capture/causal gates pass; clock-baseline/resume-reference mutations are
refused. Full/causal gates exit0 in0.35s/0.10s. Evidence:
`donotpublish/plans/workflow/timer-admission-qualification/causal-proof.json`.
Older unsampled intervals, full C1, engine/GPU equality and all72 broader blockers
remain unqualified. Next: qualify oracle timing before more animation changes.

## Explicitly corrected oracle clock — 2026-10-12

Owned native22 uses a private signature-guarded in-memory replacement of the
game-owned pause setter/cache getter. A retained double baseline fixes subtraction
precision while preserving epoch, object layout and float cached-return/reset
behavior, including resets while paused. No SDK or animation is patched; original
files remain unchanged. This is labelled corrected, not unmodified-retail evidence.

The original-byte regression is red; the32-bit authored core passes the observed
precision contract and repeated/reset/negative-duration cases. At the same natural
calendar phase formerly producing the stall,24 inputs/22 captures complete with
exit0 in97.302s.2,868 valid timer frames and one retained undrained-input refusal;
zero inactive7/2 frames. Maximum eligible deadline lead is0.044999957s; natural
idle and actual walk are observed. Verification exit0 in0.29s; old32s-delta
mutation refused. Evidence:`donotpublish/plans/workflow/clock-correction-qualification/`.
Selected single-clock route only; no complete engine/GPU/release admission.
Earlier raw/native captures retain their uncorrected identities and all72 broader
source blockers remain. Use corrected oracle context explicitly for the next C1.

## Two corrections worth keeping

> ~~`granny2.dll` is 2.5.0.11, and 2.5 cannot read Granny 1.x `.GRN`.~~ The
> DLL in use is **2.7.0.30**, renamed from `granny2_27.dll`. Version was never
> the obstacle.

> ~~The GRN reader lives in `gr2_viewer.exe`, and calling its exports faults
> because the EXE's CRT and C++ static initializers never run when it is
> mapped as a library, leaving a module global uninitialized.~~ The converter
> is `grn2gr2.dll` itself, whose export resolves and can be called directly —
> no dialog-driving and no EXE-as-library workaround. The fault was real; the
> cause was not. It is `grn2gr2.dll` null-dereferencing partway through
> parsing a genuine Sacred Gold object, as below.

## The crash

Every real-file invocation faults at one instruction:

```
wine: Unhandled page fault on read access to 00000000 at address 100061BF
0x100061bf grn2gr2+0x61bf: cmpl $0xca5e0103, (%eax)
EAX:00000000 EBX:0022fe00 ECX:00000000 EDX:00000000
```

The instruction compares against the object tag `0xCA5E0103` with `EAX = 0` —
the converter dereferences a pointer to a structure it expected to already
hold.

## What was ruled out, in order

Inputs were identity-checked before use — on-disk size plus md5, output
deleted on any mismatch — because an earlier scratch probe had reported a
false success that was really a stale file from the day before, silently
picked up after the converter no-op'd. The same class of error the log records
at rows 581 and 584.

| Hypothesis | Test | Verdict |
|---|---|---|
| A leading pak preamble confuses the parser | Full 186,842-byte pak entry vs a `--bare` slice from the `0xCA5E0000` root tag | **Refuted** — identical fault at the same address |
| The converter needs the model **paired** with its motion, as the community documentation states | Entries 589 (base) and 2845 (motion) in all four orders × bool values | **Refuted** — all 4 fault identically |
| Buffer shape — padding or declared-length truncation | Declared object length at `+0x10` is 185,584, exactly the slice size, already 16-byte aligned | **Not applicable** — nothing to vary |

**The control that makes this a result:** the same call against two
nonexistent paths returns cleanly, without crashing. So the fault is triggered
by real `.GRN` content being parsed, not by an unconditional setup error — yet
its location is completely insensitive to which file, in which order, with
which flag. Six real-file invocations, one instruction.

## Verdict

The most consistent reading is that this converter's parser does not handle a
structural feature of Sacred Gold's `.GRN` dialect at all — not a pairing,
addressing, or framing problem. **This build cannot be driven past that point
regardless of how it is called**, so no `.gr2` was ever produced and the
oracle yielded no skeleton dump and no vertex-count comparison.

The decode in [../formats/granny-grn.md](../formats/granny-grn.md) therefore
rests on the harness and parity route instead, which reached 3413 of 3421
clips without this oracle.

## Open

The historical converter question is closed. The head-mesh discrepancy
(279 versus280) it was built to settle was **closed on2026-08-15 elsewhere**:
an `apitrace` capture of the retail character-select screen contains the
engine's own `glDrawElements` index array for that batch, and it resolves to
279 distinct vertices. See
[../formats/granny-grn.md](../formats/granny-grn.md).

The lesson is worth more than the answer. This harness spent three hypotheses
trying to make RAD's converter read a Sacred `.GRN`; the question fell out of
watching the shipped game draw the mesh, which needed no converter, no SDK and
no clean-room risk. Reach for what the game *does* before what a vendor tool
*might*.

Native1 open work: explain the motion0/no-active-control startup window before
qualifying first C1 locomotion. Internal promotion call timing, complete pose/
curve equality, future-state observer noninterference, other assets/error classes,
animation-expiry equality, GPU equality and full runtime/corpus correctness remain
outside the bounded actor witness.


---
Provenance: `tools/granny_oracle/granny_dump_skeleton.c` and `grn_extract.py`,
whose run modes produced every row above; findings log rows 581 and 584 for
the identity-check discipline this investigation adopted.
