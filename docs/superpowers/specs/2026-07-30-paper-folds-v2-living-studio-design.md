# Paper Folds v2 — Living Studio (design)

Date: 2026-07-30 · App: Paper Folds (`com.paperfolds.app`) · Marketing version stays **1.0 (build 1)**.

## Goal

v1 reads like a tidy guided reference: pick a model, follow steps, done. v2 must make the app
feel like a **living paper studio** — a place with light, mess and creative freedom — so it lands
as a delightful tool, not a catalog. No regressions to the fold engine or the WebView gate.

## What ships

### 1. Free Fold Studio (new core feature)

A blank square + the existing fold engine, no instructions. The user drags from any point on the
paper (grab point A) to anywhere (target B); the paper folds across the **perpendicular bisector
of A→B**, moving the side that contains A — exactly how real paper behaves, and exactly the math
the engine already uses. Extras:

- Fold completes with the same tween/haptic as guided mode; slivers (moved area < 1.5% of paper)
  are rejected with a gentle shake so layer soup can't happen.
- Buttons: Undo (state stack), Flip, Rotate 45°, Reset, Save. Cap of 12 folds per piece
  (counter shown as "folds left"), guarding polygon blow-up.
- Paper picker works here too (same sheet).
- **Save** stores the step sequence (Codable `FreePiece`: name, date, paper id, steps) in
  UserDefaults; auto-name "Paper Sketch N", rename in save dialog. +2 XP per fold, capped +20/piece.
- Entry points: hero card on Studio tab + Gallery's new segment.

### 2. Gallery: Models / Sketches segments

Segmented control. Models = existing shelf. Sketches = saved Free Fold pieces on the same wooden
shelves, rendered by replaying their step list through the engine; tap → sheet with big render,
date, fold count, Delete and Refold (opens Free Fold pre-loaded with the sequence replayed).

### 3. Living Desk (Studio tab rework)

The home becomes a scene, not a card list:

- Time-of-day tint over the desk (morning gold / day neutral / dusk amber / night indigo with
  soft vector stars), driven by the clock.
- A **desk still-life strip**: the 3 most recently folded models lie on the desk at small
  random angles (engine renders), with soft shadows; empty desk shows a blank square + kind copy.
- Slow drifting paper-triangle particles over the scene (TimelineView canvas, subtle).
- Existing cards (daily fold, stats, next up, tip) restyled below the scene; Free Fold hero card added.

### 4. Fold quality stars

During guided folds the app scores each drag: released at progress ≥ 0.9 → crisp; 0.7–0.9 → neat;
else loose. Model result = average over fold steps → 1–3 stars, kept as best-ever per model
(`bestStars[modelId]`). Stars show on the completion card ("Crisp fold!"), library cards and
gallery sheet. Replays and turn steps don't count.

### 5. Studio life & polish

- **Onboarding**: 3-page first-launch cover (drag-to-fold, shelf & papers, learn & quiz) using
  existing banner art + vector shapes; "Start folding" ends it. Flag in store.
- **Rank-up celebration**: when XP crosses a rank threshold, a full-screen moment (crown, confetti,
  new rank name, unlocked papers count) queued after the completion overlay closes.
- **Folding calendar**: Profile gains a 5-week heat grid of active days (fold or quiz), built from
  a persisted set of day-strings.
- **Micro-stories**: every model gets a 1–2 sentence cozy story ("story" field in models.json),
  shown in the detail sheet under the thumbnail and on the completion card instead of the bare fun
  fact (fun fact moves to the detail sheet).
- **Two new badges**: Inventor (save 1 sketch), Paper Dreamer (save 5 sketches) → 20 badges total.

## Approaches considered

- **Polish-only** (stars, stories, animations): cheaper but keeps the "reference" shape — rejected.
- **Challenge/timer layer**: conflicts with the calm-studio tone — rejected.
- **Living Desk + Free Fold** (chosen): converts the app from catalog to creative tool and gives
  the home screen a heartbeat; reuses the proven engine, so risk stays low.

## Architecture

- `FreeFoldView.swift` (new): sandbox screen; owns `[PaperState]` undo stack + `[FoldStep]`-like
  action log (local `FreeAction` enum encoding fold(a,b,move)/flip/rotate for persistence).
- `FoldsStore`: `freePieces: [FreePiece]`, `bestStars: [String: Int]`, `activityDays: Set<String>`,
  `onboarded: Bool`, `pendingRankUp: FolderRank?` (+ snapshot fields, badge rules).
- `OnboardingView.swift` (new). Desk scene lives in `HomeView` (`DeskSceneView` subview).
- `FoldingView` gains star scoring (per-fold release progress collected in `foldScores`).
- Engine untouched except a helper `PaperState.applyFree(_:)` and area-ratio guard for sandbox.
- Art: one new banner (`banner_freefold`) via restored `art_src`; everything else is SwiftUI vector
  work + existing assets. `art_src` is restored from git for the batch, then removed again before
  delivery.

## Testing

Compile-clean build; code review of new flows; simulator screenshots via existing env hooks
(new: `PF_TAB=0` desk scene day/night by device clock, `PF_FREE=1` opens Free Fold). Version/build
untouched (1.0 / 1).
