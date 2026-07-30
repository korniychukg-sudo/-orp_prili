# Tin Melody v2 — Living Parlor (2026-07-30)

## Goal
Master's brief: make the app noticeably more stylish, rich and full — it must feel
like a crafted, genuinely delightful instrument, not a thin utility. Functional
additions welcome where they fit. Version/build stay 1.0 (1).

## Approaches considered
1. **Living Parlor enrichment (chosen)** — follow the proven "living edition"
   pattern of the set (Sound Grove, Hive Warm v2, Paper Folds v2): a living
   showpiece mode, deeper craft tools, more content, more atmosphere. Keeps the
   unique composer core and multiplies its charm.
2. Social/export direction (share tunes as files/QR) — rejected: network/export
   surface, review risk, offline rule.
3. Learning direction (music-theory almanac, guided lessons) — rejected: pushes
   the app toward "reference book", the exact feel Master wants to avoid.

## What v2 adds

### 1. Parlor Mode (fullscreen showpiece)
`.fullScreenCover` living room: generated parlor backdrop (window, wallpaper,
side table), the selected box's hero art on the table, real-clock time-of-day
tint (dawn/day/dusk/night) with drifting dust motes and window light shafts.
The current tune plays on loop with the live mechanism strip at the bottom;
big crank affordance (drag anywhere to crank). Enter via an "Parlor" pill in
Workshop and a button on tune cards. Exit chevron. Reduce-motion respected.

### 2. Living scene upgrade (Workshop header scene)
The mechanism card becomes a stage: selected box's palette drives a wooden-bed
backdrop with brass rails, the comb/cylinder rendering stays but gains depth
(shadow bands, glinting pins near the strike line), the crank in the scene
rotates with the transport, and a paper tape ribbon flows under the cylinder
with the actual punch pattern scrolling by. Idle state breathes (slow sheen).

### 3. Composer power tools (real usefulness)
Tool row above the grid: **Undo** (20-level stack, punches only), **Shift ◀ ▶**
(rotate pattern by one step), **Transpose ▲ ▼** (diatonic row shift, clamped,
notes that would fall off stay put — non-destructive feel), **Echo bar** (copy
last non-empty bar into the next empty one), and **row audition** — tapping a
note name in the gutter plucks that tine. Tools live in a compact PanelCard;
each action previews sound/haptic.

### 4. More classics (10 → 16)
New pre-punched tapes, all diatonic-C public domain: Scarborough Fair (dorian
on white keys), Simple Gifts, Yankee Doodle, When the Saints, My Bonnie,
Camptown Races. Six new medallion covers in artgen.

### 5. Tape papers (collection layer #2)
6 unlockable tape styles recoloring the editor paper + scene ribbon: Workshop
Cream (start), Blueprint, Kraft Parcel, Rose Letter, Midnight Ledger, Mint
Apothecary. Unlocks tied to craft level (1/3/5/7/9/12). Picker chip row in
Workshop; persisted.

### 6. Daily Air + craft streak
Deterministic pleasant melody generated from the date (seeded walk over C-major
pentachord with cadence), offered as a card on Melodies: play it or "Take to
Workshop" to remix. Crafting on consecutive days builds a streak (any punch/
save/play counts); streak shown on You with its own stat tile.

### 7. Audio polish
- AVAudioUnitReverb (smallRoom, ~16% wet) behind the pluck pool — parlor air.
- 3 tiny generated mechanical samples (audiogen additions): crank ratchet tick
  (every 1/8 turn), paper punch thock (layered under note preview), save chime
  (3-note brass arp). Volume-balanced, short, mono.

### 8. Badges 12 → 18, level titles reused
New: Streak 3, Streak 7, Parlor Guest (enter Parlor Mode), Tape Dresser (change
tape paper), Air Collector (remix 3 Daily Airs), Transposer (use transpose 10x).
Stats extended accordingly (parlorVisits, tapeChanges, airsRemixed,
transposeUses, streak fields).

### 9. Shelf & gallery dressing
Melodies shelf cards gain a mini punch-pattern preview strip (drawn from the
tune, deterministic). Boxes tab rows sit on drawn wooden shelf planks; Melodies
banner unchanged. You tab gains streak tile + new badges.

## Art additions (artgen.swift)
6 classic covers + parlor backdrop (1200×1600) + wooden shelf plank tile
(512×128). Everything else drawn in SwiftUI. Expected +3–5 MB.

## Audio additions (audiogen.swift)
tick.wav (~90 ms), thock.wav (~120 ms), chime.wav (~1.2 s). <300 KB total.

## Data/model changes
- TinSettings + tapeStyleID; TinStats + parlorVisits, tapeChanges, airsRemixed,
  transposeUses, streakDays, lastCraftDay (yyyymmdd int), bestStreak.
- TinTune unchanged (undo stack is in-memory only).
- New TinTapeStyle catalog; TinDailyAir generator (pure function of date).
- All persisted keys keep v1 names; new keys additive — existing installs keep
  their data.

## Constraints honored
iOS 15.6+, custom UI only, no SF Symbols/emoji, forced light, offline,
no notifications, version/build stay 1.0 (1), <99 MB (est. ~68 MB device).

## Out of scope
Export/import, accidentals/chromatic comb, audio recording, iCloud.
