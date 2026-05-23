# MECHANICS Variation Points

Every variation here is a rule swap, not a theme swap. Each variation declares:

1. **Default** (matches GDD and SPEC.md).
2. **Alternates** and what changes.
3. **Affected operations** in SPEC.md.
4. **Tests impacted** in `tests.yaml`.

Apply at most one alternate per variation point.

## VAR-1: Active unit replacement rule

- **Default:** `discard_previous`. Playing a unit while one is already active discards the previous unit.
- **Alternate:** `block_new`. Playing a unit while one is active is illegal; the card is unplayable.
- **Affected operations:** `card.canPlay`, `card.play`.
- **Tests impacted:** `play.replace-discards-previous` flips to expect a `throws` or `unplayable` result; `canPlay.unit-blocked-when-replace-blocked` is the alternate's canonical case.
- **Config field:** `config.activeUnitReplaceRule`.

## VAR-2: Deck-out causes loss

- **Default:** `true`. Drawing from an empty deck ends the match; the empty-deck player loses.
- **Alternate:** `false`. Drawing from an empty deck skips the draw; play continues.
- **Affected operations:** `turn.draw`, `win.check`.
- **Tests impacted:** `draw.empty-deck-causes-loss` vs `draw.empty-deck-no-loss-when-disabled`.
- **Config field:** `config.deckOutCausesLoss`.

## VAR-3: Maximum hand size

- **Default:** `null` (unlimited).
- **Alternate:** `7`. Cards drawn past the cap go directly to discard.
- **Affected operations:** `turn.draw`, `effect.apply` (when `draw_card`).
- **Tests impacted:** Add cases for hand-cap discard.
- **Config field:** `config.maxHandSize`.

## VAR-4: First player

- **Default:** `player` goes first.
- **Alternates:** `cpu`, `random` (seeded coin flip on init).
- **Affected operations:** `match.init`.
- **Tests impacted:** None of the listed cases assume which side acts first beyond init; CPU-first changes the opening trace in `examples/sample-match-trace.yaml`.
- **Config field:** `config.firstPlayer`.

## VAR-5: Counter-attack on unit-vs-unit

- **Default:** No counter-attack. Defender takes damage but deals none back.
- **Alternate:** `mutual`. After attacker deals damage, defender (if still alive) deals its `effectiveAttack` to the attacker. Both `UNIT_ATTACKED` events fire; the second has `attackerRef = defender, targetRef = attacker, counter: true`.
- **Affected operations:** `combat.attack`.
- **Tests impacted:** `attack.unit-vs-unit-damages` expects player Striker's HP unchanged. Under `mutual`, expect player Striker HP = 3 - 1 = 2 (defender Guardian's attack of 1).
- **Config field:** `config.counterAttack` (`none` | `mutual`).
- **Caveat:** Enabling this changes balance significantly. Guardian becomes much stronger; Striker much weaker.

## VAR-6: Resource refill behavior

- **Default:** Refills to current `maxResource` at start of each owned turn.
- **Alternate:** `carryover`. `currentResource` is not refilled; instead `currentResource += 1` (or whatever the per-turn gain is) and is capped at `maxResource`. MTG-style mana banking without lands.
- **Affected operations:** `turn.startPhase`.
- **Tests impacted:** `start.gains-one-resource-turn-one` checks `currentResource` post-state; under `carryover` the value depends on how much was unspent.
- **Config field:** `config.resourceRefill` (`full` | `carryover`).

## VAR-7: Resource cap

- **Default:** `6`.
- **Alternates:** Any positive integer. Reasonable ranges: 5 to 10.
- **Affected operations:** `turn.startPhase`, `card.canPlay`.
- **Tests impacted:** `start.caps-resource-at-six` uses 6; the cap is parameterized by `resourceCap`.
- **Config field:** `config.resourceCap`.

## VAR-8: Starting health

- **Default:** `20`.
- **Alternates:** Any positive integer. Common alternates: `15`, `30`.
- **Affected operations:** `match.init`.
- **Tests impacted:** Health-related cases would need re-baselining if changed.
- **Config field:** `config.startingHealth`.

## VAR-9: Starting hand size

- **Default:** `3`.
- **Alternates:** Any non-negative integer up to deck size. Common: `4`, `5`.
- **Affected operations:** `match.dealOpeningHands`.
- **Config field:** `config.startingHandSize`.

## VAR-10: Deck composition

- **Default:** Two copies of each of the six reference cards: twelve cards per player.
- **Alternates:** Any multiset of the six card ids, plus any new cards the consumer adds via the catalog extension (see "Adding new cards" below).
- **Affected operations:** `match.init`.
- **Tests impacted:** None of the cases lock to deck composition beyond `init.starting-state` which counts 12.
- **Config field:** Deck arrays passed to `match.init` (`deckP`, `deckC`).

## VAR-11: CPU difficulty

- **Default:** The priority list in SPEC.md `cpu.chooseActions`.
- **Alternates:**
  - `easy`: skip priority 1 (threat removal) and priority 2 (lethal). CPU plays reactively, not optimally.
  - `hard` (future): full priority list plus look-ahead one turn. Not specified for MVP; documented as a future extension.
- **Affected operations:** `cpu.chooseActions`.
- **Tests impacted:** CPU cases assume `default`. Mark them as `cpuDifficulty: default` if the implementation supports the variation.
- **Config field:** `config.cpuDifficulty`.

## Adding new cards (not a variation, but a controlled extension)

The card catalog accepts new entries as long as each entry uses only effect types declared in SPEC.md `effect.apply` (`deal_damage`, `heal`, `modify_attack`, `draw_card`, `summon_or_replace_unit`).

New cards must declare:

- `id` (unique, prefixed `unit_` or `action_`)
- `baseName`
- `type` (`unit` or `action`)
- `cost` (integer, 0 to `resourceCap`)
- For units: `maxHealth`, `attack`, `abilities[]`
- For actions: `effects[]`

Cards using effect types outside the declared vocabulary are out of scope for MVP and must not be added. If the consumer wants a new effect type, that is a SPEC.md change, not a MECHANICS variation.

## What is NOT a variation point

- The event names and event order within an operation. These are part of the determinism contract.
- The state schema. Adding fields is a SPEC.md change.
- The xorshift32 PRNG. Replacing it breaks the golden traces.
- The six reference cards' stats. The catalog is extended, not edited.
