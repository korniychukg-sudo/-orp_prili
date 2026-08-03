# Snug Fit — Design Spec (2026-08-03)

Replaces Tin Melody. Master's verdict on the music-box composer: no point, no
use — a toy, and a 4.3 rejection risk. Requested instead: **a real game with
hundreds of levels**, held to the standard of Fridgeful (an app that solves a
genuine everyday task).

## Concept

An open container fills the screen — carry-on suitcase, moving box, fridge
shelf, tool drawer, car boot. Beside it lies a pile of oddly shaped things.
Fit **everything** in.

Fitting alone is the base puzzle. The depth comes from rules taken straight
from real life: fragile on top, heavy low and over the axle, liquids away from
electronics, frozen goods touching each other, weight limits, items that may
not lie on their side. Later levels stack these constraints against each other.

The player leaves with a real skill: after a hundred levels they pack a
suitcase measurably better. That is the Fridgeful quality Master asked for —
a game whose rules are worth knowing.

## Why this is not a clone

- Not Water Sort, not Flow, not a block-dropper, not a jigsaw: in a jigsaw every
  piece has exactly one correct home, here you search a combinatorial layout.
- Nothing in the 28 delivered apps packs or fits. Marble Grove is physics
  construction, Puzzle Porch is jigsaw, Drop Test is prediction.
- The swap test: strip the packing rules and nothing remains — the theme is not
  a skin over a generic mechanic.

## Core loop

1. Pick an item from the tray (tap or drag).
2. Rotate it in 90° steps (rotate button, or two-finger tap).
3. Drop it on the grid; it snaps to cells. Illegal placements are refused with a
   clear reason ("fragile — nothing may sit on top").
4. Everything placed and every rule satisfied → level solved, stars awarded.

Undo is unlimited. A hint places one item from the stored solution.

## Model

**Grid, not physics.** The container is a `W × H` cell grid with blocked cells
for irregular shapes (a wheel arch in the boot, the fridge compressor bump).
Items are polyominoes of 2–6 cells. This is deliberate: a discrete model is
exactly solvable, so every shipped level can be **proved** solvable offline —
essential, because interactive playtesting on the simulator is not available.

```
Container: id, name, W, H, blockedCells, artFile, family
Item:      id, name, cells[[Int]], rotatable, weight, tags, artFile
Level:     id, chapter, container, items[], constraints[], solution, par, stars
Tag:       fragile | liquid | electronic | cold | heavy | uprightOnly
```

## Constraints (the difficulty ladder)

| Constraint | Rule |
|---|---|
| Fit | every item inside, no overlap (always on) |
| Weight limit | total weight ≤ container limit |
| Balance | left/right halves within a weight delta (boot, van) |
| Fragile on top | no cell occupied directly above a fragile item |
| Heavy low | heavy items must sit in the bottom N rows |
| Keep apart | liquid may not be orthogonally adjacent to electronic |
| Keep together | all cold items form one connected group |
| Upright only | rotation disabled for that item |
| Reachable | item must touch the opening edge (first-aid kit, snacks) |

## Content

**8 chapters, 200+ levels**, ordered by measured difficulty:
Carry-On, Moving Day, The Fridge, Tool Drawer, Picnic Basket, Camera Bag,
Car Boot, The Van. Each chapter introduces one new constraint and reuses the
earlier ones. ~110 distinct illustrated items.

## Level generation and proof (offline, ships nothing unverified)

`art_src/levelgen.swift`, compiled with `swiftc` and run as a CLI:

1. **Construct** a packing by backtracking placement into the grid to a target
   density — this yields a known-good solution by construction.
2. **Derive** constraints from that solution (an item is marked fragile only if
   nothing sits above it in the solution), so constraints are satisfiable.
3. **Verify** with an independent backtracking solver that ignores the stored
   solution: it must find at least one legal packing.
4. **Measure** difficulty: solution count (capped), fill density, item count,
   constraint count → chapter assignment and 3/2/1-star move thresholds.
5. **Assert** invariants over every level and fail the build on violation:
   solvable, every item placeable somewhere, no duplicate ids, par ≥ item count.

Output: `Levels.json` in the bundle (a few hundred KB).

## Screens (custom tab bar, 4 tabs)

1. **Play** — current chapter, big continue card, chapter progress rings.
2. **Chapters** — 8 illustrated chapter cards, per-chapter stars, locked states.
3. **Board** (full screen, pushed) — container, tray, rotate/undo/hint, rule
   chips for the level; win sheet with stars and next-level button.
4. **You** — stars total, levels solved, perfect packs, hints used, streak,
   18 badges, settings (haptics, reduce motion, colour-blind-safe tags),
   offline privacy sheet, reset.

Onboarding: 3 illustrated slides (fit it in → mind the rules → earn stars).

## Look

Warm workshop-and-luggage palette: canvas, leather tan, brass, deep navy,
signal amber for rule violations. Items drawn as clean flat-with-texture
illustrations, generated by `art_src/artgen.swift` (Core Graphics, deterministic).
Forced light appearance, custom Path glyphs only, no SF Symbols, no emoji.

## Assets

- ~110 item illustrations (512 px) + 8 container hero images + 8 chapter cards
  + 3 onboarding scenes + paper/canvas texture. Target ≥ 18 MB installed.
- The 43 MB of music-box audio is **deleted** — wrong app now. New SFX only:
  place, rotate, snap-fit, refuse, chapter complete (5 short WAVs, < 400 KB),
  synthesized by `art_src/sfxgen.swift`.

## Technical

- Rename Tin Melody → Snug Fit: folders, Xcode project, scheme, display name,
  bundle id `com.snugfit.app`, GitHub repo. Use the `ios-app-rename` skill.
- Keep: iOS 15.6+, NavigationView only, SwiftUI + Canvas, UserDefaults JSON,
  iPhone + iPad adaptive (the `TinLayout` metric approach is reused, renamed),
  all four iPad orientations (ITMS-90474).
- **No WebView, no network, no permissions, no notifications** — the gate stays
  removed, as Master ruled.
- Version stays **1.0 (1)**. Icon: abstract, opaque, no alpha.

## Out of scope

Free-form (non-grid) placement, physics, level editor, online leaderboards,
timed modes.
