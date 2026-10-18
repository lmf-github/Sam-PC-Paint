# Handover: scroll-friendly art checks for the SAM Coupé SCREEN$ Editor

**From:** the Mode 2 / Mode 4 scroller project
(`C:\Users\wowsk\Documents\Claude Code\Sam Coupe\Mode2`).
**To:** the SAM Coupé SCREEN$ Editor project
(`C:\Users\wowsk\Documents\Claude Code\Sam Coupe\Graphics editor\Sam-PC-Paint`).
**Date:** 2026-10-19.

**The feature:** a "scroll check" in the editor. It highlights, on the artwork, the places
where a picture breaks the rules that make it cheap (or possible) to scroll sideways on a
SAM. It also shows what each line would cost.

The rules, formulas and test values below were measured with the scroller project's
generator. A small reference implementation, `tools/scroll_rules.py`, computes every figure
in this document, so you can check the editor's results against it.

---

## 1. Background: why the art decides the cost

The scroller moves a band of the screen sideways, wrapping round at 256 pixels. It doesn't
redraw the band. Instead, for each step it runs pre-compiled code that stores only the
bytes that differ between the old and the new position.

So the cost of a frame is close to **the number of screen bytes that change per step**,
and that number comes straight from the art:

- **Colour edges along a line cost.** Each place where the colour changes from one pixel
  to the next sweeps across one or more bytes per step, and each of those bytes is stored.
- **Flat areas cost nothing.** Neither does a line that is all one colour.
- **Faster scrolling costs more per frame.** An edge sweeps more bytes per step.
- **Mode 2 adds one hard limit.** Each 8-pixel cell has only two colours, so a shift can
  make a cell that cannot be shown at all.

Measured on the project's own screens:

| Band | Speed | Byte stores a frame |
|---|---|---|
| `mode4.ss4` chevrons, lines 48–143 (mode 4) | 1 px/frame | 1,879 |
| `octo.ss2` hexagons, lines 8–43 + 148–183 (mode 2) | 2 px/frame | 1,300 |
| The same hexagons converted to mode 4 | 2 px/frame | 2,036 |
| The chevrons with every edge moved to an even x | 2 px/frame | 1,878 |

At 50 Hz the chevrons plus the mode 2 hexagons used about 57 % of a frame. With the hexagons
in mode 4 they used about 67 %. Moving the chevrons' edges to even x would let them scroll
twice as fast for the same cost.

**Time per store.** One store costs about 4.3 memory accesses, counting the code around
it. Accesses run at 96 a scan line in the border and about 64 a scan line during the
display. So **100 stores ≈ 4.5 border lines or ≈ 6.7 display lines**. A frame is 312 lines
(192 display, 120 border). At 50 Hz all the bands' stores must fit in one frame, each
band's while the beam is away from it.

---

## 2. What the check needs

| Input | Where it comes from | Notes |
|---|---|---|
| Mode | the screen's `mode` | Mode 4 and Mode 2 are covered. Mode 1 and 3 are out of scope. |
| Speed *p* | user setting, pixels a frame, 1–8 | Default 2. The rules depend on it. |
| Lines | user setting: all lines, or one or more ranges | A band is any set of lines that scroll together. |
| Slot per pixel | `pixelBuf` (Mode 4); ink/paper from `attrBuf` (Mode 2) | Mode 2 slot = ink or paper, + 8 if BRIGHT. |
| Colour per pixel | the line's effective CLUT (after line interrupts) | Use what `renderFull` uses. Compare palette indices. |

Everything works **one scanline at a time** and **wraps**: pixel *x* − 1 of *x* = 0 is
pixel 255. Lines never interact, so a line's result only changes when that line is edited.

**Mode 4 compares slots; Mode 2 compares colours.**
- In Mode 4 the screen byte holds slot numbers. Two slots showing the same colour are still
  different bytes, so they still cost a store.
- In Mode 2, whether a cell can be shown depends on the colours actually displayed. Compare
  the palette index each pixel shows on its line.

---

## 3. The rules

Each rule has an ID, the modes it applies to, a severity, what to highlight, and the fix
to suggest.

