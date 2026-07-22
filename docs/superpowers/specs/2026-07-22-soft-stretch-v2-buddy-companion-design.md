# Soft Stretch v2 — Buddy Companion Edition (design spec)

Date: 2026-07-22
App: Soft Stretch (`com.softstretch.app`), delivered v1.0 → rework in `development/`
Goal: turn the app from "nice follow-along player + lists" into a companion app the
user opens daily — Buddy becomes a living character you befriend, dress and whose
nook you fill by practicing — plus real utility depth (weekly programs, custom
routine builder, richer progress). Ships as v1.0 (build 1) per versioning rule.

## Problem

v1's unique asset — the live pose-animated Buddy — only performs during sessions.
Outside the player the app reads like a catalog: static cover cards, plain lists,
three stat tiles. Nothing accumulates, nothing reacts, nothing is *yours*. The user
has no reason to open the app beyond "do a routine".

## Decisions (confirmed with Master)

1. Direction: **Buddy Companion** — emotional companion loop (moods, friendship
   levels, unlockable looks, living nook) layered on top of real utility.
2. Feature set: **full** — all 6 blocks approved: living Buddy + nook, friendship &
   customization, 7-day programs, custom routine builder, richer progress, style pass.
3. No new PNG art required — nook, outfits and moods are drawn in Canvas from the
   existing rig (crisp at any size, no bundle bloat; app stays ~28 MB ≥ 18 MB rule).
4. All constraints of the set stay: iOS 15.6+, custom UI only, no SF Symbols/emoji,
   offline, no permissions, forced light, version 1.0 (build 1).

## 1. Data: CompanionStore (new file `CompanionStore.swift`)

New `CompanionState: Codable`, persisted under **new key** `soft.companion.v1`
(v1 saves untouched — no migration risk; `StretchStore` keys unchanged except a
tolerant addition, see §5):

- `xp: Int` — friendship points: `max(1, seconds/60)` per finished session
  `+ 5` session bonus `+ 3` if the session extends a streak (yesterday or today
  already had a session).
- `equippedSkin: String` (default `"mint"`), `equippedAccessory: String?`.
- `celebratedLevels: [Int]` — level-up ceremonies already shown.

Pure logic in the same file:

- `levelThresholds = [0, 10, 25, 45, 70, 100, 140, 190, 250, 320, 400, 500]`
  → levels 1…12; `level(for: xp)`, `progressToNext(xp)`.
- `FriendshipReward` table — every level unlocks exactly one thing:
  L2 peach skin, L3 sweatband, L4 sky skin, L5 scarf, L6 flower crown, L7 lilac
  skin, L8 round glasses, L9 sand skin, L10 beanie, L11 bow tie, L12 sunrise-gold
  skin. Level names: "New Friends" → … → "Best Friends" (12 short names).
- Nook items are also level-driven (§3): one per level, visible immediately.

`CompanionStore: ObservableObject` mirrors `StretchStore` style (load/save JSON in
UserDefaults, `@Published var state`). Injected as a second `environmentObject`
from `SoftStretchApp`.

## 2. Buddy looks: outfits & moods (extend `BuddyView.swift`, new `BuddyOutfits.swift`)

`BuddyCanvas` gains `outfit: BuddyOutfit` and `mood: BuddyMood` (defaults keep all
existing call sites compiling: `.classic`, `.content`).

- `BuddyOutfit { skin: BuddySkin, accessory: BuddyAccessory? }`.
  `BuddySkin` = 6 palettes (body, deep, cheek): mint (v1), peach, sky, lilac, sand,
  sunrise gold. Skin recolors both torso tints and the cheek blush.
- `BuddyAccessory` (drawn in `BuddyOutfits.swift` as pure Canvas helpers positioned
  from `Joints` + `headAngle`, so they track every pose): sweatband (coral band
  across forehead), scarf (two soft loops at neck + dangling tail), flower crown
  (5 small flowers along head top), round glasses (two rings + bridge at eye line),
  beanie (dome + folded brim + pompom), bow tie (at neck). Each ≤ 25 lines of path
  code, drawn after the head/face so they sit on top.
