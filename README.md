# Fortressy

Fortressy is a Godot 4.6 project exploring a phase-based, grid-driven “Rampart”-style loop: build walls under time pressure, defend with sequential cannon fire, and claim territory by validating enclosed areas.

The current detailed design and implementation backlog live in `research.md`.

## Requirements

- Godot **4.6** (project is configured with `config/features` including `4.6`).

## Project structure

- `project.godot` — Godot project configuration
- `research.md` — architectural analysis and feature backlog
- `addons/gut/` — GUT (Godot Unit Test) addon (vendored)
- `test/` — test directory (configured in `.gutconfig.json`)

## Open the project

1. Open Godot.
2. In the Project Manager, choose **Import** and select this folder (or select `project.godot`).
3. Open the project.

## Run tests (GUT)

This repo includes the GUT plugin and a `.gutconfig.json` that points GUT at `res://test/`.

To run tests from the editor:

1. Open the project in Godot.
2. Confirm the plugin is enabled (Project Settings → Plugins → GUT).
3. Use the GUT UI to run tests.

## License

Source-available (public read-only). See `LICENSE`.
