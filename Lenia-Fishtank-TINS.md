<!-- TINS Specification v1.0 -->
<!-- ZS:COMPLEXITY:HIGH -->
<!-- ZS:PRIORITY:HIGH -->
<!-- ZS:PLATFORM:DESKTOP -->
<!-- ZS:LANGUAGE:TYPESCRIPT,RUST,GLSL -->

# Lenia Desktop Fishtank

## Description

Lenia Desktop Fishtank is a desktop application that runs a continuous cellular automaton (Lenia) as a living wallpaper. Instead of the discrete cells of Conway's Game of Life, Lenia works on a continuous state field convolved with smooth ring-shaped kernels, which produces soft-edged organisms that swim, pair off, split, orbit each other and compete for space. The application exists to be *watched*, the way an aquarium is watched: it starts on login, fills the screen behind the icons, and keeps a colony alive indefinitely without intervention.

The reference organism is a 15-kernel, 3-channel species named **fission** — small round creatures with a white and magenta rim, a green core and a red pixel fringe, which drift, form short chains, and periodically divide. Its exact parameters are given verbatim in this document, so the flagship behaviour is reproducible without any external asset. Beyond that single species the app ships a small library of named species, can import and export species as JSON, and can procedurally **discover** new viable species from a numeric seed, so that species #4711 is the same creature on every machine, forever.

The audience is a person who wants an idle, beautiful, low-maintenance thing on their screen, and who occasionally wants to reach in and play — paint new life with the mouse, drag a slider and watch the whole ecology change character, then put it back. Everything the simulation needs runs on the GPU in a single fragment shader; the CPU only seeds noise, watches for extinction, and draws the control panel.

---

## Functionality

### Core Features

1. **Live Lenia simulation** — multi-kernel, three-channel, toroidal (edge-wrapping) world simulated entirely in WebGL2 on floating-point textures.
2. **Wallpaper mode** — a borderless fullscreen window pinned behind all other windows, ideally behind desktop icons, with a system-tray icon for control.
3. **Windowed mode** — an ordinary resizable window with the same content, for tinkering.
4. **Species library** — named species bundled with the app, selectable from a dropdown. `fission` is the default and loads on first run.
5. **Species discovery** — enter a number from 1 to 8192, or press *Discover*, to procedurally generate a species. The same number always produces the same species.
6. **Import / export** — load a species from a `.json` file (file dialog or drag-and-drop onto the window); save the current species, including any live slider changes, back to `.json`.
7. **Live rule editing** — six global modifiers (R, T, μ offset, σ scale, η scale, ring width) and per-kernel μ/σ/η sliders, all applied to the running simulation without restarting it.
8. **Hand seeding** — drag on the tank to paint circular patches of fresh noise; creatures grow out of them.
9. **Pan / zoom camera** — inspect a single organism up close or fit the whole world to the screen.
10. **Auto-refill** — when the colony dies out, floods, or freezes, the tank reseeds itself.
11. **Species cycling and drift** — optionally rotate to a new species every N minutes, and/or let the growth centre slowly wander so the colony's character changes over hours.
12. **Screenshot and wallpaper export** — save the current view as a PNG at screen resolution or at a chosen wallpaper resolution.
13. **Persistent settings** — every panel setting, the camera, and the current species survive a restart.
14. **On-screen debug console** — GPU identification, errors and performance counters, toggled with a key, for diagnosing a machine that renders nothing.

### Window Layout

Wallpaper mode, 1920×1080, panel visible:

```
+--------------------------------------------------------------------------+
|                                                          +-------------+  |
|    (o)      (o)(o)                                       | fission  [x]|  |
|        (o)                    (o)                        | 15 kernels  |  |
|                     (o)(o)                               +-------------+  |
|   (o)(o)                            (o)                  | [  Pause  ] |  |
|                (o)                                       | [New soup ] |  |
|          (o)              (o)(o)         (o)             | [Empty    ] |  |
|                  (o)                                     | [Fit view ] |  |
|      (o)(o)(o)                  (o)                      | [Save PNG ] |  |
|                       (o)                                +-------------+  |
|            (o)                        (o)(o)             | Species  v  |  |
|   (o)                (o)(o)                              | #____ [Go ] |  |
|                                 (o)                      | [Discover ] |  |
|         (o)(o)             (o)                           +-------------+  |
|                                          (o)             | fps ==O== 60|  |
|   (o)           (o)                  (o)                 | [x] Refill  |  |
|                          (o)(o)                          | > Rule      |  |
|              (o)                                         | > Kernels   |  |
|                                                          | > Tank      |  |
|  drag to seed - right-drag to pan - scroll to zoom       +-------------+  |
+--------------------------------------------------------------------------+
```

The tank fills the entire window with no chrome. The panel floats over the top-right corner, 260 px wide, translucent, and is the only UI. When hidden (key `H`), a single small button labelled with the species name remains in the same corner; clicking it brings the panel back.

Expanded `Rule` and `Kernels` sections:

```
+-------------------------+     +-------------------------+
| Rule              close |     | Kernels           close |
| R   =====O=====    9.0  |     |  mu              expand |
| T   ==O=========   2.0  |     |  sigma           expand |
| mu  =====O===== +0.000  |     |  eta           collapse |
| sig =====O=====  1.00x  |     |    K0  ==O====    0.19  |
| eta =====O=====  1.00x  |     |    K1  =====O=    0.66  |
| w   ===O=======  0.150  |     |    K2  ===O===    0.39  |
| [ Back to fission     ] |     |    ...                  |
+-------------------------+     +-------------------------+
```

### User Flows

