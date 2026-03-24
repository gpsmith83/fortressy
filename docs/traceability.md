# Fortressy — Traceability (MVP)

Date: 2026-03-24  
Primary references: `docs/requirements.md`, `docs/mvp-plan.md`

This document provides a mechanical mapping from requirements → milestones → tests → implementation files.

## Update contract (MUST follow)

When adding/implementing/changing anything that affects gameplay behavior:

- If you implement/modify a requirement (FR/NFR), you MUST:
  - add/update the row(s) in this matrix, and
  - add/update the associated test(s) (GUT) where feasible.
- If you add a new unit or integration test, you MUST add it to the relevant requirement row(s).
- If you create new core modules/scripts that implement requirement logic, you MUST list them here.

PR / task definition-of-done:
- The PR/task is not “done” until `docs/traceability.md` has been updated (or explicitly marked `TBD` with a follow-up issue).

Conventions:
- **Milestone** refers to `docs/mvp-plan.md` milestone numbers.
- **Tests** are file paths under `test/unit/` or `test/integration/`.
- **Implementation** is the primary script/scene/resource expected to satisfy the requirement.
  - If not yet created, leave as `TBD`.

---

## Functional Requirements (FR) → Milestones → Tests → Implementation

| Requirement | Milestone(s) | Test(s) | Implementation |
|---|---:|---|---|
| FR-000.1 main_scene boots | 0 | TBD | `project.godot` (main_scene), `Main.tscn` |
| FR-000.2 main menu (Start/Options/Quit) | 0 | TBD | `Main.tscn` / menu scripts (TBD) |
| FR-000.3 Start transitions to gameplay scene | 0 | TBD | `Main.tscn`, `Game.tscn` |
| FR-100.1 phase FSM w/ isolated input | 1 | `test/unit/test_fsm_phase_progression.gd` | `GameManager` (TBD), state scripts (TBD) |
| FR-100.2 MVP phases exist | 1 | `test/unit/test_fsm_phase_progression.gd` | `StateSetup`, `StateBattle`, `StateBuild`, `StateValidate`, `StateCannonPlacement` (TBD) |
| FR-100.3 transitions based on timers/conditions | 1 | `test/unit/test_fsm_phase_progression.gd` | `GameManager` (TBD) |
| FR-100.4 inputs outside phase do nothing | 1 | TBD | per-state input handlers (TBD) |
| FR-100.5 emit `phase_changed` signal/event | 1 | `test/unit/test_fsm_phase_progression.gd` | `GameManager` + event log (TBD) |
| FR-110.1 battle countdown visible | 1 | TBD | HUD (CanvasLayer) (TBD) |
| FR-110.2 build countdown visible | 1 | TBD | HUD (CanvasLayer) (TBD) |
| FR-110.3 battle end locks input + waits for drain then build | 1, 6 | TBD | `GameManager`, projectile tracking (TBD) |
| FR-110.4 build end runs validation and proceeds | 4 | TBD | `StateValidate` (TBD) |
| FR-110.4a active (unplaced) piece rejected on build end | 3, 4 | TBD | `BuildController` (TBD) |
| FR-200.1 map grid addressable by `Vector2i` | 2 | `test/unit/test_grid_manager_bounds.gd` | `GridManager` (TBD) |
| FR-200.2 layered TileMapLayer map | 2 | TBD | map scene (TBD) |
| FR-200.3 logical layers Water/Ground/Structures/Hazards | 2 | TBD | map scene + tileset metadata (TBD) |
| FR-200.4 tiles expose metadata for queries | 2 | TBD | tileset + `GridManager` (TBD) |
| FR-210.1 map has one+ castles at fixed cells | 2 | TBD | map data/scene (TBD) |
| FR-210.2 select Home castle in Setup | 2 | TBD | `StateSetup` (TBD) |
| FR-210.3 selecting Home initializes perimeter | 2 | `test/unit/test_perimeter_generation.gd` | perimeter generator module (TBD) |
| FR-210.4 castle footprint 2×2 | 2 | `test/unit/test_perimeter_generation.gd` | `Castle` model/node (TBD) |
| FR-220.1 mark enclosed cells as claimed | 4 | TBD | territory painter (TBD) |
| FR-220.2 claimed territory visually distinct | 4 | TBD | tiles/materials (TBD) |
| FR-230.1 validate at least one castle enclosed | 4 | `test/unit/test_flood_fill_validator.gd` | `FloodFillValidator` (TBD), `StateValidate` (TBD) |
| FR-230.2 flood fill rules (boundary/water fail) | 4 | `test/unit/test_flood_fill_validator.gd` | `FloodFillValidator` (TBD) |
| FR-230.3 validation across castles as needed | 4 | TBD | `StateValidate` (TBD) |
| FR-240.1 piece sequence during Build | 3 | TBD | piece generator (TBD) |
| FR-240.2 active piece follows cursor/snap | 3 | TBD | `BuildController` (TBD) |
| FR-240.3 rotate piece 90° | 3 | `test/unit/test_piece_model_rotation.gd` | `PieceModel` (TBD) |
| FR-240.4 placement validation each frame w/ feedback | 3 | TBD | `BuildController` + ghost visuals (TBD) |
| FR-240.5 placement permanent for Build | 3 | TBD | `BuildController` (TBD) |
| FR-240.6 pieces don’t overlap, must be buildable ground | 3 | `test/unit/test_placement_validator_rules.gd` | `PlacementValidator` (TBD) |
| FR-250.1 forbidden overlaps (water/wall/hazard/cannon/grunt) | 3 | `test/unit/test_placement_validator_rules.gd` | `PlacementValidator` (TBD) |
| FR-250.2 any invalid cell fails whole placement | 3 | `test/unit/test_placement_validator_rules.gd` | `PlacementValidator` (TBD) |
| FR-300.1 crosshair follows pointer during Battle | 6 | TBD | crosshair controller (TBD) |
| FR-300.2 crosshair constrained to viewport | 6 | TBD | crosshair controller (TBD) |
| FR-310.1 cannons as placeable structures | 5 | TBD | cannon placement/controller (TBD) |
| FR-310.1a cannon footprint 2×2 | 5 | `test/unit/test_cannon_placement_restrictions.gd` | cannon placement/controller (TBD) |
| FR-310.2 sequential firing in placement order | 6 | `test/unit/test_cannon_queue_order.gd` | `CannonQueue` (TBD) |
| FR-310.3 one projectile in flight per cannon | 6 | `test/unit/test_cannon_queue_order.gd` | `CannonQueue` + cannon state (TBD) |
| FR-310.4 battle end waits for in-flight to resolve | 6 | TBD | `GameManager` / projectile tracking (TBD) |
| FR-320.1 time-of-flight proportional to distance | 6 | `test/unit/test_projectile_time_of_flight.gd` | projectile script (TBD) |
| FR-320.2 arc simulation visual | 6 | TBD | projectile visuals (TBD) |
| FR-320.3 impact event + AoE | 6 | TBD | explosion/impact system (TBD) |
| FR-500.1 enter Cannon Placement after validation success | 5 | TBD | `StateCannonPlacement` (TBD) |
| FR-500.2 cannon rewards (+2 home, +1 additional) | 5 | `test/unit/test_cannon_awards.gd` | award calculator module (TBD) |
| FR-500.3 cannons placeable only within claimed | 5 | `test/unit/test_cannon_placement_restrictions.gd` | cannon placement validator (TBD) |
| FR-600.3 HUD shows phase + timers | 1 | TBD | HUD (CanvasLayer) (TBD) |

