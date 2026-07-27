# Deep Blue v2 — Expedition Edition Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn Deep Blue from a passive scroll-atlas into an expedition: a cinematic vessel-gated descent through lamp-lit darkness, a daily objective, diver ranks, body-scale personalization, a living animated ocean, 50 creatures, and iPad support — shipped as v1.0 (build 1) and published to a new public GitHub repo.

**Architecture:** Additive layering on the existing SwiftUI app in `for_human_review_apps/Deep Blue`. Pure-logic files (`VesselData`, `RankSystem`, `ExpeditionEngine`, `BodyScale`) hold all rules and are independently readable; SwiftUI views consume them. `DeepStore` gains new fields via tolerant `decodeIfPresent` so v1 saves keep loading. Art is extended in the existing pure-PIL generator.

**Tech Stack:** Swift 5 / SwiftUI (iOS 15.6 floor), Canvas + Shape custom drawing, UserDefaults JSON persistence, Python 3 + Pillow for art generation, `xcodebuild` + `xcrun simctl` for build/verification.

## Global Constraints

Copied verbatim from the spec and the ios-builder rules. **Every task's requirements implicitly include this section.**

- **iOS 15.6+ only.** No iOS 16+ APIs: no `NavigationStack`, no `Charts`, no `.contentTransition`, no `.scrollContentBackground`, no `Font.italic()` (use `Text.italic()`), no `AnyShapeStyle` init tricks. Use `NavigationView` + `StackNavigationViewStyle`.
- **Version stays `MARKETING_VERSION = 1.0`, `CURRENT_PROJECT_VERSION = 1`.** Never bump.
- **Custom UI only** — no SF Symbols, no system icons, no emoji anywhere in UI. All glyphs are `Shape`/`Canvas` drawings.
- **No SwiftUI `TabView` with custom icons** — the app's custom `HStack` tab bar in `RootView.swift` stays.
- **Theme-independent** — forced dark via `Info.plist UIUserInterfaceStyle=Dark` + `.preferredColorScheme(.dark)`.
- **Fully offline.** Local storage only (UserDefaults). No network, no cloud, **no notifications**, no permissions.
- **English (US) only** in all UI copy.
- **No UI overlaps or hidden layers.**
- **App size ≥ 18 MB, < 99 MB.** Target ~50–55 MB.
- **App icon stays opaque RGB, no alpha** (`sips -g hasAlpha` must print `hasAlpha: no`).
- **Bundle id stays `com.deepbluedive.app`.** Manual signing, `CODE_SIGN_STYLE = Manual`.
- **`TARGETED_DEVICE_FAMILY = "1,2"`** (iPhone + iPad) in **both** Debug and Release — Master's explicit instruction.
- **WebView launch gate untouched** (`DeepBlueApp.swift`, `DeepWebPanel.swift`, `DeepRedirectScout`).
- **No extra files in the app folder** — no README/summary markdown, no .sh/.py outside `art_src/`.
- **Build command:** iPhone `-destination 'platform=iOS Simulator,name=iPhone 17'`, iPad `-destination 'platform=iOS Simulator,name=iPad Pro 13-inch (M4)'` (verify exact name with `xcrun simctl list devices available`).
- **Builds are slow** (external USB drive, ~8–10 min clean): run `xcodebuild` in the background and poll the log.
- **No automated test framework exists in this project.** "Tests" in each task mean: a clean `xcodebuild` compile plus a stated runtime verification via `xcrun simctl` screenshot or a `swift` logic check. Follow each task's stated verification exactly.

**Working directory for all tasks:** `/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue`

---

## File Structure

**Create (app target — each must be added to `project.pbxproj` in its task):**

| File | Responsibility |
|---|---|
| `Deep Blue/VesselData.swift` | `Vessel` model + the five vessels. Pure data. |
| `Deep Blue/RankSystem.swift` | `DiverRank` model, 8 ranks, XP thresholds, rank lookup. Pure logic. |
| `Deep Blue/ExpeditionEngine.swift` | `Expedition` model, day-seeded generation, progress evaluation. Pure logic. |
| `Deep Blue/BodyScale.swift` | Body-surface-area + pressure-on-you math, friendly comparison strings. Pure logic. |
| `Deep Blue/DescentView.swift` | Cinematic descent screen: auto-sink, lamp cone, HUD, hull-limit card. |
| `Deep Blue/VesselPickerView.swift` | Vessel selection sheet with locked/unlocked states. |
| `Deep Blue/SonarLayer.swift` | Sonar ping rings + undiscovered-creature highlight ring. |
| `Deep Blue/RankEmblem.swift` | Canvas-drawn rank emblems (8 variants) + rank hero card. |
| `Deep Blue/Adaptive.swift` | Size-class helpers: content max width, grid column counts. |

**Modify:**

| File | Change |
|---|---|
| `Deep Blue/OceanData.swift` | +14 creatures (→50), +6 awards (→18), 6 biome records. |
| `Deep Blue/OceanModels.swift` | `Creature.lengthMeters`, `Biome` model. |
| `Deep Blue/DeepStore.swift` | +9 persisted fields, XP/rank/vessel/expedition/profile API, tolerant decode. |
| `Deep Blue/DeepEffects.swift` | Swim motion, parallax layers, discovery burst, lamp-cone overlay. |
| `Deep Blue/DiveView.swift` | Hub layout (expedition card + Begin Descent), sonar, swimming creatures. |
| `Deep Blue/JournalView.swift` | Rank hero card, expedition history, adaptive grid. |
| `Deep Blue/MoreView.swift` | About-you profile, metric toggle, vessel list, 18 awards, adaptive grid. |
| `Deep Blue/CreatureDetailView.swift` | Size-vs-you comparison row. |
| `Deep Blue/ZonesView.swift` | Biome cards, adaptive content width. |
| `Deep Blue.xcodeproj/project.pbxproj` | 9 new source files; `TARGETED_DEVICE_FAMILY = "1,2"`. |
| `art_src/generate_art.py` | +14 creature drawers, 6 biome scenes, 5 vessel silhouettes. |

---

## Task 1: Data model foundations — creature length, biomes, vessels

**Files:**
- Modify: `Deep Blue/OceanModels.swift`
- Create: `Deep Blue/VesselData.swift`
- Modify: `Deep Blue.xcodeproj/project.pbxproj`

**Interfaces:**
- Consumes: existing `Creature`, `DepthZone` from `OceanModels.swift`.
- Produces:
  - `Creature.lengthMeters: Double` (stored property, added to the memberwise init — every existing literal in `OceanData.swift` must pass it).
  - `struct Biome: Identifiable { let id: String; let name: String; let depth: Int; let art: String; let blurb: String; let accent: Color }`
  - `struct Vessel: Identifiable { let id: String; let name: String; let limit: Int; let sinkSpeed: Double; let lampRadius: CGFloat; let unlockRank: Int; let note: String; let accent: Color; let art: String }`
  - `enum Vessels { static let all: [Vessel]; static func byID(_ id: String) -> Vessel; static let defaultID = "mask" }`

- [ ] **Step 1: Add `lengthMeters` and `Biome` to `OceanModels.swift`**

In `Deep Blue/OceanModels.swift`, inside `struct Creature`, add the stored property immediately after `let size: String`:

```swift
    let lengthMeters: Double     // typical adult length, for the size-vs-you comparison
```

Then append this new type at the end of the file (after the `OceanPhysics` enum):

```swift
// MARK: - Biome (a distinctive deep-sea habitat)

struct Biome: Identifiable {
    let id: String
    let name: String
    let depth: Int          // representative depth, metres
    let art: String         // scene_<id>.png
    let blurb: String
    let accent: Color
}
```

- [ ] **Step 2: Create `Deep Blue/VesselData.swift`**

```swift
import SwiftUI

// The five craft the player descends in. Each vessel gates how deep the
// Descent mode can go, how fast it sinks and how far its lamp reaches.

struct Vessel: Identifiable {
    let id: String
    let name: String
    let limit: Int          // metres the craft can reach
    let sinkSpeed: Double   // metres per second during descent
    let lampRadius: CGFloat // points; 0 = no lamp (daylight only)
    let unlockRank: Int     // rank number required (see RankSystem)
    let note: String        // one honest real-world sentence
    let accent: Color
    let art: String         // vessel_<id>.png

    var limitLabel: String { "\(limit.formattedDepth) m" }
}

enum Vessels {
    static let defaultID = "mask"

    static let all: [Vessel] = [
        Vessel(id: "mask", name: "Snorkel Mask", limit: 30, sinkSpeed: 12,
               lampRadius: 0, unlockRank: 1,
               note: "Nothing but a mask and a breath — the sunlit shallows are as far as you go.",
               accent: Sea.aqua, art: "vessel_mask"),
        Vessel(id: "scuba", name: "Scuba Rig", limit: 200, sinkSpeed: 20,
               lampRadius: 300, unlockRank: 2,
               note: "Compressed air opens the whole sunlit zone, down to where the light gives out.",
               accent: Sea.teal, art: "vessel_scuba"),
        Vessel(id: "bathysphere", name: "Bathysphere", limit: 1_000, sinkSpeed: 35,
               lampRadius: 230, unlockRank: 3,
               note: "A steel ball on a cable — the 1930s design that first carried people into the dark.",
               accent: Sea.gold, art: "vessel_bathysphere"),
        Vessel(id: "submersible", name: "Deep Submersible", limit: 4_500, sinkSpeed: 60,
               lampRadius: 180, unlockRank: 5,
               note: "An Alvin-class craft; the real one has carried scientists since 1964 and reaches 4,500 m.",
               accent: Sea.violet, art: "vessel_submersible"),
        Vessel(id: "fullocean", name: "Full-Ocean Sub", limit: 11_000, sinkSpeed: 90,
               lampRadius: 150, unlockRank: 7,
               note: "A full-ocean-depth titanium sphere — the class of craft that has visited Challenger Deep.",
               accent: Sea.coral, art: "vessel_fullocean"),
    ]

    static func byID(_ id: String) -> Vessel {
        all.first { $0.id == id } ?? all[0]
    }
}
```

- [ ] **Step 3: Add `lengthMeters` to all 36 existing creature literals**

Every `Creature(...)` literal in `Deep Blue/OceanData.swift` needs the new argument between `size:` and `diet:`. Use these values (metres, typical adult length):

```
man_o_war 0.3 · moon_jelly 0.4 · herring 0.35 · sea_turtle 1.2 · sailfish 3.0
great_white 4.6 · manta 5.5 · dolphin 2.2 · sunfish 2.0 · lanternfish 0.1
hatchetfish 0.07 · cookiecutter 0.5 · snipe_eel 1.0 · giant_squid 12.0
vampire_squid 0.3 · blobfish 0.3 · sperm_whale 16.0 · colossal_squid 10.0
gulper_eel 1.0 · dragonfish 0.4 · anglerfish 0.18 · fangtooth 0.18
barreleye 0.15 · giant_isopod 0.4 · dumbo_octopus 0.25 · yeti_crab 0.15
tube_worm 2.4 · atolla_jelly 0.15 · grenadier 1.0 · tripod_fish 0.35
sea_pig 0.15 · sea_spider 0.7 · snailfish 0.25 · amphipod 0.05
hadal_cucumber 0.15 · xenophyophore 0.2
```

Example of the edit shape (apply the same pattern to all 36):

```swift
        Creature(id: "great_white", name: "Great White Shark", sci: "Carcharodon carcharias",
                 zone: 0, appear: 100, rangeTop: 0, rangeBottom: 1200,
                 size: "Up to 6 m", lengthMeters: 4.6, diet: "Seals, fish, other sharks",
```

- [ ] **Step 4: Register `VesselData.swift` in the Xcode project**

In `Deep Blue.xcodeproj/project.pbxproj` add one entry to each of the four sections, following the existing `DB00000000000000000000NN` id convention:

`PBXBuildFile`:
```
		DB0000000000000000000130 /* VesselData.swift in Sources */ = {isa = PBXBuildFile; fileRef = DB0000000000000000000030 /* VesselData.swift */; };
```

`PBXFileReference`:
```
		DB0000000000000000000030 /* VesselData.swift */ = {isa = PBXFileReference; lastKnownFileType = sourcecode.swift; path = VesselData.swift; sourceTree = "<group>"; };
```

Group children (inside `DB0000000000000000000302 /* Deep Blue */`), after `MoreView.swift`:
```
				DB0000000000000000000030 /* VesselData.swift */,
```

`PBXSourcesBuildPhase` files list:
```
				DB0000000000000000000130 /* VesselData.swift in Sources */,
```

- [ ] **Step 5: Compile**

Run:
```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue" && xcodebuild -project "Deep Blue.xcodeproj" -scheme "Deep Blue" -destination 'platform=iOS Simulator,name=iPhone 17' -derivedDataPath build/ build > /tmp/t1.log 2>&1; echo "EXIT $?"; grep -E "BUILD SUCCEEDED|BUILD FAILED" /tmp/t1.log | tail -1
```
Expected: `BUILD SUCCEEDED`. If it fails with "missing argument for parameter 'lengthMeters'", a creature literal was missed — the error names the line.

- [ ] **Step 6: Commit**

```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
git add "Deep Blue/OceanModels.swift" "Deep Blue/VesselData.swift" "Deep Blue/OceanData.swift" "Deep Blue.xcodeproj/project.pbxproj"
git commit -m "Add creature length, Biome model and the five vessels"
```

---

## Task 2: Rank system + expedition engine + body scale (pure logic)

**Files:**
- Create: `Deep Blue/RankSystem.swift`, `Deep Blue/ExpeditionEngine.swift`, `Deep Blue/BodyScale.swift`
- Modify: `Deep Blue.xcodeproj/project.pbxproj`

**Interfaces:**
- Consumes: `Vessels` (Task 1), `Ocean.zones`, `Ocean.creatures`.
- Produces:
  - `struct DiverRank { let number: Int; let name: String; let xp: Int }`
  - `enum Ranks { static let all: [DiverRank]; static func rank(forXP: Int) -> DiverRank; static func next(afterXP: Int) -> DiverRank?; static func progress(xp: Int) -> Double; static func unlockedVessels(xp: Int) -> [Vessel] }`
  - `enum XPAward { static let creature = 5; static let depthStep = 3; static let expedition = 15; static let quiz = 2; static let zone = 10 }`
  - `enum ExpeditionKind { case reachDepth(Int), findInZone(Int, Int), findBioluminescent(Int), findAny(Int), visitZone(Int) }`
  - `struct Expedition { let id: String; let kind: ExpeditionKind; let title: String; let detail: String; let target: Int }`
  - `enum ExpeditionEngine { static func today(dayIndex: Int) -> Expedition }`
  - `enum BodyScale { static func surfaceArea(heightCM: Double, weightKG: Double) -> Double; static func tonnesOnBody(depth: Int, heightCM: Double, weightKG: Double) -> Double; static func pressurePhrase(depth: Int, heightCM: Double, weightKG: Double) -> String; static func sizeRatio(creature: Creature, heightCM: Double) -> Double; static func sizePhrase(creature: Creature, heightCM: Double) -> String }`

- [ ] **Step 1: Create `Deep Blue/RankSystem.swift`**

```swift
import SwiftUI

// Diver ranks. XP is earned by exploring; each rank is a title, and some
// ranks hand over the keys to a deeper-diving craft.

struct DiverRank: Identifiable {
    var id: Int { number }
    let number: Int
    let name: String
    let xp: Int          // XP required to hold this rank
}

enum XPAward {
    static let creature = 5      // discovering a creature
    static let depthStep = 3     // per 500 m of new personal-best depth
    static let expedition = 15   // completing the daily expedition
    static let quiz = 2          // a correct quiz answer
    static let zone = 10         // entering a zone for the first time
}

enum Ranks {
    static let all: [DiverRank] = [
        DiverRank(number: 1, name: "Snorkeler",          xp: 0),
        DiverRank(number: 2, name: "Free Diver",         xp: 40),
        DiverRank(number: 3, name: "Scuba Diver",        xp: 100),
        DiverRank(number: 4, name: "Technical Diver",    xp: 180),
        DiverRank(number: 5, name: "Submersible Pilot",  xp: 300),
        DiverRank(number: 6, name: "Deep Explorer",      xp: 450),
        DiverRank(number: 7, name: "Trench Pioneer",     xp: 650),
        DiverRank(number: 8, name: "Master of the Abyss", xp: 900),
    ]

    static func rank(forXP xp: Int) -> DiverRank {
        var current = all[0]
        for r in all where xp >= r.xp { current = r }
        return current
    }

    static func next(afterXP xp: Int) -> DiverRank? {
        all.first { $0.xp > xp }
    }

    /// 0...1 progress from the current rank's threshold to the next one.
    static func progress(xp: Int) -> Double {
        let cur = rank(forXP: xp)
        guard let nxt = next(afterXP: xp) else { return 1 }
        let span = Double(nxt.xp - cur.xp)
        guard span > 0 else { return 1 }
        return min(1, max(0, Double(xp - cur.xp) / span))
    }

    static func unlockedVessels(xp: Int) -> [Vessel] {
        let n = rank(forXP: xp).number
        return Vessels.all.filter { $0.unlockRank <= n }
    }

    static func isVesselUnlocked(_ v: Vessel, xp: Int) -> Bool {
        rank(forXP: xp).number >= v.unlockRank
    }
}
```

