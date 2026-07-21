# Sound Grove — "Living Grove" richness update (v2.0)

**Date:** 2026-07-21
**App:** Sound Grove (`for_human_review_apps/Sound Grove`, bundle `com.soundgrove.app`)
**Goal:** Make the app feel markedly richer, more premium and alive — richer art everywhere,
living animated scenes that react to the mix, a new immersive Now Playing mode, and more content.
No "poor" or geometric-looking visuals; nothing that reads as a cheap reference app.

## Constraints (unchanged from v1)

- iOS 15.6+, iPhone family, custom SwiftUI UI only (no SF Symbols, no emoji).
- Fixed dusk-dark theme (theme-independent), fully offline, no account/permissions/notifications.
- Local storage only (UserDefaults JSON). WebView gate per template. `<99` MB.
- All audio procedurally synthesized (`art_src/audiogen.swift`); all art via Core Graphics
  (`art_src/artgen.swift`). Generators kept in `art_src/`, binaries removed. `._*` cleaned before build.

## 1. Living scene engine (centerpiece)

New file `LiveScene.swift` — one reusable `LiveSceneView(active:volumes:playing:reduceMotion:intensity:)`
rendered with `Canvas` + `TimelineView` (iOS 15-safe). Layered, back-to-front:

1. **Sky** — dusk vertical gradient whose tint leans toward the dominant active category; a soft
   drifting moon with halo; 1–2 slow parallax cloud bands.
2. **Stars** — reuse the seeded twinkle field (denser, subtle parallax).
3. **Atmosphere glow** — large soft radial bloom in the dominant category colour, low opacity.
4. **Weather / particle layers** — each keyed to active sound ids, density/alpha ∝ that sound's volume:
   - `rain` / `heavyrain` / `rain_tent` → falling rain streaks + faint splash ticks near the base.
   - `thunder` → occasional brief full-frame lightning brighten (respects reduce-motion = off).
   - `wind` / `blizzard` / `leaves` → drifting motes / leaf specks blown horizontally.
   - `campfire` → warm pulsing glow at bottom-centre + rising ember sparks.
   - `ocean` → 2–3 animated wave silhouettes sweeping at the base with gentle swell.
   - `stream` / `drips` → shimmer glints.
   - `crickets` / `owl` / `frogs` / `wolves` → blinking fireflies.
   - `birds` / `cicadas` → dawn tint + small birds crossing occasionally.
   - `bowl` / `piano` / `omdrone` / noises → slow concentric aura ripples from centre.
5. **Foreground silhouette** — layered hill/tree/grass silhouette band at the bottom for depth
   (drawn in Canvas, parallax-tinted per dominant category).

Rules: total particle count capped (≈ ≤160 active) for perf; `reduceMotion` renders a single static
frame (no `TimelineView` animation); when `playing == false`, motion slows/stills. `intensity` scales
particle density so the same engine serves a compact stage and a full-screen hero.

Used in three places:
- **Studio stage** (compact) — replaces the near-empty stage; floating tokens sit on top of it.
- **Immersive Now Playing** (full-screen hero).
- **Sleep** (subtle, behind the orb).

## 2. Immersive Now Playing mode (new headline feature)

New file `ImmersiveView.swift`, presented full-screen (`.fullScreenCover`) from a prominent
"Immerse" control on Studio and a button on Sleep. Contents:
- Full-bleed `LiveSceneView` at high intensity.
- A large, softly-glowing current-time clock (updates each minute via `TimelineView`), the loaded
  scene / blend name, and the active-sound token row.
- Minimal auto-hiding control cluster: play/pause, quick sleep-timer chips (15/30/60), exit (X).
  Tap anywhere toggles chrome; chrome auto-hides after ~4 s while playing.
- Uses the existing store; no new persistence. Fully offline. Landscape-friendly layout.

## 3. Richer art re-illustration (regenerate all assets)

Upgrade `artgen.swift` to lusher output (keep vector fallbacks in-app):
- **Tokens** (`tok_<id>.png`, 768px): layered radial disc + inner shadow + rim light + top glass
  sheen; motifs redrawn multi-tone with small accents and soft shadows; per-category richer palette.
- **Scene covers** (`scene_<id>.png`, 1000×720): true depth stack — sky gradient, atmosphere bloom,
  moon/sun, star field, mid-ground silhouette, foreground silhouette, subject motif discs, grain,
  vignette; unique composition per scene.
- **Category banners** (`banner_<cat>.png`) — richer, layered.
- **Onboarding** (`onb_1..3.png`, 1000×1000) — 3 richer illustrations that preview the living scene.
- **Grain** tile kept. App icon kept (already strong).

## 4. More content

**Sounds → 30 total (+10, 6 per category).** New synth voices added to `audiogen.swift` + tokens:
- Rain & Sky: `rain_tent` "Rain on Tent", `blizzard` "Blizzard Wind"
- Fireside & Home: `cat` "Purring Cat", `clock` "Ticking Clock"
- Water & Places: `drips` "Cave Drips", `rowboat` "Rowboat Creaks"
- Forest & Night: `cicadas` "Summer Cicadas", `wolves` "Distant Wolves"
- Focus & Tones: `pinknoise` "Pink Noise", `omdrone` "Om Drone"

**Presets → 16 total (+6).** New covers + mixes:
- `snowycabin` "Snowy Cabin" (blizzard + campfire + wind) — sky
- `reading` "Reading Nook" (cat + clock + rain) — fire
- `cave` "Dripping Cave" (drips + brownnoise + wind) — water
- `cicadaday` "Cicada Afternoon" (cicadas + birds + leaves) — forest
- `wolfridge` "Wolf Ridge" (wolves + wind + owl) — forest
- `temple` "Temple Calm" (omdrone + bowl + drips) — tones

## 5. Polish

- Token micro-animation on the stage (rain drips inside token, fire flicker) via lightweight overlays.
- Springier card interactions, richer transitions, more haptics on key moments.
- `Immerse` entry point styled prominently on Studio (near master bar).

## Architecture / files touched

- New: `Sound Grove/LiveScene.swift`, `Sound Grove/ImmersiveView.swift`.
- Edit generators: `art_src/audiogen.swift` (+10 sounds), `art_src/artgen.swift` (richer everything +
  new tokens/covers). Regenerate into `Sound Grove/Audio` and `Sound Grove/Art`.
- Edit `GroveModels.swift` (catalog +10 sounds, +6 presets), `StudioView.swift` (live stage + Immerse
  button), `SleepView.swift` (live bg + immerse), `RootView.swift` / app wiring for full-screen cover.
- Bump: `Info.plist` `CFBundleShortVersionString` 2.0 / `CFBundleVersion` 2; pbxproj `MARKETING_VERSION`
  2.0 / `CURRENT_PROJECT_VERSION` 2. Add new Swift files to pbxproj Sources.

## Verification

- Clean `xcodebuild` Debug + Release (0 errors, 0 real warnings), iOS 15.6 API only.
- Icon still opaque (no alpha); Art + Audio folder refs bundle correctly; size `<99` MB.
- Visually verify on simulator: Studio live stage, Immersive mode, Scenes covers, Sleep, Profile —
  and confirm reduce-motion path renders static.
- Update `APP_TRACKER.md` + `APP_DESCRIPTIONS.md`; commit; app stays in `for_human_review_apps`.

## Out of scope (YAGNI)

- No new "moods/journeys" content type, no cloud, no sharing/export, no new permissions, no icon change.
