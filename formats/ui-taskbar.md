# The in-game HUD

**Status:** Read
**Purpose:** Where every piece of Sacred's taskbar comes from and where it goes.

Reader: `engine/view/hud.gd`. Gate: `engine/checks/hud_check.gd`.
Art census: `engine/probes/uiart_probe.gd`. Findings log row 962.

## It is a table, not a thousand call sites

The reason this looked unapproachable for a long time is the assumption that
UI layout is scattered through UI code. It is not.

**A static array of 1887 texture sub-rects at `0x880DC68`**, stride **84**,
ids 1…1886 (id 1 is `"INVALID"`). Element *N* is at `0x880DC68 + 84*(N-1)`:

| off | type | field |
|---|---|---|
| `+0x00` | `char[32]` | sheet name (`GUI_main_03.TGA`) |
| `+0x20` | int | texture handle — 0 in the file, filled at load |
| `+0x24`…`+0x30` | float | `u0, v0, u1, v1`, texels, **inclusive** |
| `+0x34`…`+0x40` | int | anchor rect `hx0, hy0, hx1, hy1`, sheet-absolute |
| `+0x44` / `+0x48` | int | `W`, `H` — 0 in the file, computed at load |
| `+0x4C` | int | element id |
| `+0x50` | int | type flag, 0 or 9 |

The loader is `sub_8501A40`. **Every sheet is 256×256.**

### Size is `u1 − u0`

Not `u1 − u0 + 1`, and this is checkable rather than a preference: retail
places the console at `x = 512 − W/2` and its transcribed position is 397,
which needs `W = 230` — the console's rect is `(0,169)–(230,255)`.

Writing every rect one texel larger put the middle combat-art slot one pixel
off centre and pushed the collect toggle one row off the bottom of the canvas.
`hud_check` caught both, which is the argument for asserting the arithmetic
relations rather than the literals.

## The canvas is fixed at 1024×768

`cUI_Manager::createWindows` (`sub_84FA548`) hard-codes every window rect:

```
UI_WND_TASKBAR    {   0, 676, 1024,  92 }
UI_WND_CONSOLE    { 256, 640,  512, 128 }
UI_WND_QUESTBOOK  {  64,  64,  832, 576 }
UI_WND_INVENTORY  {   0, 256,  576, 272 }
UI_WND_EQUIPMENT  { 604, 256,  256, 256 }
UI_WND_MEGAMAP    {   0,   0, 1024, 768 }
```

`676 + 92 = 768` exactly — the bar is flush with the bottom.

No division against a screen-width global appears anywhere on the HUD path, so
the interface is authored in a virtual space and scaled. The port letterboxes
it for the same reason.

The vertical convention inside the window is **`y = 80 − hy0`** (`0x85E8627`:
`mov $0x50,%eax; sub <hy0>,%eax`) — every piece hangs from an anchor line at
window y 80, screen y 756.

## The bar

`cUI_Taskbar2`, layout function `sub_85E85EE`. Screen coordinates:

| element | gfx id | sheet | texels | at |
|---|---|---|---|---|
| console | 12 | `GUI_main_03` | (0,169) 230×86 | 397, 680 |
| inventory / options / map / book | 88 90 92 94 | `GUI_main_01` | 33×33 | 399,705 · 434,729 · 558,729 · 594,705 |
| collect toggle (SP) | 135 | `GUI_main_01` | 35×25 | 496, 743 |
| combat-art slots | 177…181 | `GUI_main_05` | 31×31 | an arc, middle at 497 |
| skill / spell slots | 103 | `GUI_main_01` | 63×63 | y 691; x 64…328 and 640…904 |

The five combat-art slots dip **up** at the centre, and the middle one lands at
`497 + 31/2 = 512` — exactly half the canvas. The arithmetic checks itself.

The wings are **tiled**, not stretched: each side starts with an anchor piece
(ids 6 and 9) and repeats ids 7/8 and 10/11 outward, alternating by
`rand() & 1` so the ornament does not visibly repeat. The port alternates
deterministically instead, because a recorded run has to replay identically.

## Open — the life and mana gauges

