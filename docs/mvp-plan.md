# Fortressy — MVP Plan (Web-first)

Date: 2026-03-23  
Primary references: `docs/requirements.md`, `research.md`

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

---

## Milestone 0 — Repository “playable boot” skeleton

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
- Test: unit test for phase progression ordering (at least one cycle).

---

## Milestone 2 — Grid foundation + map + castle selection

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
- Implement `Castle` nodes and selection logic in `StateSetup`.
- Implement initial perimeter placement around castle footprint.

### Verification
- Manual: click castle → perimeter appears.
- Test: perimeter generation returns expected set of cells for a known castle size.

---

## Milestone 3 — Build system: piece generation, rotation, placement validation

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
  - rotation correctness
  - validation rejects water/walls/hazards/cannons

---

## Milestone 4 — Flood fill validation + territory claiming

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

### Verification
- Tests with small synthetic grids for:
  - enclosed
  - open to boundary
  - leak to water
- Manual: build a loop → claim occurs.

---

## Milestone 5 — Cannon placement phase + cannon inventory

### Deliverable
After successful validation, player places awarded cannons within claimed territory.

### Requirements
- FR-500.1–FR-500.3
- FR-310.1

### Work items
- Represent cannon as a grid-occupying entity (MVP 1×1).
- Implement cannon award calculation.
- Implement placement UI similar to Build placement, restricted to claimed territory.
- Register cannons in a placement-order list.

### Verification
- Manual: cannons can’t be placed outside claimed.
- Test: award calculation for home vs additional castles.

---

## Milestone 6 — Battle: crosshair + sequential cannon firing + projectile impacts

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
  - queue order correctness
  - projectile time-of-flight monotonic with distance

---

## Milestone 7 — Enemies: one ship type + spawner + damage and scoring

### Deliverable
Ships spawn and move; cannon hits destroy them; score increases.

### Requirements
- FR-400.1–FR-400.3
- FR-600.1–FR-600.2
- NFR-110.3

### Work items
- Implement `FleetManager` with spawn timer and cap.
- Implement `Ship` (Regular Ship): simple deterministic movement.
- Add HP + hit handling.
- Emit score events on ship destroyed.

### Verification
- Manual: ships spawn up to cap; can be destroyed.
- Test: spawn cap holds.

---

## Milestone 8 — Craters + friendly fire differentiation (if Red-Ship is in MVP)

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
- Tests: crater lifetime counts down across rounds.

> Note: If MVP ships do **not** include Red Ship, crater logic can be stubbed and deferred.

---

## Milestone 9 — Game over + results

### Deliverable
Failing validation ends the run and shows score.

### Requirements
- FR-700.1–FR-700.2

### Work items
- Implement failure transition to `GameOver.tscn`.
- Display final score and allow restart.

### Verification
- Manual: fail on purpose → game over screen appears and restart works.

---

## Milestone 10 — Web export hardening

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

### Focus items
- Export preset for Windows.
- Optional: higher-quality rendering path, improved audio latency handling.

### Requirements
- NFR-100.3

---

## Suggested “open decision” order (to unblock engineering)

1. Confirm MVP enemy roster:
   - Regular Ship only? Or include Red Ship to justify crater/friendly-fire systems in MVP.
2. Confirm grid size + map loading approach (scene-based vs resource-driven).
3. Decide credits vs single-run game over for MVP.
4. Decide castle footprint size for MVP.

---

## Minimum test bar for MVP

- Flood fill: enclosed / open-to-boundary / leak-to-water.
- Build placement validation: rejects forbidden layers.
- Cannon queue: respects placement order and projectile-in-air rule.
- Ship cap: cannot exceed configured maximum.
