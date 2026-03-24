# Agent map for Fortressy (Godot 4.6)

This repository is optimized for **agent-first** work: small, verifiable changes with mechanical checks.

## Start here

- Product + acceptance criteria: `docs/requirements.md`
- Current execution plan: `docs/mvp-plan.md` (will be migrated into `docs/plans/` as plans mature)
- Requirement-to-tests mapping (MUST keep updated): `docs/traceability.md`
- Deep technical analysis/backlog: `research.md`
- Local dev / CI-equivalent commands: `docs/dev.md`

## How to work in this repo (rules that keep things legible)

1) **Prefer pure logic modules** for grid / validation / queues. Keep them decoupled from scene tree when possible.
2) **Add tests as you add behavior** (GUT). If you change a requirement, update tests + docs in the same PR.
	- When implementing or changing behavior tied to an FR/NFR, you MUST update `docs/traceability.md` in the same PR.
3) **Validate before you finish**: run the same commands CI runs (see `docs/dev.md`).
4) **Small PRs**: one capability at a time; include verification notes.

## Quality gates / Definition of Done (DoD)

Canonical DoD lives in `docs/mvp-plan.md` under **“Definition of Done (DoD) / quality gates”**.

As a rule, don’t mark work complete unless:

- You can exercise the change (editor run / relevant scene flow).
- There are **no new errors** in Godot output for the verification scenario.
- Tests are green (new tests added when adding/changing behavior).
- `docs/traceability.md` is updated when requirements/behavior-to-tests mapping changes.

## Tooling

### Requirements

- Godot **4.6.1** (CI pins this version)

### Run tests

- In CI we run GUT headlessly via the Godot CLI:
	- `godot --headless --path . --script res://addons/gut/gut_cmdln.gd -- -gconfig res://.gutconfig.json -gexit`

### Export + smoke test + deploy (Web)

- CI exports using preset `Web` (from `export_presets.cfg`) and deploys the `build/` directory to GitHub Pages.
- Smoke test checks that the export output includes `index.html`, at least one `.wasm`, and at least one `.pck`.

## Repository knowledge is the system of record

If a decision is important for future changes, encode it in-repo:

- Update `docs/requirements.md` for locked decisions / acceptance criteria.
- Update or add a plan in `docs/` (later: `docs/plans/active`).
- Add/update tests.

## CI expectations

CI runs on GitHub Actions and:

- downloads pinned Godot
- runs headless GUT tests
- runs doc/structure checks (lightweight)
