# Scroll-timeline engine — reverse-engineered from edolus.com

Research date: 2026-08-16. Target: `site/index.html`.

This document records what edolus.com actually is, which of its mechanisms are worth
porting, and a staged plan to build them here. Everything in §1–§2 is extracted from the
live site's shipped source, not inferred.

---

## 1. What edolus.com actually is

Not a website with scroll animations. A **PlayCanvas WebGL application** — a game build
served as a page.

```html
<!doctype html><html><head>
  <script src="playcanvas-stable.min.js"></script>
  <script src="__settings__.js"></script>
</head><body>
  <script src="__modules__.js"></script><script src="__start__.js"></script>
  <script src="__loading__.js"></script>
</body></html>
```

That is the entire HTML document. One `<canvas id="application-canvas">` plus a handful of
absolutely-positioned overlay `<div>`s. There is no scrollbar, no page height, no sections.

### The build, measured

| | |
|---|---|
| Total transfer | **48.1 MB** across 97 requests |
| GLB models | 22.9 MB (ComputeTray 7.3 MB, Tesla 7.2 MB, Starlink 3.6 MB, Diamond 1.4 MB) |
| MP4 video | 11.5 MB (two clips, used as textures on world planes) |
| PNG / WebP / Basis textures | 10.9 MB |
| OGG audio | 2.6 MB — four separately-faded stems (melody, bass, instruments, …) |
| JS | 900 KB (603 KB engine + ~1 MB uncompressed game bundle) |
| Scripts | **107** |
| Assets | 303 (61 materials, 49 renders, 35 textures, 11 containers, 11 templates) |
| Render target | 2880×1800 at DPR 2 |

Attribution and intent are stated in the site's own ending credits:

> Edolus never existed. The experience did. Now imagine your own.
> `[ STUDIO@VERTEX3D.ASIA ]`

It is a studio showreel — a demo whose job is to prove capability. The budget makes sense
for that job. It does not make sense for a portfolio that a recruiter opens on hotel wifi.

**So: take the architecture, not the render budget.** Every number below is portable. None
of the assets are — those are Vertex3D's copyrighted work, and the narrative
(satellite → city → datacenter → chip → car) is theirs too.

---

## 2. The mechanisms worth taking

### 2.1 One virtual timeline, no scrollbar

`ScrollManager` owns a single float `progress ∈ [0,1]`. Wheel events are captured with
`preventDefault()` and integrated into it. The document never scrolls.

```js
totalScrollPx  = 24000   // wheel px for the whole 0→1 journey
touchScrollPx  =  4000   // finger travel for the same journey
lerpSpeed      =   0.10
keyStep        =   0.12  // Arrow/PageUp/PageDown
```

Smoothing is frame-rate independent — this exact form, not a naive `+= (t-c)*0.1`:

```js
const k = 1 - Math.pow(1 - lerpSpeed, 60 * dt);
current += (target - current) * k;
```

Input paths: `wheel` (with `deltaMode` 1 → ×18, 2 → ×innerHeight), `touchstart`/`touchmove`,
and `keydown`. Three input methods, one integrator.

### 2.2 Scroll-stretch bands — the tuning knob that matters

Progress values are *semantic* (0.28 is where the datacenter begins, forever). But some
chapters need more scrolling than others. Stretch bands decouple the two:

```js
stretchBands: [{ name, start, end, factor }]   // factor 1–10
```

The manager compiles these into a piecewise-linear map and keeps two coordinates: `raw`
(what the wheel moves) and `progress` (what every other script reads), converting with
`_rawToProgress` / `_progressToRaw`. Retuning pace never renumbers a single keyframe.

This is the single best idea in the codebase. Without it, every pacing change is a
find-and-replace across a hundred scripts.

### 2.3 Everything is a pure function of progress

The `scrollAppear` script is the template every animated thing follows:

```js
startProgress, endProgress,
startPosition, startRotation,  endPosition, endRotation,
moveTurnSplit = 0.35   // fraction of the window spent translating before rotating
```

