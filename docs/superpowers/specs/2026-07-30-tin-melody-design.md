# Tin Melody — Design Spec (2026-07-30)

## Concept
Music / Creativity. A pocket **music box workshop**: you punch holes in a paper
tape (piano-roll strip), then turn the crank — the cylinder pulls the tape past
the steel comb and plucks **your** melody. Collection of music boxes (different
timbres + looks) and a library of melodies (your own + classic public-domain
tunes). Unique in the set: Murmora *mixes ready-made sounds*; Tin Melody is
about *composing your own music* — a maker/instrument app, not a mixer.

## Core loop
1. Punch holes on the tape (15 note rows × N steps) — each punch previews its note.
2. Turn the crank: circular drag drives the tape at your speed (or auto-play at set tempo).
3. Tines flick + glow as holes pass the comb; melody plays with real music-box timbre.
4. Save melodies, load classics, unlock new boxes by making music.

## Musical model
- **15-note diatonic comb** (like a real 15-note movement): C4→C6 white keys
  (C D E F G A B C D E F G A B C). Diatonic-only = everything sounds pleasant.
- Tape: 16–96 steps (default 32), grid editor, step = eighth note. Tempo 40–160 BPM.
- Playback: crank-driven (angular velocity → tape speed, with inertia/flywheel
  smoothing) or motor (auto-play) mode. Loop toggle.
- Audio: pre-synthesized tine samples, 15 notes × 4 timbres (WAV, mono 44.1 kHz,
  ~2.2 s exponential decay). AVAudioEngine + rotating AVAudioPlayerNode pool for
  polyphony and fast retrigger. No mic/network; UIBackgroundModes not needed.

## Timbres (4 synth voices)
1. **Bright Tin** — classic music box: sharp ping, inharmonic partial ~4.2×, long shimmer.
2. **Warm Walnut** — softer attack, darker rolloff, woody body resonance.
3. **Glass Celesta** — pure glassy partials, slow vibrato shimmer.
4. **Velvet Kalimba** — plucky thumb-piano: strong fundamental, quick decay, buzz transient.

## Boxes (collection — 8 designs, each = look + timbre + palette)
| Box | Timbre | Unlock |
|---|---|---|
| Tin Postcard | Bright Tin | start |
| Walnut Heirloom | Warm Walnut | punch 100 holes |
| Rose Porcelain | Glass Celesta | save 3 melodies |
| Velvet Kalimba | Velvet Kalimba | play 10 minutes |
| Brass Voyager | Bright Tin (deep set −5 st) | complete 5 classics listens |
| Midnight Lacquer | Glass Celesta (dark set −7 st) | punch 500 holes |
| Snowglobe Waltz | Warm Walnut (music-box high +5 st) | save 8 melodies |
| Carousel Gold | Velvet Kalimba (bright +4 st) | 60 play minutes |
Variants are pitch-shifted sample sets rendered at generation time (no runtime DSP).

## Screens (custom tab bar, 4 tabs + onboarding)
1. **Workshop** — the star. Top: living music box scene (open lid, rotating
   cylinder with pins mirroring current tape, comb, sparkle plucks). Middle:
   punch-tape editor — horizontally scrolling grid, tap to punch/unpunch (haptic
   + note preview, punched hole shows paper-punch animation), playhead line,
   note-name gutter. Bottom: crank wheel (circular drag, flywheel), motor
   play/stop, tempo stepper, loop, steps +/-, clear, save (name sheet).
2. **Melodies** — My Melodies (cards: name, box used, bars, date; play/edit/
   duplicate/delete) + 10 built-in Classics (public-domain, pre-punched tapes
   with generated cover art): Twinkle Twinkle, Ode to Joy, Mary's Lamb,
   London Bridge, Frère Jacques, Oh Susanna, Amazing Grace, Auld Lang Syne,
   Brahms' Lullaby, Home Sweet Home. Load any classic into Workshop to remix.
3. **Boxes** — gallery of 8 illustrated boxes (locked ones show silhouette +
   unlock hint + progress bar), tap to select active box; detail sheet with big
   art, timbre demo arpeggio, story blurb.
4. **You** — stats tiles (holes punched, melodies saved, play minutes, classics
   heard), craft level ring (XP from punching/saving/playing), 12 badges grid,
   sound-on-punch toggle, reduce-motion toggle, Privacy Policy (WebView sheet),
   reset data.
Onboarding: 3 illustrated slides (punch → crank → collect), shown once.

## Look & feel
Warm antique parlor, **forced light** (UIUserInterfaceStyle=Light, theme-independent).
Palette: aged-cream paper #F5EDDC, walnut #4A3324, brass #C99A3C, deep teal
accent #2E6E6A, dusty rose #C4726B, ink #2B2118. Grain-paper texture overlay,
scalloped tape edges, riveted tin panels. All icons custom Path/Canvas glyphs —
no SF Symbols, no emoji. Serif-feel display via system serif design fonts.

## Art pack (generated, art_src/artgen.swift — Core Graphics, deterministic)
- 8 music-box illustrations (hero, ~1200px) + 8 thumb crops
- 10 classic-melody covers (scene medallions)
- 3 onboarding illustrations
- paper grain tile, tape texture, workshop bench banner
Target: art + audio ≥ 18 MB installed (min-18 MB rule), well under 99 MB.

## Icon
Abstract per rules: muted brass disc (coin-like) with thin comb-teeth notch ring
+ 2 tiny sparkle dots on deep-teal radial background. No literal music box, no
notes. Opaque RGB, no alpha (CGImageAlphaInfo.noneSkipLast, verify sips).

## Tech
- SwiftUI, iOS 15.6+, NavigationView only, UIScreen.main.bounds for metrics.
- Persistence: UserDefaults JSON (melodies, unlocks, stats, settings).
- WebView gate per template: `TinRedirectWatcher`, `tinPageReady`,
  `tinSourceLink` = https://example.com, check domain "example",
  `TinWebPanel`, `TinLoadingScreen`; Settings → Privacy opens TinWebPanel sheet.
- pbxproj: manual signing, TARGETED_DEVICE_FAMILY=1, iphoneos+iphonesimulator,
  no Catalyst/XR/Mac, Portrait + Landscape L/R, bundle `com.tinmelody.app`,
  v1.0 build 1.

## Uniqueness check
No existing app in APP_DESCRIPTIONS.md does composition/instrument mechanics.
Swap test: mechanics (punch-tape sequencer + crank-driven playback + timbre
collection) can't be reskinned from any delivered app — Murmora (ambient mixer),
Doodli (drawing), Paper Folds (origami) all differ in core verbs.