All 46 functions of `cUI_Taskbar2` (`0x85E130C`…`0x85EB38F`) were enumerated.
The class references gfx ids 6–12, 40, 41, 82, 88–95, 102–104, 135, 136, 141,
142, 175–181 and the `GUI_spell*` icons, **and nothing else**. It contains no
orb or fill-bar art and computes no fraction or scissor rect.

So where the gauges are drawn is unrecovered, and they are the most
recognisable part of a Sacred screenshot — which is why the port leaves the
gap visible rather than inventing a pair of orbs.

> **Correction, 2026-08-17 (row 1015).** This section used to add that the
> orb-looking 95×107 elements of `GUI_main_02` (ids 42, 43) "are used only by
> the mercenary window". That is wrong, and a pixel match says so: the 95×107
> block at that sheet's own origin is the **player portrait frame**, and
> retail's spawn capture draws it at (932, 15) — ring, horned finial and
> leafwork exact, including the three columns it clips off the right edge of
> the canvas. The port now draws it. The two adjacent 95×107 blocks are a RED
> ring and a GREY ring, which is what ids 42/43 most likely are.
>
> This does not contradict the enumeration above; it *locates* what the
> enumeration was missing. `cUI_Taskbar2` genuinely does not reference these
> elements, because the portrait is a **different window**, and that window is
> the thing to find. Two reasons to think the gauges are in it: `GUI_MAIN_02`
> also carries three beaded SEGMENTED BARS (yellow, green, red) plus framed
> variants of each, exactly the shape of a fill gauge; and retail draws one of
> them, green, immediately under the portrait — its beads start at about
> (942, 124) and run to the right edge of the canvas. That extent is an
> eyeballed colour mask, not a pixel match, so treat it as a place to look
> rather than a rect to transcribe. Neither bar is wired, and which
> quantity the green one reads is unconfirmed.

### The bar is now a measurement, not a place to look (row 1038)

The eyeballed extent above is superseded. Measured off a retail 1024×768
frame and then pixel-matched:

| | |
|---|---|
| art | `GUI_MAIN_02` rect **(76, 125, 72, 7)** — the FRAMED GREEN beaded bar |
| drawn at | **(942, 124)** |
| structure | **10 beads**, each 5 px wide on a **7 px pitch**; rows 125–129 are solid, 124 and 130 show the gaps |
| frame | the tan surround runs about (937, 121) to (1016, 133) |
| match | mean per-channel error **0.44** — under the ~1.0 "this IS the art" threshold |

The runners-up corroborate it rather than competing: they are the same `y`
offset by exactly 7, 14, 21 px — the bead pitch — which is what a tiling bar
should produce and what a coincidental match should not.

The sheet carries **six** bar variants, not three: yellow, green and red
beaded bars in both an unframed and a framed form, plus gold and blue
Ω-segment bars beside them. Retail draws the framed green one.

**The portrait ring is confirmed, not contradicted.** Retail's frame shows a
RED ring where one might expect silver; the 95×107 block at the sheet's own
origin — the one the port already draws — **is** the red ring, and the grey
one is the block beside it at (95, 0). Matching the ring rect directly returns
error 64 and should be ignored: that rect contains the live portrait render
inside it, which is exactly the case this file's own matcher warns is answered
confidently and wrongly.

### The bar is NOT health, and the gauge was never a bar (row 1039)

The capture ran, and it refuted the natural reading. Driving retail from the
staged combat save under gdb — breakpoint trace and `SACRED_SHOT` frames from
the **same process**, so the numbers and the pixels cannot disagree about which
run they came from:

```
WHO def=0xafdf0b0 type=1 lvl=1 hp=119/119   atk=0xb82c9f0 type=313 hp=42/42
WHO def=0xafdf0b0 type=1 lvl=1 hp= 87/119   ...
WHO def=0xafdf0b0 type=1 lvl=1 hp=  5/119   atk=0xb82c9f0 type=313 hp=42/42
```

`type = 1` is a playable class, so the creature being beaten from 119 to 5 is
**the hero**; the attacker at type 313 never loses a point. Across those same
frames the bar reads **72 of 72 beads lit, in every single one**. A gauge does
not stay full while its owner is nearly killed, so **the green bar is not
health.** What it does read is still unknown — but it is now excluded, which is
the useful half.

**The health gauge is the PORTRAIT RING.** Measured in the annulus over the
same run, the red drains and the grey replaces it:

