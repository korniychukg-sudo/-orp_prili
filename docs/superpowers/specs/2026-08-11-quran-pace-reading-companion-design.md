# Quran Pace — Reading Companion edition (second pass)

Date: 2026-08-11. Version stays 1.0 (build 1). App lives in `for_human_review_apps/Quran Pace` with repo korniychukg-sudo/Quran-Pace.

## Goal

The first pass is an honest logbook. The second pass makes it a companion you open before reading, not only after: a living header, a reading-sitting timer that learns your true speed, juz seals to collect, and one-tap khatm presets.

## Features

### 1. Living Today header
When a khatm is active, the static hero is replaced by a Canvas scene in `TodayView.swift`: a sky band following the real clock (sun arc by day, crescent and seeded stars by night), a hatched skyline, and an open book on a rehal whose right-hand page block visibly fills with the khatm percentage; the ring stays in the portion card. The no-plan state keeps the existing engraved plate.

### 2. Reading sittings (the headline feature)
A "Begin a sitting" button on Today opens a full-screen cover: a quiet elapsed timer (TimelineView), the current bookmark line, and a breathing ring. Ending the sitting asks how many pages were read (stepper, prefilled with today's remaining goal, 0 allowed) and logs them through the existing `logPages`. Each sitting is stored as `SittingRecord {date, minutes, pages}` (new array in `QPState`, `decodeIfPresent`-tolerant, capped at 200). Derived stats: minutes per page (median-ish trailing average), average sitting length, total minutes. The Today pace card gains one line: "~X min/page · today's portion ≈ Y min". The Journal gains a Records card: best day (pages), longest sitting, total hours, pace.

### 3. Juz seals
Passing a juz boundary celebrates it: `logPages`/`setPosition` detect newly completed juz and queue a celebration overlay ("Juz N sealed", rosette + opening words). The Mushaf tab's parts view header gains a seals strip — 30 small medallions, gold when the juz is fully read, tappable to its card. State is computed from position; only the "already celebrated" set is persisted (`sealedJuz: Set<Int>`, tolerant decode).

### 4. Khatm presets
Four chips in the setup sheet above the mode toggle: "One juz a day" (perDay 20), "In 30 days" (byDate +30), "In 60 days" (byDate +60), "Two pages a day" (perDay 2). Tapping fills the controls; everything stays editable.

### 5. Juz quarters
Each juz card shows four small tick dots under the opening words — the rub quarters of that juz (computed by splitting its page range in four), filled as the bookmark passes them.

## Non-goals
No widget-extension changes (snapshot format untouched), no new art plates, no Quran text, no notification of any kind, no schema-breaking changes.

## Verification
Debug sim + Release device builds clean; validator re-run; changed screens re-shot (Today hero + sitting cover, Mushaf seals, Journal records, setup presets); zero comments; push to existing repo; tracker/descriptions updated. Version/build untouched.
