# STACK Variation Points

The rules engine in SPEC.md is stack-agnostic. The same `applyAction(state, action)` reducer runs unchanged across every profile. STACK choices affect rendering, input wiring, animation, and persistence.

Pick exactly one STACK profile. Mixing is not supported.

## Profile: vanilla-js-canvas (default)

The simplest target. No framework. Ship a single `index.html` plus one or two `.js` files plus a `style.css`. Deploy to any static host.

- **Renderer:** HTML5 Canvas 2D for card visuals, DOM for HUD overlays.
- **Input:** DOM event listeners on the canvas (`click`, `pointermove`).
- **State store:** A plain JS object holding the current `GameState`. After each `applyAction`, replace the object and call `render(state)`.
- **Animation:** `requestAnimationFrame` loop. Animations are driven by the event log: each emitted event maps to a brief visual cue.
- **Persistence:** None required for MVP. Use `sessionStorage` for the current match if you want refresh-resume.
- **When to pick:** Smallest agent output. Easiest to read end-to-end. Best for proving the spec works.

## Profile: react-canvas

React for HUD and card hand, canvas for the active board.

- **Renderer:** React components for `MainMenu`, `Hud`, `HandView`, `CardView`, `BattleLog`. Canvas (via a `useRef` and `useEffect`) for the active unit zone.
- **Input:** React event handlers, with explicit `onPlay`, `onAttack`, `onEndTurn` callbacks bubbling into a reducer.
- **State store:** `useReducer` over `applyAction`. Or Zustand if the agent prefers; both are fine.
- **Animation:** Framer Motion or CSS transitions for card movement; canvas for in-play unit effects.
- **Persistence:** None required. `localStorage` is fine if added.
- **When to pick:** The implementing agent is comfortable with React and the project benefits from componentized HUD.

## Profile: svelte-canvas

Svelte 5 (runes) for HUD, canvas for the board. Stack-equivalent of react-canvas with smaller bundle.

- **Renderer:** Svelte components for HUD; canvas mounted via a Svelte action.
- **Input:** Svelte event bindings (`on:click`).
- **State store:** A `$state` rune holding `GameState`. Update via `applyAction`, reassign the rune.
- **Animation:** Svelte transitions for card movement.
- **When to pick:** Agent prefers Svelte. Bundle size matters.

## Profile: phaser

Phaser 3 for the entire UI. Card battlers fit Phaser's scene model.

- **Renderer:** Phaser scenes: `BootScene`, `MainMenuScene`, `BattleScene`, `ResultsScene`.
- **Input:** Phaser interactive game objects.
- **State store:** A plain JS `GameState` held on the `BattleScene` instance. Reducer runs as pure JS, separate from Phaser internals.
- **Animation:** Phaser tweens.
- **Persistence:** None required. Phaser does not impose one.
- **When to pick:** Heavier visual juice desired (particle effects on damage, screen shake on defeat). Card battlers do not require this; pick it only if the design profile calls for it.

## Profile: pixijs

PixiJS for the board, plain HTML for HUD.

- **Renderer:** Pixi `Application` mounted into a container `<div>`. Sprites for cards and units. HUD is a sibling DOM element.
- **Input:** Pixi `eventMode` on sprites for card click; DOM listeners on HUD.
- **State store:** Plain JS object, same pattern as vanilla-js-canvas.
- **Animation:** GSAP or Pixi's built-in `Ticker`.
- **When to pick:** WebGL acceleration matters (lots of card animations, particle effects).

## Profile: three-js-cards

3D card battler. Cards are flat planes in 3D space, with depth and rotation animations.

- **Renderer:** Three.js. Cards as planes with image textures. Camera fixed angle, slight perspective.
- **Input:** Raycaster for card pick on click.
- **State store:** Plain JS object.
- **Animation:** Three's animation system or GSAP.
- **When to pick:** "Pack-opening" reveal is the headline feature and 3D card flips justify the budget.
- **Caveat:** Most agents will overbuild this. Vanilla-js-canvas + good CSS flip animations produces a similar reveal with a tenth of the code.

## Cross-profile guarantees

Every profile must:

1. Implement every operation in SPEC.md without modification to the reducer logic.
2. Pass every case in `tests.yaml` against a thin test adapter (see VERIFY.md).
3. Emit the same events in the same order from the same `(state, action)` inputs.
4. Use the xorshift32 PRNG with Fisher-Yates shuffle from SPEC.md Appendix A.
5. Render the HUD fields listed in GDD section 16.2: player health, CPU health, current turn owner, current phase, player resource and max, CPU resource and max, player hand, CPU hand count, both active units, deck count, discard count, end turn button, battle log.

Visual fidelity, animation timing, and card layout are profile-specific. Tests do not assert pixel output.

## Incompatibilities

- **phaser + design profile "minimal":** Phaser's overhead is not justified by minimal design. Pick vanilla-js-canvas instead.
- **three-js-cards + design profile "pixel":** Pixel art does not benefit from 3D card geometry. Pick vanilla-js-canvas or pixijs.

If the consumer requests an incompatible combo, the implementing agent must refuse and ask for a compatible pair, citing this section.
