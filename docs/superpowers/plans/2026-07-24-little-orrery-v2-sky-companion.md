# Little Orrery v2 — Sky Companion Edition (implementation plan)

Spec: `docs/superpowers/specs/2026-07-24-little-orrery-v2-sky-companion-design.md`
Workdir: `for_human_review_apps/Little Orrery` (rework in place, commit per phase)

## Phase 1 — Data & logic (no UI)
1. `SpaceData.swift`: add `tourText: String` to `World` + write 20 travelogue
   narrations (2–3 sentences each, wonder-voice not textbook-voice).
2. `SkyTonight.swift` (new):
   - `PlanetVisibility { world, status(evening/morning/allNight/hidden), hint }`
   - `SkyTonight.visibility(at: Date) -> [PlanetVisibility]` — geocentric
     elongation from heliocentric mean longitudes (Earth vs planet vectors).
   - `PersonalMath`: weight on world, age in local years, jump height, travel
     times (light 299792 km/s, rocket 58000 km/h, airliner 900 km/h) with smart
     formatting (min/h/days/years).
   - `RankTable`: 8 ranks (name, xp threshold); `rank(for:)`, `progress(for:)`.
3. `OrreryModels.swift`: state fields `weightKg: Double?`, `birthYear: Int?`,
   `tourStop: Int`, `tourDone: Bool`, `xp: Int`, `cometTapped: Bool` (tolerant
   decode); 6 new badges wired to store predicates.
4. `OrreryStore.swift`: mutations `setWeight/setBirthYear/advanceTour/resetTour/
   markCometTapped/addXP(reason)`; XP hooks into existing recordVisit,
   recordQuiz, addEntry, registerDailyOpen; expose `rank`, `xpValue`.
5. Build (sim) — expect clean; nothing visual yet.

## Phase 2 — Tonight tab rework
6. `DiscoverView.swift` → Tonight: hero card (date, moon disc, visible count),
   visibility list rows (art thumb + status chip + hint + footnote), Grand Tour
   teaser card (banner_tour, progress line), Object of the Day, quiz card, fun
   facts. Award Evening Watcher badge on appear when applicable.
7. `RootView.swift`: tab 2 label "Tonight"; `OrreryIcons.swift`: `.tonight`
   moon-star glyph, `.rocket` glyph, `.rank` emblem glyph.

## Phase 3 — Grand Tour
8. `TourView.swift` (new): custom pager (drag + buttons) over 20 stops in
   semi-major order; per stop: LivingGlobe, name, kind pill, odometer count-up
   (TimelineView), tourText, next-stop footer; progress dots; resume from
   `tourStop`; finish → ConfettiBurst + badge + completion card; close button.
9. Entries: Tonight teaser card opens `.fullScreenCover`; Orrery top bar rocket
   button ditto.

## Phase 4 — You & the Worlds
10. `MoreView.swift`: "About you" card — weight slider (kg/lb by metric),
    birth-year picker (custom wheel via Picker), clear button; saves to store;
    Know Thyself badge.
11. `WorldDetailView.swift`: "You on {World}" card — weight row + vs-Earth bar,
    age in local years, jump height, travel-time rows (skip for Earth: show a
    wink line "You are already here"); nudge link when profile empty.

## Phase 5 — Orrery style pass
12. `SkyEffects.swift` (new): `ConfettiBurst`, `ShootingStarsLayer` (timer-driven
    Canvas streaks), `Haptics` (impact/tick/success wrappers), `skyAppear`
    staggered-entrance modifier.
13. `OrreryView.swift`: asteroid-belt dust arc Canvas between Mars & Jupiter
    radii (seeded, slow phase drift); comet on e=0.82 ellipse with anti-solar
    gradient tail, tappable → info mini-card + `markCometTapped`; orbit trails
    (fading arc, only while playing); haptics on select/speed/badge; rocket
    button in top bar.
14. `JournalView.swift`: rank hero card (emblem, animated XP count-up, bar to
    next rank) above stats; badges grid now 22.
15. `QuizView.swift`: confetti on result screen when score ≥ 7.

## Phase 6 — Verify & deliver
16. Full grep hygiene: no SF Symbols/emoji/TabView/iOS16 API.
17. Build sim Debug + device Release (CODE_SIGNING_ALLOWED=NO) — 0 errors.
18. Sim run: seed profile+progress state; screenshot Tonight, Tour stop,
    World detail with You-card, Journal rank. Verify v1-save load (seed old JSON
    without new keys → app opens clean).
19. Size check ≥ 18 MB; version still 1.0 (1); update APP_TRACKER.md +
    APP_DESCRIPTIONS.md; git commit; keep app in for_human_review_apps.
