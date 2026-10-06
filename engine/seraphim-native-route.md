# Seraphim native opening route — 0.0.1 oracle

**Status:** Native reference route observed — one continuous new-game→acknowledgement→travel→fight→generated pickup/auto-equip→save run, fresh restart and continuation. Engine parity, aligned audio/animation/presentation and release qualification remain open.
**Purpose:** Freeze the native causality and identity used by the bounded 0.0.1 Seraphim route without substituting the port's quest-74 demonstration.

## Evidence boundary

Source-supported means the live LGP 1.0.02 reader/handler plus independent shipping Gold ENG/RUS code agree. Observed means the actual owned LGP retail process was driven and its presented frame and guarded memory read were retained privately. A file name, callback label, model render or passing engine repeat does not prove an executed quest consequence.

The measured live executable SHA256 is `8784ae11f876d1c7aeb2f9afeb4d721c8c6ac0c1788ac71b5d817095812528a4`. Raw scripts, captures, logs, memory JSON and saves remain in `donotpublish/tmp/m001-oracle/native/`; no dialogue prose or replacement data is distributed here. The original game install is input, not a fixture to package.

Canonical base content identity, measured with the existing engine
`formats/mod_manifest.gd` streamed full-byte contract, is schema1/
`whole-file-data-v1`, packages`[]`,
SHA256`623e3d9a44eaf823f4af0f6d1ab3989921d60acb9ee4b984d9c4bde45845de3e`.
It matches the earlier engine runtime profile; no second content-hash convention
or packaged retail fixture is introduced. Each private run also records launcher,
preload and settings identities. Native settings have SOUND1; configuration
alone is not audible/event parity proof.

## C0: actual opening identity

The selected base-campaign Seraphim template is `hero01.ptx`; StartPosition is cell3236,2511, layer1, sector50,39. The observed hero reference is1, type1, level1, XP pool0, HP119/119; held references `[0,18]`. Those values describe the sampled fresh session, not hardcoded future statistics or guaranteed random outcomes.

A guarded fresh baseline joins worn actor references17/type4007/family17 at
slot0 and18/type7901/family5 at slot13. Carried slots0/20/40 contain three
type5171 instances19/20/21, and slot60 contains instance22/type5633. Worn
items carry relationship44=hero1 with ownership tag0x400; carried items have
inventory_owner480=hero1. This contradicts the earlier conclusion that absence
of balance-text equip lines proves an entirely empty native starting loadout.
It does not rename type4007 from appearance or establish universal item rolls.

Quest1's `QIS_OnEnter1` is procedure11551, `funkcode.bin` offset1702864/length311. It creates handle `res:17095`, type679, script name `novizin1`; raw tag9 is `auftrag10`, followed by Goto and the compass. The observed novice is First Novice Leandra, reference694. Reference694 is a session observation; preserve the logical handle/type/role across independent runs rather than assuming a universal allocation address.

The actual opening includes native tutorial modals. Choosing the initial acknowledgement, rather than Skip Tutorial, is part of this route. A subsequent generic welcome/tip modal is distinct from the NPC's quest dialogue; dismissing it does not complete the quest.

## Tag9 is a native dialogue binding, not an inferred task name

The raw CreateNPC handler uses the tag9 string to look up an 80-byte dialogue definition by full name and case-insensitive equality. It writes the actor reference to definition+76, the definition index to unaligned actor+581 and the actor's dialogue flag0x80000. It is not sufficient to prefix an arbitrary task with `Dialog:` and declare success.

| Independent shipping build | CreateNPC handler | Relevant agreement |
|---|---|---|
| LGP live1.0.02 | `0x08283856` | Tag9 exact lookup `0x0825D466`; actor/back-reference binding and flag |
| Gold2.28 ENG | `0x00482510` | Same lookup and actor/back-reference binding |
| Gold2.28 RUS | `0x004828F0` | Same lookup and actor/back-reference binding |

The selected Seraphim definition is authored name `auftrag10`, definition ID-1, procedure11547. The procedure is exactly `Dialog:auftrag10`, offset1702362/length231, quest association1. Definition ID, runtime table index, procedure index and actor reference are separate spaces.

