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
orb or fill-bar art and computes no fraction or scissor rect. The orb-looking
95×107 elements in `GUI_main_02` (ids 42, 43) are used only by the mercenary
window.

So where the gauges are drawn is unrecovered. They are the most recognisable
part of a Sacred screenshot, which is exactly why the port leaves the gap
visible rather than inventing a pair of orbs.

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

---
Provenance: `sub_8501A40`, `sub_84FA548`, `sub_85E85EE`, `sub_85E3102`,
`sub_85E3036`, `sub_85E306A` in `install/sacred`; findings log row 962.