Subscribe to `scroll:progress`, normalise into the local window, clamp, apply. No
timelines, no `.play()`, no internal state. Which is why scrubbing backwards is perfect and
why jumping straight to `progress = 0.64` renders a correct frame.

**The nuance:** *staging* is a pure function of progress. *Ambient life* is not — the logo's
"O" loop, text scramble, nebula drift, and audio all run on their own clocks. Keep the two
separate. Anything driven by scroll must be stateless; anything alive on its own must never
touch progress.

### 2.4 Chapter map

`SceneManager.SEGMENTS` — the whole narrative in seven rows:

```js
[{ name:'sceneSatellite',  start:0,    end:0.14 },
 { name:'sceneDiamondMap', start:0.14, end:0.28 },
 { name:'sceneDatacenter', start:0.28, end:0.54, preloadNextAt:1 },
 { name:'sceneChip',       start:0.54, end:0.57, dropPrevAt:1 },
 { name:'sceneUI',         start:0.57, end:0.71, dropPrevAt:1, preloadNextAt:0 },
 { name:'sceneCar',        start:0.71, end:0.85 },
 { name:'sceneEnding',     start:0.85, end:1,    dropPrevAt:1 }]
```

Note `sceneChip` gets 3% of the timeline and is one of the most memorable moments. Length
is not importance. Note also `preloadNextAt` / `dropPrevAt`: memory is a sliding window over
the timeline, not a single upfront load.

### 2.5 Shader warm-up — why it never stutters

Before the user can scroll, `SceneManager` walks the entire timeline invisibly:

- 2 sample points per segment (at 25% and 75% of each), 3 frames each
- frustum culling **disabled** during the walk, so nothing is skipped
- every shader compiles, every material uploads, every buffer allocates
- then a 2500 ms settle
- `window.__sceneWarmup = true` gates `ScrollManager.update()` for the duration
- progress reported to the loader via `warmup:progress` CustomEvent — the walk is 70% of
  the loading bar (`WARMUP_WALK_SHARE = 0.7`), the settle is the last 30%

This is why the site is glass-smooth and every naive scroll-3D site hitches on first reveal.
It costs nothing but patience at load.

### 2.6 Adaptive resolution

`optimizeRetina`, verbatim policy:

```js
maxDpr = 2      // "DPR 3 phones cannot show the difference but pay 2.25× the pixels"
minDpr = 1
DOWN_MS = 21, UP_MS = 17          // frame-time thresholds
WINDOW  = 90                       // rolling sample count
DOWN_COOLDOWN = 1200, UP_COOLDOWN = 6000
pixelBudget = 4_600_000 desktop / 3_000_000 mobile
```

Initial DPR is `min(devicePixelRatio, 2)`, then reduced further if `w*h*dpr²` exceeds the
budget. The settled value is cached in `localStorage` keyed by `screenW×screenH@dpr` for 24h,
so a returning visitor starts at the right resolution instead of re-discovering it.

Asymmetric cooldowns (fast down, slow up) mean it degrades instantly under load and only
recovers when it's confident. Port this whole block as-is.

### 2.7 Diegetic HTML labels

Text is never rendered into the canvas. It is a `<div>` whose position is recomputed each
`postUpdate` from a projected world point:

```js
camera.worldToScreen(entity.getPosition(), screen);
if (screen.z < 0) { el.style.display = 'none'; return; }   // behind camera
const r = canvas.getBoundingClientRect();
el.style.left = r.left + screen.x + labelMargin + 'px';
el.style.top  = r.top  + screen.y + 'px';
```

Live classes on the page: `.slabel`/`.sl-right` (object labels), `.sctext` (scene copy),
`.dlabel`, `.mdlabel`, `.tcf-1` (centre-fade), `#cursor-orbit`, `#cursor-hint`,
`#pc-overlay-root`, `#transition-overlay`, `#ending-credits`.

Text stays crisp at any DPR, selectable, in the accessibility tree, and styleable in CSS.
It costs one matrix multiply and two style writes per label per frame.

### 2.8 Text scramble without reflow

The vendored `TextScramble` wraps each character:

```html
<span class="ts-char">
  <span class="ts-ghost">A</span>          <!-- final char, visibility:hidden — holds width -->
  <span class="ts-over">#</span>           <!-- absolute, the scrambling char -->
</span>
```

