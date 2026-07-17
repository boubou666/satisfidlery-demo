# Render plan — moving the map's flow layer to canvas, and water to the GPU

Status: **step 1 in progress**, step 2 planned. Written 2026-07-16.

**Progress — step 1 is complete** (bar the layering follow-up below).

- ✅ `mapSpace.ts` — the coordinate space and belt-route geometry, extracted from `MapView.tsx`
  (~190 lines) so the DOM layers and the canvas draw from one source. The unused `_zoom` params
  the survey found on `portLayer`/`beltNubWorld`/`beltRenderAnchors`/`beltStyledPath` are gone.
- ✅ `iconAtlas.ts` — item SVGs rasterized to `ImageBitmap`, cached per (id, size, dpr).
- ✅ `mapDraw.ts` + `.map-flow-under` / `.map-flow-over` — the whole flow layer is off the DOM:
  powerline wires, belt bands, the scrolling surface, bend discs, ports, and the riding items.
  Two canvases because the layer straddles the markers and the SVG z-order had to survive.
- ✅ Hit testing + hover + pointer surgery — `.belt-hit`/`.powerline-hit` replaced by mode-gated
  point-to-polyline tests; `.map-belts`/`.map-powerlines` dropped from the `closest()` bail.
- ✅ ~120 lines of orphaned CSS removed. MapView 3261 → 2931 lines; CSS 76.4 → 74.6 kB.

**Verified** (headless, seeded factory with two belts, three wires, an energized grid):
belt world y −60 → expected screen y **211.6**, DOM disc **212.5**, canvas band **212**;
powerline world y −260 → expected **100.3**, canvas **100**. Zero `.belt-item` DOM nodes (was 3
per item). Belt mode on → band renders red + `flow-hover` cursor; a click removes the belt
(2 → 1) and refunds. No regressions after the CSS cleanup.

- ✅ **Avatar layering restored** — and *not* by moving the avatar to canvas, which the first
  draft of this plan proposed. `PlayerMarker`'s own docs record that an rAF loop there was tried
  and was fragile, and that handing the tween to the compositor is deliberate; the avatar is one
  node, so it has none of the per-item cost that justified moving the items. Instead it now lives
  in `.map-player-layer` — a second copy of `.map-world`'s transform stacked above the flow
  canvas — which restores its `z-index: 3` position over the items while keeping the CSS
  transition. The interaction reach was split out into `PlayerReach` and left in `.map-world`,
  because it belongs UNDER the machine discs (`.map-reach-pos`, `z-index: 1`) — moving it up with
  the avatar would have traded one layering bug for another. Verified with the avatar standing on
  a running belt: figure over the chips, ring behind them and over the band.

The final stack, matching the pre-canvas order exactly:

    .map-terrain (z0) → .map-water (z0) → .map-flow-under (z0)   wires, belt tracks
    → .map-world (z1)                                            reach ring, nodes, ghosts, drafts
    → .map-flow-over (z2)                                        belt items, then ports
    → .map-player-layer (z3)                                     the avatar

**Parity is the rule, and it was broken three times**

This is a renderer swap: the map must look and behave exactly as it did. Every deviation below was
caught *after* the fact — twice by the user, once by an audit — and each came from the same
mistake: deciding the old behaviour was wrong and quietly improving it mid-refactor. Flagging a
change is not the same as asking for one. If something here looks like a bug, leave it, and raise
it as its own change.

- **Belt item tweening.** The DOM tweened `translate3d` — the item's POSITION — between tick
  positions, so an item crossing a corner within one tick cuts the chord. The canvas first tweened
  the FRACTION along the belt, which hugs the track instead. Identical on straight belts, visibly
  different on curved/90° ones. Reverted to position tweening.
