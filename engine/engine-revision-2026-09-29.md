# OpenHeilig engine revision — 2026-09-29

**Status:** Research audit and proposed product contract; no engine implementation authorized or changed.
**Scope:** Sacred Gold retail capability, current production paths, playable and strict parity, performance, resource/mod architecture, and release readiness.
**Implementation proposal:** [Detailed implementation plan](https://github.com/openheilig/engine/blob/main/docs/implementation-plan-2026-09-29.md).

**Historical evidence boundary:** this audit records September29 observations,
not the current engine's feature/proof status. Later work and the approved
[0.0.1 Seraphim scope](https://github.com/openheilig/engine/blob/main/docs/milestone-0.0.1.md)
are recorded separately; repository publication is not release qualification.

## 1. Conclusion

Keep the independently written retail readers, recovered rendering rules, actor registry, simulation/view boundary, and recent progressive publication work. Do not restart in another engine. Change the unit of progress from “a decoder/function is mapped” or “the chapel score improved” to **a player journey that runs, survives a restart, and agrees with retail state**.

The current product is a sophisticated world viewer with movement, partial world state and a batch encounter. It is not a general Sacred gameplay engine. Main blockers are world admission/navigation, persistent script scheduling and identity, authoritative actor/item state, interactive combat and services, and durable saves. Renderer accuracy remains important, but completing the cathedral image is not a prerequisite for implementing these systems.

The knowledge base also contains false closures. This audit found an Armalion navigation rule transferred to Gold, a sector loader attributed to creature HP, and learning-point arithmetic attributed to an XP-gauge denominator. “All binaries decompiled” is useful infrastructure, not evidence that all mechanics are understood. Neither a source-file count nor a pixel-delta percentage is a game-completion metric.

## 2. Evidence vocabulary and limits

- **Observed now:** executed during this revision; command/output or private capture retained.
- **Source-supported:** inspected current authored source or decompilation; runtime reach/behavior is not automatically proved.
- **Prior measurement:** an identified findings-log result, not rerun or rebranded as new evidence.
- **Proposal:** a product/architecture/acceptance decision, not recovered retail behavior.
- **Open:** an exact contract not sufficiently established to implement faithfully.

Paths beginning `godot-port/` refer to the engine repository via the private workspace symlink. `analysis/` and `tmp/` references are private evidence, not distributable artifacts. Findings are cited by **id**, not physical TSV line number. No permanent tests or engine source edits were made. Four read-only research audits covered gameplay, retail scope, rendering/performance, and mods/platforms; the main audit ran the actual engines and inspected IDA.

The semantic find service failed with provider HTTP402 and graft returned an empty graph. Directory inventories, literal searches, current source, existing corpus, IDA MCP, official documentation and runtime observations supplied the evidence instead. An unavailable search is not negative evidence.

## 3. What the retail game actually includes

### Product and roster

Sacred Gold contains Sacred/Plus and Underworld. There are **eight** playable classes: Seraphim, Gladiator, Battle Mage, Dark Elf, Wood Elf, Vampiress, Dwarf and Daemon. Vampiress belongs to the original six; Dwarf and Daemon are the expansion additions. The engine's seven-entry `CLASS_MODEL` omits Vampiress, and its adjacent comment misclassifies her as an Underworld class (`main.gd:66-93`). Rendering seven bodies is not complete class support.

Eight level-1 `templates/hero00.ptx`–`hero07.ptx` are distinct from the shipped advanced Dwarf/Daemon hero exports. The latter are documented as level29 in `pax-saves.md:149-184`; an older launch log says20 without identifying the supplied file. Official manuals specify a level25 Underworld admission threshold. Do not merge these three statements into an invented rule: validate template creation, supplied exports and admission separately.

### Required retail capability inventory

| Domain | Player-visible retail contract |
|---|---|
| Boot/session | Main menu, class/name/difficulty selection, template creation, hero import, new/load/quit, both campaigns and endings |
| Exploration | Click/held-direction movement, walk modifier, stairs/layers, doors/containers, interiors/roof cutaway, dungeons, portals and scripted transitions |
| World lifecycle | Sector/region initialization and entry/exit, authored NPCs/objects, encounters, persistent triggers, respawn, day/night/weather |
| Combat | Melee/ranged/projectile/area attacks, targeting/range/LOS/timing, hit/block/damage, four channels, resistances, status effects, death and rewards |
| Classes | All eight starts and skills; Vampiress forms/day effects; Daemon forms; Dwarf guns/cannon/mines; summons, companions, traps and teleports where authored |
| Progression | Six attributes, skills distinct from combat arts, rune learning, recovery-time clocks, combos, XP/level/point allocation, equipment effects and difficulty progression |
| Items | Instance ownership, pickup/drop/use/equip, full-inventory behavior, restrictions, rarity/set/modifier rolls, potion actions, stash and persistence |
| Services | Trade, prices/gold, blacksmith sockets/upgrades, Combo Master, rune exchange, horse vendors/mounting and class eligibility |
| Quests/dialogue | Choices/conditions, accept/decline, objectives, task and quest transitions, rewards, escort/followers, journal and compass |
| Information/UI | Portrait/health, XP, arts/weapons/potions, enemy stats, companions, inventory/skills/bonuses, maps/fog/markers, tooltips/cursors, tutorials/help |
| Media | Contextual music/ambience/footsteps/combat/speech, independent volume/mute, intros/outros/act movies, credits, skip/return behavior |
| Persistence | Full game save/load/autosave; hero export/import into a new campaign is a different operation; compatibility varies by save version |
| Settings/locales | Input/options persistence, detail and pickup settings, installed text/voice languages, fonts, non-ASCII character names |
| Multiplayer | Historically LAN/Open/Closed service, co-op, H&S/PvP, parties/chat, hosting and service-specific character/save/death rules |

Primary sources: private `docs/manuals/Gold/gold_manual.pdf`, printed pp.5–24 and25–35; `docs/manuals/Underworld/SU_Manual_Screen_English.pdf`, pp.4–14; `docs/manuals/Gold/Linux/manual.pdf`; official `docs/changelogs/English/Readme 2.28/Readme.html:95-178,397-410`. The later readme overrides older manual details such as `DEFAULT_SKILLS : 0`. [Current GOG product description](https://www.gog.com/en/game/sacred_gold) corroborates the roster/content and states that original internet servers no longer function, while physical/emulated LAN remains possible.

Do not introduce a generic mana pool: Sacred's arts/spells use regeneration/recovery time. Do not confuse `UI_MP` multiplayer portraits with mana. Do not apply multiplayer level bands to single-player difficulty unlocks. Do not assume every class can ride merely because a horse UI exists.

## 4. Current production capability matrix

| Capability | Current evidence | Missing contract |
|---|---|---|
| Retail file access | Real runtime binary readers, no bundled game data; world6050 sectors observed | Build/edition/locale manifests, casing, complete validation, numbered families |
| World rendering | Native FIFO-based canvas composition, cropped model targets, terrain/statics, separate liquids | Full pass/scene breadth, special cases, projected shadows, environment and cross-model depth equivalence |
| Models | Native projection/lighting, absolute authored poses, full affine scale/shear palette work exists | Default action/weapon/form selector and timing still heuristic; eighth-class mapping incomplete |
| Streaming | Cooperative sector/canvas/model staging and atomic publication exists | Atomic hitches, whole-process memory/VRAM limits and long routes unqualified |
| Movement | Actual mouse click moves player; fixed-tick movement and AStar window | Gold admission, actor-specific routing, distant routes, continuous animation, collision/action integration |
| Interiors/triggers | `Interior` consumes mutable `Triggers` for rendering | Full interaction/prerequisite/delayed-event behavior; movement binding uses a different old reader |
| Actors | Persistent registry independent of streamed views; tutorial nun registered | Complete stats/ownership/faction/task/action state and autonomous AI |
| Scripts | Eight admitted opcodes with whole-hook preflight | Control flow, calls, scheduler, persistent handles, broad effect dispatch |
| Tutorial | One OnEnter hook creates/draws nun at final destination | Actual NPC_Goto route, retained script host/handle, dialogue/task lifecycle |
| Combat | Recovered hit/damage kernels and a finishing `--fight` demonstration | Interactive targeting and retaliation; authoritative weapon/armour/HP; timers, range, AI, death/loot |
| Arts/skills | Template records and recovery helpers | Live ticking/effects, learning/allocation, combos and class-specific systems |
| Equipment | Real mesh attachment and definition readers | Inventory instances/transactions; rolled/conditional stats; actual starting loadout |
| XP | Batch encounter records an award | Mutation of persistent hero XP, level transition, points/stat/UI updates |
| HUD | Retail sheet/element readers, health update API and one art slot are production wired | Portrait content, complete controls/panels, markers, XP, service/dialogue interfaces |
| Audio | Sector music ID selected | Playback, mixing, contextual events and voice |
| Saves | PAX hero reader used for templates; movement replay exists | Full session writer/restorer; AutoSave currently counts requests |
| Mods | Primary-archive name resolution; researched retail overlays | Mounted content graph, family-specific merge, cache/save identity, explicit supported mod classes |
| Modern release | Resizable runtime and existing input paths | Settings/rebinding/accessibility, platform qualification, asset-free export and support matrix |

Current source anchors: `main.gd:750-834,874-1009,1277-1337,3355-3508`; `world/{sim,actor_state,script,quest_cast,quest_log,encounter,path_window,walkable}.gd`; `view/{sector_view,floor_view,model_canvas,model_view,hud}.gd`; `formats/{hero,equipment,wpmod,pak,texture,rigs,ui_elements}.gd`.

Important distinctions:

1. `Encounter.strike` receives hardcoded foe HP40 and physical damage7, no armour/resistance loadout, and only the batch flag drives it. Its art use changes readiness, not the art's actual effect. Killing monster107 to complete quest74 is admitted in source as a port-authored link, not recovered quest causality.
2. Main initializes hero HP100 independently. Tutorial cast HP40 is another local default. These are not retail creature-stat implementations.
3. `QuestCast.npc_goto` stores destination data. Main spawns there; the host/handle map is not retained, and the temporary actor map is keyed by creature type. Two instances of the same type are not a safe identity model. Registering the nun as an actor did not implement scheduled NPC walking.
4. `Equipment.dress_creature` returns weighted candidates; `Wpmod.apply_to_item` aggregates raw fields without complete chance/condition/roll semantics. Passing reader checks does not make either a retail item generator.
5. `Walkable.bind_statics_and_types` has check callers but no production composition call in the audited engine. Blindly connecting it would still import the wrong Gold contract; the required fix is not merely wiring.
6. Some old docs understate real integrations: Hero, Factions, Triggers and health-ring updates do have production callers. This audit does not erase those advances.

## 5. Fresh executable observations

Private evidence directory: `donotpublish/tmp/engine-revision-20260929/`.

### Engine batch combat

`timeout 90 godot --headless --path godot-port -- --install=<private install> --fight=200`
completed normally. Quest74 reached `done=true` after9 swings/6 hits; foe HP40→0; award148; four journal entries. Quest1 separately reported10 records and one placed/animated actor. This proves the batch route, not interactive combat, meaningful art effects, real initial statistics, saving or campaign completion. Summary: `combat-smoke.txt`.

### Engine actual rendering and input

Godot4.7.2.stable.arch_linux.ed1daf0bf, Forward+/Vulkan, RTX5070Ti Laptop GPU, Xvfb1024×768, Dummy audio. A fresh-process private probe loaded `main.tscn`, waited for production readiness, injected a real mouse click at700,400, sampled presented frames for5seconds and captured output. No OS disk-cache flush was performed; this is **not** a cold-storage benchmark.

| Measurement | Observed |
|---|---:|
| Synchronous scene initialization | 1669.593ms |
| Loading samples | 925 frames |
| Loading median / p95 / max | 26.010 /46.711 /251.876ms |
| Five-second input-route samples | 211 frames |
| Input-route median / p95 / max | 21.339 /34.144 /60.328ms |
| Player before → after | (3236.5,2511.5) → (3238.452,2509.5) |

The input-route distribution includes travel and any subsequent idle time; it is not a movement-only sample. Repeated IDLE↔WALK logs occurred during the route. This is an observed symptom requiring a tick/render-state investigation, not a diagnosed cause. `runtime.json` and `after-click.png` retain evidence. No sustained60FPS or comparative speedup is claimed.

### Existing image benchmark

`bash autoresearch.sh` exited1: **Repeated renders differ; no metric emitted.** Both actual scenes rendered. Comparing just those repeats found2702 differing pixels in bounding box(405,252)-(552,485), around the actors. One frame reported1706draw calls. No new retail delta is quoted. The guard was not weakened and the reference/baseline was not changed. `repeat.json` records the repeat-only measurement; the benchmark keeps logs/images in `tmp/autoresearch-start-scene/`.

Finding1275 already warned that progressive, loading-relative capture disturbed animation phase. A correct replacement must select a defined semantic state and simulation/animation time, not hide actors, loosen equality, or treat an old frame at a different quest state as truth.

### Retail actual gameplay and native helper

The canonical menu route under Xvfb/gdb reached Seraphim gameplay; captured at37929ms and46929ms, with an intervening click. Both frames were inspected. The run ended at the intentional53second timeout (exit124), not a claimed clean game exit. The first launcher attempt failed before game execution because gdb lacked `--args`; corrected launch succeeded. 64-bit gdb printed ignored32-bit preload warnings; the32-bit game demonstrably loaded autopilot and rendered.

These retail frames show the novice elsewhere than the port's final-destination placement, plus portrait/compass/HUD/shadow differences. This is reason to align **state and chronology**, not to adjust a sprite offset to fit an unsynchronized frame. Native aggregate FPS printed by autopilot is not compared to the instrumented Godot distributions as a controlled performance result.

## 6. New reverse-engineering corrections

### 6.1 Shipping Gold world-admission predicate

Read directly in IDA from the staged LGP database and independently in ENG/RUS corpus:

| Build | Base predicate | Broader companion |
|---|---|---|
| LGP1.0.02 | `0x080EE194` | `0x080EE244` |
| Gold2.28ENG | `0x00636C10` | `0x00636D20` |
| Gold2.28RUS | `0x00637040` | `0x00637150` |

All three base functions resolve a support-aware cell. Missing cell returns false. Let `c = cell[31] & 15`: base admission is `c != 1 && c != 2 && !(cell[30] & 8)`. If `cell[30] & 4`, a runtime index keyed by `(x<<18)|(y<<4)|(layer>1 ? layer : 0)` can resolve a sixteen-byte trigger record and override the answer with bit0 of byte10. A missing mapping/record preserves the base answer. The companion function additionally accepts class2 when the base rejects it. Do not merge the two predicates.

LGP's record accessor `0x080ECC2E` bounds an index into a sixteen-byte vector at world+612..616. The authored current `formats/triggers.gd` calls byte10 a mutable state, not a static creature-type mask. Index population/mutation and support/caller contracts still need an explicit join.

A bounded gdb observer captured24 entry/return pairs during new-game initialization. Positive/negative controls:

- class2, height26=0 → false; class1, height26=0 → false;
- class0, height26=0 → true;
- class0, height26=1 at(3201,2469) and(3239,2490) → true.

All observed flags were0 and layer0. These cases agree with the base predicate and disprove the height-byte substitute. They do **not** verify trigger override, bit8, support-layer or actor-specific branches. Source callers include respawn clearance checks and creature paths; a helper observation is not a proof of whole pathfinding. `native-navigation.txt` and `navigation.gdb` retain the narrow evidence. [Footprints](../formats/footprints.md) now marks its old all-Gold closure as withdrawn.

### 6.2 HP loader attribution is false

`sub_80EF028` is called with `WORLD/SECTORS.KEY`; `sub_80EF4EE` with `SECTORS.KEYX` (`linux1002/chunks/00012.c:5068-5078`). The first allocates sector objects and logs `SECTORS.PAK`; its stores at+1224/+1228 belong to sectors (`:6730-6855`). They cannot prove creature max-HP origin merely because the offsets equal0x4C8/0x4CC.

The creature HP array/accessors and prior live damage observations remain useful; the specific claim “max HP is loaded, not derived” and finding1140's spawn-loader closure are retracted in [combat formulas](combat-formulas.md). Recalculation and save restoration are distinct source questions. Do not replace the port's invented HP with an unrelated sector field.

**Replacement mechanism located, cross-build corroborated and reached live:**
`CalcResults` at `0x820E04C` conditionally calls `0x81F4FFA`. That callee
derives max HP and writes combat-block+292, which is creature+0x4CC.
ENG `0x5658F0` and RUS `0x565BA0` contain the same base arithmetic, with a
different block-relative output offset+300. Thus “CalcResults never writes
max HP” also missed a callee and a pointer-base change.

The base uses fields `b0=u16[block+74]`, `b1=u16[block+80]`, level `L=+86`,
and live fields at+16/+22. It forms integer-divided level terms
`B=b0+b1+floor(b0*(L-1)/10)+floor(b1*(L-1)/10)`, exponent
`1.5+max(0,60-b0-b1)*0.0039963233`, and truncates
`((u16[+16]+u16[+22])/B)*B^exponent/3.1` before additional skill,
special/type, difficulty and related-object modifiers. Field names remain
offset-based: the initializer's two attribute orders differ. Do not
transcribe this base alone as the complete stat formula.

A second bounded live LGP observation reached the function epilogue32times.
It saw construction/context stages with zero/100 values, then type1 level1
at119/119, type295 level4 at134/134 and type288 level4 at120/120.
`hp-live.log` and `hp.gdb` retain all input/output samples. The run reached
the scripted new-game route and ended at the intentional39second timeout.
These observations prove that the recovered callee runs and produces
changing creature HP; they do not isolate every modifier or prove exact
numerical parity across equipment, level changes and all types. The focused
HP admission ticket therefore remains necessary, but the function is no
longer unidentified.

The same inspection retracts **“ProzHP is dead.”** LGP instruction
`0x81F54D6` reads `[0x8B89420 + index*4 + 0x728]`;
`0x8B89420+0x728=0x8B89B48`, exactly the documented ProzHP table.
The old literal-address census missed indexed access. The surrounding
non-hero/type branch remaps difficulty before applying the scale.
This proves a reader exists; it is not a claim that every difficulty's
numerical effect has been observed.

### 6.3 XP gauge versus learning points

The cited addExperience block at `linux1002/chunks/00017.c:3431-3455` is inside LEVELUP. It adds `sub_821A654`'s occupied-skill count to learning-point fields and logs `skill/learn/val/remain`; this is not sufficient evidence for a ten-bead XP denominator. Actual level lookup goes through `sub_82164BC` and `sub_8216294`; its x87/pow decompile needs instruction-level checking. Finding1171's gauge interpretation is not an implementation specification. XP pool location, award, level threshold, skill points and gauge fraction must be verified separately.

### 6.4 Scheduler is not “call the hook on camera change”

Fresh IDA inspection confirms LGP `0x0829F9D4` calls region initialization first, establishes sector context, resolves `Sector%d%3.3dInit`, executes through `0x0825D7A2`, then clears context. `0x0829EFBC` uses region-init state, context and a packed procedure resolution; where present it executes the high-word procedure before the low-word one. This corroborates existing scheduler research, not a claim that the full scheduler is implemented or freshly tested. Persist once-only initialization and script identity; use actor world transitions, not render residency, as the semantic trigger.

## 7. Performance: established costs and safe direction

Prior measured evidence, explicitly not this revision's A/B:

-1271's “dominant defect fixed” claim was withdrawn by1272 because it used image correctness instead of movement time.
-1273 measured camera-pan command reuse: isolated2560×1440 sync p95≈135→2.4ms; maximized real-walk p95≈546→59ms. Residual worst frame≈760ms remained.
-1274 measured cold-view sector/canvas builds in hundreds of milliseconds and a474ms scripted-model construction group.
-1275 introduced cooperative3ms sector/canvas/model work and atomic publication, with loading max≈211ms at1024×768 and≈356ms maximized. Separate budgets and atomic calls are not a3ms total-frame guarantee.

Current architecture: `SectorView` stages decoded sectors; `FloorView` retains static command generations and256pixel pan padding; models render into cropped private3D viewports inserted in native2D order; liquids remain separate. Affine skinning and encoded-color rules must survive optimization. Do not replace this with generic material sorting or ordinary3D depth ordering without proof.

Prioritized **candidates**, not measured speedup promises:

1. Attribute remaining startup/navigation/model/GPU-upload atomic costs, including the≈204ms path-window fill observed now; profile loading and p99, not only idle meanFPS.
2. Share immutable model metadata, decoded textures and uploads using content-correct keys; preserve per-instance pose and material state. `_skin_material` repeats decoding/upload work in source.
3. CPU-only worker jobs with owned file cursors/buffers and immutable outputs; main-thread GPU/scene publication preserves current generation/cancellation rules. Never concurrently seek one `Pak`/`World` handle.
4. Bound **combined** CPU/GPU residency, including displayed/staging/retiring generations, actor render targets and caches.768 decoded terrain images alone is not a total memory budget.
5. Profile crowd palette updates, target resizing and unchanged render targets before batching/reuse. Preserve all nine scale/shear terms and native alpha/order semantics.
6. Use GDExtension only for an isolated measured hot loop that remains expensive after algorithm/cache fixes. No engine rewrite or speculative native migration.

DLSS depth experiments are opt-in, not a stock acceleration result; finding1276 reports unusable nonzero-depth output. Keep them outside fidelity and performance qualification.

## 8. Retail resources, mods and modern engine contract

### Retail precedence must remain family-specific

Current main generally opens primary archives directly. Findings1258/1259 and `install-inventory.md:99-190` establish distinct native rules:

- Textures: primary plus00..15; concatenate declared counts including empty slots; direct IDs persist; duplicate names last-win.
- Items: primary plus00..15; admitted numbered records target their own+118 type; later admitted records override;+8/+102 rebase by preceding texture counts;+112 clears. Preserve generated weapon backpointers and ascending single-pass inheritance afterward.
- Models: primary plus00..14, separate mesh/motion namespaces, names first-win. Do not invent a texture-like motion bias. Motion-patch mechanisms remain separately unqualified.

The03 packs are not “the Underworld expansion.” Generic last-file-wins cannot implement all three families. Raven Rock's DLL rewrites executable paths/code; it is not proof of a built-in loose `DLC/` override. Data-only compatibility and executable-patch-mod compatibility are different promises.

### Proposed resource architecture

One immutable resolved content set owns build/locale/campaign metadata and archive family resolvers. Compact runtime IDs stay cheap; provenance maps them to base family/ordinal/local index or stable mod package/key. Name lookup returns identity; a name or load-order list index is not persistent identity. Freeze the content graph for a session. A creator reload explicitly replaces a generation and refuses incompatible live state.

New mod manifests declare stable package ID/version, dependencies, compatibility, replacement targets and order. Base retail stays read-only. Conflicts and effective provenance are inspectable. Size-only caches in `TextureFormat` and `Rigs` are unsafe for equal-length edits; cache keys must include content graph/source identity, decoder version and output-affecting options. Saves record content identities separately from disposable caches.

Data-only mods are the first deliverable. Harden bounds, counts, offsets, decompression/resource budgets and path traversal before admitting untrusted files. Retail bytecode remains an explicit interpreter; unknown required semantics fail visibly. GDScript/PCK/GDExtension are executable trusted code, **not** an untrusted-mod sandbox. A general scripting API needs an independently justified sandbox runtime or isolation, restricted capabilities, deterministic simulation commands and budgets; no arbitrary OS/filesystem/network or raw engine objects. Never auto-enable executable plugins from a save.

### Portable retail installs and distribution

Current install validation checks only two lowercase paths. It can accept a Windows data tree that later fails because `UiElements` opens `install/sacred` at one ELF32 table address/name-offset layout. Recognized ELF/PE build profiles and runtime extraction from the user's own executable are required; do not distribute recovered table dumps or execute the original program as a production dependency. Inventory other executable-embedded balance/art/UI tables too.

Separate read-only retail root, engine-owned profile saves/settings, and content-addressed disposable caches. Explicit invalid `--install` must not silently switch corpora. Settings writes must preserve unrelated keys. Localized text, voice, UI font and class-script selection are separate concerns; retain the recovered locale-independent hash.

Asset-free release is technically distinct from distributing the game. Existing project prose both permits publishing engine source and forbids exports; obtain explicit policy reconciliation before packaging. Gitignore is not an export manifest. No retail assets, screenshots, saves, decompiler/symbol dumps, third-party proprietary binaries or caches enter distribution.

## 9. What “close”, “1:1”, and “modern” mean

**Close playable:** a new unmodified retail-data session reaches a real interaction→quest→two-sided fight→loot/equip→save→cold restart/load→continue route through ordinary UI. It uses real statistics and causal script effects, not the batch encounter. This is a milestone, not permission to omit the rest of either campaign.

**Single-player1:1 target:** all eight classes, both complete campaigns, applicable difficulties and mechanics, services, state persistence/import/export, localization and audiovisual/UI behavior match a declared Gold reference. Discrete state/order/ownership differences are failures; numerical/render tolerances are per channel and measured against oracle variation. Exact cross-backend pixel equality is a separate technical claim, not a replacement for gameplay equivalence. Preserve intentional compatibility quirks; document safety/platform adaptations instead of copying crashes or dead DRM/services.

**Literal whole-product1:1:** also requires multiplayer/service/save/protocol behavior. Standing project policy excludes multiplayer, so the proposed implementation must be named **single-player Sacred Gold parity**, not unqualified1:1. The full-product gap remains visible and needs an explicit future scope decision; no networking implementation or new network RE was undertaken here. Dead original internet services cannot honestly be advertised as reproduced.

**Modern:** the same rules core with optional high-DPI/responsive presentation, independent UI/world scaling, rebindable controls/controller navigation, subtitles/history, non-color-only information, reduced flash/motion, accessible semantic menus, safe saves, bounded streaming, explicit mod profiles and reproducible asset-free distribution. Modern options must not silently alter strict reference gameplay. Do not postpone safe saves or content identity until after mods depend on them.

## 10. Reference hierarchy and acceptance coverage

Proposal: Windows Gold2.28ENG is the canonical shipping-behavior reference; RUS corroborates localization/source lineage; precisely identified patched LGP1.0.02 is the instrumentable runtime/GL reference. Where they differ, record the difference and choose an explicit compatibility profile rather than invent an average. Armalion/demo provide leads, not authority over shipping Gold. Existing Linux shim/VRAM patches and Windows dgVoodoo/RDP wrappers belong in every capture manifest.

For each acceptance route record binary/resource hashes, locale/settings, campaign/class/level, relevant save/quest/region state, RNG/tick/event sequence, camera/zoom, rendering environment and input. Include all eight starts; both main quest chains/endings; progression/slot-unlock boundaries; every art/effect family; interiors/layers/portals/mounts/forms; equipment/services; death; save during transitions/cooldowns; unload/revisit; supported locale/build pairs. Network exclusions remain explicit rows, not missing rows.

Use four independent dashboards: behavioral journey completion, resource/semantic coverage, visual/audio fidelity, and frame-time/memory distributions. Every “supported” row needs an actual production caller and an exercised scenario. No single percentage combines them.

## 11. Documentation/API sources for implementation planning

Local source/prose references above are the factual baseline; current source wins over obsolete implementation claims. Particularly relevant: `game-wiring.md`, `combat-formulas.md`, `formats/{script-bytecode,pax-saves,install-inventory,pak-containers,granny-grn,world-sectors,ui-taskbar,global-res}.md`, findings1247–1276 and this audit's corrections.

Official external references consulted:

- [Godot runtime external loading](https://docs.godotengine.org/en/stable/tutorials/io/runtime_file_loading_and_saving.html): FileAccess/PackedByteArray and runtime image/audio loading; does not supply Sacred/Granny decoders or native MPEG-1 movie playback.
- [Thread-safe APIs](https://docs.godotengine.org/en/stable/tutorials/performance/thread_safe_apis.html), [WorkerThreadPool](https://docs.godotengine.org/en/stable/classes/class_workerthreadpool.html): active scene-tree/shared-resource restrictions and task lifetime/join requirements.
- [Profiler](https://docs.godotengine.org/en/stable/tutorials/scripting/debug/debugger_panel.html): distinguish script/physics profiling, render submission CPU and GPU timing.
- [Data paths](https://docs.godotengine.org/en/stable/tutorials/io/data_paths.html), [FileDialog](https://docs.godotengine.org/en/stable/classes/class_filedialog.html): writable user data and native filesystem permission limits.
- [PCK loading/security](https://docs.godotengine.org/en/stable/tutorials/export/exporting_pcks.html): replacement order/preload constraints; executable packs are not a sandbox.
- [GDExtension](https://docs.godotengine.org/en/stable/engine_details/engine_api/gdextension/what_is_gdextension.html), [godot-cpp compatibility](https://docs.godotengine.org/en/stable/tutorials/scripting/cpp/about_godot_cpp.html), [exporting](https://docs.godotengine.org/en/stable/tutorials/export/exporting_projects.html): native ABI/platform maintenance and export dependencies.
- [OpenMW mod installation](https://openmw.readthedocs.io/en/openmw-0.49.0/reference/modding/mod-install.html), [configuration paths](https://openmw.readthedocs.io/en/latest/reference/modding/paths.html), [Lua sandbox](https://openmw.readthedocs.io/en/stable/reference/lua-scripting/overview.html): useful separation of data layers, enabled content, profiles and constrained script roles; not Sacred semantics to copy.

These Godot stable pages identified4.7 when consulted. Pin the implementation's engine/API version; do not silently follow a future moving stable version.

## Open

This revision establishes the architecture and ordering, not a fiction that every retail mechanic is implementation-ready. Explicit blocking contracts remain: complete actor traversal and trigger-index updates; persistent task/quest/dialogue causality; true HP/stat generation and XP thresholds; item rolls/modifiers/economy; art-specific effects/action timing; world-save sections/version migration; full model-motion overlays; and unsupported executable/media/platform variants. The implementation proposal names prerequisites, owners, source anchors and acceptance witnesses for these. No missing behavior may be replaced with a plausible formula, no-op, one-scene special case, or an unmarked “temporary” gameplay value.
