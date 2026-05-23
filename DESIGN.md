# DESIGN Variation Points

The rules engine does not specify pixels. DESIGN profiles bundle a palette, typography, card frame style, and layout density. Pick one. Combine with a THEME pack for full visual identity.

## Profile: clean-modern (default)

Soft neutral palette, sans-serif type, generous whitespace.

- **Palette:**
  - background: `#F5F5F0`
  - card-face: `#FFFFFF`
  - card-border: `#1F1F1F`
  - accent-friendly: `#3B82F6`
  - accent-hostile: `#EF4444`
  - resource: `#F59E0B`
  - text-primary: `#1F1F1F`
  - text-muted: `#6B7280`
- **Type:** Inter or system-ui sans, weights 400 / 600 / 700.
- **Card frame:** 2:3 ratio. Rounded corners (12px). Thin border. Name banner at top. Cost in upper-left circle. Attack and HP in lower corners for units.
- **Density:** Spacious. Hand spreads across the bottom with overlap only when needed.
- **Reveal animation:** Cards flip in from facedown with a 250ms ease.

## Profile: pixel

Retro pixel art. 16-bit feel. NES/SNES color depth.

- **Palette:**
  - background: `#1A1C2C`
  - card-face: `#F4F4F4`
  - card-border: `#000000`
  - accent-friendly: `#3FA9F5`
  - accent-hostile: `#E64539`
  - resource: `#FFD93D`
  - text-primary: `#222034`
- **Type:** A pixel font (Press Start 2P, VT323, or similar). Nearest-neighbor scaling only.
- **Card frame:** Square edges. Heavy 2-pixel border. Card art is a sprite, no anti-aliasing.
- **Density:** Tight. Cards overlap.
- **Reveal animation:** Cards "pop" in with a single-frame scale.
- **Incompatibilities:** three-js-cards.

## Profile: neon

Synthwave. High contrast, glow effects, dark background.

- **Palette:**
  - background: `#0B0B1F`
  - card-face: `#1A1A3F`
  - card-border: `#FF00FF`
  - accent-friendly: `#00FFFF`
  - accent-hostile: `#FF1F5A`
  - resource: `#FFEE00`
  - text-primary: `#FFFFFF`
  - glow-base: `0 0 12px currentColor`
- **Type:** A geometric display sans for headers (Orbitron, Audiowide), monospace for stats.
- **Card frame:** Beveled edges with a glow outline. Inner border in the accent of the card's faction or role.
- **Density:** Medium. Glow needs breathing room.
- **Reveal animation:** Cards slide in from below with a glow pulse.

## Profile: hand-drawn

Sketchy, paper texture, slight wobble. Cute or whimsical mood.

- **Palette:**
  - background: paper texture, base `#F5EFE0`
  - card-face: `#FFFCF5`
  - card-border: ink black, slightly variable
  - accent-friendly: `#5B8C5A`
  - accent-hostile: `#C44536`
  - resource: `#E3B23C`
- **Type:** A handwritten font (Caveat, Patrick Hand) for body; a bolder one (Permanent Marker) for the card name.
- **Card frame:** Hand-drawn rectangle, slight imperfection. Stat icons look hand-inked.
- **Density:** Spacious.
- **Reveal animation:** Cards "wobble" into place.

## Profile: minimal

Wireframe-adjacent. No imagery. Type and color only.

- **Palette:** Two-tone plus an accent. background `#FFFFFF`, text `#000000`, accent `#0066FF`.
- **Type:** A single sans-serif at 2-3 sizes.
- **Card frame:** Outline only. No art. Name + cost + stats + ability text.
- **Density:** Maximal information per card. No flourish.
- **Reveal animation:** Instant. No animation.
- **When to pick:** Demos where the rules are the point and visuals would distract. Useful for the first implementation pass.

## Required HUD elements (all profiles)

Per GDD section 16.2, the HUD must surface:

- Player health
- CPU health
- Current turn owner (highlighted)
- Current phase (text or icon)
- Player resource and max
- CPU resource and max
- Player hand (faces visible)
- CPU hand count (facedown count, faces hidden)
- Player active unit (with current HP, attack, modifiers visible)
- CPU active unit
- Deck count (both players)
- Discard count (both players)
- End Turn button
- Battle log (scrolling or last-N feed)

A profile's job is to make these legible at a glance.

## Color tokens

All profiles must expose CSS custom properties (or equivalent in the chosen stack):

    --bg, --card-face, --card-border,
    --accent-friendly, --accent-hostile, --resource,
    --text-primary, --text-muted

The THEME pack overrides these tokens, not the DESIGN profile. DESIGN sets defaults; THEME overrides.
