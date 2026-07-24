# Little Orrery v2 — Sky Companion Edition (design spec)

Date: 2026-07-24
App: Little Orrery (`com.littleorrery.app`), delivered v1.0 → rework in place in
`for_human_review_apps/Little Orrery`
Goal: kill the "nice but ordinary reference" feel. Turn the app into a daily sky
companion: it answers a real question every day (*what can I see tonight?*), it
knows *you* (your weight, your age on other worlds), it takes you on a cinematic
Grand Tour, and the orrery itself gets a living style pass (asteroid belt, comet,
shooting stars, haptics). Ships as v1.0 (build 1) per versioning rule.

## Problem

v1 is solid but reads like an encyclopedia with a pretty cover: the orrery is a
diagram you watch, Worlds is a list of fact sheets, Discover rotates a card. There
is no daily reason to open it, nothing personal, no narrative. The user asked for:
more style, more richness, functionality changes welcome — "полезная вещь, а не
справочник".

## Decisions (per Master's brief — direction confirmed in chat, details delegated)

1. Direction: **Sky Companion** — daily utility + personalization + narrative tour
   layered onto the existing model. All computed offline from the schematic
   ephemeris already in AstroMath.
2. No new PNG art required — comet, belt, rank emblems, tour chrome are all
   Canvas/vector (app already 22.3 MB ≥ 18 MB rule; stays < 99 MB).
3. All set constraints stay: iOS 15.6+, custom UI only, no SF Symbols/emoji,
   offline, no permissions, forced dark, version 1.0 (build 1), WebView gate
   untouched.

## 1. Tonight tab (Discover → renamed "Tonight", DiscoverView.swift reworked)

The tab becomes a genuine daily answer to "what's up tonight?".

- **Planet visibility engine** (`SkyTonight.swift`, pure logic): for each naked-eye
  planet (Mercury…Saturn) compute elongation = angular difference between the
  planet's and Sun's heliocentric longitudes as seen from Earth (geocentric approx
  from the existing mean-longitude ephemeris). Classify:
  - elongation < 12° → `hidden` ("Lost in the Sun's glare")
  - eastern elongation ≥ 12° → `evening` ("Evening sky, look west after sunset")
  - western elongation ≥ 12° → `morning` ("Morning sky, look east before dawn")
  - for outer planets, elongation > 150° → `allNight` ("Up all night — near opposition")
  Each row: planet art thumb, status chip (color-coded), one-line hint. This is
  approximate but honest — a footnote says "schematic model".
- **Sky Tonight hero card** at top: date, Moon phase disc (existing renderer),
  count of visible planets ("4 planets visible tonight").
- Object of the Day, Sky Quiz and fun-facts stay below (quiz card unchanged).
- New **Grand Tour teaser card** (uses the shipped-but-unused `banner_tour.png`)
  → opens the Tour (§3). Shows tour progress if started.

## 2. You & the Worlds (personalization)

- **Profile inputs** (More tab → "About you" card): weight (slider 20–200 kg /
  44–440 lb honoring metric setting) and birth year (wheel 1930–2026, optional).
  Stored in the existing state JSON (tolerant decode, no migration risk).
- **New section in WorldDetailView** — "You on {World}" GlassCard, shown always:
  - your weight there (weight × gravityG) with a bar vs Earth;
  - your age in local years (age_earth × 365.25 / orbitalPeriodDays) — "You'd be
    142 in Mercury years" (only if birth year set, else a nudge row);
  - jump height (Earth 40 cm baseline / gravityG);
  - travel time from Earth: light / rocket (New-Horizons-class 58,000 km/h) /
    airliner (900 km/h) — from |orbitDistanceKm − Earth's| for Sun-orbiters, or
    the moon's own distance for moons. Formatted smartly (minutes → years).
  If no profile: weight/age rows show a "Set up in More" link instead of numbers.
- **Tonight card** "Your numbers" teaser on a random world of the day.

## 3. Grand Tour (new `TourView.swift`, full-screen cover)

A guided, cinematic outward journey: Sun → Mercury → … → Eris (20 stops in
semi-major-axis order, moons attached after their parent planet).

- Full-screen pager (TabView page style is banned → custom drag/buttons pager):
  starfield bg, big LivingGlobe of the stop, distance-from-Sun "odometer" line
  that counts up between stops, 2–3 sentence narration (new `tourText` strings in
  SpaceData — hand-written, travelogue voice, not encyclopedia voice), "next stop"
  button with the next world's silhouette.
