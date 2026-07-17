# GPU terrain — moving the world's render path to WebGL2

Status: **Done and live.** Written 2026-07-17, after the 0.35.0 full-bleed shell made a
pre-existing pan cost cross the frame budget.

**Result so far — the gate passes, zero differing pixels, nothing stubbed.**
`src/game/terrainGpuShader.ts` + `src/game/terrainGpu.ts` render the terrain **including
rivers** in WebGL2, pixel-for-pixel identical to `renderTerrainView` across all ten biomes and
the full zoom range:

```
spawn + POI clearing     0 / 144000    plains 100%
lake                     0 / 144000    lake 100%
deep sea / world edge    0 / 144000    deep-sea 100%
forest/plains/jungle     0 / 144000    forest 74% · plains 25% · jungle 1%
whole island (worst px)  0 / 144000    deep-sea 98% · sea · sand · forest · plains · jungle
deep zoom in             0 / 144000    plains 100%
MOUNTAIN                 0 / 144000    mountain 100%
SAND / shoreline         0 / 144000    sand 100%
SNOW peak                0 / 144000    snow 100%
RIVER (dense)            0 / 144000    river 99% · plains 1%
RIVER zoomed out         0 / 144000    forest 64% · plains 21% · jungle 7% · mountain 5% · sand 2%
```

Run it with `npm run dev` → `http://localhost:5173/scripts/parity-gate.html`.

**It is wired into `MapView` and live** (0.35.4). `renderTerrainView` remains as the fallback
when WebGL2 is unavailable, and as the source of truth for gameplay.

Two things the gate taught that are worth keeping:

- **Its first version passed while never drawing a mountain or snow pixel** — it never
  exercised the `inland` field, the exact thing most likely to break. Every view now prints the
  biomes it contains, and the views were found by scanning the seed, not guessed. Snow is one
  peak on seed 1337 (20 samples in 90,601); it needs aiming at deliberately.
- **`isOcean` was wrong in a way that a coarser gate would have missed**: R8 textures sample
  *normalised*, so a stored `1` reads back as `1/255`, and a `> 0.5` test made every ocean pixel
  a lake. Caught only because the gate diffs whole views rather than sampling.

**Verified on real hardware.** Both the gate and the benchmark run against an NVIDIA RTX 5070
Ti (ANGLE/D3D11) — drop `--disable-gpu` from the headless flags and Edge uses the real adapter.
This matters beyond speed: the df arithmetic is Dekker's, which **requires** the compiler not to
reassociate or contract to FMA, and NVIDIA's compiler is far more aggressive than WARP's. The
gate passes identically on both, so the double-float trick survives a real shader compiler.

**The speedup, measured** (`scripts/gpu-bench.html`):

| viewport | cells | CPU (fp64) | GPU (df + 1px sync) | speedup |
| --- | --- | --- | --- | --- |
| 1440×620 | 24,960 | 18.60 ms | 0.300 ms | **62×** |
| 1440×900 | 36,000 | 25.70 ms | 0.300 ms | **86×** |
| 1920×1080 | 57,600 | 43.20 ms | 0.300 ms | **144×** |
| 2560×1440 | 102,480 | 77.70 ms | 0.400 ms | **194×** |

The GPU column barely moves with resolution, because it is dominated by the forced 1px readback
— the real render is below it. Two traps this measurement fell into first, both worth
remembering:

- **`--virtual-time-budget` freezes the clock during synchronous work**, so every timing reads
  `0.0ms`. The benchmark therefore uses no virtual time and does everything in a module body,
  which completes before the `load` event the screenshot waits for.
- **`gl.finish()` does not force the work.** Chrome elides rendering for a canvas that is never
  displayed and never read; a batched run reported 0.001ms/render at 1440p, i.e. 3.7 Tpixel/s —
  ~18× a 5070 Ti's theoretical fillrate, so plainly nothing happened. `readPixels` forces a real
  flush, at the cost of a stall that makes the GPU column an upper bound.

This is `docs/render-plan.md`'s **Step 2**, widened. That step scoped *water* → WebGL2 and
assumed "the CPU terrain pass still produces the field exactly as today". Measurement says that
assumption is the problem: the terrain pass **is** the cost.

## The measurement

`renderTerrainView` re-renders the entire viewport on the CPU on **every camera move** — so a
60fps pan asks for 60 full terrain renders per second. Timed in Node against the real
`src/game/terrain.ts` with a stub 2D context (which charges nothing for canvas work, so these
are floors, not ceilings):