| ID | Modes | Severity | Highlight | Fix to suggest |
|---|---|---|---|---|
| **S1** cost heat | 2, 4 | info | every byte (Mode 4) or 8-px window (Mode 2) that changes when scrolled; a per-line cost in a gutter | fewer, longer edges; flat areas; detail along lines rather than across them |
| **S2** flat line | 2, 4 | info (good) | lines of one colour: cost 0 | — |
| **M4-1** edge on odd x | 4 | warning at even speeds | bytes whose two pixels differ | move the edge one pixel so it falls on an even x |
| **M4-2** uncommon stored value | 4 | low | stored bytes whose value isn't one of the band's 5 commonest | use fewer colour combinations at edges |
| **M2-1** three colours in a window | 2 | **error**: can't be shown | 8-pixel windows (at the alignments the speed visits) holding 3+ colours | colour areas at least 8 px wide; fine detail only between the same two colours |
| **R1** repeating pattern vs speed | 2, 4 | appears in S1 | (shown by S1) | a pattern repeating every *N* px is free only at speeds that are multiples of *N* |
| **G1** wrap seam | 2, 4 | info | pixels 0 and 255 of a line where the colours differ | make the picture join up across the screen width |

### S1: cost heat (the main overlay)

Show which parts of the art will be rewritten as the band scrolls, and a number per line.

- **Mode 4:** a byte is two pixels: byte *i* is pixels 2*i* and 2*i*+1.
  - *Even p:* the screen moves *k* = *p*/2 whole bytes per step. Byte *i* changes when it
    differs from byte *i* − *k*.
  - *Odd p:* the screen alternates between the art and the art shifted by half a byte, so
    there are two kinds of step (see §4).
  - The per-line cost is the **exact** number of stores the scroll code makes.
- **Mode 2:** the screen shows 8-pixel cells.
  - At each step it shows windows of the art starting at multiples of *g* = gcd(*p*, 8).
  - A window changes when its 8 colours differ from those of the window *p* pixels to its
    left.
  - Every changed cell needs at least one store, so the count is a lower bound on the
    engine's stores. It is close: for `octo.ss2` at 2 px it is 1,290.5 a frame, and the
    engine makes 1,290 pixel stores plus 10 attribute stores.
- **Display:**
  - a translucent tint over the changing bytes or windows;
  - a narrow gutter beside the canvas with one bar per line, its length the line's cost;
  - a total for the band: stores a frame, and the estimate in scan lines from §1.

### S2: flat lines

A line whose 256 pixels are all one colour (Mode 2) or one slot (Mode 4) never changes.
Show it as 0 in the gutter. Mention it in the help text: horizontal stripes and colour
changes **between** lines are free.

### M4-1: edges on odd x (Mode 4)

```
edge on an even x            edge on an odd x
|A A|A A|A A|B B|B B|        |A A|A A|A A|A B|B B|
2 px later:                  2 px later:
|A A|A A|A A|A A|B B|        |A A|A A|A A|A A|A B|
         1 byte: AA                   2 bytes: AA, AB
```

A byte whose two pixels differ, (b >> 4) ≠ (b & 15), is an edge on an odd x.

- At **even speeds** each such edge costs an extra store per step. The stored value is a
  two-colour byte that seldom sits in a register, so the store costs more too.
- At **odd speeds** the code alternates half-byte steps, so odd and even edges cost the
  same. Show M4-1 as a warning only when *p* is even.

Moving the chevrons' edges to even x took them from 2,622 to 1,878 stores at 2 px/frame.

### M4-2: uncommon stored values (Mode 4, low priority)

The scroll code keeps the five commonest stored values in registers (A–E), and a store
from a register is about a third cheaper.

1. Collect the values stored across the band (§4, `m4Stores`).
2. Rank them by count.
3. Flag stores whose value is outside the top five.

Expect this to fire mostly on odd-x edge bytes; M4-1 is the more useful message.

### M2-1: three colours in an 8-pixel window (Mode 2, error)

```
cells   |A A A A A A A A|C C C C B B B B|B B B B B B B B|   C stripe 4 px wide: fine at rest
+2 px   |B B A A A A A A|A A C C C C B B|B B B B B B B B|   middle cell: A, C, B - can't be shown
```

