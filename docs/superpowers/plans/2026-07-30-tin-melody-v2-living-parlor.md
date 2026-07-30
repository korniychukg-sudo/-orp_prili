# Tin Melody v2 Living Parlor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enrich Tin Melody into the Living Parlor edition — fullscreen parlor showpiece, composer power tools, 16 classics, tape papers, Daily Air, mechanical audio polish, 18 badges.

**Architecture:** Additive changes to the v1 SwiftUI app (in-place file edits + 2 new views + catalog additions). Deterministic generators (artgen/audiogen) grow; persisted keys stay backward compatible.

**Tech Stack:** SwiftUI (iOS 15.6), AVAudioEngine (+AVAudioUnitReverb), Core Graphics generators compiled with swiftc.

## Global Constraints
- iOS 15.6+, NavigationView only, Canvas/TimelineView allowed (iOS 15).
- Custom Path/Canvas glyphs only; no SF Symbols; no emoji; English (US).
- Forced Light appearance; fixed antique palette (TinTheme).
- MARKETING_VERSION 1.0, CURRENT_PROJECT_VERSION 1 — do NOT bump.
- Offline only; no notifications; app < 99 MB.
- Work happens in `development/Tin Melody` (move from for_human_review_apps first, move back on delivery).
- Verification: `xcodebuild` sim Debug + device Release clean; sim screenshots via `xcrun simctl` (no UI-tap tooling; use temporary env hooks, remove before delivery).

---

### Task 1: Move app to development; audio additions (reverb + mechanical samples)

**Files:**
- Modify: `art_src/audiogen.swift` (append mechanical-sample synthesis)
- Modify: `Tin Melody/TinAudio.swift`

**Interfaces:**
- Produces: `TinSampleBank.playTick()`, `playThock()`, `playChime()`; new WAVs `mech_tick.wav`, `mech_thock.wav`, `mech_chime.wav` in `Audio/`.

- [ ] `mv "for_human_review_apps/Tin Melody" "development/Tin Melody"`
- [ ] Append to audiogen: ratchet tick = 90 ms damped 2.2 kHz sine burst + noise click; thock = 120 ms 180 Hz sine thump + LP noise; chime = C5-E5-G5 tin plucks at 0/0.12/0.24 s mixed into one 1.2 s buffer. Rebuild `swiftc -O audiogen.swift -o audiogen && ./audiogen` (guard so the 120 note files regenerate identically; acceptable — deterministic).
- [ ] TinAudio: insert `AVAudioUnitReverb` between players and mainMixer (`loadFactoryPreset(.smallRoom)`, `wetDryMix = 16`); attach players to reverb instead of mixer. Load the 3 mech buffers once (not per box set); `playTick/Thock/Chime` schedule on the rotating pool at volume 0.5/0.6/0.8.
- [ ] Build sim Debug: `xcodebuild … build` → `** BUILD SUCCEEDED **`; commit "v2: parlor reverb + mechanical samples".

### Task 2: Model/catalog/content growth

**Files:**
- Modify: `Tin Melody/TinModels.swift`, `Tin Melody/TinStore.swift`

**Interfaces:**
- Produces: `TinTapeStyle` (id, name, paper/hole/line/accent Colors, unlockLevel) + `TinCatalog.tapeStyles: [TinTapeStyle]` (6: cream L1, blueprint L3, kraft L5, rose L7, midnight L9, mint L12); 6 new `TinClassic` entries (scarborough, simplegifts, yankee, saints, mybonnie, camptown) with note arrays; `TinDailyAir.tune(for: Date) -> TinTune` (seeded xorshift walk, 32 steps, tempo 96, pentachord C5–G5 + cadence to C5); `TinBadgeBook.all` grows to 18; `TinStats` gains `parlorVisits, tapeChanges, airsRemixed, transposeUses, streakDays, bestStreak, lastCraftDay:Int` (all defaulted — old JSON decodes via custom init(from:) with decodeIfPresent); `TinSettings.tapeStyleID` (default "cream"); `TinStore.touchCraftDay()`, `registerTranspose()`, `registerParlorVisit()`, `registerTapeChange()`, `registerAirRemix()`; badge logic for 6 new ids; `TinStore.tapeStyle` computed; `isTapeUnlocked(_:)`.
- New classics use only rows 0–14, diatonic; Scarborough Fair in D-dorian on white keys.

- [ ] Implement; decode-compat via `decodeIfPresent` wrappers in TinStats/TinSettings `init(from:)`.
- [ ] Build clean; commit "v2: tape styles, 6 new classics, daily air, stats/badges growth".

### Task 3: Art additions (6 covers + parlor + shelf)

**Files:**
- Modify: `art_src/artgen.swift`

**Interfaces:**
- Produces PNGs: `cover_scarborough` (herb sprig), `cover_simplegifts` (turning sun-wheel), `cover_yankee` (feathered cap star), `cover_saints` (trumpet), `cover_mybonnie` (wave + heart), `cover_camptown` (race horseshoe); `parlor_backdrop.png` 1200×1600 (wallpaper, window with light shaft, side table); `shelf_plank.png` 512×128 wood tile.

