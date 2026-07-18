# SAM Coupé Mode 4 Screen Format

## Overview

Mode 4 is the standard colour graphics mode of the SAM Coupé, providing a 256×192 pixel
resolution with 16 colours simultaneously on screen per scan line. Colours are selected via
a 16-entry Colour Look-Up Table (CLUT) which maps pixel values to entries in the SAM's
128-colour master palette.

---

## Pixel Data

The pixel data occupies the first **24,576 bytes** of the SCREEN$ file (bytes 0–24575).

Layout is strictly linear:

- Row 0 comes first, then row 1, through to row 191
- Each row is **128 bytes** wide (256 pixels ÷ 2 pixels per byte = 128 bytes)
- Each byte encodes **two horizontally adjacent pixels** as two 4-bit nibbles:

```
Byte value = (left_pixel << 4) | right_pixel
```

Each nibble is a CLUT slot index in the range 0–15. The **high nibble** (bits 7–4) is the
left pixel; the **low nibble** (bits 3–0) is the right pixel.

**Example:** a byte value of `0x4A` means the left pixel uses CLUT slot 4 and the right
pixel uses CLUT slot 10 (0xA).

All 192 rows × 128 bytes = 24,576 bytes of pixel data.

---

## Palette Block

Immediately following the pixel data at byte offset **24,576** is the palette block. This
block defines the CLUT contents and any line interrupt records.

### CLUT (Palette A) — bytes 24,576–24,591

The first 16 bytes of the palette block define the active CLUT — one byte per slot:

```
Offset 24576: CLUT slot  0  → SAM palette index (0–127)
Offset 24577: CLUT slot  1  → SAM palette index (0–127)
...
Offset 24591: CLUT slot 15  → SAM palette index (0–127)
```

Each byte is a palette index into the SAM's 128-colour master palette. The palette index
encodes the RGB colour using a 7-bit value with the following bit layout:

```
Bit: 6    5    4    3    2    1    0
     GRN1 RED1 BLU1 BRGT GRN0 RED0 BLU0
```

The actual RGB output for each channel is determined by a 3-bit index into an 8-entry
intensity table:

```
intensity = [0x00, 0x24, 0x49, 0x6D, 0x92, 0xB6, 0xDB, 0xFF]
```

The 3-bit channel indices are assembled as:

```
R = (index & 0x02)       | ((index & 0x20) >> 3) | ((index & 0x08) >> 3)
G = ((index & 0x04) >> 1)| ((index & 0x40) >> 4) | ((index & 0x08) >> 3)
B = ((index & 0x01) << 1)| ((index & 0x10) >> 2) | ((index & 0x08) >> 3)
```

The BRIGHT bit (bit 3) adds 1 to every channel's 3-bit index, brightening the colour
uniformly rather than applying a flat offset.

### Mode 3 Store — bytes 24,592–24,595

The 4 bytes immediately following the CLUT (offsets 24,592–24,595) are reserved for the
Mode 3 palette store (HMPR register state). In a pure Mode 4 file these bytes are present
but not used for display. They should be written as zeros or preserved if round-tripping.

### Palette B — bytes 24,596–24,611

A second copy of the CLUT (offsets 24,596–24,611), also 16 bytes. In standard use Palette B
mirrors Palette A. Line interrupt records reference both palettes simultaneously, allowing
the SAM to use separate palette sets for double-buffered or flicker effects. For most
purposes Palette B is identical to Palette A.

### Mode 3 Store B — bytes 24,612–24,615

A second Mode 3 store, mirroring the first. Again, zeros in a Mode 4 file.

---

## Line Interrupts

Line interrupt records begin at byte offset **24,616** (immediately after the two CLUT
copies and their Mode 3 stores).

Each record is **4 bytes**:

```
Byte 0: Y line number (0–191)
Byte 1: CLUT slot to change (0–15)
Byte 2: New palette index for Palette A (0–127)
Byte 3: New palette index for Palette B (0–127)
```

The records are stored sequentially with **no gaps or separators** between them. The list
is terminated by a **0xFF byte** in place of the first byte of what would be the next
record.

### Firing Behaviour

The Y line value stored in byte 0 of each record is the scan line **at the end of which**
the interrupt fires. The new colour becomes visible from scan line **Y+1** onwards. This
means:

- To change a colour that takes effect at the top of scan line 10, store Y = 9
- To change a colour visible from the very first line, store Y = 255 (fires before line 0)
  or place the desired colour directly in the initial CLUT

Records must be sorted in ascending Y order for correct rendering. Duplicate Y values for
different slots are permitted and both fire at the same line boundary.

### Hardware Limit

The SAM Coupé hardware supports a maximum of **127 line interrupt records** per screen.
This limit applies to the SCREEN$ file format — a 128th record cannot be stored. Within
this budget, a single CLUT slot can be changed on every scan line to produce a 192-entry
colour gradient down the screen using only that one slot.

### Example

A gradient sky fading from dark blue (palette index 1) at the top to lighter blue
(palette index 17) at the bottom using CLUT slot 0:

```
CLUT[0] = 1          (initial colour at top of screen)

Line interrupt records:
  Y=31,  slot=0, palA=2,  palB=2   (colour change kicks in at line 32)
  Y=63,  slot=0, palA=4,  palB=4   (colour change at line 64)
  Y=95,  slot=0, palA=8,  palB=8   (colour change at line 96)
  Y=127, slot=0, palA=16, palB=16  (colour change at line 128)
  0xFF                              (terminator)
```

---

## Complete File Layout Summary

```
Offset        Size      Content
────────────────────────────────────────────────────────
0             24576     Pixel data (256×192, 4bpp, 2 pixels/byte)
24576         16        CLUT / Palette A (16 × 1-byte palette indices)
24592         4         Mode 3 store A (unused in Mode 4, write as zero)
24596         16        Palette B (mirror of Palette A)
24612         4         Mode 3 store B (unused in Mode 4, write as zero)
24616         4×N+1     Line interrupt records (N records × 4 bytes) + 0xFF terminator
────────────────────────────────────────────────────────
Maximum N = 127, so maximum palette block = 40 + (127×4) + 1 = 549 bytes
Minimum file size (no line interrupts) = 24616 + 1 = 24617 bytes
Maximum file size = 24616 + 509 = 25125 bytes
```

---

## SAMDOS File Header

When stored as a file on a SAM Coupé disk (type 20 = SCREEN$), a 9-byte SAMDOS header
precedes the screen data:

```
Byte 0:   File type (20 for SCREEN$)
Bytes 1–2: Modulo — file length mod 16384, little-endian
Bytes 3–4: Load address, little-endian (typically 0x8000)
Bytes 5–6: Unused (zero)
Byte 7:   Number of 16KB pages
Byte 8:   Start page number
```

The screen mode is stored separately in the SAMDOS directory entry at byte 221 of the
directory record, as a **0-based** value: 0=Mode 1, 1=Mode 2, 2=Mode 3, **3=Mode 4**.

---

*SAM Coupé SCREEN$ Editor technical reference*