- **The `belts` store subscription.** Removed as an "unused variable" once the canvas drew belts
  from the store directly — but it existed for its RE-RENDER side effect (its comment said so, and
  `useGame((s) => s.inventory)` above it is the same idiom). Without it the DOM's belt-mode
  affordances (`nearestBeltDest`, which ports are taken, the confirm popover) went stale the moment
  a belt was built. Restored.
- **The bend disc stroke** and **the hot-port glow colour** — see the notes below and in
  `mapDraw.ts`.

**Outstanding**

- ⚠️ **Not verified**: tween *smoothness*. `--virtual-time-budget` doesn't render compositor
  frames, so the headless flow can only check per-tick target values. Needs a human eye on
  `npm run window`.
- 🐛 **Latent bug found, deliberately not reproduced**: the SVG bend disc set no `stroke-width`,
  so it inherited the default of 1 *layer* unit and its ring grew with zoom — while the `r`
  beside it was explicitly `/zoom`'d to stay constant on screen. The canvas draws the constant
  1px hairline the rest of that geometry implies. Only hand-placed bends showed it, which is
  probably why it went unnoticed.

This document exists because two questions came up together — *"is React going to hold up
as factories get big?"* and *"can we get more visually ambitious?"* — and the honest answer
to both turned out to be the same, and it wasn't "port to Unity".

## Why not Unity

Considered and rejected. The short version:

- **`src/game/` is pure.** Grep it for `document.`, `window.`, or a React import: nothing.
  All ~8.4k lines of engine/terrain/pathfind/power/belts/deposits/inventory are self-contained
  math. That's the one thing that would survive a port intact — which is exactly why the port
  isn't needed.
- **The UI is ~12k lines of TSX + 4.8k lines of CSS.** All of it would be thrown away, and
  Unity's own answer to UI (UI Toolkit) is a DOM reimplementation — USS *is* CSS, UXML *is*
  HTML. We'd spend a year rebuilding, worse, the thing we already have.
- **We'd lose the web demo** (Unity WebGL bundles are heavy) and the iteration loop.
- **The perf problem isn't React** (see below), so the port wouldn't even fix the thing that
  prompted it.

Unity stays feasible if console ports ever come up — `src/game/` being pure keeps that door
open at no cost. Nothing here closes it.

## What the perf problem actually is

Not React. The 100ms tick is fine, and the two fights that *would* have been React's fault
are already won — see the comments at `MapView.tsx:425` (avatar interpolation handed to the
compositor) and `MapView.tsx:585` (selective subscription so MapView doesn't re-render 10×/s
and starve the rAF).

The cliff is **DOM node count for particle-count entities**. `MapView.tsx:520`:

```tsx
return belt.items.map((it) => (
  <div className="map-marker-pos belt-item" style={{ transform: `translate3d(...)` }}>
    <div className="belt-item-inner" ...><ItemIcon id={it.item} size={16} />
```

Three DOM nodes per item on a belt, each with its own transform and CSS transition. Fine
today. At a genre-normal endgame — ~100 belts × ~20 items — that's ~2,000 items, **6,000+ DOM
nodes**, all reconciled and re-composited every frame. That dies well before factories get as
big as the game is designed to support.

Unity would fix this by deleting the DOM. So does canvas, and we already have two canvases
running.

## The scoping decision: the flow layer, not the entity layer

The entities inside `.map-world` split into two groups with opposite characteristics. This
split is the whole plan; get it wrong and the work balloons by 4× for no gain.

### Group A — the flow layer → **canvas**

| Entity | MapView.tsx | Count |
|---|---|---|
| Powerline wire | 1809–1816 | unbounded |
| Belt band | 1859 | unbounded |
| Belt flow centerline | 1860–1870 | unbounded |
| Belt bend dot | 1871–1874 | unbounded |
| Belt port nub (stub + disc + chevron) | 1895–1941 | unbounded (~1–3 per machine) |
| Belt item | 508–539, rendered 2534 | **unbounded — the cliff** |

Every one is `aria-hidden` or `pointer-events:none`, carries **no text**, and has **no click
handler**. This is 100% of the perf problem and ~0% of the accessibility surface.