Actual C0/C2 samples show actor694/type679 ↔ runtime definition955/nameauftrag10 ↔ procedure11547, with the actor field and definition back-reference agreeing. The runtime contains additional same-name definitions without that actor reference; a substring search or unqualified last-match lookup is invalid evidence. `auftrag1001`, `auftrag1002` and `auftrag1003` are unrelated names.

## C2: native acknowledgement and journal transition

The actual retail dialogue called `Dialog:auftrag10`, then its acknowledgement called `exit01`, procedure11544 at1702124/70. The native log then names `QIS_Trigger9`, `QIS_OnEnter9` and `QIS_OnExit1` in that order.

The compiled callback performs NPCstate on the same handle to `auftrag09`, appends the tutorial journal entry, starts quest9 and ends quest1. Its EndQuest record is four bytes with **no arguments**. Native dispatch chooses the current procedure's quest association1; it does not end the newly started quest9 or require a fabricated explicit operand.

| Channel | Before NPC acknowledgement | After actual acknowledgement |
|---|---|---|
| Actor694 binding | definition955/auftrag10 | definition954/auftrag09 |
| Quest1 metadata flags |5 |3 |
| Quest1 journal status |2 |100(success) |
| Quest9 metadata flags |1 |5 |
| Quest9 journal | absent | status2, compass raw3201,2473 |
| Symbolic variable `09` |1 |3 |
| `auftrag09` definition state |8, unbound |13, bound to694 |

These are observed transitions. Metadata flags, symbolic bitmasks, definition state and journal status must not be collapsed into a single guessed “quest complete” boolean. Native finalization bit2 alone does not distinguish success from failure; journal100/101 does.

### Lifecycle source contracts

StartQuest raw20 resolves the quest and checks its enabled state, executes Trigger, sets runtime bit4, appends a 372-byte journal record and invokes OnEnter. The LGP lifecycle `0x0825E2F2` agrees with ENG`0x0046C160` and RUS`0x0046C540`. Finish chooses OnExit/OnLose, clears running bit4, sets finalized bit2 and writes journal100(success)/101(failure); LGP`0x0825FFE2`, ENG`0x0046CBE0`, RUS`0x0046CFC0` agree.

Quest9's OnEnter is procedure11539 at1701180/345. Its actual operand shapes include a two-integer compass, NPC movement/teleport, mode/media/effect operations and dialogue-state changes. An engine that refuses those operations has a real implementation gap; skipping them or coercing a zero-argument EndQuest into another quest is not faithful execution.

### Native branches are not interchangeable

`exit010` is a different branch, including calls to `tot1`/`extQuest09` and quest11 state changes. Skip Tutorial/native alternate choices are not proof that the ordinary six-robber objective was fought. This oracle route uses observed `exit01` and preserves the actual selected branch.

`extquest09` starts quest11 and ends the current quest9. Quest11's authored destination is the next command-post objective and uses handle`res:17085`; that branch is only qualified after its native trigger and dialogue transition are observed. Do not infer it merely from a decoded raw20 record.

## C1/C3: movement and authored encounter