| frame | ring red px | bar lit |
|---|---|---|
| 70000 | 454 | 72 |
| 95000 | 311 | 72 |
| 120000 | 168 | 72 |
| 145000 | 0 | 72 |

and a healthy frame gives 839. The red is an **arc anchored at about +32° from
bottom-centre** whose far end sweeps away as health falls — only the far end
moves, the anchor never does. Visual proof, full against nearly dead:
`analysis/evidence/hp-gauge-2026-08-20/ring-full-vs-empty.png`.

**This is why `cUI_Taskbar2` has no orb art and computes no fraction.** The
gauge was never in the taskbar. It is the portrait window, and its art is the
pair this file already identified: the 95×107 block at `GUI_MAIN_02` (0,0) is
the **full** state and the one at (95,0) is the **empty** state. The port
already draws the red one at (932,15) — what it lacks is the grey composited
over it by fraction, not a new asset.

> ⚠️ **Which HP slot the gauge reads is NOT settled, and one obvious test does
> not work.** Pinning `+0x4C8` to 60 at every blow and watching the ring keep
> draining looks like proof that it reads `+0x4D0` — it is not. Printing the
> slot *before* overwriting shows the pin is gone by the next blow (set 60,
> reads back 109, 79, 78, 76 …), so the write is being undone and the ring's
> behaviour says nothing. The two slots do transiently differ, with `+0x4C8`
> the lower of the pair before they re-converge, which reads like `+0x4C8`
> authoritative and `+0x4D0` a display value easing toward it — a reading, not
> a measurement.

### The green bar is the EXPERIENCE gauge — and row 1039 read it upside down (row 1040)

**Correction first.** Row 1039 said the bar "reads 72 of 72 beads lit, in every
single one". It does not, and it never did. Those beads are the **empty
track**: muted green, about `(136,170,136)`, which a naive `g > r` mask counts
as lit. The bar was reading **zero** in every frame of every run. The row's
*conclusion* — that the bar is not health — is untouched and in fact
strengthened, but its evidence was described backwards.

**What it actually reads.** Four things were varied and moved it not at all:

| varied | result |
|---|---|
| hero's health, 119 → 5 | no change |
| walking (RMSE 13530 between frames) | no change |
| eight right-click combat-art uses | no change |
| the target monster's health, forced to 8/42 | no change |

Then the hero was made to **kill** something — clicks driven onto the hostile,
any struck monster forced to 1 hp — and the first bead turned from the track's
`(119,153,119)` to a vivid `(85,255,0)`. More kills lit more beads: **0 → 1 →
3**. Proof: `analysis/evidence/hp-gauge-2026-08-20/xp-bar-0-vs-3-beads.png`.

A ten-segment gauge under the portrait that starts empty and advances on a kill
is the **experience bar toward the next level**. Strictly, what is measured is
"advances when the hero kills"; three beads from two forced kills means the
increment is not one-per-kill, which is what an experience value does and a
kill counter does not.

**It is two elements, not one**, which is why it looked static:

| | element | rect in `GUI_MAIN_02` | colour |
|---|---|---|---|
| empty track | **67** | (74,123)–(146,132) | muted, mean (104,117,85) |
| fill | **68** | (0,119)–(73,125) | vivid, (51,170,0) |

drawn at **(942,124)**, ten beads, 5 px on a 7 px pitch, filling left to right.
The siblings are the same pair in other colours — 65/69 the yellow and red
tracks, 66/70 their fills — so the sheet holds three complete gauges, not six
loose bars.

**Not located: the experience value itself.** Dumping the hero creature's
`+0x420`…`+0x620` across a kill shows only two noise fields moving, so it lives
off the creature struct. The port can draw the track today; it cannot fill it
until that is found.


### The fill is symmetric, and the port now draws it (row 1041)

**Correction.** Row 1039 called the red "an arc anchored at about +32° from
bottom-centre whose far end sweeps away as health falls — only the far end
moves, the anchor never does". That is wrong in both halves. Classifying every
band pixel of retail's own frame as nearer the red art or the grey art puts the
boundary at **row 54 on the left side and row 54 on the right side**: it is
horizontal and mirrored about bottom-centre. Nothing is anchored off-centre and
no single end moves — the two ends move together. The +32° figure came from
eyeballing an annulus, and an annulus read by eye will always look like it
starts somewhere.

