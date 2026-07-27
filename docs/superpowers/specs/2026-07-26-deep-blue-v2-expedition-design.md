# Deep Blue v2 — Expedition Edition (design spec)

Date: 2026-07-26
App: Deep Blue (`com.deepbluedive.app`), delivered v1.0 → rework in place in
`for_human_review_apps/Deep Blue`
Goal: kill the "pretty but passive atlas" feel. Turn the continuous-scale ocean
dive into a real *expedition*: you pilot a vessel that limits how deep you can go,
the camera sinks for you through darkness lit only by your lamp, a daily objective
gives you a reason to open the app, and ranks unlock the machines that reach the
trench. Ships as **v1.0 (build 1)** per the versioning rule.

## Problem

v1 is honest and good-looking, but the loop is passive: the user scrolls 11 km by
hand and taps silhouettes. Nothing pulls them back tomorrow, nothing is *theirs*,
and there is no sense of descent — only of paging through an atlas. Master's brief:
make it stylish, rich and genuinely useful; functionality changes are welcome.

## Decisions (confirmed with Master in chat)

1. Direction: **full "Expedition" package** — daily loop + progression + cinematic
   descent + personalization + style pass.
2. Main loop: **add a Descent mode with vessels and objectives**; the free scroll
   column stays (it is the app's signature).
3. Content/art: **maximum** — grow to 50 creatures and add new art (biome scenes,
   vessels, rank emblems). Target ~50–55 MB.
4. **iPad support** (added by Master): adaptive layout, `TARGETED_DEVICE_FAMILY = "1,2"`.
5. **Publish to a new public GitHub repo** at the end of the work (added by Master).
6. Unchanged constraints: iOS 15.6+, custom UI only (no SF Symbols / emoji / system
   components), forced dark, fully offline, no notifications, no permissions, manual
   signing, WebView launch gate untouched, **version 1.0 (build 1)**.

---

## 1. Descent Mode (new heart) — `DescentView.swift`, `VesselData.swift`

The free-scroll column stays as the "Explore" mode. A second, cinematic mode is
added and becomes the app's headline feature.

### 1.1 Vessels

Five vessels, each with a real-world depth limit, sink speed and lamp radius.
The vessel *is* the progression gate: to see the trench you must earn it.

| id | Name | Limit | Sink speed | Lamp radius | Unlocked at rank |
|----|------|-------|-----------|-------------|------------------|
| `mask` | Snorkel Mask | 30 m | 12 m/s | none (daylight) | 1 (start) |
| `scuba` | Scuba Rig | 200 m | 20 m/s | wide, soft | 2 |
| `bathysphere` | Bathysphere | 1,000 m | 35 m/s | medium | 3 |
| `submersible` | Deep Submersible (*Alvin* class) | 4,500 m | 60 m/s | narrow, bright | 5 |
| `fullocean` | Full-Ocean Sub (*Limiting Factor* class) | 11,000 m | 90 m/s | narrow, bright | 7 |

Each vessel carries: display name, real-world note (one honest sentence, e.g.
"Alvin has carried scientists since 1964 and reaches 4,500 m"), a vector silhouette
(new art PNG), limit, speed, lamp radius, accent colour.

A **vessel picker** sheet lists all five: unlocked ones tappable, locked ones dimmed
with "Reach rank N" and a lock glyph. Selected vessel persists.

### 1.2 The descent itself

- Camera sinks automatically. Controls: **pause / 1× / 2×** and an **Ascend** button.
- Depth counter animates upward; the hull gives a subtle periodic shudder (a small
  offset oscillation, disabled when the motion setting is off).
- **Lamp cone**: darkness ramps in linearly between 400 m and 1,000 m (0 → full), so
  the transition is gradual rather than a hard switch; below 1,000 m the scene is
  near-black except a radial light mask centred on the viewport. Creatures swim
  *into* the cone out of darkness. This is the "wow" v1 lacks. Implemented as a
  radial-gradient overlay whose radius comes from the current vessel, softened by `blur`.
- **Depth limit**: on reaching the vessel's limit the descent stops with a "Hull
  limit reached" card offering *Ascend* or *Change vessel*; if a better vessel is
  still locked, it shows which rank unlocks it — a concrete pull to keep playing.
- Creatures encountered during descent are tappable exactly as in Explore mode
  (same `CreatureDetailView`), and the descent pauses while the sheet is open.
- Reaching a new personal best depth records it and awards XP.

### 1.3 Sonar ping

A ping button emits expanding rings (Canvas) for ~3 s and highlights **undiscovered**
creatures within ±1,500 m of the current depth with a pulsing marker ring.
This fixes a real v1 problem: the user scrolls and cannot tell where to look.
Available in both Descent and Explore modes. No cooldown gimmicks — it is a comfort
feature, not a resource.

### 1.4 Daily Expedition — `ExpeditionEngine.swift`

Deterministically seeded from the calendar day (no notifications, no network — it
simply differs when the app is opened on a new day). One objective per day, drawn
from templates:

- `reachDepth(d)` — "Descend to 6,000 m"
- `findInZone(zone, n)` — "Find 3 creatures of the Midnight Zone"
- `findBioluminescent(n)` — "Meet 2 creatures that make their own light"
- `findAny(n)` — "Identify 4 new creatures"
- `visitZone(zone)` — "Enter the Hadal Zone"

Objectives are checked against live store state (they complete retroactively within
the same day). Completion: card flips to a completed state, +15 XP, haptic, and the
day is stamped into an expedition history used by the Journal. Progress for the
current day is persisted so it survives relaunch.

### 1.5 Dive tab structure

The Dive tab becomes a small hub instead of a bare column:

```
[ Daily Expedition card — objective, progress bar, XP reward ]
[ Begin Descent — primary button, shows current vessel silhouette + limit ]
------------------------------------------------------------------
[ the existing free-scroll water column, unchanged in spirit ]
```

`DescentView` presents as a `fullScreenCover`.

---

## 2. Progression: diver ranks + XP — `RankSystem.swift`

Eight ranks, each with a Canvas emblem and an XP threshold:

| # | Rank | XP | Unlocks |
|---|------|----|---------|
| 1 | Snorkeler | 0 | Snorkel Mask |
| 2 | Free Diver | 40 | Scuba Rig |
| 3 | Scuba Diver | 100 | Bathysphere |
| 4 | Technical Diver | 180 | — |
| 5 | Submersible Pilot | 300 | Deep Submersible |
| 6 | Deep Explorer | 450 | — |
| 7 | Trench Pioneer | 650 | Full-Ocean Sub |
| 8 | Master of the Abyss | 900 | — |

XP sources: discover a creature **+5**; new personal-best depth **+3 per 500 m
crossed**; daily expedition **+15**; correct quiz answer **+2**; first entry into a
zone **+10**.

The Journal gains a **rank hero card**: emblem, animated XP count-up, progress bar
to the next rank, and the unlock hint ("Deep Submersible at Submersible Pilot").
Rank-ups raise a toast + haptic.

The existing 12 awards stay; 6 more are added for the new systems (First Descent by
vessel, expedition streak, all vessels unlocked, sonar user, 50 creatures, max rank)
→ 18 total.

---

## 3. "You at depth" — personalization

Optional profile in More → *About you*: height (cm/in) and weight (kg/lb), honoring
a metric toggle. Both optional; nothing breaks if empty.

- **In the descent HUD**: a line that makes pressure bodily rather than abstract —
  at 300 m "4.2 tonnes press on your chest", at 3,000 m "like 8 elephants on your
  palm". Computed from body surface area (Du Bois formula from height/weight) ×
  pressure at depth; phrased in friendly comparisons.
- **In `CreatureDetailView`**: "this animal is 3.4× longer than you" with a simple
  comparison bar, derived from the creature's size string (a parsed numeric length
  is added to the creature model rather than re-parsing prose).

