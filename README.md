# SAM Coupé SCREEN$ Editor

A browser-based pixel graphics editor for the SAM Coupé home computer, supporting all four
native screen modes, MGT/DSK disk image access, image import with palette quantisation and
line interrupt optimisation, and a Gamesmaster sprite editor.

**No build step, no dependencies, no server required.** Open `index.html` in any modern
browser.

---

## Features

- **All four SAM screen modes** — M1 (Spectrum-compatible), M2 (8×1 attribute), M3 (512×192
  4-colour), M4 (256×192 16-colour)
- **Full drawing toolset** — Pencil, Brush, Line, Rectangle, Ellipse, Triangle, Bezier curve,
  Flood fill, Gradient fill, Eraser, Eyedropper, Select/cut/copy/paste (with mask mode),
  Text tool
- **128-colour SAM palette** with HSV gradient view
- **CLUT editor** with per-slot reassignment and per-scanline indicator strip
- **Line interrupts** — add, edit, remove; live preview; CLUT indicator strip
- **Image import wizard** — scale/fit, adjust, quantise, dither, LI optimise
- **MGT/DSK disk browser** — load and save SCREEN$ files directly to disk images
- **Sprite Editor mode** — configurable grid overlay, animation preview
- **Gamesmaster sprite format** — import and export `.s` sprite files, save to disk
- **Two independent screen buffers** — copy/paste between them; each tracks its own filename,
  shown beside the screen tabs
- **Undo/redo** — 80 steps per screen

---

## Repository Layout

```
Sam-PC-Paint/
├── index.html              ← Complete self-contained editor (single file)
├── README.md               ← This file
├── LICENCE                 ← MIT licence
└── docs/
    ├── TECHNICAL.md        ← SAM Coupé hardware reference (screen modes, palette, SCREEN$)
    ├── MODE4_FORMAT.md     ← Detailed Mode 4 byte-level format reference
    ├── USER_GUIDE.md       ← End-user guide
    └── HANDOFF.md          ← Developer handoff: architecture, known issues, roadmap
```

---

## Quick Start

```bash
git clone https://github.com/lmf-github/Sam-PC-Paint
cd Sam-PC-Paint
start index.html         # Windows
open index.html          # macOS
xdg-open index.html      # Linux
```

Or just drag `index.html` into a browser window.

---

## Browser Compatibility

Tested in Chrome 120+, Firefox 121+, Safari 17+. Requires:
- Canvas API
- File API (FileReader, Blob, URL.createObjectURL)
- ES2020 (optional chaining, nullish coalescing)

---

## Contributing

The entire application is a single `index.html` file containing inline CSS and JavaScript.
This was a deliberate choice for portability — no build toolchain, no npm, no bundler.

The workflow is: edit `index.html`, test in a browser, commit.

`node --check` cannot read `index.html` directly. To syntax-check the JavaScript, extract the
script block to a temporary file first:

```bash
python3 -c "import re; open('/tmp/check.js','w').write(re.search(r'<script>(.*?)</script>', open('index.html').read(), re.DOTALL).group(1))"
node --check /tmp/check.js
```

This is optional — it needs both Python and Node installed, and neither is required to run or
develop the editor. Loading the page and watching the browser console catches the same errors.

See [docs/HANDOFF.md](docs/HANDOFF.md) for the architecture, the SAM hardware details the
implementation depends on, known issues, and a testing checklist.

---

## Licence

MIT — see LICENCE file.