| viewport | cells (`cell=6`) | Node floor | **real browser** |
| --- | --- | --- | --- |
| 1440×620 (the pre-0.35.0 box) | 24,960 | 12.75 ms | **18.60 ms** |
| 1440×900 (0.35.0 full-bleed) | 36,000 | 16.78 ms | **25.70 ms** |
| 1920×1080 | 57,600 | 26.21 ms | **43.20 ms** |
| 2560×1440 | 102,480 | 45.58 ms | **77.70 ms** |

The Node column stubs the 2D context, so it charges nothing for `fillStyle` parsing or 36,000
`fillRect` calls — it is a floor, and the real cost is ~50% higher. One 60fps frame is 16.7 ms.

**Panning was already over budget before 0.35.0** (18.60 ms in the old box), and going
full-bleed made it worse; it scales with the player's monitor. At 1440×900 terrain alone caps
the pan at ~39fps; at 1080p ~23fps; at 1440p ~13fps — before ambient, React, or the flow canvas
get a turn.

Where it goes, at 1440×900:

| | |
| --- | --- |
| `model.elevation` | **9.63 ms — 95% of `biomeOf`** |
| `riverAt` (string-key Map lookup per land cell) | 3.75 ms* |
| `isOcean` | 1.21 ms* |
| `hexToRgb` per cell (hoistable) | 0.50 ms |
| `` `rgb(…)` `` template per cell (removable) | 0.49 ms |

\* measured standalone; inside `biomeOf` they're behind early-outs, so they don't sum linearly.

**The micro-optimisations are not worth having.** Hoisting `hexToRgb` and killing the per-cell
colour string buys ~1 ms of ~17 ms. The cost is `elevationRaw`: ~9 octaves of domain-warped
fbm per 6px cell — about 1.3M `hash2` calls per frame. That is ALU-bound math, which is what a
fragment shader is for. Canvas2D's `fillRect` is already GPU-rasterised; we are not fill-rate
bound, so "turn on GPU acceleration" changes nothing. The math has to move.

## ⚠️ The parity gate already fired once — read this before writing GLSL

The first parity probe (below) held `wx` in **fp64 on both sides** and only varied the noise
chain. That was measuring the wrong thing, and its "0 flips, green light" was wrong. A shader
computes the **sample position itself** in fp32:

```
wx = ox + (px + cell/2) * worldPerPx
```

The world is ±50,000 units across. **fp32 ulp at 50,000 is ~0.006 world units** — the shader
cannot address a world position more precisely than that, before any noise runs. Measured over
240 panned frames at four camera positions, feeding both sides through the *same fp64
classifier* so that only the coordinate differs:

```
near spawn    0/60  frames differ
mid-world     8/60  frames differ    worst frame 240 cells   max channel Δ 163
world edge    5/60  frames differ    worst frame 240 cells   max channel Δ  10
deep edge     6/60  frames differ    worst frame 240 cells   max channel Δ   8
```

**390 of 396 differing cells were jitter-cell flips; only 6 were biome changes.** "240 cells" is
exactly `cols` — an entire row shares `wy`, so when `floor(wy/11)` rounds across a boundary the
whole 1440px-wide band changes shade at once:

```
fr47 cell(0,102)  biome lake -> lake   jitterCell 1091:733 -> 1091:734
fr47 cell(1,102)  biome lake -> lake   jitterCell 1092:733 -> 1092:734
```

The noise field is *continuous*, so its 9e-7 error is harmless — a lattice mismatch at a cell
boundary yields the same interpolated value. **The jitter is not.** It is a discontinuous
per-cell hash (`hash2(floor(wx/11), floor(wy/11))`), so a 5e-4 coordinate error becomes a **14%
brightness step across a full-width band, on ~13% of frames.** That is a visible flicker, and it
fails the gate.

### The fix, and what it costs

Never form the absolute coordinate in fp32 (the standard "relative to eye" trick — Cesium,
deck.gl). The CPU splits the view origin in **fp64** into an integer lattice base plus a small
remainder; the shader only ever does small-magnitude math on top:

```
ox = 11*q + r          // q integer, r in [0,11)   -- computed CPU-side, fp64, once per frame
floor(wx/11) = q + floor((r + px*worldPerPx) / 11)  // all terms small => fp32 is exact here
```

Verified over 240,000 samples across five world positions × 200 panned frames:

| | wrong jitter cells |
| --- | --- |
| naive fp32 (`wx = ox + px*wpp`) | 28 (0.0117%) |
| decomposed (`q + local`) | **0** |

**This is the real cost of the port, and it invalidates a claim in both this document and
`render-plan.md`: the noise is _not_ a "near-verbatim ~50 line" translation.** Every place that
touches an absolute coordinate needs the integer-base + fraction treatment, per octave:

- the jitter lattice (verified above),
- `valueNoise`'s lattice inside **every** fbm octave — warp (2×2), continental (5), inland (4),
  moisture (3), river warp (2×2): ~18 octaves, each needing `floor(X*freq) = Q*freq +
  floor(F*freq)` with `Q` exact,
- the domain warp's second stage: `x = wx + dx` re-forms an absolute coordinate, but `dx` is
  bounded (±550), so the large part is still the view origin and the local part stays ~±710 —
  decomposable, but it must be threaded through,
- `cellIndex` for the ocean/carve grids (390 units per cell, so low risk, but not zero).

So `noise.ts` needs a **parallel decomposed-coordinate implementation** in GLSL that does not
look like the JS. That is intricate, and it is the bulk of Stage 1's risk — not the shader
plumbing.

## Parity is provable — and it is not bit-exactness

The rule for this work is **pixel-identical**, per the user. That is achievable, but the honest
version needs stating, because it cannot be *bit*-exact:

- JS computes noise in **fp64**; GLSL `highp` computes in **fp32**.
- Divergence starts at the source: `hash2` ends in `(h >>> 0) / 4294967295`, and GLSL's
  `float(h)` rounds a 32-bit integer into a 24-bit mantissa before the divide.

Measured, by re-implementing the exact chain with `Math.fround()` after every operation (a
*lower bound* on real divergence — a GLSL compiler may also contract to FMA and reassociate)
and comparing against the shipped code over **1,000,000 samples spanning the whole 100 km
world**:

```
max |Δ| continental field      8.964e-7
max |Δ| landness (land/sea)    9.083e-7
samples sitting ON the coast   3        (|landness| < 1e-6)
within fp32 error of flipping  0
land/sea decision FLIPPED      0        (0.000000%)
```

The elevation gradient is ~3 orders of magnitude steeper than the error, so **no pixel changes
classification**. Pixel-identical is real; bit-identical is not, and chasing it would mean
emulating fp64 on the GPU.

**This is why parity matters, and it is not cosmetic.** `pathfind.ts`, `deposits.ts` and
`leaves.ts` all call CPU `biomeOf` as the source of truth and must keep doing so — they are not
per-frame and do not belong on the GPU. If the shader and the CPU ever disagree, a pixel renders
as land that the avatar refuses to walk on, or ore scatters onto water. The CPU stays
authoritative; the GPU only draws.

**Verification gate:** diff the GPU render against the CPU render pixel-for-pixel over a set of
seeds and camera positions chosen to include coastline, lakes, rivers, mountains and the spawn
clearing. The bar is **zero differing pixels**. If a shoreline pixel flickers, this plan is
wrong and we stop.

## The two structural findings

**1. Terrain cannot move alone.** `renderTerrainView` returns the `AmbientField` that
`renderAmbient` consumes — `kind`/`fx`/`fy`/`shore`, collected during the same per-cell pass
(`terrain.ts:591–622`). Move terrain to the GPU while the CPU still builds that field and we pay
the same `elevation` math for nothing. So terrain and ambient move **together**: two WebGL2
passes sharing one GLSL library, with the field never leaving the GPU. `AmbientField` then
disappears from the render path entirely (it has no other consumer — checked).

**2. Rivers are the only part that is not noise.** `riverAt` warps the query point by two fbm
octaves (portable) and then does a spatial-hash lookup against line segments (not portable).
It cannot be composited afterwards by stroking segments onto a canvas: the warp is **per-pixel**,
so the river's rendered shape *is* the segment set warped by a noise field. Rivers must be in
the shader.

## Plan

### Stage 1 — the GLSL core, behind a parity harness

1. `noise.ts` → GLSL: `hash2` (WebGL2 `uint`, exactly 32-bit — this is why WebGL1 is out,
   see render-plan), `smootherstep`, `smoothstep`, `valueNoise`, `fbm` — **in a decomposed
   (integer-base + fraction) form, NOT a verbatim port.** See the precision section above; a
   verbatim port flickers a full-width band on ~13% of panned frames.