Scrolling *p* pixels a frame shows cells made of art pixels *x*0 … *x*0 + 7 for every
*x*0 that is a multiple of *g* = gcd(*p*, 8):

| *p* | Window starts checked |
|---|---|
| 1, 3, 5, 7 | every *x* (*g* = 1) |
| 2, 6 | every even *x* (*g* = 2) |
| 4 | multiples of 4 |
| 8 | multiples of 8: only the art's own cells, which the editor already keeps to two colours |

Any such window with three or more colours **cannot be displayed**. The scroller then shows
the nearest two colours, so the picture changes.

- **Highlight:** outline the window's 8 pixels in red, and fill the pixels of its least
  common colour (the ones that would be lost).
- **Fix:** keep colour areas at least 8 px wide, or put fine detail only between the same
  two colours. A stipple of two colours touching a third colour breaks the rule too (test
  line U3 in §5).

### R1: repeating patterns (shown by S1)

A pattern that repeats every *N* pixels looks the same after a move of *N* pixels, so only
its ends cost anything:

- A one-pixel stipple (A B A B …, repeating every 2 px) is free at 2, 4 or 6 px/frame, but
  changes completely at 1 or 3.
- Pairs (A A B B …, repeating every 4 px) are free at 4 and 8, but change completely at 2.

S1 shows this without a separate rule. Mention it in the tooltip when a line has many
edges but a low cost, or the reverse.

### G1: wrap seam (information)

The scroller wraps the line, so pixel 255 sits next to pixel 0. If their colours differ,
that is one more edge, counted in S1 like any other.

Show it as a hint, not an error. In `octo.ss2` the black edge columns make a visible
16-pixel gap that travels with the pattern, because the 40-pixel hexagons don't divide 256.

---

## 4. Formulas (JavaScript-flavoured pseudocode)

All indices wrap: `mod(a, n) = ((a % n) + n) % n`.

```js
// ---- shared ----
function edges(v) {                       // v = 256 slots (Mode 4) or colours (Mode 2)
  let n = 0;
  for (let x = 0; x < 256; x++) if (v[x] !== v[mod(x - 1, 256)]) n++;
  return n;
}

// ---- Mode 4 ----
function m4Bytes(s) {                     // s = 256 slots of one line
  const b = new Array(128);
  for (let i = 0; i < 128; i++) b[i] = (s[2 * i] << 4) | s[2 * i + 1];
  return b;
}
function halfShift(b) {                   // the line one pixel further right
  return b.map((_, i) => ((b[mod(i - 1, 128)] & 15) << 4) | (b[i] >> 4));
}
function oddEdgeBytes(b) {                // M4-1
  return b.flatMap((v, i) => ((v >> 4) !== (v & 15)) ? [i] : []);
}
// S1 / M4-2.  Returns the average stores per step, the art bytes that change,
// and a tally of the values stored.
function m4Stores(b, p) {
  const v = [b, halfShift(b)];
  const rs = (p % 2 === 0) ? [0] : [0, 1];        // even speed: one kind of step
  let total = 0;
  const changed = new Set(), values = new Map();
  for (const r of rs) {
    const r2 = (r + p) % 2, d = Math.floor((r + p) / 2);
    for (let i = 0; i < 128; i++) {
      const nv = v[r2][mod(i - d, 128)];
      if (v[r][i] !== nv) {
        total++; changed.add(i);
        values.set(nv, (values.get(nv) || 0) + 1);
      }
    }
  }
  return { stores: total / rs.length, changed, values };
}

// ---- Mode 2 ----
const gcd = (a, b) => b ? gcd(b, a % b) : a;
function m2BadWindows(c, p) {             // M2-1; c = 256 colours (palette indices)
  const g = gcd(p, 8), out = [];
  for (let x0 = 0; x0 < 256; x0 += g) {
    const set = new Set();
    for (let k = 0; k < 8; k++) set.add(c[mod(x0 + k, 256)]);
    if (set.size > 2) out.push(x0);        // highlight pixels x0 .. x0+7 (wrapping)
  }
  return out;
}
function m2ChangedCells(c, p) {           // S1 for Mode 2: changed cells per step
  const g = gcd(p, 8);
  let total = 0, steps = 0;
  const changedWindows = new Set();        // art window starts to tint
  for (let r = 0; r < 8; r += g) {
    steps++;
    for (let cell = 0; cell < 32; cell++) {
      const x0 = mod(8 * cell - r, 256);
      for (let k = 0; k < 8; k++) {
        if (c[mod(x0 + k, 256)] !== c[mod(x0 - p + k, 256)]) {
          total++; changedWindows.add(x0); break;
        }
      }
    }
  }
  return { cells: total / steps, changedWindows };
}
```