**The band is measurable, and it is the whole gauge.** Diffing `GUI_MAIN_02`'s
(0,0,95,107) against **(96,0,95,107)** gives **2580 pixels that differ** against
**1618 identical** — the two blocks are one frame painted red and grey, and the
differing set is the gauge. It spans rows **10…102 and columns 0…84 only**: the
horned finial and the leafwork down the right side are IDENTICAL in both
blocks, so they are frame, not gauge.

> ⚠️ **The empty ring is at x 96, not 95, and the one pixel matters.** These
> numbers are the corrected ones; the first version of this section used 95,
> which shifts the two blocks against each other and invents a differing pixel
> at every edge in the art. That inflated the band to 3814 px, dragged the
> leafwork into it, and dropped agreement with retail's frame from **97.3 % to
> 87.5 %**. The value comes from retail's own element table, not from a
> guess — see the name table below.

**Which fill LAW it is remains undetermined, and the data cannot settle it.**
Red area is very nearly linear in the hit-point fraction, and both candidate
laws reproduce the measured series to within 0.033:

| hp | measured red | arc-proportional | height-proportional |
|---|---|---|---|
| 0.529 | 0.541 | 0.523 | 0.517 |
| 0.345 | 0.371 | 0.342 | 0.371 |
| 0.210 | 0.200 | 0.214 | 0.253 |
| 0.042 | 0.000 | 0.042 | 0.025 |

Mean error **0.026 for both**, to three places. They cannot be separated because
the band is near-uniform per angle — 439, 415, 417, 443, 471, 395 pixels across
six 30° bins — which makes arc length and height very nearly the same
function. A **symmetric** sweep from bottom-centre
also puts both endpoints at equal height, so the two laws draw the same
*boundary shape* and differ only in where it sits. One frame cannot tell them
apart; only a series with a much better red mask could.

**The port takes the height slice**, because a slice is two rects and a sweep is
a shader. Verified against retail's frame rather than against itself: at the
best-fitting fraction (0.50, boundary row 53) the port's classification agrees
with retail's on

agrees with retail's own on **97.3 %** of the 2119 band pixels it can classify
(98.2 % once the band extent is taken from the corrected rect too). The
residual is scattered single pixels with no shape, which is what noise looks
like and not what a wrong model looks like — a wrong model fails in a *shape*.

**At full health the port draws no grey at all.** Not an optimisation: the two
slices are complementary, so a zero-height grey rect under the red would blend
the red ring's soft edge against grey rather than against the world and move
pixels in a frame the capture runbooks md5.

**What is still missing is the number, not the gauge.** The port has no hero
hit points to feed it — `world/encounter.gd` tracks only the hostile's — and
inventing a maximum would be inventing balance data, since which table loads
`+0x4CC` is still unidentified. `view/hud.gd` exposes `set_health(frac)` and
defaults to full, which is what retail draws at spawn.


## Every element has retail's own NAME — and there is no mana (row 1042)

The identification method this file has used throughout — match a rect against
a sheet, reason about what it looks like — was never necessary. **The names
were in the binary the whole time**, and getting them wrong twice (the ring
assigned to the mercenary window, the potion belt called the combat-art slots)
was the cost of not finding them.

`tools/formats/uinames.py` reads them. A packed NUL-separated blob of 1452
strings at file offset `0x6CF9FD`, with

    gfx id = blob index − 5

**Nothing points into that blob** — no pointer to any of its strings exists
anywhere in the image, and no instruction takes one as an immediate. It is
walked, not indexed, which is exactly why every xref search for it came back
empty.

**The anchor is self-identifying**, which makes the offset a measurement:
blob index 6 is `UI_INVALID`, and gfx id 1 is the one entry whose *sheet-name*
field literally reads `INVALID`. Three shape checks agree independently, and
every entry in the named range carries its own id field equal to the derived
id — 1446 for 1446.

Coverage is partial and the end is ragged: the blob opens with four attribute
names (`control`, `type`, `tooltipFade`, `tooltipDelay`) and closes
`UI_MAX`, `UI_STATIC`, `UI_LISTBOX`. Names cover ids 1…1444 against a table of
1886, so `GUI_HERO_*` (1488…1511) is unnamed. An id the tool cannot name is
not an error.

