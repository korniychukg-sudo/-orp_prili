# Bead Steps — Living Strand edition (second pass)

Date: 2026-08-11. Version stays 1.0 (build 1). App lives in `for_human_review_apps/Bead Steps` with repo korniychukg-sudo/Bead-Steps.

## Goal

The first pass reads as a competent counter. The second pass makes it feel like a living devotional object and a daily companion: a strand that hangs in a real sky, materials earned by practice, a visible daily rhythm (wird), and a study loop for the 99 names.

## Features

### 1. Living prayer sky (Beads tab)
The strand area gains a Canvas scene behind the beads driven by the real clock: dawn (4–7) warm horizon glow and a low sun disc; day (7–16) pale wash; dusk (16–20) amber-to-violet band; night — deep indigo wash, seeded twinkling stars and a crescent. A faint hatched skyline band sits at the bottom. All colors stay low-contrast so the strand remains the subject. Implemented inside `MisbahaCanvas.swift` (`PrayerSky` drawn in the same TimelineView Canvas pass, no new files). Idle strand gets a slow breathing sway (phase-shifted sine on the curve amplitude).

### 2. Earned bead materials (8 strands)
Three new materials — lapis, rosewood, coral — join the five. Materials unlock by practice: jade free; amber 1,000 beads lifetime; walnut 7-day streak; coral 25 rounds; onyx 5,000 beads; rosewood 20 set completions; lapis 15 names read; pearl 10,000 beads. Settings shows the shelf with locked swatches and a one-line requirement; unlocking pops the badge toast pattern. `stylesTried` badge logic now counts unlocked-and-tried. Persisted state is unchanged (unlocks are computed from existing stats), so old saves work.

### 3. The Daily Wird (Adhkar tab)
A timeline card at the top of Adhkar: nine nodes for the traditional day — waking, morning, five after-prayer tasbihs (Fajr/Dhuhr/Asr/Maghrib/Isha), evening, before sleep. Node states come from existing `setDayLog`/`prayerByDay` for today; tapping a node launches the corresponding set with the existing queue flow. A ring on the card shows N of 9 done; the Journal gains a matching wird ring card. No new persistence beyond what exists (prayer count per day already stored).

### 4. Names study drill (Names tab)
A "Study ten" button opens a flashcard drill inside `NamesView.swift`: Arabic + transliteration shown, tap reveals meaning + reflection, then "Knew it" / "Again"; Again cards recycle to the deck's end; deck = 10 names biased to unread. Finishing marks knew-it names as read (existing `markNameRead`), shows a small result card. No new persistence.

### 5. Journal depth
Two additions: "Rhythm of the day" — beads summed into eight 3-hour buckets computed from the stored session log; and the wird ring card. The materials shelf appears in Settings, not Journal.

## Non-goals
No new art plates (scene is vector), no widgets (iOS 15.6 target stands), no changes to adhkar content, no new badges, no store schema changes.

## Verification
Debug sim + Release device builds clean; validator re-run; changed screens re-shot on the dedicated sim (Beads day/night via env clock is not possible — shoot at current hour; Adhkar wird; Names drill; Journal; Settings shelf); zero comments; push to the existing repo; tracker/descriptions updated. Version/build untouched.