The ghost reserves the *final* glyph's width, so the line never reflows while scrambling.
Per-character `start`/`end` frames are randomised (`start = rand(30)`, `end = start + rand(40)`),
28% chance per frame of picking a new dud char, alphabet `!<>-_\/[]{}—=+*^?#________` (the
weighted underscores are why it reads as "terminal" and not "confetti"). Frame advance is
wall-clock normalised (`speed * dt/16.667`), clamped to 100 ms.

### 2.9 Post-processing as a per-chapter preset

```js
PostFXManager.PRESETS = {
  SceneSatellite:  { bloomThreshold:0.4,  bloomIntensity:0.8, vignetteInner:0.5, vignetteOuter:1   },
  SceneDiamond:    { bloomThreshold:0.2,  bloomIntensity:1.4, vignetteInner:0.4, vignetteOuter:0.9 },
  SceneDatacenter: { bloomThreshold:0.3,  bloomIntensity:1.0, vignetteInner:0.6, vignetteOuter:1   },
  SceneChip:       { bloomThreshold:0.25, bloomIntensity:1.2, vignetteInner:0.5, vignetteOuter:1   },
  SceneUI:         { bloomThreshold:0.5,  bloomIntensity:0.5, vignetteInner:0.2, vignetteOuter:0.7 },
  SceneCar:        { bloomThreshold:0.35, bloomIntensity:0.9, vignetteInner:0.4, vignetteOuter:0.9 },
  SceneEnding:     { bloomThreshold:0.2,  bloomIntensity:1.6, vignetteInner:0.5, vignetteOuter:1   },
}
```

On `scene:changed`, GSAP tweens a plain proxy object over 0.9 s and writes the values through
on `onUpdate`. Chapters get distinct *grades*, not distinct *colours* — the palette barely
moves, the exposure does.

### 2.10 One camera, one world

`CameraController.KEYFRAMES` is a single path through a single stitched set:

```js
[{t:0,    y: 16, z:  8}, {t:0.14, y:-15, z: 8}, {t:0.24, y:-15, z:  8},
 {t:0.26, y:-20, z:  7}, {t:0.28, y:-23, z: 4}, {t:0.42, y:-23, z: -7},
 {t:0.54, y:-23, z: -7}, {t:0.71, y:-23, z:-7}, {t:0.75, y:-18, z: 15, rx:-20},
 {t:0.80, y:-10, z:110}, {t:1,    y:-10, z:110}]
```

There are no cuts. The chapters are rooms in one building and the camera walks through them.
Repeated values (0.54 → 0.71 all identical) are deliberate holds — the camera parks while
the *set* changes around it.

Mouse-look is windowed to specific chapters with edge fades so it never pops on:

```js
Look A: progress 0.28–0.40 (datacenter), pitch 3°, yaw 4°, smoothing 0.06, fade 0.01
Look B: progress 0.71–0.85 (car),        pitch 3°, yaw 4°, smoothing 0.06, fade 0.01
```

3° and 4°. Restraint is the whole trick.

### 2.11 Everything talks through one bus

A 30-line pub/sub (`on`/`once`/`off`/`emit`) with ~48 topics. Nothing holds a reference to
anything else.

```
scroll:progress · scroll:setProgress · scroll:intro
scene:goto · scene:changed
camera:moveTo · camera:snapTo
postfx:set · postfx:tween
transition:play · transition:register
cursor:set/show/hide · light:set/reset
experience:entered · experience:restart · warmup:done
audio:* (22 topics — one per cue)
```

Because progress is a broadcast float, `EventBus.emit('scroll:setProgress', 0.64)` from the
console jumps the entire experience to a correct frame. That is how the screenshots in this
research were captured, and it is a debugging superpower worth designing for deliberately.

### 2.12 Transitions are a DOM overlay, not a shader

