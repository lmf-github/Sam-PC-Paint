# SAM Coupé SCREEN$ Editor — User Guide

*For users familiar with the SAM Coupé and its graphics capabilities.*

---

## Overview

This is a browser-based pixel editor for creating and editing SAM Coupé SCREEN$ files in all
four screen modes. It works directly with the SAM's native file formats — you can load and save
`.ss1/.ss2/.ss3/.ss4` screen files, read and write files on `.mgt` and `.dsk` disk images, and
import standard image formats (PNG, BMP, JPEG) with automatic palette quantisation and line
interrupt optimisation.

The editor runs entirely in the browser with no installation required. All file operations
produce authentic SAM SCREEN$ binaries compatible with SimCoupe and real hardware.

---

## Interface Layout

```
┌─────────────────────────────────────────────────────────────┐
│  Titlebar: mode buttons, file operations, undo, zoom        │
├──────┬──────────────────────────────────────────────────────┤
│      │  Screen 1  │  Screen 2  │  → Copy to other          │
│      ├──────────────────────────────────────────────────────┤
│Tools │                                                       │
│      │         Canvas (main drawing area)                   │
│      │                                                       │
│      │                                             │ CLUT   │
│      │                                             │ strip  │
├──────┴──────────────────────────────────────────────────────┤
│  Tool options bar (brush size, gradient stops, hints)       │
├─────────────────────────────────────────────────────────────┤
│         Status bar (coordinates, colour, mode)              │
└──────────────────────────────────────────────────────────────┘
│ Left panel: Magnifier, Colours, CLUT, Full Palette, Line Interrupts │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Screen Modes

Select the active mode using the mode buttons in the titlebar: **M1**, **M2**, **M3**, **M4**.

| Button | Mode | Resolution | Colours/line | Notes |
|--------|------|------------|--------------|-------|
| M1 | Mode 1 | 256×192 | 2 (ink/paper per 8×8 cell) | ZX Spectrum-compatible attribute layout |
| M2 | Mode 2 | 256×192 | 2 (ink/paper per 8×1 cell) | Per-scanline attribute control |
| M3 | Mode 3 | 512×192 | 4 | High horizontal resolution, 2bpp |
| M4 | Mode 4 | 256×192 | 16 | Standard SAM mode, 4bpp |

When switching modes with content on the canvas, a dialog offers:
- **Convert** — attempts to translate the graphics to the target mode
- **Clear** — switches mode with a blank canvas
- **Cancel** — stays in the current mode

### Mode conversion behaviour

- **M4 → M3**: doubles each pixel horizontally to fill the 512-wide canvas, remaps to the 4 most-used colours
- **M3 → M4**: halves width, keeps existing colours in the first 4 CLUT slots
- **M4/M3 → M1/M2**: for each attribute cell, finds the two most-used colours and assigns ink/paper accordingly
- **M1/M2 → M4/M3**: maps each pixel's effective colour to the nearest CLUT slot
- **M1 ↔ M2**: pixel bits are preserved; attributes expand (M1→M2: each 8×8 attr becomes 8 rows) or collapse (M2→M1: takes the top row's attribute)

---

## The CLUT

The SAM Coupé uses a **Colour Look-Up Table** to map pixel values to palette entries:

- Mode 4: 16 slots (pixel values 0–15)
- Mode 3: 4 slots (pixel values 0–3)
- Modes 1 & 2: 8 slots (ink/paper values 0–7)

The **Active CLUT** panel on the right shows the current slot assignments. Each swatch
displays the SAM palette colour assigned to that slot.

- **Left-click** a CLUT swatch — set as the **foreground** drawing colour
- **Right-click** a CLUT swatch — set as the **background** colour
- **Shift-click** a CLUT swatch — reassign that slot by clicking any colour in the full palette below

The **CLUT indicator strip** to the right of the canvas shows a vertical bar for each slot,
colour-coded per scanline. White tick marks show exactly where line interrupts fire on each slot.
This gives an at-a-glance view of how many distinct colours are active at each position
down the screen.

---

## Colours and Palette

The **Colours** panel shows the current FG (foreground) and BG (background) colour swatches,
a **⇄ Swap** button to exchange them, and a **BG→FG** button that replaces every pixel on
the canvas matching the BG colour with the FG colour.

The **SAM Palette (128)** panel shows all 128 SAM colours as clickable swatches:
- **Left-click** — assigns a colour to the CLUT slot currently used as FG, and sets FG to that slot
- **Right-click** — same for BG

Tick the **Gradient** checkbox to reorder the 128 colours by HSV (hue, saturation, value):
black first, then all chromatic colours sweeping red → orange → yellow → green → cyan →
blue → purple → magenta, with greyscales at the end. Useful for picking closely related
shades.

---

## Drawing Tools

Select tools from the left toolbar or use keyboard shortcuts:

| Tool | Key | Description |
|------|-----|-------------|
| Pencil | P | Single-pixel drawing |
| Brush | B | Multi-pixel brush (size 1–16) |
| Line | L | Straight line between two points |
| Rectangle | R | Outline rectangle |
| Filled Rect | — | Solid filled rectangle |
| Ellipse | E | Outline ellipse |
| Filled Ellipse | — | Solid filled ellipse |
| Triangle | — | Outline triangle (3-click) |
| Filled Triangle | — | Solid filled triangle (3-click) |
| Bezier Curve | — | Quadratic bezier (3-click) |
| Flood Fill | F | Fills connected same-colour region |
| Gradient Fill | G | Fills region with CLUT colour gradient |
| Eraser | X | Paints with BG colour |
| Eyedropper | I | Click to pick FG; right-click to pick BG |
| Select | S | Drag to select rectangular region |
| Text | T | Click canvas to place text |

**Left-click** draws with FG colour; **right-click** draws with BG colour.

### Three-click tools (Triangle, Bezier Curve)

These tools require three separate clicks:
- **Triangle**: click point 1, click point 2, click point 3 — the triangle is drawn on the third click
- **Bezier Curve**: click start point, click end point, click control point — the curve bows towards the control point
- A dashed preview updates after each click. Press **Escape** to cancel at any stage.

### Gradient Fill

Select the Gradient Fill tool (G) to reveal the gradient options bar:

1. Click colour swatches in the **Stops** strip to add/remove them from the gradient (numbered in order)
2. Set the **Angle** (0–359°) or drag on the canvas in the direction you want the gradient to run
3. Adjust **Dither** (0–100%) — 0% = hard colour bands, 100% = maximum Bayer dither blend
4. Click inside a filled region to apply

The fill uses flood fill to find the connected region, then projects each pixel onto the gradient
axis and assigns colour stops using ordered (Bayer 4×4) dithering.

### Text Tool

Click on the canvas to place a text cursor, then type. Options in the tool bar allow selecting
system fonts, size, bold and italic. An 8×8 SAM ROM-style font is available as a fallback.

---

## Selection and Clipboard

1. Select the **Select** tool (S) and drag to mark a rectangular region
2. The selection toolbar appears above the canvas:
   - **✂ Cut** — removes the selection and places it on the clipboard
   - **⧉ Copy** — copies the selection to clipboard
   - **⧉ Mask** — copies with the current **BG colour as transparent** — when pasted, pixels matching the BG colour are not written, letting the destination show through
   - **↻ Rotate** — opens rotation controls (±1°, ±45°, apply/cancel)
   - **✕** — clears the selection

3. After copying, a floating preview follows the cursor. Click to place the pasted content.
   Press **Escape** to cancel a paste in progress.

The clipboard persists when switching between Screen 1 and Screen 2, making it easy to copy
elements from one screen to the other.

---

## Canvas Operations

Buttons in the left panel below the tools:

- **↔ Flip Horizontal** — mirrors the entire canvas left-right
- **↕ Flip Vertical** — mirrors top-bottom
- **↻ Rotate 90°** — rotates the entire canvas 90° clockwise

---

## Zoom and Navigation

- **− / +** buttons in the titlebar change zoom level
- **Fit** — zooms to fit the canvas in the available area
- **Mouse wheel** — zooms in/out
- The **Magnifier** panel (top of right panel) shows a 20× zoom around the cursor position

---

## Two Screens

The editor maintains **two independent screen buffers** — Screen 1 and Screen 2 — each with
its own pixel data, CLUT, mode, line interrupts, and undo/redo history.

Click the **Screen 1** or **Screen 2** tab above the canvas to switch. The canvas, CLUT panel,
and line interrupt list all update instantly.

The **→ Copy to other** button copies the entire current screen (pixels, CLUT, mode, line
interrupts) to the other slot. The clipboard is shared between screens, so select → copy on
Screen 1, switch to Screen 2, and paste to transfer graphics.

---

## Line Interrupts

The SAM Coupé hardware fires a colour interrupt at the end of a specified scanline, changing
one or more CLUT slot values. This allows more than 16 colours to appear on screen
simultaneously — one CLUT slot can hold a different colour in each horizontal band.

The **Line Interrupts** panel lists all current interrupt records. Each record specifies:
- **Y** — the scanline (0–191); the colour change takes effect from **Y+1**
- **Slot** — which CLUT slot (0–15) to change
- **Colour** — the new palette index for that slot

The SAM hardware supports a maximum of **127 line interrupt records** per screen. This limit
is enforced in the editor at all times.

### Adding interrupts manually

Click **+ Add interrupt** to open the guided wizard:
1. Click the desired scanline on the canvas preview
2. Select which CLUT slot to change
3. Click the new palette colour

### CLUT indicator strip

The thin strip to the right of the canvas shows, for each CLUT slot (each 2-pixel-wide column),
the colour that slot holds at every scanline. White tick marks indicate where interrupts fire.
This makes it easy to visualise the colour distribution down the screen.

---

## Importing Images (PNG, BMP, JPEG)

Click **Import IMG** to open the image import wizard. This converts any standard image to a
SAM SCREEN$ with automatic palette selection and optional line interrupt optimisation.

### Scale / Fit

| Option | Behaviour |
|--------|-----------|
| Fit (preserve aspect) | Letterboxes with black bars to fit within 256×192 |
| Stretch | Stretches to exactly 256×192 |
| Crop to centre | Centre-crops the source to fill 256×192 |
| Fit width (crop height) | Scales to full width, crops top/bottom |
| Fit height (crop width) | Scales to full height, crops left/right |

The **X/Y offset** sliders reposition the source within the cropped/fitted frame.
**Resample** selects bilinear (smoother) or nearest-neighbour (sharper pixel art).

### Adjustments

Brightness, Contrast, Gamma, Saturation, and per-channel Red/Green/Blue offsets are applied
before quantisation. Adjust these to compensate for the SAM's limited colour range — boosting
saturation and contrast often improves the final result significantly.

### Palette

- **Auto (best 16 from 128)** — scores all 128 SAM palette entries by how well they represent
  the image pixels, selects the best 16 by weighted frequency. Produces the most accurate result.
- **Use current CLUT** — forces mapping to whatever 16 slots are already loaded
- **SAM default palette** — uses the standard 16 Spectrum-compatible colours

**Max colours** (2–16) limits how many CLUT entries are used.

### Dithering

| Option | Description |
|--------|-------------|
| None | Nearest-colour only, no dithering |
| Floyd-Steinberg | Error diffusion — generally best for photographs |
| Ordered 2×2 / 4×4 / 8×8 | Bayer ordered dither — better for pixel art and gradients |

**Dither strength** (0–100%) blends between full dithering and nearest-colour.

### Line Interrupt Optimisation

Enable the **Line Interrupts** checkbox to allow more than 16 colours by swapping CLUT entries
at scanline boundaries:

- **Band height** — how many scanlines form each colour zone (1 = maximum granularity, one
  zone per scanline; higher values use fewer LI records)
- **Colours per band** — how many CLUT slots each band can use (up to 16)
- **Shared slots** — CLUT slots that remain constant across all bands (0 = all slots free
  to change; useful for locking a background colour)
- **Min improvement** — suppresses LI records for colour changes that are perceptually
  insignificant. Higher values use fewer interrupts. 5% is a good starting point.

The stats line shows how many LI records will be generated and whether the 127-record hardware
limit is exceeded. If the limit is hit, increase band height or reduce colours per band.

The import **preview** shows exactly what the image will look like after the hardware limit
is applied — the preview and the canvas after import are always identical.

Click **↺ Reset** to restore all wizard settings to defaults.
Click **Import to Canvas** to commit. The CLUT and line interrupts are written to the active screen.

---

## Importing SCREEN$ Files

Click **Import SCR** to load a native SAM SCREEN$ file (`.ss1`, `.ss2`, `.ss3`, `.ss4`).
The file's pixel data, palette, and line interrupts are loaded directly. The editor detects
the mode from the file extension or prompts if ambiguous.

---

## Exporting

- **Export SCR** — saves the current screen as a native SAM SCREEN$ binary. The correct
  extension (`.ss1`–`.ss4`) is added automatically based on the active mode. Line interrupts
  are included and truncated to 127 records if necessary.
- **PNG** — exports a rendered PNG image of the canvas at current zoom, with line interrupts
  applied. Useful for previews and sharing.

---

## Disk Image Browser

Click **Open Disk** to load an MGT or DSK disk image (must be exactly 819,200 bytes —
standard 80-track, 2-side, 10-sector format).

The browser lists **all files** on the disk, not just SCREEN$ files. For each file:

- **SCREEN$ files (type 20)** show their stored screen mode and a **Load** button
- **All other files** (CODE, BASIC, etc.) show a mode selector dropdown and a **Load as SCR**
  button — use this for files transferred to the disk via SAMdisk or similar tools, which are
  often stored as type 19 (CODE) rather than type 20 (SCREEN$)

**Saving to disk**: click **Save Screen$ to Disk** in the browser (or **Save→Disk** in
the titlebar after loading from a disk). Enter a filename (up to 10 characters). The current
canvas is written as a SCREEN$ file using the SAMDOS sector chain format, and the updated
disk image is downloaded for use with your emulator or physical hardware transfer tool.

---

## Undo / Redo

- **↩** or **Ctrl+Z** — undo (up to 80 steps per screen)
- **↪** or **Ctrl+Y / Ctrl+Shift+Z** — redo

Each screen has its own independent undo history.

---

## Keyboard Shortcuts

| Key | Action |
|-----|--------|
| P | Pencil |
| B | Brush |
| L | Line |
| R | Rectangle |
| E | Ellipse |
| F | Flood Fill |
| G | Gradient Fill |
| X | Eraser |
| I | Eyedropper |
| S | Select |
| T | Text |
| Ctrl+Z | Undo |
| Ctrl+Y | Redo |
| Ctrl+C | Copy selection |
| Ctrl+X | Cut selection |
| [ / ] | Decrease / increase brush size |
| G | Toggle grid (when no tool shortcut active) |
| Escape | Cancel multi-click tool / cancel paste / clear selection |
| Scroll wheel | Zoom in/out |

---

## Tips for SAM Graphics

- **Line interrupts are most effective in Mode 4** where you have 16 slots to work with.
  Use them to create colour gradients down the sky or background that would otherwise
  require more colours than a single CLUT allows.

- **Mode 2 is underrated** — per-scanline ink/paper gives you effective dithering between
  two colours on every row without consuming any line interrupt records.

- **When importing photos**, try Floyd-Steinberg dithering with 80–100% strength, saturation
  boosted to 120–140%, and contrast at +10 to +20. Enable line interrupts with band height 1,
  16 colours/band, and minimum improvement 5% for maximum colour depth.

- **The CLUT indicator strip** is the quickest way to spot wasted interrupt records — if
  two adjacent bands show the same colour in a slot, that interrupt is doing nothing useful.
  Increase the minimum improvement threshold in the import wizard to prune these.

- **Gradient fill with dithering** can simulate smooth colour transitions within a region
  using only the colours already in the CLUT — no extra palette entries needed.

- **Use two screens** when working on a scene with multiple layers or elements. Keep a
  clean background on Screen 2 and work on foreground elements on Screen 1, copying
  pieces between them as needed.

- **The BG→FG replace tool** is useful for quickly recolouring large areas — set the
  colour you want to change as BG, the new colour as FG, and click the button.

- **Masked copy** (⧉ Mask) is essential for compositing sprites over backgrounds.
  Set BG to black (or whichever colour surrounds your sprite), copy with mask, then
  paste over your background on the other screen.

---

*SAM Coupé SCREEN$ Editor — browser-based, no installation required*
