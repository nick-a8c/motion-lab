# Motion Lab

A single-file, WebGL motion & visual-effects lab that runs entirely in your browser — no build step, no dependencies, no install.

## ▶ Try it live

### **[nick-a8c.github.io/motion-lab](https://nick-a8c.github.io/motion-lab/)**

Just click and play — it opens the tool straight in your browser.

## What it is

Two pages, switched by the **`PATTERN | THERMAL`** toggle at the top:

- **Pattern** — a real-time wave-interference engine (a gradient strip pushed through two wave warps, mirror/kaleido symmetry, blur and a Colorama gradient LUT). 20 palettes, a live gradient editor with a mirror-loop, a field/mask system you can draw and drag, and a stack of post effects (halftone, dots, edge detection, chromatic aberration, film grade, grain…).
- **Thermal** — a page for standalone generative effects:
  - **Flow 1 — Column Radiate**: a glowing SDF column with a seamless, phase-drifting colour loop and a "core lock".
  - **Flow 2 — Blur → Colorama**: type a letter/word, shape a variable blur with four draggable pins, and cycle a looping colour gradient across it.

Everything is adjustable live, and looks can be exported as PNG.

## Run it locally

It's one static file. Either:

- **Open `index.html`** directly in a modern browser, or
- **Serve the folder** (recommended, avoids any `file://` quirks):

  ```bash
  python3 -m http.server 5353
  ```

  then visit **http://localhost:5353**.

## Tech

- A single `index.html` — HTML + CSS + JS in one file.
- **WebGL1** fragment shaders do the rendering.
- Zero dependencies, zero build tooling.

## License

Personal/creative project. Use it, fork it, remix it.
