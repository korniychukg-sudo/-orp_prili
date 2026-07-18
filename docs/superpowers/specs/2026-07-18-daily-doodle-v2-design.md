# Daily Doodle v2.0 — Journey Edition: Design

Date: 2026-07-18
Status: approved by Master (full package, journey map form)
Project: `development/Daily Doodle` (iOS 15.6+, SwiftUI, custom UI, offline)

## Goal

Turn Daily Doodle from a "lesson catalog + canvas" into a guided, richly styled
drawing companion. Two levers, both approved: a journey feature set that leads
the user by the hand, and a deep visual polish pass. Nothing about the v1
constraints changes: iOS 15.6+, no SF Symbols/emoji, custom components only,
forced light, local storage, WebView gate untouched, app < 99 MB.

## Feature set

### 1. Journey tab (new, becomes the 2nd tab)

A vertically scrolling hand-drawn winding path through all 33 lessons, ordered
easy → tricky, grouped into 6 named chapters (~5–6 lessons each). The order is
a static curated list (`JourneyMap`), independent of catalog categories.

- Node states, derived purely from existing `completions` (no migration):
  - **done** — inked node showing the mini lesson cover, accent ring;
  - **current** — first not-done node on the path, pulsing accent halo;
  - **upcoming** — dashed "pencil sketch" circle with faint lesson silhouette.
  No hard locks: every node is tappable and opens the normal LessonPlayerView.
- **Warm-up nodes**, one at the start of each chapter (6 total): quick trace drills (straight
  lines, circles, waves, loops, zigzags, spirals). Each drill shows guide
  shapes; on finish the app scores accuracy = mean distance of user stroke
  points to the nearest guide-path point, mapped to 1–3 stars. Results persist
  (best stars per warmup). Lessons themselves stay unscored by design.
- **Chapter milestone rewards**: completing all lessons of a chapter earns a
  generated sticker (6 new art PNGs) with a confetti + haptic moment on the
  Journey screen. Stickers display on a shelf in Studio.
- Chapter headers on the path: name, ribbon art, progress x/y.

### 2. Animated stroke guides in the lesson player

The current step's guide paths "draw themselves" in a repeating loop — a trim
animation with a small pen dot travelling along the stroke, one path after
another, showing direction and order. Driven by `TimelineView(.animation)`
recomputing `trimmedPath(from:to:)` inside the existing Canvas (iOS 15 OK).
A toggle button next to the guide-eye button turns the animation on/off
(default on). Static guide remains beneath at the user's chosen opacity.

### 3. Artwork replay

When saving artwork (lesson finish or free draw), the normalized stroke data is
also written as JSON next to the PNG (`<id>.strokes.json`). The gallery viewer
gains a **Replay** button: the drawing re-draws itself stroke by stroke on a
paper card (time-scaled so any drawing replays in ~4–6 s). Works only for new
artworks; old ones without JSON simply hide the button (nil-safe).

### 4. Deep visual polish

- **Paper grain texture** (`paper_grain.png`, generated, tileable, ~4% alpha)
  overlaid on the app background and canvas paper.
- **Wobbly hand-drawn borders**: a custom `WobblyRoundedRect` Shape (seeded
  sine-perturbed rounded rect) replaces plain RoundedRectangle strokes on key
  cards (hero, chapter headers, completion overlay) for a sketchbook feel.
- **Tape & sticker decor**: small tinted "washi tape" strips on gallery cards
  and Today hero; slight per-card random tilt (seeded by id) in the gallery so
  it reads as an art wall.
- **Confetti**: Canvas-based particle burst on lesson completion, chapter
  sticker unlock, and 3-star warmups.
- **Haptics** (UIKit feedback generators — allowed, not notifications):
  success on finish/sticker, light impact on step advance and tool select.
- **Animated streak flame**: subtle pulse when streak > 0.
- **Today hero backdrop**: the previously unused `hero_today` art becomes the
  Today header backdrop with the greeting overlaid.
- Springy button press styles (scale effect) app-wide.

### 5. New generated art (extend `art_src/main.swift`)

6 chapter stickers, 6 warm-up tiles, paper grain tile, journey chapter ribbons.
Same palette/style; deterministic; keeps total app well under 99 MB.

### 6. Model/persistence additions

- `warmupStars: [String: Int]`, `earnedStickers: [String]` in `DoodleStore`
  (UserDefaults, same pattern as existing keys).
- Stroke JSON save/load helpers in the artwork pipeline.
- Badges extended 12 → 14: "Limber Wrist" (finish all 6 warmups),
  "Sticker Collector" (earn all 6 chapter stickers).
- Tab bar: Today, Journey, Lessons, Gallery, Studio (5 tabs).

## Non-goals / rejected alternatives

- Hard-locking lessons on the path (conflicts with the open catalog; frustrating).
- Accuracy scoring inside lessons (kills creative confidence; warmups only).
- Sound effects (haptics only), cloud anything, new permissions.

## Architecture notes

New files: `JourneyView.swift`, `JourneyMap.swift` (curated order + chapters +
warmup definitions), `WarmupPlayerView.swift`, `DoodleEffects.swift` (confetti,
wobbly shape, haptics, texture overlay, press style), `ReplayView.swift`.
Modified: store (new state), player (guide animation toggle + confetti +
haptics), gallery viewer (replay), Today (hero backdrop, flame pulse), Studio
(sticker shelf, new badges), RootView (5th tab), art generator (new assets).
Existing lesson data and drawing engine are untouched; warmups reuse
`DoodleCanvasModel` + `DrawingCanvasView` with their own guide paths.

Accuracy scoring: sample guide paths into dense point lists once per drill
(reuse `SketchPath.cgPath` flattening via the existing 4° arc sampling; for
lines/curves sample `trimmedPath` at fixed t steps). Score = mean over user
points of distance to nearest guide point, normalized by canvas side:
< 0.020 → 3 stars, < 0.035 → 2, else 1. Empty drawing → no score prompt.

## Testing / verification

- Clean `xcodebuild` (sim Debug + device Release), zero real warnings.
- Code review of all new screens and state transitions.
- Sim smoke test: launch, Journey render, warmup flow, replay button hidden for
  pre-v2 artworks, completion confetti. Contact-sheet visual check of new art.
- Size check (≥ 18 MB, < 99 MB) and icon alpha check unchanged.
