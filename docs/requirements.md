# Fortressy — Requirements (MVP + Fast-Follow)

Date: 2026-03-23  
Source: `research.md` + repo review  
Target engines: Godot 4.6.x  
Primary platform: **Web (HTML5/WebGL)**  
Fast-follow: **Windows desktop**

This document converts the design/backlog in `research.md` into implementable **functional requirements (FR)** and **non-functional requirements (NFR)**, with explicit MVP scope for a first playable release.

## 1) Scope and release milestones

### 1.1 MVP (Release 0.1) — Single-player Web
MVP goal: a complete, replayable core loop that demonstrates the Rampart-style phase cycle: **setup → battle → build → validate → cannon placement → repeat**, with scoring, a basic UI, and at least one map.

MVP explicitly includes:
- Single-player only
- One map (minimum)
- One playable “run” loop with round progression
- Basic enemy fleet (minimum set)
- Core wall placement + territory validation

MVP explicitly excludes (deferred beyond 0.1):
- Local multiplayer
- Rogue-lite meta progression / meta shop / drafting (Epic 7)
- Advanced ship types beyond the minimum roster
- Extensive content (multiple maps, campaigns)

### 1.2 Fast-follow (Release 0.2) — Windows desktop
Fast-follow goal: parity with MVP gameplay plus desktop-specific UX improvements.

## 2) Definitions and shared assumptions

### 2.1 Core terms
- **Round**: one full cycle of phases that ends after Build Phase validation and Cannon Placement.
- **Phase**: a gameplay mode with exclusive inputs and rules (Setup/Battle/Build/Cannon/etc).
- **Cell**: a grid tile coordinate represented as `Vector2i`.
- **Territory**: the set of cells enclosed by a sealed wall loop around at least one castle.

### 2.2 MVP design decisions
These are requirements because they remove ambiguity (treat as locked for MVP implementation):
- Enclosure adjacency rules are **cardinal only** (N/E/S/W). Diagonals do not seal territory.
- Cannons count as **solid** for flood-fill blocking purposes.
- The game is designed to feel responsive at **60 FPS**.
- **MVP grid size: 42×30 cells.**

### 2.3 Data contracts (MVP)
These contracts define the minimum expected inputs/outputs for the MVP’s core **pure logic** modules (NFR-160.1). They are intentionally engine-agnostic so they can be unit-tested in isolation.

#### 2.3.1 `FloodFillValidator` (FR-230)
**Purpose**: Determine whether a castle seed can reach boundary/water through passable cells, using cardinal adjacency.

**Inputs**
- `bounds_size: Vector2i` — grid width/height in cells (MVP: 42×30).
- `solid: Dictionary[Vector2i, bool]` — impassable cells (walls + player cannons). Presence of a key means true.
- `water: Dictionary[Vector2i, bool]` — water cells. Presence of a key means true.
- `seed: Vector2i` — starting cell (a castle cell).

**Outputs**
- `success: bool` — true iff the region is enclosed (did NOT reach boundary or water).
- `visited: Array[Vector2i]` — all visited passable cells (for optional territory painting).
- `failure_reason: String` (or enum) — one of:
  - `REACHED_BOUNDARY`
  - `REACHED_WATER`
  - `INVALID_SEED`

**Rules**
- Expansion is cardinal only.
- Cells in `solid` are not visited.
- If expansion touches any out-of-bounds neighbor, treat as boundary reached → fail.
- If a visited cell is water (or expansion reaches water), fail.

#### 2.3.2 `PieceModel` (FR-240)
**Purpose**: Represent a build-piece shape as a set of cell offsets and apply 90° rotations deterministically.

**Representation**
- `cells: Array[Vector2i]` — offsets relative to an anchor at (0,0).
- `rotation: int` — 0..3 representing clockwise 90° steps.

**Operations**
- `rotated_cells(rotation: int) -> Array[Vector2i]`
- `cells_at(anchor: Vector2i, rotation: int) -> Array[Vector2i]`

