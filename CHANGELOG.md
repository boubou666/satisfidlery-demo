# Changelog

All notable changes to **Satisfidlery** are recorded here, newest first. Each
completed piece of work gets an entry — with a screenshot whenever the change is
visible in-game.

**Versioning** (pre-1.0, the game is in active development):

- **MAJOR** (`X.0.0`) — a landmark: the first Steam release, or a sweeping
  overhaul of the game's systems or content.
- **MINOR** (`0.X.0`) — a new player-facing feature or system.
- **PATCH** (`0.0.X`) — improvements, fixes, and tweaks to existing features.

Screenshots live in [`docs/images/changelog/`](docs/images/changelog). History
before this file is in the git log.

## [Unreleased]

_Nothing yet._

## [0.22.0] — 2026-07-15

### Changed

- **Any belt source can now feed any automatable machine — mismatched items clog instead of
  being blocked.** Previously a belt was refused unless the destination had *some* recipe that
  accepted the source's output (e.g. Copper Ore → Constructor was disallowed). Now the link is
  always allowed: the belt carries the items and, if the destination's **currently selected
  recipe** doesn't consume them, they pile up at the far end (Satisfactory-style clogging)
  until you switch the recipe. The confirm popover's blocking "doesn't accept…" error is
  replaced by a non-blocking heads-up ("_…current recipe won't use this — it'll pile up on the
  belt_") and the **Build belt** button is enabled. The engine already clogged on a recipe
  mismatch; this removes the load-time gate so genuinely-unused items ride the belt too.

### Added

- **One belt per output port.** A machine can now run at most one outgoing belt per output
  slot — a single-output machine (or a Miner Mk1, which has one output) is capped at a single
  belt out. Trying to run a second is refused with "_That output already feeds a belt — one
  belt per output._" (Destinations are unchanged — a machine's input can still merge several
  belts.)

![Belt to any machine, clogging warning](docs/images/changelog/v0.22.0-belt-any-clog.png)

## [0.21.9] — 2026-07-15

### Added