Band totals are the sums over the band's lines.

- **M4-2:** add up the `values` maps of all the band's lines, then take the top five.
- **Estimated time:** lines ≈ stores × 4.3 / 96 if run in the border, or stores × 4.3 / 64
  during the display.

**Cost of running the checks.** These functions, run in Node over both test screens at
every speed in §5, took about 50 ms in all. A whole screen at one speed is a few
milliseconds. Recompute only the edited lines while drawing, and the whole screen on
demand.

---

## 5. Test values

Every figure here comes from `python tools/scroll_rules.py` in the scroller project, and
the JavaScript in §4 reproduces all of them when run in Node. The Mode 4 store counts also
match `tools/art_cost.py`, which builds the real scroll code. The script prints rounded
figures; the exact values are given here.

### Synthetic lines (unit tests)

Mode 4 (one line of 256 slots):

| Line | Edges | Odd-x bytes | Stores at 1 / 2 / 4 px |
|---|---|---|---|
| T1: slot 1 for *x* < 100, else 2 | 2 | 0 | 2 / 2 / 4 |
| T2: slot 1 for *x* < 101, else 2 | 2 | 1 | 2 / 3 / 5 |
| T3: 1, 2, 1, 2, … (one-pixel stipple) | 256 | 128 | 128 / 0 / 0 |
| T4: 1, 1, 2, 2, … (pairs) | 128 | 0 | 128 / 128 / 0 |

Mode 2 (colours A, B, C; window starts listed):

| Line | 1 px | 2 px | 4 px | 8 px | Changed cells (any speed) |
|---|---|---|---|---|---|
| U1: A for *x* < 96, C 96–99 (4 px), B from 100 | 93, 94, 95 | 94 | none | none | 2.5, except 3 at 8 px |
| U2: A for *x* < 96, C 96–103 (8 px), B from 104 | none | none | none | none | 3 |
| U3: stipple A, B, A, B for *x* < 128, then C | 121–126, 250–255 | 122, 124, 126, 250, 252, 254 | 124, 252 | none | 17 at 1 px, 2 at 2/4/8 px |

### The project's screens

**`mode4.ss4`, lines 48–143 (Mode 4):**

- Edges: 2,254 in all, at most 24 a line. Lines 48 and 143 are flat (0).
- Odd-x edge bytes: 1,118.
- Per speed:

  | | 1 px | 2 px | 4 px |
  |---|---|---|---|
  | Stores a frame | 1,879 | 2,622 | 4,126 |
  | Distinct values stored | 11 | 10 | 10 |
  | Stores outside the 5 commonest values | 50 % | 35 % | 22 % |

- Each byte made its left pixel twice (every edge on an even x): 1,878 stores at 2 px.
- Per line at 2 px:

  | Line | 48 | 49 | 50 | 51 | 52 | 53 | 142 | 143 |
  |---|---|---|---|---|---|---|---|---|
  | Edges | 0 | 23 | 24 | 24 | 24 | 24 | 23 | 0 |
  | Odd-x bytes | 0 | 7 | 16 | 8 | 16 | 8 | 7 | 0 |
  | Stores | 0 | 23 | 32 | 24 | 32 | 24 | 23 | 0 |

**`octo.ss2`, lines 8–43 and 148–183 (Mode 2):**

- Edges: 1,682 in all, at most 109 a line (line 10, the stipple).
- Per speed:

  | | 1 px | 2 px | 4 px | 8 px |
  |---|---|---|---|---|
  | Three-colour windows | 134 | 62 | 24 | 0 |
  | Changed cells a frame | 1,203 | 1,290.5 | 1,471 | 1,858 |

  All the three-colour windows are on lines 10 and 181, half on each.