### What it settles immediately

| id | retail's name | what this file had said |
|---|---|---|
| 42 / 43 | **`UI_CHR_HEALTH_01` / `_02`** | "most likely a RED ring and a GREY ring" — now named, and named *health* |
| 65…70 | `UI_BAR_YELLOW/GREEN/RED_EMPTY` and `_FULL` | "three beaded gauges" — confirmed, and the names carry no meaning, so the green bar's XP role stays an empirical result |
| 76…81 | `UI_HORSE_VHP_*`, `UI_HORSE_HHP_*`, `UI_HORSE_SHOE_*` | unidentified — they are the HORSE's gauges, vertical and horizontal |
| 103 / 104 / 175 | `UI_ACTION` / `UI_ACTION_BRIGHT` / **`UI_ACTION_GRAYED`** | unidentified |
| 176…181 | `UI_POTION_EMPTY/RED/BLUE/GREEN/PURPLE/YELLOW` | called "combat-art slots 177…181" in the table above — **wrong**, they are the potion belt, which the port already draws correctly for a different reason |
| 351…353 | `UI_PORTRAIT_MP` / `_MP_H` / `_MP_E` | — the one trap: **MP is MULTIPLAYER**, a 127×189 frame between `UI_NET_*` and `UI_HORSEMERC`, not mana points |

### There is no mana gauge, because there is no mana

Searching all 1446 names for `MANA`, `_MP_`, `ENERG`, `STAMIN` or `AUSDAUER`
returns **only the multiplayer portrait trio**. That is not an absence of art;
it is an absence of the mechanic, and four independent lines say so:

1. **The attribute list has no mana.** Retail's own character sheet, from
   `global.res` slots 1401…1406, reads Strength, Endurance, Dexterity,
   **Physical Regeneration**, **Mental Regeneration**, Charisma. The whole
   23 123-entry text tree contains no `Mana` stat label at all.
2. **What a spell costs is TIME.** Slots 1459/1460 are `Regeneration Spells`
   and `Regeneration Special Move`; 6970 reads *"Accelerates the regeneration
   of spells."*
3. **The mana potions are dead types.** `TYPE_OBJECT_POTION_MANA_MINOR/MAJOR/
   FULL` (ids 5060…5062) exist in the type table beside `_HEALTH_` and
   `_STAMINA_`, and the item-description function `sub_815DDA2` **returns 0 for
   every one of them** while the live coloured potions (5137…5172) all get
   text. The live blue one is a `Potion of Concentration`, not a mana potion.
4. **The pool triple is entirely hit points.** `sub_819AE44` indexes
   `creature + 0x4C8` by 0…2 but ceilings all three against the *fixed*
   `+0x4CC`, and the healer at `sub_85667FC` writes `get(1)` into **both**
   index 0 and index 2 on a full heal. So `+0x4CC` is max HP and the other two
   are current-HP-like. There is no mana slot in it.

**The resource is per-art regeneration, and it shows on the combat-art slots** —
which is where a mana bar would have been. `UI_ACTION_GRAYED` is the slot while
its art regenerates, against `UI_ACTION` / `UI_ACTION_BRIGHT` when it is ready,
and the element table carries a full `UI_SPELL_nn_EMPTY / _LOAD / _FULL` set
for 105 arts — a **`_LOAD`** state per art, which is a timer per art and not a
pool shared between them.

> Not yet read: what fraction drives `_LOAD`, and where the per-art timer lives
> in the creature. The *mechanic* is settled; its field is not.


## What drives `_LOAD`: it is a waterline, and the quantity is regeneration (row 1043)

`sub_85E676A` is the combat-art slot renderer, and the slot turns out to use
**the same idiom as the health ring** — two elements stacked, split at a
waterline, no compositing rule at all.

Per slot it obtains three element indices and one fraction `f`, then:

```
h    = control height − 1
split = h × (1 − f)
        top    ← element[0], rows 0 … split          (EMPTY)
        bottom ← element[1], rows split … h          (LOAD)
if f ≥ 1.0 and this slot is the SELECTED one:
        whole  ← element[2]                          (FULL)
```