- [ ] Append cover specs + two scene functions; rebuild/run artgen; verify 26 art files, icon untouched.
- [ ] Commit "v2: six classic covers, parlor backdrop, shelf plank".

### Task 4: Composer tools + tape styles in Workshop

**Files:**
- Modify: `Tin Melody/WorkshopView.swift` (tool row, tape style chips, undo), `Tin Melody/TinStore.swift` (undo stack)

**Interfaces:**
- Produces: `TinStore.pushUndoSnapshot()`, `undoPunches()` (in-memory `[Set<TinPunch>]`, cap 20, cleared on tune load); WorkshopView tool row: Undo, Shift◀, Shift▶, Trans▲, Trans▼, Echo bar; gutter note tap auditions row (`TinSampleBank.pluck`).
- Shift rotates steps mod tune.steps; Transpose moves rows ±1 clamping (skip punches that would leave 0..14 — they stay); Echo copies last non-empty bar to first empty bar after it (no-op + haptic if none).

- [ ] Implement; every mutating tool calls `pushUndoSnapshot()` first, `touchCraftDay()`, thock/tick sounds.
- [ ] Tape style chip row under the editor card: circles filled with style.paper + accent ring; locked chips show LockGlyph + level hint; switching calls `registerTapeChange()`.
- [ ] TapeGridView + scene ribbon read `store.tapeStyle` colors (paper bg, hole color, gridline, bar shading).
- [ ] Build; screenshot check; commit "v2: composer tools, undo, tape papers".

### Task 5: Living scene upgrade + Parlor Mode

**Files:**
- Modify: `Tin Melody/BoxSceneView.swift`
- Create: `Tin Melody/ParlorView.swift`
- Modify: `Tin Melody/WorkshopView.swift` (Parlor pill), `Tin Melody/MelodiesView.swift` (Parlor from tune card)

**Interfaces:**
- BoxSceneView adds: wooden bed backdrop bands, tape ribbon under cylinder scrolling actual punches (uses tapeStyle colors), crank widget in-scene rotating with `player.crankAngle`, idle sheen (sin of timeline date).
- Produces: `ParlorView(tune: TinTune, onClose: () -> Void)` fullscreen: `parlor_backdrop` image, time-of-day tint via real `Date()` hour (dawn 5–8 gold, day clear, dusk 17–20 amber, night 20–5 deep blue overlay + stars), dust motes (TimelineView canvas, 14 seeded motes), selected box hero art on table, auto-loop playback of tune, drag-anywhere crank (reuses `player.crank`), close chevron top-left, `registerParlorVisit()` on appear.

- [ ] Implement both; wire `.fullScreenCover(isPresented:)`; player transport shared (stopAll on close).
- [ ] Build; commit "v2: living scene ribbon + Parlor Mode".

### Task 6: Daily Air, shelf previews, You additions

**Files:**
- Modify: `Tin Melody/MelodiesView.swift` (Daily Air card, mini previews), `Tin Melody/ProfileView.swift` (streak tile, 18 badges), `Tin Melody/Components.swift` (TunePreviewStrip)

**Interfaces:**
- Produces: `TunePreviewStrip(tune:style:)` — Canvas drawing punch dots (x=step,y=row) on tape-colored strip; Daily Air card on Melodies top: date-named air ("Air for July 30"), Play (inline via player) + "Remix in Workshop" (`loadIntoDraft`, `registerAirRemix()`, switch tab).
- ProfileView: 5th stat tile "Day streak"; badges grid now 18.

- [ ] Implement; build; commit "v2: daily air, shelf previews, profile growth".

### Task 7: Verify, polish pass, deliver

- [ ] Temporary env hooks (TIN_SKIP_ONBOARD/TIN_TAB/TIN_DEMO_DRAFT/TIN_PARLOR) → sim screenshots of Workshop (tools+chips), Parlor, Melodies (Daily Air + covers), Boxes, You; fix visual defects found; refresh `screenshots/` set.
- [ ] Remove hooks; final builds: sim Debug + Release, device Release (CODE_SIGNING_ALLOWED=NO) — all `** BUILD SUCCEEDED **`, 0 errors.
- [ ] Checklist greps: `key=`, `systemName`, emoji, iOS16 API — all empty; icon hasAlpha:no; version 1.0/1 intact.
- [ ] `rm -rf build`, delete generator binaries; app repo commit; `mv` to `for_human_review_apps/`; update APP_TRACKER.md + APP_DESCRIPTIONS.md; workspace commit.

## Self-Review
- Spec coverage: §1→T5, §2→T5, §3→T4, §4→T2+T3, §5→T2+T4, §6→T2+T6, §7→T1, §8→T2+T6, §9→T3+T6. ✓
- No placeholders; types named consistently (TinTapeStyle, TunePreviewStrip, touchCraftDay). ✓
- Single-plan scope OK (one app, additive). ✓