`#transition-overlay` — `position:fixed; inset:0; z-index:9998; transform-origin:bottom center`.
GSAP tweens it. Four builtins: `fade-out`, `fade-in`, `wipe-up` (`scaleY 0→1`, `expo.inOut`),
`wipe-down`, and `slice` (wipe-up `expo.in` → **`onMidpoint` callback** → wipe-down `expo.out`).
The hard scene swap happens inside `onMidpoint`, hidden behind an opaque wipe. Default 0.7 s.

The most expensive-looking moments in the site are a `<div>` scaling on the Y axis.

### 2.13 Intro gate

A three-state machine — `locked` → `hold` → `open`. After the CTA click, the timeline is
pinned at 0 while the first 1200 px of scroll flies the hero object in. Overshoot spills into
real progress rather than being dropped. The user is scrolling from the first gesture; the
timeline just hasn't started yet.

### 2.14 Loader → header handoff

The splash logo is docked to the header logo's exact final metrics (`logoWidth: 156`,
`logoMarginTop: 40`) so the handoff has no reflow and no blink. The logo crosses white → dark
at `progress 0.72` with a `±0.02` fade — because the car chapter's background turns white.
Reversible on scroll-up.

---

## 3. What to deliberately not take

| | Why |
|---|---|
| PlayCanvas / an engine editor | Needs a scene-authoring workflow and a team. Wrong tool for a one-file site. |
| GLB models | 23 MB. A portfolio that costs 48 MB to open is a portfolio nobody opens. |
| Video-textured planes | 11.5 MB for two clips. |
| Four-stem adaptive audio + "EXPERIENCE WITH HEADPHONES" | Correct for a showreel. Hostile on a page someone opens at work. Ship muted-by-default or not at all. |
| A 30 s load with a percentage counter | Acceptable when the load *is* the pitch. Not here. |
| Their subject matter | Satellite → city → datacenter → chip → car is Vertex3D's story. Borrowing it makes this site a cover version. |
| `preventDefault()` on wheel, unconditionally | Hijacks the scrollbar, breaks find-in-page, breaks deep links. See §6.3 for the constraint. |

---

## 4. The translation

### 4.1 Narrative spine

edolus argues *"intelligence descends from orbit into a chip into a car."* The camera
literally travels down that hierarchy — the descent **is** the argument.

The equivalent argument for this site is already written in its own copy:

> "Coding agents fail quietly. They pass tests, open a PR, and leave a reviewer to
> reconstruct intent from a diff. Rig makes the agent show its work: every action is a
> replayable step with the context that produced it."

That is a descent too — from an instruction, down through a repo, into a sandbox, into a
diff, out to production. Same structural move, native subject matter.

Proposed chapter map, in the shape of §2.4:

```js
SEGMENTS = [
  { name:'instruction', start:0,    end:0.12 },  // a task enters — one point of light in the dark
  { name:'repo',        start:0.12, end:0.30 },  // the codebase as structure: a file/import graph
  { name:'sandbox',     start:0.30, end:0.52 },  // Rig. Execution in a contained cell.
  { name:'review',      start:0.52, end:0.58 },  // SIGNATURE. The loop catches what the model missed.
  { name:'shipped',     start:0.58, end:0.74 },  // the diff lands; selected work
  { name:'trajectory',  start:0.74, end:0.88 },  // FMD racing line — the other autonomy
  { name:'contact',     start:0.88, end:1    },
]
```

`review` gets 6% — deliberately, following `sceneChip`'s 3%. It is the shortest chapter and
the one the site is about. Everything before it is setup; everything after is consequence.

Existing sections map on cleanly: Hero → `instruction`, Selected work → `shipped`,
How I work → `repo`/`sandbox`, Rig case study → `sandbox`/`review`, About → `trajectory`,
Contact → `contact`.

### 4.2 What each chapter renders

No models. Everything procedural — points, lines, instanced quads, one shader each.