So `_LOAD` is not "loading" — it is **loaded**, and it fills the slot from the
bottom as the art recharges. `_EMPTY` is the drained remainder above the line.
`_FULL` is a distinct brighter art drawn only when the slot is both ready and
selected. The UV is stepped by `split/256 + 1/512` — texel-to-UV with the usual
half-texel — on the source rect's `v0`, which is what makes it a slice of the
art rather than a scale of it.

**This answers the older open question at the bottom of this file.** A filled
slot is not a composite of icon + backing + frame; it is one of three whole
64×64 arts named per combat art in the element table, chosen and sliced. The
greyscale `GUI_MOVE_ATTACKE` that looked like an un-tinted icon is the EMPTY
state.

### Where the fraction comes from

Two lists hang off `creature + 936`, and which one is read depends on the
slot's kind (`sub_833EBBE` → 2, 3 or 4):

| kind | list | stride | fraction |
|---|---|---|---|
| 2, 3 — combat arts | `+250` | **22 B** | `1 − remaining / total`, from the record's `+0x12` and `+0x0A` floats |
| 4 — spells, ids ≥ 512 | `+262` | **38 B** | `min(elapsed / total, 1)`, from `+0x22` and `+0x1A` |

The 22-byte record also carries the art id at `+0x04`, which is how
`sub_8219C44` finds a slot's entry. **So the regeneration timer is per art, in
a per-creature list — not a pool, and not one clock shared between arts.** That
is the field row 1042 said was still missing.

Two overrides sit on top:

- If the art is unusable — wrong weapon in hand, checked against
  `sub_819BA46` — all three elements are forced to `element[0]`, so the slot
  shows EMPTY at every fraction. That is the greyed-out slot.
- If the hero has a linked creature (`sub_81A3AD0`: flag `0x2` at `+20` and a
  `cCreature` at `+0x1EC` — the mount), the triple is replaced wholesale and
  the fraction comes from **`creature + 0x4E0` over `creature + 0x4E4`**
  instead. A second timer, at creature scope rather than art scope.

> Not read: which art each of the two lists is populated from, and what
> `+0x4E0/+0x4E4` count while mounted.

### ⚠️ The element-name offset is NOT constant — row 1042 was wrong about that

Row 1042 gave `gfx id = blob index − 5` flat. It holds up to id **351** and
fails above it: from id **443** on the offset is **− 3**, because two elements
somewhere in 352…442 carry no name at all.

**Retail's own data is what caught it.** The combat-art table at `0x8793D00`
(stride 120) carries at `+48/+52/+56` the three element indices each art draws
with, and those must be an EMPTY/LOAD/FULL triple. At offset − 5 they straddle
triple boundaries; at − 3, **52 of 52 land clean**. The low end is pinned the
other way: id 351 is `UI_PORTRAIT_MP` at 127×189, and the − 3 reading puts a
60×25 arrow there.

`uinames.py` now returns `None` for ids 352…442 rather than guessing, and its
self-check asserts both the refusal and the 52 clean triples.

**Nothing row 1042 concluded depends on this.** Every element it identified —
42/43, 65…70, 76…81, 103/104/175, 176…181, 351 — is at or below the drift, and
the mana result is a search over the *name list*, which is complete whatever
the offsets are.


## Open — a filled art slot is a composite, not a blit

Recorded because the obvious searches are already spent (row 1021).

The two FILLED skill/spell slots are the last interface elements the port
draws as empty rings. Their art is **not** in `GUI_MAIN_01`…`06`: matching
retail's own 63×63 against all six sheets bottoms out at a mean per-channel
error of 56, where a genuine find scores under 1.0.

It lives one texture per art instead. `texture.pak` carries 440 `GUI_*`
entries outside `GUI_MAIN_*`, named per art in German across four prefixes —
`GUI_MOVE_ATTACKE`, `GUI_MOVES_KRIEGSSCHREI`, `GUI_SPELL_REIKI`,
`GUI_SPELLS_HOELLENFEUER` — each a 256×256 whose picture occupies only the
**top-left ~64×64**, the rest fully transparent.

But the slot is a composite of at least three layers. `GUI_MOVE_ATTACKE` is
crossed swords on a plain **dark greyscale** disc, while retail's spell slot
shows those same swords in **blue over a green orb inside a gold ring**. So a
backing colour and a frame are applied over the icon, and the backing is
presumably the art's school. Shape correlation across all 440 candidates tops
out at 0.40 with implausible winners and no separation, so identification by
search is exhausted from both directions. What is missing is the compositing
rule, not a better matcher.

