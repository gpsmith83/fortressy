# Fortressy — MVP Plan (Web-first)

Date: 2026-03-23  
Primary references: `docs/requirements.md`, `research.md`

Working doc: `docs/traceability.md` (update alongside code/tests)

This plan breaks the MVP into incremental, testable milestones. Each milestone lists:
- **Deliverable**: what becomes playable/verified
- **Requirements**: which FR/NFR items it satisfies
- **Work items**: concrete engineering tasks
- **Verification**: how we’ll confirm it’s done

## MVP constraints and guiding rules

- Target: **Single-player Web** first. Windows is fast-follow.
- Prefer **data-driven config** for timers, scoring, caps (NFR-160.2).
- Keep systems **testable**: pure logic modules whenever possible (NFR-160.1).
- Add **GUT tests** alongside the first implementation of each system (NFR-130).

Test conventions:
- Unit tests live in `test/unit/` and target pure modules/contracts (no scene setup where feasible).
- Integration tests live in `test/integration/` and validate scene/state transitions end-to-end.

## Definition of Done (DoD) / quality gates (applies to every milestone)

Each milestone is considered “done” only when:

- **Playable / runnable**: The deliverable can be exercised from the Godot editor (or exported build, if milestone includes export work).
- **No new errors**: No new errors in the Godot output while performing the milestone’s manual verification.
- **Automated verification**: All newly-added tests for the milestone pass, and existing tests remain green.
- **Traceability updated**: `docs/traceability.md` is updated to reflect new/changed features, tests, and requirement links.
- **Web-first sanity**: If the milestone adds input/rendering/game-loop behavior, it should remain compatible with Web export constraints (avoid platform-only APIs; avoid unbounded allocations per frame).

Recommended (non-blocking unless the milestone explicitly targets it):

- **Performance sanity**: No obvious per-frame spikes in the Godot profiler when running the milestone’s scenario.
- **Determinism hooks**: Any RNG introduced is controllable/seedable (aligns with NFR-120.1).

---

## Milestone 0 — Repository “playable boot” skeleton

Depends on: none

### Deliverable
Press Play in Godot and reach a menu. Start loads a placeholder gameplay scene.

### Requirements
- FR-000.1, FR-000.2, FR-000.3
- NFR-100.1 (basic Web-compat assumptions begin here)

### Work items
- Create `Main.tscn` (menu) and `Game.tscn` (placeholder).
- Set `main_scene` in `project.godot`.
- Add minimal input actions (fire, rotate, place) in Project Settings.

### Verification
- Manual: Editor Play → Menu → Start → Game scene.
- Smoke test: no errors in output.

---

## Milestone 1 — FSM scaffold + timers + HUD

Depends on: Milestone 0 (boot/menu -> game scene wiring)

### Deliverable
A phase label and timers tick down and transition through phases (even if gameplay is stubbed).

### Requirements
- FR-100.1–FR-100.4
- FR-110.1–FR-110.4
- FR-600.3
- NFR-120.2

### Work items
- Implement `GameManager` with child state nodes: `StateSetup`, `StateBattle`, `StateBuild`, `StateValidate`, `StateCannonPlacement`.
- Centralize phase timer handling.
- Create HUD (CanvasLayer) showing phase + time.
- Add a debug log/event when phase changes.

### Verification
- Manual: watch timer transitions.
- Tests:
  - `test/unit/test_fsm_phase_progression.gd`
    - Asserts the phase ordering for at least one full cycle: Setup → Battle → Build → Validate → CannonPlacement → Battle.
    - Asserts `phase_changed` events are emitted (FR-100.5 / FR-900.3 `phase_changed`).

---

## Milestone 2 — Grid foundation + map + castle selection

Depends on: Milestone 1 (GameManager + StateSetup + phase transitions)

### Deliverable
A single map loads with layers; player can select Home castle and see an initial perimeter wall.

### Requirements
- FR-200.1–FR-200.4
- FR-210.1–FR-210.3
- FR-220.1–FR-220.2 (initial claim visual can be basic)
- NFR-160.1

### Work items
- Create a minimal map scene with TileMapLayer nodes (Water/Ground/Structures/Hazards).
- Add a small tileset placeholder (can be programmer art) with required metadata.
- Implement `GridManager`:
  - world↔cell conversion helpers
  - `is_buildable(cell)`, `is_solid(cell)`, `is_water(cell)`