| Chapter | Visualization | Technique |
|---|---|---|
| `instruction` | A single lit point resolving out of a dark field; text scrambles in around it | Existing starfield canvas, promoted. `TextScramble` (§2.8). |
| `repo` | A file/import graph that assembles from scattered points into structure as you scroll | `plexus`-style: instanced points + a line buffer rebuilt on a distance threshold. Node count adaptive. |
| `sandbox` | A contained volume — an agent stepping through actions, each step leaving a trace | Instanced quads on a reflective plane, `infinityFloor` grid shader, volumetric cone. Cheapest scene in the set. |
| `review` | **Signature.** The diff arrives whole and correct-looking; the review pass sweeps it; one hunk fails and the sweep stops on it | Custom fragment shader on a full-screen quad. The scan line is a uniform driven by progress. |
| `shipped` | Horizontal card rail of the work index over a flowing light ribbon | `cardSystem` pattern — but 2D, over the existing `.wk-row` markup. |
| `trajectory` | A racing line optimising itself over a track outline, curvature-coloured | Line geometry, progress drives the optimiser's iteration count. Real data if the FMD numbers exist. |
| `contact` | Aperture close to a letterbox band | `#transition-overlay` scaleY (§2.12), exactly edolus's ending. |

Post-FX presets per chapter (§2.9) using the existing tokens — `--color-accent #9c1f22` and
`--color-accent-600 #a6791f` against `--color-bg #0c0b0a`. The palette does not change per
chapter; bloom and vignette do.

### 4.3 Stack

**Recommendation: three.js, procedural geometry only, no loaders.**

| Option | Cost | Ceiling |
|---|---|---|
| (a) 2D canvas + CSS 3D — current stack | +0 KB | No real depth, no post-FX, no camera dolly. Chapters 2–4 don't land. |
| **(b) three.js, procedural only** | **~150 KB gz** | Real perspective, instancing, bloom, one camera path. **Recommended.** |
| (c) PlayCanvas like edolus | 603 KB + editor + pipeline | Overkill; needs a scene-authoring workflow that doesn't exist here. |

With (b), tree-shaken to `WebGLRenderer` + `PerspectiveCamera` + `Points`/`LineSegments`/
`InstancedMesh` + `ShaderMaterial` and no `GLTFLoader`, the whole site lands around 300–400 KB
— roughly 1/120th of edolus, with the same architecture.

**§5 Phase 1 is stack-independent.** The engine is plain JS over the existing Lenis/GSAP and
ships useful on its own, so this decision does not block the start of work.

---

## 5. Build phases

Each phase ends in a shippable state. Do not start the next until the gate passes.

### Phase 1 — The engine (no new visuals, stack-independent)

Extract from `site/index.html` into `site/timeline.js`:

- `EventBus` — §2.11, 30 lines, same API
- `ScrollManager` — §2.1 + §2.2: virtual progress, frame-rate-independent lerp, three input
  paths, stretch bands, snap points, intro gate (§2.13)
- `SEGMENTS` + a `scene:changed` emitter — §2.4
- `scrollAppear(el, {startProgress, endProgress, from, to})` — §2.3, the primitive everything
  else is built on
- `optimizeRetina` — §2.6, verbatim, wired to the existing starfield canvas
- `TextScramble` — §2.8, replacing whatever the loader currently does

Re-cut the existing sections as chapters on the new timeline. Same content, same look.

**Gate:** `EventBus.emit('scroll:setProgress', t)` for any `t ∈ [0,1]` renders a correct
frame with no visual difference from scrolling there. Reduced-motion still works. No new
bytes over the wire beyond `timeline.js`.

### Phase 2 — Warm-up and the load sequence

- Timeline walk at 2 points/segment × 3 frames — §2.5
- `warmup:progress` → loading bar at the 70/30 split
- Splash → header logo dock with no reflow — §2.14
- Intro gate wired to the CTA

**Gate:** cold load on a throttled connection reaches interactive with no first-scroll hitch.
Loading bar advances monotonically.

### Phase 3 — The signature chapter first

Build `review` (0.52–0.58) before anything else. It is 6% of the timeline and the reason the
site exists. If it doesn't land, the plan is wrong and it's cheap to find out now.

**Gate:** the review sweep reads as *"it caught something"* to someone who has not been told
what it is.

### Phase 4 — Remaining chapters

`repo` → `sandbox` → `shipped` → `trajectory` → `instruction` → `contact`, in that order
(hardest first, hero last — the hero is the most-revised thing on any site).

