# Wave Lab — Handoff

A single-file, WebGL-based "motion lab". It now has **two pages**, switched by a top-center
`PATTERN | THERMAL` pill:
- **Pattern** — the original tool: gradient strip → Wave Warp ×2 → Mirror → Blur → Colorama,
  grown into a full generative + glitch playground (three-column UI). §1–§15 are all Pattern.
- **Thermal** — a newer page for building fresh effects from scratch, each isolated. Two so far:
  **Flow 1** (column-radiate) and **Flow 2** (shape → variable blur → Colorama). See **§16**.

Everything lives in **one file**: `index.html`. No build step, no dependencies.

---

## 1. Quick start

- **File:** `/Users/automattnick/Desktop/Claude Work/index.html` (fixed three-column layout).
  `index-floating-backup.html` is the prior floating-dockable-panel version, kept for reference.
- **Run it:** it's a static file. The launch config `wave-warp-lab` serves the folder on
  **port 5353** (`python3 -m http.server 5353`). With the Claude Preview MCP:
  `preview_start({ name: "wave-warp-lab" })` → then `preview_eval`, `preview_screenshot`,
  `preview_console_logs`.
- **⚠️ Preview viewport bug:** after `window.location.reload()` the headless preview
  sometimes collapses the window to **1px wide** (canvas renders fine, but screenshots are a
  sliver). Fix with `preview_resize({ width:1280, height:860 })`. Always resize after a reload
  before screenshotting.
- **Memory:** a running project memory lives at
  `~/.claude/projects/-Users-automattnick-Desktop-Claude-Work/memory/wave-warp-lab.md`
  (indexed in `MEMORY.md`). Keep it updated.

The default opening look is the user's tuned blue wave-interference (`PRESET()`); the page
opens with the left Presets/Looks rail and the right Settings rail both visible (see §8).

---

## 2. The original AE effect (context for the "why")

The whole thing began by replicating this AE stack, applied top→down:
1. **Source:** vertical rectangle 240×1260, gradient **black → white → black**.
2. **Blur** (Effect 4) — applied FIRST in the real stack, hence the source is treated as soft.
3. **Wave Warp 1** (semicircle, height 200, width 400, dir 180, speed 1).
4. **Wave Warp 2** (semicircle, height 200, width 100, dir 180, speed 1).
5. **Mirror** (center 600,600, angle 180).
6. **Colorama** (luminance → colour LUT; blue ramp).

**Two findings that matter if you touch the wave math:**
- AE **"Semicircle" is an *asymmetric* scallop** (rounded crest, sharp cusped trough), NOT a
  symmetric sine. A symmetric wave makes pointed lens/diamond shapes; the asymmetric scallop
  makes the correct crescents. Implemented as `(sqrt(1-(2f-1)²)-0.5)*2`.