- Lock MVP map dimensions to **42×30** cells (drives camera clamp bounds + validation perf assumptions).
- Implement `Castle` nodes and selection logic in `StateSetup`.
- Implement initial perimeter placement around castle footprint.

### Verification
- Manual: click castle → perimeter appears.
- Tests:
  - `test/unit/test_perimeter_generation.gd`
    - Given a 2×2 castle footprint at a known cell, asserts the returned perimeter cell set matches expected.
  - `test/unit/test_grid_manager_bounds.gd`
    - Asserts the MVP grid bounds are treated as 42×30 for in-bounds checks.

---

## Milestone 3 — Build system: piece generation, rotation, placement validation

Depends on: Milestone 2 (GridManager + map layers + bounds contract)

### Deliverable
During Build phase, a piece follows cursor, rotates, shows valid/invalid, and places walls.

### Requirements
- FR-240.1–FR-240.5
- FR-250.1–FR-250.2
- NFR-140.2
- NFR-120.1 (seedable RNG for pieces)

### Work items
- Define a small MVP piece set (e.g., 5–7 shapes) and weights.
- Implement `PieceModel` (pure data): cells + rotation.
- Implement `BuildController` (state-owned):
  - projection to grid
  - validation using GridManager
  - placement into Structures layer
- Add visual ghost piece (green/red tint).
- Add placement SFX hooks (optional for MVP).

### Verification
- Manual: place multiple pieces; invalid placements blocked.
- Tests:
  - `test/unit/test_piece_model_rotation.gd`
    - Asserts `PieceModel` rotation matches the contract (90° clockwise) for a known shape.
  - `test/unit/test_placement_validator_rules.gd`
    - Asserts placement validation fails if any footprint cell overlaps water/wall/hazard/cannon/grunt.
    - Asserts one invalid cell invalidates the whole placement.

---

## Milestone 4 — Flood fill validation + territory claiming

Depends on: Milestone 2 (grid + solids/water representation), Milestone 3 (walls/structures placement)

### Deliverable
At build end, game validates enclosure via flood fill and paints claimed territory.

### Requirements
- FR-230.1–FR-230.3
- FR-220.1–FR-220.2
- NFR-110.2
- NFR-130.1 (flood fill tests)

### Work items
- Implement `FloodFillValidator` as a pure module:
  - inputs: grid bounds, solid mask, water mask, seed cell
  - outputs: `{success: bool, visited: Array[Vector2i]}`
- Integrate into `StateValidate`.
- Paint claimed territory in Ground layer.
- Define “success” rule: at least one castle enclosed.
- Territory is recomputed each round from current walls/cannons; broken enclosures during Battle can cause validation failure.

### Verification
- Tests:
  - `test/unit/test_flood_fill_validator.gd`
    - Uses small synthetic grids to assert:
      - enclosed succeeds
      - open-to-boundary fails (`REACHED_BOUNDARY`)
  - leak-to-water fails (`REACHED_WATER`) when the fill reaches any passable cell adjacent (cardinally) to a water cell
- Manual: build a loop → claim occurs.

---

## Milestone 5 — Cannon placement phase + cannon inventory

Depends on: Milestone 4 (claimed territory output and/or representation), Milestone 1 (FSM includes CannonPlacement phase)

### Deliverable
After successful validation, player places awarded cannons within claimed territory.

### Requirements
- FR-500.1–FR-500.3
- FR-310.1

### Work items
- Represent cannon as a grid-occupying entity (MVP **2×2** footprint).
- Implement cannon award calculation.
- Implement placement UI similar to Build placement, restricted to claimed territory.
- Register cannons in a placement-order list.

### Verification
- Manual: cannons can’t be placed outside claimed.
- Tests:
  - `test/unit/test_cannon_awards.gd`
    - Asserts +2 for home, +1 per additional enclosed castle.
  - `test/unit/test_cannon_placement_restrictions.gd`
    - Asserts a 2×2 cannon footprint cannot be placed if any cell is outside claimed territory.

---

## Milestone 6 — Battle: crosshair + sequential cannon firing + projectile impacts

Depends on: Milestone 5 (cannons placed + ordered list), Milestone 1 (FSM includes Battle phase)

### Deliverable
During Battle, crosshair aims, Fire triggers sequential cannons, projectiles arc and explode.

### Requirements
- FR-300.1–FR-300.2
- FR-310.2–FR-310.4
- FR-320.1–FR-320.3
- NFR-110.1

