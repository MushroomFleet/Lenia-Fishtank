# Lenia Fishtank

A living wallpaper made of artificial organisms. Continuous cellular automata running on your GPU, filling the screen with small creatures that swim, pair off, split and compete for space — indefinitely, without supervision.

![Lenia Fishtank running the fission species](docs/screenshot.png)

---

## What this is

Conway's Game of Life works on a grid of dead-or-alive cells. **Lenia**, discovered by Bert Wang-Chak Chan in 2018, replaces that with a continuous field convolved by smooth ring-shaped kernels. The result is not pixel art that blinks — it is soft-edged organisms with rims, cores and membranes, which move under their own steam, react to each other, and reproduce.

This repository packages one particular Lenia species, **fission**, as something you can leave running on a second monitor. Fifteen kernels across three colour channels: nine feeding each channel from itself, six cross-wiring the channels together. That cross-feeding is what makes the creatures pair up and divide rather than sit still. They are about a dozen cells across, with a white and magenta rim, a green core and a red single-pixel fringe.

It is not a toy simulation of anything real. It is an aquarium for things that only exist while the shader is running.

## What's in the repo

| File | What it is |
|---|---|
| `fission-tank.html` | The tank, complete. One self-contained file — shaders, species and UI inlined. Open it and it runs. |
| `Lenia-Fishtank-TINS.md` | A full build specification for the desktop application, written so an AI coding agent can generate the entire project from it. |
| `species/fission.json` | The fission rule on its own, if you want to load it into another Lenia implementation. |

## Quick start

Download `fission-tank.html` and open it in any browser with WebGL2 — Chrome, Edge, Firefox or Safari 15+. There is nothing to install, no server to run, and no network access involved. The colony seeds itself from noise and recognisable organisms condense out of it within about five seconds.

For a wallpaper-ish effect right now: open it, press `H` to hide the panel, then fullscreen the browser with `F11`.

### Controls

| Input | Action |
|---|---|
| Drag | Paint new life into the tank |
| Right-drag | Pan the camera |
| Scroll | Zoom, anchored on the cursor |
| Double-click | Fit the world to the window |
| `Space` | Pause |
| `D` | Fresh soup |
| `S` | Empty the tank |
| `F` | Fit view |
| `H` | Hide the panel |

The panel carries a speed slider, a self-refill toggle that reseeds the tank if the colony dies out, a PNG export, and six rule sliders under **Rule**. Those sliders are worth playing with: nudging the growth centre (μ) by ±0.01 changes the species' entire character, and pushing it further turns the creatures into soup. `Back to fission` returns everything to the original rule.

One practical note — the world is sized to your window when the page loads, so it won't grow if you resize afterwards. Reload to get a tank that fills the new size.

## The desktop application

`Lenia-Fishtank-TINS.md` specifies the full application: a Tauri desktop shell that runs the tank as an actual wallpaper, pinned behind every other window and, on Windows, behind the desktop icons themselves.

Beyond the single-file tank it covers:

- **Species library** with JSON import and export
- **Procedural species discovery** — enter a number from 1 to 8192 and get a deterministic creature; the same number gives the same species on every machine, forever
- **Per-kernel sliders** for all 15 kernels' μ, σ and η, applied live
- **Self-sustaining behaviour** — extinction watchdog, optional species rotation on a timer, optional slow parameter drift so the colony's character changes over hours
- **Tray control, autostart, persistent settings, wallpaper-resolution PNG export**

### Building it

The spec is written in [TINS](https://thereisnosource.com) format — *There Is No Source*. The idea is that a sufficiently complete README **is** the distribution: an AI model generates the implementation on demand, and the same document produces better code as models improve. Hand the file to a coding agent:

```bash
git clone https://github.com/MushroomFleet/Lenia-Fishtank
cd Lenia-Fishtank
claude   # then: "Build the project described in Lenia-Fishtank-TINS.md"
```

Expect a Tauri 2 + Vite + TypeScript project with a Rust shell. The spec pins down the parts that are genuinely hard to get right and leaves the rest open:

- The simulation fragment shader is given verbatim, including the three-tier neighbourhood traversal that computes each kernel weight once per distinct radius rather than once per neighbour.
- The `mat4` lane-packing scheme is specified as a **correctness** requirement, not an optimisation. All fifteen kernels live in matrix lanes and are processed component-wise, because a straightforward `for (int k = 0; k < numKernels; k++)` loop over uniform arrays miscompiles under ANGLE — which is every Chromium browser and every WebView on Windows. The symptom is silently wrong output on exactly the machines this targets.
- The fission parameters are included in full, annotated.

### Checking that it worked

Lenia is easy to get *moving* and hard to get *right*. A wrong kernel index still produces motion — it just produces a uniform wash, or a blank screen, or square-edged blobs. The spec therefore includes a twenty-line NumPy reference implementation and the numbers it converges to: starting from smooth value noise, after 1200 steps at R=9, T=2, ring width 0.15, the field settles at a mean of about **0.134** with roughly **16.6%** of cells alive. Run that before touching WebGL and you'll know whether the rule was understood.

## Requirements

WebGL2 with `EXT_color_buffer_float` — anything from the last decade. The simulation runs at 60 steps per second across roughly 480×270 cells on integrated graphics (Intel Iris Xe class); an adaptive throttle backs off automatically on weaker hardware instead of freezing, so the frame rate plateaus rather than stutters.

If you get a black screen and an error message, your browser is refusing float render targets. On Linux this is usually software rendering; check `chrome://gpu`.

## Credits

Lenia was discovered and formalised by **Bert Wang-Chak Chan** — [*Lenia: Biology of Artificial Life*](https://arxiv.org/abs/1812.05433) (2018). The multi-kernel, multi-channel extension used here follows the Lenia family described in that line of work. The fission species was found by parameter search, not designed.

## License

MIT.

---

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{lenia_fishtank,
  title = {Lenia Fishtank: a continuous cellular automaton living wallpaper},
  author = {[Drift Johnson]},
  year = {2026},
  url = {https://github.com/MushroomFleet/Lenia-Fishtank},
  version = {1.0.0}
}
```

### Donate:

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
