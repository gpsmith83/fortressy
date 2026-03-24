# Development (local)

The CI harness is the source of truth. These commands mirror what runs in GitHub Actions.

## Requirements

- Godot **4.6.1** installed locally
- `godot` available on your PATH

## Run tests (GUT, headless)

```sh
godot --headless --path . --script res://addons/gut/gut_cmdln.gd -- -gconfig res://.gutconfig.json -gexit
```

## Export Web + smoke test

Ensure `export_presets.cfg` contains a preset named `Web`.

```sh
mkdir -p build
godot --headless --path . --export-release "Web" build/index.html

test -f build/index.html
test "$(ls build/*.wasm 2>/dev/null | wc -l)" -ge 1
test "$(ls build/*.pck 2>/dev/null | wc -l)" -ge 1
```

## Notes

- CI installs export templates automatically via `chickensoft-games/setup-godot@v1`.
- Locally, you may need to install export templates via the official Godot installer/download for your OS.