- Progress dots + "Stop 7 of 20"; progress persisted (`tourStop` in state) so the
  tour resumes; finishing → confetti + new badge + tour completion card.
- Entry points: Tonight teaser card + a small rocket Glyph button on the Orrery
  top bar.

## 4. Orrery style pass (OrreryView)

- **Asteroid belt**: seeded dust arc (Canvas, ~90 tiny dots with per-dot phase
  drift) between Mars and Jupiter radii; only when Dwarfs toggle is on it gains a
  "Ceres" label context. Cheap: one Canvas layer.
- **Comet**: one long-period comet on a visibly eccentric ellipse (e≈0.82, period
  ~76y — Halley-ish), drawn as a glowing head + tail that always points away from
  the Sun, length/opacity scaling near perihelion. Tappable → mini info card
  ("Comet Aurelia — a schematic long-period visitor") + counts toward a badge.
- **Shooting stars**: every 20–40 s a brief streak across a random screen corner
  (0.6 s, Canvas particle) — pure ambience.
- **Haptics**: light impact on planet select, soft tick when changing speed,
  success notification on badge toast (UIFeedbackGenerator wrappers, no
  permissions).
- **Trails**: faint arc behind each planet along its orbit (last ~40° of travel,
  gradient fade) — makes motion readable even when paused… trails only while
  playing.

## 5. Progression: Explorer ranks (OrreryModels/Store + Journal)

- `xp` accumulates: +4 first visit of a world, +2 per correct quiz answer,
  +6 per tour stop completed, +3 per journal note, +5 first comet tap, +2 per
  daily open (existing dailyDays hook).
- 8 ranks: Stargazer (0) → Moonwatcher (30) → Sky Sketcher (70) → Orbit Tracer
  (130) → Planet Hopper (210) → Comet Chaser (320) → Voyager (460) → Cosmographer
  (640). Rank emblem = Canvas glyph (ringed orb variants) — no PNGs.
- Journal Progress gains a rank hero card (emblem, XP bar to next rank, rank
  name) and the stats grid moves below it.
- 6 new badges (total 22): Grand Tourist (finish tour), First Steps Out There
  (first tour stop), Comet Spotter, Know Thyself (set both profile fields),
  Evening Watcher (open Tonight when ≥1 planet is evening-visible), Cosmographer
  (reach top rank).

## 6. Style pass (global)

- Staggered fade/slide-in appearances on Tonight and Journal cards (`skyAppear`
  modifier, respects reduce-motion by simply always animating short).
- Animated count-up numbers on Journal stats and the tour odometer
  (TimelineView-driven).
- Sky Quiz result screen gains confetti (reusable `ConfettiBurst` Canvas view,
  also used on tour finish and badge toasts).
- Tab bar label "Discover" → "Tonight"; glyph swaps to a moon-with-star.

## Non-goals

- No real astronomical precision upgrades (stays schematic, honestly labeled).
- No notifications, no location, no AR, no new PNG assets, no compare mode.

## State & compatibility

All new fields (`weightKg`, `birthYear`, `tourStop`, `tourDone`, `xp`,
`cometTapped`, plus badge ids) are optional-decoded additions to
`little.orrery.state.v1` via the existing tolerant `init(from:)` — v1 saves load
cleanly. No key renames.

## File plan

- new: `SkyTonight.swift` (visibility engine + personal-numbers math + rank table)
- new: `TourView.swift` (pager, odometer, narration)
- new: `SkyEffects.swift` (ConfettiBurst, ShootingStars, Haptics, skyAppear)
- edit: `SpaceData.swift` (+`tourText` per world), `OrreryModels.swift` (badges,
  state fields), `OrreryStore.swift` (xp/rank/profile/tour mutations),
  `DiscoverView.swift` (Tonight rework), `WorldDetailView.swift` (You-on-world),
  `OrreryView.swift` (belt/comet/trails/shooting stars/haptics/tour button),
  `JournalView.swift` (rank hero), `MoreView.swift` (About you), `RootView.swift`
  (tab label/glyph), `OrreryIcons.swift` (rocket, moon-star, rank emblem glyphs).
- pbxproj: +3 source files.

## Acceptance

Build clean (sim Debug + device Release), all six previous screens still render,
Tonight shows a correct-looking visibility list, tour completes end-to-end with
resume, personal numbers react to profile, v1 seeded save loads, app ≥ 18 MB,
version stays 1.0 (1). Screenshots refreshed (Tonight, Tour, You-on-world, rank).
