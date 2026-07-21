# Sound Grove "Living Grove" v2.0 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Upgrade the delivered Sound Grove app to feel markedly richer — living animated scenes reacting to the mix, fully re-illustrated richer art, a new immersive Now Playing mode, and +10 sounds / +6 presets.

**Architecture:** Keep existing SwiftUI + AVAudioPlayer architecture. Add a Canvas/TimelineView particle engine (`LiveScene.swift`) reused in Studio, Sleep and a new `ImmersiveView.swift` full-screen cover. Regenerate audio (`audiogen.swift`) and art (`artgen.swift`) via the existing Core Graphics/synth generators. Extend catalogs in `GroveModels.swift`.

**Tech Stack:** Swift 5, SwiftUI (iOS 15.6+), AVFoundation, Core Graphics generators run with `swiftc`.

**Working dir:** `for_human_review_apps/Sound Grove` (app has its own git repo). Verification = `swiftc` generator runs + `xcodebuild` compile + simulator screenshots (no XCTest harness exists; matches how v1 was built/verified).

## Global Constraints

- iOS 15.6+ only; no iOS 16+ APIs (no NavigationStack, `.tracking`, Charts, `.fontWeight` modifier, etc.).
- iPhone family (TARGETED_DEVICE_FAMILY=1); Portrait + Landscape L/R.
- Custom SwiftUI only — no SF Symbols, no emoji, no system icons.
- Fixed dusk-dark theme (UIUserInterfaceStyle=Dark); fully offline; no account/permissions/notifications.
- Local storage only (UserDefaults JSON). WebView gate unchanged (`GroveRedirectWatcher`, example.com).
- App icon stays opaque RGB, no alpha (`sips -g hasAlpha` must print `no`). App `<99` MB.
- Audio: 16-bit PCM mono WAV @44.1kHz, seamless loop via equal-power crossfade. Art: Core Graphics PNGs.
- Clean the `._*` AppleDouble files before every build. Generators kept in `art_src/` (binaries removed).
- Version → MARKETING_VERSION 2.0 / CURRENT_PROJECT_VERSION 2 (pbxproj + Info.plist).

---

### Task 1: Add 10 new procedural sounds

**Files:**
- Modify: `art_src/audiogen.swift` (add 10 `build(...)` blocks)
- Output: `Sound Grove/Audio/{rain_tent,blizzard,cat,clock,drips,rowboat,cicadas,wolves,pinknoise,omdrone}.wav`

**New sounds (id → synth sketch):**
- `rain_tent` — soft high-pass patter (lighter than `rain`), sparse taps.
- `blizzard` — brown+band noise with strong slow amplitude LFO (gusts), low howl partial.
- `cat` — periodic purr: low buzzy AM (~25 Hz) bursts in a slow inhale/exhale envelope.
- `clock` — steady tick/tock: short filtered click every 1.0 s, alternating pitch, faint room bed.
- `drips` — sparse water drips: bell blips (600–1400 Hz, short) at random 0.4–1.6 s + faint cave reverb bed.
- `rowboat` — periodic wooden creak (band-passed noise swell ~2 s) + gentle water lap bed.
- `cicadas` — dense daytime buzz: several detuned 5–7 kHz AM voices, near-constant.
- `wolves` — faint night bed + occasional low howl (gliding 300→500→350 Hz sine with vibrato, long env), spaced.
- `pinknoise` — pink-ish noise (cascaded LP of white), steady.
- `omdrone` — sustained vocal-ish drone: fundamental ~110 Hz + formant partials + slow beating.

- [ ] **Step 1:** Add the 10 `build("<id>", dur:…, fadeSec:…, peak:…, seed:…) { … }` blocks to `audiogen.swift`, reusing existing helpers (`LP`, `HP`, `addBell`, `addThump`, `addNote`, `softClip`, seeded `RNG`). Distinct seeds (2121…3030).
- [ ] **Step 2:** Compile: `cd art_src && swiftc -O audiogen.swift -o audiogen` — expect no errors.
- [ ] **Step 3:** Run: `./audiogen "../Sound Grove/Audio"` — expect all 30 `.wav` lines printed.
- [ ] **Step 4:** Verify: `afinfo "../Sound Grove/Audio/blizzard.wav"` prints `1 ch, 44100 Hz, Int16`; `ls Sound Grove/Audio/*.wav | wc -l` == 30.
- [ ] **Step 5:** Remove binary: `rm -f art_src/audiogen`. Commit (`git add -A && git commit`).

---

### Task 2: Richer art generator + new tokens/covers