The official [Sacred Gold manual](https://cdn.akamai.steamstatic.com/steam/apps/12320/manuals/manual.pdf), Controls pp14–16, distinguishes ground movement, held-left directional movement, enemy attack, NPC interaction, stair use and inventory right-click equip/use. Native held-left travel was observed crossing the chapel's layer1 onto outdoor layer0. The same-frame position, layer and animation are recorded, not guessed from camera motion or an unstable heap offset.

The first bank objective is **six authored CreateNPC records carrying exact callback `tot1`**, not quest74/monster107. In the selected StartCode these records begin at offsets551098,551138,551178,551218,551258,551298, using body313/308 and main-hand definition1712. Their creation tag5 is inserted into the script's callback vector as actor reference plus function name.

The observed callback table at script+856 has 68-byte rows; selected entries refer to actors695,697,699,701,703,705. Their current positions are dynamic. In sampled sessions they have level1/2 and HP50/58 (one sample42); these are actual state, not a universal foe constant. Native movement/aggro can change positions before the player reaches the bank.

The native bank approach produced a real hostile encounter: hero live HP
changed119→63 while the actual frame showed the registered robber attacking.
Another ordinary held-attack/potion run reached XP35, `09gruppe=1`, one removed
death callback and live HP92/119 before pausing. Other preserved fights reached
XP70/group2 and XP105/group3, but later died. These are native outcomes, not a
fixed per-robber award or a passed complete objective.

## C4 source contracts and observational boundary

### Live HP is not its eased display/cache

LGP's embedded combat block begins at actor+936. Actor+0x4c8/+0x4cc/+0x4d0
are cached/max/live HP. `sub_81F118A` advances cached toward live; the full
`sub_81FAC30` damage pipeline subtracts/clamps live HP. Native alive predicate
`sub_819AA32` rejects state252=9, state336=6 or zero live HP. Gold ENG/RUS
damage/alive implementations independently agree, at their own offsets.
Positive cached HP is not survival: preserved `[11,119,0]` and `[12,119,0]`
samples are dead, not alive at11/12. An early interpretation of the former
save as “alive11/119” is withdrawn.

### Callback, reward and generation are separate

Native death finalization removes the matching68-byte callback row before
calling its named procedure with that actor. `tot1` increments symbolic
`09gruppe`; it is not the XP calculation. Victim/beneficiary level, native
creature-row coefficients, difficulty/modifiers and eligibility produce the
reward event; its consumer adds to hero+0x3b4 and performs native level-up work.
Currency is a different pool at hero+0x3ee, not either inventory selection field.

Generic generation `sub_81B0BC0` applies an outer random budget test, conditional
category/quality/creature gates and a filtered definition picker that can return
no eligible item. A real kill/callback/XP change therefore need not yield useful
equipment. No forced drop, fixed item or guaranteed first-kill-per-type bonus is
admitted. The two generation caches compare **source actor reference+12**, not
type+16; ENG/RUS independently agree. Older first-type interpretations are
withdrawn. `cEngine::dropItem` places an existing instance and is not the entire
generic-generation path.

### Same-instance ownership and equipment

Definition, instance reference/address, inventory slot and worn slot are
different spaces. Native pickup may consume currency/quest items, carry the
same ref or auto-equip directly into an empty compatible slot. Class remapping
can change its definition ID during acquisition; do not require type IDs to
remain unchanged or assume newly allocated refs append monotonically.

Ordinary carried insertion preserves the ref and sets item+480 to the hero's
reference. Actual equipment is actor+420's19-reference array: hands are
slots12/13; slot18 is a linked creature/mount, not ordinary armour. Equip stores
the ref, sets item+44=hero_ref with tag0x400 and removes its world position.
Without tag0x400, nonzero+44 may be a world/static support relation.

Inventory entry+3216 is **five slots × two banks ×20-byte combat-action
descriptors**, not armour. Kind3/payload1071 was observed in the fresh baseline.
Entry+3416/+3418 are selected weapon-set/action-slot u16s; +4456 remains raw
merchant-context selection. All are separate from worn refs, XP and currency.

The observer's complete bounded item census retains source-proved cItemBase
instances, including pre-existing, destroyed and off-world entries; it does
not filter discovery to nearby/new refs. It records definition/family, raw
flags/position, tagged relationship/owner, carried/worn joins, inline130-byte
instance stats and bounded modifier rows without mutating native getters.
Overflow, unsupported/short reads or ref/root instability invalidate qualification.
Ground-candidate flags are not proof of world registration, pickup acceptance
or generation by a particular victim. Retained `tot1` identities survive
callback removal; original-set completeness is unknown until an actual baseline
establishes it.

The fresh baseline captured418 items across1553 manager slots, with six live
registered robbers and only two pre-existing ground candidates elsewhere.
That baseline can exclude pre-existing loot from a later temporal join.
Retained fight04's pickup-entry log names ref984/type7809, but the old sampler
did not capture its ground or ownership result. That entry alone is not C4.

### Observed generated-weapon lifecycle

`bank-census-03` completed14 guarded frame/state pairs. The full pre-fight
census excludes instance1526/type1729 from pre-existing item state. At120250,
121250 and122100 it is a qualified family5 world candidate at cell3253,2490,
with no actor ownership tag or inventory owner; the selected native death
and `tot1`/XP35/group1 transition occur in the same bounded encounter.
Ordinary `A` subsequently puts that same ref/address/type in carried slot100
with item+480=hero1, observed at126800. Generation cause relies on the native
source/death trace and complete temporal census, not a fabricated item or
a claim that proximity alone identifies a generator.

The ordinary save freezes liveHP65/119. The later150000 capture has live12
after combat resumed; it does **not** describe the state written while the
save menu paused simulation. A first follow-up equipped1526 only after
death; that run is a diagnostic, not a live C4/C5 pass.

In `loot-equip-05`, native Space/scancode65 consumes the remaining carried
potion and ordinary right-click equips the acquired weapon while the hero is
alive. A100ms-separated maximum-corner→origin→target cursor pin delivers the
real UI action; no memory write or forced equip is used. At21050, ref1526/
type1729 leaves carried slots, enters worn slot13, and has relationship44=1/
ownership tag0x400 with no inventory owner. The presented inventory shows
weapon33–37, attack27 and defence28. Subsequent native Save retains that
equipped ref while liveHP108/119, XP35/group1 and quest9 remain unchanged.

These recordings establish the same-item lifecycle and live equip/save
component. The intervening save/load is explicit; they are not edited into
a falsely continuous C0–C5 reel.


## C5 component witness, not the completed approved route

An ordinary right-click on starting carried instance22 swapped held sword18
for torch22; the frame showed its flame and the inventory reference joins
changed accordingly. Ordinary Save wrote the private `game01.pak`.
Fresh-process load of that owned save matched the compared HP119/cache/max/live,
XP50, position, carried/held references, quest metadata/journal, dialogue bindings
and five remaining death callbacks. Subsequent ordinary held walking changed
layer1→0 while XP50 and `09gruppe=1` remained unchanged.

This is real inventory/persistence/continuation proof for those compared fields,
not the generated-item C4–C5 chain. The source profile originated in a saved
dead-hero run; native loading recovered HP and returned her to the chapel.
The torch was starting inventory. Neither may silently replace the approved
continuous live fight→generated acquisition/equip→save→restart journey.

### Live acquired-weapon restart and continuation

`loot-restart-06` loads the `loot-equip-05` save in a fresh native process.
At19100 the hero is alive at108/119, XP35, cell3253,2490/support state0,
with the same acquired ref1526/type1729 equipped in slot13. Its inline
instance stat block and modifier rows match byte-for-byte; native process
addresses are not treated as durable identity. Quest/journal, dialogue
binding and five remaining death callbacks preserve the selected progress.

After ordinary Continue and held walking, cell3248,2521/support state0 is
observed at30800, liveHP100/119 with XP35 and `09gruppe=1` unchanged.
The intervening108→97 HP decline occurs after loading as the native foes
resume attacking; it is not an unexplained restore mismatch or ignored death.
The same equipped item/generated state survives that actual continuation.

The staged lifecycle is observed end-to-end but is not spliced into a reel.
The independently recorded continuous new-game reference below supplies the
whole-route witness that these earlier staged controls could not.

## Frozen C0–C5 native reference — 2026-10-06

The successful continuous reference is `continuous-native-08`, followed by
fresh processes `continuous-restart-09` and its duplicate-reload control.
The authored ordinary-input timeline is emitted by the tools repository's
existing `drive/menu.sh m001-new`; `m001-load` emits fresh load/Continue/walk.
`--shots` and `--duration` declare the required observation set. Neither route
forces loot, seeds randomness, writes actor state or guarantees checkpoint acceptance.
Historical `new` still skips the tutorial and is not this route.

| Role | Native witness in the continuous reference |
|---|---|
| C0 opening | `hero01.ptx`, cell3236,2511, support state1/static support758530, HP119/119, XP0; native tutorial acknowledgements, novice694/type679 bound to `auftrag10`/procedure11547. |
| C1 approach/exit | Ordinary NPC approach reaches3237,2516/state1; subsequent held exit reaches3234,2559/state0/support0, then the bank's50,38 sector. Screen requests and resulting cells remain separate spaces. |
| C2 acknowledgement | Actual `exit01`, not Skip Tutorial: quest1 flags3/journal100, quest9 flags5/journal2; novice rebinds `auftrag09`/procedure11536. |
| C3 encounter | Native registered foes701/703 die; `tot1` removals, XP75 and `09gruppe=2`. Original six are sampled before the fight; victim/state/stat identity is retained after callback removal. |
| C4 generated pickup/equip | New family6 armour1535/type6673 is grounded at3250,2480, absent from the pre-fight item census. OrdinaryA performs native direct auto-equip to worn slot1, tagged relationship=hero1, inventory owner0 and world position cleared. Resulting inventory/appearance shows defence31/resistance30; liveHP119 before ordinary Save. |
| C5 restart/continue | Fresh load restores cell3250,2480, HP119, XP75/group2, equipped1535/6673, generated stat/modifier bytes and four remaining callbacks. Ordinary Continue/walk reaches3248,2510, liveHP111, with the same item and no repeated reward. |

Those numeric refs/types/rewards describe this observed run, not fixed values to
inject or assumptions about every fresh game. In `continuous-native-07`, a real
kill reached XP50/group1 but produced no qualifying acquired/equipped item;
that capture-complete run fails C4. Natural variation must remain visible.
The successful route auto-equips armour, not the manually equipped weapon in
the staged control above. No missing loot or different owner may be substituted
to make a replay pass.

The continuous run supplies14 presented full1024×768 frame/state pairs; fresh
continuation supplies four more. Exact inputs, actual dispatch/drain/frame times,
item bytes, profiles and saves remain private. This is a recovered native oracle,
not a published retail-media reel or proof that the engine agrees with it.

## Observation-safe ownership

The observer reads fixed roots only after the launcher pins the exact executable; it does not call native accessors, several of which allocate or repair object references. Object-manager vector bounds, reference/type/embedded-ref identity and table roots are checked and revalidated around a sample. Actor+581 ↔ definition+76 is required for the selected dialogue witness.

State is sampled immediately before the screenshot on the same SwapBuffers presentation hook and includes requested time, actual time and frame. Input delivery has its own log; a matching requested millisecond does not alone prove a selected command was consumed.

`HOME` alone is **not** LGP profile isolation. Native userdata construction queries `getpwuid(geteuid())` first and only uses HOME on failure. The owned oracle redirects only that process's passwd directory under explicit `SACRED_ORACLE_HOME`, leaving game mechanics unchanged. Captures, logs and each run's profile are private. Inherited alternate-binary/runner selectors are removed; every requested native shot must have a valid/stable sidecar. Observer-required mode refuses a missing callback rather than quietly delivering image-only “evidence”.

Never use `SACRED_WATCH`'s first-anonymous-region offset as authoritative player position: it does not establish owning hero/reference/type and previously tracked a destination. Do not call getData/Instance() from a passive observer or reuse memory addresses across process restarts.

## Open

- Use the frozen native roles to qualify the actual engine journey and aligned full-frame/animation/audio chronology. Native-reference capture and strict engine-repeat instruments do not prove engine-to-retail equality.
- Preserve actual random generated state and refuse replay runs lacking a qualifying same-instance pickup/equip, live save, valid binding/profile or reward-preserving continuation. The no-item natural control must not become PASS.
- Physical height/support surface selection and scripted trigger details beyond captured cell/support identity remain separate movement/render qualification; a support ordinal is not height.
- Engine ordinary talk/pickup/equip/save UI, renderer/continuous travel and state-owner gaps remain separate milestone blockers; this research does not implement or waive them.

---
Provenance: selected retail bytecode via existing `tools/formats/startcode.py` and Vectoren layout; source joins in LGP/Gold ENG/RUS; private source/runtime controls named above, especially `continuous-native-07/08` and `continuous-restart-09/10`. Exact evidence remains in `donotpublish/tmp/m001-oracle/native/`. No raw retail writing, captures, item bytes or decompiler output is included.
