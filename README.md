# card-battler-ghost

A **ghost library** for a theme-agnostic single-player browser card battler. Spec-only. No code. Hand the contents of this directory to a coding agent and it produces a playable game.

## What is a ghost library?

A ghost library is the abstract idea of an open-source package without the implementation. It contains:

- A specification of behavior (this repo: `SPEC.md`)
- An executable test suite that any implementation must pass (`tests.yaml`)
- Variation point documents that let consumers customize stack, design, and rules without forking (`STACK.md`, `DESIGN.md`, `MECHANICS.md`, `THEME.md`)
- A consumer-facing prompt for generating implementations (`INSTALL.md`)
- Verification provenance (`VERIFY.md`)
- Worked examples (`examples/`)

When you "install" a ghost library, you do not fetch its code. You hand the spec to an agent, which writes the code fresh into your project, tailored to your stack and design choices. The agent's output is yours: you own it, edit it, and ship it.

## Prior art and attribution

The ghost-library pattern was originated by **Drew Breunig** (`dbreunig`) with [`whenwords`](https://github.com/dbreunig/whenwords), a relative-time formatting library distributed as a SPEC.md + tests.yaml + INSTALL.md with no code. Drew introduced the approach in his post [*A Software Library with No Code*](https://www.dbreunig.com/2026/01/08/a-software-library-with-no-code.html) and uses the term "ghost library" in [whenwords/INSTALL.md](https://github.com/dbreunig/whenwords/blob/main/INSTALL.md). Example downstream implementations live at [`dbreunig/whenwords-examples`](https://github.com/dbreunig/whenwords-examples).

The pattern was generalized into a reusable Codex skill by **Tim Kersey** (`tkersey`) at [`tkersey/dotfiles/codex/skills/ghost`](https://github.com/tkersey/dotfiles/tree/main/codex/skills/ghost). Tim's `$ghost` skill takes any input project description and produces a ghost-library directory in the whenwords shape.

This library follows that shape and credits both. Bugs in `card-battler-ghost` are mine; the model is theirs.

## What is in this directory

| File | Purpose |
|------|---------|
| `SPEC.md` | The behavior contract. State schema, operation signatures, invariants. Source of truth. |
| `tests.yaml` | 38 language-agnostic test cases plus 1 full-match scenario. The executable contract. |
| `STACK.md` | Renderer / input / state-store profiles (vanilla, React, Svelte, Phaser, Pixi, Three). |
| `DESIGN.md` | Visual profiles (clean-modern, pixel, neon, hand-drawn, minimal). |
| `MECHANICS.md` | Rule variation points (counter-attack, replace rules, deck-out behavior, etc). |
| `THEME.md` | Theme pack contract. Theme is data; rules never read it. |
| `INSTALL.md` | The paste-prompt you hand to a coding agent. |
| `VERIFY.md` | What "passing" means, test adapter recipe, coverage table, limitations. |
| `examples/sample-match-trace.yaml` | Golden deterministic match trace from `seed=42`. |
| `examples/theme-pack-example.yaml` | "Forest Friends" sample theme pack. |
| `LICENSE` | MIT. |

## Quickstart

Read `INSTALL.md`. Pick a stack, a design, and a theme. Paste the prompt template into Claude Code, Codex CLI, Cursor, or any capable coding agent. The agent reads the spec, writes the code, and ships a playable game.

## Provenance

- Upstream GDD: Theme-Agnostic Card Battler MVP, version `gdd-2026-05-23`
- Ghost library built: hand-authored on 2026-05-23
- Intended use: run the same configuration through two different agents and compare outputs. The implementations should differ visually and stylistically but pass every case in `tests.yaml` identically.

## A note on scope

This ghost library covers the MVP only. Out of scope: multiplayer, accounts, persistence, deck builder, ranked play, card packs. See GDD section 5 for the full inclusion/exclusion list.

The spec is intentionally tight enough that two agents will produce convergent implementations of the rules engine, and loose enough that they will produce divergent (interesting) implementations of the UI.

## Caveats

This is the first ghost library produced for game logic, not infrastructure. Some failure modes documented in `VERIFY.md` are specific to game-as-spec rather than library-as-spec. Read those before assuming the model generalizes.