**Files:**
- Modify: `art_src/artgen.swift` (depth on tokens/covers/banners/onboarding; add 10 tokens + 6 covers)
- Output: overwrite all `Sound Grove/Art/*.png`; new `tok_{rain_tent,blizzard,cat,clock,drips,rowboat,cicadas,wolves,pinknoise,omdrone}.png`, `scene_{snowycabin,reading,cave,cicadaday,wolfridge,temple}.png`

- [ ] **Step 1:** In `artgen.swift`, enrich `token(...)`: layered radial disc, inner-shadow ring, rim light, top glass sheen; enrich `cover(...)`: add foreground silhouette band + atmosphere bloom + vignette; enrich `banner(...)` and `onboard(...)`. Add motif `case`s for the 10 new ids in `drawMotif` (tent, snow/gust lines, cat face, clock face, water drip, boat, cicada, wolf silhouette/moon, pink-noise radial, om lotus). Extend `catOf` with the 10 ids and their categories.
- [ ] **Step 2:** Add the 6 new `cover(...)` calls with their mixes' key motifs.
- [ ] **Step 3:** Compile+run: `cd art_src && swiftc -O artgen.swift -o artgen && ./artgen "../Sound Grove/Art" "AppIcon-1024.png"` — expect Done.
- [ ] **Step 4:** Verify: `ls Sound Grove/Art/*.png | wc -l` == 55 (30 tokens + 5 banners + 16 covers + 3 onboarding + grain); `sips -g hasAlpha art_src/AppIcon-1024.png` == `no`; visually Read 3–4 samples (a new token, a new cover, onboarding) to confirm quality.
- [ ] **Step 5:** Do NOT reinstall the icon (unchanged emblem) unless regenerated; if regenerated, copy to `Assets.xcassets/AppIcon.appiconset/AppIcon-1024.png` and re-verify alpha. Remove `art_src/artgen` + `art_src/AppIcon-1024.png`. Commit.

---

### Task 3: LiveScene particle engine

**Files:**
- Create: `Sound Grove/LiveScene.swift`

**Interfaces:**
- Produces: `struct LiveSceneView: View { init(active: [GroveSound], volumes: [String: Double], playing: Bool, reduceMotion: Bool, intensity: CGFloat = 1) }`

- [ ] **Step 1:** Implement `LiveSceneView` using `TimelineView(.animation(paused: !playing || reduceMotion))` wrapping a `Canvas`. Draw layers back-to-front: dusk sky gradient tinted to dominant category, drifting moon + halo, seeded twinkling stars, atmosphere bloom, per-sound particle layers (rain streaks, embers, wave silhouettes, fireflies, motes, aura ripples, lightning flash), foreground hill silhouette. Density/alpha per layer ∝ that sound's volume; global particle cap ≤160. `reduceMotion` → single static frame (no time term). Deterministic seeded positions (no `Math.random` per-frame allocation churn).
- [ ] **Step 2:** Add helper `dominantCat(active:volumes:)` local or reuse a small inline computation (do not depend on store).
- [ ] **Step 3:** Compile-check via the app build in Task 8 (no standalone test). Commit after file compiles in Task 8's first build; for now `git add` the file.

---

### Task 4: Wire LiveScene into Studio stage + Immerse button

**Files:**
- Modify: `Sound Grove/StudioView.swift`

- [ ] **Step 1:** Replace the plain stage background with `LiveSceneView(active: store.activeSounds, volumes: store.mix, playing: store.isPlaying, reduceMotion: store.reduceMotion, intensity: 0.7)` clipped to the stage `RoundedRectangle`; keep floating tokens on top; keep empty-state prompt when no sounds.
- [ ] **Step 2:** Add an "Immerse" pill button (styled, uses `GroveIcon`) near the master bar that sets a `@State showImmersive = true`; present `.fullScreenCover(isPresented:) { ImmersiveView() .environmentObject(store) }`. Disabled/hidden when `!store.hasMix`.
- [ ] **Step 3:** Verify in Task 8/9 build + screenshot.

---

### Task 5: Immersive Now Playing view

**Files:**
- Create: `Sound Grove/ImmersiveView.swift`

**Interfaces:**
- Consumes: `GroveStore` via `@EnvironmentObject`; `LiveSceneView`.

- [ ] **Step 1:** Implement full-screen `ImmersiveView`: full-bleed `LiveSceneView(intensity: 1.2)`, a large glowing clock via `TimelineView(.periodic(from:.now, by:60))` formatted `H:mm` (24h-agnostic using `DateFormatter` short time), scene/blend name + active token row, auto-hiding control cluster (play/pause, sleep chips 15/30/60 calling `store.startTimer(mode:.sleep,minutes:)`, exit X dismiss). Tap toggles chrome; auto-hide after 4 s while playing via a `DispatchQueue.main.asyncAfter` guarded by a token. `.preferredColorScheme(.dark)`.
- [ ] **Step 2:** Verify in Task 8/9 build + screenshot.

