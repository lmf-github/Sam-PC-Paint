# SAM Coupé SCREEN$ Editor — Developer Handoff

This document covers the complete technical architecture of the editor, all SAM Coupé
hardware details that the implementation relies on, and known issues and future work.

---

## 1. SAM Coupé Screen Modes

The SAM Coupé has four screen modes, all rendering a 256×192 pixel display area.

### 1.1 Mode 1 — Spectrum Compatible

- **Resolution:** 256×192, 1 bit per pixel
- **Pixel data:** 6,144 bytes, ZX Spectrum non-linear layout
  - Address bits: `[15:14]=01, [13:11]=row_in_third, [10:8]=third, [7:5]=char_row, [4:0]=col`
  - Each byte = 8 horizontal pixels, MSB = leftmost. Pixel 0 = paper, pixel 1 = ink.
- **Attribute data:** 768 bytes at offset 6,144. 32×24 cells (8×8 pixels per cell).
  - `Bit 7=Flash, Bit 6=Bright, Bits 5-3=Paper (0-7), Bits 2-0=Ink (0-7)`
  - Ink/Paper values 0–7 are indices into the lower 8 entries of the CLUT.
- **Total:** 6,912 bytes + palette block

### 1.2 Mode 2 — Enhanced Attribute

- **Resolution:** 256×192, 1 bit per pixel
- **Pixel data:** 6,144 bytes at offset 0, linear (row 0 first, 32 bytes/row)
- **Attribute data:** 6,144 bytes at offset 8,192. 32×192 cells (8×1 pixels per cell).
  - Same attribute byte format as Mode 1 (Flash/Bright/Paper/Ink)
  - This gives per-scanline attribute colour control without consuming line interrupt records
- **Padding:** 2,048 bytes of zeros between pixel and attribute blocks
- **Total:** 14,336 bytes + palette block

### 1.3 Mode 3 — High Resolution

- **Resolution:** 512×192, 2 bits per pixel
- **Pixel data:** 24,576 bytes. Each byte stores 4 pixels as 2-bit pairs.
- **CRITICAL — Bit reversal:** Each 2-bit pair is stored with bits reversed:
  ```
  stored_byte_pair = ((pixel & 1) << 1) | (pixel >> 1)
  ```
  This reversal must be applied on **both import AND export**. Failing to do so produces
  a mirrored/corrupted display. See `buildExportBuffer()` and `doImport()`.
- **Colours:** 4 simultaneous colours from CLUT slots 0–3
- **Total:** 24,576 bytes + palette block

### 1.4 Mode 4 — Standard

- **Resolution:** 256×192, 4 bits per pixel
- **Pixel data:** 24,576 bytes. Each byte stores 2 pixels:
  ```
  byte = (left_pixel_clut_slot << 4) | right_pixel_clut_slot
  ```
- **Colours:** 16 simultaneous colours from CLUT slots 0–15
- **Total:** 24,576 bytes + palette block

---

## 2. The SAM Palette

The SAM Coupé has a master palette of **128 colours**, indexed 0–127.

Each palette index is a 7-bit value with the following bit layout:

```
Bit: 6    5    4    3    2    1    0
     GRN1 RED1 BLU1 BRGT GRN0 RED0 BLU0
```

The 3-bit per-channel indices are assembled as:

```javascript
r = ((i & 0x02))      | ((i & 0x20) >> 3) | ((i & 0x08) >> 3)
g = ((i & 0x04) >> 1) | ((i & 0x40) >> 4) | ((i & 0x08) >> 3)
b = ((i & 0x01) << 1) | ((i & 0x10) >> 2) | ((i & 0x08) >> 3)
```

Each 3-bit value indexes an intensity table:

```javascript
intensity = [0x00, 0x24, 0x49, 0x6d, 0x92, 0xb6, 0xdb, 0xff]
```

The **BRIGHT bit (bit 3) adds 1 to every channel's 3-bit index** — it is NOT a flat
multiplier. BRIGHT on a zero-intensity channel gives 0x24 (36), not 0. This is a common
source of confusion.