**Rotation rule**
- Clockwise 90° about origin: `(x, y) -> (y, -x)`.

#### 2.3.3 `PlacementValidator` (FR-250)
**Purpose**: Validate whether a multi-cell footprint can be placed.

**Inputs**
- `cells: Array[Vector2i]` — target footprint.
- Query callbacks (or an adapter interface) that answer in constant time:
  - `is_in_bounds(cell) -> bool`
  - `is_buildable(cell) -> bool`
  - `is_water(cell) -> bool`
  - `is_hazard(cell) -> bool`
  - `has_wall(cell) -> bool`
  - `has_cannon(cell) -> bool`
  - `has_grunt(cell) -> bool`

**Outputs**
- `valid: bool`
- `first_invalid_cell: Vector2i` (optional)
- `reason: String` (optional; e.g. `WATER`, `HAZARD`, `OCCUPIED`, `OUT_OF_BOUNDS`)

**Rule**
- Any single invalid cell fails the whole placement.

#### 2.3.4 `CannonQueue` (FR-310)
**Purpose**: Provide deterministic sequential firing order with a per-cannon “projectile in flight” lock.

**Inputs / state**
- `cannon_ids: Array[String]` (or int IDs) in **placement order**.
- `in_flight: Dictionary[id, bool]`.
- `cursor_index: int` — index of last-selected cannon (persisted across shots).

**Operations**
- `next_fireable() -> String` — returns next fireable `id`, or empty string (`""`) if none.
- `mark_fired(id) -> void` — sets `in_flight[id] = true`.
- `mark_resolved(id) -> void` — sets `in_flight[id] = false`.

**Rule**
- Selection always scans forward from `cursor_index + 1` wrapping around, skipping cannons currently `in_flight`.

## 3) Functional Requirements (FR)

### FR-000: Boot + navigation
- **FR-000.1** The project SHALL define a `main_scene` that launches via the editor “Play” button.
- **FR-000.2** The game SHALL present a main menu with: Start, Options, Quit (Quit may be hidden/disabled on Web).
- **FR-000.3** Starting a game SHALL transition into a gameplay scene without errors.

**Acceptance criteria**
- From a clean project import, Play → Main Menu appears.
- Start → gameplay scene loads on Web export.

---

### FR-010: Camera (MVP)
- **FR-010.1** Gameplay SHALL support **edge-pan** camera movement using mouse/pointer position near the viewport edges.
- **FR-010.2** Gameplay camera SHALL have **no zoom** in MVP.
- **FR-010.3** Camera movement SHALL be **clamped to the map rectangle** (no scrolling past map bounds).

**Acceptance criteria**
- Moving the pointer to the viewport edge pans the camera.
- The camera view cannot scroll beyond the playable map area.

---

### FR-100: Phase-based finite state machine (FSM)
- **FR-100.1** The game SHALL implement a phase-based FSM with isolated input handling per phase.
- **FR-100.2** MVP phases SHALL include at minimum: `Setup`, `Battle`, `Build`, `Validate`, `CannonPlacement`.
- **FR-100.3** The FSM SHALL transition phases based on timers and/or completion conditions as defined below.
- **FR-100.4** During a phase, inputs not allowed for that phase SHALL have no gameplay effect.

**Acceptance criteria**
- A debug overlay (or logs) shows the current phase and transitions.
- Player cannot build during Battle and cannot fire during Build.

---

### FR-110: Timers and phase durations
- **FR-110.1** Battle Phase SHALL have a visible countdown timer.
- **FR-110.2** Build Phase SHALL have a visible countdown timer.
- **FR-110.3** When the Battle timer expires, the game SHALL stop accepting Battle input and proceed to Build after resolving in-flight projectiles (FR-310.4).
- **FR-110.4** When the Build timer expires, the game SHALL run territory validation and proceed based on success/failure.
- **FR-110.4a** When the Build timer expires, any currently active (unplaced) wall piece SHALL be **rejected** (not auto-placed), then territory validation runs immediately.
- **FR-110.5** MVP default phase durations SHALL be:
  - Battle: **10s**
  - Build: **10s**
  - Cannon Placement: **10s**

