# Verification

How an implementation proves it satisfies this ghost library.

## Provenance

- **Upstream GDD:** Theme-Agnostic Card Battler MVP, hand-written, version `gdd-2026-05-23`.
- **Ghost library generated:** 2026-05-23.
- **Generation method:** Hand-built. Not the output of any skill (yet).
- **Spec version:** `gdd-2026-05-23` (matches the `version` field in `tests.yaml` and `examples/sample-match-trace.yaml`).

## What "passing" means

An implementation passes this ghost library iff:

1. Every operation in `SPEC.md` is implemented with the signatures and semantics described.
2. Every case in `tests.yaml` passes against the implementation via the test adapter described below.
3. The scenario `full-match-trace` reproduces every checkpoint in `examples/sample-match-trace.yaml`, including the deck-shuffle output, opening hands, every per-turn `state` block, and the final `winner = cpu` / `reason = deck_out` outcome.
4. The playable build meets every applicable acceptance criterion in GDD section 24.

Visual presentation, animation timing, code organization, and naming style are not part of "passing." They differ across implementations by design.

## Test adapter recipe

The agent generates a thin adapter in the implementation language that:

1. Parses `tests.yaml` (any YAML parser).
2. For each case:
   - Builds the initial state from `setup.state` (deep clone) or runs `setup.from_op` to get one (e.g., `match.init` then optional `then: [{op: match.dealOpeningHands}]`).
   - Dispatches `operation` to the implementation's reducer.
   - For each entry in `checks`, evaluates `op` against the result:
     - `equals`: deep equality at `path`
     - `not_equals`: inverse
     - `count`: array length equals `value`
     - `gte` / `lte`: numeric comparison
     - `contains`: array membership
     - `truthy`: value is truthy
     - `event_kind`: the events array contains an event whose `kind` and listed payload fields match
     - `event_order`: the events array contains all listed kinds in the given order (interleaving other events is allowed)
     - `throws`: the operation throws an error whose message includes `value`
3. Records pass/fail per case. The adapter exits non-zero if any case fails.

The adapter is implementation-specific and is NOT part of the ghost library. It can be deleted after verification; the contract lives in `tests.yaml`.

## Coverage table

| Operation              | Cases | Notes |
|------------------------|-------|-------|
| match.init             | 4     | Starting state, determinism, validation errors |
| match.dealOpeningHands | 2     | Standard deal, short deck |
| turn.startPhase        | 3     | Resource gain, cap at 6, attack flag reset |
| turn.draw              | 3     | Basic draw, deck-out causes loss, deck-out disabled |
| card.canPlay           | 4     | Affordability, target validity, replace-blocked |
| card.play              | 7     | All 6 reference cards plus replace behavior |
| effect.apply           | (via card.play) | Implicitly covered through card.play cases |
| combat.canAttack       | (via combat.attack) | Implicitly covered |
| combat.attack          | 7     | Unit-vs-unit, vs-player, Charger bonus, Surge buff, attack-twice error, lethal |
| turn.endPhase          | 2     | Active player swap, buff expiry |
| win.check              | 2     | Winner detection, no-winner state |
| cpu.chooseActions      | 4     | Priority 1, 3, 4, end-turn-when-stuck |
| **Total**              | **38** | |
| Scenarios              | 1     | full-match-trace (golden 18-turn trace) |

## What is NOT verified by this library

- **Animations and timing.** Smooth card flips, screen shake on damage, etc., are profile choices. Tests do not assert frame budgets or transition durations.
- **Audio playback.** Sound hooks are documented in `THEME.md` and `DESIGN.md` but tests do not assert specific sounds fire.
- **Pixel output.** No screenshot or visual diff testing. Two passing implementations will look different.
- **Performance.** No latency, FPS, or memory budget is enforced. Card battlers are not CPU-bound; this is a deliberate omission.
- **Accessibility.** Color contrast, keyboard navigation, and screen reader behavior are good practice but not asserted by tests. Implementations are encouraged to follow WCAG AA at minimum; verification is manual.
- **Cross-browser.** Build artifact must run in modern Chromium, Firefox, and WebKit. Verification is manual.

## Manual verification checklist (post-tests)

After `tests.yaml` passes, the implementer should manually verify:

- [ ] A new match starts from the main menu.
- [ ] Opening hand reveal animation plays and is skippable.
- [ ] HUD shows every required element from GDD section 16.2.
- [ ] Player can play every reference card type and observe its effect.
- [ ] CPU takes a full turn without input.
- [ ] Match terminates with a clear victory/defeat screen.
- [ ] Return to menu works.
- [ ] At least two themes load. Theme swap during a match does not reset state.
- [ ] Battle log shows every required event from GDD section 16.5.

## Regenerating the ghost library

This library is hand-built. To regenerate from the upstream GDD:

1. Run a `ghost-game` skill (when one exists) over the GDD.
2. Diff the regenerated `SPEC.md` against this one.
3. Differences indicate either (a) a skill bug, or (b) a GDD update that needs propagating to existing implementations.

The `version` field in `tests.yaml` and `sample-match-trace.yaml` must change whenever the spec changes. Implementations should bump their own version to match.

## Known limitations of this ghost library

These are documented honestly, not papered over.

1. **The "feel" of the game is not captured.** A passing implementation can still be unplayable if card animations stall, hover feedback is missing, or the click target hitboxes are wrong. The spec describes the rules engine. UI quality is implementer judgment.
2. **CPU is fully deterministic.** A real player will find this exploitable after a few matches. The spec keeps the CPU deterministic for testability; a future MECHANICS variation adds randomization for "fun" CPU at the cost of testability.
3. **No counter-attack.** The default rule set has no defender retaliation. This makes high-attack, low-HP units (Charger) dominant. The `counterAttack: mutual` MECHANICS variation rebalances but changes the game significantly.
4. **The golden trace is auto-vs-auto.** Both sides run `cpu.chooseActions` in the golden trace. A human-driven match will not reproduce it. The trace exists to verify the engine, not to demonstrate gameplay.
5. **No anti-cheat or save validation.** Single-player MVP. Adding network play would change this contract substantially.
6. **No deck builder.** Deck composition is passed to `match.init` by the consumer. A UI for building decks is an implementer choice and not in the spec.