This formula is an exact port of `get_sam_palette()` from `scrimage/palette.py` and
matches SimCoupe's output. The precomputed RGB lookup table is `SAM_PALETTE_RGB` (a
128-entry Uint32Array built at startup by `buildSAMPalette()`).

---

## 3. The SCREEN$ File Format

All four modes share a common structure:

```
[pixel data]  [palette block]
```

The palette block always starts immediately after the pixel data:

```
Offset (from pixel data start):
+0      16 bytes  Palette A / CLUT (16 × palette index, one per CLUT slot)
+16      4 bytes  Mode 3 store A (HMPR state; unused in M4, write zeros)
+20     16 bytes  Palette B (duplicate of A in most files)
+36      4 bytes  Mode 3 store B
+40      4×N+1   Line interrupt records, terminated by 0xFF
```

Absolute byte offsets by mode:

| Mode | Pixel bytes | Palette block starts at |
|------|------------|------------------------|
| M1   | 6,912      | 6,912                  |
| M2   | 14,336     | 14,336                 |
| M3   | 24,576     | 24,576                 |
| M4   | 24,576     | 24,576                 |

---

## 4. Line Interrupts

### 4.1 What They Do

Line interrupts allow CLUT slot values to change mid-frame, giving more colours per screen
than the 16-slot limit. The hardware fires each interrupt at the end of a specified scan
line, making the new colour visible from the **next** line.

### 4.2 Record Format (4 bytes each)

```
Byte 0: Y line (0–191) — fires at END of this line, visible from Y+1
Byte 1: CLUT slot to change (0–15 for M4, 0–3 for M3)
Byte 2: New palette index for Palette A
Byte 3: New palette index for Palette B (usually same as A)
```

Records are stored sequentially in ascending Y order. A **0xFF terminator byte** follows
the last record.

### 4.3 The Y-1 Convention

**This is the most common source of bugs.** To make a colour change visible at the TOP of
scan line Y, the record must store `Y-1` in byte 0. The editor stores and renders using
this convention throughout:

- `setLineInterrupts()` / `addLineInterrupt()` — store raw Y value (already Y-1 if desired from line Y)
- `renderFull()` — applies records where `li.line === currentScanlineY - 1`
- `iwOptimiseLI()` — stores `line: bandStartY - 1` for each band boundary

### 4.4 Hardware Limit

Maximum **127 records** per screen file. Enforced by:
- `SAM_LI_MAX = 127` constant
- `setLineInterrupts(arr)` — always slices to 127
- `addLineInterrupt(li)` — returns false if at limit
- `writePaletteAndLI()` — truncates to 127 on export

### 4.5 Rendering

`renderFull()` processes line interrupts by maintaining a working copy of the CLUT that is
updated as each scanline is rendered:

```javascript
const sortedLI = [...lineInterrupts].sort((a, b) => a.line - b.line);
let liIdx = 0;
for (let y = 0; y < 192; y++) {
  while (liIdx < sortedLI.length && sortedLI[liIdx].line === y - 1) {
    workClut[sortedLI[liIdx].slot] = sortedLI[liIdx].palIdx;
    liIdx++;
  }
  // render row y using workClut
}
```

The same logic is duplicated in `spriteRenderFrame()` for the sprite preview.

---

## 5. MGT Disk Image Format

### 5.1 Physical Layout

MGT disk images are exactly **819,200 bytes** (80 tracks × 2 sides × 10 sectors × 512 bytes).

Sides are **interleaved per track**:
```
[track0_side0][track0_side1][track1_side0][track1_side1]...[track79_side0][track79_side1]
```

**Sector offset formula** (confirmed from SAMdisk `mgt.cpp`):
```javascript
function mgtSectorOffset(track, sector) {
  const side = track >= 128 ? 1 : 0;
  const t    = track >= 128 ? track - 128 : track;
  return (t * 2 * 10 + side * 10 + (sector - 1)) * 512;
}
```

Chain addressing: tracks 0–79 = side 0, tracks 128–207 = side 1.

### 5.2 Sector Chains

