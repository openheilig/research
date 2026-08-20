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