### Work items
- Crosshair controller.
- `CannonQueue` module:
  - maintains ordered list
  - selects next available cannon (projectile-in-air rule)
- Projectile scene:
  - time-of-flight proportional to distance
  - arc visual offset
  - impact event + AoE
- Ensure end-of-battle waits for in-flight impacts before Build.

### Verification
- Manual: rapid clicks rotate through cannons; far shots take longer.
- Tests:
  - `test/unit/test_cannon_queue_order.gd`
    - Asserts sequential order respects placement order and skips cannons with `in_flight=true`.
  - `test/unit/test_projectile_time_of_flight.gd`
    - Asserts time-of-flight is monotonic with distance.

---

## Milestone 7 — Enemies: MVP roster + spawner + damage and scoring

Depends on: Milestone 6 (projectile impacts/damage events), Milestone 1 (Battle phase timing)

### Deliverable
Ships spawn and move; cannon hits destroy them; score increases.

### Requirements
- FR-400.1–FR-400.3
- FR-600.1–FR-600.2
- NFR-110.3

### Work items
- Implement `FleetManager` with spawn timer and cap.
- Implement ship logic for MVP roster: Regular Ship, Lander, Red Ship (+ Dark variants).
- Add HP + hit handling.
- Emit score events on ship destroyed.

### Verification
- Manual: ships spawn up to cap; can be destroyed.
- Tests:
  - `test/unit/test_fleet_spawn_cap.gd`
    - Asserts the active ship count cannot exceed the configured cap.
  - `test/unit/test_spawn_uniqueness.gd`
    - Asserts spawn cells are unique per wave.

---

## Milestone 8 — Craters + friendly fire differentiation

Depends on: Milestone 6 (damage source tagging + impacts), Milestone 3 (walls/structures exist to be damaged)

### Deliverable
AI-caused wall destruction produces craters; player-caused does not.

### Requirements
- FR-330.1–FR-330.3
- FR-410.1–FR-410.2

### Work items
- Add damage source tagging to projectile/explosion.
- Add crater placement + round-lifetime decrement.

### Verification
- Manual: simulate AI hit on wall cell → crater appears.
- Tests:
  - `test/unit/test_crater_lifetime.gd`
    - Asserts crater lifetime decrements by round and clears after N rounds.
  - `test/unit/test_friendly_fire_crater_rules.gd`
    - Asserts AI-caused wall destruction creates a crater, player-caused does not.

> Note: If MVP ships do **not** include Red Ship, crater logic can be stubbed and deferred.

> Note: Red Ship (and Dark Red Ship) are in MVP; crater logic is required.

---

## Milestone 9 — Game over + results

Depends on: Milestone 4 (validation failure condition), Milestone 1 (state transitions)

### Deliverable
Failing validation ends the run and shows score.

### Requirements
- FR-700.1–FR-700.2

### Work items
- Implement failure transition to `GameOver.tscn`.
- Display final score and allow restart.

### Verification
- Manual: fail on purpose → game over screen appears and restart works.
- Tests:
  - `test/integration/test_game_over_flow.gd`
    - Forces a validation failure and asserts transition to Game Over.
    - Asserts the displayed run seed matches the active run seed.

---

## Milestone 10 — Web export hardening

Depends on: Milestone 0 (boot), plus at least one full round loop implemented (Milestones 1–6)

### Deliverable
Playable Web build, stable and performant enough for external testing.

### Requirements
- NFR-100.1–NFR-100.2
- NFR-110.1
- NFR-140.1
- NFR-170.1–NFR-170.2

### Work items
- Add Web export preset (and document how to export).
- Audit assets for licensing.
- Add performance guardrails (caps, simplified particles).
- Ensure input works in browser (pointer lock not required).

### Verification
- Manual: export to Web and run in browser locally.
- Smoke test: run a few rounds without major stutters.

---

## Fast-follow (0.2) — Windows desktop parity

Depends on: Milestone 10 (Web export hardening complete)

### Focus items
- Export preset for Windows.
- Optional: higher-quality rendering path, improved audio latency handling.

### Requirements
- NFR-100.3

---

---

## Minimum test bar for MVP

- Flood fill: enclosed / open-to-boundary / leak-to-water.
- Build placement validation: rejects forbidden layers.
- Cannon queue: respects placement order and projectile-in-air rule.
- Ship cap: cannot exceed configured maximum.