### Group B — the nodes → **stay DOM**

HUB (1512), sources (1547), buildings (1612), revealed deposits (1661), poles (1695), leaf
patches (2406).

Every one carries i18n text, a `title=` tooltip, a click handler, and is a real `<button>` —
so it gets tab focus and Enter/Space activation for free. Between them they hold ~30 `role=`
and ~32 `aria-label`. Their count scales with *buildings placed* (hundreds at most), and they
re-render on state change, not per frame.

Moving them to canvas would cost every tooltip, all keyboard reachability, and six CSS
`:hover` rules — to fix a bottleneck that doesn't exist. **Don't.**

Also staying DOM: every ghost/draft/build overlay (1983–2371), the player marker, the scan
ring, and all screen-space chrome (fabs, popovers, hints, confirms). Fixed or small counts.

### Dead code found while surveying — delete, don't port

- `CLUSTER_SCREEN_PX = 0` (`:370`) disables clustering, so the cluster scrim (2470) and
  cluster bubble (2496) are inert paths.
- `drag.current.onNode` is written `false` at `:1058`/`:1114` and **never set true**. Node
  clicks are excluded by the `closest()` bail instead.

## Step 1 — flow layer → Canvas2D

A third viewport-sized canvas, `.map-entities`, between `.map-water` and `.map-world`.

### Work items

1. **Icon atlas.** `ItemIcon` is a React SVG component; canvas needs `ImageBitmap`s cached by
   `(id, size)`. 12 items today. Note the CLAUDE.md rule — icons are SVG, never emoji — so the
   atlas rasterizes the same source art the HUD uses; there's no second art path, just a second
   *render* path.
2. **Draw loop.** Paint order: wires → belt bands → flow → bend dots → ports → items. Reuses
   `beltStyledPath` / `segmentLengths` / `pointAtFraction` unchanged. The `_zoom` params on
   `portLayer` / `beltNubWorld` / `beltRenderAnchors` are already unused — that geometry is
   pure world-space math and ports as-is.
3. **Tick interpolation.** ⚠️ *The hidden work.* `.map-marker-pos { transition: transform .12s
   linear }` (`index.css:3450`) is what turns the 10Hz sim into 60fps motion — CSS is currently
   the animation engine, for free. On canvas we implement it: keep prev/next tick positions per
   item and lerp on elapsed.
4. **Belt flow dash parity.** The scrolling `stroke-dashoffset` keyframe becomes a manual
   `lineDashOffset` off the wall clock. `BELT_FLOW_DASH/OFFSET/DURATION` (`:127–130`) are
   speed-matched to `BELT_MK1_SPEED`; preserve the "dashes are world units so they cancel the
   zoom" invariant or the belt surface visibly desyncs from the items riding it.
5. **Hit testing + hover.** Only the two mode-gated hit strokes need it: `.powerline-hit`
   (power mode only, `:1793`) and `.belt-hit` (belt mode only, `:1845`). Point-to-polyline
   distance at the current `16/zoom` and `18/zoom` tolerances. Requires a new `hoveredId` —
   there is no hovered-entity state today, because `.powerline-hit:hover + .powerline-wire`
   (`index.css:4180`) and `.belt-hit:hover + .belt-band` (`index.css:4359`) do the "this will
   be removed" red via sibling selector.
6. **Pointer routing surgery.** ⚠️ *Highest risk.* The `closest()` call at `:1100` is the only
   thing separating "pan the map" from "click a thing". Drop `.map-belts` / `.map-powerlines`
   from that selector and run the mode-gated hit test before starting a drag. Get the guards
   wrong and panning breaks over every wire.
7. **Parity polish.** `drop-shadow` on bands/wires → canvas `shadowBlur` is per-draw and
   expensive; bake the glow into the sprite or drop it. `@keyframes powerGlow` becomes a sine.

### Free wins

