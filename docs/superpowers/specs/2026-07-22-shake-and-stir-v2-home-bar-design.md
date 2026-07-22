# Shake and Stir v2 — Home Bar Edition (design spec)

Date: 2026-07-22
App: Shake and Stir (`com.coppershaker.shakeandstir`), delivered v1.0 → rework in `development/`
Goal: turn the app from "pretty reference + minigame" into a genuinely useful home-bar
assistant, and make it visually richer (living bar scene + new art pack). Ships as
v1.0 (build 1) per versioning rule.

## Problem

v1 outside the mix session reads like a catalog: static cards, text, badges. Nothing
answers the user's actual question on a Friday night: **"what can I make right now with
what I have?"** — that is the job a real user keeps the app for.

## Decisions (confirmed with Master)

1. Direction: **home bar assistant** (utility first; the motion mix game stays as the
   craft/learning layer).
2. Feature set: **full** — My Bar shelf, "what can I mix", "buy one bottle → +N recipes",
   shopping list, occasion picker.
3. Style: **both** — living animated bar scene AND a new generated art pack for
   ingredients/bottles.
4. Navigation: **5th tab My Bar** (Tonight · Menu · My Bar · Bar Book · More);
   Tonight rebuilt around mixability; Menu gets stock filter + readiness badges.

## 1. Data: ingredient catalog

New `IngredientLibrary.swift`:

- ~46 canonical ingredients, each `Ingredient { id, name, group, tokenArt }`.
- 5 groups: **Spirits** (bourbon, white rum, dark rum, tequila, gin, vodka, brandy,
  whiskey…), **Liqueurs & Wine** (orange liqueur, coffee liqueur, cream liqueur,
  campari, sweet vermouth, red wine, crème de menthe/cacao…), **Mixers** (soda water,
  tonic, cola, ginger beer/ale, orange/pineapple/cranberry juice, lemonade, coconut
  cream, espresso, tea, grenadine…), **Fresh** (lime, lemon, orange, mint, cucumber,
  berries, cherry, pineapple…), **Pantry** (sugar, salt, honey, cinnamon, nutmeg,
  cream, milk, egg, bitters, ice…).
- Every drink in `DrinkLibrary` gets `ingredientIDs: [String]` mapped from its existing
  "You will need" list. Display strings unchanged. **Every recipe line maps to an id**
  (garnish included); ice is excluded from need-math (assumed always available) but
  shown on the shelf as flavor.

`MixStore` additions (tolerant decoding, v1 saves must load):

- `ownedIngredients: Set<String>` (default empty)
- `shoppingList: Set<String>` (default empty)
- mutations: `toggleOwned(_:)`, `addToShopping(_:)`, `removeFromShopping(_:)`,
  `purchaseFromShopping(_:)` (moves item shopping → owned), all `save()`.

`BarMath.swift` — pure functions, unit-testable by review:

- `readiness(drink, owned) -> .ready | .almost(missing: id) | .missing(count)`
- `mixableCount(owned)`, `almostList(owned)`
- `bestBuys(owned, limit: 3) -> [(ingredientID, unlockCount)]` — for each missing
  ingredient, how many drinks flip to ready if bought; sorted desc, ties by name.
- `coverage(owned) -> Double` overall and per drink category.

## 2. My Bar tab (new, index 2)

- **Shelf**: sections per ingredient group rendered as wooden shelves (tiled wood
  texture + brass edge line). Items are generated art tokens (distinct bottle
  silhouettes/labels per spirit & liqueur; fruit/jar art for fresh/pantry). Owned =
  full color + sheen; not owned = dark 35 %-opacity outline version (same PNG, dimmed
  + desaturated via overlay — no second asset). Tap toggles owned (haptic tap + soft
  clink sfx reuse `sfx_ice`).
- **Header**: "You can mix **N** drinks tonight" (live), coverage ring, top advice
  chip "Buy lime → +4 drinks" (from `bestBuys`, hidden when bar empty or complete).
- **Search field** filters tokens across groups.
- **Shopping list card** (below header, above shelves; also reachable from advice
  chip): auto-suggest rows from `bestBuys` (with "+N drinks" labels), user's own
  items, tap row circle = bought → moves to owned with sparkle; swipe-delete;
  "Suggested" vs "My list" sections.
- Empty state: friendly copy + "Start with the basics" button that pre-selects a
  starter set (vodka, white rum, lime, sugar, soda water, orange juice) — undoable
  by tapping tokens off.

## 3. Tonight rebuild

Order top → bottom:

1. Greeting (unchanged) over **live bar scene hero** (see §5) instead of static card.
   Special of the Day sits on the scene with its portrait medallion.
2. **Ready to mix** rail — drinks with `.ready`, sorted by (not yet mastered first,
   then name). Empty state links to My Bar ("Stock your shelf to see what you can
   pour").
3. **So close** rail — `.almost`, card shows "just add lime" chip; tapping chip adds
   the item to the shopping list (toast confirm).
4. **Occasions** — 6 chips: Date Night, Party Crowd, Zero-Proof Guest, Quick & Easy,
   Cozy Night, Sunny Afternoon. Each opens `OccasionView`: generated cover art
   header, curated drink list (static per-occasion drink id lists chosen editorially
   in `OccasionLibrary.swift`) ranked ready-first with readiness badges.
5. Category rails (kept from v1, now with readiness badges on cards).

## 4. Menu + detail upgrades

- Menu cards: readiness badge — brass "Ready" pip, amber "+1" (one missing), dim none.
- New filter chip **In Stock** (only `.ready`) next to All/Favorites/Mixed/Zero Proof.
- Drink detail: "You will need" rows become interactive — owned rows get brass check,
  missing rows show hollow circle + "Add to list" mini-button (or "In list ✓" state);
  a one-line summary above ("You have 4 of 6 — missing lime, mint").
- Mix session unchanged (may be launched regardless of stock — it's practice).

## 5. Living bar scene (`BarSceneView.swift`)

Single Canvas + TimelineView engine (pattern proven in Sound Grove LiveScene):

- Layers: back-bar silhouette with bottle row (subtle specular blinks), warm lamp
  glow breathing (2 radial pulses, ~7 s period), slow dust motes drifting in a light
  cone (~24 particles), faint smoke wisp (2 blurred sin-curves), occasional star
  sparkle near glassware, thin neon underline flicker on the shelf edge.
- Modes: `.full` (Tonight hero, ~160 pt tall), `.ambient` (My Bar background,
  opacity 0.35, fewer particles), `.intro` (mix-session intro card, light).
- Reduce Motion → single static frame (no TimelineView).
- Particle cap ≤ 60; engine only lives inside visible views (SwiftUI lifecycle).

## 6. Art pack v2 (`art_src/artgen2.swift`, deterministic CG)

- 46 ingredient tokens 512×512 with transparent backgrounds (they layer on shelf
  wood; the opaque-no-alpha rule applies only to the app icon).
  Distinct silhouettes: shouldered bourbon bottle, tall gin, round brandy snifter
  bottle, cane-sugar rum, agave tequila, jars, fruit forms; each with group-tinted
  label + tiny motif.
- 6 occasion covers 1200×600 (scene-style: table, glasses, palette per occasion).
- `texture_wood` shelf tile + `scene_backbar` wide backdrop for the hero.
- All join existing `Art/` folder (v1 art untouched). Expected app ~50–60 MB
  (≥18 MB rule, <99 MB cap).

## 7. Gamification hooks (small)

- 2 new badges: **Stocked Shelf** (own 15 ingredients), **Quartermaster** (first
  purchase via shopping list). Badge evaluation extends existing `MixBadge.evaluate`.
- Mixing a `.ready` drink grants +5 bonus XP ("mixed from your own bar").

## 8. Unchanged / constraints

- Mix stages, glass engine, sounds, WebView gate, onboarding (page 3 text updated to
  mention My Bar), iOS 15.6+, iPhone family, Portrait+Landscape, forced dark,
  custom glyphs only (no SF Symbols/emoji), UserDefaults JSON only.
- Version stays **1.0 (build 1)**.
- Rework happens in `development/Shake and Stir` (moved back from
  `for_human_review_apps/`), re-delivered on completion.

## Error handling

- Missing art token → live vector fallback (rounded-rect bottle silhouette via
  existing Canvas glyph system) — same pattern as v1 `MixArt` fallback.
- Corrupt/absent saved state → defaults (existing tolerant decoder covers new keys).
- `bestBuys` with empty owned set → suggests the starter pack instead of noise.

## Testing

- Compile-clean sim Debug + device Release.
- Logic check: swiftc harness (scratchpad) asserting `readiness`/`bestBuys` on
  fixture bars (empty bar → 0 ready; full catalog → 28 ready; known one-missing case).
- Visual verify on sim: My Bar shelf (owned/unowned/tap), Tonight rails + live scene,
  In Stock filter, detail interactive needs, occasion view, shopping list flow.