Each 512-byte sector uses bytes 510–511 as a chain pointer `[next_track, next_sector]`.
Both zero = end of chain. Usable bytes per sector = 510.

File data = concatenation of all sector bodies (bytes 0–509 of each sector in chain order).
The first 9 bytes of the concatenated data are the SAMDOS file header; the rest is the
actual file content.

### 5.3 SAMDOS File Header (9 bytes)

```
Byte 0:   File type (0=BASIC, 16=CODE, 19=CODE, 20=SCREEN$)
Bytes 1-2: Modulo (file_length mod 16384), little-endian
Bytes 3-4: Load address, little-endian (for CODE files)
Bytes 5-6: Unused (zero)
Byte 7:   Number of 16KB pages
Byte 8:   Start page number
```

Actual file length = `pages × 16384 + modulo`.

### 5.4 Directory Structure

Directory occupies tracks 0–3 (40 sectors × 2 entries per sector = 80 directory slots).
Each entry is 256 bytes:

```
Byte 0:     File type (low 5 bits used; 0 = free slot; 20 = SCREEN$)
Bytes 1-10: Filename (space-padded, no null terminator)
Byte 11:    Sector count MSB
Byte 12:    Sector count LSB
Byte 13:    Start track
Byte 14:    Start sector
Bytes 15-209: BAM (sector address map, 1 bit per sector)
Byte 221:   Screen mode, 0-based (0=M1, 1=M2, 2=M3, 3=M4) — SCREEN$ only
Bytes 237-238: Modulo, little-endian
Byte 239:   Number of pages
Byte 240:   Start page (code load position)
```

The screen mode at byte 221 uses 0-based indexing. The editor adds 1 on read
(`mgtScreenMode()`) and subtracts 1 on write.

### 5.5 BAM (Block Allocation Map)

Tracks 0–3 are reserved for the directory and are never allocated for file data. The BAM
uses one bit per sector, starting from track 4 (sector index 0 = track 4, sector 1 of
side 0). `mgtBuildBAM()` builds a usage bitmap by scanning all directory entries.

---

## 6. Architecture

### 6.1 Single-File Design

The entire application is `index.html` — a single HTML file with inline CSS and JavaScript.
No build step, no npm, no external dependencies. This was a deliberate portability choice.

File size: ~240KB, ~6,160 lines, 197 functions.

### 6.2 Two-Screen System

All per-screen state is encapsulated in a `Screen` object created by `makeScreen()`:

```javascript
{
  label,          // 'Screen 1' or 'Screen 2'
  mode,           // 1, 2, 3, or 4
  pixelBuf,       // Uint8Array(512 * 192) — always max-size
  attrBuf,        // Uint8Array(32 * 192)
  clut,           // Uint8Array(16) — CLUT slot → palette index
  clut3,          // Uint8Array(4)  — Mode 3 palette
  lineInterrupts, // Array of {line, slot, palIdx}
  fgColor,        // CLUT slot index
  bgColor,        // CLUT slot index
  undoStack,      // Array of {pixelBuf, attrBuf, clut, lineInterrupts} snapshots
  redoStack,
}
```

`screens[0]` and `screens[1]` hold the two screen objects. `activeScreenIdx` tracks which
is active. Global state variables (`pixelBuf`, `mode`, `clut`, etc.) are proxied to the
active screen via `Object.defineProperties()`:

```javascript
Object.defineProperties(window, {
  mode:     { get: () => S().mode,     set: v => S().mode = v,     configurable: true },
  pixelBuf: { get: () => S().pixelBuf, set: v => S().pixelBuf = v, configurable: true },
  // ... all per-screen state
});
```

This allows all existing code to use global variable names without modification.

### 6.3 Rendering Pipeline

`renderFull()` is the main render function. It:
1. Clears the main canvas
2. Applies CLUT state with LI updates per scanline
3. Dispatches to mode-specific pixel rendering
4. Calls `renderClutIndicator()` to update the CLUT strip on the right

`renderFullAttr()` handles the attribute modes (M1/M2) where ink/paper colours per cell
determine the displayed colour rather than a direct CLUT slot value.