If the profile is empty, these rows render a soft "Set up in More" link instead.

---

## 4. Living ocean (style pass) — `DeepEffects.swift` extensions

The changes that most remove the "reference book" feel:

- **Creatures swim**: gentle bobbing, sine drift, slight tilt — each seeded per
  creature id so they do not move in lockstep. Currently they are nailed to their depth.
- **Parallax**: three particle layers at different speeds for scene depth.
- **Discovery moment**: light flash, expanding rings, name sliding up, haptic.
  Currently it is just a colour change.
- **Bioluminescence breathes** in the dark; lamp light adds a specular sheen on bodies.
- **Darkness vignette** tightens as you descend.

All of it respects the existing *Motion & bubbles* setting — with it off, everything
renders static.

---

## 5. Content & art (+~20 MB)

### 5.1 +14 creatures → 50 total

whale shark, orca, hammerhead shark, chambered nautilus, coelacanth, oarfish,
frilled shark, goblin shark, Japanese spider crab, siphonophore, comb jelly,
glass squid, sea angel, giant clam.

Each gets the full data record (sci name, zone, appear depth, real habitat range,
size, diet, 3 facts, bioluminescent flag, accent) plus a generated cutout and a
portrait card, exactly like the existing 36.

### 5.2 New art

- **6 biome scenes**: coral reef, kelp forest, whale fall, hydrothermal vents,
  brine pool, trench floor — used as richer zone backdrops and biome cards.