- `BuddyMood` affects only the face: `content` (v1 face), `sleepy` (half-lid eyes,
  small o-mouth, tiny "z z" drifting above head), `happy` (arc eyes ∪∪, open smile),
  `proud` (normal eyes + big smile + 2 sparkles beside head). Mood picked by
  context, never persisted.

Everywhere Buddy renders (nook, player, library previews, sheets) the equipped
outfit is passed in, so the customization is felt across the whole app.

## 3. Buddy Nook — living home hero (new `BuddyNookView.swift`)

Replaces the static cover-art hero on Home with a live indoor scene
(`TimelineView(.animation)` + single `Canvas`, ≤ 40 fps, static frame under
reduce-motion):

- **Room**: warm wall + floor, a window with sky gradient by real time of day
  (dawn 5–9, day 9–17, dusk 17–21, night 21–5) with sun/moon disc and 2 drifting
  cloud puffs; a soft rug and Buddy's mat.
- **Buddy idle**: stands centered in a gentle idle cycle — breathing bob (reuse
  `breathe` layering), blink every 3–5 s, occasional slow side-lean "mini stretch"
  every ~14 s. Mood: sleepy at night/before first session of the day at dawn,
  proud when streak ≥ 3 and today done, happy when today done, content otherwise.
- **Tap reaction**: Buddy waves (arm raise animation ~1.2 s) + haptic tap + a
  speech bubble picks a contextual line (12+ lines: morning/evening, streak,
  "ready for Desk Undo?", after-session praise…). Bubble also appears on its own
  every ~20 s idle.
- **Nook items** (one per friendship level, drawn as vector props layered around
  the room): L1 potted sprout → grows into leafy plant by L5 (3 stages), L2 wall
  poster (tiny buddy doodle), L3 floor lamp (warm glow ellipse at night), L4 round
  rug upgrade, L5 hanging shelf + trophy, L6 string lights along the window
  (twinkle at night), L7 second plant (hanging vine), L8 wall clock (hands show
  real time), L9 sleeping cat on the rug (tail flick), L10 framed "best friends"
  photo, L11 candle with flame flicker, L12 golden sun catcher (slow sparkle).
- Header above the scene keeps greeting + date; a small **friendship chip** (level
  ring + "Lv 4 · Warm Pals") sits on the scene corner → opens Buddy Studio.

## 4. Buddy Studio (new `BuddyStudioView.swift`, pushed from Home)

- Top: large live Buddy preview (current outfit, happy mood) on a soft podium.
- **Friendship card**: level name, XP ring, "N xp to next", next reward teaser
  ("Lv 5 unlocks: cozy scarf" with a small locked silhouette).
- **Skins row**: 6 swatch circles (locked ones dimmed with level tag), tap equips.
- **Accessories grid**: none + 6 accessory tiles (each a mini buddy-head preview
  wearing it), locked state shows level requirement. Tap equips/removes.
- Equipping saves instantly; every equip pulses the preview + haptic.

## 5. Programs — 7-day journeys (new `ProgramLibrary.swift`, UI in Routines tab)

- 3 programs × 7 days, each day = existing routine + custom day title/blurb:
  **Gentle Week** (full-body ease-in), **Desk Rescue** (neck/shoulders/posture),
  **Wind-Down Week** (evening/hips/hamstring). Programs reference only existing
  `RoutineLibrary` ids — no new stretch content needed.
- Progress: `programDays: [String: [Int: Date]]` — stored in **CompanionState**
  (same new key, still no v1 migration risk).
- Routines tab becomes segmented: **Routines | Programs | My Own** (custom
  segmented control, coral pill).
- Programs list: 3 tall cards (routine cover art + progress dots 1–7 + "Day N
  next"). Detail screen: hero, 7 day rows (done ✓ tinted / next highlighted with
  Start button / future soft), day completion = finishing that routine session
  from the day row (PlayerView gets optional `onComplete` context, see §7).
  Days can be repeated; completing out of order is allowed (next = first
  incomplete). Finishing all 7 → one-time program celebration overlay + the
  program card gets a "Completed" ribbon.