**Acceptance criteria**
- Timer reaches zero → phase transition occurs deterministically.

---

### FR-200: Grid and map layers
- **FR-200.1** The gameplay field SHALL be represented as a grid addressable by `Vector2i`.
- **FR-200.2** The game SHALL use TileMapLayer (Godot 4.6) or an equivalent layered grid abstraction.
- **FR-200.3** MVP MUST include separate logical layers for: Water, Ground, Structures (walls/cannons), Hazards.
- **FR-200.4** Tiles SHALL expose metadata (custom data) or an equivalent data model to allow queries:
  - is_water
  - is_buildable
  - is_solid
  - is_hazard

- **FR-200.5** MVP maps SHALL be **data-authored** (custom `MapData` resource or equivalent) and loaded by the gameplay scene at runtime.
- **FR-200.6** MVP SHALL ship with **at least 1** `MapData` map, and the map pipeline SHALL support adding additional maps without code changes.

**Acceptance criteria**
- For any cell, code can query buildability in constant time.

---

### FR-210: Castles
- **FR-210.1** The map SHALL contain one or more castles placed at fixed cells.
- **FR-210.2** In Setup Phase, the player SHALL select a starting (“Home”) castle.
- **FR-210.3** Selecting Home castle SHALL initialize a perimeter wall around the castle footprint.
- **FR-210.4** MVP castle footprint SHALL be **2×2 cells**.

**Acceptance criteria**
- On selection, walls appear around the chosen castle and initial territory is claimed.

---

### FR-220: Territory claiming and painting
- **FR-220.1** After a successful enclosure validation, the game SHALL mark enclosed ground cells as “claimed territory”.
- **FR-220.2** Claimed territory SHALL be visually distinct from unclaimed ground.

**Acceptance criteria**
- After validation success, enclosed region changes appearance.

---

### FR-230: Territory validation (flood fill)
- **FR-230.1** At Build timer expiry, the game SHALL determine whether at least one castle is enclosed by a sealed wall loop.
- **FR-230.2** Validation SHALL use a flood fill (BFS/DFS) conceptually equivalent to `research.md`:
  - Expand cardinally from a castle cell through passable cells.
  - Stop expansion on solid cells (walls and cannons).
  - Fail immediately if expansion reaches map boundary or water.
- **FR-230.3** Validation SHALL be performed for all surviving castles (or all candidate castles) necessary to determine survival.

**Acceptance criteria**
- A set of test maps (unit tests) assert expected enclosed/not enclosed results.

---

### FR-240: Build Phase — wall pieces (tetromino/polyomino)
- **FR-240.1** During Build Phase, the player SHALL be provided a sequence of wall pieces (shapes) composed of multiple cells.
- **FR-240.2** The active piece SHALL follow the cursor and snap to the grid.
- **FR-240.3** The player SHALL be able to rotate the active piece in 90° increments.
- **FR-240.4** Placement SHALL be validated each frame; invalid placements SHALL be visibly indicated.
- **FR-240.5** When placed, the piece SHALL become permanent (no undo) for the remainder of the Build Phase.
- **FR-240.6** Wall pieces SHALL NOT overlap existing walls; each placed cell MUST be empty buildable ground.

**Acceptance criteria**
- The system refuses placements on non-buildable cells; valid placements become walls.

---

### FR-250: Build validation rules (collision constraints)
- **FR-250.1** Wall pieces SHALL NOT be placeable on:
  - water
  - existing walls
  - hazards (e.g., craters)
  - cannons
  - enemy grunts