- [ ] **Step 2: Create `Deep Blue/ExpeditionEngine.swift`**

```swift
import Foundation

// One objective per calendar day, generated deterministically from the day
// number so it is stable all day and needs no network and no notifications.

enum ExpeditionKind {
    case reachDepth(Int)          // metres
    case findInZone(Int, Int)     // zone index, count
    case findBioluminescent(Int)  // count
    case findAny(Int)             // count
    case visitZone(Int)           // zone index
}

struct Expedition {
    let id: String
    let kind: ExpeditionKind
    let title: String
    let detail: String
    let target: Int
}

enum ExpeditionEngine {

    /// Deterministic objective for a given day index (days since era).
    static func today(dayIndex: Int) -> Expedition {
        let slot = ((dayIndex % 8) + 8) % 8
        switch slot {
        case 0:
            return Expedition(id: "d\(dayIndex)", kind: .findAny(3),
                              title: "Identify Three",
                              detail: "Identify 3 creatures you have never seen before.", target: 3)
        case 1:
            return Expedition(id: "d\(dayIndex)", kind: .reachDepth(1_000),
                              title: "Into the Dark",
                              detail: "Descend to 1,000 m, where sunlight ends.", target: 1_000)
        case 2:
            return Expedition(id: "d\(dayIndex)", kind: .findBioluminescent(2),
                              title: "Living Light",
                              detail: "Meet 2 creatures that make their own light.", target: 2)
        case 3:
            return Expedition(id: "d\(dayIndex)", kind: .findInZone(2, 2),
                              title: "Midnight Watch",
                              detail: "Find 2 creatures of the Midnight Zone.", target: 2)
        case 4:
            return Expedition(id: "d\(dayIndex)", kind: .reachDepth(4_000),
                              title: "Abyss Bound",
                              detail: "Descend to 4,000 m, onto the abyssal plains.", target: 4_000)
        case 5:
            return Expedition(id: "d\(dayIndex)", kind: .findInZone(1, 2),
                              title: "Twilight Survey",
                              detail: "Find 2 creatures of the Twilight Zone.", target: 2)
        case 6:
            return Expedition(id: "d\(dayIndex)", kind: .visitZone(4),
                              title: "Trench Run",
                              detail: "Reach the Hadal Zone, past 6,000 m.", target: 6_000)
        default:
            return Expedition(id: "d\(dayIndex)", kind: .findAny(5),
                              title: "Field Day",
                              detail: "Identify 5 creatures you have never seen before.", target: 5)
        }
    }

    /// Current day index used for seeding.
    static func dayIndex(for date: Date = Date()) -> Int {
        Calendar.current.ordinality(of: .day, in: .era, for: date) ?? 0
    }
}
```

- [ ] **Step 3: Create `Deep Blue/BodyScale.swift`**

```swift
import Foundation

// Turns abstract pressure into something bodily, and creature sizes into
// something you can picture next to yourself.

enum BodyScale {

    /// Du Bois body-surface-area estimate, in square metres.
    static func surfaceArea(heightCM: Double, weightKG: Double) -> Double {
        guard heightCM > 0, weightKG > 0 else { return 1.8 }
        return 0.007184 * pow(weightKG, 0.425) * pow(heightCM, 0.725)
    }

    /// Force of the water on a whole body at depth, in tonnes.
    /// Gauge pressure ≈ 1 atm per 10 m; 1 atm ≈ 10.1325 tonnes per square metre.
    static func tonnesOnBody(depth: Int, heightCM: Double, weightKG: Double) -> Double {
        let gaugeAtm = Double(max(0, depth)) / 10.0
        return gaugeAtm * 10.1325 * surfaceArea(heightCM: heightCM, weightKG: weightKG)
    }

    /// A friendly, bodily sentence about the pressure at this depth.
    static func pressurePhrase(depth: Int, heightCM: Double, weightKG: Double) -> String {
        let t = tonnesOnBody(depth: depth, heightCM: heightCM, weightKG: weightKG)
        if t < 1 { return "Barely more than a hug of water." }
        if t < 60 {
            return String(format: "About %.1f tonnes press on you here.", t)
        }
        let elephants = t / 6.0     // ~6 t for an African elephant
        if elephants < 1000 {
            return String(format: "Like %.0f elephants resting on you.", max(1, elephants.rounded()))
        }
        return String(format: "Around %.0f tonnes — far past what a body could take.", t)
    }

    /// How many times longer the creature is than the person.
    static func sizeRatio(creature: Creature, heightCM: Double) -> Double {
        let h = heightCM > 0 ? heightCM / 100.0 : 1.7
        guard h > 0 else { return 1 }
        return creature.lengthMeters / h
    }

    static func sizePhrase(creature: Creature, heightCM: Double) -> String {
        let r = sizeRatio(creature: creature, heightCM: heightCM)
        if r >= 1.15 {
            return String(format: "About %.1f× longer than you.", r)
        }
        if r <= 0.85 {
            let inv = r > 0 ? 1.0 / r : 1
            return String(format: "You are about %.1f× longer than it.", inv)
        }
        return "Just about your own length."
    }
}
```

- [ ] **Step 4: Register the three files in `project.pbxproj`**

Add to all four sections exactly as in Task 1 Step 4, using ids `…0031`/`…0131` (RankSystem), `…0032`/`…0132` (ExpeditionEngine), `…0033`/`…0133` (BodyScale). Example for one:

```
		DB0000000000000000000131 /* RankSystem.swift in Sources */ = {isa = PBXBuildFile; fileRef = DB0000000000000000000031 /* RankSystem.swift */; };
		DB0000000000000000000031 /* RankSystem.swift */ = {isa = PBXFileReference; lastKnownFileType = sourcecode.swift; path = RankSystem.swift; sourceTree = "<group>"; };
```

- [ ] **Step 5: Verify the logic before wiring any UI**

Run this standalone check (it re-declares the thresholds so it needs no app target):

```bash
cat > /tmp/rankcheck.swift <<'EOF'
let thresholds = [0, 40, 100, 180, 300, 450, 650, 900]
func rankNumber(_ xp: Int) -> Int {
    var n = 1
    for (i, t) in thresholds.enumerated() where xp >= t { n = i + 1 }
    return n
}
assert(rankNumber(0) == 1)
assert(rankNumber(39) == 1)
assert(rankNumber(40) == 2)
assert(rankNumber(299) == 4)
assert(rankNumber(300) == 5)
assert(rankNumber(10_000) == 8)
// Du Bois BSA for 175 cm / 75 kg ≈ 1.90 m²
let bsa = 0.007184 * pow(75.0, 0.425) * pow(175.0, 0.725)
assert(bsa > 1.85 && bsa < 1.95, "BSA out of range: \(bsa)")
// 300 m on that body ≈ 577 t
let t = (300.0/10.0) * 10.1325 * bsa
assert(t > 550 && t < 600, "tonnes out of range: \(t)")
print("rank + body-scale checks passed")
EOF
swift /tmp/rankcheck.swift
```
Expected output: `rank + body-scale checks passed`

- [ ] **Step 6: Compile**

Run:
```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue" && xcodebuild -project "Deep Blue.xcodeproj" -scheme "Deep Blue" -destination 'platform=iOS Simulator,name=iPhone 17' -derivedDataPath build/ build > /tmp/t2.log 2>&1; echo "EXIT $?"; grep -E "BUILD SUCCEEDED|BUILD FAILED" /tmp/t2.log | tail -1
```
Expected: `BUILD SUCCEEDED`

- [ ] **Step 7: Commit**

```bash
git add "Deep Blue/RankSystem.swift" "Deep Blue/ExpeditionEngine.swift" "Deep Blue/BodyScale.swift" "Deep Blue.xcodeproj/project.pbxproj"
git commit -m "Add rank system, daily expedition engine and body-scale math"
```

---

## Task 3: Store — XP, vessel, profile, expedition state (v1 saves must survive)

**Files:**
- Modify: `Deep Blue/DeepStore.swift`

**Interfaces:**
- Consumes: `Ranks`, `XPAward`, `Vessels`, `ExpeditionEngine`, `Expedition`, `ExpeditionKind` (Tasks 1–2).
- Produces on `DeepStore`:
  - `@Published private(set) var xp: Int`, `currentVesselID: String`, `zonesVisited: Set<Int>`, `expeditionDay: Int`, `expeditionDoneDay: Int`, `expeditionHistory: [Int]`
  - `@Published var profileHeightCM: Double`, `profileWeightKG: Double`, `metric: Bool`
  - `var rank: DiverRank`, `var nextRank: DiverRank?`, `var rankProgress: Double`, `var vessel: Vessel`, `var hasProfile: Bool`
  - `func addXP(_ n: Int)`, `func selectVessel(_ id: String)`, `func markZoneVisited(_ z: Int)`, `func setProfile(heightCM: Double, weightKG: Double)`, `func clearProfile()`
  - `var todayExpedition: Expedition`, `var expeditionProgressValue: Int`, `var expeditionComplete: Bool`, `func refreshExpedition()`

- [ ] **Step 1: Add the new stored state**

In `Deep Blue/DeepStore.swift`, after the existing `@Published var showLatin` line, insert:

```swift
    // v2 — expedition state
    @Published private(set) var xp: Int = 0
    @Published private(set) var currentVesselID: String = Vessels.defaultID
    @Published private(set) var zonesVisited: Set<Int> = []
    @Published private(set) var expeditionDoneDay: Int = -1
    @Published private(set) var expeditionHistory: [Int] = []

    @Published var profileHeightCM: Double = 0 { didSet { save() } }
    @Published var profileWeightKG: Double = 0 { didSet { save() } }
    @Published var metric: Bool = true { didSet { save() } }
```

- [ ] **Step 2: Add the derived accessors and mutators**

Insert this block just before the `// MARK: - Persistence` comment:

```swift
    // MARK: - Rank & XP

    var rank: DiverRank { Ranks.rank(forXP: xp) }
    var nextRank: DiverRank? { Ranks.next(afterXP: xp) }
    var rankProgress: Double { Ranks.progress(xp: xp) }

    func addXP(_ n: Int) {
        guard n > 0 else { return }
        xp += n
        save()
    }

    // MARK: - Vessel

    var vessel: Vessel { Vessels.byID(currentVesselID) }

    func isVesselUnlocked(_ v: Vessel) -> Bool { Ranks.isVesselUnlocked(v, xp: xp) }

    func selectVessel(_ id: String) {
        let v = Vessels.byID(id)
        guard isVesselUnlocked(v) else { return }
        currentVesselID = v.id
        save()
    }

    /// The deepest-reaching craft the diver has earned.
    var bestVessel: Vessel {
        Ranks.unlockedVessels(xp: xp).max(by: { $0.limit < $1.limit }) ?? Vessels.all[0]
    }

    // MARK: - Zones

    func markZoneVisited(_ z: Int) {
        guard !zonesVisited.contains(z) else { return }
        zonesVisited.insert(z)
        xp += XPAward.zone
        save()
    }

    // MARK: - Profile

    var hasProfile: Bool { profileHeightCM > 0 && profileWeightKG > 0 }

    func setProfile(heightCM: Double, weightKG: Double) {
        profileHeightCM = heightCM
        profileWeightKG = weightKG
    }

    func clearProfile() {
        profileHeightCM = 0
        profileWeightKG = 0
    }

    // MARK: - Daily expedition

    var todayExpedition: Expedition {
        ExpeditionEngine.today(dayIndex: ExpeditionEngine.dayIndex())
    }

    var expeditionComplete: Bool {
        expeditionDoneDay == ExpeditionEngine.dayIndex()
    }

    /// How far along today's objective the diver is, in the objective's own units.
    var expeditionProgressValue: Int {
        switch todayExpedition.kind {
        case .reachDepth:
            return deepestReached
        case .visitZone(let z):
            return zonesVisited.contains(z) ? Ocean.zone(z).top : deepestReached
        case .findAny(let n):
            return min(discoveredCount, n)
        case .findBioluminescent(let n):
            return min(bioDiscoveredCount, n)
        case .findInZone(let z, let n):
            return min(discoveredIn(z), n)
        }
    }

    /// 0...1 fraction of today's objective.
    var expeditionFraction: Double {
        let t = Double(todayExpedition.target)
        guard t > 0 else { return 0 }
        return min(1, Double(expeditionProgressValue) / t)
    }

    /// Checks today's objective and banks the reward the first time it is met.
    func refreshExpedition() {
        guard !expeditionComplete else { return }
        let day = ExpeditionEngine.dayIndex()
        let met: Bool
        switch todayExpedition.kind {
        case .reachDepth(let d):            met = deepestReached >= d
        case .visitZone(let z):             met = zonesVisited.contains(z)
        case .findAny(let n):               met = discoveredCount >= n
        case .findBioluminescent(let n):    met = bioDiscoveredCount >= n
        case .findInZone(let z, let n):     met = discoveredIn(z) >= n
        }
        guard met else { return }
        expeditionDoneDay = day
        if !expeditionHistory.contains(day) { expeditionHistory.append(day) }
        if expeditionHistory.count > 60 { expeditionHistory.removeFirst(expeditionHistory.count - 60) }
        xp += XPAward.expedition
        if hapticsOn { Haptics.success() }
        save()
    }
```

- [ ] **Step 3: Award XP on discovery and on new depth**

Replace the existing `discover(_:)` body's `discovered.insert(id)` line region so the method reads:

```swift
    @discardableResult
    func discover(_ id: String) -> Bool {
        guard !discovered.contains(id) else { return false }
        discovered.insert(id)
        xp += XPAward.creature
        if hapticsOn { Haptics.success() }
        save()
        refreshExpedition()
        return true
    }
```

Replace `recordDepth(_:)` entirely with:

```swift
    func recordDepth(_ depth: Int) {
        let clamped = max(0, min(Ocean.maxDepth, depth))
        if clamped > deepestReached {
            let steps = (clamped / 500) - (deepestReached / 500)
            deepestReached = clamped
            if steps > 0 { xp += steps * XPAward.depthStep }
            markZoneVisited(Ocean.zoneIndex(forDepth: clamped))
            save()
            refreshExpedition()
        }
    }
```

Note: `markZoneVisited` and `refreshExpedition` call `save()` themselves; the extra call is harmless and keeps each mutation self-contained.

- [ ] **Step 4: Extend persistence with tolerant decode**

Replace the `Persist` struct and the `load()` method with:

```swift
    private struct Persist: Codable {
        var discovered: [String]
        var deepest: Int
        var onboarding: Bool
        var bubbles: Bool
        var haptics: Bool
        var latin: Bool
        // v2 — all optional so v1 saves still decode
        var xp: Int?
        var vessel: String?
        var zones: [Int]?
        var expDoneDay: Int?
        var expHistory: [Int]?
        var heightCM: Double?
        var weightKG: Double?
        var metric: Bool?
    }

    private func save() {
        let p = Persist(discovered: Array(discovered), deepest: deepestReached,
                        onboarding: onboardingSeen, bubbles: bubblesOn,
                        haptics: hapticsOn, latin: showLatin,
                        xp: xp, vessel: currentVesselID, zones: Array(zonesVisited),
                        expDoneDay: expeditionDoneDay, expHistory: expeditionHistory,
                        heightCM: profileHeightCM, weightKG: profileWeightKG,
                        metric: metric)
        if let data = try? JSONEncoder().encode(p) {
            UserDefaults.standard.set(data, forKey: key)
        }
    }

    private func load() {
        guard let data = UserDefaults.standard.data(forKey: key),
              let p = try? JSONDecoder().decode(Persist.self, from: data) else { return }
        discovered = Set(p.discovered)
        deepestReached = p.deepest
        onboardingSeen = p.onboarding
        bubblesOn = p.bubbles
        hapticsOn = p.haptics
        showLatin = p.latin
        xp = p.xp ?? 0
        currentVesselID = p.vessel ?? Vessels.defaultID
        zonesVisited = Set(p.zones ?? [])
        expeditionDoneDay = p.expDoneDay ?? -1
        expeditionHistory = p.expHistory ?? []
        profileHeightCM = p.heightCM ?? 0
        profileWeightKG = p.weightKG ?? 0
        metric = p.metric ?? true
    }
```