## 6. My Own — custom routine builder (new `RoutineBuilderView.swift`)

- `CustomRoutine: Codable { id: UUID, name: String, tint: String (area id),
  steps: [CustomStep { stretchID, seconds ∈ {20,30,45,60} }] }` stored under
  `soft.customroutines.v1` in `StretchStore` (new published array, tolerant load).
- "My Own" segment: created routines as cards (gradient tint + live BuddyPreview
  of first stretch + total minutes) + a dashed "New routine" card.
- Builder sheet: name field (custom styled TextField), tint picker (4 area
  pastels), stretch picker grouped by area (tap adds; added rows show ▲▼ reorder
  and seconds chip cycling 20→30→45→60, remove button). Save requires name +
  ≥ 2 steps. Edit = same sheet prefilled; delete with confirm.
- Custom routines play in the existing `PlayerView` by mapping to the `Routine`
  struct on the fly (id `"custom-<uuid>"`); sessions record with the custom name.

## 7. Player & finish-screen upgrades (`PlayerView.swift`)

- Buddy wears the equipped outfit during sessions; backdrop gets a soft
  time-of-day tint wash (reuses nook sky logic, very subtle).
- Finish screen, in order: confetti v2 (mixed capsules/dots/tiny stars in theme
  colors), stat row, **XP tally** — "+N friendship xp" counts up with ticks; if a
  level boundary crossed → **level-up ceremony**: burst of sparkles, "Lv 5 —
  Cozy Pals!", reveal card of the unlocked wearable (skin swatch / accessory on
  a mini buddy) with an **Equip now** button, plus a secondary line "…and a new
  something appeared in Buddy's nook"; then fresh badges as in v1. If session came from a program day → "Day N of
  <Program> complete" chip on the finish screen.
- `CompanionStore.award(seconds:)` computes xp and publishes
  `lastReward: (gained: Int, oldLevel: Int, newLevel: Int)?` (the `freshBadges`
  pattern) so the finish screen drives the ceremony without recomputing.

## 8. Progress tab upgrades (`ProgressTabView.swift`)

- Top: **friendship summary card** (level ring, name, xp bar → Studio link).
- **Month heatmap**: custom calendar grid (Mon-start), day cells tinted by
  minutes (4 intensity steps of coral), month switcher ‹ ›, capped at months
  with data; today outlined.
- **Body balance**: 4 rows (Neck & Shoulders / Back & Core / Hips & Legs /
  Arms & Wrists) with soft progress bars showing share of stretched minutes,
  computed from per-session `areaSeconds` — `SessionRecord` gains **optional**
  `areaSeconds: [String: Int]?` (old records decode as nil and are simply
  excluded from balance; totals/streaks unaffected).
- Existing tiles/badges stay below.

## 9. Style pass (across files, tokens in `StretchTheme.swift`)

- New tokens: `heroGradient(timeOfDay)`, elevated card shadow style, pill
  segmented control, springy `softAppear` modifier (staggered fade+rise on list
  items), pressed-scale style everywhere interactive.
- Tab bar: active tab gets a soft coral pill behind icon+label.
- Onboarding: page 3 text refreshed to mention growing a friendship; pager dots
  animated width.
- More tab: adds a "Meet Buddy" row (opens Studio) and keeps everything else.

## 10. Out of scope

No audio, no notifications, no new PNG art pack, no iPad-specific layouts beyond
existing adaptivity, no data export.

## Verification

- Clean `xcodebuild` Debug sim build, zero errors; code-review pass of all new
  screens (skill rule: no simulator UI loops; one screenshot pass at the end for
  delivery: home nook, studio, programs, builder, progress, player finish).
- v1 saves load: old `soft.settings.v1` / `soft.sessions.v1` untouched;
  `SessionRecord.areaMinutes` optional; companion state fresh-starts at Lv 1.
- All new UI: custom shapes only, no SF Symbols/emoji; light-forced palette.
- Bundle stays ≥ 18 MB (art unchanged) and far under 99 MB.
- Version stays 1.0 (build 1); tracker + descriptions updated on re-delivery.