- **FR-250.2** Validation SHALL operate on the entire shape footprint (all cells must be valid).

**Acceptance criteria**
- Any single invalid cell causes the whole placement to be rejected.

---

### FR-300: Battle Phase — crosshair targeting
- **FR-300.1** During Battle Phase, a crosshair SHALL follow the player’s pointer input.
- **FR-300.2** Crosshair SHALL be constrained within the playable viewport.

**Acceptance criteria**
- Crosshair never leaves the visible play area.

---

### FR-310: Cannons and sequential cannon queue
- **FR-310.1** Cannons SHALL exist as placeable structures occupying one or more cells.
- **FR-310.1a** MVP cannon footprint SHALL be **2×2 cells**.
- **FR-310.2** Cannons SHALL be fired sequentially in placement order.
- **FR-310.3** Each cannon SHALL allow at most one projectile in flight at a time.
- **FR-310.4** When Battle timer ends, the game SHALL wait for in-flight projectiles to resolve (impact + damage) before entering Build.
- **FR-310.5** When the Battle timer ends, ships SHALL NOT fire new shots, and the player SHALL NOT be able to fire.
- **FR-310.6** “Projectile drain” has **no timeout**; phase transition waits until all projectiles resolve.
- **FR-310.7** There is **no special UI message/indicator** for projectile drain in MVP (beyond the phase/input lock).

**Notes**
- Cannons in this section refer to **player-placed cannons**.
- Enemies do **not** place cannons; they attack only via ship-fired projectiles.

**Acceptance criteria**
- With 2+ cannons, repeated Fire cycles through cannons in order.

---

### FR-320: Cannon projectiles and impacts
- **FR-320.1** Projectiles SHALL travel from cannon to target with a flight time proportional to distance.
- **FR-320.2** Projectiles SHALL visually simulate an arc (pseudo-3D) as described in `research.md`.
- **FR-320.3** On impact, the game SHALL spawn an explosion/hit event that can apply damage in an area.
- **FR-320.4** Player-fired projectiles SHALL affect a **1×1** area (the impact cell).
- **FR-320.5** Enemy-fired projectiles SHALL affect an area centered on the impact point:
  - **2×2** for non-dark ships
  - **3×3** for dark ships

**Acceptance criteria**
- A projectile fired at a farther target lands later than a nearer one.

---

### FR-330: Damage rules + friendly fire
- **FR-330.1** The game SHALL attribute damage source as Player or AI.
- **FR-330.2** If AI damage destroys a wall cell, the game SHALL create a crater hazard (FR-410).
- **FR-330.3** If Player damage destroys a wall cell, the wall cell SHALL revert to buildable ground and SHALL NOT create a crater.
- **FR-330.4** Player-fired cannonballs SHALL NOT create craters on any impact type in MVP; crater creation is reserved for enemy incendiary impacts only.
- **FR-330.5** Enemy incendiary projectiles SHALL behave like normal enemy projectiles for **wall destruction**, in addition to crater creation rules (FR-410).

**Acceptance criteria**
- Same wall hit by AI vs player produces different aftermath (crater vs empty).

---

### FR-400: Enemies — MVP roster and spawning
- **FR-400.1** The game SHALL spawn enemy ships during Battle Phase.
- **FR-400.2** MVP SHALL include the following ship types:
  - Regular Ship
  - Lander
  - Red Ship
- **FR-400.2a** MVP SHALL also include a **Dark** variant of each ship:
  - Dark Ship
  - Dark Lander
  - Dark Red Ship
- **FR-400.3** The game SHALL cap active ships to a configurable maximum (default 16).

- **FR-400.4** Spawn timing: each round’s wave SHALL spawn **all ships immediately at the start of Battle** (subject to the max-ship cap).
- **FR-400.5** Spawn positions SHALL be randomly selected from **boundary water cells**.
- **FR-400.6** Spawn positions SHALL be **unique per wave** (no stacking multiple ships on the same spawn cell).
- **FR-400.7** Each spawned ship SHALL receive an initial heading based on which boundary it spawned on, with slight randomness.

