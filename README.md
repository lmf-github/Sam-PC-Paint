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
- **Two independent screen buffers** — copy/paste between them
- **Undo/redo** — 80 steps per screen

---

## Repository Layout

```
sam-coupe-editor/
├── index.html              ← Complete self-contained editor (single file)
├── README.md               ← This file
├── docs/
│   ├── TECHNICAL.md        ← SAM Coupé hardware reference (screen modes, palette, SCREEN$)
│   ├── MODE4_FORMAT.md     ← Detailed Mode 4 byte-level format reference
│   ├── USER_GUIDE.md       ← End-user guide
│   └── HANDOFF.md          ← Developer handoff: architecture, known issues, roadmap
└── test-assets/
│   ├── test-mode4.ss4      ← Sample Mode 4 SCREEN$ file for testing import
│   └── bird.s              ← Sample Gamesmaster sprite file
```

---

## Quick Start

```bash
git clone https://github.com/YOUR_USERNAME/sam-coupe-editor
cd sam-coupe-editor
open index.html          # macOS
xdg-open index.html      # Linux
start index.html         # Windows
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

When editing:
1. Make changes to `index.html`
2. Syntax-check the JS: `node --check index.html` won't work directly; extract the script
   block first: `python3 -c "import re,sys; c=open('index.html').read(); js=re.search(r'<script>(.*?)</script>',c,re.DOTALL).group(1); open('/tmp/check.js','w').write(js)"` then `node --check /tmp/check.js`
3. Test in browser
4. Commit

---

## Licence

MIT — see LICENCE file.
