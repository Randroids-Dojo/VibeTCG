# Theme-Agnostic Card Battler: Spec

## Provenance

- Upstream GDD: Theme-Agnostic Card Battler MVP, sections 1 through 25
- Ghost library generated: 2026-05-23
- Tool: hand-built (v0)

## Overview

A single-player browser card battler. Human vs CPU. Six reference cards. Six-turn maximum resource ramp. Discrete state transitions, integer math, no floats in the rules. Theme is data layered on top.

## Pillars

The rules engine is theme-agnostic. The same engine drives any reskin via a theme pack. Card behavior is composed from a small effect vocabulary. State is fully serializable. CPU is deterministic given identical inputs.

## Contract shape

Pure functional reducer:

    applyAction(state, action) -> { state, events }

Every operation in this spec is either a reducer of that shape or a pure query against `state`. No clocks. No `Date.now()`. No I/O. Determinism is required: identical `(state, action)` always produces identical `{ state, events }`.

RNG is a seedable xorshift32. Reference implementation in Appendix A. Every operation that consumes randomness reads from and advances `state.rngState`.

## State schema

Plain serializable JSON. No engine types.

    GameState := {
      matchId: string,
      seed: number,                   // initial seed, immutable after init
      rngState: number,               // mutable PRNG state
      phase: Phase,
      activePlayerId: PlayerId,
      turnNumber: number,             // increments on each turn START, starting at 1
      players: { player: PlayerState, cpu: PlayerState },
      pendingAction: PendingAction | null,
      log: GameEvent[],
      winner: PlayerId | null,
      config: MatchConfig
    }

    PlayerId := "player" | "cpu"

    Phase :=
      | "StartTurn" | "Draw" | "MainPhase"
      | "TargetSelection" | "BattlePhase"
      | "ResolveEffects" | "EndTurn" | "GameOver"

    PlayerState := {
      id: PlayerId,
      health: number,                 // integer, clamped to [0, maxHealth]
      maxHealth: number,              // default 20
      deck: CardInstance[],
      hand: CardInstance[],
      discard: CardInstance[],
      activeUnit: UnitInstance | null,
      currentResource: number,        // integer, in [0, maxResource]
      maxResource: number,            // integer, in [0, resourceCap]
      resourceCap: number             // default 6
    }

    CardInstance := {
      instanceId: string,             // unique per match
      cardId: string,                 // refers to card definition
      ownerId: PlayerId
    }

    UnitInstance := {
      instanceId: string,
      cardId: string,
      ownerId: PlayerId,
      maxHealth: number,
      currentHealth: number,
      baseAttack: number,
      attackModifiers: AttackModifier[],
      abilities: Ability[],
      hasAttackedThisTurn: boolean
    }

    AttackModifier := { amount: number, expiresEndOfTurn: boolean }

    Ability := { id: string, params: object }

    MatchConfig := {
      startingHealth: number,         // default 20
      startingHandSize: number,       // default 3
      resourceCap: number,            // default 6
      activeUnitReplaceRule: "discard_previous" | "block_new",
      deckOutCausesLoss: boolean,     // default true
      maxHandSize: number | null,     // default null (unlimited)
      firstPlayer: PlayerId           // default "player"
    }

    PendingAction := {
      kind: "play_card_needs_target",
      playerId: PlayerId,
      cardInstanceId: string,
      validTargets: TargetRef[]
    }

    TargetRef :=
      | { kind: "unit", playerId: PlayerId, instanceId: string }
      | { kind: "player", playerId: PlayerId }

## Reference cards

Card definitions live in a static catalog. The six MVP cards:

    {
      "unit_striker": {
        "baseName": "Striker", "type": "unit", "cost": 1,
        "maxHealth": 3, "attack": 2, "abilities": []
      },
      "unit_guardian": {
        "baseName": "Guardian", "type": "unit", "cost": 2,
        "maxHealth": 5, "attack": 1, "abilities": []
      },
      "unit_charger": {
        "baseName": "Charger", "type": "unit", "cost": 3,
        "maxHealth": 4, "attack": 3,
        "abilities": [
          { "id": "bonus_damage_on_direct_attack", "params": { "amount": 1 } }
        ]
      },
      "action_spark": {
        "baseName": "Spark", "type": "action", "cost": 1,
        "effects": [
          { "type": "deal_damage", "amount": 2, "target": "enemy_unit" }
        ]
      },
      "action_mend": {
        "baseName": "Mend", "type": "action", "cost": 1,
        "effects": [
          { "type": "heal", "amount": 2, "target": "friendly_active_unit", "capAtMax": true }
        ]
      },
      "action_surge": {
        "baseName": "Surge", "type": "action", "cost": 2,
        "effects": [
          { "type": "modify_attack", "amount": 2, "target": "friendly_active_unit", "duration": "until_end_of_turn" }
        ]
      }
    }