- **FR-400.8** MVP wave composition SHALL be driven by a **points budget** per round.
  - Costs (default): Regular **2**, Lander **5**, Red **8**
  - Budget (default): `10 + 3*(round-1)`, cap **40**
  - Max ships (default): **16**
  - Weighted selection (default): Regular **70**, Lander **20**, Red **10**
  - Early waves MAY be low ship count (no minimum count guarantee)

**Acceptance criteria**
- Ships appear, move, and can be destroyed by cannon fire.

---

### FR-430: Enemy projectiles (MVP)
- **FR-430.1** All enemy projectiles that hit walls/terrain SHALL destroy walls within their damage footprint (FR-320.5).
- **FR-430.2** Incendiary shots SHALL be fired only by **Red Ships** and **Dark Red Ships**.
- **FR-430.3** Enemy target selection SHALL choose a **random wall tile** from **all** wall tiles on the map (uniform selection).
- **FR-430.4** Enemy firing SHALL have **no line-of-sight constraints** and **no range constraints** in MVP.

---

### FR-450: Grunts (MVP)
- **FR-450.1** MVP SHALL include **enemy grunts** as ground units.
- **FR-450.2** During Build Phase, any cell occupied by a grunt SHALL be treated as **not buildable** (build placement is rejected per FR-250).
- **FR-450.3** On successful territory enclosure/validation, grunts within the enclosed territory SHALL be **removed**.

**Acceptance criteria**
- Attempting to place a wall piece overlapping a grunt fails.
- After a successful enclosure, grunts inside the territory are removed.

---

### FR-410: Craters (hazards)
- **FR-410.1** Craters SHALL be unbuildable hazard cells.
- **FR-410.2** Craters created by AI impacts SHALL persist for a defined number of rounds (default 3), then revert to buildable ground.
- **FR-410.3** Craters SHALL be **1×1 cell** at the impact cell.
- **FR-410.4** Craters MAY occur on claimed territory.
- **FR-410.5** If an AI incendiary impact hits a wall tile, the crater occupies that wall tile cell.
- **FR-410.6** Craters SHALL be created **only** by **enemy incendiary** impacts.

**Acceptance criteria**
- A crater blocks building; after N rounds it disappears.

---

### FR-500: Cannon placement phase
- **FR-500.1** After successful Build validation, the game SHALL enter a Cannon Placement phase.
- **FR-500.2** The player SHALL receive cannon rewards as:
  - +2 for enclosing the Home castle
  - +1 for each additional enclosed castle
- **FR-500.3** Cannons SHALL be placeable only within claimed territory.

**Acceptance criteria**
- Attempting to place a cannon outside claimed territory is rejected.

---

### FR-600: Scoring and HUD
- **FR-600.1** The game SHALL track and display score.
- **FR-600.2** MVP scoring MUST include at least:
  - ship destroyed (+30)
  - territory bonus (enclosed tiles × 7)
  - home castle bonus (+150)
  - secondary castle bonus (+50 each) if multiple castles exist in MVP
- **FR-600.3** The HUD SHALL display current phase and timers.

**Acceptance criteria**
- Score changes immediately upon events and is visible during play.

---

### FR-700: Failure and game over (MVP)
- **FR-700.1** If Build validation fails to enclose at least one castle, the player SHALL lose (MVP) and be shown a results/game-over screen.
- **FR-700.2** MVP failure ends the run immediately (no arcade credits).
- **FR-700.3** Game Over screen SHALL provide: **Restart Run** and **Main Menu**.

- **FR-710: Determinism and seed**
  - **FR-710.1** Each run SHALL have a deterministic RNG seed.
  - **FR-710.2** Restart Run SHALL reuse the **same seed** as the failed run.
  - **FR-710.3** The Game Over screen SHALL display the run seed.
  - **FR-710.4** MVP SHALL NOT provide manual seed entry.