---

## Key Non-Functional Requirements (NFR) → Milestones → Tests → Implementation

| Requirement | Milestone(s) | Test(s) | Implementation |
|---|---:|---|---|
| NFR-100.1 runs on Web export | 0, 10 | TBD | export preset + web-safe input/assets (TBD) |
| NFR-110.2 flood fill completes within a frame (42×30) | 4 | TBD | `FloodFillValidator` (TBD) |
| NFR-120.1 seeded RNG reproducibility (pieces/ships/enemy fire) | 3, 7, 10 | TBD | `RunConfig`/RNG streams (TBD) |
| NFR-120.2 deterministic phase transitions | 1 | `test/unit/test_fsm_phase_progression.gd` | `GameManager` (TBD) |
| NFR-130.1 minimum test bar (flood fill, placement, queue, crater) | 3, 4, 6, 8 | see unit tests named in plan | GUT suite |
| NFR-160.1 core systems separated into testable modules | 1–8 | evidence via unit tests | `scripts/` pure modules (TBD) |
| NFR-160.2 constants data-driven (timers/scoring/caps) | 1, 7, 10 | TBD | config resources (TBD) |

---

## Determinism golden outputs (planned artifacts)

These files do not exist yet; they are required by FR-710.8 once the event log + RNG-driven systems land.

- Test: `test/integration/test_determinism_golden_seed.gd` (TBD)
- Golden data: `test/data/golden/seed_<run_seed>.jsonl` (TBD)