## Default deck

Two copies of each reference card, twelve cards total per player. The deck composition is a MECHANICS variation point.

## Operations

Each operation lists Inputs, Outputs, Rules, Edge Cases, Errors. Operations are referenced from `tests.yaml` by operation id.

### match.init

**Inputs:** `config: MatchConfig`, `seed: number`, `deckP: cardId[]`, `deckC: cardId[]`.

**Outputs:** `GameState` with both decks shuffled (seeded Fisher-Yates over xorshift32), opening hands NOT yet dealt, `phase = "StartTurn"`, `activePlayerId = config.firstPlayer`, `turnNumber = 0`, both players at `currentResource = 0`, `maxResource = 0`, `health = config.startingHealth`, `activeUnit = null`.

**Rules:**
- Generate stable `instanceId` for each card. Use deterministic scheme: `"i" + index + "p"|"c"` (index in original deck order).
- Shuffle each deck with Fisher-Yates using the shared PRNG; the player deck shuffles first, then the CPU deck. RNG state advances accordingly.
- `matchId` is derived from `seed`: `"m" + seed.toString(16)`.

**Edge cases:**
- Empty deck input: returns state with `winner = opponentOf(emptyDeckOwner)` if `deckOutCausesLoss` is true and the empty deck belongs to the first player to draw. Otherwise sets winner only when the draw operation triggers it.
- `startingHandSize > deck.length`: caps opening hand at `deck.length`. Does NOT trigger deck-out at init.

**Errors:** `config.startingHealth < 1`, `config.resourceCap < 1`, unknown `cardId` in deck.

### match.dealOpeningHands

**Inputs:** `state` from `match.init`.

**Outputs:** `{ state, events }` where each player has drawn `config.startingHandSize` cards (or fewer if the deck is short), `phase` remains `"StartTurn"`.

**Rules:**
- Draws alternate one card at a time, starting with `activePlayerId`, then opponent, repeat. This makes hand composition stable.
- Emits one `CARD_DRAWN` event per draw, with `revealed: true` for the player and `revealed: false` for the CPU.

**Edge cases:** Empty deck mid-deal: stop dealing for that player, continue for the other. Do not trigger deck-out.

### turn.startPhase

**Inputs:** `state` where `phase = "StartTurn"`.

**Outputs:** `{ state, events }`. `state.turnNumber` increments by 1. Active player's `maxResource = min(maxResource + 1, resourceCap)`. Active player's `currentResource = maxResource`. Active player's `activeUnit.hasAttackedThisTurn = false` if present. Phase advances to `"Draw"`.

**Rules:**
- Only the active player's resource changes.
- Start-of-turn ability triggers (none in MVP) would fire here.
- Emits `TURN_STARTED { playerId, turnNumber }` and `RESOURCE_CHANGED { playerId, current, max }`.

**Errors:** Phase is not `"StartTurn"`.

### turn.draw

**Inputs:** `state` where `phase = "Draw"`, `playerId` (must equal `activePlayerId`).

**Outputs:** `{ state, events }`. One card moves from top of deck to hand. Phase advances to `"MainPhase"`.

**Rules:**
- Top of deck is `deck[0]`. Removing it shifts the array.
- Emits `CARD_DRAWN { playerId, instanceId, revealed }`.
- If `maxHandSize` is set and hand is at cap, the drawn card is discarded immediately (emits `CARD_DRAWN` then `CARD_DISCARDED { reason: "hand_full" }`).

**Edge cases:**
- Empty deck: if `deckOutCausesLoss` is true, set `winner = opponentOf(playerId)`, set `phase = "GameOver"`, emit `GAME_ENDED { winner, reason: "deck_out" }`. If false, just skip the draw with `CARD_DRAWN_SKIPPED`.

**Errors:** Phase is not `"Draw"`. `playerId !== activePlayerId`.

### card.canPlay

**Inputs:** `state`, `playerId`, `cardInstanceId`.

**Outputs:** `{ playable: boolean, reason?: string, requiresTarget: boolean, validTargets: TargetRef[] }`.

**Rules:**
- Pure query. No state mutation.
- `playable` is true iff:
  1. `playerId === activePlayerId`
  2. `phase === "MainPhase"`
  3. The card exists in `playerId`'s hand
  4. `currentResource >= card.cost`
  5. If the card's effects require a target, at least one valid target exists
  6. For unit cards: if `activeUnitReplaceRule === "block_new"`, the active slot must be empty