The cursor canvas (`cursor-canvas`) is a separate transparent overlay used for:
- Tool previews during drag (line, rect, ellipse in-progress)
- Clipboard paste preview (floating selection)
- Selection box

### 6.4 Drawing Tools

All drawing tools share the same entry points:
- `onMouseDown(e)` — starts drawing, captures `drawSlot` (BG if right-click, FG if left)
- `onMouseMove(e)` — continues drawing using `drawSlot` (NOT `e.button`, which is
  unreliable during drag)
- `onMouseUp(e)` — finalises using `drawSlot`

**Critical fix:** `drawSlot` is captured at `mousedown` and used throughout the drag. Using
`e.button` during `mousemove` always returns 0, which was causing right-click drags to
draw in the foreground colour.

Multi-click tools (Triangle, Bezier curve) use a `multiClick` state machine. Each click
adds a point; the final click completes the shape. Escape cancels.

### 6.5 Selection and Clipboard

`selBuffer` holds the copied pixel data:
```javascript
{
  data,       // Uint8Array of CLUT slot values
  w, h,       // pixel dimensions
  attr,       // Uint8Array (modes 1/2 only)
  aw, ah,     // attribute cell dimensions
  maskColor,  // CLUT slot to treat as transparent (-1 if no mask)
}
```

`copyClip(masked)` — copies with or without mask. When `masked=true`, `bgColor` at the
time of copy is stored as `maskColor`. During paste, pixels matching `maskColor` are
skipped, leaving the destination canvas unchanged.

`enterPasteMode()` activates the floating paste preview on the cursor canvas.
`pasteClipAt(cx, cy)` commits the paste to `pixelBuf`.

### 6.6 Undo System

`pushUndo()` takes a snapshot of `pixelBuf`, `attrBuf`, `clut`, and `lineInterrupts`
before any destructive operation. Each screen has its own 80-entry undo stack. Redo stack
is cleared on any new drawing operation.

The undo snapshot uses `Uint8Array.slice()` for deep copies of buffers, and
`JSON.parse(JSON.stringify(...))` for the line interrupt array.

### 6.7 Image Import Wizard

The wizard pipeline:
1. **Scale/Fit** — `iwScale()` produces a 256×192 RGBA pixel array from the source image
2. **Adjust** — brightness, contrast, gamma, saturation, per-channel offsets applied
3. **Quantise** — `iwQuantise(pixels, w, h, n)` scores all 128 SAM palette entries and
   returns the best N unique entries
4. **LI Optimise** — `iwOptimiseLI()` runs a per-band palette optimiser with lookahead
   eviction to stay within 127 LI records
5. **Dither** — Floyd-Steinberg or Bayer ordered dither applied to the quantised palette
6. **Preview** — identical simulation to `renderFull()` to guarantee preview = output
7. **Commit** — `commitImgImport()` writes to `pixelBuf`, sets CLUT and LI records

### 6.8 Sprite Editor

Sprite mode adds:
- A grid overlay canvas (`sprite-grid-canvas`) positioned absolutely over the main canvas
  with `pointer-events: none` (drawing still works)
- A settings bar (compact 30px strip) between the screen tabs and canvas-wrap
- A draggable floating preview popup

**Full vs partial blocks:** `getSpriteGrid()` returns both `fullCols`/`fullRows` (blocks
that fit entirely within the canvas) and `cols`/`rows` (including partial edge blocks).
Only full blocks get a number and participate in animation. `spriteGetBlock(n)` returns
`null` for partial blocks.

**Grid re-render:** `refreshSpriteGrid()` is called from `resizeCanvases()` to keep the
grid overlay aligned when zoom changes.

### 6.9 Gamesmaster Sprite Format

Reverse-engineered from `bird.s`, `orb.s`, `explode.s` in the Gamesmaster directory,
confirmed against the Games Master PDF spec.

**File structure:**