Per chapter: camera keyframes (§2.10), post-FX preset (§2.9), diegetic labels (§2.7),
transition at each boundary (§2.12).

### Phase 5 — Pace and grade

Only now touch `stretchBands`. This is what §2.2 buys: pacing is retuned without renumbering
a single keyframe. Set snap points at chapter boundaries that need to be landed on squarely.

---

## 6. Budgets and constraints

### 6.1 Weight

| | edolus | Target here |
|---|---|---|
| Total transfer | 48.1 MB | **< 600 KB** |
| Requests | 97 | < 15 |
| Time to interactive | ~30 s | < 3 s on 4G |
| JS | 900 KB | < 250 KB gz |

Hard rule: **no binary 3D assets.** If a chapter needs a model, the chapter is wrong.

### 6.2 Performance

- DPR capped at 2, adaptive per §2.6, cached in `localStorage`
- Pixel budget 4.6M desktop / 3.0M mobile
- One `requestAnimationFrame` loop for the whole page. Every subscriber reads the same
  `progress`. No script owns its own rAF except `TextScramble` and ambient drift (§2.3).
- Draw calls < 30. Instancing over individual meshes, always.

### 6.3 Accessibility — where edolus is a bad model

edolus captures the wheel with `preventDefault()` and never scrolls the document. Correct for
a showreel; wrong here. Constraints for this build:

- **`prefers-reduced-motion`**: the timeline still advances, but camera moves are cuts and
  transitions are instant. The narrative must survive with zero motion. Never a blank page.
- **Keyboard**: Arrow/PageUp/PageDown already work via `keyStep` (§2.1). Additionally, `Tab`
  to any focusable element must jump the timeline to that element's chapter — otherwise focus
  lands on invisible content.
- **Real text**: all copy stays in the DOM as diegetic labels (§2.7). Nothing readable is
  drawn into the canvas. The page must be usable with WebGL disabled — chapters degrade to
  the current static sections.
- **No audio without a click.** The CTA may unmute; nothing else may.
- **Deep links**: `#work`, `#rig`, `#about`, `#contact` already exist and must keep working —
  map each to `EventBus.emit('scroll:setProgress', segment.start)`.

### 6.4 Mobile

`touchScrollPx = 4000` vs `24000` desktop (§2.1) — a swipe is a few hundred px; the same
journey must cost proportionally less finger travel. The existing `@media (max-width:860px)`
block in `site/index.html` already reflows the layout; chapters must respect it rather than
fight it.

---

## 7. Open decisions

1. **Stack** — three.js recommended (§4.3). Phase 1 does not depend on the answer, so work
   can start before this is settled.
2. **Audio** — edolus's four-stem score is a large part of why it lands. Muted-by-default with
   a visible toggle is the only version defensible on a portfolio. Or skip entirely.
3. **`trajectory` chapter** — worth building only with real FMD racing-line data. Invented
   telemetry on a page about not faking work would be the wrong note.
4. **Loader** — a percentage counter earns its keep when the load is the pitch. Under a 3 s
   budget it may be better to have no loader at all.

---

## Appendix: reproducing this research

```bash
curl -s https://edolus.com/config.json -o config.json         # 303 assets, 107 script names
curl -s https://edolus.com/__game-scripts.js -o gs.js         # 1 MB minified bundle
curl -s https://edolus.com/__settings__.js                    # scene id, script ids, WASM preload
```

To drive the live timeline from the console — the technique that produced every screenshot
and copy extract in §1–§2:

```js
window.dispatchEvent(new Event('experienceStart'));
EventBus.emit('experience:entered');
EventBus.emit('scroll:setProgress', 0.36);   // → the datacenter frame
document.getElementById('custom-splash-wrapper').style.display = 'none';
```

Chapter frames captured for reference: `0.06` satellite + plexus + diegetic label ·
`0.20` orbital city map, HUD frame, glowing street grid · `0.36` datacenter rack, light cone,
infinity floor · `0.50` chip spec table, typewriter-revealed, monospace key:value ·
`0.64` card rail over light ribbon · `0.78` palette inverts to white, car + video plane ·
`0.97` letterbox aperture close.