## `GUI_HERO_*` is not a HUD portrait

Ids 1488–1511, eight classes × 3, rects `(0,0)–(255,255)`, `(0,0)–(255,255)`,
`(0,0)–(128,255)`. The 1/2/3 suffix is **not a variant selector** — the three
are horizontal slices of one 638×255 image, and the referencing code is
`cUI_Character` / `cUI_EscMenu`. It is the character-selection full-body art.

The in-game portraits are `GUI_PORTRAIT00`–`15`, `GUI_portraitX01`–`82`, and
the quest-book class portraits `GUI_QBP_*` (ids 1480–1487).

## A second pixel format

`texture.pak` `kind == 6` is **raw BGRA8888**, uncompressed, `w*h*4` bytes from
`+80`, against `kind == 4`'s zlib ARGB4444. 233 entries carry it, and they are
exactly the ones an interface needs — every `GUI_CHAR_*`, `GUI_HERO_*` and
`GUI_UW_*`. Channel order confirmed by eye: `GUI_MAIN_LOGO` reads as the
Sacred logo in **gold** under BGRA and in blue under RGBA.

## The slot count is READ OFF A FRAME, not fixed at five

Retail draws one skill and one spell slot at the Seraphim's spawn. The
placement formula the port already transcribed makes the count recoverable
from a screenshot:

    skill slot i:  x = 394 + 66*(i - n)      spell slot i:  x = 640 + 66*i

with `n` the visible slot count. Retail's capture puts the leftmost skill slot
at **328**, and `394 - 66n = 328` gives `n = 1`. The spell side agrees from a
different direction: `640 + 66*i` would put a second spell slot at 706, and
retail draws bare wall there.

The ornamental rails follow the slots rather than the screen. Each side butts
the console (left edge 397, right edge 627) and tiles outward one piece per
slot, which puts the right rail's end at `627 + 104 = 731` against retail's
measured 732. Tiling them from a fixed `x 32` / `x 890` instead laid ten rail
tiles and eight 63×63 rings over open terrain — together 13% of the frame
delta in a two-engine compare.

## Two pieces recovered by pixel match rather than from the table

Matching retail's own frame against the decoded sheets resolves art the gfx
table did not lead to. Both land at a mean per-channel error **under 1.0**,
which is the art itself and not a resemblance:

| element | sheet | rect | screen |
|---|---|---|---|
| portrait frame | `GUI_MAIN_02` | `(0, 0, 95, 107)` | `(932, 15)` |
| empty potion flask | `GUI_MAIN_05` | `(195, 65, 31, 31)` | slots 2…5 |

The portrait frame is exact including its clipping — 95 wide at x 932 runs
three columns past 1024, and retail clips it the same way, so a gate that
requires every piece to fit inside the canvas is wrong for this one.

The potion belt is a **contents** table, not a slot table: column 129 of
`GUI_MAIN_05` holds five *different* potions stacked vertically, and taking
one per slot down that column draws the belt as a full rainbow. Retail's
spawn frame carries one potion and shows a single shared empty flask in the
other four.

> **What is NOT in `GUI_MAIN_01`…`06`.** The two FILLED art-slot icons. The
> same match against all six sheets returns nothing under an error of 56 for
> retail's 63×63 slot content, where the pieces that are present match under
> 1.0 — so the combat-art icons live somewhere else entirely.

## The day/night dial is 56 px, and Godot will not let you say so

`GUI_DAYNIGHTDISC` is a 128×128 texture drawn as a 56×56 quad. A
`TextureRect`'s minimum size comes from its texture and the recalculation is
*deferred*, so assigning `size = 56` is clamped straight back up to 128 —
before `add_child` and after it alike. The dial painted a night-sky annulus
across the whole console until the size was baked into the image instead.

---
Provenance: `sub_8501A40`, `sub_84FA548`, `sub_85E85EE`, `sub_85E3102`,
`sub_85E3036`, `sub_85E306A` in `install/sacred`; findings log rows 962, 1015,
1021. The 2026-08-17 additions are measured against retail's own spawn capture
rather than read out of the binary — see `tools/parity/sheet_match.py`.