Also extend `resetProgress()` so the new fields clear too — replace it with:

```swift
    func resetProgress() {
        discovered.removeAll()
        deepestReached = 0
        xp = 0
        currentVesselID = Vessels.defaultID
        zonesVisited.removeAll()
        expeditionDoneDay = -1
        expeditionHistory.removeAll()
        save()
    }
```

**Important:** the `didSet { save() }` observers on `profileHeightCM`/`profileWeightKG`/`metric` fire during `load()`. That is safe (it just re-writes the same values) but it must not run before `discovered` is populated — since `load()` assigns them last, this ordering is already correct. Do not reorder those assignments.

- [ ] **Step 5: Compile**

Run:
```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue" && xcodebuild -project "Deep Blue.xcodeproj" -scheme "Deep Blue" -destination 'platform=iOS Simulator,name=iPhone 17' -derivedDataPath build/ build > /tmp/t3.log 2>&1; echo "EXIT $?"; grep -E "BUILD SUCCEEDED|BUILD FAILED" /tmp/t3.log | tail -1
```
Expected: `BUILD SUCCEEDED`

- [ ] **Step 6: Verify a v1 save still loads (this is the migration risk)**

```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
xcrun simctl install booted "build/Build/Products/Debug-iphonesimulator/Deep Blue.app"
CONT=$(xcrun simctl get_app_container booted com.deepbluedive.app data)
python3 - "$CONT" <<'PY'
import sys, os, json, plistlib
prefs = os.path.join(sys.argv[1], "Library", "Preferences")
os.makedirs(prefs, exist_ok=True)
path = os.path.join(prefs, "com.deepbluedive.app.plist")
# a v1-shaped blob: no v2 keys at all
v1 = {"discovered": ["moon_jelly", "great_white"], "deepest": 1200,
      "onboarding": True, "bubbles": True, "haptics": True, "latin": True}
plistlib.dump({"deepblue.state.v1": json.dumps(v1).encode()}, open(path, "wb"))
print("seeded v1-shaped save")
PY
xcrun simctl launch booted com.deepbluedive.app
```
Then screenshot after ~8 s and open it:
```bash
xcrun simctl io booted screenshot /tmp/v1compat.png
```
Expected: the app opens to the Dive hub (not onboarding), the HUD shows `2/50 found`, and the Journal shows rank **Snorkeler** with 0 XP. A crash or a reset-to-onboarding means the tolerant decode is wrong.

- [ ] **Step 7: Commit**

```bash
git add "Deep Blue/DeepStore.swift"
git commit -m "Add XP, ranks, vessel, profile and daily expedition state to the store"
```

---

## Task 4: Art — 14 new creatures, 6 biome scenes, 5 vessel silhouettes

**Files:**
- Modify: `art_src/generate_art.py`

**Interfaces:**
- Produces PNGs in `Deep Blue/Art/`: `creature_<slug>.png` and `card_<slug>.png` for 14 new slugs; `scene_<id>.png` ×6; `vessel_<id>.png` ×5.