- **A live "ghost" preview while routing a conveyor belt.** Once you arm a belt source, a
  faint dashed segment now trails from the last placed point (the source, or the most recent
  bend) to your cursor, so you can see exactly where the belt will run before you commit the
  destination or drop a bend. It's fainter than the committed route with a softly pulsing tip
  dot, and it disappears the moment you pick a destination (the real route to the machine takes
  over). Implemented by tracking the pointer in world space on `onPointerMove` (skipped while
  actively panning so the ghost doesn't chase the drag) and appending it as the draft's tail.

![Belt ghost preview](docs/images/changelog/v0.21.9-belt-ghost-preview.png)

## [0.21.8] — 2026-07-15

### Fixed

- **You can now pan the map while placing powerlines or conveyor belts.** In power/belt mode
  the full-layer overlay SVG (`.map-powerlines` / `.map-belts`) had `pointer-events: auto` on
  its root, so it swallowed every pointer-down across the whole map — and since those classes
  are in the drag-to-pan exclusion list, dragging never started. The overlay root is now inert
  (`pointer-events: none`) and only the thin per-line hit strokes (`.powerline-hit` /
  `.belt-hit`, rendered only in those modes) opt back in, so dragging empty ground pans as
  usual while a line can still be clicked to remove. Empty-ground taps still drop a pole / bend
  a belt route. (No screenshot: it's an interaction change a still frame can't show.)

## [0.21.7] — 2026-07-15

### Changed

- **The browser context menu is now disabled app-wide.** A right-click anywhere in the game
  (map, HUB, inventory, header — everywhere) no longer pops the native menu (Back / Reload /
  Save image / Inspect …); this is a game, not a document. Done with a single document-level
  listener in `main.tsx`, replacing the map-only suppression added in 0.21.6 (the map's
  right-click-doesn't-move-the-player behavior stays). (No screenshot: it's an interaction
  change a still frame can't show.)

## [0.21.6] — 2026-07-15

### Changed

- **Right-click does nothing on the map.** Previously a right-click on empty ground both
  walked the player there *and* popped the browser's context menu. The map now only responds
  to the primary button (left mouse / touch / pen) — right- and middle-clicks start no pan or
  travel — and the browser context menu is suppressed over the map viewport. Left-click
  tap-to-move is unchanged. (No screenshot: it's an interaction change a still frame can't show.)

## [0.21.5] — 2026-07-15

### Changed

- **Player walks 4× faster.** `MOVE_SPEED` went from 20 to 80 world-units/second, so every
  walk across the map — to a deposit, a building, the HUB, a scan result — takes a quarter of
  the time it used to. Travel time scales straight off this one constant (`time = distance /
  MOVE_SPEED`), so nothing else needed tuning. (No screenshot: a still frame can't show motion.)

## [0.21.4] — 2026-07-15

### Added

- **Smelt Copper Ingots by hand at the HUB.** Copper smelting first becomes possible at the
  **Field Fabrication** milestone (it's when copper scanning — and hand-crafting — come online),
  so from that moment Copper Ingots join Iron Ingots as a HUB hand-craft: 1 Copper Ore → 1
  Copper Ingot, 3s per cycle, no Smelter required. Nothing else changes — the Smelter still
  smelts copper too; this just means you're no longer forced to build one before you can turn
  your first copper ore into ingots (and feed the Wire recipe).

![The HUB's Hand-Crafting panel now offers both an Iron Ingot and a Copper Ingot tile once Field Fabrication is built](docs/images/changelog/v0.21.4-copper-ingot-hub-craft.png)

## [0.21.3] — 2026-07-15

### Changed

- **A clogged belt now packs completely full.** Previously stuck items sat at the belt's
  wide throughput spacing (~160 m apart), so a jammed belt only ever held a handful. Items
  now bunch up at their true physical size (~24 m), so a backed-up belt fills **end-to-end**
  with items until there's genuinely no room left — and only then does the source machine
  stop loading onto it. A freely-flowing belt is still sparse and still moves 30 items/min;
  this only changes how densely items pile when they can't get off.

![A conveyor belt packed end-to-end with 20 Iron Ingots, clogged because the destination Constructor's recipe won't accept them](docs/images/changelog/v0.21.3-belt-packs-full.png)

## [0.21.2] — 2026-07-15

### Changed

- **Belts clog like Satisfactory when the item doesn't match the recipe.** An item now
  enters a machine only if the machine's **currently-selected recipe** uses it. Deliver Iron
  Ore to a Smelter set to make Copper Ingots and the ore no longer vanishes into the input —
  it **gets stuck at the end of the belt**, and the items behind advance until they bump into
  it and pile up. Switch the machine's recipe to one that uses the item and the clog clears
  and flows in. (A belt full of stuck items also stops the source machine from loading more
  onto it — backpressure all the way to the source.)
- **Removing a belt gives back its cargo too.** Deleting a conveyor belt now refunds the iron
  plates it cost **and** every item currently riding it — nothing on the belt is lost.
- **Dismantling a machine no longer destroys its belts.** Tearing down a building (or a Miner
  Mk1) on either end of a belt leaves the belt in place; it simply dangles (items still drain
  toward a surviving end, or pile up at a removed one) until you remove it by hand. Building
  ids are now stable so a rebuilt machine never silently inherits a leftover belt.

![A conveyor belt carrying Iron Ingots from a Smelter toward a Constructor across the world map](docs/images/changelog/v0.21.2-belt-clog.png)

## [0.21.1] — 2026-07-15

### Fixed

- **Belt-delivered items now show in the destination's input.** A building card only ever
  drew input slots for its *currently-selected* recipe, so items a conveyor belt dropped in
  that the current recipe doesn't use (e.g. Iron Ingot arriving at a Constructor set to make
  Wire) landed in the input buffer but were **invisible** — they looked like they "weren't
  added." The card now surfaces a slot for **every** item sitting in the input buffer, so
  belt-fed items are always visible and can be unloaded.

### Changed

- **Building inputs now have a max stack (500), like outputs.** Each input resource caps at
  500 — shown as `X / 500` on the slot — enforced for hand-loading too (Load buttons disable
  when a slot is full). A full input backs its feeding belt up to the source, which then
  idles, mirroring how a full output already backs a machine up.
- **The conveyor surface animation now scrolls at the item travel speed.** The belt's moving
  texture was running at a fixed rate unrelated to how fast items actually travel; it now
  matches the item speed exactly, at every zoom level, so the belt and the items on it move
  together.

![A Constructor's input slot showing 4 / 500 Iron Ingot delivered by a conveyor belt, with the new per-input max-stack cap visible](docs/images/changelog/v0.21.1-belt-input-visible.png)

## [0.21.0] — 2026-07-15

### Added

- **Conveyor Belt MK1 — automatic item logistics.** A new HUB milestone, **Conveyor
  Logistics** (unlocked by submitting iron plates, iron rods, wire, cable, and concrete),
  brings the first hands-free way to move items between machines. On the map, a new **Belt**
  mode lets you draw a belt by hand: **click a source** (a Smelter, a Constructor, or a
  Miner Mk1 ore node), **tap empty ground** to bend the route as many times as you like,
  then **click a destination machine** to finish. It costs iron plates by length (≈1 plate
  per 120 m, refunded on teardown) and — like a powerline — **lays itself segment by
  segment** over a few seconds.
- **Items ride the belt.** Once a belt is fully laid it pulls the source's output and
  carries it to the destination's input at **30 items/min**, one whole item at a time. The
  items genuinely **travel** — a longer belt takes longer to cross — and you watch them
  glide along the track on the map. A backed-up destination (input full) makes the belt back
  up all the way to the source, which then idles; draining the destination gets it flowing
  again. Belts run live and offline, and dismantling a machine (or a Miner Mk1) pulls the
  belts attached to it, refunding their plates. The Equipment Workshop is not belt-connectable.

![A conveyor belt running from a Smelter through two hand-placed bends to a Constructor, with Iron Ingots riding along it, and the new Belt tool in the map toolbar](docs/images/changelog/v0.21.0-conveyor-belts.png)

## [0.20.4] — 2026-07-15

### Fixed

- **Unload buttons are disabled when the inventory is full too.** Extending the previous
  fix to every "move items back to the grid" action: a building input's **Unload**, a
  generator's (Biomass Burner) leaf-fuel **Unload**, and the HUB burner bank's fuel
  **Unload** now disable when the grid has no room for what they'd return — same
  "Inventory full — free up space to collect" tooltip. Previously these clicked through to
  a silent no-op on a full grid. Room is per-item, so freeing a single slot re-enables them.

![A Smelter and a Biomass Burner on a full 17 / 17 grid, with the Smelter's input Unload, its output Collect, and the burner's fuel Unload all greyed out](docs/images/changelog/v0.20.4-unload-disabled-when-full.png)

## [0.20.3] — 2026-07-15

### Fixed

- **Collect buttons are disabled when the inventory is full.** A source's **Collect**
  (and a building's output **Collect**) button stayed clickable even with no room in the
  grid — clicking just did nothing (the transfer is a lossless partial-fill, so a full
  grid moves zero). The buttons are now **disabled** whenever the grid has no room for
  what they'd collect, with an "Inventory full — free up space to collect" tooltip, so the
  dead click is impossible. Room is per-item: as soon as a single unit fits again (a stack
  freed or partially collected elsewhere), the button re-enables.

![A source holding 122 Iron Ore with its Collect button greyed out because the inventory grid is full at 17 / 17 stacks](docs/images/changelog/v0.20.3-collect-disabled-when-full.png)

## [0.20.2] — 2026-07-15

### Fixed

- **Miners never deliver a partial ore.** A **poor** deposit yields ×0.5, so a mining
  cycle produced **half an ore** — which accumulated into the buffer (and then the
  inventory on collect) as fractional counts like `9.5 Iron Ore`. Mining now works like
  the burner's whole-leaf burn: a source's progress bar counts toward the next **whole
  ore**, advancing at the deposit's quality-scaled rate (yield ÷ cycle time, boosted for
  a Miner Mk1), and only whole ore is ever added to the buffer. A poor node simply takes
  twice as long to hand over each ore — the bar completing now **always** means exactly
  one ore. Rich (pure ×2) and normal nodes are unchanged in rate. The source's rate label
  reflects this too: a poor node reads **"1 per 4s"** instead of a fractional "0.5 per 2s".
  Same average throughput as before, but a collected stack can never be fractional.

![A poor-quality Iron Ore source being hand-mined, its rate reading "1 per 4s" and its buffer holding a whole 9 Iron Ore](docs/images/changelog/v0.20.2-whole-ore-mining.png)

## [0.20.1] — 2026-07-15

### Fixed

- **Leaves are burned as whole items — no more fractional leaf counts.** A running
  Biomass Burner (and the HUB burner bank) drained its leaf fuel *continuously*, so its
  fuel store sat at values like `97.86`. Loading more leaves into a partly-burnt store
  then computed a fractional `room` (`cap − fuel`) and moved a fractional quantity out
  of the inventory — leaving a floating stack behind (e.g. **2.14 leaves**). Fuel is now
  a **discrete whole-leaf count**: the burn advances a `burnProgress` toward the next
  leaf (mirroring how mining and smelting accumulate toward the next unit) and deducts a
  **whole leaf** each time one completes, so a leaf in a generator is either fully present
  or fully spent — never half-consumed. Loading only ever moves whole leaves, so the
  inventory can never hold a fraction. The fuel countdown ("empties in …") accounts for
  the leaf mid-burn. Loading a save cleans up any legacy fractional fuel *and* any
  floating item stack left by the old behavior (both are floored to whole units).

![A running Biomass Burner showing a whole 7 / 100 leaves of fuel ("empties in 2:05") and an inventory holding a clean whole stack of 20 leaves](docs/images/changelog/v0.20.1-discrete-leaf-fuel.png)

## [0.20.0] — 2026-07-15

### Added

- **Buildings back up when their output is full — and stop drawing power.** A placed
  machine's output buffer now caps at **500** units of a product (`BUILDING_MAX_BUFFER`).
  Once there's no room for another cycle, the machine idles: it stops producing and, if
  it's wired to the grid, drops out of the grid's demand entirely — so it draws **no
  power** and a generator feeding it stops spending its capacity on it (exactly how a
  full **Miner Mk1** already idles and stops drawing). Collect the backlog and it resumes.
  The building card shows an **"Output full — collect to resume"** status, and the cap is
  respected live and offline (a long offline stretch can't overshoot it). Previously a
  building ran and drew power forever while it had inputs, hoarding output without bound.

![A Smelter wired to a Biomass Burner, its output buffer full at 500 Iron Ingot, showing "Output full — collect to resume" and drawing no power](docs/images/changelog/v0.20.0-output-full-idle.png)

## [0.19.2] — 2026-07-15

### Fixed

- **Build radial nodes drifted off the ring.** Each option was positioned by its whole
  box (disc **+** label **+** cost), so a node with more cost lines — the Biomass Burner
  carries three — pushed its disc higher than its neighbours, and the four-building menu
  looked lopsided. Each node is now anchored by its **disc**, with the label and cost
  floating below it, so all discs sit exactly on the ring (a clean, symmetric cross);
  bumped the ring radius a touch for breathing room.

![The build radial with four options — Smelter, Biomass Burner, Constructor, Equipment Workshop — discs evenly aligned on the ring](docs/images/changelog/v0.19.2-build-radial-aligned.png)

## [0.19.1] — 2026-07-15

### Fixed

- **Biomass Burner shared the Smelter's flame icon.** In the build radial and on the
  map, the new Biomass Burner rendered the generic `flame` glyph — which is nearly
  identical to the Smelter's own small-flame glyph, so the two build options were hard
  to tell apart. Gave the burner its own `burner` glyph: a furnace housing with a lit
  stoke window and stubby feet, clearly distinct from the Smelter's bare flame.

![The build radial: the Biomass Burner now shows a furnace-housing icon, distinct from the Smelter's flame](docs/images/changelog/v0.19.1-burner-icon.png)

## [0.19.0] — 2026-07-15

### Added

- **Biomass Burner — a placeable power generator.** Unlocked by the **Mining Automation**
  milestone, the Biomass Burner is the portable counterpart to the HUB burner bank: build one
  anywhere for **15 iron plates · 15 iron rods · 25 wires**, wire it into your grid, and load
  leaves to run your machines hands-free. It works exactly like the HUB burners — its own leaf
  fuel store (drag leaves onto the bar or use Load / Unload while standing at it), a master
  pause/resume, a live **30 kW** output, and a breaker that trips on overload. Each burner is a
  real generator on the network: `powerNetwork` now sums generation from the HUB **and** every
  burner building on a grid, so a smelter wired to a burner auto-runs (live and offline) off the
  30 kW it supplies.
- **Grid-wide breaker reset.** When an overloaded grid trips, resetting the breaker from **any**
  generator on it — a burner building or the HUB bank — relaunches the whole network at once
  (every generator on that grid un-trips together), instead of resetting one at a time.

Save bumped to **v23** (buildings now persist generator fuel / running / tripped state;
older saves load forward-tolerantly).

![The Buildings tab: a Biomass Burner generating 30 kW (79/100 leaves, empties in 26:32) wired to a Smelter that reads Powered · 4 kW and auto-runs on grid power](docs/images/changelog/v0.19.0-biomass-burner-building.png)

## [0.18.1] — 2026-07-15

### Fixed

- **Mining Automation milestone card showed raw string keys.** The card rendered
  the literal `hub.automation.stakes` and `hub.automation.doneHint` instead of
  text, because those i18n strings were never authored. Added them (EN + FR), and
  filled the French `hub.power.stakes` / `hub.power.doneHint` strings that were
  missing the same way — so both the Power Grid and Mining Automation milestones
  now read correctly in both languages.

## [0.18.0] — 2026-07-15

### Added

- **Inventory slots now grow with your progress.** A fresh grid starts at **12** stack slots
  (down from the old flat 24), and every **HUB milestone** you complete (Field Fabrication,
  Assembly Line, Equipment Workshop, Power Grid, Mining Automation) permanently unlocks **one
  more slot** — so building out the factory is also what expands your carrying capacity. The
  live capacity is `12 + milestones built`; the count in the Inventory header (`used / max`)
  reflects it, and the grid resizes the moment a milestone is built. Older saves are migrated
  losslessly: the grid is resized to the capacity your milestones earn, and any items that sat
  in the old larger grid are folded back into stacks rather than dropped (save bumped to v22).

![Inventory showing 6 / 14 slots — the base 12 plus one each for two completed HUB milestones](docs/images/changelog/0.18.0-inventory-slots.png)

## [0.17.0] — 2026-07-15

### Added

- **In-game changelog viewer in the demo.** The DEV BUILD banner now has a button (next to
  the version) that opens a **Changelog** panel, styled like the game's own dialogs, listing
  every version's changes right inside the game — no need to leave for GitHub. It reads the
  same `CHANGELOG.md` that ships with the demo, rendered by a small built-in Markdown reader
  (bold, `code`, and links; screenshots are omitted). Like the banner, the button and its
  changelog are **demo-only** — the changelog text is lazy-loaded on open and isn't bundled
  into the Steam/Electron build at all.

![In-game changelog viewer opened from the demo's DEV BUILD banner](docs/images/changelog/v0.17.0-demo-changelog-viewer.png)

## [0.16.0] — 2026-07-15

### Added

- **Playable web demo, published from a separate public repo.** The game's source stays in
  the private repo; a compiled, in-browser demo is now served via GitHub Pages at
  **https://boubou666.github.io/satisfidlery-demo/** from the public
  `satisfidlery-demo` repo (built artifacts + a copy of this changelog only — no source).
  `npm run deploy:demo` builds and publishes it in one step, and it re-publishes automatically
  on every MINOR/MAJOR version bump.
- **"DEV BUILD" version banner in the demo.** The public demo shows a small bottom-right
  marker with the current version (e.g. `DEV BUILD · v0.16.0`) so players always know it's a
  work-in-progress dev build and exactly which version they're on. It appears **only** in the
  demo build (`npm run build:demo`) — the Steam/Electron build and the dev server stay clean.

![Dev-build version banner in the public web demo](docs/images/changelog/v0.16.0-demo-dev-banner.png)

## [0.15.1] — 2026-07-15

### Fixed

- **Map node popover was too narrow.** The fixed 300px popover cut off the roomiest card
  row — the **Build Miner Mk1** offer (its portable-miner + plates + concrete cost chips
  alongside the button) — forcing an ugly horizontal scrollbar. The popover now sizes to
  360px (capped to the viewport) and the miner rows wrap their button if space is ever
  still tight, so nothing scrolls sideways.

## [0.15.0] — 2026-07-15

### Added

- **Tap a nearby node on the map to act on it — no tab-switching.** When you're standing
  within reach of a **deposit**, **building**, or the **HUB**, tapping its map icon now
  opens an **action popover anchored to that node**, holding **everything you can do
  there** — the exact same card the Sources / Buildings tabs show. From a deposit: Mine,
  Collect its buffer, Install/remove a Portable Miner, build/dismantle a Miner Mk1. From a
  building: run/stop, pick the recipe, Load / ½ / 1× / Load N / Unload inputs, Collect the
  output, Dismantle. The HUB popover shows its live generation and an **Open HUB panel**
  shortcut. So the map becomes the one place you play from.

  ![Tapping a nearby deposit opens its full action card right on the map](docs/images/changelog/v0.15.0-node-popover.png)

  - **In-range only** — tapping a _distant_ node still walks you toward it (as before); the
    popover appears once you're in proximity. The card is positioned in screen space, so it
    tracks the node as you pan and zoom, and is flipped/clamped to stay fully on-screen.
  - Close it with the ✕, the `Esc` key, an outside click, or by re-tapping the node.
  - Under the hood the deposit and building cards are now shared components (`SourceCard`,
    `BuildingCard`) reused by both the list tabs and the map — one source of truth, no
    duplicated logic.

## [0.14.0] — 2026-07-15

### Added

- **One-tap "Claim" button when you're standing by a revealed deposit.** The moment the
  player walks within reach of a scanned-but-unclaimed deposit, a prominent accent
  **Claim _<ore>_** button appears at the bottom-center of the map — so you no longer
  have to find the small on-node button or fan open a cluster to claim the node. It
  targets the **nearest** in-range deposit; if several are reachable it shows a
  **+N more** badge and claiming one promotes the next. Hidden while laying powerlines
  (which owns that bottom-center coach line).

  ![The Claim button appears on the map when near a claimable deposit](docs/images/changelog/v0.14.0-claim-nearby-button.png)

## [0.13.0] — 2026-07-15

### Added

- **Mining Automation milestone — Limestone, Concrete, and the Miner Mk1.** A new HUB
  milestone (unlocked after the Power Grid) that you complete mainly with **5 Portable
  Miners** plus plates and cable. It opens up a whole automation tier:

  - **Limestone** — a new ore. Once the milestone's built you can **scan for it**, and
    it joins the procedural deposit field so nodes turn up as you prospect. An authored
    limestone deposit sits near spawn to get you started.
  - **Concrete** — a new material pressed from limestone (**3 limestone → 1 concrete**),
    craftable both in the **Constructor** and by hand at the **HUB**.
  - **Miner Mk1** — a powered auto-miner you **build directly onto an ore node** (from
    the Sources tab) for **1 Portable Miner + 10 Iron Plates + 10 Concrete**. Wire it
    into the power grid from the map — the node becomes a grid endpoint — and while
    powered it **mines the node hands-free** (and offline), drawing **5 kW**. Output
    scales with the node's purity (poor/normal/pure) and it mines faster than a hand or
    Portable Miner. It **stops and draws nothing once the node's buffer fills**, and
    like a Portable Miner it disables manual mining while installed. **Dismantle it
    straight from the node** — you get the parts back and any wire is pulled.

  ![A Miner Mk1 wired to the HUB, drawing 5 kW and auto-mining a pure iron node](docs/images/changelog/v0.13.0-miner-mk1-map.png)

  ![The Sources tab: a Miner Mk1 mining a pure node, and the Build Miner Mk1 offer with its cost](docs/images/changelog/v0.13.0-miner-mk1-sources.png)

## [0.12.1] — 2026-07-15

### Fixed

- **"Infinite resources" no longer lets you duplicate materials.** The first cut of
  the cheat faked your stock (reading every item as ~1T) and skipped debits, which
  meant loading a machine while the cheat was on stuffed a phantom pile into its
  **input buffer** — so you could toggle the cheat off, **unload the machine (or a
  burner), and walk away with materials you never actually had.** The cheat is
  reworked so it can never fabricate materials into a reclaimable place: it now only
  **waives affordability** (everything is buildable/craftable regardless of what you
  hold), and **machines and hand-crafts simply run without consuming their inputs**.
  Every real transfer — loading, unloading, collecting, refunds — moves only
  materials you genuinely hold, so nothing can be laundered back into the grid.
  Bonus: the inventory and load buttons now show your true counts again (no more
  phantom "1.00T"), and you can still build the whole factory and run production
  chains from an empty inventory.

## [0.12.0] — 2026-07-15

### Added

- **"Infinite resources" cheat — a testing toggle in Options → Cheats.** Flip it on
  and the inventory behaves as bottomless: **every cost is affordable and spends
  nothing**, so you can build machines, raise milestones, run powerlines, and
  hand-craft freely — even from a completely empty inventory — without gathering a
  single ore first. It sits right below the existing Game-speed slider, persists
  across reloads (separate from your save, like the language and speed settings), and
  is off by default. Handy for exercising later systems without grinding up to them.

  ![The Infinite resources toggle in the Options → Cheats section](docs/images/changelog/v0.12.0-infinite-resources.png)

## [0.11.0] — 2026-07-15

### Added

- **Laying a powerline now takes time — watch it get built from A to B.** Connecting
  two nodes no longer snaps the whole run into existence. You pay the full cost up
  front (as before), then the connection is **strung segment-by-segment from the first
  node to the second**: the wire visibly grows along the route, and each auto-placed
  **Power Pole pops up as the wire reaches it**. Longer runs take longer to lay (each
  segment's time scales with its length), and the map shows the part still ahead as a
  faint dashed **blueprint** with ghost discs where poles will rise, plus a spark
  riding the growing tip. A live **"Laying powerline… 2/3 segments"** readout tracks
  progress. The build advances live and offline (like walking), so a long line keeps
  laying itself while you're away — and only one connection lays at a time.

  ![Laying a powerline from a smelter to a constructor](docs/images/changelog/v0.11.0-powerline-construction.png)

## [0.10.0] — 2026-07-15

### Added

- **Idle machines draw no power.** A wired building only consumes power while it's
  actually working (it has the inputs to run a cycle). An idle machine — out of
  inputs — draws nothing and no longer counts toward its grid's demand. On the map an
  idle-but-connected machine still lights up (it's on a live grid) but shows no draw
  pill; in the Buildings tab it reads **"Powered — waiting for input."**

- **Overloaded grids trip, and stay down until you reset them.** If the buildings
  working on a grid draw more than it can generate, the breaker **trips**: power cuts
  out across that grid, the buildings stop, and **the generators shut down too** (they
  stop burning fuel). It's latched — power does **not** come back on its own. The map's
  balance readout turns red with a **Reset** button (and the HUB burner panel shows a
  **Reset breaker** control); resetting only holds once you've shed enough load (unwire
  a machine, or let one run dry), otherwise it trips straight back.

- **Grids are evaluated independently, per network.** Power is now computed per
  connected component rather than assuming a single HUB grid — groundwork for multiple
  separate power grids. Each grid weighs its own generation against its own working
  demand; only a grid with a generator (the HUB burner bank, for now) can carry power.

### Fixed

- **The HUB power graph no longer resets when you switch tabs.** Its sample history
  moved out of the graph component (which unmounts when you leave the HUB) into a
  module-level sampler that keeps running, so the trace stays continuous across tab
  changes instead of rebuilding from empty each time you return.

![Overloaded grid tripped — power down, reset required](docs/images/changelog/v0.10.0-grid-tripped.png)

### Changed

- **The map now tells generation from consumption at a glance.** The kW pills used to
  all look the same warm gold, so you couldn't tell what was *feeding* the grid from
  what was *pulling* off it. Now **generation** (the HUB burners) shows a **green
  `+40 kW`** pill with a flame, and each wired machine shows an **amber `−4 kW` /
  `−6 kW`** draw pill with a bolt. A new **grid-balance readout** in the map's
  top-right corner sums it up — total **`+gen`** (green) vs. total **`−draw`** (amber)
  across the HUB network — and turns red with an **"Overloaded"** tag when demand
  exceeds supply (the network trips).

  ![Generation vs. draw on the map](docs/images/changelog/v0.9.1-gen-vs-draw.png)

## [0.9.0] — 2026-07-14

### Added

- **Power now flows — wired buildings run themselves.** The power grid stops being
  purely structural: a **Smelter** (draws **4 kW**) or a **Constructor** (draws
  **6 kW**) can be wired into a network, and when that network carries running power
  it **runs automatically** — hands-off, and while you're offline — instead of being
  operated by hand. Only these two building types are connectable; a hand-only bench
  (the Equipment Workshop) has no power intake and stays inert in Powerline mode.

  Power flows out from the **HUB's biomass burners** (the sole generator for now).
  The HUB component energizes when its generation covers the **summed draw** of every
  building wired onto it; if demand ever exceeds supply the whole network trips and
  nothing on it runs (a Satisfactory-style blackout), and it comes back the moment
  supply recovers (e.g. you load more fuel).

  **On the map**, a live network reads at a glance: energized **wires glow gold and
  pulse**, powered **poles and building discs light up**, and each running machine
  shows its **kW draw** in a pill (beside the HUB's kW generation readout). **In the
  Buildings tab**, a powered machine swaps its manual Run control for a **"⚡ Powered ·
  N kW"** badge and reads **"Auto-running (grid power)…"** — you still load its inputs
  and collect its outputs by hand, but it can't be hand-started while it's on the grid.

  ![Powered grid on the map](docs/images/changelog/v0.9.0-powered-grid.png)

## [0.8.3] — 2026-07-14

### Changed

- **The burner fuel bar now shows a live "empties in X" countdown.** Beside the fuel
  count it reads **"40 / 100 leaves · empties in 10:00"** — the time until the loaded fuel
  runs out at the current draw, ticking down as it burns. It's derived from the amount
  loaded and the **fuel type's energy** (a leaf holds 600 kJ → 30 s in one burner, so
  40 leaves across the 2-burner bank last 10 min), so when other fuels arrive the estimate
  will reflect whatever is loaded. Shows **· paused** when the bank is paused (the bar
  isn't draining) and **· empty** when it's out. Replaces the static "30s per leaf" text.

  ![The burner fuel bar reading "40 / 100 leaves · empties in 10:00"](docs/images/changelog/v0.8.3-fuel-countdown.png)

## [0.8.2] — 2026-07-14

### Changed

- **The burner fuel readout now shows how long ONE leaf lasts, not a whole-store figure.**
  Next to the fuel count it reads **"39 / 100 leaves · 30s per leaf"**, and the info line
  is now **"1 leaf ≈ 600 kJ — one leaf powers a burner for 30 s"** (previously phrased as a
  consumption cadence, "burns a leaf every 30s", which read as the whole store). The number
  is unchanged — a single leaf powers one burner for 30 s — just stated per-leaf and put
  where you look while loading fuel.

  ![The Biomass Burners fuel row reading "39 / 100 leaves · 30s per leaf" with the per-leaf burn-time info line](docs/images/changelog/v0.8.2-one-leaf-burn-time.png)

## [0.8.1] — 2026-07-14

### Changed

- **Biomass Burners burn leaves far slower, and loading them is easier.** A leaf now
  holds **600 kJ** (was 300), so a single leaf powers a burner for **30 s** instead of
  15 — a full bank of two burns one leaf every 15 s. Leaves last long enough that a
  forage run actually keeps the burners fed. Loading fuel gained two conveniences on the
  HUB panel: **drag a leaf stack straight from your inventory onto the fuel bar** to load
  it, and a **Qty field + Load** to load an exact number of leaves (alongside the existing
  **Load all** / **Unload**).

  ![The Biomass Burners panel with drag-to-load fuel bar, a quantity field, and the updated 600 kJ / 30 s burn rate](docs/images/changelog/v0.8.1-burner-fuel-loading.png)

## [0.8.0] — 2026-07-14

### Added

- **Biomass Burners — the first power generation.** Completing the **Power Grid**
  milestone now also brings **two Biomass Burners online directly in the HUB** (they'll
  become placeable buildings in a later feature). Each burner generates **20 kW** while
  running and fueled — **40 kW** for the pair. For now the only fuel is **leaves**: a leaf
  carries **300 kJ**, and that energy sets the burn speed — a burner draws 20 kW, so it
  eats one leaf every **15 s** (the bank of two burns one every 7.5 s). Load leaves into
  the bank's shared fuel store at the HUB and it burns down over time — live and while
  you're away (like a Portable Miner).

  A new **HUB panel** shows the bank: its live **kW output**, a rolling **graph of power
  generated over the last 30 s**, the leaf fuel store with **Load / Unload**, a master
  **pause/resume** switch (pausing stops both burning and generation), and the joules
  explainer. While the burners are running, the **HUB node on the world map** wears a
  glowing **⚡ kW badge** so you can see it's generating at a glance.

  Power isn't consumed by anything yet — this is the generation groundwork (and the graph
  payoff). Existing saves that already built the Power Grid get their two burners on load.

  ![The HUB's Biomass Burners panel: 40 kW output with a live power graph, leaf fuel bar, Load/Unload, and a pause switch](docs/images/changelog/v0.8.0-biomass-burners.png)

  ![The HUB node on the world map showing a 40 kW badge while the burners run](docs/images/changelog/v0.8.0-burners-on-map.png)

## [0.7.8] — 2026-07-14

### Fixed

- **Power poles now sit exactly on their wire at every zoom.** Auto-inserted poles
  (the ones dropped along a too-long powerline) looked slightly off their line when
  zoomed in — the disc drifted a pixel or two off the wire it belonged to. Cause: map
  markers were anchored with CSS `left`/`top`, which the browser **pixel-snaps** inside
  the scaled world layer; the wires are SVG coordinates and keep full float precision,
  so the tiny snap gap between them got magnified by the zoom (invisible zoomed out,
  ~1px+ zoomed all the way in, more on a large/hi-DPI display). Markers are now anchored
  with a CSS `translate()` (transforms aren't snapped) instead of `left`/`top`, so the
  disc lands on its wire endpoint to the sub-pixel at any zoom — measured **0.98px →
  0.00px** offset at max zoom. Applies to every map marker, not just poles.

  ![A power pole disc sitting exactly on the vertex where its two wires meet, zoomed in](docs/images/changelog/v0.7.8-pole-on-wire.png)

## [0.7.7] — 2026-07-14

### Changed

- **Disassembling the drop pod now takes time — it's a timed manual action, not a
  click.** Stripping the crash-landed pod for its salvage used to grant the 100 iron
  ingots instantly. It's now the game's opening **12-second timed action**, in the
  same idiom as mining, smelting, hand-crafting, foraging, and scanning: a progress
  bar runs to completion (`DROP_POD.disassembleSeconds`), the salvage lands only when
  the bar fills, and the button reads **Disassembling…** meanwhile. Like every other
  manual action it's **exclusive** — starting anything else stops it, and starting it
  stops everything else (one action at a time). It advances live and offline through
  a new `simulatePod()` (which — like `simulateCraft`/`simulateForage` — owns the
  inventory write the salvage needs), so leaving mid-strip resumes on return. If the
  grid can't hold the full salvage the bar holds full rather than voiding anything.
  Save shape bumped to **v17** (the in-progress disassembly persists); older saves
  load idle.

  ![The drop pod mid-disassembly: a progress bar partway across, the button reading "Disassembling…", salvage listed as 100 Iron Ingot · 12s](docs/images/changelog/v0.7.7-pod-disassembly.png)

## [0.7.6] — 2026-07-14

### Fixed

- **Power poles now always sit on their powerline (no more poles floating beside
  the wire).** Poles were treated like any other map marker and folded into the
  on-screen **clustering**: when several poles landed close together (a long line
  zoomed out), they collapsed into a count bubble, and tapping it **fanned the
  discs out into a ring** — up to 66px away from where their wires actually connect.
  So a *connected* pole would render well off its straight powerline (measured
  47–65px off after a fan). Poles are now excluded from clustering entirely and
  render at their true position, so each disc sits exactly on its wire (verified
  0.0–0.2px) at every zoom. Only the big selectable nodes (HUB, deposits, buildings)
  still cluster/fan — they have no wires to detach from.

  ![Three power poles sitting exactly on their powerline, which bends through each pole](docs/images/changelog/v0.7.6-pole-on-wire.png)

## [0.7.5] — 2026-07-14

### Fixed

- **Map disc glyphs (and the pole's ✕ remove button) are now truly centered.** The
  round map buttons — power poles, node discs, and the pole's remove button — center
  their icon with `display: grid; place-items: center`, but kept the browser's
  native button box (`appearance: auto` + UA `padding: 1px 6px`), which shoved the
  grid-centered glyph to the right. Measured: the ✕ sat **4.5px** right of center in
  its 18px button (clearly lopsided), and the ⚡ bolt **1px** right in its disc —
  just enough that a pole looked slightly off the powerline it actually sits dead-on.
  Reset those buttons with `appearance: none; padding: 0`, so the glyph is now
  centered to the sub-pixel (verified: offset 0.0 on both axes). The pole disc was
  always geometrically on the line; now it *looks* it too.

  ![Two power poles on a straight powerline, bolts centered in their discs and the ✕ centered in each remove button](docs/images/changelog/v0.7.5-pole-centering.png)

## [0.7.4] — 2026-07-14

### Changed

- **The HUB finally looks like a HUB.** On the map (and in the build radial) the
  HUB reused the generic `factory` glyph — a smokestack, indistinguishable from an
  actual factory. Gave it a dedicated **domed-headquarters** icon: a rounded dome
  cap over a base structure with a central doorway and lit windows, so the player's
  central base reads at a glance as a command HQ rather than just another building.
  New `hub` pictograph in `UiIcon.tsx`; `factory` stays put as the generic building
  fallback.

![HUB icon reworked](docs/images/changelog/v0.7.4-hub-icon.png)

## [0.7.3] — 2026-07-14

### Fixed

- **Smooth walk, take three — hand the interpolation to the browser's compositor.**
  The `requestAnimationFrame` loops from 0.7.1/0.7.2 never reliably repainted the
  avatar (an instrumented walk showed its position frozen at ~2 distinct values over
  3+ seconds — the rAF simply wasn't driving the element in practice). Replaced the
  whole JS-prediction scheme with a CSS approach that can't misfire: the avatar and
  reach ring are positioned with a GPU-compositable `translate3d(...)` that the sim
  updates to the new position each 100ms tick, and a short `transition: transform
  0.12s linear` lets the compositor tween between those tick positions at the
  display's native frame rate. Position and scale are split across two elements
  (outer = transitioned translate, inner = `scale(1/zoom)`) so walking is smooth
  while zooming stays crisp. Tick positions are still sampled along the exact planned
  polyline (`playerPos` → `travelPos`), so the glide hugs the pathfinding route.
- **`PlayerMarker` now subscribes to just the player's x/y**, re-rendering once per
  tick during a walk (to update the transform target) and not at all while idle; the
  avatar SVG is memoized so it isn't rebuilt each tick. `MapView` keeps its ~1×/s
  countdown subscription, so the heavy map tree no longer reconciles 10×/s.

  ![The avatar mid-walk, centered on the dashed pathfinding route with its reach ring, walking away from the HUB](docs/images/changelog/v0.7.2-smooth-walk.png)

## [0.7.2] — 2026-07-14

### Fixed

- **The smooth walk from 0.7.1 was still stepping — now it genuinely glides at
  60 fps.** 0.7.1 added a `requestAnimationFrame` interpolation, but it lived
  inside `MapView`, which re-renders on **every** 100ms sim tick (it subscribed to
  the whole `player` object). Each of those 10×/s re-renders re-applied the
  avatar's `left`/`top` from the stepped, tick-quantized `playerPos(state)` —
  clobbering the smoothing — and the constant reconciliation starved the animation
  frames. Two changes fix it for real:
  - **The avatar + interaction ring moved into a memoized `PlayerMarker` component**
    that subscribes to nothing per-tick, so React never re-renders (and never
    repositions) it during a walk. Its own rAF reads the store directly each frame
    and interpolates along the exact planned polyline (`travelPos` over
    `travel.path` — the same line the dashed route draws), predicting forward by
    wall-clock time between the 10 fps sim updates. Each tick's fresh `travel`
    object re-anchors the prediction, so it never drifts or overshoots the goal.
  - **`MapView` no longer re-renders 10×/s during a walk.** It now subscribes to the
    whole-second travel countdown instead of the full `player` object, so it
    re-renders about once per second (for the "Traveling… Ns" readout) rather than
    on every tick — leaving the main thread free for the 60 fps avatar animation.

  ![The avatar mid-walk, centered on the dashed pathfinding route with its reach ring, walking away from the HUB](docs/images/changelog/v0.7.2-smooth-walk.png)

## [0.7.1] — 2026-07-14

### Changed

- **The player now glides smoothly along the planned route instead of shuffling
  in steps.** The simulation only advances a walk on the game tick (`TICK_MS` =
  100ms → 10 fps), so the avatar visibly jumped ~2 world units at a time and read
  as a stepped, gridlike hop. The avatar and its interaction ring are now
  interpolated every animation frame: a `requestAnimationFrame` loop in `MapView`
  reads the store's authoritative `travel` and predicts the position forward by
  wall-clock time between ticks (respecting the game-speed cheat), positioning the
  two elements imperatively so React re-renders aren't needed per frame. It follows
  the exact same smoothed/curved polyline (`travelPos` over `travel.path`) that the
  dashed route preview draws — so the walk now traces the pathfinding line rather
  than snapping across a grid. The store stays the single source of truth: each
  tick re-anchors the prediction, so it can never drift or overshoot the goal.

  ![The avatar mid-walk, centered on the dashed pathfinding route with its reach ring, walking away from the HUB](docs/images/changelog/v0.7.1-smooth-walk.png)

## [0.7.0] — 2026-07-14

### Changed

- **Foraging is now a timed manual action — you strip a patch leaf-by-leaf, not
  all at once.** Standing at a leaf patch and foraging arms it as an exclusive
  manual action (like mining/crafting): `simulateForage()` pulls one leaf into the
  inventory every `LEAF_FORAGE_SECONDS` (1.2s), so a full patch fills your grid
  "petit à petit" over ~18s rather than in a single click. It obeys the one-action-
  at-a-time rule (starting a mine/craft/scan/walk stops the forage, and vice
  versa), resumes across a reload, and runs the same offline — but offline harvest
  is still capped at what has actually grown, so it can't be exploited by waiting.
  The old instant `forageLeafPatch` action is replaced by a `toggleForage` toggle;
  in range, one click on the patch (or its button) starts or stops it.
- **Leaf patches show an outer regrow ring so a depleted patch reads at a glance.**
  Previously an empty patch differed from a full one only by a faint opacity shift,
  which was hard to see. Each patch now draws a progress ring around the leaf icon
  whose filled arc tracks how regrown it is (empty → full): a freshly-stripped
  patch is a near-empty ring that visibly fills back up as it regrows, and the
  patch you're actively foraging pulses. Leaf regrow moved from `simulate()` into
  `simulateForage()` so a single function owns all leaf-patch state.

![Leaf patches with regrow rings at varied fullness](docs/images/changelog/v0.7.0-forage-ring.png)

## [0.6.3] — 2026-07-14

### Changed

- **`npm run window` (Edge app-mode) is again the default launch; the Electron
  shell is opt-in.** Reverted `scripts/window.mjs` to the simple original launcher
  — dropped the dedicated Edge profile that had been added while chasing a browser
  taskbar icon (which is instead handled by the Electron shell). The native
  Electron path (`npm run electron:dev`, with its WSLg GPU fix and window/taskbar
  icon) stays available but is only launched when native/WSL2 is explicitly
  requested.

## [0.6.2] — 2026-07-14

### Fixed

- **Smooth scrolling in the Electron shell under WSLg.** Chromium's default GL
  backend fails to initialize against WSLg's virtual GPU (`/dev/dxg`, Mesa's
  `d3d12` driver) and silently falls back to CPU/software rendering, which makes
  scrolling and compositing janky. `electron/main.cjs` now detects WSL and routes
  GL through **ANGLE→GL** (`--use-gl=angle --use-angle=gl --ignore-gpu-blocklist`),
  which uses the vGPU and restores hardware-accelerated rendering — so
  `npm run electron:dev` no longer needs `--disable-gpu`. The switch is gated to
  WSL only, so native Linux, Windows, and macOS builds keep their default
  (optimal) GPU path.

## [0.6.1] — 2026-07-14

### Changed

- **The UI icons are now colored, and better centered.** The pictographic
  `UiIcon`s (crate, factory, smelter, gear, sensor, build, plug, bolt, trash,
  lock, bug, hourglass, pickaxe, wrench) are redrawn as two-tone shapes on the
  game's warm palette — a 3D orange crate, a yellow power bolt, a red ladybug for
  cheats, etc. — instead of flat single-color line marks. Control glyphs
  (chevrons, check, close, crosshair) stay subtle and inherit `currentColor`. The
  gear was also rebuilt from symmetric rotated teeth (the old hand-drawn one was
  lopsided), and the Options button now flex-centers its icon so it sits square
  with the label.

  ![The colored UiIcon set with a symmetric gear](docs/images/changelog/v0.6.1-colored-icons.png)

## [0.6.0] — 2026-07-14

### Added

- **A generated UI icon set — no more emoji anywhere.** Every emoji/dingbat used
  as a UI glyph (📦 drop pod, 🏭 factory, 🔥 smelter, 📡 sensor, 🏗️ build, 🔌
  powerline, ⚡ power, 🗑 destroy, 🔒 locked, 🐞 cheats, ⏳ saving, ⛏ mining, plus
  the ⚙ gear, ⌖ recenter, ▾▸ chevrons, ✓ checks and ✕ closes) is now a hand-drawn
  inline SVG in a new `src/components/UiIcon.tsx` (`<UiIcon name="…" />`). Icons use
  `currentColor`, so each inherits its surrounding text color. **Why it matters:**
  the shipped Electron build targets Windows *and* Linux, and the Linux runtime has
  no color-emoji font — so emoji rendered as tofu boxes (e.g. the Drop Pod). SVGs
  are font-independent and render identically on every platform. Added a project
  rule (CLAUDE.md) to always generate icons and never use emoji.

  ![The generated UiIcon set — 19 inline SVG icons replacing the former emoji](docs/images/changelog/v0.6.0-ui-icons.png)

### Changed

- **The Electron shell is now the icon-bearing, cross-platform desktop app.** The
  `BrowserWindow` picks a per-OS window icon (`favicon.ico` on Windows,
  `icon-512.png` on Linux/WSLg) and sets an explicit Windows AppUserModelID, so the
  game shows its own taskbar icon rather than a generic host icon. Added an
  `electron-builder` `build` config (`package.json`) with per-OS packaging icons
  for future Windows/Linux distributables. (Running the native shell needs the
  Linux GUI libs — `libnss3 libnspr4 libasound2t64` — installed once.)

## [0.5.3] — 2026-07-14

### Fixed

- **The app icon now shows on the Windows taskbar, not just in the header.** A
  Chromium `--app` window takes its taskbar icon from a raster `favicon.ico` and
  from having its own app identity — neither of which existed, so the taskbar
  showed the generic browser icon. Added a real multi-resolution
  `public/favicon.ico` (16/32/48/64/256 px, rendered from the icon SVG), a web
  app manifest (`public/manifest.webmanifest`) with 192/256/512 px icons, and
  the matching `<link>` tags. `npm run window` now also launches the browser
  with a **persistent, app-dedicated profile** (`--user-data-dir` under
  `%LOCALAPPDATA%\Satisfidlery`), so the window is treated as its own app and
  adopts the factory icon on the taskbar instead of grouping under Edge.

## [0.5.2] — 2026-07-14

### Added

- **A dedicated app icon — a factory & gear mark.** Satisfidlery now ships its
  own icon (`src/assets/icon.svg`): an orange factory silhouette with twin
  smokestacks and a gear, on the game's dark-industrial rounded-square
  background. It appears both in the **window header** brand (replacing the bare
  🏭 emoji) and, via a `<link rel="icon">` favicon (SVG + a 256×256 PNG
  fallback), in the **OS taskbar** when the game is launched with `npm run
  window`. The Electron shell also points its `BrowserWindow` `icon` at the same
  PNG for the future Steam build.

  ![The new Satisfidlery app icon shown as a tile and in the window header](docs/images/changelog/v0.5.2-app-icon.png)

## [0.5.1] — 2026-07-14

### Changed

- **Tearing down power infrastructure now shows the refund flying home — and
  never gets blocked.** Removing a powerline, a power pole, or a whole connection
  sends the refunded materials (cable, and a pole's cost) **flying to the
  inventory** as flying-item feedback, the same way collecting does. Previously,
  if the grid was full the teardown was silently refused to avoid destroying the
  refund; now it always goes through, and any overflow that doesn't fit a full
  grid is simply dropped.

  ![Tearing down a power grid on the map with refunded materials flying toward the inventory](docs/images/changelog/v0.5.1-refund-fly.png)

## [0.5.0] — 2026-07-14

### Added

- **Leaf foraging — a perishable, regenerating wild resource.** The Power Grid
  milestone now also opens up **leaf foraging**. Leaf patches are scattered across
  the map, **biome-weighted**: thick in forest and jungle, a light scattering on
  plains, a rare few on sand, and **none on mountains, snow, or water**. Walk to a
  patch and **Forage** it to harvest its grown leaves into your inventory; the
  patch then **regrows over time** back to full. Grown leaves **cap** at the
  patch's yield, so an un-foraged patch never hoards past full — waiting past ripe
  is wasted growth. A patch's disc fades on the map to show how much has regrown.

  Under the hood, patches are generated deterministically from the world seed
  (like ore deposits) and only *regrowing* patches are stored in the save, so the
  vast field costs nothing until you actually forage. Regrowth runs in `simulate()`
  (live and offline) without touching the inventory; foraging is the explicit
  action that moves leaves in.

  ![A forest dotted with scattered leaf patches, the player standing on one with a Forage button, leaves in the inventory](docs/images/changelog/v0.5.0-leaf-foraging.png)

### Notes

- Save format bumped to **v15** (adds `leafPatches`); older saves load
  forward-tolerantly with an empty (all-full) field.

## [0.4.3] — 2026-07-14

### Added

- **A cost confirmation popover when running a powerline.** Picking the two ends
  of a connection no longer builds it immediately — a popover now shows the exact
  cost (cable per segment plus any auto-placed poles) and a note of how many Power
  Poles the route will insert, with **Connect** / **Cancel**. Connect is disabled
  (with a reason) when a node has no free intake or you can't afford the run.

  ![The powerline cost popover: "Run a powerline · Smelter → Smelter · Cost 4 Cable, 4 Iron Rod · Includes 1 power pole auto-placed along the route" with Cancel / Connect](docs/images/changelog/v0.4.3-power-connection.png)

- **Remove a whole connection between two nodes.** In Powerline mode, picking two
  nodes that are *already* connected offers to **Disconnect** them instead: the
  entire chain of segments linking them is torn down in one go, along with any
  intermediate poles it orphans (a pole that also feeds elsewhere is kept), and
  everything is refunded. Single-segment removal (click a line) and pole teardown
  (a pole's ✕, which also drops its powerlines) remain.

### Fixed

- **Powerlines now meet each node dead-center.** Labeled map markers (buildings,
  the HUB, deposits) sized their box to include the name label below the disc, so
  a powerline — which attaches at the marker's world point — met the disc slightly
  below its center. The label is now positioned without growing the marker box, so
  the disc sits exactly on its world point and cables connect to disc centers.

## [0.4.2] — 2026-07-14

### Changed

- **Bigger inventory stacks — now matching Satisfactory's real sizes.** The old
  product cap of 50 filled the 24-slot grid far too fast. Stacks now follow the
  base game: **Iron/Copper Ingot 100**, **Iron Plate / Iron Rod / Cable 200**,
  and **Wire 500** (ores and leaves stay at 100; the Portable Miner still takes a
  whole slot). A full wire run that used to sprawl across ten slots now fits in
  one. Existing saves keep their stacks — partial slots simply have more headroom.

  ![Inventory holding product stacks in single slots at the raised caps: Iron Plate 200, Iron Rod 200, Wire 500, Cable 200, ingots at 100](docs/images/changelog/v0.4.2-bigger-stacks.png)

## [0.4.1] — 2026-07-14

### Changed

- **HUB milestones moved to the sidebar, under the inventory.** They used to sit
  in a right-hand rail inside the HUB tab; now the milestone chain (Field
  Fabrication → Assembly Line → …) renders directly beneath the inventory card,
  so a milestone's submission cost sits next to the stacks you pay it with. The
  HUB tab's main column is now just hand-crafting and the appearance bench, giving
  the craft grid the full width. The panel only shows while you're on the HUB tab.

  ![HUB milestones now render under the inventory in the sidebar, with the hand-crafting grid filling the main column](docs/images/changelog/v0.4.1-milestones-under-inventory.png)

## [0.4.0] — 2026-07-14

### Added

- **New milestone: the Power Grid.** After the Equipment Workshop, submit iron
  rods and plates at the HUB to unlock **Cable** and the power-grid infrastructure.
  Cable (`2 Wire → 1 Cable`) is craftable both in the **Constructor** and by hand
  at the **HUB** — and the HUB now also hand-crafts the earlier Assembly-Line parts
  (Iron Plate, Iron Rod, Wire), so you can make them without a Constructor.
- **Powerlines on the map.** A new **Powerline** map tool (alongside Build/Scan):
  enter it, click two grid nodes — buildings, the HUB, or power poles — to run a
  powerline between them. A run longer than the max segment length auto-inserts
  **Power Pole MK1**s along the shortest terrain-avoiding path (reusing the A\*
  router), so every hop stays within reach. Tap empty ground to drop a pole by
  hand; tap a line, or a pole's ✕, to remove it (with a refund). Buildings and the
  HUB host two powerlines each; poles host four, so the grid can daisy-chain.
  Powerlines and poles are drawn on the world map. This is structural for now —
  the electricity that flows through the grid comes in a later milestone. See
  `docs/images/changelog/v0.4.0-power-grid.png`.

### Notes

- Save format bumped to **v14** (adds `powerPoles`/`powerLines`); older saves load
  forward-tolerantly with empty power grids.

## [0.3.3] — 2026-07-14

### Changed

- **Hand-crafting in the HUB is now a compact tile grid.** The bench was a stack
  of full-width recipe cards that would clutter fast as recipes pile up; it's now
  a dense grid of small tiles (icon + name, a mini icon-based recipe line, a slim
  progress bar, and a status/cycle-time footer). Each tile is itself the control —
  click to start or stop crafting (or to travel to the HUB when you're away) — so
  five recipes now sit in the space the old layout gave two.

  ![The HUB hand-crafting bench as a compact grid of recipe tiles, with Iron Ingot mid-craft](docs/images/changelog/v0.3.3-hub-craft-grid.png)

## [0.3.2] — 2026-07-14

### Added

- **Ctrl+drop on the bin destroys every stack of an item.** Hold **Ctrl** while
  dropping a stack onto the Destroy bin and it clears _all_ slots holding that
  item at once, not just the one you dragged — a fast way to dump a resource you
  no longer want. Drop without Ctrl to trash only the single stack as before.

- **An inventory help "?" with the rules.** A **?** button in the inventory header
  raises a popover on hover listing how the inventory works — stacking and
  overflow, the capacity limit, dragging to load / rearrange, the Destroy bin and
  its Ctrl shortcut, and item hover descriptions.

  ![The inventory help popover listing the rules, with the Destroy bin showing its Ctrl shortcut](docs/images/changelog/v0.3.2-inventory-help-rules.png)

## [0.3.1] — 2026-07-14

### Added

- **Destroy items with the new bin.** A **🗑 Destroy** drop target now sits below
  the inventory grid — drag any stack onto it to throw it away and free the slot.
  The bin arms with a red warning tint while a stack hovers over it. There's no
  undo, so it's a deliberate drag.

- **Hover an item for a quick description.** Pointing at any filled slot raises a
  popover with the item's name, its category, a one-line blurb of what it is and
  what it's for, and the current stack against its max (e.g. `Stack · 50 / 50`).
  Every item ships with its own description, authored alongside the item in
  `content.ts`.

  ![The inventory grid with an Iron Ingot hover popover open and the Destroy bin below the slots](docs/images/changelog/v0.3.1-inventory-bin-popover.png)

## [0.3.0] — 2026-07-14

### Added

- **The inventory is now a Satisfactory-style grid.** Your holdings live in a
  fixed set of **24 stack slots** instead of an unbounded tally. Each slot holds a
  stack of one item up to that item's **maximum stack size** — ores and leaves
  stack to **100**, refined products (ingots, plates, rods, wire) to **50**, and a
  **Portable Miner** takes a whole slot on its own (equipment doesn't stack). Mine
  past a stack's cap and it spills into the next slot; the header shows how many
  slots are in use (e.g. `12 / 24`).

- **Capacity actually bites, and nothing is ever lost.** When the grid is full,
  collecting from a source or a building takes only what fits and leaves the rest
  sitting in that machine's buffer, and hand-crafting pauses rather than voiding
  its output. Refund paths with nowhere to spill back to (dismantling a machine or
  the HUB, pod salvage) simply won't complete unless the whole refund fits — so
  capacity pressure never destroys your materials.

- **Rearrange stacks by dragging.** Drag a stack onto another cell to move it,
  merge it into a matching stack (up to the cap, with any overflow staying put),
  or swap two stacks — on top of the existing drag-to-load / drop-buffers-to-
  collect flow. Old saves migrate automatically: your former totals are packed
  into the grid on first load.

  ![The new grid inventory: iron ore split across 100 / 100 / 46 stacks, ingots at 50 / 28, and a lone Portable Miner, with a 12 / 24 capacity badge](docs/images/changelog/v0.3.0-grid-inventory.png)

## [0.2.3] — 2026-07-14

### Added

- **"1×" load button on every building input.** Alongside _Load all_ and _Half_,
  each input slot now has a **1×** button that tops that slot up to exactly one
  cycle's requirement — just enough to craft a single item, with nothing left
  over. On a multi-input bench (the Equipment Workshop) each slot's 1× fills its
  own share, so one click per slot loads precisely one item's worth. The button
  is disabled once the buffer already holds enough for one, and (like the others)
  is capped by what's in your inventory and by being in reach of the building.

  ![The Constructor's input slot showing the new 1× button next to Load all and Half](docs/images/changelog/v0.2.3-load-one.png)

## [0.2.2] — 2026-07-14

### Changed

- **Portable Miners now have a 500-ore buffer cap.** A deployed miner fills its
  node's buffer hands-free (live and offline), but stops once the buffer reaches
  **500** — its onboard hopper is full. This bounds passive/offline hoarding: the
  miner idles until you walk over and collect, which frees room and resumes it
  automatically. Hand-mining is player-attended and stays uncapped.

- **The map reacts when a miner fills up.** A full miner's node swaps its calm
  green ring for an insistent amber pulse and pops a bouncing **MAX** bubble
  above it, so you can spot at a glance which deposits need emptying. The Sources
  tab shows the same "miner full" status in amber.

  ![A full Portable Miner pops a MAX bubble on the map; a still-mining node stays green](docs/images/changelog/v0.2.2-miner-full.png)

## [0.2.1] — 2026-07-14

### Improved

- **Map markers declutter when zoomed out.** Overlapping markers (deposits,
  buildings, the HUB) now merge into a single count bubble instead of piling up
  into a mess of discs and labels. Click the bubble to fan the members out and
  pick any one — this also separates markers that share a coordinate, like the
  buildings granted at the HUB. Labels hide when zoomed far out and return on
  hover / when zoomed in.

  ![Overlapping markers collapse into one count bubble when zoomed out](docs/images/changelog/v0.2.1-cluster-collapsed.png)

  _Clicking the bubble fans every member out, each individually selectable:_

  ![Clicking the bubble fans the members out, each labelled and selectable](docs/images/changelog/v0.2.1-cluster-expanded.png)

## [0.2.0] — 2026-07-14

### Added

- **Player avatar customization.** Customize your pioneer at the HUB's new
  **Appearance** bench — head shape, skin, eyes, headgear, and suit color, each
  with a live preview. Your choices show up as the player marker on the world
  map. New pioneers start as a neutral grey silhouette.

  ![The HUB Appearance bench: live avatar preview beside per-part chip pickers](docs/images/changelog/v0.2.0-avatar-customizer.png)

  _The customized avatar represents the player on the world map:_

  ![The customized avatar standing on the world map](docs/images/changelog/v0.2.0-avatar-on-map.png)