2. `elevationRaw` → GLSL: domain warp, continental field, inland field, radial threshold, the
   POI levelling loop (POIs → a uniform array), the spawn cap.
3. Biome classification → GLSL, minus rivers: sea/deep-sea/lake/sand/mountain/snow/plains/
   forest/jungle, plus the `hash2` jitter.
4. `isOcean`: the 256×256 `ocean` `Uint8Array` → an R8 texture; the neighbour test is a
   3×3 texel gather. The `carveSet` goes the same way.
5. **Gate:** render CPU and GPU side by side and diff. Zero differing pixels or stop.

### Stage 2 — rivers in the shader ✅ done

695 segments / 815 buckets on seed 1337. `segs` upload as an RGBA32F texture (2 texels each:
`ax,ay,bx,by` then `w,dx,dy`), the `buckets` map flattens to an RG32UI (offset,count) grid over
an R32UI index list, and the per-bucket ORDER is preserved — `riverAt` returns the FIRST hit,
and Stage 3 advects water along *that* segment's flow direction, so a different winner would
move the current even where the pixel stays blue.

Two precision notes specific to rivers:

- **Segment endpoints upload as plain fp32, and that is safe** — unlike everything else at world
  scale. They come off the D8 grid as `(i + 0.5) * (2*WORLD_RADIUS/N) - WORLD_RADIUS`, needing
  ~20 significand bits, so fp32 holds them exactly. It is the *query* point that cannot be held.
- **`distToSeg` must not subtract in fp32.** `q` is ~50,000 and so is `ax`, but their difference
  is tens of units: subtracting them in fp32 cancels catastrophically and leaves ~0.006 of noise
  on a test whose threshold is a river half-width. The shader subtracts in df, then collapses —
  everything after that is small, so plain fp32 is exact enough.

### Stage 3 — the field, not the overlay ✅ done (and NOT what this plan said)

The plan here was "port `renderAmbient` to a second pass and delete `AmbientField`". **That was
the wrong job.** Reading `ambientfx.ts` properly showed why:

- `finishAmbientField`'s shore term is an iterative **BFS dilation** through water cells, and it
  is **view-dependent** (off-viewport neighbours count as "not land", deliberately, so the
  screen border doesn't foam). That is a stencil op, not a per-pixel function — it would need
  ping-pong passes to reproduce, and its edge behaviour is a quirk that would have to be
  reproduced too.
- More to the point: **the animation was never the cost.** `renderAmbient` walks cells and skips
  every dry one. The 25.7 ms was `biomeOf` per cell inside `renderTerrainView`, which produced
  the field as a side effect of painting.

So the overlay does not move. The GPU produces the **field** in a second pass at **one texel per
terrain cell** (240×150 for 1440×900, ~36× smaller than the viewport), reads it back, and
`ambientFieldFromGpu` hands it to the untouched CPU `renderAmbient`. Readback measured at
**0.20 ms** (563 KB RGBA32F) — against the 25.70 ms it replaces:

```
RGBA8   1440x900 field   240x150   141KB   median 0.20ms
RGBA32F 1440x900 field   240x150   563KB   median 0.20ms
RGBA32F 2560x1440 field  427x240  1601KB   median 0.30ms
```

RGBA32F rather than RGBA8 because `fx`/`fy` are unit flow vectors that `renderAmbient`
multiplies by hundreds of world units — quantising them to a byte would visibly move the
current. The gate compares the GPU field against the CPU's cell-for-cell, `kind` and flow alike,
bit-for-bit.

The payoff of doing it this way: **ambient parity is free rather than gated.** The water is
still drawn by the same code, so it still looks like the water. Both mains are spliced from one
shared `GLSL_CORE`, so the colour pass and the field pass cannot disagree about what a cell is.

### Fallback

WebGL2 is present in Electron and every modern browser, so this is a constraint to honour, not a
risk to mitigate (render-plan's words, and they hold). But `electron/main.cjs` force-enables GPU
under WSLg via ANGLE, and a software GL path would be *slower* than the CPU renderer we have.
Keep `renderTerrainView` alive as the fallback when `getContext('webgl2')` returns null, and
because `pathfind`/`deposits`/`leaves` need the CPU model regardless.

## Not in scope

`simulate()` and the whole of `src/game/`'s gameplay math. The DOM/canvas split from
render-plan step 1. The HUD.