- **5 vessel silhouettes** for the picker and the Begin Descent button.
- **8 rank emblems** — Canvas-drawn, no PNG.

All generated by extending `art_src/generate_art.py` (pure PIL, no numpy), opaque
RGB for cards/scenes, transparent cutouts for creatures. Final app ~50–55 MB
(≥18 MB rule satisfied, well under 99 MB).

---

## 6. iPad support (new)

- `TARGETED_DEVICE_FAMILY = "1,2"` in **both** Debug and Release.
- Adaptive layout driven by `horizontalSizeClass`:
  - Journal / awards grids: 2 columns on compact, 4 on regular.
  - Content columns capped with `.frame(maxWidth: 700)` and centred so text does not
    stretch across a 12.9" screen.
  - Dive column: lane positions and the left rail use fractions of the available
    width (already the case), verified at iPad widths; creature art scales up with
    size class.
  - Descent HUD and vessel picker centred with a max width.
- Orientation stays Portrait + Landscape Left/Right (already in Info.plist).
- Verified by building and screenshotting on an iPad simulator.

Note: the ios-builder skill's checklist contains both "Adaptive layout: iPhone +
iPad" and "TARGETED_DEVICE_FAMILY = 1". Master's explicit instruction to support
iPad resolves that contradiction in favour of `"1,2"`.

---

## 7. Architecture & files

New:
- `VesselData.swift` — vessel model + the five vessels.
- `DescentView.swift` — cinematic descent, lamp cone, HUD, hull-limit card.
- `ExpeditionEngine.swift` — daily objective generation + evaluation.
- `RankSystem.swift` — ranks, XP thresholds, emblem drawing.
- `SonarLayer.swift` — ping rings + undiscovered-creature highlighting.

Extended:
- `OceanData.swift` — +14 creatures, +6 awards, biome records, creature length field.
- `DeepStore.swift` — `xp`, `currentVessel`, `profileHeight`, `profileWeight`,
  `metric`, `expeditionDay`, `expeditionProgress`, `expeditionHistory`, `zonesVisited`.
  **Tolerant decode**: all new fields via `decodeIfPresent` with defaults so existing
  v1 saves load cleanly.
- `DeepEffects.swift` — swim motion, parallax, discovery burst, lamp cone.
- `DiveView.swift` — hub layout (expedition card + Begin Descent), sonar, swim motion.
- `JournalView.swift` — rank hero card, expedition history.
- `MoreView.swift` — About you profile, metric toggle, vessel list, 18 awards.
- `CreatureDetailView.swift` — size-vs-you comparison.
- Adaptive-layout pass across all grid/content views.

Each new file has one clear job and stays independently readable; `DescentView` owns
presentation only, with vessel/expedition/rank rules living in their own pure-logic
files so they can be reasoned about without SwiftUI.

---

## 8. Verification

- Clean `xcodebuild` on an iPhone simulator **and** an iPad simulator: zero errors,
  zero code warnings.
- Runtime verification via `xcrun simctl` (per the established workflow): install,
  launch, seed the app container plist, screenshot each screen — Dive hub, Descent
  mid-water, Descent in lamp-lit darkness, vessel picker, Journal rank card, More
  profile, creature detail with size comparison, plus one iPad screenshot.
- v1 save-compatibility check: seed a v1-shaped state blob and confirm it loads with
  new fields defaulted.
- Icon still opaque (`sips -g hasAlpha` → no); `AppIcon60x60` present in the built
  Info.plist.
- Temporary verification hooks removed before the final clean build (grep-verified).

---

## 9. Delivery

- Rework happens in place in `for_human_review_apps/Deep Blue`.
- Version stays **1.0 (build 1)**; no bump.
- `APP_TRACKER.md` and `APP_DESCRIPTIONS.md` rows updated to describe v2.
- Screenshots refreshed in `screenshots/`.
- Final step: create a **new public GitHub repository** and push the app there,
  using the token in `~/.config/github-token` (no `gh` CLI). The repo contains the
  app source, art generator and screenshots — no build artefacts.

## Out of scope

- Notifications of any kind (banned).
- Networked or cloud features; everything stays offline and on-device.
- Any resource/failure mechanic (oxygen timers, hull breach): the app stays calm and
  educational rather than becoming a survival game.
- Rewriting the free-scroll column's depth mapping — the power-curve scale from v1
  is correct and stays.