- For unit cards, `requiresTarget` is false; placement is automatic.
- For action cards, `requiresTarget` is true iff any effect references a target that does not auto-resolve (`enemy_unit`, `friendly_active_unit` resolve automatically when unambiguous; `any_unit` requires selection).

**Reference card target resolution:**
- Spark: target = `enemy_unit`. Auto-resolves if opponent has exactly one active unit. Otherwise no valid target and the card is unplayable (single active unit slot in MVP means it's always auto-resolved when present, unplayable when absent).
- Mend: target = `friendly_active_unit`. Auto-resolves if you have an active unit. Unplayable otherwise.
- Surge: same as Mend.

### card.play

**Inputs:** `state`, `playerId`, `cardInstanceId`, `target?: TargetRef`.

**Outputs:** `{ state, events }`.

**Rules:**
1. Validate via `card.canPlay`. If not playable, throw.
2. Deduct `currentResource -= card.cost`. Emit `RESOURCE_CHANGED`.
3. Remove card from hand. Emit `CARD_PLAYED { playerId, cardId, instanceId }`.
4. If card is a unit:
   - If `activeUnit` is null OR `activeUnitReplaceRule === "discard_previous"`: place new unit as `activeUnit`. If replacing, move previous to discard first; emit `UNIT_REPLACED` and `UNIT_DISCARDED`.
   - Initialize `currentHealth = maxHealth`, `hasAttackedThisTurn = false`.
   - Emit `UNIT_SUMMONED { playerId, unit }`.
5. If card is an action:
   - For each effect, call `effect.apply` in declaration order.
   - After all effects resolve, move card to discard. Emit `CARD_DISCARDED { reason: "action_resolved" }`.
6. After resolution, run `win.check`. If `winner !== null`, set `phase = "GameOver"`.

**Errors:** Unplayable card, missing target when required, invalid target.

### effect.apply

**Inputs:** `state`, `effect`, `sourceRef`, `targetRef`.

**Outputs:** `{ state, events }`.

**Supported effect types (MVP):**

#### `deal_damage`

Reduce target's `currentHealth` (unit) or `health` (player) by `effect.amount`. Floor at 0. Emit `DAMAGE_DEALT { sourceRef, targetRef, amount }`. If target is a unit reduced to 0 HP, call the unit-defeat sub-routine (move to discard, clear `activeUnit`, emit `UNIT_DEFEATED`). If target is a player reduced to 0 HP, win check fires after.

#### `heal`

Increase target unit's `currentHealth` by `effect.amount`. If `effect.capAtMax` is true (always true for Mend), cap at `maxHealth`. Emit `HEALTH_CHANGED { targetRef, before, after }`. Healing a unit at full HP emits the event with `before === after` and no other change.

#### `modify_attack`

Push `{ amount: effect.amount, expiresEndOfTurn: (effect.duration === "until_end_of_turn") }` onto target unit's `attackModifiers`. Emit `ATTACK_MODIFIED { targetRef, amount, duration }`.

#### `draw_card` (optional in MVP; honor if a card declares it)

Move top of `playerId`'s deck to hand. Identical semantics to `turn.draw` regarding deck-out.

#### `summon_or_replace_unit` (reserved for future cards; not used by the six MVP cards)

**Effective attack of a unit** = `baseAttack + sum(attackModifiers[*].amount)`. Always recomputed at attack time, never stored.

**Errors:** Unknown effect type. Target type does not match effect target spec.

### combat.canAttack

**Inputs:** `state`, `playerId`.

**Outputs:** `{ canAttack: boolean, reason?: string, defaultTarget?: TargetRef }`.

**Rules:**
- `canAttack` is true iff `playerId === activePlayerId`, `phase` is `"MainPhase"` or `"BattlePhase"`, `activeUnit` exists, and `activeUnit.hasAttackedThisTurn === false`.
- `defaultTarget` is the opponent's `activeUnit` if present, otherwise the opponent player.

### combat.attack

**Inputs:** `state`, `playerId`, optional `targetRef` (overrides default).

**Outputs:** `{ state, events }`.

**Rules:**
1. Validate via `combat.canAttack`. Throw if not allowed.
2. Compute `effectiveAttack` of attacker.
3. If `targetRef.kind === "unit"`:
   - Deal `effectiveAttack` damage to target unit.
   - If target unit's `currentHealth` reaches 0, run unit-defeat sub-routine.
   - **No counter-attack in MVP.** The defending unit does not deal damage back. (Variation point in MECHANICS.md.)
4. If `targetRef.kind === "player"`:
   - Compute bonus damage: for each ability on attacker matching `bonus_damage_on_direct_attack`, add `params.amount`.
   - Deal `effectiveAttack + bonus` damage to target player.
5. Set `attacker.hasAttackedThisTurn = true`.
6. Emit `UNIT_ATTACKED { attackerRef, targetRef, damage, bonus }`.
7. Run `win.check`.

**Edge cases:**
- Target is a unit but opponent's `activeUnit` was just defeated by an effect this turn: target is invalid; reject before damage.
- Surge buff makes effective attack 0 or negative: damage is clamped to 0 (no healing via negative attack).

**Errors:** No active unit. Already attacked. Wrong phase.

### turn.endPhase

**Inputs:** `state` where `phase` is `"MainPhase"` or `"BattlePhase"`.

**Outputs:** `{ state, events }`.

**Rules:**
1. End-of-turn ability triggers (none in MVP) would fire here.
2. For each unit on the board (both players' `activeUnit`), remove all `attackModifiers` where `expiresEndOfTurn === true`. Emit `ATTACK_MODIFIER_EXPIRED` for each removed.
3. Reset `hasAttackedThisTurn = false` on both players' active units.
4. Swap `activePlayerId`.
5. Set `phase = "StartTurn"`.
6. Emit `TURN_ENDED { previousActivePlayerId, nextActivePlayerId }`.

**Errors:** Phase is not Main or Battle.

### win.check

**Inputs:** `state`.

**Outputs:** `{ winner: PlayerId | null, reason?: "health_zero" | "deck_out" }`.

**Rules:**
- Pure query. Returns the first matching condition:
  1. If `players.player.health <= 0` and `players.cpu.health <= 0`, the player who hit 0 LAST wins. If both hit 0 in the same action, the non-active player wins (the attacker who pushed both to 0 is the active player; they win, not lose; for the simultaneous-effect case, active player wins). This is a tie-break, not a common case.
  2. If `players.cpu.health <= 0`, winner = `"player"`.
  3. If `players.player.health <= 0`, winner = `"cpu"`.
  4. Otherwise null.
- If `state.winner` is already set, this query returns that without re-evaluating.

### cpu.chooseActions

**Inputs:** `state` where `activePlayerId === "cpu"`, `phase === "MainPhase"`.

**Outputs:** `Action[]`: an ordered sequence of actions the CPU intends to take this turn, ending with an `END_TURN` action.

**Action :=**
- `{ kind: "play_card", instanceId, target? }`
- `{ kind: "attack", target? }`
- `{ kind: "end_turn" }`

**Decision priority (in order; pick the first that applies, repeat until END_TURN):**

1. **Threat removal:** If a `deal_damage` action card in hand can reduce an opposing unit to 0 HP, play it.
2. **Lethal:** If the player's HP is `<= effectiveAttack + applicable bonuses`, attack the player. If a sequence of (buff, attack) achieves lethal, do that sequence.
3. **Heal:** If active unit's `currentHealth <= maxHealth / 2` (integer division) AND a `heal` action is in hand AND CPU can afford it, play it.
4. **Establish board:** If no active unit, play the highest-cost affordable unit card.
5. **Buff and attack:** If CPU can attack this turn AND a `modify_attack` action is in hand AND the buffed attack would either kill the opposing active unit OR exceed it (when no opposing unit exists, any positive buff is "useful"), play the buff then attack.
6. **Default attack:** If CPU has an active unit that has not attacked, attack with default target.
7. **Play highest-cost affordable useful card:** Where "useful" means a target exists for action cards; a unit card is always useful unless the active slot is occupied and replacement is blocked.
8. **End turn:** Otherwise, end turn.

**Tie-breaks:**
- Among equal-priority cards, prefer lowest `instanceId` lexicographically. Deterministic.
- Among multiple valid targets, prefer the target with lowest `currentHealth` (for damage) or highest `maxHealth - currentHealth` (for heal).

**Edge cases:** CPU has no playable cards and no legal attack: returns `[{ kind: "end_turn" }]`.

**Errors:** None. The function must always return at least one action ending in `end_turn`.

### theme.resolve

**Inputs:** `cardId`, `themePack`, optional `locale`.

**Outputs:** `{ displayName, artKey, flavorText, terminology }` where every field falls back through: themePack[locale] -> themePack["default"] -> card baseName.

**Rules:**
- Pure. No state mutation. No I/O for art loading; only returns keys.
- See THEME.md for the full theme pack contract.

## Cross-operation invariants

The following must hold at every state transition. Implementations must add assertions, at least in dev builds.

- **INV-HP-NONNEGATIVE:** `players.*.health >= 0` and `players.*.activeUnit?.currentHealth >= 0`.
- **INV-HP-CAPPED:** `players.*.health <= players.*.maxHealth` and `unit.currentHealth <= unit.maxHealth`.
- **INV-RESOURCE-BOUNDS:** `0 <= currentResource <= maxResource <= resourceCap`.
- **INV-RESOURCE-CAP-MONOTONIC:** Per player, `maxResource` is non-decreasing within a match until it reaches `resourceCap`.
- **INV-TURN-MONOTONIC:** `turnNumber` is non-decreasing.
- **INV-ATTACK-ONCE:** Once a unit's `hasAttackedThisTurn` is true, it remains true until the next `turn.endPhase` involving it.
- **INV-BUFF-EXPIRE:** No `attackModifier` with `expiresEndOfTurn = true` survives a `turn.endPhase` call.
- **INV-DETERMINISM:** Given identical `(state, action)`, two implementations of this spec produce identical `(state', events)` ignoring event timestamps. Event order within a single operation is fixed by the spec.
- **INV-WIN-TERMINAL:** If `winner !== null`, all subsequent operations return `state` unchanged with an empty events array. No further mutations.
- **INV-CARD-CONSERVATION:** For each player, the total count of cards across `deck + hand + discard + (activeUnit ? 1 : 0)` is constant after `match.dealOpeningHands` completes, except when a `draw_card` effect targets the opposing player (none in MVP) or `deckOutCausesLoss` triggers.

## Termination

A match ends when `win.check` returns a non-null winner. The terminal phase is `"GameOver"`. No restart operation is part of this spec; restart is the consumer's responsibility (create a new `match.init`).

## Determinism contract

- All randomness flows through `state.rngState` via the xorshift32 PRNG in Appendix A.
- Card draws read the top of the (already shuffled) deck. No mid-match shuffles in MVP.
- CPU choice is deterministic per priority rules above.
- Event order within an operation is specified per operation. Implementations must emit events in that order.
- Timestamps are not part of the event contract. Event `kind` and payload fields are.

## Event contract

Events have shape `{ kind: string, ...payload }`. The payload schema per kind:

    CARD_DRAWN          { playerId, instanceId, revealed }
    CARD_PLAYED         { playerId, cardId, instanceId }
    CARD_DISCARDED      { playerId, instanceId, reason }
    UNIT_SUMMONED       { playerId, unit }
    UNIT_REPLACED       { playerId, previousInstanceId, newInstanceId }
    UNIT_ATTACKED       { attackerRef, targetRef, damage, bonus }
    UNIT_DEFEATED       { ref }
    DAMAGE_DEALT        { sourceRef, targetRef, amount }
    HEALTH_CHANGED      { targetRef, before, after }
    ATTACK_MODIFIED     { targetRef, amount, duration }
    ATTACK_MODIFIER_EXPIRED { targetRef, amount }
    RESOURCE_CHANGED    { playerId, current, max }
    TURN_STARTED        { playerId, turnNumber }
    TURN_ENDED          { previousActivePlayerId, nextActivePlayerId }
    GAME_ENDED          { winner, reason }

Unknown event kinds emitted by extensions are not part of this spec but must not collide with the names above.

## Variation point indices

- **STACK** profiles: see STACK.md
- **DESIGN** profiles: see DESIGN.md
- **MECHANICS** rule variations: see MECHANICS.md
- **THEME** pack contract: see THEME.md

## Appendix A: xorshift32 reference

    function xorshift32(state: number): { value: number, state: number } {
      let x = state | 0
      x ^= x << 13
      x ^= x >>> 17
      x ^= x << 5
      return { value: (x >>> 0), state: (x | 0) }
    }

    function nextInt(state: number, maxExclusive: number): { value: number, state: number } {
      const { value, state: s2 } = xorshift32(state)
      return { value: value % maxExclusive, state: s2 }
    }

    function shuffle<T>(arr: T[], state: number): { arr: T[], state: number } {
      const a = arr.slice()
      let s = state
      for (let i = a.length - 1; i > 0; i--) {
        const r = nextInt(s, i + 1)
        s = r.state
        const j = r.value
        const tmp = a[i]
        a[i] = a[j]
        a[j] = tmp
      }
      return { arr: a, state: s }
    }

This is a reference, not a mandate. Any seedable PRNG that produces identical sequences from identical seeds is acceptable, but `tests.yaml` golden traces were generated against xorshift32 with the above Fisher-Yates. Different PRNGs will fail the golden traces. Use this one for the MVP.