```
[0]      = 0x7B (magic byte / SAMDOS file type)
[1-2]    = SSLO/SSHI = WDTH × LNGT (bytes per pixel section, LE word)
           Also serves as SAMDOS modulo field
[3-4]    = FSLO/FSHI = same value (also SAMDOS load address field)
[5-14]   = SPOKE runtime fields (zeros in saved files)
[15]     = WDTH = bytes per row = ceil(sprite_width_px / 2) for Mode 4
[16]     = LNGT = allocated height in rows
[17-19]  = GRPG/GROL/GROH = zeros
[20]     = FRMS = number of frames
[21-44]  = remaining SPOKE fields (zeros)
[45..]   = FRMS × (pixel_section + mask_section)
```

Each pixel/mask section = `WDTH × LNGT` bytes. Pixels: 4bpp packed nibbles (left=high,
right=low). Mask: 0xFF=both opaque, 0xF0=left opaque, 0x0F=right opaque, 0x00=both
transparent. CLUT slot 0 is treated as transparent when building masks.

**Key insight:** the SAMDOS 9-byte file header and the SPOKE system variable block share
the same bytes (the file type byte 0x7B is the same as the SPOKE magic check byte). This
means a `.s` file IS also a valid SAMDOS CODE file.

---

## 7. Known Issues and Outstanding Work

### 7.1 Confirmed Bugs

**GM Sprite mask generation** — The mask is currently derived from CLUT slot 0 = transparent.
This works for most sprites but may not match the original Gamesmaster intent if a sprite
uses slot 0 as a visible colour. A per-sprite "transparent slot" selector would be needed
for full correctness.

**Mode 3 bit reversal on GM sprites** — `buildGMSpriteBuffer()` currently reads from
`pixelBuf` without applying the Mode 3 bit reversal. If a sprite is drawn in Mode 3,
the output `.s` file will have incorrect pixel data. The fix is to detect `mode === 3`
and apply the reversal per byte. (Mode 4 is unaffected.)

**GM sprite pixel width** — `wdth * 2` assumes Mode 4 (2 pixels per byte). If Mode 3
support is ever added to sprite export, this formula changes.

**Disk write BAM alignment** — `saveGMSpriteToDisk()` searches for free sectors starting
from the beginning of the BAM. If the disk has fragmented free space, the chain might
allocate non-contiguous sectors in a suboptimal order. This is functionally correct but
could produce a fragmented disk image.

**Screen tabs layout** — The `sprite-bar` is inserted into the DOM dynamically by
`toggleSpriteMode()`. If `canvas-column` is restructured, the insertion logic in
`toggleSpriteMode` must be updated accordingly.

### 7.2 Missing Features / Planned Work

**Gamesmaster sprite — pixel-shift variant** — The PDF spec mentions that if a sprite
allows pixel X movement, "a second set of graphics and masks, shifted right by 1 pixel,
follows the first set." `buildGMSpriteBuffer()` does not generate this second set. A
checkbox "allow X pixel movement" would need to be added to generate the shifted copy.

**Gamesmaster sprite — SPOKE runtime fields** — Fields like XSPD/YSPD (velocity), XCRD/YCRD
(initial position), ANTY (animation type), COLF (collision flags), FLGS/FLG2 (behaviour
flags) are written as zeros. An optional "SPOKE configuration" panel would allow setting
these for sprites intended for direct use with the Gamesmaster engine.

**Mode 2 attribute handling in paste** — When pasting into a Mode 2 screen, the attribute
data paste correctly but uses the source screen's attribute layout. There is no per-paste
attribute blending.

**Text tool fonts** — The text tool uses browser canvas fonts. A SAM ROM font (bitmap) was
planned but not implemented. ROM font data would need to be embedded as a lookup table.

**Line interrupt wizard** — The existing wizard (`startLintWizard`) guides the user through
adding one LI record at a time. A "batch optimise" flow (re-run the import wizard's LI
engine on the current canvas) would be useful for iterative editing.

**Export to SimCoupe-compatible snapshots** — Currently exports only raw SCREEN$ files.
A `.SNA` snapshot export would allow direct loading into SimCoupe.

**Undo across screen switches** — Switching screens does not undo across both screens; each
screen's undo history is independent. This is correct behaviour but could be confusing.

**Zoom to cursor** — Mouse wheel zoom currently zooms to the top-left corner of the canvas.
Zooming to the cursor position would be more ergonomic.

