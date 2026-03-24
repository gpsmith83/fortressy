# Fortressy — Asset List (MVP + Fast-Follow)

Date: 2026-03-24  
Primary references: `docs/mvp-plan.md`, `docs/requirements.md`, `research.md`

This document lists **art/audio/UI assets** needed to ship the MVP (Web-first). It’s intended to be practical: what to make, how many, and what can be placeholder/programmer-art initially.

Conventions:
- **MVP** = required for Release 0.1.
- **Fast-follow** = nice-to-have for Release 0.2 (Windows).
- “Programmer art ok” means a simple placeholder is acceptable as long as gameplay is testable.

---

## MVP gameplay implies these asset groups

From `docs/mvp-plan.md` and `docs/requirements.md`, MVP includes:
- Menu → Start → Gameplay scene
- Phase cycle: Setup → Battle → Build → Validate → CannonPlacement → repeat
- 42×30 grid, layered TileMapLayers: Water/Ground/Structures/Hazards
- Castles (2×2 footprint), perimeter walls, territory “claimed” visual
- Build pieces (5–7 shapes) with ghost valid/invalid tint
- Battle: crosshair + cannon firing queue + projectiles + explosions
- Enemy fleet (minimum roster) + optional grunts/craters depending on scope

---

## 1) Tileset + map visuals (MVP)

### TileSet textures (programmer art ok)
These can all live in one atlas image initially.

- **Ground tiles**
  - `ground_unclaimed` (1)
  - `ground_claimed` (1) — visually distinct (FR-220.2)
  - Optional variation tiles (2–6) to reduce repetition (nice-to-have)

- **Water tiles**
  - `water` (1)
  - Optional animated water frames (2–6) (nice-to-have)

- **Structure tiles**
  - `wall` (1) — placed via build pieces
  - `castle` (1) (but used as a 2×2 footprint)
  - `cannon_base` (1) (2×2 footprint; could also be a scene instead of tiles)

- **Hazard tiles** (only if you implement hazards in MVP)
  - `crater` (1) — unbuildable

### Map/background
- **1 authored map layout** (FR-200.6)
  - If using `MapData`, this is data rather than “art”, but it will depend on the tileset above.
- Optional: background/parallax layer image (1) (nice-to-have)

---

## 2) UI assets (MVP)

### Menu UI
- Title/logo text (can be plain Label + font) (MVP)
- Button styling:
  - Either theme-driven (no images) OR
  - Button sprites: normal/hover/pressed (3)

### HUD (FR-600.3)
- Phase label background panel (1) (can be theme-driven)
- Timer panel background (1) (can be theme-driven)
- Optional: small icons
  - `icon_timer` (1)
  - `icon_phase` (1)

### Debug overlay (recommended in MVP plan)
- Prefer text-only overlay (0 images required)

### Fonts
- 1 readable UI font (TTF/OTF) (MVP)
  - Godot default can be used temporarily.

---

## 3) Battle mode assets (MVP)

### Crosshair
- Crosshair sprite (1) (FR-300.1)

### Projectile + impact
- Cannonball sprite (1)
- Explosion animation frames (6–12) OR simple circle sprite (1) for MVP
- Optional: splash/impact on water animation (4–8) (nice-to-have)

### Cannon visuals
If cannons are their own scene(s) rather than tiles:
- Cannon sprite (1)
- Optional: firing flash (2–4 frames)

---

## 4) Build mode assets (MVP)

### Ghost piece visuals
- Use the same wall tile art, tinted green/red (0 extra art), OR:
  - Ghost wall tile variant (1) (nice-to-have)

### Cursor/placement indicator (optional)
- Cell highlight sprite (1) (nice-to-have)

---

## 5) Entities: enemies and ground troops (MVP roster)

Per `docs/requirements.md` **FR-400.2/FR-400.2a**, MVP ships include **Regular, Lander, Red** and **Dark** variants:
- `REGULAR`
- `LANDER`
- `RED`
- `DARK_REGULAR`
- `DARK_LANDER`
- `DARK_RED`

### Enemy ships (MVP)

For each of the 6 ship types above, plan at minimum:
- Ship sprite (1) or a small animation (2–4 frames)
- Optional damaged variant (1)

Notes:
- Dark variants can be implemented as recolors/palette swaps of the base ship art (recommended for MVP).
- If you use palette swaps, you still want separate exported textures (or a shader/tint plan) so it’s deterministic and easy to author.

### Grunts (MVP)

Per `docs/requirements.md` **FR-450**, grunts are MVP content.
- Grunt sprite (1) or 2-frame walk
- Optional: grunt hit/death sprite/VFX (can reuse explosion/hit spark)

### Craters (MVP)

Per `docs/requirements.md` **FR-410** and **FR-430.2**, craters are required in MVP (created by Red/Dark Red incendiary impacts).
- `crater` hazard tile (1)
- Optional: crater burning animation (2–6 frames) (nice-to-have)

---

## 6) Audio (MVP)

The MVP plan mentions structured logs and phase transitions; audio is optional but strongly helps feel.

### UI SFX (MVP)
- `ui_click` (1)
- `ui_back/cancel` (1)

### Phase transition cues (recommended)
- `phase_start_setup` (optional voice or beep) (1)
- `phase_start_battle` (1)
- `phase_start_build` (1)
- `phase_start_validate` (1)
- `phase_start_cannon` (1)

### Gameplay SFX (MVP)
- `cannon_fire` (1)
- `explosion` (1)
- `place_wall` (1)
- `invalid_place` (1) (or “dull click” for fire-queue empty)

### Music (optional for MVP)
- 1 looping gameplay track (1)
- 1 menu loop (1)

---

## 7) Asset directory suggestion (optional)

If you want a clean layout early:
- `res://assets/art/tiles/`
- `res://assets/art/ui/`
- `res://assets/art/entities/`
- `res://assets/audio/sfx/`
- `res://assets/audio/music/`
- `res://assets/fonts/`

---

## 8) Scope knobs: what you can keep as placeholders

Safe to keep as placeholder in MVP:
- Menu visuals (labels + default theme)
- Tileset (simple colored tiles)
- Explosion (single sprite + scaling)

Try not to placeholder for too long:
- Claimed vs unclaimed ground (needs to be obvious for gameplay)
- Crosshair (clarity)
- Wall vs water readability

---

## 9) Fast-follow (0.2) additions

- More map themes (2–3 additional tilesets)
- More enemy ship types + animations
- Richer VFX (water splash, crater burn animation)
- Audio mixing polish (bus ducking, toggles in Options)
- Accessibility/UI polish (scaling, colorblind-safe claimed territory)