- `markerStyle`'s pixel-snapping workaround (`:337`) evaporates — canvas draws in float.
- Belt items stop allocating 3 DOM nodes each.

### Renderer choice: Canvas2D now, defer Pixi

PixiJS would let step 2's water be a filter on the same context. Not yet — **the expensive
parts of step 1 are renderer-agnostic**: the atlas, tick interpolation, hit testing, hover
state, and pointer surgery are ~80% of the work and don't change if we later swap to Pixi.
Only the ~200-line draw loop does. Canvas2D `drawImage` handles 5–10k sprites at 60fps;
2,000 belt items sit well inside that.

Keep the draw loop behind a small interface. Reach for Pixi when profiling or a specific
effect demands it, not before.

## Step 2 — water → WebGL2

`renderWater` (`src/game/waterfx.ts`) is already a fragment shader written in TypeScript: a
per-cell loop over `cols × rows` at `cell = 6`, purely `(world position, time) → light`. At
1080p that's ~57,600 cells/frame, each doing `fbm` (2 octaves → 2 `valueNoise` → 8 `hash2`)
on rivers — on the CPU, every frame.

### Work items

1. **`noise.ts` → GLSL.** `hash2` / `smootherstep` / `smoothstep` / `valueNoise` / `fbm` are
   ~50 lines that translate almost verbatim.
2. **`WaterField` → textures.** `kind: Uint8Array` → R8UI, `fx`/`fy` → RG16F, `shore` → R8.
   Uploaded only on camera move — the CPU terrain pass still produces the field exactly as
   today.
3. **Shader + uniforms.** `ox`, `oy`, `worldPerPx`, `t`, `seed`, `detail`. The `renderWater`
   body becomes the fragment main nearly line-for-line.
4. **Parity + zoom fade.** The `DETAIL_FULL_WPP` / `DETAIL_GONE_WPP` smoothstep fade
   (`waterfx.ts`) becomes a uniform.

### ⚠️ Requires WebGL2, not WebGL1

`hash2` depends on `Math.imul` and `>>> 0` — 32-bit **unsigned integer bitwise** ops. GLSL
ES 1.0 (WebGL1) has no integer bitwise operators, so the hash cannot be reproduced exactly
there. GLSL ES 3.00 (WebGL2) can. Exactness is load-bearing: terrain and water must agree on
the same field, and a drifting hash puts glints on land. Electron and every modern browser
have WebGL2, so this is a constraint to honour, not a risk to mitigate.

### Payoff

Per-6px-cell on the CPU → **per-pixel on the GPU**. Strictly better-looking *and* essentially
free. That's the headroom for particles, bloom, animated belt flow, and day/night — the
visual ambition this was really about.

## What this does not touch

`src/game/` stays pure. `waterfx.ts` remains a pure function — it just gets compiled to GLSL
instead of interpreted. `simulate()` is untouched. The HUD (Inventory, HubView, BuildingList,
SplitStack, OptionsModal) stays React, because it is a widget tree — scrolling containers, a
real accessibility tree, cross-panel drag-and-drop via `dnd.tsx`, and i18n text of
unpredictable length — and that is precisely what DOM is best at and canvas worst at.

Cohesion between the HUD and the world is an **art-direction** problem (palette, type,
spacing, motion), not a rendering-technology one. Half-Life 2, Factorio, and Satisfactory all
render UI in a completely different system from their world; nobody notices.

**Done in 0.35.0** — and the prediction held: the fix was type, material, and layout, with no
renderer involved. The map went full-bleed (`.app` is `position: fixed; inset: 0`; MapView's
existing `ResizeObserver` absorbed it), the HUD became panels plated over the live world, and
the DOM/canvas split above is untouched. The one engine-adjacent change was a `covered` prop
that stops the rAF loops painting behind an opaque panel — the same bargain `document.hidden`
already made. See the 0.35.0 entry in `CHANGELOG.md`.
