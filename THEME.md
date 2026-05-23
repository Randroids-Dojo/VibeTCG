# THEME Pack Contract

A theme is data, not code. The rules engine never reads from a theme pack. Themes affect display only.

## Theme pack shape

A theme pack is a JSON file (or equivalent in the chosen stack). Every field is optional. Missing fields fall back through:

    themePack[locale] -> themePack["default"] -> SPEC card baseName

## Schema

    ThemePack := {
      id: string,                       // stable identifier
      name: string,                     // human-readable
      author: string,
      version: string,                  // semver

      // Card-level overrides, keyed by cardId from SPEC.md
      cards: {
        [cardId: string]: {
          displayName?: string,
          artKey?: string,              // a logical key; the loader resolves to an asset path
          flavorText?: string
        }
      },

      // Optional terminology overrides
      terminology?: {
        unit?: string,                  // default "Unit"
        action?: string,                // default "Action"
        resource?: string,              // default "Resource"
        health?: string,                // default "Health"
        attack?: string,                // default "Attack"
        discard?: string,               // default "Discard"
        deck?: string,                  // default "Deck"
        hand?: string                   // default "Hand"
      },

      // Optional palette overrides (override DESIGN profile defaults)
      palette?: {
        background?: string,
        cardFace?: string,
        cardBorder?: string,
        accentFriendly?: string,
        accentHostile?: string,
        resource?: string,
        textPrimary?: string,
        textMuted?: string
      },

      // Optional typography
      typography?: {
        fontFamilyBody?: string,
        fontFamilyDisplay?: string
      },

      // Optional asset references
      assets?: {
        backgroundArt?: string,         // path or URL
        cardBack?: string,
        ui?: {
          buttonClick?: string,
          cardPlay?: string,
          damage?: string,
          heal?: string,
          victory?: string,
          defeat?: string
        }
      },

      // Optional locale variants. If present, each locale is a partial ThemePack.
      locales?: {
        [localeCode: string]: PartialThemePack
      }
    }

## Resolver contract

`theme.resolve(cardId, themePack, locale)` returns a `ThemedCardView`:

    ThemedCardView := {
      displayName: string,
      artKey: string,
      flavorText: string,
      terminology: ResolvedTerminology
    }

Resolution order for each field:

1. `themePack.locales[locale].cards[cardId][field]`
2. `themePack.cards[cardId][field]`
3. SPEC `baseName` (for `displayName`); empty string (for `flavorText`); `cardId + ".art"` (for `artKey`).

## Required example theme

Every implementation must ship with the default neutral theme (the names as they appear in SPEC.md) and at least one alternate theme. Per GDD section 6 ("The game is intentionally theme-agnostic. The same rules can later support cute collectible animals, mythical creatures, sci-fi units, robots, monsters, fantasy heroes, or any other skin layered on top.") and GDD acceptance criterion: "The same rules work with generic card names and at least one alternate theme mapping."

See `examples/theme-pack-example.yaml` for a full sample theme pack ("Forest Friends" cute-animals theme).

## Theme switching at runtime

The implementation must support live theme swapping during a match without restarting:

- Themes are pure display. Swapping the active theme re-renders the HUD and cards but does not touch `GameState`.
- The HUD must reflect the new theme within one frame.
- The battle log retains its existing event history; subsequent events render with the new theme.

## Forbidden in theme packs

- Changing card costs, HP, attack, or effect amounts. Those are MECHANICS or new cards, not theme.
- Adding new cards via theme. Theme only renames and reskins existing cards.
- Code or executable content. Themes are pure data.
- Theme-injected rules ("when this theme is active, units have +1 HP"). That is balance, not theme.

## Authoring a theme

1. Pick a thematic frame: cute animals, robots, fantasy, sci-fi, monsters, foods, abstract shapes.
2. For each of the six MVP cards, write a `displayName` and optional `flavorText`.
3. Pick or commission art for each card (or use placeholder `artKey`s and resolve to a placeholder asset).
4. Optionally adjust palette and typography for mood.
5. Optionally add terminology overrides (e.g., "Resource" becomes "Energy", "Unit" becomes "Creature").
6. Save as a JSON file. Drop into the implementation's theme directory. The theme selector picks it up.

## Theme manifest

Implementations should expose a list of installed themes via:

    listThemes() -> { id, name, author }[]

This drives any theme selector UI.