New creature slugs (must match Task 5's data records exactly):
`whale_shark, orca, hammerhead, nautilus, coelacanth, oarfish, frilled_shark, goblin_shark, spider_crab, siphonophore, comb_jelly, glass_squid, sea_angel, giant_clam`

- [ ] **Step 1: Add the 14 creatures to the generator's roster**

In `art_src/generate_art.py`, add these entries inside the `ROSTER` dictionary, reusing the existing archetype drawers:

```python
 "whale_shark":   lambda: draw_shark(dict(L=680,H=210,top=(90,110,135),bot=(35,48,68),glow=(90,130,180))),
 "orca":          lambda: draw_cetacean(dict(L=660,H=200,top=(235,240,245),bot=(20,24,34),glow=(150,180,210),dorsal=True)),
 "hammerhead":    lambda: draw_shark(dict(L=600,H=150,top=(130,140,155),bot=(50,62,80),glow=(110,145,185))),
 "nautilus":      lambda: draw_jelly(dict(R=150,top=(245,225,200),bot=(180,120,90),glow=(230,190,150),tentacles=22)),
 "coelacanth":    lambda: draw_fish(dict(L=560,H=190,top=(70,95,120),bot=(25,40,60),glow=(80,130,170),tail="round",dorsal="low",eyeR=15)),
 "oarfish":       lambda: draw_eel(dict(L=800,H=44,top=(200,205,220),bot=(90,105,130),glow=(150,180,220))),
 "frilled_shark": lambda: draw_eel(dict(L=760,H=52,top=(110,95,90),bot=(45,38,36),glow=(120,100,95),head_big=True)),
 "goblin_shark":  lambda: draw_shark(dict(L=600,H=150,top=(225,180,185),bot=(120,80,90),glow=(215,160,170))),
 "spider_crab":   lambda: draw_seaspider(dict()),
 "siphonophore":  lambda: draw_jelly(dict(R=120,top=(200,235,255),bot=(110,170,220),glow=(150,215,255),tentacles=30,longtent=True)),
 "comb_jelly":    lambda: draw_jelly(dict(R=140,top=(210,240,255),bot=(140,180,235),glow=(170,220,255),tentacles=12,crown=True)),
 "glass_squid":   lambda: draw_squid(dict(L=300,Wd=120,top=(200,230,245),bot=(120,160,200),glow=(170,215,245),arms=8,tentacles=True,eyeR=24,eyeglow=(150,225,255))),
 "sea_angel":     lambda: draw_jelly(dict(R=110,top=(255,200,215),bot=(200,110,140),glow=(250,170,200),tentacles=8,oral=True)),
 "giant_clam":    lambda: draw_xeno(dict()),
```

- [ ] **Step 2: Add the 14 slugs to the portrait-card zone map**

In the same file, add these keys to the `SLUG_ZONE` dictionary so the cards get the right background tint:

```python
 "whale_shark":0,"orca":0,"hammerhead":0,"giant_clam":0,"sea_angel":0,
 "nautilus":1,"coelacanth":1,"oarfish":1,"siphonophore":1,"comb_jelly":1,"glass_squid":1,
 "frilled_shark":2,"goblin_shark":2,
 "spider_crab":3,
```

- [ ] **Step 3: Add the biome-scene and vessel generators**

Insert this block immediately before the `# ============================================================ APP ICON` comment:

```python
# ============================================================ BIOME SCENES

BIOMES = [
    ("scene_reef",   (30,140,175), (8,60,105),  (255,150,120)),
    ("scene_kelp",   (25,110,120), (6,45,60),   (120,200,120)),
    ("scene_whalefall", (10,45,90), (3,16,40),  (200,205,215)),
    ("scene_vent",   (14,30,60),  (4,10,24),    (255,120,90)),
    ("scene_brine",  (8,26,58),   (2,9,22),     (150,220,230)),
    ("scene_trench", (5,16,40),   (1,5,14),     (120,160,220)),
]

def gen_scenes():
    W, H = 1400, 900
    for name, top, bot, accent in BIOMES:
        if not want(name):
            continue
        img = vgrad(W, H, top, bot).convert("RGBA")
        d = ImageDraw.Draw(img)
        import random as _r
        _r.seed(hash(name) % 9999)

        if name == "scene_reef":
            for i in range(26):                       # coral fans
                x = _r.randint(0, W); y = H - _r.randint(0, 260)
                h = _r.randint(90, 240)
                col = (_r.randint(200,255), _r.randint(90,160), _r.randint(90,170))
                for b in range(5):
                    d.line([(x, y), (x + _r.randint(-60, 60), y - h)], fill=col + (190,), width=_r.randint(5, 12))
        elif name == "scene_kelp":
            for i in range(22):                       # kelp stipes
                x = _r.randint(0, W)
                pts = [(x + 34 * math.sin(k / 3.0 + i), H - k * 26) for k in range(34)]
                d.line(pts, fill=(90, 170, 110, 200), width=_r.randint(7, 14))
        elif name == "scene_whalefall":
            cx, cy = W * 0.5, H * 0.66                # ribcage on the seafloor
            d.ellipse([cx-330, cy-40, cx+330, cy+90], fill=(220, 222, 228, 70))
            for i in range(11):
                t = i / 10.0
                x = cx - 300 + 600 * t
                d.arc([x-56, cy-190, x+56, cy+60], 200, 340, fill=(232, 234, 240, 225), width=11)
            d.line([(cx-330, cy+20), (cx+330, cy+20)], fill=(240, 242, 246, 220), width=9)
        elif name == "scene_vent":
            for (x, w, h) in [(W*0.28, 90, 380), (W*0.52, 130, 470), (W*0.75, 80, 320)]:
                d.polygon([(x-w, H), (x-w*0.4, H-h), (x+w*0.4, H-h), (x+w, H)], fill=(28, 24, 30, 255))
                plume = soft_glow((W, H), lambda dd, x=x, h=h: dd.ellipse(
                    [x-70, H-h-260, x+70, H-h+40], fill=255), (60, 70, 90), blur=60, alpha=150)
                img.alpha_composite(plume)
                g = soft_glow((W, H), lambda dd, x=x, h=h: dd.ellipse(
                    [x-34, H-h-30, x+34, H-h+30], fill=255), accent, blur=26, alpha=220)
                img.alpha_composite(g)
        elif name == "scene_brine":
            d.ellipse([W*0.16, H*0.60, W*0.84, H*0.96], fill=(20, 60, 80, 235))
            d.ellipse([W*0.19, H*0.62, W*0.81, H*0.94], outline=(170, 230, 240, 200), width=7)
            for i in range(9):                        # shimmering shore
                yy = H*0.60 + i*7
                d.arc([W*0.16, yy, W*0.84, yy + H*0.34], 0, 180, fill=(150, 220, 230, 60), width=3)
        else:  # scene_trench
            d.polygon([(0, H), (W*0.34, H*0.30), (W*0.42, H)], fill=(6, 12, 26, 255))
            d.polygon([(W, H), (W*0.68, H*0.24), (W*0.58, H)], fill=(5, 10, 22, 255))
            d.line([(W*0.42, H), (W*0.5, H*0.55), (W*0.58, H)], fill=(30, 50, 90, 150), width=5)

        for _ in range(220):                          # marine snow over everything
            x = _r.randint(0, W); y = _r.randint(0, H); r = _r.randint(1, 3)
            d.ellipse([x-r, y-r, x+r, y+r], fill=(225, 238, 248, _r.randint(15, 60)))
        img.convert("RGB").save(os.path.join(OUT, name + ".png"))
        print("  ", name)


# ============================================================ VESSELS

def gen_vessels():
    W, H = 900, 560
    specs = {
        "vessel_mask":        ((150, 210, 240), (60, 120, 170)),
        "vessel_scuba":       ((120, 200, 205), (40, 105, 120)),
        "vessel_bathysphere": ((225, 195, 130), (120, 90, 45)),
        "vessel_submersible": ((175, 165, 235), (80, 70, 140)),
        "vessel_fullocean":   ((240, 150, 130), (135, 60, 55)),
    }
    for name, (top, bot) in specs.items():
        if not want(name):
            continue
        c = Image.new("RGBA", (W, H), (0, 0, 0, 0))
        cx, cy = W * 0.5, H * 0.5
        d = ImageDraw.Draw(c)

        if name == "vessel_mask":
            m = poly_mask_at(c, [(cx-150, cy-90), (cx+150, cy-90), (cx+130, cy+70), (cx-130, cy+70)], r=34)
            c.alpha_composite(clip_grad_size(m, top, bot, c.size))
            d.rounded_rectangle([cx-120, cy-64, cx+120, cy+34], radius=26, fill=(210, 240, 255, 130))
            d.line([(cx-150, cy-60), (cx-250, cy-40)], fill=rgb(bot) + (255,), width=12)
            d.line([(cx+150, cy-60), (cx+250, cy-40)], fill=rgb(bot) + (255,), width=12)
        elif name == "vessel_scuba":
            for dx in (-46, 46):                      # twin tanks
                d.rounded_rectangle([cx+dx-40, cy-150, cx+dx+40, cy+150], radius=40,
                                    fill=rgb(top) + (255,))
                d.rounded_rectangle([cx+dx-40, cy+10, cx+dx+40, cy+150], radius=40,
                                    fill=rgb(bot) + (255,))
            d.rounded_rectangle([cx-110, cy-176, cx+110, cy-140], radius=16, fill=rgb(bot) + (255,))
            d.arc([cx-150, cy-200, cx+150, cy-80], 200, 340, fill=rgb(bot) + (255,), width=12)
        elif name == "vessel_bathysphere":
            mask = ellipse_mask_at(c, cx, cy, 165, 165)
            c.alpha_composite(clip_grad_size(mask, top, bot, c.size))
            for i in range(3):                        # portholes
                d.ellipse([cx-108+i*88, cy-34, cx-40+i*88, cy+34], fill=(20, 40, 60, 255))
                d.ellipse([cx-100+i*88, cy-26, cx-48+i*88, cy+26], fill=(150, 220, 245, 220))
            d.line([(cx, cy-165), (cx, cy-260)], fill=(200, 205, 215, 255), width=10)
        elif name == "vessel_submersible":
            mask = ellipse_mask_at(c, cx, cy, 230, 110)
            c.alpha_composite(clip_grad_size(mask, top, bot, c.size))
            d.ellipse([cx+90, cy-70, cx+215, cy+55], fill=(150, 225, 250, 210))   # bubble cockpit
            d.rounded_rectangle([cx-235, cy-40, cx-150, cy+40], radius=18, fill=rgb(bot) + (255,))
            for s in (-1, 1):                          # manipulator arms
                d.line([(cx+120, cy+70), (cx+210, cy+s*30+110)], fill=rgb(bot) + (255,), width=13)
            d.ellipse([cx+150, cy+92, cx+196, cy+138], fill=(255, 245, 200, 255))  # lamp
        else:  # vessel_fullocean
            mask = ellipse_mask_at(c, cx, cy, 210, 130)
            c.alpha_composite(clip_grad_size(mask, top, bot, c.size))
            d.ellipse([cx-70, cy-72, cx+70, cy+68], fill=(30, 45, 65, 255))
            d.ellipse([cx-56, cy-58, cx+56, cy+54], fill=(160, 230, 250, 225))
            d.rounded_rectangle([cx-215, cy+60, cx+215, cy+112], radius=24, fill=rgb(bot) + (255,))
            for dx in (-130, 0, 130):
                g = soft_glow(c.size, lambda dd, dx=dx: dd.ellipse(
                    [cx+dx-26, cy+118, cx+dx+26, cy+170], fill=255), (255, 240, 190), blur=20, alpha=230)
                c.alpha_composite(g)
        c.resize((int(W * 0.8), int(H * 0.8)), Image.LANCZOS).save(os.path.join(OUT, name + ".png"))
        print("  ", name)


def poly_mask_at(canvas, pts, r=0):
    m = Image.new("L", canvas.size, 0)
    ImageDraw.Draw(m).polygon(pts, fill=255)
    return m


def ellipse_mask_at(canvas, cx, cy, rx, ry):
    m = Image.new("L", canvas.size, 0)
    ImageDraw.Draw(m).ellipse([cx-rx, cy-ry, cx+rx, cy+ry], fill=255)
    return m


def clip_grad_size(mask, top, bot, size):
    g = vgrad(size[0], size[1], top, bot).convert("RGBA")
    g.putalpha(mask)
    return g
```

- [ ] **Step 4: Call the new generators from `__main__`**

Replace the `__main__` block's body with:

```python
if __name__ == "__main__":
    print("Deep Blue art:")
    gen_backdrops()
    gen_banners()
    gen_scenes()
    gen_vessels()
    gen_creatures()
    gen_portraits()
    gen_icon()
    print("done.")
```

- [ ] **Step 5: Run the generator (slow — run in the background)**

```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue/art_src" && python3 generate_art.py > /tmp/art.log 2>&1; echo "ART_EXIT $?"
```
Expected in `/tmp/art.log`: the six `scene_*`, five `vessel_*`, all 50 `creature_*`, all 50 `card_*`, then `done.`

- [ ] **Step 6: Verify the output**

```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue/Deep Blue"
echo "creatures: $(ls Art/creature_*.png | wc -l)   (expect 50)"
echo "cards:     $(ls Art/card_*.png | wc -l)       (expect 50)"
echo "scenes:    $(ls Art/scene_*.png | wc -l)      (expect 6)"
echo "vessels:   $(ls Art/vessel_*.png | wc -l)     (expect 5)"
du -sh Art
sips -g hasAlpha Assets.xcassets/AppIcon.appiconset/AppIcon-1024.png | tail -1
```
Expected: the four counts above, `Art` around 33–40 MB, and `hasAlpha: no`.

- [ ] **Step 7: Eyeball the new art**

Build a contact sheet and look at it before trusting it:
```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue/art_src" && python3 - <<'PY'
from PIL import Image
import glob, os
OUT = "../Deep Blue/Art"
names = ["creature_whale_shark","creature_orca","creature_nautilus","creature_oarfish",
         "creature_goblin_shark","creature_comb_jelly","vessel_bathysphere","vessel_submersible",
         "vessel_fullocean","scene_vent","scene_whalefall","scene_trench"]
sheet = Image.new("RGB", (4*340, 3*250), (6,18,42))
for i, n in enumerate(names):
    p = os.path.join(OUT, n + ".png")
    if not os.path.exists(p): continue
    im = Image.open(p).convert("RGBA"); im.thumbnail((330, 240))
    sheet.paste(im, ((i % 4)*340 + (330-im.width)//2, (i//4)*250), im)
sheet.save("/tmp/newart.png"); print("ok")
PY
```
Open `/tmp/newart.png` and confirm every tile is a recognisable subject, not an empty or garbled shape. Fix any drawer that produced nothing before moving on.

- [ ] **Step 8: Commit**

```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
git add art_src/generate_art.py "Deep Blue/Art"
git commit -m "Generate 14 new creatures, 6 biome scenes and 5 vessel silhouettes"
```

---

## Task 5: Content — 14 creature records, 6 awards, 6 biomes

**Files:**
- Modify: `Deep Blue/OceanData.swift`

**Interfaces:**
- Consumes: `Creature`, `Biome`, `Award` (Tasks 1–2), `DeepStore` accessors (Task 3).
- Produces: `Ocean.creatures` at 50 entries, `Ocean.awards` at 18, `Ocean.biomes` (6 entries), `Ocean.biome(_:)`.

- [ ] **Step 1: Append the 14 creature records**

Add these inside the `Ocean.creatures` array, each in its zone's section (the array is grouped by zone with `// ----` comments):

```swift
        // → Sunlight section
        Creature(id: "giant_clam", name: "Giant Clam", sci: "Tridacna gigas",
                 zone: 0, appear: 18, rangeTop: 0, rangeBottom: 20,
                 size: "Up to 1.2 m", lengthMeters: 1.2, diet: "Algae it farms in its own flesh",
                 facts: [
                    "It can live more than a century anchored to one spot on the reef.",
                    "Colourful algae live inside its mantle and feed it with sunlight.",
                    "The largest shells weigh as much as two adult people."
                 ], glows: false, accent: Sea.teal),
        Creature(id: "sea_angel", name: "Sea Angel", sci: "Clione limacina",
                 zone: 0, appear: 45, rangeTop: 0, rangeBottom: 500,
                 size: "Up to 5 cm", lengthMeters: 0.05, diet: "Sea butterflies (small snails)",
                 facts: [
                    "A snail that gave up its shell and swims with two tiny wings.",
                    "It looks delicate but hunts its cousins with hidden grasping tentacles.",
                    "Vast numbers of them drift through cold polar seas."
                 ], glows: false, accent: Sea.coral),
        Creature(id: "hammerhead", name: "Great Hammerhead", sci: "Sphyrna mokarran",
                 zone: 0, appear: 85, rangeTop: 0, rangeBottom: 300,
                 size: "Up to 6 m", lengthMeters: 4.5, diet: "Rays, fish, squid",
                 facts: [
                    "Its wide head spreads its electric sensors, sweeping the seabed like a detector.",
                    "Eyes at each end of the 'hammer' give it superb all-round vision.",
                    "It pins stingrays to the bottom with the flat of its head."
                 ], glows: false, accent: Sea.aqua),
        Creature(id: "whale_shark", name: "Whale Shark", sci: "Rhincodon typus",
                 zone: 0, appear: 130, rangeTop: 0, rangeBottom: 1900,
                 size: "Up to 18 m", lengthMeters: 12.0, diet: "Plankton, small fish",
                 facts: [
                    "The largest fish alive — and a harmless filter feeder.",
                    "The pattern of spots on its back is unique, like a fingerprint.",
                    "It can filter over 6,000 litres of water an hour through its gills."
                 ], glows: false, accent: Sea.teal),
        Creature(id: "orca", name: "Orca", sci: "Orcinus orca",
                 zone: 0, appear: 175, rangeTop: 0, rangeBottom: 300,
                 size: "Up to 9 m", lengthMeters: 7.0, diet: "Fish, seals, whales",
                 facts: [
                    "The ocean's top predator, hunting in coordinated family pods.",
                    "Each pod has its own dialect of calls, passed down through generations.",
                    "Different populations specialise — some eat only fish, others only mammals."
                 ], glows: false, accent: Sea.ink),

        // → Twilight section
        Creature(id: "comb_jelly", name: "Comb Jelly", sci: "Ctenophora",
                 zone: 1, appear: 240, rangeTop: 0, rangeBottom: 3000,
                 size: "1–15 cm", lengthMeters: 0.08, diet: "Plankton, tiny larvae",
                 facts: [
                    "Rows of beating hair-like combs push it along, splitting light into rainbows.",
                    "That shimmer is not bioluminescence — it is light scattering off moving combs.",
                    "Many species do also glow blue-green when disturbed."
                 ], glows: true, accent: Sea.glow),
        Creature(id: "nautilus", name: "Chambered Nautilus", sci: "Nautilus pompilius",
                 zone: 1, appear: 300, rangeTop: 100, rangeBottom: 700,
                 size: "Shell up to 25 cm", lengthMeters: 0.25, diet: "Shrimp, carrion",
                 facts: [
                    "Its coiled shell has survived almost unchanged for hundreds of millions of years.",
                    "It floats by adjusting gas in the sealed chambers behind it.",
                    "Around ninety small tentacles feel for food in the dark."
                 ], glows: false, accent: Sea.gold),
        Creature(id: "siphonophore", name: "Giant Siphonophore", sci: "Praya dubia",
                 zone: 1, appear: 480, rangeTop: 200, rangeBottom: 1000,
                 size: "Up to 40 m", lengthMeters: 30.0, diet: "Small fish, crustaceans",
                 facts: [
                    "It can stretch longer than a blue whale — among the longest animals on Earth.",
                    "Like the man o' war it is a colony of specialised bodies acting as one.",
                    "A disturbed colony ripples with waves of blue light."
                 ], glows: true, accent: Sea.glow),
        Creature(id: "glass_squid", name: "Glass Squid", sci: "Cranchiidae",
                 zone: 1, appear: 600, rangeTop: 200, rangeBottom: 2000,
                 size: "Up to 30 cm", lengthMeters: 0.3, diet: "Small fish, crustaceans",
                 facts: [
                    "Almost completely transparent, so predators look straight through it.",
                    "Only its eyes and gut stay opaque — and light organs hide their shadow.",
                    "Threatened, it pulls its head and arms inside its body cavity."
                 ], glows: true, accent: Sea.glow),
        Creature(id: "coelacanth", name: "Coelacanth", sci: "Latimeria chalumnae",
                 zone: 1, appear: 700, rangeTop: 150, rangeBottom: 700,
                 size: "Up to 2 m", lengthMeters: 1.8, diet: "Fish, squid",
                 facts: [
                    "Thought extinct for 65 million years until one was caught alive in 1938.",
                    "Its fleshy, limb-like fins hint at how animals first walked onto land.",
                    "It rests in caves by day and drifts out to hunt at night."
                 ], glows: false, accent: Sea.kelp),
        Creature(id: "oarfish", name: "Giant Oarfish", sci: "Regalecus glesne",
                 zone: 1, appear: 800, rangeTop: 200, rangeBottom: 1000,
                 size: "Up to 8 m", lengthMeters: 8.0, diet: "Plankton, small squid",
                 facts: [
                    "The longest bony fish in the sea, ribboned in silver with a scarlet crest.",
                    "Sightings of it washed ashore inspired old tales of sea serpents.",
                    "It swims upright, head to the surface, rippling a long fin."
                 ], glows: false, accent: Sea.aqua),

        // → Midnight section
        Creature(id: "frilled_shark", name: "Frilled Shark", sci: "Chlamydoselachus anguineus",
                 zone: 2, appear: 1150, rangeTop: 500, rangeBottom: 1500,
                 size: "Up to 2 m", lengthMeters: 1.7, diet: "Squid, fish",
                 facts: [
                    "An eel-shaped shark with a body plan little changed in millions of years.",
                    "Three hundred needle teeth in trident rows hold soft squid fast.",
                    "It may strike by lunging forward like a snake."
                 ], glows: false, accent: Sea.violet),
        Creature(id: "goblin_shark", name: "Goblin Shark", sci: "Mitsukurina owstoni",
                 zone: 2, appear: 1250, rangeTop: 270, rangeBottom: 1300,
                 size: "Up to 4 m", lengthMeters: 3.5, diet: "Fish, squid, crustaceans",
                 facts: [
                    "Its jaws shoot forward out of its head to snatch prey — then fold back.",
                    "Pink colour comes from blood vessels showing through translucent skin.",
                    "A long flat snout is packed with sensors for hunting in the dark."
                 ], glows: false, accent: Sea.coral),

        // → Abyssal section
        Creature(id: "spider_crab", name: "Japanese Spider Crab", sci: "Macrocheira kaempferi",
                 zone: 3, appear: 4600, rangeTop: 50, rangeBottom: 600,
                 size: "Leg span up to 3.7 m", lengthMeters: 3.7, diet: "Carrion, shellfish, plants",
                 facts: [
                    "The largest arthropod alive — its legs can span nearly four metres.",
                    "Some are believed to live for a century on the cold seabed.",
                    "It decorates its shell with sponges to disappear against the bottom."
                 ], glows: false, accent: Sea.gold),
```

**Note on `spider_crab`:** its real habitat (50–600 m) is shallower than the abyssal section it is listed under. Keep `rangeTop`/`rangeBottom` factual as written, but set `zone: 1` and `appear: 560` instead of the values above, so the app never claims it lives in the abyss. Use:

```swift
        Creature(id: "spider_crab", name: "Japanese Spider Crab", sci: "Macrocheira kaempferi",
                 zone: 1, appear: 560, rangeTop: 50, rangeBottom: 600,
                 size: "Leg span up to 3.7 m", lengthMeters: 3.7, diet: "Carrion, shellfish, plants",
                 facts: [
                    "The largest arthropod alive — its legs can span nearly four metres.",
                    "Some are believed to live for a century on the cold seabed.",
                    "It decorates its shell with sponges to disappear against the bottom."
                 ], glows: false, accent: Sea.gold),
```

- [ ] **Step 2: Add the 6 biomes**

Append to the `Ocean` enum, after the `landmarks` array:

```swift
    // MARK: - Biomes

    static let biomes: [Biome] = [
        Biome(id: "reef", name: "Coral Reef", depth: 20, art: "scene_reef",
              blurb: "Built by tiny animals over centuries, reefs cover a sliver of the seafloor yet shelter around a quarter of all marine species.",
              accent: Sea.coral),
        Biome(id: "kelp", name: "Kelp Forest", depth: 30, art: "scene_kelp",
              blurb: "Cool-water forests where kelp can grow half a metre a day, sheltering otters, fish and urchins among the stipes.",
              accent: Sea.kelp),
        Biome(id: "whalefall", name: "Whale Fall", depth: 2_500, art: "scene_whalefall",
              blurb: "When a whale dies and sinks, its body feeds a whole community for decades — first scavengers, then bone-eating worms.",
              accent: Sea.ink),
        Biome(id: "vent", name: "Hydrothermal Vents", depth: 2_400, art: "scene_vent",
              blurb: "Mineral chimneys gush water hot enough to melt lead. Bacteria here feed on chemistry, not sunlight, powering the whole vent community.",
              accent: Sea.coral),
        Biome(id: "brine", name: "Brine Pool", depth: 1_000, art: "scene_brine",
              blurb: "Water so salty it sinks into pools with their own shorelines and waves — an underwater lake, lethal to most animals that swim in.",
              accent: Sea.teal),
        Biome(id: "trench", name: "Trench Floor", depth: 10_000, art: "scene_trench",
              blurb: "Steep walls funnel sinking food down into the deepest places on Earth, where scavengers gather under crushing pressure.",
              accent: Sea.violet),
    ]

    static func biome(_ id: String) -> Biome? { biomes.first { $0.id == id } }
```

- [ ] **Step 3: Add the 6 new awards**

Append these inside the `Ocean.awards` array (before its closing `]`):

```swift
        Award(id: "vessel_all", title: "Full Fleet",
              detail: "Unlock every vessel.",
              accent: Sea.gold,
              requirement: { Ranks.unlockedVessels(xp: $0.xp).count >= Vessels.all.count },
              progress: { min(1, Double(Ranks.unlockedVessels(xp: $0.xp).count) / Double(Vessels.all.count)) }),
        Award(id: "rank_mid", title: "Submersible Pilot",
              detail: "Reach the rank of Submersible Pilot.",
              accent: Sea.violet,
              requirement: { $0.rank.number >= 5 },
              progress: { min(1, Double($0.xp) / 300.0) }),
        Award(id: "rank_max", title: "Master of the Abyss",
              detail: "Reach the highest diver rank.",
              accent: Sea.gold,
              requirement: { $0.rank.number >= 8 },
              progress: { min(1, Double($0.xp) / 900.0) }),
        Award(id: "exp_first", title: "Expedition Log",
              detail: "Complete your first daily expedition.",
              accent: Sea.aqua,
              requirement: { !$0.expeditionHistory.isEmpty },
              progress: { $0.expeditionHistory.isEmpty ? 0 : 1 }),
        Award(id: "exp_five", title: "Seasoned Diver",
              detail: "Complete 5 daily expeditions.",
              accent: Sea.teal,
              requirement: { $0.expeditionHistory.count >= 5 },
              progress: { min(1, Double($0.expeditionHistory.count) / 5.0) }),
        Award(id: "zones_all", title: "Every Layer",
              detail: "Visit all five ocean zones.",
              accent: Sea.kelp,
              requirement: { $0.zonesVisited.count >= Ocean.zones.count },
              progress: { min(1, Double($0.zonesVisited.count) / Double(Ocean.zones.count)) }),
```

- [ ] **Step 4: Compile**

Run:
```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue" && xcodebuild -project "Deep Blue.xcodeproj" -scheme "Deep Blue" -destination 'platform=iOS Simulator,name=iPhone 17' -derivedDataPath build/ build > /tmp/t5.log 2>&1; echo "EXIT $?"; grep -E "BUILD SUCCEEDED|BUILD FAILED" /tmp/t5.log | tail -1
```
Expected: `BUILD SUCCEEDED`

- [ ] **Step 5: Verify counts at runtime**

Reinstall, launch, screenshot the Dive hub:
```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
xcrun simctl terminate booted com.deepbluedive.app 2>/dev/null
xcrun simctl install booted "build/Build/Products/Debug-iphonesimulator/Deep Blue.app"
xcrun simctl launch booted com.deepbluedive.app
sleep 9; xcrun simctl io booted screenshot /tmp/t5shot.png
```
Expected in the screenshot: the HUD reads `… / 50 found` (not 36). Every new creature must have art — a blank slot means a slug mismatch between Task 4 and this task.

- [ ] **Step 6: Commit**

```bash
git add "Deep Blue/OceanData.swift"
git commit -m "Add 14 creatures, 6 biomes and 6 new awards"
```

---

## Task 6: Living ocean — swim motion, parallax, discovery burst, lamp cone

**Files:**
- Modify: `Deep Blue/DeepEffects.swift`

**Interfaces:**
- Produces:
  - `struct SwimMotion: ViewModifier { let seed: Int; let enabled: Bool }` and `extension View { func swimming(seed: Int, enabled: Bool) -> some View }`
  - `struct ParallaxDrift: View { var layers: Int; var enabled: Bool }`
  - `struct DiscoveryBurst: View { let color: Color; @Binding var trigger: Bool }`
  - `struct LampCone: View { var radius: CGFloat; var darkness: Double }`
  - `func lampDarkness(atDepth: Double) -> Double`

- [ ] **Step 1: Append the new effects to `DeepEffects.swift`**

```swift
// MARK: - Swim motion (each creature drifts on its own rhythm)

struct SwimMotion: ViewModifier {
    let seed: Int
    let enabled: Bool

    private var phase: Double { Double(abs(seed) % 100) / 100.0 * 6.283 }
    private var period: Double { 3.4 + Double(abs(seed) % 7) * 0.45 }

    func body(content: Content) -> some View {
        if enabled {
            TimelineView(.animation) { timeline in
                let t = timeline.date.timeIntervalSinceReferenceDate
                let a = (t / period) * 6.283 + phase
                content
                    .offset(x: CGFloat(sin(a) * 9), y: CGFloat(cos(a * 0.7) * 5))
                    .rotationEffect(.degrees(sin(a * 0.85) * 2.6))
            }
        } else {
            content
        }
    }
}

extension View {
    /// Gentle, per-creature drifting so the scene never looks nailed down.
    func swimming(seed: Int, enabled: Bool) -> some View {
        modifier(SwimMotion(seed: seed, enabled: enabled))
    }
}

// MARK: - Parallax particle layers

struct ParallaxDrift: View {
    var layers: Int = 3
    var enabled: Bool = true

    private struct Mote { let x: Double, y: Double, r: Double }

    private static func motes(_ n: Int, seedStart: UInt64) -> [Mote] {
        var seed = seedStart
        func rnd() -> Double {
            seed ^= seed << 13; seed ^= seed >> 7; seed ^= seed << 17
            return Double(seed % 10_000) / 10_000.0
        }
        return (0..<n).map { _ in Mote(x: rnd(), y: rnd(), r: 0.7 + rnd() * 2.3) }
    }

    private let sets: [[Mote]] = [
        ParallaxDrift.motes(26, seedStart: 0x2545F4914F6CDD1D),
        ParallaxDrift.motes(20, seedStart: 0x9E3779B97F4A7C15),
        ParallaxDrift.motes(14, seedStart: 0xD1B54A32D192ED03),
    ]

    var body: some View {
        if enabled {
            TimelineView(.animation) { timeline in
                Canvas { ctx, size in
                    let t = timeline.date.timeIntervalSinceReferenceDate
                    for (i, set) in sets.prefix(max(1, layers)).enumerated() {
                        let speed = 0.010 + Double(i) * 0.012
                        let alpha = 0.30 - Double(i) * 0.07
                        let scale = 1.0 + Double(i) * 0.5
                        for m in set {
                            let prog = (m.y + t * speed).truncatingRemainder(dividingBy: 1.0)
                            let y = prog * size.height
                            let x = m.x * size.width + sin(t * 0.3 + m.x * 9) * 6
                            let r = m.r * scale
                            ctx.fill(Path(ellipseIn: CGRect(x: x - r, y: y - r, width: r * 2, height: r * 2)),
                                     with: .color(Sea.ink.opacity(alpha)))
                        }
                    }
                }
            }
            .allowsHitTesting(false)
        }
    }
}

// MARK: - Discovery burst

struct DiscoveryBurst: View {
    let color: Color
    @Binding var trigger: Bool
    @State private var wave: CGFloat = 0
    @State private var flash: Double = 0

    var body: some View {
        ZStack {
            Circle().stroke(color.opacity(0.85), lineWidth: 3)
                .scaleEffect(0.2 + wave * 1.6)
                .opacity(Double(1 - wave))
            Circle().stroke(color.opacity(0.5), lineWidth: 2)
                .scaleEffect(0.2 + wave * 2.4)
                .opacity(Double(1 - wave) * 0.7)
            Circle().fill(color.opacity(flash * 0.55))
                .blur(radius: 26)
                .scaleEffect(1.4)
        }
        .allowsHitTesting(false)
        .onChange(of: trigger) { fired in
            guard fired else { return }
            wave = 0; flash = 1
            withAnimation(.easeOut(duration: 0.75)) { wave = 1 }
            withAnimation(.easeOut(duration: 0.55)) { flash = 0 }
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.8) { trigger = false }
        }
    }
}

// MARK: - Lamp cone (the descent's darkness)

/// 0 at the surface, ramping to 1 between 400 m and 1,000 m.
func lampDarkness(atDepth depth: Double) -> Double {
    if depth <= 400 { return 0 }
    if depth >= 1_000 { return 1 }
    return (depth - 400) / 600.0
}

struct LampCone: View {
    var radius: CGFloat
    var darkness: Double      // 0...1

    var body: some View {
        GeometryReader { geo in
            let w = geo.size.width, h = geo.size.height
            ZStack {
                // The dark water, punched through by the lamp.
                Color.black.opacity(0.92 * darkness)
                    .mask(
                        ZStack {
                            Rectangle()
                            RadialGradient(colors: [.black, .black.opacity(0.35), .clear],
                                           center: .center,
                                           startRadius: 0,
                                           endRadius: max(1, radius))
                                .blendMode(.destinationOut)
                        }
                        .compositingGroup()
                    )
                // A soft warm halo where the beam falls.
                if radius > 0 && darkness > 0.05 {
                    RadialGradient(colors: [Sea.glow.opacity(0.16 * darkness), .clear],
                                   center: .center, startRadius: 0, endRadius: max(1, radius * 0.9))
                }
            }
            .frame(width: w, height: h)
        }
        .allowsHitTesting(false)
        .ignoresSafeArea()
    }
}
```

- [ ] **Step 2: Compile**

Run:
```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue" && xcodebuild -project "Deep Blue.xcodeproj" -scheme "Deep Blue" -destination 'platform=iOS Simulator,name=iPhone 17' -derivedDataPath build/ build > /tmp/t6.log 2>&1; echo "EXIT $?"; grep -E "BUILD SUCCEEDED|BUILD FAILED" /tmp/t6.log | tail -1
```
Expected: `BUILD SUCCEEDED`. If `onChange(of:)` warns about deprecation, ignore — the two-parameter iOS 17 form is **not** allowed here (iOS 15.6 floor).

- [ ] **Step 3: Commit**

```bash
git add "Deep Blue/DeepEffects.swift"
git commit -m "Add swim motion, parallax, discovery burst and lamp cone effects"
```

---

## Task 7: Descent mode + vessel picker

**Files:**
- Create: `Deep Blue/DescentView.swift`, `Deep Blue/VesselPickerView.swift`
- Modify: `Deep Blue.xcodeproj/project.pbxproj`

**Interfaces:**
- Consumes: `Vessels`, `Vessel`, `Ranks`, `DeepStore`, `LampCone`, `lampDarkness`, `SwimMotion`, `waterColor(_:)` (already `internal` in `DiveView.swift`), `OceanPhysics`, `CreatureDetailView`.
- Produces: `struct DescentView: View`, `struct VesselPickerView: View`.

- [ ] **Step 1: Create `Deep Blue/VesselPickerView.swift`**

```swift
import SwiftUI

struct VesselPickerView: View {
    @EnvironmentObject var store: DeepStore
    @Environment(\.presentationMode) var presentation

    var body: some View {
        ZStack {
            LinearGradient(colors: [Sea.deep, Sea.trench], startPoint: .top, endPoint: .bottom)
                .ignoresSafeArea()
            ScrollView(showsIndicators: false) {
                VStack(spacing: 14) {
                    VStack(spacing: 4) {
                        Text("Choose Your Craft").font(.deepTitle(24)).foregroundColor(Sea.ink)
                        Text("Each vessel reaches only so deep.")
                            .font(.deepBody(13)).foregroundColor(Sea.inkSoft)
                    }
                    .padding(.top, 18)

                    ForEach(Vessels.all) { v in
                        let unlocked = store.isVesselUnlocked(v)
                        Button {
                            guard unlocked else { return }
                            Haptics.light()
                            store.selectVessel(v.id)
                            presentation.wrappedValue.dismiss()
                        } label: {
                            VesselRow(vessel: v, unlocked: unlocked,
                                      selected: store.currentVesselID == v.id)
                        }
                        .buttonStyle(PressStyle())
                        .disabled(!unlocked)
                    }
                    Color.clear.frame(height: 24)
                }
                .frame(maxWidth: Adaptive.contentWidth)
                .frame(maxWidth: .infinity)
                .padding(.horizontal, 16)
            }
        }
        .overlay(alignment: .topTrailing) {
            Button { presentation.wrappedValue.dismiss() } label: {
                ZStack {
                    Circle().fill(Sea.trench.opacity(0.6))
                        .overlay(Circle().stroke(Sea.stroke.opacity(0.5), lineWidth: 1))
                    CloseGlyph(size: 16, color: Sea.ink)
                }
                .frame(width: 34, height: 34)
            }
            .padding(.trailing, 20).padding(.top, 16)
        }
    }
}

private struct VesselRow: View {
    let vessel: Vessel
    let unlocked: Bool
    let selected: Bool

    var body: some View {
        HStack(spacing: 14) {
            ZStack {
                RoundedRectangle(cornerRadius: 14, style: .continuous)
                    .fill(unlocked ? vessel.accent.opacity(0.14) : Sea.panel.opacity(0.5))
                ArtImage(name: vessel.art, contentMode: .fit, fallback: .clear)
                    .frame(height: 54).padding(6)
                    .opacity(unlocked ? 1 : 0.35)
                if !unlocked { LockGlyph(size: 22, color: Sea.inkFaint) }
            }
            .frame(width: 84, height: 68)

            VStack(alignment: .leading, spacing: 3) {
                Text(vessel.name)
                    .font(.deepBody(16, .bold))
                    .foregroundColor(unlocked ? Sea.ink : Sea.inkSoft)
                Text(unlocked ? vessel.note : "Unlocks at \(Ranks.all[vessel.unlockRank - 1].name)")
                    .font(.deepBody(11.5)).foregroundColor(Sea.inkSoft)
                    .fixedSize(horizontal: false, vertical: true)
                Pill(text: "Reaches \(vessel.limitLabel)", color: vessel.accent)
                    .padding(.top, 2)
            }
            Spacer(minLength: 0)
            if selected { CheckGlyph(size: 20, color: vessel.accent) }
        }
        .padding(12)
        .background(
            RoundedRectangle(cornerRadius: 18, style: .continuous)
                .fill(selected ? vessel.accent.opacity(0.10) : Sea.card.opacity(0.7))
                .overlay(RoundedRectangle(cornerRadius: 18, style: .continuous)
                    .stroke(selected ? vessel.accent.opacity(0.55) : Sea.stroke.opacity(0.3), lineWidth: 1))
        )
    }
}
```

- [ ] **Step 2: Create `Deep Blue/DescentView.swift`**

```swift
import SwiftUI

// The cinematic descent: the craft sinks on its own, the water darkens until
// only the lamp is left, and creatures drift into the beam out of the black.

struct DescentView: View {
    @EnvironmentObject var store: DeepStore
    @Environment(\.presentationMode) var presentation

    @State private var depth: Double = 0
    @State private var speed: Double = 1          // 0 = paused, 1 = 1x, 2 = 2x
    @State private var ascending = false
    @State private var selected: Creature? = nil
    @State private var atLimit = false
    @State private var showPicker = false
    @State private var burst = false
    @State private var lastTick: Date = Date()

    private var vessel: Vessel { store.vessel }
    private var limit: Double { Double(vessel.limit) }
    private var darkness: Double { lampDarkness(atDepth: depth) }

    /// Creatures whose home depth is close enough to be in view right now.
    private var nearby: [Creature] {
        Ocean.creatures.filter { abs(Double($0.appear) - depth) < 190 }
    }

    var body: some View {
        ZStack {
            LinearGradient(colors: [waterColor(depth), waterColor(depth + 700)],
                           startPoint: .top, endPoint: .bottom)
                .ignoresSafeArea()

            ParallaxDrift(layers: 3, enabled: store.bubblesOn).ignoresSafeArea()
            if depth < 900 {
                BubblesLayer(count: 14, tint: Sea.glow, speed: 1.1, enabled: store.bubblesOn)
                    .ignoresSafeArea()
            }

            creatureField
            LampCone(radius: vessel.lampRadius > 0 ? vessel.lampRadius : 420, darkness: darkness)

            VStack {
                hud
                Spacer()
                if atLimit { limitCard } else { controls }
            }
            .padding(.horizontal, 16)
            .padding(.top, 8)
            .padding(.bottom, 22)
            .frame(maxWidth: Adaptive.contentWidth)
        }
        .onAppear { lastTick = Date(); depth = 0 }
        .background(tick)
        .sheet(item: $selected) { c in
            CreatureDetailView(creature: c).environmentObject(store)
        }
        .sheet(isPresented: $showPicker) {
            VesselPickerView().environmentObject(store)
        }
    }

    // Drives the sink using wall-clock deltas so pausing is exact.
    private var tick: some View {
        TimelineView(.animation) { timeline in
            Color.clear.onChange(of: timeline.date) { now in
                let dt = min(0.05, now.timeIntervalSince(lastTick))
                lastTick = now
                guard !atLimit, speed > 0 || ascending else { return }
                if ascending {
                    depth = max(0, depth - vessel.sinkSpeed * 2.2 * dt)
                    if depth <= 0 { ascending = false }
                } else {
                    let next = depth + vessel.sinkSpeed * speed * dt
                    if next >= limit {
                        depth = limit
                        atLimit = true
                        store.recordDepth(Int(limit))
                        if store.hapticsOn { Haptics.light() }
                    } else {
                        depth = next
                        if Int(depth) % 100 == 0 { store.recordDepth(Int(depth)) }
                    }
                }
            }
        }
        .frame(width: 0, height: 0)
    }

    // MARK: - Creatures in the beam

    private var creatureField: some View {
        GeometryReader { geo in
            let w = geo.size.width, h = geo.size.height
            ForEach(Array(nearby.enumerated()), id: \.element.id) { idx, c in
                let delta = Double(c.appear) - depth              // -190...190
                let y = h * 0.5 + CGFloat(delta / 190.0) * h * 0.42
                let x = idx.isMultiple(of: 2) ? w * 0.32 : w * 0.68
                let found = store.isDiscovered(c.id)
                Button {
                    if store.discover(c.id) { burst = true }
                    selected = c
                } label: {
                    ZStack {
                        if found || c.glows {
                            RadialGlow(color: c.accent, radius: 66,
                                       opacity: c.glows ? 0.45 : 0.22)
                        }
                        CreatureImage(slug: c.id)
                            .frame(height: 104)
                            .modifier(DescentDim(active: !found))
                        if found {
                            Text(c.name)
                                .font(.deepBody(11.5, .bold)).foregroundColor(Sea.ink)
                                .lineLimit(1).minimumScaleFactor(0.7)
                                .padding(.horizontal, 9).padding(.vertical, 3)
                                .background(Capsule().fill(Sea.trench.opacity(0.6)))
                                .offset(y: 64)
                        }
                    }
                }
                .buttonStyle(PressStyle())
                .swimming(seed: c.id.hashValue, enabled: store.bubblesOn)
                .position(x: x, y: y)
                .opacity(abs(delta) > 170 ? 0 : 1)
            }
            DiscoveryBurst(color: Sea.glow, trigger: $burst)
                .frame(width: 260, height: 260)
                .position(x: w * 0.5, y: h * 0.5)
        }
    }

    // MARK: - HUD

    private var hud: some View {
        VStack(spacing: 8) {
            HStack(alignment: .top) {
                VStack(alignment: .leading, spacing: 1) {
                    Text(vessel.name.uppercased())
                        .font(.deepBody(10, .bold)).foregroundColor(vessel.accent).tracking(2)
                    Text("\(Int(depth).formattedDepth) m")
                        .font(.deepMono(32, .bold)).foregroundColor(Sea.ink)
                    Text("Hull limit \(vessel.limitLabel)")
                        .font(.deepBody(11)).foregroundColor(Sea.inkFaint)
                }
                Spacer()
                Button { presentation.wrappedValue.dismiss() } label: {
                    ZStack {
                        Circle().fill(Sea.trench.opacity(0.6))
                            .overlay(Circle().stroke(Sea.stroke.opacity(0.5), lineWidth: 1))
                        CloseGlyph(size: 15, color: Sea.ink)
                    }
                    .frame(width: 32, height: 32)
                }
            }
            HStack(spacing: 8) {
                stat(AnyView(ThermoGlyph(size: 15, color: Sea.coral)), OceanPhysics.tempText(at: Int(depth)))
                stat(AnyView(GaugeGlyph(size: 15, color: Sea.teal)), OceanPhysics.pressureText(at: Int(depth)))
                stat(AnyView(SunGlyph(size: 15, color: Sea.gold)), OceanPhysics.lightText(at: Int(depth)))
            }
            if store.hasProfile {
                Text(BodyScale.pressurePhrase(depth: Int(depth),
                                              heightCM: store.profileHeightCM,
                                              weightKG: store.profileWeightKG))
                    .font(.deepBody(12, .semibold)).foregroundColor(Sea.glow)
                    .frame(maxWidth: .infinity, alignment: .leading)
            }
        }
        .padding(.horizontal, 14).padding(.vertical, 12)
        .background(
            RoundedRectangle(cornerRadius: 20, style: .continuous)
                .fill(Sea.trench.opacity(0.62))
                .overlay(RoundedRectangle(cornerRadius: 20, style: .continuous)
                    .stroke(Sea.stroke.opacity(0.35), lineWidth: 1))
        )
    }

    private func stat(_ icon: AnyView, _ value: String) -> some View {
        HStack(spacing: 6) {
            icon
            Text(value).font(.deepBody(12.5, .semibold)).foregroundColor(Sea.ink)
                .lineLimit(1).minimumScaleFactor(0.7)
        }
        .frame(maxWidth: .infinity).padding(.vertical, 6)
        .background(RoundedRectangle(cornerRadius: 11, style: .continuous).fill(Sea.panel.opacity(0.55)))
    }

    // MARK: - Controls

    private var controls: some View {
        HStack(spacing: 10) {
            ctlButton(speed == 0 ? "Resume" : "Pause", Sea.panelSoft) {
                Haptics.soft(); speed = speed == 0 ? 1 : 0; ascending = false
            }
            ctlButton(speed >= 2 ? "1×" : "2×", Sea.panelSoft) {
                Haptics.soft(); speed = speed >= 2 ? 1 : 2; ascending = false
            }
            ctlButton("Ascend", vessel.accent, dark: true) {
                Haptics.soft(); ascending = true; speed = 0
            }
        }
    }

    private func ctlButton(_ title: String, _ tint: Color, dark: Bool = false,
                           _ action: @escaping () -> Void) -> some View {
        Button(action: action) {
            Text(title)
                .font(.deepBody(15, .bold))
                .foregroundColor(dark ? Sea.trench : Sea.ink)
                .frame(maxWidth: .infinity).padding(.vertical, 13)
                .background(Capsule().fill(tint))
        }
        .buttonStyle(PressStyle())
    }

    private var limitCard: some View {
        VStack(spacing: 12) {
            Text("Hull limit reached").font(.deepTitle(20)).foregroundColor(Sea.ink)
            Text(deeperHint)
                .font(.deepBody(13)).foregroundColor(Sea.inkSoft)
                .multilineTextAlignment(.center)
                .fixedSize(horizontal: false, vertical: true)
            HStack(spacing: 10) {
                ctlButton("Ascend", Sea.panelSoft) {
                    atLimit = false; ascending = true; speed = 0
                }
                ctlButton("Change craft", vessel.accent, dark: true) { showPicker = true }
            }
        }
        .padding(16)
        .background(
            RoundedRectangle(cornerRadius: 22, style: .continuous)
                .fill(Sea.trench.opacity(0.8))
                .overlay(RoundedRectangle(cornerRadius: 22, style: .continuous)
                    .stroke(vessel.accent.opacity(0.5), lineWidth: 1))
        )
    }

    private var deeperHint: String {
        let deeper = Vessels.all.filter { $0.limit > vessel.limit }
        guard let next = deeper.min(by: { $0.limit < $1.limit }) else {
            return "This is the deepest place in any ocean. There is nothing below."
        }
        if store.isVesselUnlocked(next) {
            return "\(next.name) can take you to \(next.limitLabel)."
        }
        return "\(next.name) reaches \(next.limitLabel) — it unlocks at \(Ranks.all[next.unlockRank - 1].name)."
    }
}

private struct DescentDim: ViewModifier {
    let active: Bool
    func body(content: Content) -> some View {
        if active { content.colorMultiply(Color(red: 0.07, green: 0.12, blue: 0.2)).opacity(0.95) }
        else { content }
    }
}
```

- [ ] **Step 3: Register both files in `project.pbxproj`**

Use ids `…0034`/`…0134` (DescentView) and `…0035`/`…0135` (VesselPickerView), in all four sections, exactly as in Task 1 Step 4.

- [ ] **Step 4: Compile**

Run:
```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue" && xcodebuild -project "Deep Blue.xcodeproj" -scheme "Deep Blue" -destination 'platform=iOS Simulator,name=iPhone 17' -derivedDataPath build/ build > /tmp/t7.log 2>&1; echo "EXIT $?"; grep -E "BUILD SUCCEEDED|BUILD FAILED" /tmp/t7.log | tail -1
```
Expected: `BUILD SUCCEEDED`. `Adaptive.contentWidth` does not exist yet — if the compiler flags it, add Task 8's `Adaptive.swift` first, then return here.

- [ ] **Step 5: Commit**

```bash
git add "Deep Blue/DescentView.swift" "Deep Blue/VesselPickerView.swift" "Deep Blue.xcodeproj/project.pbxproj"
git commit -m "Add cinematic descent mode and vessel picker"
```

---

## Task 8: Adaptive layout helpers + iPad target

**Files:**
- Create: `Deep Blue/Adaptive.swift`
- Modify: `Deep Blue.xcodeproj/project.pbxproj`

**Interfaces:**
- Produces: `enum Adaptive { static var contentWidth: CGFloat; static func columns(_ compact: Int, _ regular: Int, isRegular: Bool) -> Int; static var isPad: Bool }`

- [ ] **Step 1: Create `Deep Blue/Adaptive.swift`**

```swift
import SwiftUI

// Keeps content readable on a 13" iPad without stretching lines of text
// across the whole screen.

enum Adaptive {
    /// Widest a column of content should ever get.
    static let contentWidth: CGFloat = 700

    static var isPad: Bool {
        UIDevice.current.userInterfaceIdiom == .pad
    }

    /// Grid column count for the current size class.
    static func columns(_ compact: Int, _ regular: Int, isRegular: Bool) -> Int {
        isRegular ? regular : compact
    }

    /// A grid of `n` flexible columns.
    static func grid(_ n: Int, spacing: CGFloat = 12) -> [GridItem] {
        Array(repeating: GridItem(.flexible(), spacing: spacing), count: max(1, n))
    }
}

extension View {
    /// Centres a scrolling column and caps its width on iPad.
    func adaptiveColumn() -> some View {
        self
            .frame(maxWidth: Adaptive.contentWidth)
            .frame(maxWidth: .infinity)
    }
}
```

- [ ] **Step 2: Register `Adaptive.swift` in `project.pbxproj`**

Ids `…0036`/`…0136`, all four sections, as in Task 1 Step 4.

- [ ] **Step 3: Switch the target to iPhone + iPad**

In `Deep Blue.xcodeproj/project.pbxproj`, in **both** `DB0000000000000000000803 /* Debug */` and `DB0000000000000000000804 /* Release */`, change:

```
				TARGETED_DEVICE_FAMILY = 1;
```
to:
```
				TARGETED_DEVICE_FAMILY = "1,2";
```

Verify both were changed:
```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
grep -c 'TARGETED_DEVICE_FAMILY = "1,2"' "Deep Blue.xcodeproj/project.pbxproj"
```
Expected: `2`

- [ ] **Step 4: Apply adaptive width to the four scrolling tabs**

In each of `JournalView.swift`, `ZonesView.swift`, `MoreView.swift`, apply `.adaptiveColumn()` to the top-level `VStack` inside the `ScrollView` — i.e. change the existing modifier chain

```swift
                .padding(.horizontal, 16)
                .padding(.top, 8)
```
to
```swift
                .adaptiveColumn()
                .padding(.horizontal, 16)
                .padding(.top, 8)
```

Then make the grids size-class aware. In `JournalView.swift`, replace the stored `cols` property and add an environment read:

```swift
    @Environment(\.horizontalSizeClass) private var hSize
    private var cols: [GridItem] {
        Adaptive.grid(Adaptive.columns(2, 4, isRegular: hSize == .regular))
    }
```

In `MoreView.swift`, replace `awardCols` the same way:

```swift
    @Environment(\.horizontalSizeClass) private var hSize
    private var awardCols: [GridItem] {
        Adaptive.grid(Adaptive.columns(2, 3, isRegular: hSize == .regular))
    }
```

- [ ] **Step 5: Build for iPhone and iPad**

First confirm the iPad simulator's exact name:
```bash
xcrun simctl list devices available | grep -i ipad
```
Then build for both (background, poll the log):
```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
xcodebuild -project "Deep Blue.xcodeproj" -scheme "Deep Blue" -destination 'platform=iOS Simulator,name=iPhone 17' -derivedDataPath build/ build > /tmp/t8i.log 2>&1; echo "IPHONE $?"
xcodebuild -project "Deep Blue.xcodeproj" -scheme "Deep Blue" -destination 'platform=iOS Simulator,name=<exact iPad name from above>' -derivedDataPath build-ipad/ build > /tmp/t8p.log 2>&1; echo "IPAD $?"
grep -E "BUILD SUCCEEDED|BUILD FAILED" /tmp/t8i.log /tmp/t8p.log | tail -2
```
Expected: `BUILD SUCCEEDED` for both.

- [ ] **Step 6: Commit**

```bash
git add "Deep Blue/Adaptive.swift" "Deep Blue/JournalView.swift" "Deep Blue/ZonesView.swift" "Deep Blue/MoreView.swift" "Deep Blue.xcodeproj/project.pbxproj"
git commit -m "Add adaptive layout helpers and enable iPad support"
```

---

## Task 9: Sonar + Dive hub (expedition card, Begin Descent, swimming creatures)

**Files:**
- Create: `Deep Blue/SonarLayer.swift`
- Modify: `Deep Blue/DiveView.swift`
- Modify: `Deep Blue.xcodeproj/project.pbxproj`

**Interfaces:**
- Consumes: `ExpeditionEngine`, `DeepStore.todayExpedition/expeditionFraction/expeditionComplete/refreshExpedition`, `Vessels`, `DescentView`, `SwimMotion`, `DiscoveryBurst`.
- Produces: `struct SonarPingButton: View`, `struct SonarRings: View`, `struct ExpeditionCard: View`.

- [ ] **Step 1: Create `Deep Blue/SonarLayer.swift`**

```swift
import SwiftUI

// A ping that briefly shows where undiscovered creatures are hiding.
// It is a comfort feature, not a resource — no cooldown, no penalty.

struct SonarRings: View {
    @Binding var active: Bool
    var color: Color = Sea.aqua
    @State private var wave: CGFloat = 0

    var body: some View {
        ZStack {
            ForEach(0..<3, id: \.self) { i in
                Circle()
                    .stroke(color.opacity(0.5 - Double(i) * 0.13), lineWidth: 2)
                    .scaleEffect(0.15 + wave * (1.1 + CGFloat(i) * 0.5))
                    .opacity(Double(1 - wave))
            }
        }
        .allowsHitTesting(false)
        .onChange(of: active) { on in
            guard on else { return }
            wave = 0
            withAnimation(.easeOut(duration: 1.5)) { wave = 1 }
            DispatchQueue.main.asyncAfter(deadline: .now() + 3.0) { active = false }
        }
    }
}

struct SonarPingButton: View {
    @Binding var active: Bool
    var action: () -> Void

    var body: some View {
        Button {
            Haptics.soft()
            active = true
            action()
        } label: {
            ZStack {
                Circle().fill(Sea.trench.opacity(0.55))
                    .overlay(Circle().stroke(Sea.aqua.opacity(0.5), lineWidth: 1))
                Canvas { ctx, size in
                    let c = CGPoint(x: size.width / 2, y: size.height / 2)
                    for i in 0..<3 {
                        let r = size.width * (0.12 + Double(i) * 0.13)
                        var p = Path()
                        p.addArc(center: c, radius: r, startAngle: .degrees(-50),
                                 endAngle: .degrees(50), clockwise: false)
                        ctx.stroke(p, with: .color(Sea.aqua.opacity(1 - Double(i) * 0.25)),
                                   style: .init(lineWidth: 2, lineCap: .round))
                    }
                    ctx.fill(Path(ellipseIn: CGRect(x: c.x - 3, y: c.y - 3, width: 6, height: 6)),
                             with: .color(Sea.aqua))
                }
                .frame(width: 30, height: 30)
            }
            .frame(width: 46, height: 46)
        }
        .buttonStyle(PressStyle())
    }
}

/// The pulsing ring drawn around an undiscovered creature after a ping.
struct SonarHighlight: View {
    var color: Color = Sea.aqua
    @State private var pulse = false

    var body: some View {
        Circle()
            .stroke(color.opacity(0.8), lineWidth: 2)
            .frame(width: 96, height: 96)
            .scaleEffect(pulse ? 1.12 : 0.88)
            .opacity(pulse ? 0.25 : 0.85)
            .onAppear {
                withAnimation(.easeInOut(duration: 0.9).repeatForever(autoreverses: true)) {
                    pulse = true
                }
            }
            .allowsHitTesting(false)
    }
}

// MARK: - Daily expedition card

struct ExpeditionCard: View {
    @EnvironmentObject var store: DeepStore

    var body: some View {
        let exp = store.todayExpedition
        let done = store.expeditionComplete
        return GlassCard(tint: done ? Sea.panel : Sea.card) {
            VStack(alignment: .leading, spacing: 10) {
                HStack(spacing: 8) {
                    Text("EXPEDITION OF THE DAY")
                        .font(.deepBody(10, .bold)).foregroundColor(Sea.gold).tracking(1.5)
                    Spacer()
                    if done { CheckGlyph(size: 17, color: Sea.gold) }
                    else { Text("+\(XPAward.expedition) XP")
                        .font(.deepBody(11, .bold)).foregroundColor(Sea.inkFaint) }
                }
                Text(exp.title).font(.deepTitle(19)).foregroundColor(Sea.ink)
                Text(done ? "Completed — a new objective arrives tomorrow." : exp.detail)
                    .font(.deepBody(13)).foregroundColor(Sea.inkSoft)
                    .fixedSize(horizontal: false, vertical: true)
                if !done {
                    GeometryReader { g in
                        ZStack(alignment: .leading) {
                            Capsule().fill(Sea.stroke.opacity(0.3)).frame(height: 6)
                            Capsule().fill(Sea.gold)
                                .frame(width: g.size.width * CGFloat(store.expeditionFraction), height: 6)
                        }
                    }
                    .frame(height: 6)
                }
            }
        }
    }
}
```

- [ ] **Step 2: Turn the Dive tab into a hub**

In `Deep Blue/DiveView.swift`, add these `@State`s to `DiveView` next to the existing ones:

```swift
    @State private var showDescent = false
    @State private var sonarActive = false
    @State private var sonarDepth: Double = -99_999
    @State private var burst = false
```

Then, inside the `ZStack(alignment: .top)` and **above** the `DiveHUD(...)` line, replace the HUD block so the hub sits under it. Change:

```swift
                DiveHUD(depth: curDepth)
                    .padding(.horizontal, 14).padding(.top, 6)
```
to:
```swift
                VStack(spacing: 10) {
                    DiveHUD(depth: curDepth)
                    if curDepth < 60 {          // the hub only shows near the surface
                        ExpeditionCard()
                        Button {
                            Haptics.light()
                            showDescent = true
                        } label: {
                            HStack(spacing: 12) {
                                ArtImage(name: store.vessel.art, contentMode: .fit, fallback: .clear)
                                    .frame(width: 54, height: 40)
                                VStack(alignment: .leading, spacing: 1) {
                                    Text("Begin Descent")
                                        .font(.deepBody(16, .bold)).foregroundColor(Sea.trench)
                                    Text("\(store.vessel.name) · to \(store.vessel.limitLabel)")
                                        .font(.deepBody(11.5, .semibold)).foregroundColor(Sea.trench.opacity(0.75))
                                }
                                Spacer()
                                ChevronGlyph(pointing: .trailing, size: 16, color: Sea.trench.opacity(0.7))
                            }
                            .padding(.horizontal, 16).padding(.vertical, 12)
                            .background(Capsule().fill(Sea.aqua)
                                .shadow(color: Sea.aqua.opacity(0.45), radius: 12, x: 0, y: 5))
                        }
                        .buttonStyle(PressStyle())
                    }
                }
                .padding(.horizontal, 14).padding(.top, 6)
                .adaptiveColumn()
```

- [ ] **Step 3: Add the sonar button and wire discovery burst**

Still in `DiveView.body`, add these two modifiers to the outer `ZStack` (after `.onDisappear { ... }`):

```swift
            .overlay(alignment: .bottomLeading) {
                SonarPingButton(active: $sonarActive) { sonarDepth = curDepth }
                    .padding(.leading, 18).padding(.bottom, 96)
            }
            .overlay {
                SonarRings(active: $sonarActive)
                    .frame(width: 320, height: 320)
            }
            .fullScreenCover(isPresented: $showDescent) {
                DescentView().environmentObject(store)
            }
            .onAppear { store.refreshExpedition() }
```

- [ ] **Step 4: Make creatures swim and highlight under sonar**

In `DiveView.creatureMarkers(width:center:)`, change the marker construction so it carries motion and the sonar ring. Replace the body of the `ForEach` closure with:

```swift
            let idx = Ocean.byDepth.firstIndex(where: { $0.id == c.id }) ?? 0
            let pinged = sonarActive
                && !store.isDiscovered(c.id)
                && abs(Double(c.appear) - sonarDepth) < 1_500
            ZStack {
                if pinged { SonarHighlight(color: Sea.aqua) }
                CreatureMarker(creature: c,
                               discovered: store.isDiscovered(c.id),
                               bubbles: store.bubblesOn) { tap(c) }
            }
            .swimming(seed: c.id.hashValue, enabled: store.bubblesOn)
            .frame(width: width * 0.30)
            .position(x: idx.isMultiple(of: 2) ? laneA : laneB, y: depthY(c.appear))
```

Then update `tap(_:)` to fire the burst:

```swift
    private func tap(_ c: Creature) {
        withAnimation(.spring(response: 0.4, dampingFraction: 0.7)) {
            if store.discover(c.id) { burst = true }
        }
        selected = c
    }
```

And add the burst overlay next to the sonar overlays:

```swift
            .overlay {
                DiscoveryBurst(color: Sea.glow, trigger: $burst)
                    .frame(width: 240, height: 240)
            }
```

- [ ] **Step 5: Register `SonarLayer.swift` in `project.pbxproj`**

Ids `…0037`/`…0137`, all four sections.

- [ ] **Step 6: Compile**

Run:
```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue" && xcodebuild -project "Deep Blue.xcodeproj" -scheme "Deep Blue" -destination 'platform=iOS Simulator,name=iPhone 17' -derivedDataPath build/ build > /tmp/t9.log 2>&1; echo "EXIT $?"; grep -E "BUILD SUCCEEDED|BUILD FAILED" /tmp/t9.log | tail -1
```
Expected: `BUILD SUCCEEDED`

- [ ] **Step 7: Verify the hub and the descent on the simulator**

```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
xcrun simctl terminate booted com.deepbluedive.app 2>/dev/null
xcrun simctl install booted "build/Build/Products/Debug-iphonesimulator/Deep Blue.app"
xcrun simctl launch booted com.deepbluedive.app
sleep 9; xcrun simctl io booted screenshot /tmp/t9hub.png
```
Open `/tmp/t9hub.png`. Expected: the Expedition card and the **Begin Descent** button sit under the HUD, nothing overlaps, the vessel silhouette renders.

- [ ] **Step 8: Commit**

```bash
git add "Deep Blue/SonarLayer.swift" "Deep Blue/DiveView.swift" "Deep Blue.xcodeproj/project.pbxproj"
git commit -m "Add sonar ping, expedition card and Begin Descent hub"
```

---

## Task 10: Rank emblems + Journal rank card + size-vs-you + profile UI

**Files:**
- Create: `Deep Blue/RankEmblem.swift`
- Modify: `Deep Blue/JournalView.swift`, `Deep Blue/CreatureDetailView.swift`, `Deep Blue/MoreView.swift`
- Modify: `Deep Blue.xcodeproj/project.pbxproj`

**Interfaces:**
- Consumes: `Ranks`, `DiverRank`, `BodyScale`, `DeepStore` profile/rank API.
- Produces: `struct RankEmblem: View`, `struct RankHeroCard: View`.

- [ ] **Step 1: Create `Deep Blue/RankEmblem.swift`**

```swift
import SwiftUI

// Canvas-drawn rank badges — one silhouette per rank, no PNGs.

struct RankEmblem: View {
    let rank: Int          // 1...8
    var size: CGFloat = 62
    var color: Color = Sea.aqua

    var body: some View {
        ZStack {
            Circle().fill(color.opacity(0.16))
            Circle().stroke(color.opacity(0.75), lineWidth: size * 0.045)
            Canvas { ctx, s in
                let w = s.width, h = s.height
                let c = CGPoint(x: w / 2, y: h / 2)
                let lw = size * 0.05

                // Rings grow with rank.
                let rings = min(3, max(0, (rank - 1) / 3))
                for i in 0..<rings {
                    let r = w * (0.30 + Double(i) * 0.07)
                    ctx.stroke(Path(ellipseIn: CGRect(x: c.x - r, y: c.y - r, width: r * 2, height: r * 2)),
                               with: .color(color.opacity(0.35)), lineWidth: lw * 0.7)
                }

                // Chevrons: one per rank, up to four, stacked.
                let chev = min(4, (rank + 1) / 2)
                for i in 0..<chev {
                    let y = c.y - CGFloat(chev - 1) * h * 0.055 + CGFloat(i) * h * 0.11
                    var p = Path()
                    p.move(to: CGPoint(x: c.x - w * 0.17, y: y - h * 0.035))
                    p.addLine(to: CGPoint(x: c.x, y: y + h * 0.045))
                    p.addLine(to: CGPoint(x: c.x + w * 0.17, y: y - h * 0.035))
                    ctx.stroke(p, with: .color(color),
                               style: .init(lineWidth: lw, lineCap: .round, lineJoin: .round))
                }

                // The top ranks get a crowning dot.
                if rank >= 7 {
                    let r = w * 0.05
                    ctx.fill(Path(ellipseIn: CGRect(x: c.x - r, y: h * 0.16 - r, width: r * 2, height: r * 2)),
                             with: .color(color))
                }
            }
            .frame(width: size, height: size)
        }
        .frame(width: size, height: size)
    }
}

struct RankHeroCard: View {
    @EnvironmentObject var store: DeepStore
    @State private var shownXP: Int = 0

    var body: some View {
        GlassCard(tint: Sea.card) {
            HStack(spacing: 14) {
                RankEmblem(rank: store.rank.number, size: 64, color: Sea.aqua)
                VStack(alignment: .leading, spacing: 4) {
                    Text(store.rank.name).font(.deepTitle(19)).foregroundColor(Sea.ink)
                    Text("\(shownXP) XP").font(.deepMono(13, .bold)).foregroundColor(Sea.aqua)
                    if let next = store.nextRank {
                        GeometryReader { g in
                            ZStack(alignment: .leading) {
                                Capsule().fill(Sea.stroke.opacity(0.3)).frame(height: 5)
                                Capsule().fill(Sea.aqua)
                                    .frame(width: g.size.width * CGFloat(store.rankProgress), height: 5)
                            }
                        }
                        .frame(height: 5)
                        Text("\(next.xp - store.xp) XP to \(next.name)")
                            .font(.deepBody(11)).foregroundColor(Sea.inkFaint)
                    } else {
                        Text("Highest rank reached.")
                            .font(.deepBody(11, .semibold)).foregroundColor(Sea.gold)
                    }
                }
                Spacer(minLength: 0)
            }
        }
        .onAppear {
            shownXP = 0
            withAnimation(.easeOut(duration: 0.9)) { shownXP = store.xp }
        }
    }
}
```

Note: `withAnimation` cannot tween an `Int` label directly; the count-up is a
deliberate one-shot assignment so the number appears with the card rather than
ticking. Do not try to animate it per-digit — that needs iOS 16 APIs.

- [ ] **Step 2: Put the rank card at the top of the Journal**

In `Deep Blue/JournalView.swift`, inside the `VStack` in the `ScrollView`, insert `RankHeroCard()` immediately after `BannerHeader(...)` and before `overviewCard`:

```swift
                    RankHeroCard()
```

Then add an expedition-history line inside `overviewCard`'s inner `VStack`, after the "Deepest dive" text:

```swift
                    Text("\(store.expeditionHistory.count) expeditions completed")
                        .font(.deepBody(12, .semibold)).foregroundColor(Sea.gold)
```

- [ ] **Step 3: Add the size-vs-you row to the creature detail**

In `Deep Blue/CreatureDetailView.swift`, inside `depthCard`'s `VStack`, after the `Text("Diet: …")` line, add:

```swift
                Divider().background(Sea.stroke.opacity(0.3))
                if store.hasProfile {
                    VStack(alignment: .leading, spacing: 7) {
                        Text(BodyScale.sizePhrase(creature: creature, heightCM: store.profileHeightCM))
                            .font(.deepBody(13, .bold)).foregroundColor(Sea.glow)
                        GeometryReader { g in
                            let ratio = BodyScale.sizeRatio(creature: creature, heightCM: store.profileHeightCM)
                            let youW = ratio >= 1 ? g.size.width / CGFloat(max(1, ratio)) : g.size.width
                            let creW = ratio >= 1 ? g.size.width : g.size.width * CGFloat(ratio)
                            VStack(alignment: .leading, spacing: 5) {
                                HStack(spacing: 6) {
                                    Capsule().fill(Sea.inkFaint).frame(width: max(6, youW), height: 8)
                                    Text("you").font(.deepBody(10, .semibold)).foregroundColor(Sea.inkFaint)
                                }
                                HStack(spacing: 6) {
                                    Capsule().fill(creature.accent).frame(width: max(6, creW), height: 8)
                                    Text(creature.name).font(.deepBody(10, .semibold))
                                        .foregroundColor(creature.accent).lineLimit(1)
                                }
                            }
                        }
                        .frame(height: 34)
                    }
                } else {
                    Text("Add your height in More to compare yourself with this animal.")
                        .font(.deepBody(12)).foregroundColor(Sea.inkFaint)
                        .fixedSize(horizontal: false, vertical: true)
                }
```

- [ ] **Step 4: Add the profile and vessel sections to More**

In `Deep Blue/MoreView.swift`, add `@State private var showVessels = false` next to the existing states, then insert two sections into the main `VStack`, between `awardsSection` and `settingsSection`:

```swift
                    profileSection
                    vesselSection
```

Add these computed properties to `MoreView`:

```swift
    private var profileSection: some View {
        VStack(alignment: .leading, spacing: 12) {
            SectionHeader(title: "About you",
                          caption: "Optional — makes depth and size personal")
            GlassCard {
                VStack(alignment: .leading, spacing: 14) {
                    VStack(alignment: .leading, spacing: 6) {
                        HStack {
                            Text("Height").font(.deepBody(14, .semibold)).foregroundColor(Sea.ink)
                            Spacer()
                            Text(store.profileHeightCM > 0
                                 ? String(format: "%.0f cm", store.profileHeightCM)
                                 : "not set")
                                .font(.deepMono(13, .bold)).foregroundColor(Sea.aqua)
                        }
                        Slider(value: $store.profileHeightCM, in: 0...220, step: 1)
                            .accentColor(Sea.aqua)
                    }
                    VStack(alignment: .leading, spacing: 6) {
                        HStack {
                            Text("Weight").font(.deepBody(14, .semibold)).foregroundColor(Sea.ink)
                            Spacer()
                            Text(store.profileWeightKG > 0
                                 ? String(format: "%.0f kg", store.profileWeightKG)
                                 : "not set")
                                .font(.deepMono(13, .bold)).foregroundColor(Sea.aqua)
                        }
                        Slider(value: $store.profileWeightKG, in: 0...200, step: 1)
                            .accentColor(Sea.aqua)
                    }
                    if store.hasProfile {
                        Text(BodyScale.pressurePhrase(depth: 1_000,
                                                      heightCM: store.profileHeightCM,
                                                      weightKG: store.profileWeightKG)
                             + " — at 1,000 m.")
                            .font(.deepBody(12)).foregroundColor(Sea.glow)
                            .fixedSize(horizontal: false, vertical: true)
                        Button { store.clearProfile() } label: {
                            Text("Clear").font(.deepBody(13, .semibold)).foregroundColor(Sea.coral)
                        }
                    }
                    Text("Stored only on this device.")
                        .font(.deepBody(11)).foregroundColor(Sea.inkFaint)
                }
            }
        }
    }

    private var vesselSection: some View {
        VStack(alignment: .leading, spacing: 12) {
            SectionHeader(title: "Your craft",
                          caption: "\(Ranks.unlockedVessels(xp: store.xp).count) of \(Vessels.all.count) unlocked")
            Button { showVessels = true } label: {
                GlassCard(tint: Sea.panel) {
                    HStack(spacing: 12) {
                        ArtImage(name: store.vessel.art, contentMode: .fit, fallback: .clear)
                            .frame(width: 58, height: 44)
                        VStack(alignment: .leading, spacing: 2) {
                            Text(store.vessel.name)
                                .font(.deepBody(15, .bold)).foregroundColor(Sea.ink)
                            Text("Reaches \(store.vessel.limitLabel)")
                                .font(.deepBody(12)).foregroundColor(Sea.inkSoft)
                        }
                        Spacer()
                        ChevronGlyph(pointing: .trailing, size: 15, color: Sea.inkFaint)
                    }
                }
            }
            .buttonStyle(PressStyle())
        }
    }
```

And add the sheet to `MoreView.body`, next to the existing `.sheet(isPresented: $showPrivacy)`:

```swift
        .sheet(isPresented: $showVessels) {
            VesselPickerView().environmentObject(store)
        }
```

- [ ] **Step 5: Register `RankEmblem.swift` in `project.pbxproj`**

Ids `…0038`/`…0138`, all four sections.

- [ ] **Step 6: Compile**

Run:
```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue" && xcodebuild -project "Deep Blue.xcodeproj" -scheme "Deep Blue" -destination 'platform=iOS Simulator,name=iPhone 17' -derivedDataPath build/ build > /tmp/t10.log 2>&1; echo "EXIT $?"; grep -E "BUILD SUCCEEDED|BUILD FAILED" /tmp/t10.log | tail -1
```
Expected: `BUILD SUCCEEDED`

- [ ] **Step 7: Commit**

```bash
git add "Deep Blue/RankEmblem.swift" "Deep Blue/JournalView.swift" "Deep Blue/CreatureDetailView.swift" "Deep Blue/MoreView.swift" "Deep Blue.xcodeproj/project.pbxproj"
git commit -m "Add rank emblems, rank hero card, size comparison and profile UI"
```

---

## Task 11: Biome cards in Zones

**Files:**
- Modify: `Deep Blue/ZonesView.swift`

**Interfaces:**
- Consumes: `Ocean.biomes`, `Biome`, `Adaptive`.
- Produces: `struct BiomesView: View` (a pushed screen), plus a link row on the Zones tab.

- [ ] **Step 1: Add the biomes screen and its link**

In `Deep Blue/ZonesView.swift`, add a link row inside the main `VStack`, immediately after the existing "Marks Along the Dive" `NavigationLink`:

```swift
                    NavigationLink(destination: BiomesView()) {
                        linkRow(title: "Places in the Deep",
                                subtitle: "\(Ocean.biomes.count) worlds within the ocean",
                                accent: Sea.coral)
                    }.buttonStyle(PressStyle())
```

Then append this view at the end of the file:

```swift
// MARK: - Biomes screen

struct BiomesView: View {
    var body: some View {
        ZStack {
            OceanBackdrop(colors: [Sea.deep, Sea.abyss, Sea.trench])
            ScrollView(showsIndicators: false) {
                VStack(spacing: 14) {
                    Text("The ocean is not one place. These are worlds inside it.")
                        .font(.deepBody(13)).foregroundColor(Sea.inkSoft)
                        .multilineTextAlignment(.center)
                        .padding(.horizontal, 20).padding(.top, 8)

                    ForEach(Ocean.biomes) { b in
                        VStack(alignment: .leading, spacing: 0) {
                            ZStack(alignment: .bottomLeading) {
                                ArtImage(name: b.art, contentMode: .fill, fallback: Sea.panel)
                                    .frame(height: 170)
                                    .frame(maxWidth: .infinity)
                                    .clipped()
                                LinearGradient(colors: [.clear, Sea.trench.opacity(0.85)],
                                               startPoint: .center, endPoint: .bottom)
                                VStack(alignment: .leading, spacing: 3) {
                                    Text(b.name).font(.deepTitle(20)).foregroundColor(.white)
                                    Pill(text: "≈ \(b.depth.formattedDepth) m", color: b.accent)
                                }
                                .padding(14)
                            }
                            .frame(height: 170)

                            Text(b.blurb)
                                .font(.deepBody(13)).foregroundColor(Sea.inkSoft)
                                .fixedSize(horizontal: false, vertical: true)
                                .lineSpacing(2)
                                .padding(14)
                        }
                        .background(
                            RoundedRectangle(cornerRadius: 20, style: .continuous)
                                .fill(Sea.card.opacity(0.7))
                        )
                        .clipShape(RoundedRectangle(cornerRadius: 20, style: .continuous))
                        .overlay(RoundedRectangle(cornerRadius: 20, style: .continuous)
                            .stroke(Sea.stroke.opacity(0.3), lineWidth: 1))
                    }
                    Color.clear.frame(height: 90)
                }
                .adaptiveColumn()
                .padding(.horizontal, 16)
            }
        }
        .navigationTitle("Places in the Deep")
        .navigationBarTitleDisplayMode(.inline)
    }
}
```

- [ ] **Step 2: Compile**

Run:
```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue" && xcodebuild -project "Deep Blue.xcodeproj" -scheme "Deep Blue" -destination 'platform=iOS Simulator,name=iPhone 17' -derivedDataPath build/ build > /tmp/t11.log 2>&1; echo "EXIT $?"; grep -E "BUILD SUCCEEDED|BUILD FAILED" /tmp/t11.log | tail -1
```
Expected: `BUILD SUCCEEDED`

- [ ] **Step 3: Commit**

```bash
git add "Deep Blue/ZonesView.swift"
git commit -m "Add Places in the Deep biome screen"
```

---

## Task 12: Onboarding refresh + full verification pass

**Files:**
- Modify: `Deep Blue/OnboardingView.swift`

- [ ] **Step 1: Update the onboarding copy for v2**

In `Deep Blue/OnboardingView.swift`, replace the `pages` array with:

```swift
    private let pages: [Page] = [
        Page(title: "Welcome to Deep Blue",
             body: "Descend from the sunlit surface to the floor of the deepest ocean trench — over 10 kilometres straight down, on one continuous scale."),
        Page(title: "Pilot Your Craft",
             body: "Begin a descent and your vessel sinks for you. Below 1,000 m only your lamp is left, and creatures drift into the beam out of the dark."),
        Page(title: "Earn the Depths",
             body: "Identify 50 real animals at their true depths, complete a new expedition each day, and rise through eight diver ranks — each one unlocking a craft that reaches further down."),
    ]
```

- [ ] **Step 2: Full clean build for iPhone**

```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
xcodebuild -project "Deep Blue.xcodeproj" -scheme "Deep Blue" -destination 'platform=iOS Simulator,name=iPhone 17' -derivedDataPath build/ clean build > /tmp/final_iphone.log 2>&1; echo "EXIT $?"
grep -E "BUILD SUCCEEDED|BUILD FAILED" /tmp/final_iphone.log | tail -1
grep -E "\.swift.*(error:|warning:)" /tmp/final_iphone.log | sort -u
```
Expected: `BUILD SUCCEEDED` and **no** lines from the second grep.

- [ ] **Step 3: Full clean build for iPad**

```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
xcodebuild -project "Deep Blue.xcodeproj" -scheme "Deep Blue" -destination 'platform=iOS Simulator,name=<exact iPad name>' -derivedDataPath build-ipad/ clean build > /tmp/final_ipad.log 2>&1; echo "EXIT $?"
grep -E "BUILD SUCCEEDED|BUILD FAILED" /tmp/final_ipad.log | tail -1
```
Expected: `BUILD SUCCEEDED`

- [ ] **Step 4: Screenshot every screen on iPhone**

Boot, install, seed a rich state, then capture. Seed:
```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
xcrun simctl terminate booted com.deepbluedive.app 2>/dev/null
xcrun simctl install booted "build/Build/Products/Debug-iphonesimulator/Deep Blue.app"
CONT=$(xcrun simctl get_app_container booted com.deepbluedive.app data)
python3 - "$CONT" <<'PY'
import sys, os, json, plistlib
prefs = os.path.join(sys.argv[1], "Library", "Preferences"); os.makedirs(prefs, exist_ok=True)
state = {"discovered": ["moon_jelly","great_white","dolphin","lanternfish","anglerfish",
                        "gulper_eel","dumbo_octopus","giant_isopod","vampire_squid","sunfish",
                        "tube_worm","snailfish","whale_shark","orca","nautilus","oarfish"],
         "deepest": 4200, "onboarding": True, "bubbles": True, "haptics": True, "latin": True,
         "xp": 320, "vessel": "submersible", "zones": [0,1,2,3],
         "expDoneDay": -1, "expHistory": [20990, 20991, 20992],
         "heightCM": 178.0, "weightKG": 76.0, "metric": True}
plistlib.dump({"deepblue.state.v1": json.dumps(state).encode()},
              open(os.path.join(prefs, "com.deepbluedive.app.plist"), "wb"))
print("seeded")
PY
xcrun simctl launch booted com.deepbluedive.app
sleep 9; xcrun simctl io booted screenshot screenshots/02_dive.png
```
Capture the remaining screens by adding the temporary env-var hook from the prior session's workflow to `RootView` (`DB_TAB`, `DB_DETAIL`), rebuilding once, and launching with `SIMCTL_CHILD_DB_TAB=1|2|3`. **Record here which hook you added so Step 6 can remove it.**

Screens to capture: `01_onboarding`, `02_dive` (hub), `03_descent` (mid-water), `04_descent_dark` (below 1,000 m, lamp visible), `05_vessels`, `06_journal` (rank card), `07_zones`, `08_biomes`, `09_more` (profile), `10_detail` (size comparison).

Open each PNG and confirm: no overlapping elements, no clipped text, art present, no emoji.

- [ ] **Step 5: Screenshot on iPad**

```bash
IPAD=$(xcrun simctl list devices available | grep -i "iPad" | head -1 | sed -E 's/.*\(([-A-F0-9]+)\).*/\1/')
xcrun simctl boot "$IPAD" 2>/dev/null
xcrun simctl install "$IPAD" "build-ipad/Build/Products/Debug-iphonesimulator/Deep Blue.app"
xcrun simctl launch "$IPAD" com.deepbluedive.app
sleep 10; xcrun simctl io "$IPAD" screenshot screenshots/11_ipad.png
```
Open it. Expected: content is centred and capped (not stretched edge to edge), the tab bar spans the bottom, grids show more columns, nothing overlaps.

- [ ] **Step 6: Remove every temporary hook and rebuild clean**

```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
grep -rn "DB_TAB\|DB_DETAIL\|debugDetail\|DEBUG hook" "Deep Blue"/*.swift || echo "CLEAN - no debug hooks"
```
Expected: `CLEAN - no debug hooks`. If anything prints, remove it, then re-run Step 2's clean build and confirm `BUILD SUCCEEDED` with no code warnings.

- [ ] **Step 7: Final artefact checks**

```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
APP="build/Build/Products/Debug-iphonesimulator/Deep Blue.app"
du -sh "$APP"
sips -g hasAlpha "Deep Blue/Assets.xcassets/AppIcon.appiconset/AppIcon-1024.png" | tail -1
/usr/libexec/PlistBuddy -c 'Print :CFBundleIcons:CFBundlePrimaryIcon:CFBundleIconFiles' "$APP/Info.plist"
/usr/libexec/PlistBuddy -c 'Print :CFBundleShortVersionString' "$APP/Info.plist"
/usr/libexec/PlistBuddy -c 'Print :CFBundleVersion' "$APP/Info.plist"
grep -c 'TARGETED_DEVICE_FAMILY = "1,2"' "Deep Blue.xcodeproj/project.pbxproj"
```
Expected: app between 18 MB and 99 MB (~50–60 MB), `hasAlpha: no`, `AppIcon60x60` listed, version `1.0`, build `1`, and `2` for the device-family count.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "Refresh onboarding for v2 and add verification screenshots"
```

---

## Task 13: Delivery — trackers, cleanup, public GitHub repo

**Files:**
- Modify: `APP_TRACKER.md`, `APP_DESCRIPTIONS.md` (in the workspace root, **not** the app folder)

- [ ] **Step 1: Update the app description**

In `/Volumes/ADATA SE880/работа/APP_DESCRIPTIONS.md`, **replace** the existing `| Deep Blue | … |` row with one that describes v2: the vessel-gated descent, lamp-lit darkness, daily expedition, 8 ranks, 50 creatures, 6 biomes, body-scale personalization, iPad support. One row, one line, same table format as the neighbouring rows.

- [ ] **Step 2: Update the tracker**

In `/Volumes/ADATA SE880/работа/development/APP_TRACKER.md`, **append** a new row (do not delete the v1 row) in the established style:

`| Deep Blue (v2 Expedition) | Education / Reference | Delivered → for_human_review_apps | com.deepbluedive.app | v1.0 (build 1) — EXPEDITION EDITION … |`

The notes must record: the five vessels and their limits, rank/XP table, expedition kinds, lamp-cone darkness ramp (400→1,000 m), sonar window (±1,500 m), 50 creatures / 18 awards / 6 biomes, new files, store fields with tolerant decode, `TARGETED_DEVICE_FAMILY = "1,2"`, final app size, and that hooks were removed before the final clean build.

- [ ] **Step 3: Remove build artefacts before publishing**

```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
rm -rf build build-ipad
ls -a
```
Expected: only `.git`, `.gitignore`, `Deep Blue`, `Deep Blue.xcodeproj`, `art_src`, `screenshots` (plus macOS `._*` files). **No `build/` directories** — deleting them on the external drive is slow, so run this in the background and wait.

- [ ] **Step 4: Commit the app repo**

```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
git add -A
git commit -m "Deep Blue v2 Expedition Edition"
git --no-pager log --oneline -1
```

- [ ] **Step 5: Show the user what is about to be published, and wait**

Publishing is outward-facing and irreversible. Before creating anything, print exactly what will be pushed and confirm the repo name with the user:

```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
echo "Repo name: deep-blue-ios"
echo "Visibility: PUBLIC"
git --no-pager log --oneline | head -20
echo "--- files that would be pushed ---"
git ls-files | sed 's|/.*||' | sort -u
du -sh .
```
State the repo name and visibility to the user and get their go-ahead before Step 6.

- [ ] **Step 6: Create the public repo and push**

Per the workspace convention, use the API token — **not** the `gh` CLI:

```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
source ~/.config/github-token
OWNER=$(curl -s -H "Authorization: token $GITHUB_TOKEN" https://api.github.com/user | python3 -c "import sys,json;print(json.load(sys.stdin)['login'])")
echo "owner: $OWNER"
curl -s -X POST -H "Authorization: token $GITHUB_TOKEN" \
     -H "Accept: application/vnd.github+json" \
     https://api.github.com/user/repos \
     -d '{"name":"deep-blue-ios","description":"Deep Blue — a continuous-scale ocean dive for iOS: pilot a vessel from the surface to the Mariana Trench and meet 50 real deep-sea creatures at their true depths.","private":false,"has_issues":true,"has_wiki":false}' \
     | python3 -c "import sys,json;d=json.load(sys.stdin);print(d.get('html_url') or d)"

git branch -M main
git remote add origin "https://$GITHUB_TOKEN@github.com/$OWNER/deep-blue-ios.git"
git push -u origin main
```

- [ ] **Step 7: Verify the published repo**

```bash
cd "/Volumes/ADATA SE880/работа/for_human_review_apps/Deep Blue"
source ~/.config/github-token
OWNER=$(curl -s -H "Authorization: token $GITHUB_TOKEN" https://api.github.com/user | python3 -c "import sys,json;print(json.load(sys.stdin)['login'])")
curl -s -H "Authorization: token $GITHUB_TOKEN" "https://api.github.com/repos/$OWNER/deep-blue-ios" \
  | python3 -c "import sys,json;d=json.load(sys.stdin);print('url:',d['html_url']);print('public:',not d['private']);print('size KB:',d['size'])"
git remote -v
```
Expected: the URL prints, `public: True`, and a non-zero size.

**Then scrub the token out of the git remote** so it is not left in `.git/config`:
```bash
git remote set-url origin "https://github.com/$OWNER/deep-blue-ios.git"
grep -c "$GITHUB_TOKEN" .git/config || echo "token not stored in .git/config"
```
Expected: `token not stored in .git/config`

- [ ] **Step 8: Commit the workspace tracker changes**

```bash
cd "/Volumes/ADATA SE880/работа"
git add APP_DESCRIPTIONS.md development/APP_TRACKER.md
git commit -m "Update tracker and description for Deep Blue v2 Expedition Edition"
```

---

## Self-Review

**Spec coverage** — every spec section maps to a task:

| Spec section | Task |
|---|---|
| §1.1 Vessels | 1 (data), 7 (picker UI) |
| §1.2 Descent, lamp cone, hull limit | 6 (lamp), 7 (view) |
| §1.3 Sonar ping | 9 |
| §1.4 Daily Expedition | 2 (engine), 3 (state), 9 (card) |
| §1.5 Dive tab hub | 9 |
| §2 Ranks + XP, 18 awards | 2, 3, 5, 10 |
| §3 "You at depth" | 2 (math), 10 (HUD line, size row, profile UI) |
| §4 Living ocean | 6, 9 |
| §5.1 +14 creatures | 4 (art), 5 (data) |
| §5.2 Biome scenes, vessel art, rank emblems | 4, 10, 11 |
| §6 iPad support | 8 |
| §7 Architecture / new files | 1, 2, 7, 8, 9, 10 |
| §8 Verification | 3 (v1-save compat), 12 |
| §9 Delivery + public repo | 13 |

**Placeholder scan:** no TBD/TODO; every code step carries complete code; the one deliberate `<exact iPad name>` substitution is paired with the command that discovers it.

**Type consistency checked:**
- `Vessels.byID`, `Vessels.all`, `Vessels.defaultID` — defined Task 1, used Tasks 3, 7, 10.
- `Ranks.rank(forXP:)`, `Ranks.next(afterXP:)`, `Ranks.progress(xp:)`, `Ranks.unlockedVessels(xp:)`, `Ranks.isVesselUnlocked(_:xp:)` — defined Task 2, used Tasks 3, 5, 7, 10.
- `XPAward.*` — defined Task 2, used Tasks 3, 9.
- `ExpeditionEngine.today(dayIndex:)` / `.dayIndex(for:)` — defined Task 2, used Task 3.
- `BodyScale.pressurePhrase/sizeRatio/sizePhrase` — defined Task 2, used Tasks 7, 10.
- `store.xp/rank/nextRank/rankProgress/vessel/isVesselUnlocked/hasProfile/expeditionHistory/zonesVisited/expeditionFraction/expeditionComplete/todayExpedition/refreshExpedition` — defined Task 3, used Tasks 5, 7, 9, 10.
- `lampDarkness(atDepth:)`, `LampCone`, `swimming(seed:enabled:)`, `DiscoveryBurst`, `ParallaxDrift` — defined Task 6, used Tasks 7, 9.
- `Adaptive.contentWidth/grid/columns`, `adaptiveColumn()` — defined Task 8; **used in Task 7, which runs earlier** — Task 7 Step 4 flags this explicitly and tells the implementer to pull Task 8 forward if the compiler complains.
- `Creature.lengthMeters` — added Task 1, populated Tasks 1 & 5, consumed Task 2/10.
- `waterColor(_:)` is already non-private in `DiveView.swift` — Task 7 relies on that; if it were made private the descent would not compile.

**Fixed inline during review:** `spider_crab` was originally listed in the abyssal group with an `appear` deeper than its real range; Task 5 now instructs zone 1 / `appear: 560` so the app never states a false habitat.
