# INSTALL: Generating an Implementation

This is a ghost library. No code ships in this package. To get a playable game, hand the files in this directory to a coding agent (Claude Code, Codex CLI, Cursor, etc.) along with the prompt below.

## Quick-start prompt

Paste this into your agent of choice. Fill in `[STACK]`, `[DESIGN]`, `[THEME]`, `[MECHANICS_OVERRIDES]`.

````
You are implementing a single-player browser card battler from a ghost library
(spec only, no code). Generate a complete, playable implementation.

Read every file in the ghost library before writing code:
  SPEC.md        - the behavior contract. Source of truth for game logic.
  tests.yaml     - 38 cases + 1 scenario the implementation must pass.
  STACK.md       - pick a stack profile.
  DESIGN.md      - pick a visual profile.
  MECHANICS.md   - pick rule variations (or use defaults).
  THEME.md       - theme pack contract.
  VERIFY.md      - how to verify the implementation against tests.yaml.
  examples/sample-match-trace.yaml   - golden deterministic match trace.
  examples/theme-pack-example.yaml   - sample theme pack.

Your configuration:
  STACK:    [STACK]              (one of: vanilla-js-canvas, react-canvas, svelte-canvas, phaser, pixijs, three-js-cards)
  DESIGN:   [DESIGN]             (one of: clean-modern, pixel, neon, hand-drawn, minimal)
  THEME:    [THEME]              (default neutral plus one of your invention, or use the Forest Friends sample)
  MECHANICS: [MECHANICS_OVERRIDES]  (e.g., counterAttack=mutual, or "use SPEC defaults")

Hard rules:
1. The rules engine (applyAction reducer + every operation in SPEC.md) MUST be
   implemented exactly as specified. Do not invent new rules. Do not change
   event order within an operation. Do not change the state schema.
2. Use the xorshift32 PRNG and Fisher-Yates shuffle from SPEC.md Appendix A.
   Any other PRNG fails the golden traces.
3. Implement every operation in SPEC.md. Each operation must be callable from
   the test adapter described in VERIFY.md.
4. Pass every case in tests.yaml. Generate a test file in the target language's
   conventional test framework (Vitest, Jest, Playwright unit, whatever fits).
   Do not stub or skip cases. Mark a case skipped only if it tests a MECHANICS
   variation that your configuration does not enable, and only after explaining
   why in your output.
5. Ship a playable single-page build at index.html (or the stack equivalent)
   that meets every GDD acceptance criterion (section 24 of the upstream GDD).

Process:
1. Read SPEC.md cover to cover. Sketch the state shape and the operation
   signatures in the target language. Do not write game logic yet.
2. Build the rules engine first. No UI. Just types + pure functions for every
   operation. Run tests.yaml against it. Iterate until all cases pass.
3. Build the UI per the STACK and DESIGN profiles. Wire it to the rules engine.
4. Build the theme layer per THEME.md. Ship the default neutral theme plus one
   alternate.
5. Verify per VERIFY.md.

What you DO NOT have to do:
- Multiplayer, accounts, persistence, deck builder, card packs, ranked play.
  All of these are explicitly out of MVP scope per the GDD.
- Final art. Use placeholders. Note where real art would go.
- Final audio. Optional hooks per DESIGN.md sound asset slots.

Style:
- No em dashes. Use commas, periods, or parentheses.
- Terse, action-oriented commit messages.
- Fail fast. Throw on invalid actions; do not silently no-op.

Output a single sentence summary of your STACK / DESIGN / THEME choices, then
proceed. Ask for clarification only if a required input is missing.
````

## Filling in the blanks

### STACK

- `vanilla-js-canvas` (recommended for first implementation; smallest)
- `react-canvas` (good if you want componentized HUD)
- `svelte-canvas` (smaller bundle than React)
- `phaser` (heavier; pick only for juicy visual feedback)
- `pixijs` (WebGL acceleration, plain HTML HUD)
- `three-js-cards` (3D card flips; usually overkill)

### DESIGN

- `clean-modern` (default; safe for any theme)
- `pixel` (retro; pairs with cute or fantasy themes)
- `neon` (synthwave; pairs with sci-fi or robot themes)
- `hand-drawn` (cute or whimsical mood)
- `minimal` (no art; great for first proof)

### THEME

Either:

- Use the sample `Forest Friends` theme in `examples/theme-pack-example.yaml`.
- Or specify a frame ("sci-fi robots", "dinosaurs", "kaiju", "fantasy heroes") and let the implementing agent author names, flavor text, and palette overrides per THEME.md.

### MECHANICS

If unsure, write `use SPEC defaults`. Otherwise list overrides explicitly:

    counterAttack=mutual
    activeUnitReplaceRule=block_new
    deckOutCausesLoss=false
    cpuDifficulty=easy

The agent reads MECHANICS.md to apply them.

## Two-agent comparison run (Randy's intended use)

To prove the ghost library is portable, hand identical configurations to two different agents (or the same agent in two sessions with cleared context). Compare:

1. Both implementations pass every case in `tests.yaml`.
2. The `examples/sample-match-trace.yaml` golden trace reproduces identically across both.
3. Visual presentation differs (this is expected and good).
4. Code style, file organization, and animation choices differ (also expected).

Where the two implementations diverge in spec-relevant behavior, that is a bug in one of them, not a bug in the spec. File the bug against the implementation; the spec is the contract.

## What if my agent struggles?

Common failure modes and fixes:

- **PRNG drift:** Implementation used `Math.random()` or a non-xorshift PRNG. The golden trace will fail. Force the agent to use Appendix A verbatim.
- **Event order off:** Implementation emits `UNIT_DEFEATED` before `DAMAGE_DEALT` or similar. SPEC.md is explicit about order; point the agent at the affected operation.
- **Skipped tests:** Implementation marked cases as "TODO" or "skipped." This violates the contract. Require all 38 cases pass before declaring done.
- **Schema drift:** Implementation added or renamed state fields. SPEC.md state schema is the contract. Field additions must be discussed; renames are not allowed.
- **CPU non-deterministic:** Implementation introduced randomness in CPU choice. SPEC.md `cpu.chooseActions` is fully deterministic given identical inputs. No PRNG calls in the CPU.

## Regenerating the ghost library

Re-run the `ghost-game` skill on the upstream `gdd.md` to regenerate this directory. Diff the regenerated `SPEC.md` against the previous version to surface drift between spec authors. The tests.yaml and golden trace are stable across regenerations as long as the upstream GDD is unchanged.