**Acceptance criteria**
- Intentional failure reliably transitions to game over.

---

### FR-720: End-of-battle leftover ships behavior (option)
MVP supports a settings flag that controls what happens to ships when Battle ends (timer reaches zero):
- **FR-720.1** The game SHALL support a configuration option:
  - **Persist ships**: ships remain on screen but are **frozen during Build/CannonPlacement**
  - **Despawn ships**: ships are removed at end of Battle
- **FR-720.2** The default behavior SHALL be **Persist ships**.
- **FR-720.3** Regardless of the ship behavior, Build SHALL NOT begin until the projectile drain completes (FR-310.4–FR-310.7).

**Acceptance criteria**
- Toggling the setting changes leftover-ship behavior without changing other phase rules.

---

### FR-800: Rogue-lite systems (Deferred)
The following are NOT in MVP but are captured as fast-follow/backlog requirements from Epic 7 in `research.md`.
- **FR-800.1** Meta progression save system (`MetaSaveManager`).
- **FR-800.2** Meta-shop phase (`StateMetaShop`).
- **FR-800.3** Draft phase (`StateDraft`).
- **FR-800.4** Lieutenant enemies + shape-aware item spawning + multi-cell acquisition.

## 4) Non-Functional Requirements (NFR)

### NFR-100: Platform compatibility
- **NFR-100.1** MVP SHALL run in modern desktop browsers via Godot Web export.
- **NFR-100.2** The build SHALL not rely on platform features unavailable on the Web target.
- **NFR-100.3** Windows desktop export SHALL be supported as a fast-follow.

### NFR-110: Performance
- **NFR-110.1** The game SHOULD maintain 60 FPS on a reasonable Web target.
- **NFR-110.2** Flood fill validation SHOULD complete within a single frame for the MVP grid size.
- **NFR-110.3** The game SHALL enforce caps (ships, projectiles, particles) to avoid runaway performance degradation.

### NFR-120: Determinism and reproducibility
- **NFR-120.1** When running with a fixed RNG seed, ship spawns and build-piece generation SHOULD be reproducible.
- **NFR-120.2** Phase transitions SHALL be deterministic given the same initial conditions.

### NFR-130: Test coverage (GUT)
- **NFR-130.1** The repo SHALL include automated tests for at minimum:
  - flood fill enclosure success/failure
  - build placement validation
  - cannon queue firing order
  - crater lifetime
- **NFR-130.2** Tests SHALL be runnable headlessly (where feasible) or from the editor via GUT.

### NFR-140: Usability and controls
- **NFR-140.1** Web MVP SHALL support mouse input end-to-end.
- **NFR-140.2** The game SHOULD provide clear visual feedback for invalid placement and phase transitions.

### NFR-150: Accessibility (baseline)
- **NFR-150.1** Claimed vs unclaimed territory visuals SHOULD not rely solely on hue (include pattern/brightness differences).

### NFR-160: Maintainability
- **NFR-160.1** Core game systems (FSM, grid, validation, scoring) SHALL be separated into testable modules/scripts.
- **NFR-160.2** Game constants (timers, scoring, caps) SHOULD be data-driven (resource or config file).

### NFR-170: Legal and content provenance
- **NFR-170.1** All art/audio used in releases SHALL be original or appropriately licensed.
- **NFR-170.2** The project SHALL avoid using protected trademarks/branding from the original arcade game.

## 6) Traceability to `research.md`
- FSM / phases: Epic 1
- Grid/layers + flood fill: Epic 2
- Combat / cannon queue / trajectory / crater / friendly fire: Epic 3
- Build phase tetromino system + validation + cannon placement: Epic 4
- AI fleet manager and ships/grunts: Epic 5
- Scoring/UI/audio: Epic 6
- Roguelite meta progression: Epic 7 (deferred)