**First run.** The app opens in wallpaper mode, fullscreen, behind other windows. `fission` loads, the world is seeded with smooth random noise, and within about 200 simulation steps recognisable organisms have condensed out of the soup. The panel is visible for 8 seconds, then fades out automatically; moving the mouse over the top-right corner brings it back. (Auto-hide only happens on first run; after that the panel's visibility is whatever the user last left it.)

**Painting life into an empty tank.** The user clicks *Empty*, the world goes black. The user drags across the screen; each pointer sample stamps a disc of noise of radius `max(18, round(R * 2))` cells. Within a few hundred steps each disc has resolved into one or more organisms, or died — both outcomes are correct and expected.

**Finding a new species.** The user types `4711` into the species number field and presses Enter. The app generates species 4711 from its seed, runs a viability test, and loads it. If that seed is not viable, the app reports `Species 4711 doesn't survive — try Discover` and keeps the current species running. Pressing *Discover* instead searches forward from a random seed until it finds a viable species, loads it, and writes the found number into the field so it can be returned to later.

**Saving a favourite.** After nudging μ and σ into something interesting, the user clicks *Export*. A save dialog offers `lenia_species.json`. The exported file has the live modifiers baked into the per-kernel values, so re-importing it reproduces exactly what was on screen, with all sliders back at neutral.

**Leaving it running.** With *Refill* enabled and cycling set to 60 minutes, the tank sustains itself indefinitely: every four seconds it checks whether the colony has died, flooded or frozen, and reseeds if so; every hour it fades out, switches species and fades back in.

### Behaviour Specifications

#### Simulation update

One simulation step advances the whole world by one tick. The number of steps per second is the user's *fps* setting (1–240, default 60); this is decoupled from the render rate, which is one draw per animation frame regardless.

#### Extinction watchdog

Every 4000 ms of unpaused simulation, sample the state texture over at most 96×96 cells from the origin and compute:

- `mean` — the average of (R+G+B)/3 across sampled cells
- `liveFraction` — the proportion of sampled cells whose (R+G+B)/3 lies strictly between 0.04 and 0.99

The tank is **dead** if `mean < 0.008`, **barren** if `liveFraction < 0.004`, and **flooded** if `mean > 0.93`. Any of the three triggers a reseed with fresh noise, if *Refill* is enabled. If the readback throws (some drivers refuse float readback), disable *Refill*, log the error to the debug console, and never attempt it again in that session.

#### Species cycling

When `cycleMinutes > 0`, on each expiry: multiply the visualisation output by a fade factor animated 1 → 0 over 600 ms, load the next species (library order, wrapping; or a fresh discovery if the source is set to *discovered*), reseed the world, then animate the fade 0 → 1 over 600 ms. The simulation keeps stepping throughout. Cycling never fires while the user is interacting (pointer down, or a slider focused); it defers to the next second.

#### Parameter drift

When `drift > 0` (a 0–1 amount, default 0), every 30 seconds choose a new μ-offset target `clamp(current + g * 0.004, -0.02 * drift, +0.02 * drift)` where `g` is a standard normal sample, and linearly interpolate the live μ offset towards it across the following 30 seconds. Drift writes to the same uniform as the μ slider; while the user is dragging that slider, drift is suspended and resumes from the user's value.

#### Window resize

Resizing changes the drawing buffer immediately. The world grid is rebuilt only if the new window differs from the grid's nominal size by more than 5% in either axis, and only after the resize has been quiet for 500 ms. On rebuild: read the old state back, allocate new textures at the new size, copy the overlapping top-left rectangle into them, and fill the remainder with fresh noise. The camera refits unless the user has panned or zoomed since the last fit.

### Edge Cases and Error States

| Condition | Required behaviour |
|---|---|
| No WebGL2 | Full-window message: `This tank needs WebGL2, which this browser isn't providing.` No crash, no blank window. |
| `EXT_color_buffer_float` missing | Full-window message: `This tank needs float render targets (EXT_color_buffer_float).` |
| WebGL context lost | Catch `webglcontextlost`, prevent default, show `Rebuilding the tank...`; on `webglcontextrestored` recreate programs, textures and framebuffers, reload the current species, reseed. |
| Shader compile or link failure | Show the driver's info log verbatim in the error panel and in the debug console. Never fail silently. |
| Imported JSON is not a species | `That file isn't a Lenia species.` Keep the current species running. |
| Imported species has inconsistent arrays | `Species is malformed: mu has 12 entries, numKernels is 15.` Name the offending field. |
| Imported species has `numKernels > 16` | `This build supports at most 16 kernels; that species has 21.` |
| Species number out of range | Disable the load button; the field's placeholder reads `(1-8192)`. |
| Discovery finds nothing in 40 attempts | `No viable species found in 40 tries — try again.` Keep running. |
| Float readback unsupported | Disable *Refill*, log once, continue simulating. |
| Simulation slower than the requested fps | Never bank backlog. Cap accumulated time at one step and let the achieved rate plateau; report the real rate in the panel. |
| Window minimised / occluded | `requestAnimationFrame` stops; on resume, discard accumulated time rather than running a catch-up burst. |

---

## Technical Implementation

### Architecture

A Tauri desktop shell hosting a TypeScript web frontend that owns the simulation. The Rust side does only what the web layer cannot: window placement, tray, autostart, native file dialogs, and settings on disk.

```
+----------------------------- Tauri (Rust) ------------------------------+
|  window setup  |  tray + menu  |  autostart  |  fs/dialog  |  store     |
+------------------------------- IPC -------------------------------------+
|                          Frontend (TypeScript)                          |
|                                                                         |
|  app/loop.ts      rAF loop, AIMD step throttle                          |
|  app/tank.ts      watchdog, cycling, drift                              |
|  app/settings.ts  load/save settings, apply to UI                       |
|                                                                         |
|  gl/context.ts    WebGL2 context, extension checks, context-loss        |
|  gl/shaders.ts    VERT / SIM_FS / VIS_FS sources                        |
|  gl/simulator.ts  textures, FBOs, uniforms, step(), draw(), readback    |
|                                                                         |
|  sim/species.ts   Species type, validation, mat4 packing                |
|  sim/generator.ts seeded species generation + viability test            |
|  sim/seeding.ts   value noise, soup, empty, stamp                       |
|                                                                         |
|  view/camera.ts   pan / zoom / fit                                      |
|  view/input.ts    pointer + keyboard bindings                           |
|  ui/panel.ts      control panel, species picker, sliders                |
|  ui/kernels.ts    per-kernel slider groups                              |
|  ui/debug.ts      on-screen console                                     |
+-------------------------------------------------------------------------+
```

Suggested layout:

```
lenia-fishtank/
  package.json
  vite.config.ts
  index.html
  src/
    main.ts
    styles.css
    app/{loop,tank,settings}.ts
    gl/{context,shaders,simulator}.ts
    sim/{species,generator,seeding}.ts
    view/{camera,input}.ts
    ui/{panel,kernels,debug}.ts
    species/fission.json
    species/index.ts
  src-tauri/
    Cargo.toml
    tauri.conf.json
    capabilities/default.json
    src/{main.rs,lib.rs,wallpaper.rs}
    icons/
```

Use Tauri 2.x, Vite 5+, TypeScript 5+ in strict mode, and no UI framework — the panel is a few dozen DOM nodes and a framework earns nothing here. Rust dependencies: `tauri`, `tauri-plugin-dialog`, `tauri-plugin-fs`, `tauri-plugin-store`, `tauri-plugin-autostart`, plus `windows` (Windows only) and `objc2`/`objc2-app-kit` (macOS only) if implementing desktop-level pinning.

### Data Models

```typescript
/** A complete Lenia rule. Arrays are parallel; index i describes kernel i. */
interface Species {
  name: string;              // display name, e.g. "fission"
  worldSize: number;         // nominal grid size the species was tuned at; default 128
  R: number;                 // kernel radius in cells, 3..14 (fission: 9)
  T: number;                 // time-step divisor; larger = slower, smoother (fission: 2)
  numKernels: number;        // 1..16 inclusive. Hard ceiling: mat4 lanes.
  init: SpeciesInit;

  betaLen: number[];         // rings per kernel, 1..3
  beta0: number[];           // height of ring 0, 0..1
  beta1: number[];           // height of ring 1, 0..1 (0 when betaLen < 2)
  beta2: number[];           // height of ring 2, 0..1 (0 when betaLen < 3)
  relR: number[];            // per-kernel radius scale, 0.1..1.0
  mu: number[];              // growth centre, 0..1
  sigma: number[];           // growth width, > 0
  eta: number[];             // growth strength, 0..1
  src: number[];             // channel this kernel reads,  0|1|2
  dst: number[];             // channel this kernel feeds,  0|1|2

  mono?: boolean;            // render R broadcast to grey instead of RGB
  seed?: number;             // set when produced by the generator
}

interface SpeciesInit {
  type: "random" | "seed";
  baseNoise?: number;        // random: constant added to noise before clamping (0.1)
  randomScale?: number;      // random: noise lattice spacing as a multiple of R (1.0)
  size?: [number, number, number];  // seed: [height, width, channels]
  cells?: number[];          // seed: row-major h*w*c values in 0..1
}
```

Invariants an implementation must enforce on load: every listed array has exactly `numKernels` entries; `1 <= numKernels <= 16`; every `src` and `dst` is 0, 1 or 2; every `sigma` is strictly positive; every `relR` is strictly positive; `R >= 1`; `T > 0`.

```typescript
interface Settings {
  speciesSource: "library" | "discovered" | "imported";
  speciesName: string;          // library key, e.g. "fission"
  speciesNumber: number | null; // when source is "discovered"
  stepsPerSec: number;          // 1..240, default 60
  paused: boolean;              // default false
  autoRefill: boolean;          // default true
  cycleMinutes: number;         // 0 = off, default 0
  drift: number;                // 0..1, default 0
  panelHidden: boolean;         // default false
  wallpaperMode: boolean;       // default true
  autostart: boolean;           // default false
  cellPx: number;               // CSS px per cell at fit zoom, default 4
  camera: { cx: number; cy: number; px: number };
  modifiers: { R: number; T: number; muOff: number; sigmaScale: number;
               etaScale: number; ringW: number; relRScale: number };
}
```

Settings live in the Tauri store at the platform config dir (`lenia-fishtank/settings.json`). Write is debounced to at most one per 2 seconds. A corrupt or unreadable store is replaced by defaults without prompting.

### Key Algorithm 1 — Ring kernels

Each kernel is a radially symmetric function of distance `r` from the centre cell, made of up to three concentric Gaussian rings. For kernel `k`:

```
if r > R:                      weight = 0
Br    = betaLen[k] * (r / R) / (relR[k] * relRScale)
ring  = floor(Br)
height = beta0[k] if ring == 0 else beta1[k] if ring == 1 else beta2[k] if ring == 2 else 0
weight = height * exp(-0.5 * ((Br - ring - 0.5) / ringW)^2)
```

`ringW` is a single global (default **0.15**) shared by all kernels, not a per-kernel value. `relRScale` and the other `*Scale` names are the live panel modifiers and are 1.0 unless the user moves a slider.

The weighted neighbourhood average for kernel `k` at a cell is `sum(weight * state[src[k]]) / (sum(weight) + 1e-6)`, where both sums run over every cell within radius `R`, wrapped toroidally.

### Key Algorithm 2 — Growth and update

```
growth[k] = eta[k] * (2 * exp(-0.5 * ((avg[k] - (mu[k] + muOff)) / (sigma[k] * sigmaScale))^2) - 1)
            * etaScale / T
newState[c] = clamp(state[c] + sum of growth[k] for all k where dst[k] == c, 0, 1)
```

The bell term is in (0,1], so `2*bell - 1` is in (-1,1]: a neighbourhood density near `mu` grows the target channel, anything far from it decays it. Everything is clamped to [0,1] each step, and that clamping is load-bearing — without it the field diverges.

### Key Algorithm 3 — mat4 lane packing (do not skip this)

All per-kernel values are uploaded as `mat4` uniforms, kernel `i` occupying lane `i` of 16, and the shader operates on them **only** with component-wise operations (`matrixCompMult`, `+`, `-`, `/`, `exp` per column). There is no dynamic array indexing anywhere in the simulation shader.

This is not a micro-optimisation; it is a correctness requirement. A loop of the form `for (int k = 0; k < numKernels; k++) { ... mu[k] ... }` over a uniform array miscompiles under ANGLE (Chrome and any WebView on Windows, translating to D3D11) and produces silently wrong output on the exact machines this app targets. It is also substantially faster, because each neighbour texel is fetched once and distributed to all 16 lanes rather than being re-fetched per kernel.

Packing helpers:

```typescript
/** Per-kernel array -> 16 mat4 lanes. Unused lanes get `fill`. */
function packMat(a: number[], fill: number): Float32Array {
  const o = new Float32Array(16).fill(fill);
  for (let i = 0; i < Math.min(16, a.length); i++) o[i] = a[i];
  return o;
}

/** Channel selector: lane i = 1 when kernel i reads (or writes) channel c. */
function maskMat(arr: number[], c: number, nk: number): Float32Array {
  const o = new Float32Array(16);
  for (let i = 0; i < nk; i++) o[i] = arr[i] === c ? 1 : 0;
  return o;
}
```

Padding values matter: `sigma` pads with **1** (a zero would divide by zero inside the bell), `relR` pads with **1** (same reason in the kernel weight), `eta` pads with **0** so dead lanes contribute no growth, everything else pads with 0.

Upload on species load:

```typescript
gl.uniformMatrix4fv(u.mBetaLen, false, packMat(s.betaLen, 0));
gl.uniformMatrix4fv(u.mBeta0,   false, packMat(s.beta0,   0));
gl.uniformMatrix4fv(u.mBeta1,   false, packMat(s.beta1,   0));
gl.uniformMatrix4fv(u.mBeta2,   false, packMat(s.beta2,   0));
gl.uniformMatrix4fv(u.mMu,      false, packMat(s.mu,      0));
gl.uniformMatrix4fv(u.mSigma,   false, packMat(s.sigma,   1));
gl.uniformMatrix4fv(u.mEta,     false, packMat(s.eta,     0));
gl.uniformMatrix4fv(u.mRelR,    false, packMat(s.relR,    1));
gl.uniformMatrix4fv(u.srcK0,    false, maskMat(s.src, 0, s.numKernels));
gl.uniformMatrix4fv(u.srcK1,    false, maskMat(s.src, 1, s.numKernels));
gl.uniformMatrix4fv(u.srcK2,    false, maskMat(s.src, 2, s.numKernels));
gl.uniformMatrix4fv(u.dstK0,    false, maskMat(s.dst, 0, s.numKernels));
gl.uniformMatrix4fv(u.dstK1,    false, maskMat(s.dst, 1, s.numKernels));
gl.uniformMatrix4fv(u.dstK2,    false, maskMat(s.dst, 2, s.numKernels));
```

### Shaders

Vertex shader, shared by both passes — a single oversized triangle covering the viewport:

```glsl
#version 300 es
in vec2 aPos;
void main(){ gl_Position = vec4(aPos, 0.0, 1.0); }
```

Draw it with `gl.drawArrays(gl.TRIANGLES, 0, 3)` from a buffer holding `[-1,-1, 3,-1, -1,3]`.

**Simulation fragment shader.** Use this as given; it is the load-bearing artefact of the whole project.

```glsl
#version 300 es
precision highp float;
uniform highp sampler2D uState;
uniform ivec2 uSizeI;
uniform float uR, uT;
uniform float uMuOff, uSigmaScale, uEtaScale, uRingW, uRelRScale;   // live, panel-controlled modifiers
uniform mat4 mBetaLen, mBeta0, mBeta1, mBeta2;
uniform mat4 mMu, mSigma, mEta, mRelR;
uniform mat4 srcK0, srcK1, srcK2, dstK0, dstK1, dstK2;
out vec4 fragColor;

#define cm matrixCompMult
const mat4 M0 = mat4(0.0);
const mat4 M1 = mat4(vec4(1.0), vec4(1.0), vec4(1.0), vec4(1.0));
const mat4 EPS = mat4(vec4(1e-6), vec4(1e-6), vec4(1e-6), vec4(1e-6));
const mat4 KM = mat4(vec4(0.5), vec4(0.5), vec4(0.5), vec4(0.5));   // kernel ring center

mat4 exp4(mat4 m){ return mat4(exp(m[0]), exp(m[1]), exp(m[2]), exp(m[3])); }
mat4 floor4(mat4 m){ return mat4(floor(m[0]), floor(m[1]), floor(m[2]), floor(m[3])); }
mat4 eqRing(mat4 r, float v){
  vec4 vv = vec4(v);
  return mat4(vec4(equal(r[0], vv)), vec4(equal(r[1], vv)), vec4(equal(r[2], vv)), vec4(equal(r[3], vv)));
}
// bell-shaped curve, component-wise over a mat4
mat4 bell4(mat4 x, mat4 m, mat4 s){
  mat4 d = (x - m) / s;
  return exp4(cm(d, d) * -0.5);
}
// per-kernel neighbour weight at radius r (0 outside support)
mat4 getWeight(float r){
  if (r > uR) return M0;
  mat4 Br = mBetaLen * (r / uR) / (mRelR * uRelRScale);
  mat4 ring = floor4(Br);
  mat4 height = cm(mBeta0, eqRing(ring, 0.0)) + cm(mBeta1, eqRing(ring, 1.0)) + cm(mBeta2, eqRing(ring, 2.0));
  mat4 ks = mat4(vec4(uRingW), vec4(uRingW), vec4(uRingW), vec4(uRingW));   // global ring width
  return cm(height, bell4(Br - ring, KM, ks));
}
float mdot(mat4 a, mat4 b){ return dot(a[0],b[0]) + dot(a[1],b[1]) + dot(a[2],b[2]) + dot(a[3],b[3]); }

void addN(ivec2 q, mat4 w, inout mat4 sum, inout mat4 total){
  q = (q % uSizeI + uSizeI) % uSizeI;   // toroidal wrap (texelFetch ignores REPEAT)
  vec3 v = texelFetch(uState, q, 0).rgb;
  mat4 valueK = srcK0 * v.r + srcK1 * v.g + srcK2 * v.b;
  sum += cm(valueK, w);
  total += w;
}

void main(){
  ivec2 ip = ivec2(gl_FragCoord.xy);
  vec3 self = texelFetch(uState, ip, 0).rgb;

  mat4 sum = M0, total = M0;
  addN(ip, getWeight(0.0), sum, total);

  int IR = int(ceil(uR));
  for (int x = 1; x <= IR; x++){
    mat4 wo = getWeight(float(x));            // orthogonal: 4 neighbours share this weight
    addN(ip + ivec2( x, 0), wo, sum, total);
    addN(ip + ivec2(-x, 0), wo, sum, total);
    addN(ip + ivec2( 0, x), wo, sum, total);
    addN(ip + ivec2( 0,-x), wo, sum, total);
    float rd = sqrt(2.0) * float(x);          // diagonal: 4 neighbours share this weight
    if (rd <= uR){
      mat4 wd = getWeight(rd);
      addN(ip + ivec2( x, x), wd, sum, total);
      addN(ip + ivec2( x,-x), wd, sum, total);
      addN(ip + ivec2(-x, x), wd, sum, total);
      addN(ip + ivec2(-x,-x), wd, sum, total);
    }
  }
  for (int y = 1; y <= IR; y++){
    for (int x = y + 1; x <= IR; x++){        // general octant: 8 neighbours share this weight
      float r = sqrt(float(x*x + y*y));
      if (r > uR) continue;
      mat4 w = getWeight(r);
      addN(ip + ivec2( x, y), w, sum, total);
      addN(ip + ivec2( x,-y), w, sum, total);
      addN(ip + ivec2(-x, y), w, sum, total);
      addN(ip + ivec2(-x,-y), w, sum, total);
      addN(ip + ivec2( y, x), w, sum, total);
      addN(ip + ivec2( y,-x), w, sum, total);
      addN(ip + ivec2(-y, x), w, sum, total);
      addN(ip + ivec2(-y,-x), w, sum, total);
    }
  }

  mat4 avg = sum / (total + EPS);
  mat4 growthK = cm(mEta, bell4(avg, mMu + uMuOff * M1, mSigma * uSigmaScale) * 2.0 - M1) * (uEtaScale / uT);
  vec3 growth = vec3(mdot(growthK, dstK0), mdot(growthK, dstK1), mdot(growthK, dstK2));
  fragColor = vec4(clamp(self + growth, 0.0, 1.0), 1.0);
}
```

Note the three-tier neighbourhood traversal: the centre cell, then the four orthogonal and four diagonal neighbours at each integer offset (which share a radius and therefore a weight), then the general octant with its eight-fold symmetry. Kernel weights are computed once per distinct radius and reused across all symmetric neighbours, which is where most of the saving comes from.

**Visualisation fragment shader.** The three state channels map directly to red, green and blue — no palette, no tone mapping. `uFade` is the crossfade used by species cycling; it is 1.0 at all other times.

```glsl
#version 300 es
precision highp float;
uniform highp sampler2D uState;
uniform ivec2 uGrid;
uniform vec2 uRes;
uniform vec2 uCenter;   // grid-cell coordinate shown at the screen centre
uniform float uPx;      // drawing-buffer pixels per cell
uniform float uFade;    // 1.0 normally; animated 1->0->1 when cycling species
uniform int uMono;      // 1 = broadcast R to grey
out vec4 o;
void main(){
  vec2 screen = vec2(gl_FragCoord.x, uRes.y - gl_FragCoord.y);  // top-left origin
  vec2 cellf = uCenter + (screen - uRes * 0.5) / uPx;
  ivec2 c = ivec2(floor(cellf));
  if (c.x < 0 || c.y < 0 || c.x >= uGrid.x || c.y >= uGrid.y){
    o = vec4(vec3(0.04, 0.04, 0.06) * uFade, 1.0);
    return;
  }
  vec3 v = texelFetch(uState, c, 0).rgb;
  if (uMono == 1) v = vec3(v.r);
  o = vec4(v * uFade, 1.0);
}
```

### Textures and the step loop

Two `RGBA16F` textures of `WW × WH`, each with a framebuffer, ping-ponged. Half-float halves the state bandwidth against `RGBA32F` and Lenia tolerates the precision loss; the alpha channel is unused and held at 1.

```typescript
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.REPEAT);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.REPEAT);
gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA16F, WW, WH, 0, gl.RGBA, gl.FLOAT, data);
```

`REPEAT` is set for completeness but does nothing for `texelFetch`, which ignores wrap modes — the shader wraps coordinates itself with `q = (q % uSizeI + uSizeI) % uSizeI`, and the double modulo is required because GLSL `%` keeps the sign of the dividend.

A step binds the destination framebuffer, sets the viewport to `WW × WH` (not the canvas size), binds the source texture, draws the triangle, and flips. A draw binds the default framebuffer, sets the viewport to the canvas size, and draws the same triangle with the visualisation program.

Grid size is `WW = max(64, round(innerWidth / cellPx))`, `WH = max(64, round(innerHeight / cellPx))` with `cellPx` defaulting to 4. Render the drawing buffer at CSS resolution (device pixel ratio 1) and let the compositor upscale: the visualisation pass runs per physical pixel and this costs nothing visually on a nearest-neighbour cell grid.

### Key Algorithm 4 — Value noise soup

Every seeding operation uses the same smooth noise. Lattice spacing is `R * randomScale` cells, values are interpolated with smoothstep, and the result is `clamp(baseNoise + noise, 0, 1)` per channel with an independent noise field per channel.

```typescript
function valueNoise(width: number, channelSeed: number, scale: number) {
  const cells = Math.max(2, Math.ceil(width / scale) + 1);
  const grid = new Float32Array(cells * cells);
  let s = 1234.567 * (channelSeed + 1) + Math.random() * 9999.0;
  const rnd = () => { s = Math.sin(s) * 43758.5453; return (s - Math.floor(s)) * 2 - 1; };
  for (let i = 0; i < grid.length; i++) grid[i] = rnd();
  const smooth = (t: number) => t * t * (3 - 2 * t);
  return (x: number, y: number) => {
    const gx = x / scale, gy = y / scale;
    const x0 = Math.floor(gx), y0 = Math.floor(gy);
    const fx = smooth(gx - x0), fy = smooth(gy - y0);
    const g = (ix: number, iy: number) => grid[(iy % cells) * cells + (ix % cells)];
    const a = g(x0, y0), b = g(x0 + 1, y0), c = g(x0, y0 + 1), d = g(x0 + 1, y0 + 1);
    return a + (b - a) * fx + (c - a) * fy + (a - b - c + d) * fx * fy;
  };
}
```

With `baseNoise = 0.1` and `scale = R`, the initial field has a mean of roughly 0.24 and about half the cells at zero. This is the correct starting density — a much sparser soup starves before anything can condense.

**Stamping** (hand seeding) reads the existing patch back first so that everything outside the disc is preserved, which is what makes dragging leave a trail rather than a row of squares:

```
rad = max(18, round(R * 2))
bounds = the cell rect [gx-rad, gy-rad] .. [gx+rad, gy+rad], clipped to the world
read back that rect from the current front framebuffer
for each cell in the rect:
    alpha = 1
    if distance from cell centre to (gx, gy) > rad: leave the read-back value
    else: write clamp(0.5 + 0.5 * noise_ch(px, py), 0, 1) per channel
upload the rect with texSubImage2D
```

### Key Algorithm 5 — Float readback

`RGBA16F` framebuffers are not guaranteed to be readable as `FLOAT`; many drivers only offer `HALF_FLOAT`. Query once and cache:

```typescript
let readType: number | null = null;
function readPatch(fbo, x, y, w, h): Float32Array {
  gl.bindFramebuffer(gl.FRAMEBUFFER, fbo);
  if (readType === null) readType = gl.getParameter(gl.IMPLEMENTATION_COLOR_READ_TYPE);
  if (readType === gl.FLOAT) {
    const out = new Float32Array(w * h * 4);
    gl.readPixels(x, y, w, h, gl.RGBA, gl.FLOAT, out);
    return out;
  }
  const raw = new Uint16Array(w * h * 4);
  gl.readPixels(x, y, w, h, gl.RGBA, gl.HALF_FLOAT, raw);
  const out = new Float32Array(raw.length);
  for (let i = 0; i < raw.length; i++) out[i] = halfToFloat(raw[i]);
  return out;
}

function halfToFloat(h: number): number {
  const s = (h & 0x8000) ? -1 : 1, e = (h >> 10) & 0x1f, m = h & 0x3ff;
  if (e === 0) return s * m * 5.9604644775390625e-8;   // subnormal
  if (e === 31) return m ? NaN : s * Infinity;
  return s * (m + 1024) * Math.pow(2, e - 25);
}
```

Framebuffer row 0 is the same row 0 that `texelFetch` sees, so read-back coordinates need no vertical flip.

### Key Algorithm 6 — Adaptive step throttle

GL commands are submitted asynchronously, so wall-clock timing wrapped around `step()` measures nothing. Use the animation-frame interval — which the browser paces to actual GPU completion — as the load signal, and adapt a per-frame step budget (additive increase, multiplicative decrease):

```typescript
const HARD_MAX = 32, MAX_DT = 0.05, TARGET_MS = 1000 / 60;
let maxSteps = 4, lastT = performance.now(), lastN = 0, acc = 0;

function loop(now: number) {
  requestAnimationFrame(loop);
  const rawMs = now - lastT;
  let dt = Math.min((now - lastT) / 1000, MAX_DT);
  lastT = now;

  if (!paused) {
    if (rawMs > TARGET_MS * 2) maxSteps = Math.max(1, maxSteps * 0.5);
    else if (rawMs < TARGET_MS * 1.25 && lastN >= Math.floor(maxSteps))
      maxSteps = Math.min(HARD_MAX, maxSteps + 0.5);

    acc += dt;
    const stepDur = 1 / stepsPerSec, cap = Math.max(1, Math.floor(maxSteps));
    let n = 0;
    while (acc >= stepDur && n < cap) { step(); acc -= stepDur; n++; }
    if (acc > stepDur) acc = stepDur;   // never bank backlog
    lastN = n;
  }
  draw();
}
```

Halving on a single slow frame is deliberate: a weak GPU that would otherwise freeze for seconds recovers within one frame, and the achieved rate cleanly plateaus at whatever the hardware can sustain. Reset `maxSteps` to 4 whenever a species loads, so the probe re-runs for the new cost.

### Key Algorithm 7 — Species generation

Species numbers are deterministic: number `n` always yields the same species on every machine. Use mulberry32 seeded with `n` mixed by Knuth's multiplicative constant.

```typescript
function mulberry32(a: number) {
  return function () {
    a |= 0; a = (a + 0x6D2B79F5) | 0;
    let t = Math.imul(a ^ (a >>> 15), 1 | a);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}

function generateSpecies(n: number): Species {
  const rng = mulberry32(Math.imul(n, 2654435761) >>> 0);
  const nk = 10 + Math.floor(rng() * 7);          // 10..16
  const R = 8 + Math.floor(rng() * 6);            // 8..13
  const q12 = () => Math.round(rng() * 12) / 12;  // 1/12 quantisation, as in hand-tuned species

  const betaLen: number[] = [], beta0: number[] = [], beta1: number[] = [], beta2: number[] = [];
  const mu: number[] = [], sigma: number[] = [], eta: number[] = [], relR: number[] = [];
  for (let k = 0; k < nk; k++) {
    const len = 1 + Math.floor(rng() * 3);        // 1..3 rings
    betaLen.push(len);
    beta0.push(Math.max(1 / 12, q12()));
    beta1.push(len > 1 ? Math.max(1 / 12, q12()) : 0);
    beta2.push(len > 2 ? Math.max(1 / 12, q12()) : 0);
    mu.push(+(0.1 + rng() * 0.4).toFixed(3));           // 0.100 .. 0.500
    sigma.push(+(0.03 + rng() ** 2 * 0.16).toFixed(4)); // biased small
    eta.push(+(0.15 + rng() * 0.8).toFixed(2));
    relR.push(+(0.5 + rng() * 0.5).toFixed(2));
  }

  // Wiring: three self-kernels per channel first, then the six cross-channel pairs,
  // repeating if more kernels are needed. This is the wiring `fission` uses.
  const wiring: Array<[number, number]> = [
    [0,0],[0,0],[0,0],[1,1],[1,1],[1,1],[2,2],[2,2],[2,2],
    [0,1],[0,2],[1,0],[1,2],[2,0],[2,1],
  ];
  const src: number[] = [], dst: number[] = [];
  for (let k = 0; k < nk; k++) { const [s, d] = wiring[k % wiring.length]; src.push(s); dst.push(d); }

  return { name: `#${n}`, seed: n, worldSize: 128, R, T: 2, numKernels: nk,
           init: { type: "random", baseNoise: 0.1, randomScale: 1 },
           betaLen, beta0, beta1, beta2, mu, sigma, eta, relR, src, dst };
}
```

**Viability test.** Run the candidate on a throwaway 128×128 pair of textures, at neutral modifiers, for 400 steps, sampling the state after steps 100, 250 and 400. A candidate is viable when the step-400 sample satisfies all of:

- `0.01 <= mean <= 0.75` — neither starved nor flooded
- `0.02 <= liveFraction <= 0.60` — something is there, and it has not filled the world
- `spatialStd >= 0.05` — the field has structure rather than a uniform wash
- `abs(mean400 - mean250) <= 0.25 * mean250` — the population has settled rather than still collapsing
- `abs(mean400 - mean250) >= 1e-5` OR `spatialStd >= 0.15` — not frozen flat

*Discover* picks a random start in 1..8192 and tests consecutive numbers (wrapping) until one passes or 40 have failed. Roughly one seed in four passes with these ranges, so a search normally succeeds within a handful of tries. The viability run must be invisible: it uses its own framebuffers and never touches the displayed state.

### Reference species: fission

Ship this verbatim as `src/species/fission.json`. It is the default on first run and the anchor for the visual check below.

```json
{
  "name": "fission",
  "worldSize": 128,
  "R": 9,
  "T": 2,
  "numKernels": 15,
  "init": {"type": "random", "baseNoise": 0.1, "randomScale": 1},
  "betaLen": [1, 1, 2, 2, 1, 2, 1, 1, 1, 2, 2, 2, 1, 2, 1],
  "beta0": [1, 1, 1, 0, 1, 0.8333333333333334, 1, 1, 1, 0.9166666666666666, 0.75, 0.9166666666666666, 1, 0.16666666666666666, 1],
  "beta1": [0, 0, 0.25, 1, 0, 1, 0, 0, 0, 1, 1, 1, 0, 1, 0],
  "beta2": [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
  "relR": [0.91, 0.62, 0.5, 0.97, 0.72, 0.8, 0.96, 0.56, 0.78, 0.79, 0.5, 0.72, 0.68, 0.55, 0.82],
  "mu": [0.272, 0.349, 0.2, 0.114, 0.447, 0.247, 0.21, 0.462, 0.446, 0.327, 0.476, 0.379, 0.262, 0.412, 0.201],
  "sigma": [0.0595, 0.1585, 0.0332, 0.0528, 0.0777, 0.0342, 0.0617, 0.1192, 0.1793, 0.1408, 0.0995, 0.0697, 0.0877, 0.1101, 0.0786],
  "eta": [0.19, 0.66, 0.39, 0.38, 0.74, 0.92, 0.59, 0.37, 0.94, 0.51, 0.77, 0.92, 0.71, 0.59, 0.41],
  "src": [0, 0, 0, 1, 1, 1, 2, 2, 2, 0, 0, 1, 1, 2, 2],
  "dst": [0, 0, 0, 1, 1, 1, 2, 2, 2, 1, 2, 0, 2, 0, 1]
}
```

Reading it: 15 kernels, `R = 9`, `T = 2`. Kernels 0–8 are self-kernels, three per channel. Kernels 9–14 cross-wire every ordered pair of distinct channels, and that cross-feeding is what makes the organisms pair up and divide rather than sit still. Seven kernels have two rings (`betaLen = 2`); the other eight have one, and none has three. Note kernel 13, with `beta0 = 1/6` and `beta1 = 1` — a weak inner ring and a strong outer one.

### Tauri shell

`tauri.conf.json` essentials:

```json
{
  "productName": "Lenia Fishtank",
  "identifier": "com.lenia.fishtank",
  "build": { "frontendDist": "../dist", "devUrl": "http://localhost:1420",
             "beforeDevCommand": "npm run dev", "beforeBuildCommand": "npm run build" },
  "app": {
    "windows": [{
      "label": "tank",
      "title": "Lenia Fishtank",
      "width": 1280, "height": 720,
      "decorations": false,
      "transparent": false,
      "resizable": true,
      "fullscreen": false,
      "alwaysOnBottom": true,
      "skipTaskbar": true,
      "shadow": false
    }],
    "security": { "csp": "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'" }
  }
}
```

**Wallpaper mode, in three tiers.** Implement tier 1 always; tiers 2 and 3 are per-platform upgrades and must degrade to tier 1 if they fail, never crash the app.

- *Tier 1 — every platform.* Borderless, `alwaysOnBottom`, `skipTaskbar`, sized and positioned to the primary monitor's full work area. The window sits under every other window but above the desktop, so icons are covered. This is the guaranteed baseline.
- *Tier 2 — Windows.* Reparent the window behind the desktop icons: send `0x052C` to `Progman` to make Windows spawn the `WorkerW` layer, enumerate top-level windows to find the `WorkerW` that follows the one owning `SHELLDLL_DefView`, then `SetParent(ourHwnd, workerW)`. Icons then draw on top of the tank. Re-apply on display change (`WM_DISPLAYCHANGE`) and when Explorer restarts.
- *Tier 3 — macOS.* Set the `NSWindow` level to `CGWindowLevelForKey(kCGDesktopWindowLevel)`, set `collectionBehavior` to `canJoinAllSpaces | stationary`, and `ignoresMouseEvents` to false so painting still works. On Linux/X11, set `_NET_WM_WINDOW_TYPE_DESKTOP` before mapping; on Wayland there is no portable equivalent, so stay on tier 1 and say so in the panel.

The tray menu carries: *Show panel*, *Pause*, *New soup*, *Windowed mode* (checkbox), *Start with system* (checkbox), *Quit*. Closing the window in wallpaper mode hides it to the tray rather than exiting.

Native calls the frontend needs, exposed as Tauri commands:

```rust
#[tauri::command] fn set_wallpaper_mode(app: AppHandle, on: bool) -> Result<(), String>;
#[tauri::command] fn wallpaper_tier(app: AppHandle) -> u8;      // 1, 2 or 3 — shown in the panel
#[tauri::command] fn set_autostart(app: AppHandle, on: bool) -> Result<(), String>;
```

Import, export and PNG saving go through `tauri-plugin-dialog` plus `tauri-plugin-fs`; settings through `tauri-plugin-store`. Grant only `dialog:default`, `fs:allow-write-file`, `fs:allow-read-file`, `store:default` and `autostart:default` in `capabilities/default.json`.

**Screenshot export.** Create the context with `preserveDrawingBuffer: true`, redraw, then `canvas.toBlob`. For wallpaper export at a resolution other than the window's, render the visualisation pass into an offscreen framebuffer of the requested size with the camera temporarily set to fit that aspect, read it back, and encode via an `OffscreenCanvas`. Default filename `lenia-<species>-<timestamp>.png`.

### Input Bindings

| Input | Action |
|---|---|
| Left-drag on tank | Stamp noise discs (paint life) |
| Right-drag, middle-drag, or Shift+drag | Pan the camera |
| Scroll wheel | Zoom about the pointer, clamped to `fitPx * 0.15` .. `80` px/cell |
| Double-click | Fit the world to the window |
| `Space` | Pause / resume |
| `D` | New soup |
| `S` | Empty the tank |
| `F` | Fit view |
| `H` | Hide / show the panel |
| `R` | Discover a new species |
| `` ` `` | Toggle the debug console |

Key handling is suppressed while a text or number input has focus. Zoom about the pointer must keep the cell under the cursor fixed: convert the pointer to a cell coordinate before scaling `px`, then set `cx`/`cy` so that the same cell maps back to the same pixel.

---

## Style Guide

The tank is the entire visual identity; the panel is a quiet instrument beside it, never a dashboard.

- **Palette.** Out-of-bounds background `#0A0A0E`. Panel `rgba(14,12,20,0.82)` with a 14 px backdrop blur and a `rgba(236,72,201,0.28)` border, picking up the magenta rim of the organisms. Text `#E9E4F2`, secondary `#948DA6`, slider accent `#EC48C9`, affirmative/running accent `#4ADE9B`. No other colours — the organisms' own RGB is the only saturation the design needs.
- **Type.** One family: the platform UI sans (`ui-sans-serif, system-ui, "Segoe UI", Helvetica, Arial`). 13 px base, 14 px semibold for the species name, 11.5 px for hints and slider read-outs. Sentence case throughout; no all-caps labels. Numeric read-outs use `font-variant-numeric: tabular-nums` so they stop jittering as values change.
- **Panel.** 260 px wide, 10 px radius, 12 px padding, fixed 14 px from the top-right. Controls stack in a single column; the four-button grid is two columns. Collapsible sections use a `summary` that reads `open` / `close` on the right rather than a chevron.
- **Motion.** Only two animated things exist: the 600 ms species crossfade, and the 200 ms panel fade on first-run auto-hide. No hover transitions, no entrance animations. Respect `prefers-reduced-motion: reduce` by making both instantaneous.
- **Cursor.** Crosshair over the tank, `grabbing` while panning.

---

## Testing Scenarios

1. **Fission renders correctly.** Load `fission` at defaults on a 1920×1080 window and let it run 1200 steps. Expect roughly 15–20% of cells alive (value > 0.1), a mean field value near 0.13, and discrete round organisms 10–14 cells across with a white/magenta rim, a green core and a red single-pixel fringe. A uniform wash, a blank screen, or square-edged blobs all mean the kernel indexing is wrong.

2. **CPU cross-check.** Independently verify the rule with this reference implementation before trusting the GPU path. It reproduces the same statistics and the same visual signature:

```python
import json, numpy as np
sp = json.load(open('fission.json'))
R, T, nk, N, ringW = sp['R'], sp['T'], sp['numKernels'], 128, 0.15
IR = int(np.ceil(R)); ys, xs = np.mgrid[-IR:IR+1, -IR:IR+1]; rr = np.hypot(xs, ys)
K, FK = [], []
for k in range(nk):
    Br = sp['betaLen'][k]*(rr/R)/sp['relR'][k]; ring = np.floor(Br)
    h = sum(b*(ring == i) for i, b in enumerate([sp['beta0'][k], sp['beta1'][k], sp['beta2'][k]]))
    w = h*np.exp(-0.5*((Br-ring-0.5)/ringW)**2); w[rr > R] = 0
    W = np.zeros((N, N)); W[:w.shape[0], :w.shape[1]] = w
    K.append(w); FK.append(np.fft.fft2(np.roll(W, (-IR, -IR), (0, 1))))
rng = np.random.default_rng(7)
A = np.clip(0.1 + rng.random((3, N, N))*2 - 1, 0, 1)   # crude soup; smooth noise is better
for step in range(1201):
    G = np.zeros_like(A)
    for k in range(nk):
        avg = np.real(np.fft.ifft2(np.fft.fft2(A[sp['src'][k]])*FK[k]))/(K[k].sum()+1e-6)
        G[sp['dst'][k]] += sp['eta'][k]*(2*np.exp(-0.5*((avg-sp['mu'][k])/sp['sigma'][k])**2)-1)/T
    A = np.clip(A+G, 0, 1)
print(A.mean(), (A > 0.1).mean())
```

   With the smooth value-noise soup described above rather than white noise, this settles at `mean ≈ 0.134`, `live ≈ 0.166` after 1200 steps.

3. **ANGLE correctness.** Run on Windows in a Chromium-based WebView. If the organisms differ from the same species on another platform, or the screen is uniformly one colour, the shader has picked up dynamic indexing somewhere.

4. **Toroidal wrap.** Pan past the world edge. An organism leaving the right edge must re-enter on the left with no distortion, and the region outside the grid must render as the flat background colour.

5. **Zoom anchoring.** Put the cursor on a specific organism and scroll both ways through the full zoom range. That organism stays under the cursor throughout.

6. **Painting into an empty tank.** *Empty*, then drag a long stroke. A continuous band of noise appears with no square artefacts and no gaps between pointer samples, and resolves into organisms within ~300 steps.

7. **Extinction recovery.** *Empty* with *Refill* on. Within 4 seconds the tank reseeds itself and life returns.

8. **Flood recovery.** Push η to 2.0× until the field saturates. The watchdog detects `mean > 0.93` and reseeds.

9. **Throttle.** Set fps to 240 on a modest GPU. The reported achieved rate plateaus below 240 and the window stays responsive; it never freezes for seconds at a time and never accumulates backlog that fast-forwards later.

10. **Determinism of discovery.** Enter 4711, note the organism. Load `fission`, then enter 4711 again — identical species. Restart the app and repeat — still identical.

11. **Round-trip export.** Move μ to +0.02, export, restart, import. The loaded species behaves identically to the pre-export state, with all modifier sliders at neutral.

12. **Persistence.** Set fps 30, cycling 15 minutes, panel hidden, camera zoomed; quit; relaunch. All restored, including the camera.

13. **Wallpaper layering.** Enable wallpaper mode. The tank covers the desktop, sits behind every other window, and the panel is still clickable. On Windows with tier 2, desktop icons remain visible and clickable on top of the tank.

14. **Resize.** Drag the window across a 2× size change. After the resize settles, the grid rebuilds, the existing colony survives in the overlapping region, and new area arrives as fresh soup.

---

## Accessibility Requirements

- Every control is reachable by keyboard and has a visible focus ring (`outline: 2px solid #4ADE9B; outline-offset: 2px`). The panel is a landmark region labelled "Tank controls".
- Every icon-only button carries an `aria-label`; every slider carries an `aria-label` naming the parameter, not just its symbol (`Growth centre offset`, not `mu`).
- Sliders respond to arrow keys at their `step`, and to Page Up/Down at ten steps.
- Numeric read-outs sit adjacent to their slider in the DOM, so a screen reader announces value with parameter.
- `prefers-reduced-motion: reduce` disables the crossfade and the panel fade. It does not pause the simulation — the simulation is the content, not decoration.
- Colour is never the only signal: the pause control changes its text between `Pause` and `Play`, not just its colour.
- The panel must remain legible over a bright tank; the backdrop blur plus 82% opaque fill is the mechanism, so do not reduce the opacity below 0.8.

## Performance Goals

| Metric | Target |
|---|---|
| Simulation | 60 steps/s at 480×270 cells with a 15-kernel species on integrated graphics (Intel Iris Xe / Arc 140V class) |
| Render | One draw per animation frame, 60 fps, independent of simulation rate |
| Frame time | No frame over 33 ms in steady state; a single slow frame halves the step budget and recovery is immediate |
| Idle CPU | Under 3% with the panel hidden — the CPU submits draw calls and nothing else |
| Memory | Under 150 MB resident; state is two `RGBA16F` textures, about 2 MB at 480×270 |
| Cold start | Organisms visible within 5 seconds of launch (roughly 200 steps) |
| Species load | Under 100 ms, excluding the viability test |
| Viability test | Under 1.5 s for 400 steps at 128×128 |

Cheap wins worth keeping: fetch each neighbour texel once and distribute it to all 16 lanes; compute each kernel weight once per distinct radius and reuse it across the 8-fold symmetric neighbours; render the drawing buffer at DPR 1; use `RGBA16F` rather than `RGBA32F`.

---

## Decisions Taken Where the Brief Was Silent

These are choices, not requirements handed down; an implementation may revisit them if it says so.

- **No UI framework.** The panel is small and static enough that vanilla DOM is less code than any framework's setup.
- **Kernel ceiling of 16.** Set by the `mat4` lane packing. Supporting more would mean a second set of matrices and is not worth it; species above 16 kernels are rejected on import with a clear message.
- **Ring width is global.** The original rule has no per-kernel ring width, and adding one would change the meaning of every existing species file. It stays a single slider.
- **Cycling defaults off.** A tank that changes species unasked is surprising; the feature is there for people who want it.
- **Discovery range 1–8192.** Arbitrary but matched to the reference implementation's UI, and large enough that exhausting it is not a concern.
- **Wallpaper mode is the default.** The app is a fishtank first and a tool second; windowed mode is one tray click away.