- AE **"Wave Width" = span of ONE scallop** (one period). The shader phase is `c = y/width`
  (don't double it).

---

## 3. Architecture

### Rendering: 3-pass WebGL1 pipeline (in `render()`)
1. **Source pre-blur (CPU/GPU, conditional):** if `srcMode===1` (image/text/generator) and
   `blur>0`, the source texture is pre-blurred with `blurProg` (3 iterated separable passes
   into `srcBlurFBO`) → `sourceTex`. This is why image blur is smooth, not ghosted.
2. **Scene pass** (`prog`, the big `FS`) → `fboA`. Computes the wave engine (folds, kaleido,
   wave displacement, field warp/brighten, source sample, posterise, colorama LUT).
3. **Global blur** (optional, separable `blurProg`, `fboA↔fboB`) → then **streak blur**
   (optional, single 1D directional `blurProg` pass).
4. **Post pass** (`postProg`, `POST_FS`) → canvas. All the post-FX.

### Shaders (template strings near the top)
- `VS` — trivial full-screen-triangle vertex shader (attribute `a_pos`).
- `FS` — the **scene / wave engine**.
- `BLUR_FS` — 17-tap separable gaussian (constant `-8..8` loop). Reused for: global blur,
  streak blur, source pre-blur. (Loop bound MUST stay constant — WebGL1 forbids a uniform
  for-limit.)
- `POST_FS` — the post-FX stack.

### Programs & helpers
- `buildProgram(fsSrc)` compiles VS+fs. Programs: `prog`, `blurProg`, `postProg`.
- `drawQuad(p)` binds the shared quad buffer for program `p` and draws.
- `makeFBO(w,h)` → `{fb, tex}` (RGBA8, LINEAR, CLAMP). FBOs: `fboA`, `fboB` (**RW×RH**),
  `srcTmpFBO`, `srcBlurFBO` (SRC=**1024**).

### Responsive render resolution (`RW`/`RH` are dynamic, not 1200²)
`RW`/`RH` are `let`, not const. `resizeCanvas()` (called at the top of `render()` each frame,
no-op when unchanged) sets `canvas.width/height` = the `#stage` client size (capped at 1600 on
the long side), and **recreates `fboA`/`fboB` at that size**, so the effect fills the whole
center area between the rails at the real aspect ratio — no square letterbox, no stretching.
Consequences:
- The scene centers on `u_res*0.5` (was a hardcoded `600.0`); the gradient centers on the new
  `u_gradCenter` uniform (`P.gradCenter*RH`, default 0.5 → vertical centre).
- `u_mc` (mirror/wave centre) is passed scaled: `P.mcx/1200*RW, P.mcy/1200*RH` (Center X/Y
  sliders stay 0..1200 = 0..1 of the canvas).
- The px-based spatial controls are scaled to a **1200 reference** so the look is aspect-stable:
  `u_stripW=P.stripW*RW/1200`, `u_gradHalf=P.gradHalf*RH/1200`, `u_blur=P.blur*RW/1200`. The
  image-source pre-blur uses a fixed `SRC/1200` (not `/RW`) so it's independent of canvas size.

### Textures (by unit)
- **unit 0** `lutTex` — 256×1 Colorama LUT, rebuilt from `stops` by `updateLUT()`.
- **unit 1** `srcTex` — 1024² image / text / generator source (`srcCanvas`).
- **unit 2** `fieldTex` — 1024² composed Field (SVG shape × gradient mask).

### Uniform-location maps
- `U` (scene), `Ub` (blur), `Up` (post). To add a uniform: **(a)** declare it in the shader,
  **(b)** add its name to the matching map's array, **(c)** set it in the matching pass inside
  `render()`, **(d)** add a default to `PRESET()`.

### State
- `P` — the single mutable settings object. `PRESET()` returns the default `P`.
- Module vars NOT in `P` (runtime/view state): `playing`, `last`, `colorPhase`, `fxBypass`,
  `audioCtx/analyser/audioData/audioActive/audioLevel`, `srcRaw`, `fieldShapeDef`.
- `P.time` is the playback clock — excluded from Copy-settings and presets.
- `stops` — the Colorama gradient stop list (separate from `P`).

---

## 4. Scene shader (`FS`) — the wave engine

`main()` order:
1. `p` = top-left-origin pixel; sample **field** `fv` in screen space.
2. `fold` (mirror, `u_mon`/`u_mhon`/`u_ma`/`u_mc`) → `kaleido` (`u_kon`/`u_kseg`/`u_krot`).
3. `waveDisp` for Wave 1 + Wave 2 (+ perpendicular copies if `u_cross`); `u_radial` = ripples
   from center.
4. field distort: `disp += field warp` (vertical) + `refract`; then `sc = q - disp`, glitch the
   sample coord, `lum = sourceLum(sc)`. (Time offset feeds `te` into step 3's `waveDisp`.)
5. `lum = field brighten (screen)` → posterise (`u_post_on`/`u_post_lv`) → Colorama LUT
   (`u_con`/`u_cycles`/`u_cphase`, capped `min(lum,0.998)` — see §11) → field colour effects
   (cycle/hue/sat/invert). See **§7** for the full field-effect list.

- `waveFn(type, c)`: 0 **semicircle** (asymmetric scallop), 1 sine, 2 triangle, 3 square,
  4 sawtooth.
- `sourceLum(p)`: `srcMode 0` = analytic soft gradient strip (strip centred on `u_res.x*0.5`,
  vertical gradient centred on `u_gradCenter`, half-height `u_gradHalf`); `srcMode 1` = sample
  `srcTex` luminance. Render res is dynamic — see **§3** "Responsive render resolution".

---

## 5. Post shader (`POST_FS`)

`main()` order (all gated by `(fx && P.x_on)`, where `fx = !fxBypass`):
1. **Pixelate** (`u_pix*`) — quantize `suv`.
2. **Colour fetch** (priority chain): **Edge detection** (`u_edge_*`) → **Dot dither**
   (`u_dot_*`) → **Halftone** (`u_half_*`) → **Dots** (`u_dots_*`) → **Chromatic aberration**
   (`u_ca_*`, radial+directional R/G/B split) → plain.
3. **Film grade + vignette** (`u_grade_*`, `u_vig*`) — gain/crush/sat/cool-tint + vignette.
4. **Grain** (`u_grain*`).

Helpers: `hash21`, `bayer4`, `dotMask` (circle/square/triangle/plus by `u_dot_shape` int),
`dotDither` (Bayer-threshold dot screen, tint-by-effect or fixed colors), `halftoneFx`
(variable-size newsprint dots on a rotated `u_half_ang` screen, radius by darkness), `dotsFx`
(even grid, dot size by brightness, source colour kept), `edgeFx` (Sobel on luminance, colour
edges on black).

**Removed effects** (Post FX slimmed down): **Global blur** + **Streak blur** (were separate
`blurProg` passes in `render()`), **Low-poly**, **Color dither**, **Slice / block tear**,
**Data dashes**. Their uniforms, shader code, `Up`-map entries, `FX_SLIDERS` rows and PRESET
keys are all gone — plus the now-unused second FBO `fboB` (only the two blur passes used it).
The earlier **Glitch** and **Noise** were already gone.

---

## 6. Source system (Source panel)

- `srcMode`: **0** = analytic gradient strip; **1** = texture (from **Text** only).
- `srcCanvas` (1024²) is drawn by `srcFromText` (the sole remaining texture source; called once
  with `'WARP'` at load so the texture is never empty).
- **Removed:** the **Generators** (`srcFromBars/Gradient/Noise/Wavelab/City`, `SRC_GENERATORS`),
  **Upload image** (`srcFromImage`), and the `gen:` branch in `applyConfig` — all dead code,
  stripped in the tidy-up. The Source panel is now just **Text** + **Pixel sort**.
- **Pixel sort = a CPU bake** (true vertical span sort, `pixelSortColumns`). A fragment shader
  can't truly sort. `uploadSrcTexture()` keeps a clean copy in `srcRaw` then bakes the sort if
  `psort_on`; `resortSource()` re-bakes from `srcRaw` on threshold/toggle change.
- Orientation was fixed (`uv = p/res`, no Y-flip) so text isn't upside down.

---

## 7. Field system (Field panel) + Dot dither

The **Field** is a grayscale control layer = **SVG shape × gradient mask**:
- Shape: built-ins (`FIELD_SHAPES`: now just **Hexagon/Circle** — Star/Heart/Blob removed),
  **Upload SVG** (`parseFieldSVG` → Path2D; handles path + basic shapes, ignores
  transforms/groups), or **Draw shape** (see below).
  Fill or centered-stroke **Outline** (thickness), per-shape **Edge softness** (blur), size /
  rotation. **Position is now drag-on-canvas, not Pos X/Y sliders** (those were removed).
- **Draw shape** (`#fieldDrawBtn`): click points on the canvas to build a polygon; clicking
  within ~10px of the first point (≥3 pts) closes it. `finishFieldDraw()` bakes the points into
  a Path2D `d` in `FW` coords and sets `fieldShapeDef` + `f_px/f_py/f_size` so it lands as drawn.
- **Drag to move**: with the Field panel open and `field_on`, dragging on the canvas sets
  `f_px/f_py`. Both this and drawing run through the `#fieldOverlay` 2D canvas (a sibling of
  `#glcanvas` in `#stage`), driven by `updateFieldEdit()` each frame — which also renders the
  in-progress polygon and, when **Show mask** (`fm_show`) is on with a linear/radial mask, a
  translucent preview of `fMaskC`. **Gotcha:** the overlay must be `display:block` (not just
  `pointer-events:auto`) to catch real drags — a `display:none` element gets no pointer events.
- Gradient mask (`fm_type` none/linear/radial, rotation/softness/size/invert, **Show mask**).
- `renderFieldShape()` → `fShapeC`; `renderFieldMask()` → `fMaskC`; `buildFieldTex()`
  multiplies them → `fFieldC` → uploads to `fieldTex`.
- **One mask drives many effects** (each a 0-default amount slider in the Field panel, so they
  only act where the mask `fv` is bright — the saved look is unchanged until dialed up). All live
  in the scene `FS`, gated by `u_field_on`:
  - **Warp** (`field_warp`) — vertical source displacement.
  - **Refract** (`field_refract`) — displaces the sample coord along the mask's screen-space
    gradient (`fgrad()`), a glass/lens bend around the shape's edges.
  - **Glitch** (`field_glitch`) — pixelates + hashed horizontal tear on the sample coord.
  - **Time offset** (`field_time`, signed) — per-pixel time `te = u_time + fv*field_time*3` fed
    into `waveDisp`, so the masked region runs ahead/behind.
  - **Brighten** (`field_mod`) — screen-lift luminance.
  - **Hue shift** (`field_hue`, signed) — `hueRot()` Rodrigues rotation about the gray axis (post-LUT).
  - **Saturation** (`field_sat`, signed), **Invert** (`field_invert`), **Color cycle**
    (`field_cycle`, signed — shifts the Colorama LUT phase locally).
  - Wired via `FX_SLIDERS` + the Field panel "Field drives — distort / timing / color" groups;
    `P.gradCenter` aside, all default 0.
- **Dot dither** (Post FX → Stylize) renders the effect as shape-dots, density by brightness.

---

## 8. UI — fixed three-column layout ("Grainrad" skin)

As of the layout migration, the chrome is a **static three-column app**, not a floating-panel
window-manager. (The old dockable/bottom-tab version is preserved verbatim in
`index-floating-backup.html` if you ever need to diff or revert.)

- `#app` is a flex row: **`#leftrail` · `#center` · `#rightrail`**.
- **Left rail**: brand "Wave Lab", an Input status block, then the **Presets panel** in
  `#leftpresets` — a **New / Source** button row on top, the Showcase "Looks" list, Style chips,
  Surprise me, rndFx/`fxBypass` toggles, save/preset list, Copy settings — and a
  `Follow · About · Changelog` footer.
  - **New** (`#newBtn`): `applyConfig({...})` with waves/symmetry/field all off → a bare colored
    strip (Colorama kept) to build up from.
  - **Source** (`#srcViewBtn`): master effect bypass. Saves the current `SRC_VIEW_KEYS` toggle
    states, forces them off (Colorama stays) to reveal the colored strip, and restores the exact
    prior state on re-click. `srcBypass` is cleared by Reset/New so a stale restore can't fire.
- **Center**: a `.ctop` title bar (`● Wave Lab [WebGL]` + decorative export glyphs), the **confined**
  `#stage`/`#glcanvas` (bordered, `aspect-ratio:1/1`, fits the column), and a `.cbar` transport
  (**⏸/▶ · scrub · ↻ Replay · ↺ Reset** — same IDs as before, just relocated).
- **Right rail**: a single scrolling **Settings** column. Each former panel (`Waves · Symmetry ·
  Source · Color · Field · Post FX · Export`) is now a static **collapsible section** — click its
  `.phead` to toggle `.collapsed` (the `−`/`+` glyph is a CSS `::before`). Symmetry / Source /
  Field / Post FX start collapsed; the list is set in the layout shim. (**Audio panel removed** —
  see §12.)
- **Color panel** additions (earlier sessions): **20 palette chips** under a collapsible
  ▾ Palettes header (`#palHead`); a **Mirror loop** toggle (`P.mirror` — ping-pong LUT baked in
  `updateLUT()`, motion-only, preview left un-mirrored); and a richer **gradient stop editor**
  (draggable handles on `#gradBar`, editable %/HEX rows, ⇄ reverse, +/−). **Cycles** max 10,
  **Color flow** (`cps`) 0–0.5 / 0.005 step / 3-dp readout.
- **The whole window-manager is gone** — no `openPanel`/`positionNear`/`panelsByTab`/`orientBtn`/
  drag/pin/dock. It's replaced by a ~4-line shim (collapse toggles + a `railReset` that proxies
  `resetBtn.click()`). All panel markup (and every control ID) is unchanged, so the engine wiring
  and `syncAllControls()` work untouched.
- **Styling**: appended override block at the end of `<style>` (`/* Grainrad three-column layout
  override */`). Dark near-black + mono font; sliders are restyled to a thin grey line + dot
  (`input[type=range]` set to `appearance:none` with custom track/thumb — accent-color no longer
  applies); checkboxes muted grey.
- `fxBypass` ("Preview without Post FX", in the left Presets section) gates ALL post-fx for A/B — a
  view-only toggle that never changes settings.
- **Removed** by user request: the left Input dropzone (only `Input`/`Standby` status remains) and
  the centre title bar (canvas now starts at the top of `#center`).
- **On-canvas gradient handles** (`#gradedit` overlay inside `#stage`): when the **Source** section
  is open AND the analytic strip is the source (`srcMode 0`), two draggable square handles + a
  centre line appear on the strip. They set the gradient's height (symmetric, band stays centred).
  The handle sits at the **visible** colour edge, not the gradient's true endpoint: it's drawn at
  `GRAD_EDGE_F` (0.78) × the full `gradHalf` reach, because the LUT's dark tail (navy) fades into
  the near-black canvas before `lum` actually hits 0 — so a 1:1 handle floated out in the black.
  Drag inverts the factor (`gradHalf = dist / GRAD_EDGE_F`). `updateGradEdit()` runs each frame
  (positions them, clamps to stay on-screen, shows/hides); `_gradDrag()` handles the mouse drag and
  syncs the `Grad span` slider via the `ctl_gradHalf`/`val_gradHalf` ids that `slider()` now stamps
  on every built control.

**Still rough / polish levers**: the export glyphs are decorative; the title-bar font is a system
mono stack (not Grainrad's exact face); which sections start open is a one-line list in the shim.
`P.gradCenter` exists (movable gradient centre) but is currently pinned to 0.5 — the handles are
symmetric; wire it up if you want an off-centre gradient band.

---

## 9. Recipes — how to add things (IMPORTANT conventions)

**Add a uniform-backed effect slider:**
1. Add the `uniform` to the shader; reference it in `main()`.
2. Add its name to `U`/`Ub`/`Up` array; set it in the matching pass in `render()`.
3. Add a default to `PRESET()`.
4. Add the slider HTML (`<input type="range" id="x">` + `<span id="xV">`) to the right panel.
5. Add `['x', fmt]` to **`FX_SLIDERS`** (id-wired live + synced in `syncAllControls`).

**Add a toggle:** add the checkbox `<input type="checkbox" id="x_on">` and add `'x_on'` to the
checkbox-array list inside `syncAllControls` (sets `P.x_on`, `onchange`, stop-propagation).

**Add a slider with a side effect** (e.g. re-bake a texture): wire it manually like the
`psort` / `FSHAPE_SLIDERS` / `FMASK_SLIDERS` blocks (set `P`, update display, call the
re-render fn) AND sync it in `syncAllControls`.

**Add a source generator:** write `srcFromX()` that draws to `srcCanvas` then calls
`uploadSrcTexture()`; register in `SRC_GENERATORS`; it auto-becomes a chip.

**Add a showcase preset:** add `Name: ()=>({ ...PRESET(), <overrides>, stops:PALETTES.X,
gen:'Bars' })` to `SHOWCASE`, then list `Name` in the right group in `SHOWCASE_GROUPS`.
(`applyConfig` no longer has a `gen` field — the source generators were removed; see §6.)

**The golden rule:** `syncAllControls()` is the single source of truth that rebuilds every
control from `P` + `stops`. It's called by load, Reset, Randomize, load-preset, showcase, and
style. Anything you add must be synced there or it won't reflect on Reset/preset-apply.

---

## 10. Presets, palettes, randomizer

- `PALETTES` (**20**): Ocean (default blue), Chrome, Aurora, Fire, Sunset, Ink, Neon, Vaporwave,
  Cyberpunk, Matrix, Toxic, Infrared, Spectrum, Sepia, Ice, Mint, Gold, Ultraviolet, Sakura,
  Autumn. (Colorama chips under the collapsible ▾ Palettes header.)
- `SHOWCASE` — now **one merged `SHOWCASE_GROUPS` group, "Looks"** (12): Blue, Sonar, Cymatics,
  Chrome Ripple, Aurora Veil, Mandala, Inferno, Nebula, Copper, Halftone, Retro, Scanlines. The
  five glitch presets (Scan Static/Datamosh/Corrupt/Prism/VHS) and Faceted were removed; **Nebula**
  + **Copper** added; **Halftone** repointed to the new `half_on` effect.
- `STYLES` ("build from"): Circle/Smooth/Sharp/Square/Saw — layer a wave-shape onto the
  current look.
- Saved presets → localStorage `wwl_presets`; **Copy settings** → JSON (excludes `time`).
- **Randomizer** ("Surprise me"): gated by `rndFx` (localStorage `wwl_rndfx`). Sprinkles the
  surviving post-fx (pixelate / halftone / dots / edge / grain / dot-dither) tastefully.

---

## 11. Gotchas / non-obvious

- **WebGL1 constraints:** loop bounds must be compile-time constants; `bool` uniforms are set
  with `uniform1i`; there's one `int` uniform (`u_dot_shape`).
- **`preserveDrawingBuffer:true`** is set on the GL context (needed for WebM record +
  reliable screenshots).
- **Preview viewport collapse to 1px** after reload — `preview_resize` to fix (see §1).
- **Chroma fringes appear even on grayscale** because R/G/B are sampled at different x — no
  colour source needed for chromatic aberration to read.
- Field SVG parser ignores `<g>`/transforms — fine for simple shapes; complex SVGs may need
  flattening.
- **Overlay pointer events need `display:block`** — `#fieldOverlay` (drag-to-move / draw) only
  catches real mouse events when it's `display:block`; `pointer-events:auto` on a `display:none`
  element does nothing. (This is why the first drag-to-move build silently failed — programmatic
  `dispatchEvent` worked, real dragging didn't.)
- **Colorama LUT wrap seam:** the lookup is `texture2D(u_lut, fract(min(lum,0.998)*cycles + phase))`.
  The `min(lum,0.998)` is essential — without it, the gradient's bright peak (`lum=1.0`) makes
  `fract(1.0)=0.0`, sampling the dark *first* LUT stop and drawing a 1px dark seam across the
  gradient centre. It only shows when a pixel lands exactly on the peak (odd backing height), so
  it flickers in/out with window size — easy to think it's "fixed" when you just resized.

---

## 12. Built but not fully exercisable headless

- **Audio reactive** was **removed** (panel, `enableMic`, analyser vars, and the `aBoost`
  wave-amplitude multiply in `render()`). If you want it back, it lived between the wave-height
  uniforms and a mic `getUserMedia` analyser.
- **WebM export** (`canvas.captureStream` + MediaRecorder, in Export panel): needs a real
  browser tab to download.

## 13. Roadmap / not built (with the "how")

- **Block-freeze / repeat datamosh** — needs a previous-frame **feedback FBO** (copy current
  scene into a history texture, sample it for frozen bands, reset every K frames). Flagged in
  research as the one non-drop-in piece.
- **True-colour source passthrough** (`srcMode 2`) — currently sources collapse to luminance →
  Colorama recolours them. A passthrough branch (`col = texture2D(u_srcTex, uv).rgb`) would let
  photos keep native colour for true datamosh.
- **Bloom** — researched (bright-pass from `fboA` + 2 `BLUR_FS` passes into a small FBO,
  additively combined in `POST_FS`); not built.
- UI polish on the three-column layout: real export icons (currently decorative glyphs), a
  closer-matching mono font, tuning which Settings sections start open.
- **Gradient handles → independent top/bottom**: `P.gradCenter` (movable centre) is already
  plumbed (uniform `u_gradCenter`, default 0.5) but the handles are currently symmetric (both set
  `gradHalf`, centre pinned). Wire each handle to move its own endpoint → set `gradCenter` +
  `gradHalf` from the two points for an off-centre band.
- **`GRAD_EDGE_F` exactness**: the handle sits at `0.78 ×` the gradient reach to land on the
  *visible* colour edge — tuned for the default dark-navy palette. For palettes with a brighter
  bottom stop it'll read slightly off; could be computed from the actual LUT instead of a constant.
- **Field Glitch in post space**: the field-driven Glitch currently pixelates the *source* coord
  (pre-Colorama). Wiring `fieldTex` into `POST_FS` would let it glitch the final coloured image
  (and let the field locally gate any post-FX).

There is a completed research workflow output (datamosh decomposition + GLSL recipes) saved at
`tasks/wgfxiiu5w.output` in the session dir if the glitch internals need revisiting.

---

## 14. Verification workflow (how this project is tested)

No automated tests — verify in the live preview:
1. After any shader edit: `preview_console_logs({ level:"error" })` — a shader compile error
   shows here (the page throws on `LINK_STATUS`/`COMPILE_STATUS`).
2. Behavior: `preview_eval` to set `P.*`, call `render()`, and `gl.readPixels` a grid of
   points into a string "hash"; compare before/after to confirm an effect changed the output.
   (Pattern used throughout this project.)
3. Visual: `preview_resize` then `preview_screenshot`. For panels, open via
   `document.querySelector('.tab[data-tab="..."]').click()` or set `.panel.open`.
4. Always **Reset** to the default at the end and confirm `P.w1h===206`, `P.blur===153`,
   `srcMode===0` (clean default).

---

## 15. Working style (the user's preferences)

- Surface ambiguity and ask before guessing on decisions that are genuinely theirs; otherwise
  pick a sensible default and proceed.
- Brief replies; plan multi-step work as `Step → verify`.
- The collaboration pattern that worked: **prototype new ideas in a chat sandbox**
  (the `show_widget` Canvas2D/SVG widgets) first, iterate to confirm the behavior, *then* port
  into `index.html`. Several features (line-field, datamosh, the Field+dither system) were
  proven this way before porting.

---

## 16. Two-page architecture (Pattern | Thermal)

Everything in §1–§15 is the **Pattern** page. A newer **Thermal** page hosts fresh effects,
each fully isolated so they can't destabilise Pattern.

### The mode switch
- `#modebar` (fixed top-center pill) with `#modePattern` / `#modeThermal`. `setMode('pattern'
  |'thermal')` toggles the `body.mode-thermal` class; CSS hides `#app` and shows `#thermal`
  (both full-viewport). Default is Pattern.
- **Pattern is untouched** — it's the entire existing `#app`, just show/hidden. Its loop early-
  returns while Thermal is active (`if(body.classList.contains('mode-thermal')) return`) so the
  hidden canvas doesn't render. `resizeCanvas` floors at 64px so a `display:none` `#stage`
  can't break it.

### Thermal layout & effect multiplexing
- `#thermal` = **`#thermLeft` (effects list) · `#thermStage` (canvases) · `#thermPanel`
  (control panels)**.
- Shared state `const THERM={eff:'flow1'}`. `setThermEffect(eff)` sets `THERM.eff`, toggles the
  `.theff.on` chip, and shows/hides `.effcanvas[data-eff]` (in `#thermStage`) + `.effpanel
  [data-eff]` (in `#thermPanel`) via the `hidden` attribute.
- **Each effect is its own IIFE with its own WebGL context** (`flow1Canvas`, `flow2Canvas` →
  separate `getContext('webgl')`). Its render loop is gated:
  `if(body.classList.contains('mode-thermal') && THERM.eff==='flowN'){ …render… }`. Each keeps
  its own state object, gradient editor, LUT texture and `requestAnimationFrame` loop. Nothing
  is shared between effects except the `THERM` flag and the mode class → total isolation.
- **Add Flow 3**: a `<button class="theff" data-eff="flow3">` in `#thermEffects`, a
  `<canvas class="effcanvas" data-eff="flow3" hidden>` in `#thermStage`, a `<div class="effpanel"
  data-eff="flow3" hidden>` in `#thermPanel`, and a new IIFE gated on `THERM.eff==='flow3'`.

### Shared "thermal color" machinery (both flows use it)
The Thermal effects standardise on a **circular Colorama** that Pattern does *not* use:
- **`sampleStops` is circular** — last stop blends back into the first (a looped palette), so the
  gradient editor's wrap segment is a real blend, not a seam.
- LUT texture wrap = **`REPEAT`**, sampled with **plain `fract`** (no `min(lum,0.998)` cap — that
  cap is a Pattern-only seam workaround; unnecessary here because the palette itself loops).
- **Phase = drift + manual offset**: `u_phase` advances over time (a `speed`/Drift control),
  `u_offset` is a static manual Phase. `t = value*cycles + u_phase*w + u_offset`.
- Each effect has its own gradient editor (draggable stops on a `#*_gbar` canvas + `#*_gstops`,
  dbl-click to add, − stop), duplicated per module (prefixed `th_`/`f2_`).

### Flow 1 — "Column radiate"
- Scene: an analytic **SDF field** of stacked discs + capsules + a base flare (`disc`/`capsule`
  smooth-min bulges) forming a glowing column, all in the fragment shader.
- Colour: the circular Colorama above. **Core lock** (`u_lock`) is the notable extra — the
  either/or from the AE reference: `lock<0.005` → one-way drift (`raw`); `lock>0` → the drift
  weight fades out above the threshold (`1 - smoothstep(lock±.08, f)`) **and** drift switches to
  time **ping-pong** (`|…*2-1|`), because a pinned core with one-way drift accumulates a seam.
- Controls: gradient · Band cycles · Drift · Phase · Core lock · Blur (SDF softness) · Bar glow ·
  Base flare · Export PNG · Reset. Adapted from `~/Desktop/column-radiate.html` (math only).

### Flow 2 — shape → variable blur → Colorama
The AE pipeline `A2 → Blur Comp (4-Color Gradient) → Adj.1 (Compound + Lens Blur) → Adj.2
(Colorama + Phase Shift)`, ported as:
1. **Source**: typed text (default "A") drawn white-on-black to a **1024² POT canvas** →
   `srcTex` with **mipmaps** (`generateMipmap`, `LINEAR_MIPMAP_LINEAR`). Square+POT so mip-blur
   works; the display samples it letterboxed (`uv = (p.x/asp, (p.y-0.5)/asp+0.5)`) to keep the
   letter's proportions.
2. **Blur map** (= AE's 4-Color Gradient): 4 pins, each a **grayscale value** (default 2 white =
   blur / 2 black = sharp), smoothly blended by inverse-distance weighting (`bmap`, `u_blend` =
   IDW power). **Show blur map** (`f2_showMap`) renders the grayscale field + pin rings; drag the
   pins on the canvas (only when the map is shown). Pin value buttons in the panel.
3. **Variable blur**: per-pixel radius = `blurMap * blurMax`; the shader picks a **mip LOD**
   (`log2(radiusInTexels)`) and does **one trilinear fetch** = smooth, cheap mip-blur.
4. **Colour**: the blurred luminance → the circular Colorama + phase drift.
- Baked defaults (the user's tuned look): `blurMax 1.0, blend 3.4, cycles 1, speed 0.576,
  phase 0`, pins 2-white/2-black, a fire ramp (dark→orange→cream peak→orange→dark loop).
  The ramp hexes were **estimated from a screenshot** — verify/adjust against the intended look.
- **Known limitation / next step**: the mip-LOD blur is box-flavoured and can look slightly
  chunky / show faint edge aliasing at low blur on hard glyph edges. The user flagged this.
  The proper fix (proposed, **not built**): a **Gaussian blur pyramid** — pre-blur the source
  into N FBO levels with a separable Gaussian, then interpolate between levels by the blur
  amount in the final shader. That replaces the single mip fetch with a true, smooth,
  alias-free variable blur.

### Reference files
- `~/Desktop/column-radiate.html` — the standalone AE-derived effect Flow 1 was adapted from
  (kept for the math; scene/UI were re-done in our style).