- Per line at 2 px:

  | Line | 8 | 9 | 10 | 11 | 12 |
  |---|---|---|---|---|---|
  | Edges | 14 | 14 | 109 | 26 | 24 |
  | Three-colour windows | 0 | 0 | 31 | 0 | 0 |
  | Changed cells | 16.25 | 13.25 | 14.5 | 25 | 22.75 |

- Cross-check with the real scroll code: 1,290 pixel stores + 10 attribute stores = 1,300 a
  frame at 2 px.

To regenerate any of these, from the scroller project folder:

```bash
python tools/scroll_rules.py mode4.ss4 48 143 --lines --speed 2
```

```bash
python tools/scroll_rules.py octo.ss2 8 43 148 183 --lines --speed 2
```

---

## 6. Suggested UI

- **Toggle.** A **Scroll check** button near Grid/Magnifier opens a small panel with:
  - Speed: 1–8 px/frame, default 2.
  - Lines: all, or ranges such as "8-43, 148-183".
  - A checkbox per rule: S1 heat, M4-1, M4-2, M2-1, G1.
- **Overlay.** Draw on a dedicated transparent overlay canvas stacked like
  `cursor-canvas`, so the artwork and undo history are untouched. Suggested colours:

  | Rule | Highlight |
  |---|---|
  | M2-1 | red: solid outline round the window, filled pixels for the colour that would be lost |
  | M4-1 | amber outline per byte |
  | M4-2 | thin amber underline |
  | S1 | blue tint, stronger where a pixel belongs to more changed windows |
  | G1 | small marker at *x* = 0 / 255 |

  Also give each a second cue besides colour (a dashed outline for errors, say), so they
  read in either theme.
- **Gutter.** One bar per line beside the canvas (next to the CLUT strip): its length is
  the line's stores (Mode 4) or changed cells (Mode 2), with a red dot on lines that have
  M2-1 errors. Hover shows the line's numbers.
- **Summary** in the panel:
  - stores a frame for the band;
  - the estimate in scan lines (border / display) and as a share of the 312-line frame;
  - counts per rule and the three worst lines.
- **Refresh** after each edit. Rerun only the lines touched: strokes already go through
  `renderRegion`, and `onMouseUp` already triggers `updateClutUsage`, a similar whole-screen
  scan that could host this.
- **Help text,** in short form:
  - Mode 4: put edges on even x and scroll in whole bytes (2, 4 px a frame).
  - Use fewer edges per line.
  - Mode 2: never three colours within 8 px; change colours between lines, not along them.
  - A pattern that repeats every *N* px is free when the speed is a multiple of *N*.

---

## 7. Caveats and scope

- **These costs are for this project's scroller,** which stores only changed bytes from
  straight-line code. A scroller that redraws everything costs the same whatever the art.
  One that flips pre-drawn pages per band costs memory instead.
- **Horizontal scrolling only.** Mode 1 (Spectrum layout) and Mode 3 (4 pixels a byte)
  aren't covered. Mode 3 would follow the Mode 4 rules with 4-pixel bytes, but is unmeasured.
- **Mode 2 counts are close but not exact.** The engine chooses among equivalent
  pixel/attribute encodings to save stores, so its total is a little above the changed-cell
  count. Use `tools/art_cost.py` in the scroller project for the exact figure.
- **Timings come from SimCoupe's contention model,** checked against SimCoupe's own
  rendering of the demo. Real hardware is the final test.
- **Line interrupts that change colours mid-line** can't be represented. Treat each line's
  effective CLUT as fixed for that line, as the editor's renderer does.

## 8. Reference files in the scroller project

| File | What it is |
|---|---|
| `tools/scroll_rules.py` | These formulas, and the test values above |
| `tools/art_cost.py` | Builds the real scroll code for a band and reports the exact cost |
| `tools/gen_scroll.py` | The generator used by the demo disks; the Mode 2 encoding chooser is `fit_row` |
| `README.md` | The demo, its measured results and the Mode 2 / Mode 4 comparison |