---

### Task 6: Catalog — +10 sounds, +6 presets

**Files:**
- Modify: `Sound Grove/GroveModels.swift`

- [ ] **Step 1:** Append 10 `GroveSound(...)` entries to `SoundCatalog.all` (ids/names/cats/blurbs from spec §4), keeping 6 per category order.
- [ ] **Step 2:** Append 6 `ScenePreset(...)` entries to `SceneCatalog.all` (ids/names/blurbs/covers/cats/mixes from spec §4).
- [ ] **Step 3:** Verify ids match generated `tok_*`/`scene_*` files and `*.wav` files (grep). Compile in Task 8.

---

### Task 7: Sleep live background + immerse entry

**Files:**
- Modify: `Sound Grove/SleepView.swift`

- [ ] **Step 1:** Add a subtle `LiveSceneView(intensity: 0.5)` behind the setup content (below the `AuroraBackground` or replacing it), keeping the breathing orb on top.
- [ ] **Step 2:** Add an "Immerse" text button on the now-playing card that opens the same `.fullScreenCover { ImmersiveView() }` (local `@State`).
- [ ] **Step 3:** Verify in Task 8/9.

---

### Task 8: Project wiring, version bump, build

**Files:**
- Modify: `Sound Grove.xcodeproj/project.pbxproj` (add `LiveScene.swift`, `ImmersiveView.swift` to PBXFileReference, PBXBuildFile, group children, Sources phase; bump `MARKETING_VERSION=2.0`, `CURRENT_PROJECT_VERSION=2` in both configs)
- Modify: `Sound Grove/Info.plist` (`CFBundleShortVersionString`=2.0, `CFBundleVersion`=2)

- [ ] **Step 1:** Add the two new Swift files to pbxproj (unique `SG…` ids following existing scheme) and bump versions.
- [ ] **Step 2:** Clean AppleDouble: `find "Sound Grove" -name '._*' -delete; find . -name '.DS_Store' -delete`.
- [ ] **Step 3:** Build Debug: `xcodebuild -project "Sound Grove.xcodeproj" -scheme "Sound Grove" -destination 'platform=iOS Simulator,id=<iPhone16 18.2 id>' -derivedDataPath build/ build` — expect `BUILD SUCCEEDED`, 0 errors, 0 real warnings. Fix any iOS-16 API / type-check issues inline and rebuild.
- [ ] **Step 4:** Build Release similarly (`-configuration Release`) — expect success.
- [ ] **Step 5:** Verify bundle: `Art` count 55, `Audio` count 30, icon `AppIcon60x60` present, size `<99` MB. Commit.

---

### Task 9: Visual verification on simulator

**Files:** none (verification only)

- [ ] **Step 1:** Boot iPhone 16 sim, install, seed a mix via `simctl spawn … defaults write` (onb=true + activeMix with rain/campfire/ocean/cicadas), launch, screenshot Studio → confirm live scene renders under floating tokens.
- [ ] **Step 2:** Temporarily set `RootView` initial tab to reach each screen (or drive Immerse); screenshot Scenes (new covers), Sleep (live bg), and Immersive mode. Read each screenshot to confirm richness and no glitches/overlaps. Revert temp tab change; rebuild.
- [ ] **Step 3:** Verify reduce-motion path by toggling the stored `reduceMotion` pref and confirming the scene renders a static frame (no crash).

---

### Task 10: Finalize delivery

**Files:**
- Modify: `development/APP_TRACKER.md` (update Sound Grove row → v2.0)
- Modify: `APP_DESCRIPTIONS.md` (refresh Sound Grove line with new features)

- [ ] **Step 1:** Update tracker row (v2.0 build 2: living scene engine, immersive mode, 30 sounds, 16 presets, richer art, new sizes).
- [ ] **Step 2:** Refresh the description line.
- [ ] **Step 3:** Clean `._*`, remove `build/`/`build_rel/`; copy new screenshots to `screenshots/`. Commit in the app repo. Confirm app remains in `for_human_review_apps/Sound Grove`.

## Self-Review

- Spec coverage: §1 live engine → T3/T4/T7; §2 immersive → T5/T4/T7; §3 richer art → T2; §4 content → T1/T2/T6; §5 polish → T2/T4; versioning/verification → T8/T9/T10. All covered.
- Type consistency: `LiveSceneView(active:volumes:playing:reduceMotion:intensity:)` used identically in T4/T5/T7. Sound/preset ids consistent across T1/T2/T6.
- No placeholders: sound synth sketches, art layers, and file lists are concrete; counts (30 wav, 55 png) explicit.
