# Hive Warm v2 — "Living Apiary" enrichment design

Date: 2026-07-30
Status: approved by Master's standing instruction ("сделай прилу более стильной и богатой... сделай это"); interactive gates skipped per that instruction.
Version stays **1.0 (build 1)** per pipeline rule.

## Goal

Take the delivered Hive Warm colony sim from "works and is educational" to "feels alive,
rich, and hand-crafted". Fix the one visibly weak spot (a flat vector hive floating over
painted art), add depth to the daily loop, and give every screen a style pass — without
bloating scope or touching the iOS 15.6 / custom-UI constraints.

## Weaknesses being addressed

1. Apiary header: static backdrop + flat rectangle hive; nothing moves, nothing reacts.
2. Dashboard is a wall of identical bars; no single glanceable "how are we doing".
3. Journal starts empty and stays dry (numbers only, no keepsakes).
4. Little charm between actions: no celebration on harvest, queen is anonymous,
   day-to-day play has no ambient flavor.
5. Cards/buttons are flat single-color; screens read as "plain reference app".

## Approaches considered

- **A. Pure cosmetic re-skin** (gradients, shadows, textures only). Cheap, but the apiary
  stays dead and the loop stays dry — doesn't meet "richer and fuller".
- **B. Living-apiary + engagement layer + style pass (chosen).** Animated scene, one
  composite wellbeing dial, forecast planning, keepsake postcards, named queens,
  celebrations, plus the re-skin. Real richness, bounded scope, no new sim mechanics to
  balance.
- **C. Deep sim expansion** (multiple hives, breeding genetics, market economy). Highest
  ceiling but re-balances the whole engine, multiplies UI, and risks bugs before review.
  Rejected as inappropriate for a polish pass.

## Design (approach B)

### 1. Living apiary scene (`ApiaryScene.swift`, new)

Replaces the backdrop block in `ApiaryView`:

- Painted seasonal backdrop stays; a weather tint layer (rain slate, cold blue,
  heat amber, overcast gray) sits above it.
- `TimelineView(.animation)` + `Canvas` draws ambient bees orbiting the hive on
  per-bee elliptical paths (count scales with colony size, 0 in rain/cold/winter —
  the educational truth that bees don't fly in bad weather).
- Same canvas renders weather particles: falling rain streaks, drifting snow.
- The vector hive is redrawn (`HiveStackView` v2): wood-grain gradient boxes with
  plank lines, roof overhang, landing board with crawling bee dots, burlap wrap band
  when insulated, snow cap in winter.
- Header chips: season/day glass pill, today's weather, and a **tomorrow forecast**
  chip (engine weather is seeded → `rollWeather(day+1)` is honest).
- Tapping the hive: soft haptic + popover card with a mood line derived from state
  ("The hum is deep and content..." / "The hive sounds thin and anxious...").

### 2. Wellbeing dial + keeper's wisdom (`HiveComponents` + `WisdomVault.swift`, new)

- Composite **Colony wellbeing** ring on the dashboard: weighted blend of population,
  stores-vs-season-need, mite pressure (inverted), queen vitality. Color arc
  red→amber→green, animated needle, one word verdict (Thriving / Steady / Strained /
  Failing). The six detail bars stay below it.
- **Keeper's wisdom** card: 40 short craft tips, rotated deterministically by day.
- **Named queens**: every queen gets a pastoral name (Willow, Clover, Marigold...).
  Shown in the vitals row ("Queen Willow"), in the inspection reveal toast, and
  renamed on requeen/swarm succession.

### 3. Journal keepsakes (`JournalView`, models)

- **Season postcards**: at each season end the store records {season, year, jars,
  bees, honey, flavor line}; Journal renders them as horizontal art postcards
  (season art thumbnail + stats). Kept to the last 12.
- **Year wheel**: four-arc season ring with a marker at today's position, year number
  in the center.
- Badge tiles restyled: gradient hex tiles, shine stroke when earned.

### 4. Charm & feedback

- Harvest: 1.2 s full-screen celebration (falling honey drops on Canvas, jar count
  springs in) before the sheet closes.
- Market: coin pill bounces on sale.
- All gauge fills animate on change; tab icons get a spring + active honey pill.

### 5. Global style pass

- `HiveBackground`: cream base + faint hex lattice (Canvas, ~3% opacity) behind
  every tab.
- Cards: warm two-stop gradient fill, hairline wax border, softer shadow.
- Primary buttons: honey gradient with pressed scale.
- Screen headers get a small comb accent; almanac cards get a "N min read" chip.

### Persistence & compatibility

New stored fields are **optionals** (`queenName: String?` on `ColonyState`,
`postcards: [SeasonPostcard]?` in the save blob) so existing saves decode unchanged
(missing keys → nil → defaults applied at use sites). No version bump (stays 1.0/1).

### Out of scope

Multiple hives, genetics, audio, notifications, any iOS 16+ API, art-pack regeneration
(existing 38-piece pack is reused; postcards reuse season art).

### Verification

Clean `xcodebuild` (zero errors/warnings), simctl walkthrough of every screen incl.
new scene in all four seasons via the established env-hook workflow, refreshed
screenshots, delivery checklist re-run (no `key=`, no SF Symbols/emoji, icon alpha,
orientations, manual signing).