**Drag to scroll** — When the canvas is larger than the viewport at high zoom, there is no
middle-mouse-button pan. Only scrollbar scrolling is available.

**GM sprite import — CLUT not restored** — When importing a `.s` file, the current CLUT is
used to interpret CLUT slot indices. The original sprite's intended palette is not stored
in the `.s` file (SPOKE palette fields are zeros). The user must manually set up the CLUT
to match the sprite's intended colours before importing.

### 7.3 Refactoring Opportunities

**Extract JS to separate file** — The single-file design is good for portability but makes
the codebase hard to navigate. A build step (simple concatenation) could split the JS into
logical modules while still producing a single deployable file.

**Sprite preview LI rendering** — `spriteRenderFrame()` contains a near-duplicate of the LI
rendering logic from `renderFull()`. Extracting a shared `buildScanlinesWithLI(y0, y1)`
helper would eliminate this duplication.

**`buildExportBuffer()` / `doImport()` symmetry** — These functions are the critical
import/export pair and contain mode-specific branches. They should be kept in sync; any new
mode or format addition must update both.

---

## 8. Testing Checklist

When making changes, verify:

- [ ] All four modes render correctly at zoom 1× and 4×
- [ ] Import SCR round-trip: export a file, re-import it, compare visually
- [ ] Mode conversion M4→M1, M1→M4, M4→M3, M3→M4
- [ ] Right-click drag draws in BG colour (not FG) throughout the drag
- [ ] Line interrupt preview matches rendered output exactly
- [ ] Undo/redo works for pencil, flood fill, paste, line interrupt add/remove
- [ ] Screen switch preserves both screens' state
- [ ] MGT save: open a disk image, save a screen, re-open the disk image and load it back
- [ ] GM sprite: export `.s`, load it in SimCoupe or compare bytes against a reference file
- [ ] GM sprite import: load `bird.s`, verify grid is 19×14, 4 frames placed correctly
- [ ] Sprite preview: animation plays through all full blocks only (no partial blocks)
- [ ] CLUT uniqueness removed: clicking the same palette colour twice into two different
      slots should work

---

## 9. Development Environment

The project was developed iteratively in Claude.ai chat with file editing via the bash
tool and Filesystem MCP (for reading local SAM Coupé source repositories).

**Syntax checking workflow:**

```bash
python3 -c "
import re
with open('index.html') as f:
    c = f.read()
js = re.search(r'<script>(.*?)</script>', c, re.DOTALL).group(1)
with open('/tmp/check.js', 'w') as f:
    f.write(js)
"
node --check /tmp/check.js
```

**Reference repositories consulted:**
- `SAMCoupe-GITs/samdos/` — SAMDOS assembly source (disk format, directory structure)
- `SAMCoupe-GITs/samdisk/src/types/mgt.cpp` — MGT disk image format reference
- `SAMCoupe-GITs/sam-coupe-technical-manual/techmanual.md` — Hardware reference
- `SAMCoupe-GITs/Gamesmaster/` — GM sprite files and spec PDF
- `scrimage/palette.py` — Authoritative SAM palette formula

---

## 10. File Format Quick Reference

| File type | Extension | Magic / type byte | Notes |
|-----------|-----------|-------------------|-------|
| SCREEN$ M1 | `.ss1` | SAMDOS type 20, dir byte 221 = 0 | 6,912 + palette bytes |
| SCREEN$ M2 | `.ss2` | SAMDOS type 20, dir byte 221 = 1 | 14,336 + palette bytes |
| SCREEN$ M3 | `.ss3` | SAMDOS type 20, dir byte 221 = 2 | 24,576 + palette bytes |
| SCREEN$ M4 | `.ss4` | SAMDOS type 20, dir byte 221 = 3 | 24,576 + palette bytes |
| GM Sprite  | `.s`   | First byte = 0x7B (123) | SPOKE header 45 bytes + frame data |
| MGT disk   | `.mgt` / `.dsk` | 819,200 bytes exactly | 80T × 2S × 10sec × 512 |
