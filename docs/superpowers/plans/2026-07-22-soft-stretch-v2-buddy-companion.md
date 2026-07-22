# Soft Stretch v2 — Buddy Companion Edition Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn Soft Stretch into a companion app: Buddy gets moods, a living nook on Home, friendship levels with unlockable skins/accessories, 7-day programs, a custom routine builder, a calendar heatmap + body balance, and a full style pass.

**Architecture:** New `CompanionStore` (second environment object, new UserDefaults key — zero v1 migration risk) owns xp/levels/outfit/program progress. `BuddyCanvas` gains `outfit`/`mood` parameters with all-default call-site compatibility; accessories are pure Canvas helpers positioned from the existing `Joints` rig. Home hero is replaced by a single-Canvas living nook. Programs reference existing routines; custom routines map onto the existing `Routine` struct so `PlayerView` plays them unchanged.

**Tech Stack:** SwiftUI (iOS 15.6), Canvas/TimelineView, UserDefaults JSON, swiftc scratchpad harness for pure logic.

## Global Constraints

- iOS 15.6+ — `NavigationView` only, no iOS-16 API (no NavigationStack/Charts/.scrollContentBackground…).
- Custom UI only: no SF Symbols, no emoji anywhere; glyphs via `SoftIcon`/Canvas paths.
- Forced light (`UIUserInterfaceStyle = Light` + `.preferredColorScheme(.light)`); theme-independent.
- Version stays **1.0 (build 1)** — do not bump.
- English (US) copy only. Local storage only. No notifications/network (WebView gate stays as-is).
- App folder: `/Volumes/ADATA SE880/работа/development/Soft Stretch` (project `Soft Stretch.xcodeproj`, sources in `Soft Stretch/`).
- Build check after every task: `cd "/Volumes/ADATA SE880/работа/development/Soft Stretch" && xcodebuild -project "Soft Stretch.xcodeproj" -scheme "Soft Stretch" -destination 'generic/platform=iOS Simulator' -derivedDataPath build/ build 2>&1 | grep -E "error:|BUILD"` → `** BUILD SUCCEEDED **`.
- v1 saves must keep loading: existing keys `soft.sessions.v1` / `soft.settings.v1` / `soft.badges.v1` untouched; `SessionRecord.areaSeconds` optional; all new state under new keys.
- New .swift files must be added to the Xcode target: the project uses `PBXFileSystemSynchronizedRootGroup`? — **No**: it is a classic pbxproj; add each new file to `project.pbxproj` (PBXBuildFile + PBXFileReference + group + Sources phase) exactly the way existing files are listed, or regenerate membership by following an existing file's four entries.
- Commit after every task in the app repo (`git -c user.name="Grigorii" -c user.email="grigoriikorniychuk@gmail.com" commit`).
- Scratchpad for harness tests: `/private/tmp/claude-501/-Volumes-ADATA-SE880-------/3e5be0ad-48f1-4c58-96e7-bbc1775f867c/scratchpad`.

---

### Task 1: CompanionStore — xp, levels, rewards, program progress

**Files:**
- Create: `Soft Stretch/CompanionStore.swift`
- Modify: `Soft Stretch/SoftStretchApp.swift` (inject environment object)
- Modify: `Soft Stretch.xcodeproj/project.pbxproj` (register new file)
- Test: swiftc harness (Step 3)

**Interfaces:**
- Produces:
  - `enum Friendship { static let levelThresholds: [Int]; static func level(for xp: Int) -> Int; static func progressToNext(_ xp: Int) -> (have: Int, need: Int, fraction: CGFloat); static func levelName(_ level: Int) -> String }`
  - `enum RewardKind { case skin(String), accessory(String) }`
  - `struct FriendshipReward { let level: Int; let kind: RewardKind; let title: String }`
  - `enum RewardTable { static let all: [FriendshipReward]; static func reward(at level: Int) -> FriendshipReward?; static func skinUnlockLevel(_ id: String) -> Int; static func accessoryUnlockLevel(_ id: String) -> Int }`
  - `struct CompanionState: Codable { var xp: Int; var equippedSkin: String; var equippedAccessory: String?; var celebratedLevels: [Int]; var programDays: [String: [Int: Date]] }` (all defaulted via custom `init(from:)` tolerant decoding)
  - `final class CompanionStore: ObservableObject { @Published var state: CompanionState; @Published var lastReward: SessionReward?; var level: Int; var xp: Int; func award(seconds: Int, streakActive: Bool); func isSkinUnlocked(_ id: String) -> Bool; func isAccessoryUnlocked(_ id: String) -> Bool; func equipSkin(_ id: String); func equipAccessory(_ id: String?); func markCelebrated(_ level: Int); func completeProgramDay(programID: String, day: Int); func completedDays(_ programID: String) -> Set<Int>; func nextDay(_ programID: String, totalDays: Int) -> Int?; var outfit: BuddyOutfit }`
  - `struct SessionReward { let gained: Int; let oldLevel: Int; let newLevel: Int }`
- Consumes: `BuddyOutfit` arrives in Task 2 — in THIS task declare `outfit` as computed returning `BuddyOutfit(skinID: state.equippedSkin, accessoryID: state.equippedAccessory)` and add a minimal placeholder `struct BuddyOutfit { var skinID: String = "mint"; var accessoryID: String? = nil }` at the top of `CompanionStore.swift` with a `// moved to BuddyOutfits.swift in Task 2? NO — it lives here permanently` comment; Task 2 imports it from here.

- [ ] **Step 1: Write `Soft Stretch/CompanionStore.swift`**

```swift
import SwiftUI

// Buddy's friendship: xp, levels, unlockable looks and program progress.
// Stored under its own key so v1 saves are never touched.

struct BuddyOutfit: Equatable {
    var skinID: String = "mint"
    var accessoryID: String? = nil
    static let classic = BuddyOutfit()
}

enum Friendship {
    // Cumulative xp needed for levels 1...12.
    static let levelThresholds = [0, 10, 25, 45, 70, 100, 140, 190, 250, 320, 400, 500]

    static func level(for xp: Int) -> Int {
        var lvl = 1
        for (i, t) in levelThresholds.enumerated() where xp >= t { lvl = i + 1 }
        return lvl
    }

    static func progressToNext(_ xp: Int) -> (have: Int, need: Int, fraction: CGFloat) {
        let lvl = level(for: xp)
        guard lvl < levelThresholds.count else { return (0, 0, 1) }
        let base = levelThresholds[lvl - 1]
        let next = levelThresholds[lvl]
        let have = xp - base, need = next - base
        return (have, need, CGFloat(have) / CGFloat(max(need, 1)))
    }

    static let names = ["New Friends", "Hello Pals", "Warm Pals", "Good Company",
                        "Cozy Pals", "Stretch Mates", "True Buddies", "Close Friends",
                        "Dear Friends", "Soul Stretchers", "Heart Friends", "Best Friends"]

    static func levelName(_ level: Int) -> String {
        names[min(max(level, 1), names.count) - 1]
    }
}

enum RewardKind {
    case skin(String)
    case accessory(String)
}

struct FriendshipReward {
    let level: Int
    let kind: RewardKind
    let title: String       // "Peach skin", "Cozy scarf"...
}

enum RewardTable {
    static let all: [FriendshipReward] = [
        FriendshipReward(level: 2, kind: .skin("peach"), title: "Peach skin"),
        FriendshipReward(level: 3, kind: .accessory("sweatband"), title: "Sporty sweatband"),
        FriendshipReward(level: 4, kind: .skin("sky"), title: "Sky skin"),
        FriendshipReward(level: 5, kind: .accessory("scarf"), title: "Cozy scarf"),
        FriendshipReward(level: 6, kind: .accessory("flowers"), title: "Flower crown"),
        FriendshipReward(level: 7, kind: .skin("lilac"), title: "Lilac skin"),
        FriendshipReward(level: 8, kind: .accessory("glasses"), title: "Round glasses"),
        FriendshipReward(level: 9, kind: .skin("sand"), title: "Sand skin"),
        FriendshipReward(level: 10, kind: .accessory("beanie"), title: "Pompom beanie"),
        FriendshipReward(level: 11, kind: .accessory("bowtie"), title: "Bow tie"),
        FriendshipReward(level: 12, kind: .skin("sunrise"), title: "Sunrise gold skin")
    ]

    static func reward(at level: Int) -> FriendshipReward? {
        all.first { $0.level == level }
    }

    static func skinUnlockLevel(_ id: String) -> Int {
        if id == "mint" { return 1 }
        for r in all { if case .skin(let s) = r.kind, s == id { return r.level } }
        return 1
    }

    static func accessoryUnlockLevel(_ id: String) -> Int {
        for r in all { if case .accessory(let a) = r.kind, a == id { return r.level } }
        return 1
    }
}

struct SessionReward {
    let gained: Int
    let oldLevel: Int
    let newLevel: Int
}

struct CompanionState: Codable {
    var xp: Int = 0
    var equippedSkin: String = "mint"
    var equippedAccessory: String? = nil
    var celebratedLevels: [Int] = []
    var programDays: [String: [Int: Date]] = [:]

    init() {}

    // Tolerant decoding: any missing key falls back to its default.
    enum CodingKeys: String, CodingKey {
        case xp, equippedSkin, equippedAccessory, celebratedLevels, programDays
    }

    init(from decoder: Decoder) throws {
        let c = try decoder.container(keyedBy: CodingKeys.self)
        xp = (try? c.decode(Int.self, forKey: .xp)) ?? 0
        equippedSkin = (try? c.decode(String.self, forKey: .equippedSkin)) ?? "mint"
        equippedAccessory = try? c.decode(String.self, forKey: .equippedAccessory)
        celebratedLevels = (try? c.decode([Int].self, forKey: .celebratedLevels)) ?? []
        programDays = (try? c.decode([String: [Int: Date]].self, forKey: .programDays)) ?? [:]
    }
}

final class CompanionStore: ObservableObject {
    static let key = "soft.companion.v1"

    @Published var state = CompanionState()
    // Set right after a session so the finish screen can run the xp tally.
    @Published var lastReward: SessionReward? = nil

    private let defaults = UserDefaults.standard

    init() {
        if let data = defaults.data(forKey: Self.key),
           let s = try? JSONDecoder().decode(CompanionState.self, from: data) {
            state = s
        }
    }

    func save() {
        if let data = try? JSONEncoder().encode(state) {
            defaults.set(data, forKey: Self.key)
        }
    }

    var xp: Int { state.xp }
    var level: Int { Friendship.level(for: state.xp) }
    var outfit: BuddyOutfit { BuddyOutfit(skinID: state.equippedSkin, accessoryID: state.equippedAccessory) }

    func award(seconds: Int, streakActive: Bool) {
        let old = level
        let gained = max(1, seconds / 60) + 5 + (streakActive ? 3 : 0)
        state.xp += gained
        lastReward = SessionReward(gained: gained, oldLevel: old, newLevel: level)
        save()
    }

    func isSkinUnlocked(_ id: String) -> Bool { level >= RewardTable.skinUnlockLevel(id) }
    func isAccessoryUnlocked(_ id: String) -> Bool { level >= RewardTable.accessoryUnlockLevel(id) }

    func equipSkin(_ id: String) {
        guard isSkinUnlocked(id) else { return }
        state.equippedSkin = id
        save()
    }

    func equipAccessory(_ id: String?) {
        if let id = id, !isAccessoryUnlocked(id) { return }
        state.equippedAccessory = id
        save()
    }

    func markCelebrated(_ level: Int) {
        guard !state.celebratedLevels.contains(level) else { return }
        state.celebratedLevels.append(level)
        save()
    }

    // MARK: Programs

    func completeProgramDay(programID: String, day: Int) {
        var days = state.programDays[programID] ?? [:]
        if days[day] == nil { days[day] = Date() }
        state.programDays[programID] = days
        save()
    }

    func completedDays(_ programID: String) -> Set<Int> {
        Set(state.programDays[programID]?.keys.map { $0 } ?? [])
    }

    // First incomplete day (1-based), nil when the program is done.
    func nextDay(_ programID: String, totalDays: Int) -> Int? {
        let done = completedDays(programID)
        for d in 1...totalDays where !done.contains(d) { return d }
        return nil
    }
}
```

- [ ] **Step 2: Inject into the app.** In `Soft Stretch/SoftStretchApp.swift` add `@StateObject private var companion = CompanionStore()` next to the existing `store`, and chain `.environmentObject(companion)` right after `.environmentObject(store)` on `RootView()`:

```swift
                    } else {
                        RootView()
                            .environmentObject(store)
                            .environmentObject(companion)
                    }
```

- [ ] **Step 3: Harness-test the pure logic.** Write `scratchpad/comp_test.swift` (standalone copy-import is impossible — instead compile the app file together with a tiny main):

```swift
// comp_test.swift — compiled together with CompanionStore.swift
// swiftc -o comp_test CompanionStore.swift comp_test.swift  (run from a temp dir with SwiftUI available on macOS)
import SwiftUI

assert(Friendship.level(for: 0) == 1)
assert(Friendship.level(for: 9) == 1)
assert(Friendship.level(for: 10) == 2)
assert(Friendship.level(for: 500) == 12)
assert(Friendship.level(for: 9_999) == 12)
let p = Friendship.progressToNext(12)   // level 2, base 10, next 25
assert(p.have == 2 && p.need == 15)
assert(Friendship.progressToNext(500).fraction == 1)
assert(RewardTable.reward(at: 5)?.title == "Cozy scarf")
assert(RewardTable.skinUnlockLevel("mint") == 1)
assert(RewardTable.skinUnlockLevel("sunrise") == 12)
assert(RewardTable.accessoryUnlockLevel("beanie") == 10)
print("companion logic OK")
```

Run: `cd scratchpad && cp "/Volumes/ADATA SE880/работа/development/Soft Stretch/Soft Stretch/CompanionStore.swift" . && swiftc -o comp_test CompanionStore.swift comp_test.swift && ./comp_test`
Expected: `companion logic OK`

- [ ] **Step 4: Register the file in pbxproj.** Open `Soft Stretch.xcodeproj/project.pbxproj`, find the four entries for `StretchStore.swift` (PBXBuildFile line, PBXFileReference line, the children list of the `Soft Stretch` PBXGroup, and the PBXSourcesBuildPhase files list) and add parallel entries for `CompanionStore.swift` with fresh UUIDs (e.g. take StretchStore's UUID and bump the last hex digits — any unique 24-hex string works).

- [ ] **Step 5: Build.** Run the Global-Constraints build command. Expected: `** BUILD SUCCEEDED **`.

- [ ] **Step 6: Commit.**

```bash
cd "/Volumes/ADATA SE880/работа/development/Soft Stretch"
git add -A && git -c user.name="Grigorii" -c user.email="grigoriikorniychuk@gmail.com" commit -m "feat: CompanionStore - friendship xp, levels, rewards, program progress"
```

---

### Task 2: Buddy outfits & moods

**Files:**
- Create: `Soft Stretch/BuddyOutfits.swift`
- Modify: `Soft Stretch/BuddyView.swift` (BuddyCanvas gains `outfit`/`mood`, face variants, accessory draw call; AnimatedBuddy/BuddyPreview pass-through params)
- Modify: `Soft Stretch.xcodeproj/project.pbxproj`

**Interfaces:**
- Consumes: `BuddyOutfit` from Task 1.
- Produces:
  - `struct BuddySkin { let id: String; let name: String; let body: Color; let deep: Color; let cheek: Color }`
  - `enum BuddySkins { static let all: [BuddySkin]; static func byID(_ id: String) -> BuddySkin }` (ids: mint, peach, sky, lilac, sand, sunrise)
  - `struct BuddyAccessoryInfo { let id: String; let name: String }`, `enum BuddyAccessories { static let all: [BuddyAccessoryInfo] }` (ids: sweatband, scarf, flowers, glasses, beanie, bowtie)
  - `enum BuddyMood { case content, sleepy, happy, proud }`
  - `BuddyCanvas(pose:facing:groundLevel:muscles:glowPhase:showMat:outfit:mood:)` — new params default to `.classic` / `.content` (all existing call sites compile unchanged)
  - `AnimatedBuddy(stretch:mirrored:reduceMotion:showMat:outfit:)`, `BuddyPreview(stretch:frameT:showGlow:outfit:)` — same defaults
  - `enum BuddyAccessoryDrawer { static func draw(_ id: String, in ctx: GraphicsContext, joints: BuddyRig.Joints, skin: BuddySkin) }`

- [ ] **Step 1: Write `Soft Stretch/BuddyOutfits.swift`**

```swift
import SwiftUI

// Buddy's wardrobe: unlockable skins (recolors) and Canvas-drawn accessories
// that track the skeleton joints, so they fit every pose.

struct BuddySkin {
    let id: String
    let name: String
    let body: Color
    let deep: Color
    let cheek: Color
}

enum BuddySkins {
    static let all: [BuddySkin] = [
        BuddySkin(id: "mint", name: "Classic Mint",
                  body: Color(red: 0.561, green: 0.796, blue: 0.706),
                  deep: Color(red: 0.435, green: 0.671, blue: 0.580),
                  cheek: Color(red: 0.965, green: 0.663, blue: 0.584)),
        BuddySkin(id: "peach", name: "Warm Peach",
                  body: Color(red: 0.973, green: 0.714, blue: 0.600),
                  deep: Color(red: 0.898, green: 0.580, blue: 0.463),
                  cheek: Color(red: 0.937, green: 0.494, blue: 0.475)),
        BuddySkin(id: "sky", name: "Morning Sky",
                  body: Color(red: 0.596, green: 0.769, blue: 0.851),
                  deep: Color(red: 0.463, green: 0.647, blue: 0.745),
                  cheek: Color(red: 0.949, green: 0.639, blue: 0.616)),
        BuddySkin(id: "lilac", name: "Soft Lilac",
                  body: Color(red: 0.741, green: 0.678, blue: 0.878),
                  deep: Color(red: 0.616, green: 0.549, blue: 0.780),
                  cheek: Color(red: 0.957, green: 0.647, blue: 0.667)),
        BuddySkin(id: "sand", name: "Dune Sand",
                  body: Color(red: 0.890, green: 0.796, blue: 0.635),
                  deep: Color(red: 0.788, green: 0.671, blue: 0.494),
                  cheek: Color(red: 0.937, green: 0.588, blue: 0.514)),
        BuddySkin(id: "sunrise", name: "Sunrise Gold",
                  body: Color(red: 0.949, green: 0.769, blue: 0.478),
                  deep: Color(red: 0.867, green: 0.643, blue: 0.333),
                  cheek: Color(red: 0.945, green: 0.545, blue: 0.455))
    ]

    static func byID(_ id: String) -> BuddySkin {
        all.first { $0.id == id } ?? all[0]
    }
}

struct BuddyAccessoryInfo: Identifiable {
    let id: String
    let name: String
}

enum BuddyAccessories {
    static let all: [BuddyAccessoryInfo] = [
        BuddyAccessoryInfo(id: "sweatband", name: "Sweatband"),
        BuddyAccessoryInfo(id: "scarf", name: "Cozy Scarf"),
        BuddyAccessoryInfo(id: "flowers", name: "Flower Crown"),
        BuddyAccessoryInfo(id: "glasses", name: "Round Glasses"),
        BuddyAccessoryInfo(id: "beanie", name: "Pompom Beanie"),
        BuddyAccessoryInfo(id: "bowtie", name: "Bow Tie")
    ]
}

enum BuddyMood {
    case content   // v1 face
    case sleepy    // half-lid eyes, small o mouth, drifting z z
    case happy     // arc eyes, open smile
    case proud     // wide smile + sparkles by the head
}

// All accessories are drawn AFTER the head/face so they sit on top.
// Positions derive from the head center and headAngle so they follow tilts.
enum BuddyAccessoryDrawer {
    static func draw(_ id: String, in ctx: GraphicsContext,
                     joints j: BuddyRig.Joints, skin: BuddySkin) {
        let c = j.head
        // Head tilt in radians relative to upright (-90 = upright).
        let tilt = (j.headAngle + 90) * .pi / 180
        func onHead(_ dx: CGFloat, _ dy: CGFloat) -> CGPoint {
            // Rotate the offset by the head tilt around the head center.
            let x = dx * cos(tilt) - dy * sin(tilt)
            let y = dx * sin(tilt) + dy * cos(tilt)
            return CGPoint(x: c.x + x, y: c.y + y)
        }
        switch id {
        case "sweatband":
            var band = Path()
            band.move(to: onHead(-27, -13))
            band.addQuadCurve(to: onHead(27, -13), control: onHead(0, -22))
            ctx.stroke(band, with: .color(Color(red: 0.949, green: 0.475, blue: 0.361)),
                       style: StrokeStyle(lineWidth: 9, lineCap: .round))
        case "scarf":
            let neckP = j.neck
            let loop = Path(ellipseIn: CGRect(x: neckP.x - 17, y: neckP.y - 6, width: 34, height: 16))
            ctx.fill(loop, with: .color(Color(red: 0.910, green: 0.588, blue: 0.643)))
            var tail = Path()
            tail.move(to: CGPoint(x: neckP.x + 8, y: neckP.y + 2))
            tail.addQuadCurve(to: CGPoint(x: neckP.x + 14, y: neckP.y + 26),
                              control: CGPoint(x: neckP.x + 18, y: neckP.y + 12))
            ctx.stroke(tail, with: .color(Color(red: 0.910, green: 0.588, blue: 0.643)),
                       style: StrokeStyle(lineWidth: 10, lineCap: .round))
            let fringe = Path(roundedRect: CGRect(x: neckP.x + 9, y: neckP.y + 24, width: 11, height: 7), cornerRadius: 2)
            ctx.fill(fringe, with: .color(Color(red: 0.855, green: 0.373, blue: 0.278)))
        case "flowers":
            let petals: [Color] = [Color(red: 0.957, green: 0.647, blue: 0.667),
                                   Color(red: 0.961, green: 0.757, blue: 0.361),
                                   Color(red: 0.741, green: 0.678, blue: 0.878),
                                   Color(red: 0.957, green: 0.647, blue: 0.667),
                                   Color(red: 0.961, green: 0.757, blue: 0.361)]
            for (i, color) in petals.enumerated() {
                let dx = CGFloat(i - 2) * 12
                let p = onHead(dx, -26 + abs(dx) * 0.18)
                for k in 0..<5 {
                    let a = CGFloat(k) / 5 * 2 * .pi
                    let petal = Path(ellipseIn: CGRect(x: p.x + cos(a) * 3.6 - 2.4,
                                                       y: p.y + sin(a) * 3.6 - 2.4, width: 4.8, height: 4.8))
                    ctx.fill(petal, with: .color(color))
                }
                let heart = Path(ellipseIn: CGRect(x: p.x - 2.2, y: p.y - 2.2, width: 4.4, height: 4.4))
                ctx.fill(heart, with: .color(.white.opacity(0.9)))
            }
        case "glasses":
            let ink = Color(red: 0.239, green: 0.220, blue: 0.278).opacity(0.75)
            for sx in [CGFloat(-11), 11] {
                let lens = Path(ellipseIn: CGRect(x: onHead(sx, -2).x - 8.5, y: onHead(sx, -2).y - 8.5,
                                                  width: 17, height: 17))
                ctx.stroke(lens, with: .color(ink), lineWidth: 2.4)
            }
            var bridge = Path()
            bridge.move(to: onHead(-3, -3))
            bridge.addQuadCurve(to: onHead(3, -3), control: onHead(0, -6))
            ctx.stroke(bridge, with: .color(ink), lineWidth: 2.4)
        case "beanie":
            var dome = Path()
            dome.move(to: onHead(-25, -14))
            dome.addQuadCurve(to: onHead(25, -14), control: onHead(0, -46))
            dome.closeSubpath()
            ctx.fill(dome, with: .color(Color(red: 0.616, green: 0.549, blue: 0.839)))
            var brim = Path()
            brim.move(to: onHead(-26, -13))
            brim.addQuadCurve(to: onHead(26, -13), control: onHead(0, -21))
            ctx.stroke(brim, with: .color(Color(red: 0.518, green: 0.451, blue: 0.741)),
                       style: StrokeStyle(lineWidth: 8, lineCap: .round))
            let pom = Path(ellipseIn: CGRect(x: onHead(0, -38).x - 6, y: onHead(0, -38).y - 6, width: 12, height: 12))
            ctx.fill(pom, with: .color(.white.opacity(0.92)))
        case "bowtie":
            let n = j.neck
            let tie = Color(red: 0.949, green: 0.475, blue: 0.361)
            var left = Path()
            left.move(to: CGPoint(x: n.x, y: n.y + 6))
            left.addLine(to: CGPoint(x: n.x - 13, y: n.y))
            left.addLine(to: CGPoint(x: n.x - 13, y: n.y + 12))
            left.closeSubpath()
            var right = Path()
            right.move(to: CGPoint(x: n.x, y: n.y + 6))
            right.addLine(to: CGPoint(x: n.x + 13, y: n.y))
            right.addLine(to: CGPoint(x: n.x + 13, y: n.y + 12))
            right.closeSubpath()
            ctx.fill(left, with: .color(tie))
            ctx.fill(right, with: .color(tie))
            let knot = Path(ellipseIn: CGRect(x: n.x - 3.5, y: n.y + 2.5, width: 7, height: 7))
            ctx.fill(knot, with: .color(Color(red: 0.855, green: 0.373, blue: 0.278)))
        default:
            break
        }
    }
}
```

- [ ] **Step 2: Extend `BuddyCanvas`.** In `Soft Stretch/BuddyView.swift`:

2a. Add the two new properties after `showMat`:

```swift
    var outfit: BuddyOutfit = .classic
    var mood: BuddyMood = .content
```

2b. Replace the fixed tints (`let back = SoftTheme.buddyBodyDeep` / `let front = SoftTheme.buddyBody`) with the skin:

```swift
            let skin = BuddySkins.byID(outfit.skinID)
            let back = skin.deep
            let front = skin.body
```

2c. In `drawFace`, replace `SoftTheme.buddyCheek` with a new `cheek` parameter — change the signature to `drawFace(_ context: GraphicsContext, j: BuddyRig.Joints, skin: BuddySkin)` and use `skin.cheek`; update the call site to `drawFace(context, j: j, skin: skin)`.

2d. Mood faces — inside `drawFace` front branch, wrap the eye/smile drawing in a `switch mood`:

```swift
            switch mood {
            case .content:
                for sx in [CGFloat(-11), 11] {
                    let eye = Path(ellipseIn: CGRect(x: c.x + sx + drift - 3.2, y: c.y - 6, width: 6.4, height: 8.6))
                    context.fill(eye, with: .color(ink))
                }
                var smile = Path()
                smile.move(to: CGPoint(x: c.x - 7 + drift, y: c.y + 9))
                smile.addQuadCurve(to: CGPoint(x: c.x + 7 + drift, y: c.y + 9),
                                   control: CGPoint(x: c.x + drift, y: c.y + 15))
                context.stroke(smile, with: .color(ink), style: StrokeStyle(lineWidth: 2.6, lineCap: .round))
            case .sleepy:
                for sx in [CGFloat(-11), 11] {
                    var lid = Path()
                    lid.move(to: CGPoint(x: c.x + sx - 4 + drift, y: c.y - 1))
                    lid.addQuadCurve(to: CGPoint(x: c.x + sx + 4 + drift, y: c.y - 1),
                                     control: CGPoint(x: c.x + sx + drift, y: c.y + 2.5))
                    context.stroke(lid, with: .color(ink), style: StrokeStyle(lineWidth: 2.6, lineCap: .round))
                }
                let o = Path(ellipseIn: CGRect(x: c.x - 3 + drift, y: c.y + 8, width: 6, height: 7))
                context.stroke(o, with: .color(ink), lineWidth: 2.2)
            case .happy:
                for sx in [CGFloat(-11), 11] {
                    var arc = Path()
                    arc.move(to: CGPoint(x: c.x + sx - 4 + drift, y: c.y - 2))
                    arc.addQuadCurve(to: CGPoint(x: c.x + sx + 4 + drift, y: c.y - 2),
                                     control: CGPoint(x: c.x + sx + drift, y: c.y - 8))
                    context.stroke(arc, with: .color(ink), style: StrokeStyle(lineWidth: 2.6, lineCap: .round))
                }
                var grin = Path()
                grin.move(to: CGPoint(x: c.x - 8 + drift, y: c.y + 8))
                grin.addQuadCurve(to: CGPoint(x: c.x + 8 + drift, y: c.y + 8),
                                  control: CGPoint(x: c.x + drift, y: c.y + 17))
                grin.closeSubpath()
                context.fill(grin, with: .color(ink))
            case .proud:
                for sx in [CGFloat(-11), 11] {
                    let eye = Path(ellipseIn: CGRect(x: c.x + sx + drift - 3.2, y: c.y - 6, width: 6.4, height: 8.6))
                    context.fill(eye, with: .color(ink))
                }
                var grin = Path()
                grin.move(to: CGPoint(x: c.x - 9 + drift, y: c.y + 8))
                grin.addQuadCurve(to: CGPoint(x: c.x + 9 + drift, y: c.y + 8),
                                  control: CGPoint(x: c.x + drift, y: c.y + 18))
                context.stroke(grin, with: .color(ink), style: StrokeStyle(lineWidth: 2.6, lineCap: .round))
                for (dx, dy, r) in [(CGFloat(40), CGFloat(-18), CGFloat(4)), (46, -4, 2.6)] {
                    drawSparkle(context, at: CGPoint(x: c.x + dx, y: c.y + dy), r: r,
                                color: Color(red: 0.961, green: 0.757, blue: 0.361))
                }
            }
```

Keep the cheeks after the switch (both branches), using `skin.cheek`. Apply the same `switch mood` treatment to the side-view branch with the same eye/mouth substitutions (side view: one eye, one smile — sleepy uses one lid + small o, happy one arc + filled grin, proud default eye + bigger smile; sparkles for proud in side view too).

2e. Add the sparkle helper + accessory call at the END of the Canvas body (after the muscle sparkles), plus "z z" for sleepy:

```swift
            // Wardrobe on top of everything
            if let acc = outfit.accessoryID {
                BuddyAccessoryDrawer.draw(acc, in: context, joints: j, skin: skin)
            }
            if mood == .sleepy {
                let zink = SoftTheme.ink.opacity(0.4)
                for (i, s) in [CGFloat(9), 6].enumerated() {
                    let p = CGPoint(x: j.head.x + 42 + CGFloat(i) * 12,
                                    y: j.head.y - 24 - CGFloat(i) * 14 - glowPhase * 6)
                    var z = Path()
                    z.move(to: CGPoint(x: p.x - s / 2, y: p.y - s / 2))
                    z.addLine(to: CGPoint(x: p.x + s / 2, y: p.y - s / 2))
                    z.addLine(to: CGPoint(x: p.x - s / 2, y: p.y + s / 2))
                    z.addLine(to: CGPoint(x: p.x + s / 2, y: p.y + s / 2))
                    z.closeSubpath()
                    context.stroke(z, with: .color(zink), style: StrokeStyle(lineWidth: 2, lineCap: .round, lineJoin: .round))
                }
            }
```

and file-scope in BuddyView.swift:

```swift
func drawSparkle(_ ctx: GraphicsContext, at p: CGPoint, r: CGFloat, color: Color) {
    var star = Path()
    star.move(to: CGPoint(x: p.x, y: p.y - r))
    star.addQuadCurve(to: CGPoint(x: p.x + r, y: p.y), control: p)
    star.addQuadCurve(to: CGPoint(x: p.x, y: p.y + r), control: p)
    star.addQuadCurve(to: CGPoint(x: p.x - r, y: p.y), control: p)
    star.addQuadCurve(to: CGPoint(x: p.x, y: p.y - r), control: p)
    ctx.fill(star, with: .color(color))
}
```

2f. Pass-through params: `AnimatedBuddy` gains `var outfit: BuddyOutfit = .classic` and forwards `outfit: outfit` to `BuddyCanvas`; `BuddyPreview` gains the same. (Mood stays `.content` during exercise.)

- [ ] **Step 3: Register `BuddyOutfits.swift` in pbxproj** (same four-entry pattern as Task 1 Step 4).

- [ ] **Step 4: Build.** Expected `** BUILD SUCCEEDED **` (all old call sites compile via defaults).

- [ ] **Step 5: Commit** — `feat: buddy skins, accessories and moods`.

---

### Task 3: Buddy Nook — living Home hero

**Files:**
- Create: `Soft Stretch/BuddyNookView.swift`
- Modify: `Soft Stretch/StretchTheme.swift` (daypart sky tokens)
- Modify: `Soft Stretch/HomeView.swift` (hero = nook; friendship chip; today's pick compact card; remove old `buddyCorner`)
- Modify: `Soft Stretch.xcodeproj/project.pbxproj`

**Interfaces:**
- Consumes: `CompanionStore` (level, outfit), `StretchStore` (streak, stretchedToday), `BuddySkins`, `BuddyCanvas(outfit:mood:)`, `drawSparkle`.
- Produces:
  - `enum SoftDaypart { case dawn, day, dusk, night; static func current(hour: Int) -> SoftDaypart; var skyTop: Color; var skyBottom: Color; var isDark: Bool }` (in StretchTheme.swift)
  - `struct BuddyNookView: View` — `init(level: Int, outfit: BuddyOutfit, mood: BuddyMood, reduceMotion: Bool, onTapBuddy: (() -> Void)?)`; fixed height content, no own navigation.
  - `func nookMood(store: StretchStore) -> BuddyMood` free helper in BuddyNookView.swift.

- [ ] **Step 1: Add dayparts to `StretchTheme.swift`** (append at file end):

```swift
// Time-of-day flavor for the nook window and player wash.
enum SoftDaypart {
    case dawn, day, dusk, night

    static func current(hour: Int = Calendar.current.component(.hour, from: Date())) -> SoftDaypart {
        switch hour {
        case 5..<9: return .dawn
        case 9..<17: return .day
        case 17..<21: return .dusk
        default: return .night
        }
    }

    var skyTop: Color {
        switch self {
        case .dawn: return Color(red: 0.973, green: 0.800, blue: 0.694)
        case .day: return Color(red: 0.702, green: 0.851, blue: 0.914)
        case .dusk: return Color(red: 0.894, green: 0.635, blue: 0.588)
        case .night: return Color(red: 0.294, green: 0.294, blue: 0.451)
        }
    }

    var skyBottom: Color {
        switch self {
        case .dawn: return Color(red: 0.984, green: 0.902, blue: 0.776)
        case .day: return Color(red: 0.851, green: 0.929, blue: 0.902)
        case .dusk: return Color(red: 0.729, green: 0.616, blue: 0.788)
        case .night: return Color(red: 0.412, green: 0.427, blue: 0.584)
        }
    }

    var isDark: Bool { self == .night }
}
```

- [ ] **Step 2: Write `Soft Stretch/BuddyNookView.swift`.** One `TimelineView(.animation(minimumInterval: 1/30))` + one `Canvas` (fixed design space 360×250 scaled to fit width, like BuddyCanvas). Structure:

```swift
import SwiftUI

// Buddy's nook — the living Home hero. A cozy room that fills with keepsakes
// as friendship grows; Buddy idles inside, reacts to taps and chats.

func nookMood(store: StretchStore) -> BuddyMood {
    let hour = Calendar.current.component(.hour, from: Date())
    if hour >= 21 || hour < 6 { return .sleepy }
    if store.stretchedToday { return store.currentStreak >= 3 ? .proud : .happy }
    if hour < 9 { return .sleepy }
    return .content
}

struct BuddyNookView: View {
    let level: Int
    let outfit: BuddyOutfit
    let mood: BuddyMood
    var reduceMotion: Bool = false
    var waveTrigger: Int = 0          // increment to make Buddy wave

    private static let design = CGSize(width: 360, height: 250)

    var body: some View {
        TimelineView(.animation(minimumInterval: reduceMotion ? 1 : 1.0 / 30, paused: false)) { timeline in
            Canvas { ctx, size in
                let t = reduceMotion ? 0 : timeline.date.timeIntervalSinceReferenceDate
                let scale = size.width / Self.design.width
                ctx.scaleBy(x: scale, y: scale)
                let daypart = SoftDaypart.current()
                drawRoom(ctx, daypart: daypart, t: t)
                drawItems(ctx, daypart: daypart, t: t)
                drawBuddy(ctx, t: t)
            }
        }
        .aspectRatio(Self.design.width / Self.design.height, contentMode: .fit)
    }
    // drawRoom / drawItems / drawBuddy detailed below
}
```

`drawRoom(ctx, daypart:, t:)` — design space 360×250:
- Wall: rounded rect fill `Color(red: 0.976, green: 0.937, blue: 0.882)`; at night multiply-darken by overlaying `SoftTheme.ink.opacity(0.10)`.
- Floor: rect from y 196 to bottom, `Color(red: 0.925, green: 0.847, blue: 0.757)` (+ same night overlay).
- Window (x 226, y 26, w 104, h 96, corner 14): sky `LinearGradient` daypart.skyTop→skyBottom drawn as two stacked rects (Canvas: use `.linearGradient` shading); sun disc (dawn/day/dusk) or moon crescent (night) at (278, 54) r 13; two cloud puffs drifting: `x = 226 + fmod(t * 6 + offset, 140) - 20`, clipped to the window rect (use `ctx.clip(to:)` inside a saved layer); white frame stroke 5, cross bar.
- Rug under Buddy: ellipse center (150, 216) w 150 h 30, `SoftTheme.mat.opacity(0.55)`; level ≥ 4 upgrades to two-tone (inner ellipse `SoftTheme.rose.opacity(0.25)`).
- Buddy's small mat: rounded rect (96, 206, 108, 10) `SoftTheme.mat`.

`drawItems(ctx, daypart:, t:)` — each guarded by `if level >= N`:
- L1 sprout→plant (pot at (44, 196)): pot trapezoid `SoftTheme.mat` darkened; stem + `min(level, 5)` leaves (ellipse pairs up the stem) `SoftTheme.sage`; growth = stem height `26 + CGFloat(min(level,5)) * 7`.
- L2 poster (x 36, y 44, w 56, h 72): card fill white-ish, inner mini-buddy: just head circle + smile using `BuddySkins.byID(outfit.skinID).body`, thin `SoftTheme.line` frame.
- L3 floor lamp at (312, 120): stem line ink-soft, shade trapezoid `SoftTheme.sun.opacity(0.8)`; if `daypart.isDark` add radial glow ellipse `SoftTheme.sun.opacity(0.18)` r 46 under the shade.
- L5 shelf (x 120, y 58, w 80, h 6, `SoftTheme.mat` deep) + trophy: small cup path `SoftTheme.sun`.
- L6 string lights along window top: 6 dots alternating coral/sun/sage at `y = 20 + sin(t * 2 + i) * 1.5`; night → opacity pulses `0.5 + 0.5 * sin(t * 2.4 + i)`.
- L7 hanging vine at (226 − 14, 26): 3 drooping quad-curve strands `SoftTheme.sage`, tiny leaves.
- L8 wall clock at (150, 34) r 15: face white, ink ticks, hour/minute hands from real `Date()` components.
- L9 cat on rug (x 210, y 206): sleeping curl — body ellipse 34×18 `SoftTheme.ink.opacity(0.75)`, head circle, ear triangles, tail quad-curve that flicks: end offset `sin(t * 0.8) * 4`; breathing scaleY via yOffset `sin(t * 1.3) * 0.8`.
- L10 framed photo on shelf: 18×14 frame `SoftTheme.coralDeep`, two tiny head dots (buddy body + coral).
- L11 candle right of photo: body 8×16 cream-deep, flame ellipse `SoftTheme.sun` with flicker `scale 0.8 + 0.2 * sin(t * 7)` (static when reduceMotion).
- L12 sun catcher hanging in window: small gold rhombus + 2 sparkles via `drawSparkle` with slow alpha pulse.

`drawBuddy(ctx, t:)` — reuse the rig directly:

```swift
    private func drawBuddy(_ ctx: GraphicsContext, t: TimeInterval) {
        var pose = BuddyPose.standing
        // Idle: breath bob + occasional gentle side lean ("mini stretch").
        pose.hipY = CGFloat(sin(t * 1.5)) * 1.8
        let leanPhase = fmod(t, 14)
        if leanPhase > 11 {                     // 3-second lean window every 14 s
            let k = CGFloat((leanPhase - 11) / 3)
            let ease = sin(k * .pi)             // up and back down
            pose.torsoLean = ease * 10
            pose.armRaiseL = ease * 130
        }
        // Wave overrides for ~1.2 s after a tap.
        let sinceWave = t - waveStartTime
        if waveTrigger > 0, sinceWave < 1.2, sinceWave >= 0 {
            pose.armRaiseR = 150 + CGFloat(sin(sinceWave * 14)) * 18
            pose.elbowR = 20
        }
        var buddyCtx = ctx
        // Scale the 320x360 buddy rig down into the room: draw at 0.62 scale, feet on the mat.
        buddyCtx.translateBy(x: 150 - 320 * 0.62 / 2, y: 214 - 322 * 0.62)
        buddyCtx.scaleBy(x: 0.62, y: 0.62)
        let rig = BuddyRig(pose: pose, facing: .front, groundLevel: 0)
        _ = rig // rig computed inside BuddyCanvas drawing normally; here re-draw via helper below
    }
```

**Simpler and required approach for Buddy-in-Canvas:** extract the body-drawing block of `BuddyCanvas` (from `let rig = ...` through the accessory/mood drawing) into a new file-scope function in `BuddyView.swift`:

```swift
// Draws Buddy into any GraphicsContext already scaled to the 320x360 design space.
func renderBuddy(_ context: GraphicsContext, pose: BuddyPose, facing: BuddyFacing,
                 groundLevel: CGFloat, muscles: [MuscleZone], glowPhase: CGFloat,
                 showMat: Bool, outfit: BuddyOutfit, mood: BuddyMood)
```

`BuddyCanvas.body` becomes: scale-to-fit math + `renderBuddy(...)` call. `BuddyNookView.drawBuddy` then calls `renderBuddy(buddyCtx, pose: pose, facing: .front, groundLevel: 0, muscles: [], glowPhase: CGFloat(fmod(t / 1.6, 1)), showMat: false, outfit: outfit, mood: mood)`. Do this refactor as part of THIS task (it belongs to the nook deliverable; Task 2 keeps BuddyCanvas intact).

Wave timing: store `waveStartTime` as `@State private var waveStartTime: TimeInterval = -10`; HomeView increments `waveTrigger` and the view sets `waveStartTime = Date().timeIntervalSinceReferenceDate` in `.onChange(of: waveTrigger)`. (iOS 15-safe `.onChange` is fine.)

- [ ] **Step 3: Rework `HomeView`.**

3a. Add `@EnvironmentObject var companion: CompanionStore`, `@State private var waveCount = 0`, `@State private var bubbleLine: String? = nil`, `@State private var bubbleAt = Date.distantPast`.

3b. Replace `heroCard` usage in `body`: order becomes `header`, **nookCard**, `todaysPickCard` (compact), stat tiles, `quickRow`, favorites. Delete the old `buddyCorner` section and the old big `heroCard`.

3c. `nookCard`:

```swift
    private var nookCard: some View {
        VStack(spacing: 0) {
            ZStack(alignment: .topTrailing) {
                ZStack(alignment: .bottom) {
                    BuddyNookView(level: companion.level,
                                  outfit: companion.outfit,
                                  mood: nookMood(store: store),
                                  reduceMotion: store.settings.reduceMotion,
                                  waveTrigger: waveCount)
                    if let line = bubbleLine {
                        SpeechBubble(text: line)
                            .padding(.bottom, 8)
                            .transition(.opacity.combined(with: .scale(scale: 0.9)))
                    }
                }
                .contentShape(Rectangle())
                .onTapGesture { buddyTapped() }

                NavigationLink(destination: BuddyStudioView()) {
                    FriendshipChip(level: companion.level,
                                   fraction: Friendship.progressToNext(companion.xp).fraction)
                }
                .buttonStyle(SoftPressStyle())
                .padding(10)
            }
        }
        .background(SoftTheme.card)
        .clipShape(RoundedRectangle(cornerRadius: SoftTheme.cardCorner, style: .continuous))
        .shadow(color: SoftTheme.cardShadow, radius: 12, x: 0, y: 6)
    }

    private func buddyTapped() {
        SoftHaptics.tap(store)
        waveCount += 1
        withAnimation(.spring(response: 0.35, dampingFraction: 0.8)) {
            bubbleLine = contextualLine()
        }
        bubbleAt = Date()
        DispatchQueue.main.asyncAfter(deadline: .now() + 3.5) {
            if Date().timeIntervalSince(bubbleAt) >= 3.4 {
                withAnimation(.easeOut(duration: 0.3)) { bubbleLine = nil }
            }
        }
    }
```

`BuddyStudioView` does not exist until Task 4 — in THIS task point the NavigationLink at a placeholder `EmptyView()`? **No placeholders.** Instead: Task 3 creates `FriendshipChip` + `SpeechBubble` in `BuddyNookView.swift` and the NavigationLink destination is added in Task 4; for Task 3 the chip is wrapped in a plain (non-navigating) `HStack` and Task 4 swaps it to a NavigationLink. Mark with nothing — Task 4's diff covers it.

3d. `contextualLine()` — 14 lines, picked by context then random-of-day:

```swift
    private func contextualLine() -> String {
        let hour = Calendar.current.component(.hour, from: Date())
        var pool: [String] = []
        if store.stretchedToday {
            pool = ["We already stretched - I feel taller!",
                    "Best part of my day, honestly.",
                    "Round two later? No pressure."]
        } else if hour < 9 {
            pool = ["Morning! My arms are still asleep.",
                    "A sunrise stretch would be lovely.",
                    "Yawn... shall we wake up gently?"]
        } else if hour >= 21 {
            pool = ["A slow wind-down, then bed?",
                    "My shoulders vote for Evening Unwind.",
                    "Sleepy stretches are the softest."]
        } else if store.currentStreak >= 3 {
            pool = ["Day \(store.currentStreak) together - we're on a roll!",
                    "Streak buddies! Let's keep it warm.",
                    "You keep showing up. I love that."]
        } else {
            pool = ["Got five soft minutes for me?",
                    "Desk Undo is my favorite, just saying.",
                    "Stretch like a cat. Zero guilt.",
                    "I practiced my toe-touch. Still can't.",
                    "Your neck called - it wants a tilt."]
        }
        return pool[(waveCount + (Calendar.current.ordinality(of: .day, in: .year, for: Date()) ?? 0)) % pool.count]
    }
```

3e. `SpeechBubble` + `FriendshipChip` (in `BuddyNookView.swift`):

```swift
struct SpeechBubble: View {
    let text: String
    var body: some View {
        Text(text)
            .font(SoftTheme.body(13, .semibold))
            .foregroundColor(SoftTheme.ink)
            .padding(.horizontal, 14)
            .padding(.vertical, 9)
            .background(
                Capsule().fill(Color.white.opacity(0.95))
                    .shadow(color: SoftTheme.ink.opacity(0.12), radius: 6, x: 0, y: 3)
            )
            .padding(.horizontal, 24)
    }
}

struct FriendshipChip: View {
    let level: Int
    let fraction: CGFloat
    var body: some View {
        HStack(spacing: 7) {
            ZStack {
                Circle().stroke(SoftTheme.coral.opacity(0.2), lineWidth: 3.5).frame(width: 26, height: 26)
                Circle().trim(from: 0, to: max(fraction, 0.02))
                    .stroke(SoftTheme.coral, style: StrokeStyle(lineWidth: 3.5, lineCap: .round))
                    .rotationEffect(.degrees(-90))
                    .frame(width: 26, height: 26)
                Text("\(level)")
                    .font(SoftTheme.body(11, .bold))
                    .foregroundColor(SoftTheme.coral)
            }
            Text(Friendship.levelName(level))
                .font(SoftTheme.body(12, .bold))
                .foregroundColor(SoftTheme.ink)
        }
        .padding(.horizontal, 10)
        .padding(.vertical, 6)
        .background(Capsule().fill(Color.white.opacity(0.92))
            .shadow(color: SoftTheme.ink.opacity(0.1), radius: 5, x: 0, y: 2))
    }
}
```

3f. `todaysPickCard` — compact row card (art thumb 68×68, name, minutes+subtitle, coral play circle 40) reusing the old heroCard's action; same visual grammar as the favorites rows.

- [ ] **Step 4: Register `BuddyNookView.swift` in pbxproj; build.** Expected `** BUILD SUCCEEDED **`.

- [ ] **Step 5: Commit** — `feat: living Buddy nook replaces static home hero`.

---

### Task 4: Buddy Studio

**Files:**
- Create: `Soft Stretch/BuddyStudioView.swift`
- Modify: `Soft Stretch/HomeView.swift` (chip becomes NavigationLink → `BuddyStudioView()`)
- Modify: `Soft Stretch/MoreView.swift` (add "Meet Buddy" row above Privacy Policy, NavigationLink to the studio)
- Modify: `Soft Stretch.xcodeproj/project.pbxproj`

**Interfaces:**
- Consumes: `CompanionStore` (equip/unlock API), `BuddySkins`, `BuddyAccessories`, `BuddyCanvas(outfit:mood:)`, `Friendship`, `RewardTable`.
- Produces: `struct BuddyStudioView: View` (no init params — reads environment objects).

- [ ] **Step 1: Write `Soft Stretch/BuddyStudioView.swift`**

```swift
import SwiftUI

// Buddy Studio: friendship progress + wardrobe.
struct BuddyStudioView: View {
    @EnvironmentObject var store: StretchStore
    @EnvironmentObject var companion: CompanionStore
    @Environment(\.presentationMode) var presentationMode
    @State private var previewPulse = false

    var body: some View {
        ScrollView(showsIndicators: false) {
            VStack(spacing: 16) {
                topBar
                preview
                friendshipCard
                skinsCard
                accessoriesCard
            }
            .padding(.horizontal, SoftTheme.screenPad)
            .padding(.bottom, 24)
            .frame(maxWidth: SoftTheme.contentMaxWidth)
            .frame(maxWidth: .infinity)
        }
        .background(SoftTheme.cream.ignoresSafeArea())
        .navigationBarHidden(true)
    }

    private var topBar: some View {
        HStack {
            Button(action: { presentationMode.wrappedValue.dismiss() }) {
                ZStack {
                    Circle().fill(SoftTheme.card).frame(width: 36, height: 36)
                        .shadow(color: SoftTheme.cardShadow, radius: 5, x: 0, y: 2)
                    SoftIcon(kind: .chevronLeft, size: 18, color: SoftTheme.ink)
                }
            }
            .buttonStyle(SoftPressStyle())
            Spacer()
            Text("Buddy Studio")
                .font(SoftTheme.display(20))
                .foregroundColor(SoftTheme.ink)
            Spacer()
            Color.clear.frame(width: 36, height: 36)
        }
        .padding(.top, 8)
    }

    private var preview: some View {
        ZStack {
            Circle()
                .fill(LinearGradient(colors: [BuddySkins.byID(companion.outfit.skinID).body.opacity(0.25),
                                              SoftTheme.cream],
                                     startPoint: .top, endPoint: .bottom))
                .frame(width: 230, height: 230)
            BuddyCanvas(pose: .standing, facing: .front, groundLevel: 0,
                        muscles: [], glowPhase: 0.25, showMat: false,
                        outfit: companion.outfit, mood: .happy)
                .frame(width: 210, height: 235)
        }
        .scaleEffect(previewPulse ? 1.05 : 1)
        .animation(.spring(response: 0.3, dampingFraction: 0.6), value: previewPulse)
    }

    private func pulse() {
        previewPulse = true
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.18) { previewPulse = false }
    }

    private var friendshipCard: some View {
        let prog = Friendship.progressToNext(companion.xp)
        return SoftCard {
            VStack(alignment: .leading, spacing: 10) {
                HStack {
                    VStack(alignment: .leading, spacing: 2) {
                        Text("LEVEL \(companion.level)")
                            .font(SoftTheme.body(11, .bold))
                            .foregroundColor(SoftTheme.coral)
                            .kerning(1.2)
                        Text(Friendship.levelName(companion.level))
                            .font(SoftTheme.display(20))
                            .foregroundColor(SoftTheme.ink)
                    }
                    Spacer()
                    Text(companion.level >= 12 ? "MAX" : "\(prog.need - prog.have) xp to next")
                        .font(SoftTheme.body(12, .semibold))
                        .foregroundColor(SoftTheme.inkSoft)
                }
                GeometryReader { g in
                    ZStack(alignment: .leading) {
                        Capsule().fill(SoftTheme.coral.opacity(0.12))
                        Capsule().fill(SoftTheme.coral)
                            .frame(width: max(g.size.width * prog.fraction, 8))
                    }
                }
                .frame(height: 10)
                if let next = RewardTable.reward(at: companion.level + 1) {
                    HStack(spacing: 6) {
                        SoftIcon(kind: .lock, size: 13, color: SoftTheme.inkSoft)
                        Text("Level \(next.level) unlocks: \(next.title)")
                            .font(SoftTheme.body(12, .medium))
                            .foregroundColor(SoftTheme.inkSoft)
                    }
                }
                Text("Earn friendship xp with every stretching session.")
                    .font(SoftTheme.body(12))
                    .foregroundColor(SoftTheme.inkSoft)
            }
        }
    }

    private var skinsCard: some View {
        SoftCard {
            VStack(alignment: .leading, spacing: 12) {
                Text("Skins")
                    .font(SoftTheme.display(17))
                    .foregroundColor(SoftTheme.ink)
                HStack(spacing: 12) {
                    ForEach(BuddySkins.all, id: \.id) { skin in
                        skinSwatch(skin)
                    }
                }
            }
        }
    }

    private func skinSwatch(_ skin: BuddySkin) -> some View {
        let unlocked = companion.isSkinUnlocked(skin.id)
        let equipped = companion.outfit.skinID == skin.id
        return Button(action: {
            guard unlocked else { return }
            SoftHaptics.tap(store)
            companion.equipSkin(skin.id)
            pulse()
        }) {
            VStack(spacing: 5) {
                ZStack {
                    Circle().fill(skin.body).frame(width: 40, height: 40)
                        .overlay(Circle().stroke(equipped ? SoftTheme.coral : Color.clear, lineWidth: 3))
                    if !unlocked {
                        Circle().fill(Color.white.opacity(0.55)).frame(width: 40, height: 40)
                        SoftIcon(kind: .lock, size: 14, color: SoftTheme.ink.opacity(0.55))
                    }
                }
                Text(unlocked ? skin.name.split(separator: " ").last.map(String.init) ?? skin.name
                              : "Lv \(RewardTable.skinUnlockLevel(skin.id))")
                    .font(SoftTheme.body(10, .semibold))
                    .foregroundColor(unlocked ? SoftTheme.inkSoft : SoftTheme.ink.opacity(0.35))
            }
            .frame(maxWidth: .infinity)
        }
        .buttonStyle(SoftPressStyle())
    }

    private var accessoriesCard: some View {
        SoftCard {
            VStack(alignment: .leading, spacing: 12) {
                Text("Accessories")
                    .font(SoftTheme.display(17))
                    .foregroundColor(SoftTheme.ink)
                let columns = [GridItem(.flexible()), GridItem(.flexible()), GridItem(.flexible())]
                LazyVGrid(columns: columns, spacing: 12) {
                    accessoryTile(nil, name: "None")
                    ForEach(BuddyAccessories.all) { acc in
                        accessoryTile(acc.id, name: acc.name)
                    }
                }
            }
        }
    }

    private func accessoryTile(_ id: String?, name: String) -> some View {
        let unlocked = id == nil || companion.isAccessoryUnlocked(id!)
        let equipped = companion.outfit.accessoryID == id
        return Button(action: {
            guard unlocked else { return }
            SoftHaptics.tap(store)
            companion.equipAccessory(id)
            pulse()
        }) {
            VStack(spacing: 6) {
                ZStack {
                    RoundedRectangle(cornerRadius: 14, style: .continuous)
                        .fill(SoftTheme.cream)
                        .frame(height: 62)
                        .overlay(RoundedRectangle(cornerRadius: 14, style: .continuous)
                            .stroke(equipped ? SoftTheme.coral : SoftTheme.line, lineWidth: equipped ? 2.5 : 1))
                    if let id = id {
                        AccessoryMini(accessoryID: id, skinID: companion.outfit.skinID)
                            .frame(width: 54, height: 54)
                            .opacity(unlocked ? 1 : 0.35)
                    } else {
                        SoftIcon(kind: .close, size: 18, color: SoftTheme.inkSoft)
                    }
                    if !unlocked, let id = id {
                        VStack {
                            Spacer()
                            Text("Lv \(RewardTable.accessoryUnlockLevel(id))")
                                .font(SoftTheme.body(10, .bold))
                                .foregroundColor(SoftTheme.ink.opacity(0.5))
                                .padding(.bottom, 4)
                        }
                    }
                }
                Text(name)
                    .font(SoftTheme.body(11, .semibold))
                    .foregroundColor(SoftTheme.inkSoft)
                    .lineLimit(1)
            }
        }
        .buttonStyle(SoftPressStyle())
    }
}

// Small buddy-head preview wearing one accessory (for the studio grid).
struct AccessoryMini: View {
    let accessoryID: String
    let skinID: String

    var body: some View {
        Canvas { ctx, size in
            let scale = size.width / 320
            ctx.scaleBy(x: scale, y: scale)
            // Head-only crop: translate so the head area fills the tile.
            ctx.translateBy(x: 0, y: -30)
            var pose = BuddyPose.standing
            pose.breathe = 0
            renderBuddy(ctx, pose: pose, facing: .front, groundLevel: -240,
                        muscles: [], glowPhase: 0, showMat: false,
                        outfit: BuddyOutfit(skinID: skinID, accessoryID: accessoryID),
                        mood: .content)
        }
    }
}
```

Note: `groundLevel: -240` pushes the floor far down so the standing buddy's head lands in the visible tile; tune by eye in the screenshot pass (acceptable range −220…−260).

- [ ] **Step 2: Home chip → NavigationLink** (`HomeView.nookCard`): wrap `FriendshipChip` in `NavigationLink(destination: BuddyStudioView())` with `SoftPressStyle`.

- [ ] **Step 3: More tab row.** In `MoreView.swift`, above the Privacy row add a nav row "Meet Buddy — friendship, skins and keepsakes" with `SoftIcon(kind: .heart)` tint coral, `NavigationLink` to `BuddyStudioView()` (match the existing row style used for Privacy).

- [ ] **Step 4: Register file; build.** Expected `** BUILD SUCCEEDED **`.

- [ ] **Step 5: Commit** — `feat: Buddy Studio - friendship progress and wardrobe`.

---

### Task 5: Programs — 7-day journeys

**Files:**
- Create: `Soft Stretch/ProgramLibrary.swift`
- Create: `Soft Stretch/ProgramsView.swift`
- Modify: `Soft Stretch/RootView.swift` (`activePlayer` becomes `PlayerLaunch`; `startProgramDay`)
- Modify: `Soft Stretch/RoutinesView.swift` (segmented header: Routines | Programs | My Own)
- Modify: `Soft Stretch/PlayerView.swift` (accepts optional program context, reports completion)
- Modify: `Soft Stretch.xcodeproj/project.pbxproj` (2 files)

**Interfaces:**
- Consumes: `RoutineLibrary.byID`, `CompanionStore.completeProgramDay/completedDays/nextDay`.
- Produces:
  - `struct ProgramDay { let day: Int; let routineID: String; let title: String; let blurb: String }`
  - `struct StretchProgram: Identifiable { let id: String; let name: String; let tagline: String; let artName: String; let tint: Color; let days: [ProgramDay] }`
  - `enum ProgramLibrary { static let all: [StretchProgram]; static func byID(_ id: String) -> StretchProgram? }`
  - `struct PlayerLaunch: Identifiable { var id: String { routine.id + String(programDay ?? -1) }; let routine: Routine; var programID: String?; var programDay: Int? }`
  - `RootView.startRoutine(_ routine: Routine)` unchanged signature; new `startProgramDay(program: StretchProgram, day: ProgramDay)`.
  - `struct ProgramsListView: View { let startProgramDay: (StretchProgram, ProgramDay) -> Void }`, `struct ProgramDetailView: View` (same closure param).
  - `struct SoftSegmented: View { let items: [String]; @Binding var selection: Int }` (goes in `SoftComponents.swift`).

- [ ] **Step 1: Write `Soft Stretch/ProgramLibrary.swift`** — 3 programs, day routines drawn from existing ids `morning, desk, evening, fullbody, neckmelt, hips, hamstring, posture` (verify exact ids with `grep 'Routine(id:' PoseLibrary.swift` → they are: morning, desk, evening, fullbody, neckmelt, hips, hamstring, posture):

```swift
import SwiftUI

// 7-day guided journeys built from the existing routine library.

struct ProgramDay {
    let day: Int
    let routineID: String
    let title: String
    let blurb: String
}

struct StretchProgram: Identifiable {
    let id: String
    let name: String
    let tagline: String
    let artName: String     // reuses a routine cover
    let tint: Color
    let days: [ProgramDay]
}

enum ProgramLibrary {
    static let all: [StretchProgram] = [
        StretchProgram(
            id: "gentle-week", name: "Gentle Week",
            tagline: "Seven soft days to wake your whole body",
            artName: "cover_fullbody", tint: SoftTheme.sage,
            days: [
                ProgramDay(day: 1, routineID: "morning", title: "Say hello", blurb: "Easy sunrise wake-up"),
                ProgramDay(day: 2, routineID: "neckmelt", title: "Soft neck", blurb: "Melt the top tension"),
                ProgramDay(day: 3, routineID: "fullbody", title: "Head to toe", blurb: "Your first full flow"),
                ProgramDay(day: 4, routineID: "hips", title: "Happy hips", blurb: "Open what sitting closed"),
                ProgramDay(day: 5, routineID: "posture", title: "Stand tall", blurb: "Chest open, shoulders back"),
                ProgramDay(day: 6, routineID: "hamstring", title: "Long legs", blurb: "Gentle hamstring care"),
                ProgramDay(day: 7, routineID: "fullbody", title: "Full circle", blurb: "The flow, now familiar")
            ]),
        StretchProgram(
            id: "desk-rescue", name: "Desk Rescue",
            tagline: "A week against screen-shaped shoulders",
            artName: "cover_desk", tint: SoftTheme.lavender,
            days: [
                ProgramDay(day: 1, routineID: "desk", title: "Undo the day", blurb: "The classic desk reset"),
                ProgramDay(day: 2, routineID: "neckmelt", title: "Neck melt", blurb: "Tilt away the stiffness"),
                ProgramDay(day: 3, routineID: "posture", title: "Reset posture", blurb: "Like you mean it"),
                ProgramDay(day: 4, routineID: "desk", title: "Desk undo II", blurb: "Deeper this time"),
                ProgramDay(day: 5, routineID: "hips", title: "Chair-free hips", blurb: "Sitting undone"),
                ProgramDay(day: 6, routineID: "neckmelt", title: "Soft again", blurb: "Neck and traps, round two"),
                ProgramDay(day: 7, routineID: "posture", title: "Tall finish", blurb: "Walk out taller")
            ]),
        StretchProgram(
            id: "winddown-week", name: "Wind-Down Week",
            tagline: "Seven calm evenings, softer sleep",
            artName: "cover_evening", tint: SoftTheme.sky,
            days: [
                ProgramDay(day: 1, routineID: "evening", title: "First unwind", blurb: "Slow evening ritual"),
                ProgramDay(day: 2, routineID: "hamstring", title: "Leg release", blurb: "Let the legs let go"),
                ProgramDay(day: 3, routineID: "evening", title: "Unwind again", blurb: "Deeper calm tonight"),
                ProgramDay(day: 4, routineID: "hips", title: "Hip lullaby", blurb: "Rock the hips loose"),
                ProgramDay(day: 5, routineID: "neckmelt", title: "Pillow prep", blurb: "A soft neck sleeps well"),
                ProgramDay(day: 6, routineID: "hamstring", title: "Long and light", blurb: "Hamstrings, kindly"),
                ProgramDay(day: 7, routineID: "evening", title: "Softest night", blurb: "The full wind-down")
            ])
    ]

    static func byID(_ id: String) -> StretchProgram? {
        all.first { $0.id == id }
    }
}
```

(Before writing, confirm routine ids: `grep -o 'Routine(id: "[a-z]*"' "Soft Stretch/PoseLibrary.swift"` — adjust `neckmelt` if the actual id differs; HomeView references `RoutineLibrary.byID["neckmelt"]` — wait, HomeView uses `byID["neckmelt"]`? It uses `byID["desk"]` and `byID["neckmelt"]`. Verify and use the real ids.)

- [ ] **Step 2: `SoftSegmented` in `SoftComponents.swift`** (append):

```swift
// Pill segmented control used on the Routines tab.
struct SoftSegmented: View {
    let items: [String]
    @Binding var selection: Int

    var body: some View {
        HStack(spacing: 4) {
            ForEach(Array(items.enumerated()), id: \.offset) { i, item in
                Button(action: {
                    withAnimation(.spring(response: 0.3, dampingFraction: 0.85)) { selection = i }
                }) {
                    Text(item)
                        .font(SoftTheme.body(13, selection == i ? .bold : .medium))
                        .foregroundColor(selection == i ? .white : SoftTheme.inkSoft)
                        .padding(.vertical, 8)
                        .frame(maxWidth: .infinity)
                        .background(
                            Capsule().fill(selection == i ? SoftTheme.coral : Color.clear)
                        )
                }
                .buttonStyle(SoftPressStyle())
            }
        }
        .padding(4)
        .background(Capsule().fill(SoftTheme.card)
            .shadow(color: SoftTheme.cardShadow, radius: 6, x: 0, y: 3))
    }
}
```

- [ ] **Step 3: `PlayerLaunch` plumbing in `RootView.swift`.**

```swift
struct PlayerLaunch: Identifiable {
    let routine: Routine
    var programID: String? = nil
    var programDay: Int? = nil
    var id: String { routine.id + "-" + String(programDay ?? -1) }
}
```

- `@State private var activePlayer: PlayerLaunch? = nil`
- `startRoutine` body: `activePlayer = PlayerLaunch(routine: routine)`
- new: `private func startProgramDay(_ program: StretchProgram, _ day: ProgramDay) { guard let routine = RoutineLibrary.byID[day.routineID] else { return }; SoftHaptics.step(store); activePlayer = PlayerLaunch(routine: routine, programID: program.id, programDay: day.day) }`
- `.fullScreenCover(item: $activePlayer) { launch in PlayerView(routine: launch.routine, programID: launch.programID, programDay: launch.programDay).environmentObject(store).environmentObject(companion) }` — also add `@EnvironmentObject var companion: CompanionStore` to RootView.
- Pass `startProgramDay` into `RoutinesView(startRoutine: startRoutine, startProgramDay: startProgramDay)`.

- [ ] **Step 4: PlayerView signature.** Add stored `var programID: String? = nil` and `var programDay: Int? = nil` (defaulted — HomeView/RoutineDetail call sites compile). In `finishSession()` after `store.recordSession(...)` add:

```swift
        if let pid = programID, let day = programDay {
            companion.completeProgramDay(programID: pid, day: day)
        }
```

with `@EnvironmentObject var companion: CompanionStore` added to PlayerView. (XP award itself is Task 7 — this task only wires program completion; the ceremony comes later. `companion.award` is NOT called yet.)

- [ ] **Step 5: `RoutinesView` segmented rework.** Add `@State private var segment = 0`, closure `let startProgramDay: (StretchProgram, ProgramDay) -> Void`. Body becomes:

```swift
            VStack(spacing: 16) {
                SectionHeader(title: "Routines",
                              subtitle: "Follow along with Buddy")
                    .padding(.top, 8)
                SoftSegmented(items: ["Routines", "Programs", "My Own"], selection: $segment)
                switch segment {
                case 0: routineList          // the existing ForEach extracted into a private var
                case 1: ProgramsListView(startProgramDay: startProgramDay)
                default: MyRoutinesSection(startRoutine: startRoutine)   // Task 6; in THIS task use programs-only: see below
                }
            }
```

**Ordering note:** Task 6 creates `MyRoutinesSection`. In THIS task, use `items: ["Routines", "Programs"]` (two segments only); Task 6 flips it to three. No dead references.

- [ ] **Step 6: Write `Soft Stretch/ProgramsView.swift`** — `ProgramsListView` (3 tall cards: cover art 130 high, name, tagline, 7 progress dots — filled `program.tint` for done days, ring for next, faint for rest, "Day N next" pill or "Completed" ribbon) and `ProgramDetailView` (hero cover, tagline, overall progress capsule bar, 7 day rows: day number disc, title+blurb, state — done: sage check disc + date; next: coral Start button calling `startProgramDay(program, day)`; future: soft lock icon; rows are repeatable — tapping a done row's small replay icon also starts it). All-days-done → celebration card at top ("You finished \(name)! Buddy is very proud." + proud mini buddy `BuddyCanvas(... mood: .proud)` 90×100). Use `@EnvironmentObject var companion: CompanionStore` for `completedDays`/`nextDay`. NavigationLink from list card → detail.

Complete code for both views is written at implementation time following exactly the visual grammar of `RoutineCard`/`RoutineDetailView` (card corner `SoftTheme.cardCorner`, shadows `SoftTheme.cardShadow`, pills `SoftPill`, buttons `SoftPrimaryButton`); dots row:

```swift
    HStack(spacing: 6) {
        ForEach(1...7, id: \.self) { d in
            let done = companion.completedDays(program.id).contains(d)
            let next = companion.nextDay(program.id, totalDays: 7) == d
            Circle()
                .fill(done ? program.tint : program.tint.opacity(next ? 0.0 : 0.15))
                .overlay(Circle().stroke(next ? program.tint : Color.clear, lineWidth: 2))
                .frame(width: 10, height: 10)
        }
    }
```

- [ ] **Step 7: Register both files; build.** Expected `** BUILD SUCCEEDED **`.

- [ ] **Step 8: Commit** — `feat: 7-day programs with day tracking`.

---

### Task 6: My Own — custom routine builder

**Files:**
- Create: `Soft Stretch/RoutineBuilderView.swift`
- Modify: `Soft Stretch/StretchStore.swift` (custom routines storage)
- Modify: `Soft Stretch/StretchModels.swift` (CustomRoutine model)
- Modify: `Soft Stretch/RoutinesView.swift` (third segment)
- Modify: `Soft Stretch.xcodeproj/project.pbxproj`

**Interfaces:**
- Consumes: `PoseLibrary.all/byID`, `BodyArea.tint`, `SoftSegmented`, `startRoutine` closure.
- Produces:
  - `struct CustomStep: Codable { var stretchID: String; var seconds: Int }`
  - `struct CustomRoutine: Codable, Identifiable { var id: UUID; var name: String; var tintAreaRaw: String; var steps: [CustomStep]; func asRoutine() -> Routine }`
  - `StretchStore`: `@Published var customRoutines: [CustomRoutine]`, `customKey = "soft.customroutines.v1"`, `func saveCustomRoutine(_ r: CustomRoutine)` (insert-or-replace by id), `func deleteCustomRoutine(_ id: UUID)`; loaded in `load()`, saved in `saveAll()`.
  - `struct MyRoutinesSection: View { let startRoutine: (Routine) -> Void }`
  - `struct RoutineBuilderView: View { var editing: CustomRoutine? = nil; let onSave: (CustomRoutine) -> Void }` (presented as `.sheet`)

- [ ] **Step 1: Model in `StretchModels.swift`** (append):

```swift
// User-built routines (My Own tab).
struct CustomStep: Codable {
    var stretchID: String
    var seconds: Int
}

struct CustomRoutine: Codable, Identifiable {
    var id: UUID = UUID()
    var name: String
    var tintAreaRaw: String = BodyArea.fullBody.rawValue
    var steps: [CustomStep]

    var tintArea: BodyArea { BodyArea(rawValue: tintAreaRaw) ?? .fullBody }

    func asRoutine() -> Routine {
        Routine(id: "custom-\(id.uuidString)",
                name: name,
                subtitle: "Your own routine",
                artName: "",
                tintArea: tintArea,
                steps: steps.map { RoutineStep(stretchID: $0.stretchID, secondsPerSide: $0.seconds) })
    }
}
```

- [ ] **Step 2: Storage in `StretchStore.swift`.** Add `static let customKey = "soft.customroutines.v1"`, `@Published var customRoutines: [CustomRoutine] = []`; in `load()` decode `[CustomRoutine]` from `customKey` with `try?` fallback `[]`; in `saveAll()` encode and set; add:

```swift
    func saveCustomRoutine(_ r: CustomRoutine) {
        if let idx = customRoutines.firstIndex(where: { $0.id == r.id }) {
            customRoutines[idx] = r
        } else {
            customRoutines.append(r)
        }
        saveAll()
    }

    func deleteCustomRoutine(_ id: UUID) {
        customRoutines.removeAll { $0.id == id }
        saveAll()
    }
```

- [ ] **Step 3: Write `Soft Stretch/RoutineBuilderView.swift`** containing BOTH `MyRoutinesSection` and `RoutineBuilderView`:

`MyRoutinesSection`: `@EnvironmentObject var store: StretchStore`, `@State private var editorTarget: BuilderTarget? = nil` where `struct BuilderTarget: Identifiable { let routine: CustomRoutine?; let id: String }` (id = routine id or "new" — makes one `.sheet(item:)` handle create + edit). Cards for each custom routine: tint gradient header block (LinearGradient `tintArea.tint.opacity(0.5)→0.25`, height 84) with a `BuddyPreview(stretch: firstStretch, showGlow: false)` floating on it, name, "\(steps.count) stretches · \(minutes) min", play circle button calling `startRoutine(r.asRoutine())`, small edit (pencil = reuse `.more` icon rotated? NO — use text button "Edit") and delete (uses `.close` icon in a soft red circle with a confirm `alert`). Dashed "New routine" card: `RoundedRectangle` stroked `style: StrokeStyle(lineWidth: 2, dash: [7])` in `SoftTheme.coral.opacity(0.5)`, plus a coral `+`-like sparkle icon (use `.sparkle`) and "Build your own" label → sets `editorTarget = BuilderTarget(routine: nil, id: "new")`. `.sheet(item: $editorTarget) { t in RoutineBuilderView(editing: t.routine) { store.saveCustomRoutine($0) } }`. Empty state (no customs yet): a friendly SoftCard "No routines of your own yet - build one and Buddy will learn it." above the dashed card.

`RoutineBuilderView`: `@Environment(\.presentationMode)`, `@State var name: String`, `@State var tint: BodyArea`, `@State var steps: [CustomStep]`, initialized from `editing` (or "", .fullBody, []). Layout (`NavigationView`-free custom sheet):
- Grab bar + "New Routine"/"Edit Routine" title + Close circle.
- Name field: `TextField("Name it something soft", text: $name)` styled: `.font(SoftTheme.body(16, .semibold))`, padding 14, background rounded 16 `SoftTheme.cream`, `.disableAutocorrection(true)`.
- Tint row: 4 circles (`BodyArea.allCases.filter { $0 != .fullBody } + [.fullBody]` → actually all 5) filled with `area.tint`, ring when selected.
- "Your flow" list: for each step index — row with `BuddyPreview` thumb 44, stretch name, seconds chip button cycling 20→30→45→60 (`SoftPill` in a Button), up/down arrows (`.arrowUp` icon, second one rotated 180°) disabled at edges, remove `.close` button. (Reorder via arrows: `steps.swapAt`.)
- "Add stretches" section: `ForEach(BodyArea.allCases.filter { $0 != .fullBody })` group headers + rows for `PoseLibrary.area(area)` with `+ Add` pill button appending `CustomStep(stretchID: s.id, seconds: 30)`; a stretch can be added multiple times.
- Footer summary "N stretches · ~M min" (`M = steps.reduce(0){ $0 + $1.seconds * ((PoseLibrary.byID[$1.stretchID]?.bilateral ?? false) ? 2 : 1) } / 60`) + `SoftPrimaryButton("Save routine")` disabled (opacity 0.45, no action) unless `name.trimmingCharacters(in: .whitespaces).count >= 1 && steps.count >= 2`; on save: build `CustomRoutine(id: editing?.id ?? UUID(), name: trimmed, tintAreaRaw: tint.rawValue, steps: steps)`, call `onSave`, haptic success, dismiss.

- [ ] **Step 4: Third segment in `RoutinesView`.** `SoftSegmented(items: ["Routines", "Programs", "My Own"], ...)`; `default:` case renders `MyRoutinesSection(startRoutine: startRoutine)`.

- [ ] **Step 5: Register file; build; manual reasoning check** — `PlayerView` already plays any `Routine`; custom ids `"custom-…"` flow into `SessionRecord.routineID` harmlessly (badges' `explorer` counts distinct ids from `RoutineLibrary.all` only — verify `evaluateBadges` uses `RoutineLibrary.all.count` comparison: `distinctRoutinesTried.count >= RoutineLibrary.all.count` — custom ids INFLATE the left side. **Fix in this task:** change to `Set(sessions.map { $0.routineID }).intersection(Set(RoutineLibrary.all.map { $0.id })).count >= RoutineLibrary.all.count` — i.e. replace the `distinctRoutinesTried` computed var body with `Set(sessions.map { $0.routineID }).intersection(Set(RoutineLibrary.all.map(\.id)))`.)

- [ ] **Step 6: Commit** — `feat: custom routine builder (My Own)`.

---

### Task 7: Player finish — xp tally, level-up ceremony, confetti v2, outfit

**Files:**
- Modify: `Soft Stretch/PlayerView.swift`
- Modify: `Soft Stretch/SoftComponents.swift` (ConfettiBurst v2)

**Interfaces:**
- Consumes: `CompanionStore.award(seconds:streakActive:)`, `lastReward`, `RewardTable.reward(at:)`, `markCelebrated`, `BuddyCanvas(outfit:mood:)`, `AnimatedBuddy(outfit:)`, `SoftDaypart`.
- Produces: nothing new outside the file.

- [ ] **Step 1: Outfit + backdrop.** `buddyStage`'s `AnimatedBuddy(...)` gains `outfit: companion.outfit`. `backgroundGradient` becomes a 4-stop gradient mixing the area tint with the daypart: `[SoftTheme.cream, daypart.skyBottom.opacity(0.18), tint.opacity(0.16), SoftTheme.cream]` where `let daypart = SoftDaypart.current()`.

- [ ] **Step 2: Award xp on finish.** In `finishSession()` (and the ≥60 s branch of `closeEarly()` — award there too, but WITHOUT ceremony since the view dismisses):

```swift
        let hadStreakYesterday: Bool = {
            guard let y = Calendar.current.date(byAdding: .day, value: -1, to: Date()) else { return false }
            return !store.sessions(on: y).isEmpty
        }()
        companion.award(seconds: Int(totalActiveSeconds),
                        streakActive: hadStreakYesterday || store.sessions(on: Date()).count > 1)
```

(`recordSession` ran first, so "today has another session" means count > 1.)

- [ ] **Step 3: Completion view additions**, right after the stat tiles `HStack`:

```swift
                    if let reward = companion.lastReward {
                        XPTallyCard(reward: reward)
                            .padding(.horizontal, SoftTheme.screenPad)
                    }
                    if let reward = companion.lastReward, reward.newLevel > reward.oldLevel {
                        LevelUpCard(newLevel: reward.newLevel,
                                    onEquip: { equipFreshReward(reward.newLevel) })
                            .padding(.horizontal, SoftTheme.screenPad)
                            .onAppear { companion.markCelebrated(reward.newLevel) }
                    }
                    if let day = programDay, let pid = programID,
                       let program = ProgramLibrary.byID(pid) {
                        SoftPill(text: "Day \(day) of \(program.name) complete",
                                 tint: program.tint)
                    }
```

with, in the same file:

```swift
    private func equipFreshReward(_ level: Int) {
        guard let reward = RewardTable.reward(at: level) else { return }
        switch reward.kind {
        case .skin(let id): companion.equipSkin(id)
        case .accessory(let id): companion.equipAccessory(id)
        }
        SoftHaptics.success(store)
    }
```

`XPTallyCard` (new struct in PlayerView.swift) — counts up `shown` from 0 to `reward.gained` on a 0.06 s timer with tick haptic every 3:

```swift
struct XPTallyCard: View {
    let reward: SessionReward
    @State private var shown = 0
    private let timer = Timer.publish(every: 0.06, on: .main, in: .common).autoconnect()

    var body: some View {
        HStack(spacing: 10) {
            ZStack {
                Circle().fill(SoftTheme.coral.opacity(0.14)).frame(width: 42, height: 42)
                SoftIcon(kind: .heartFill, size: 20, color: SoftTheme.coral)
            }
            VStack(alignment: .leading, spacing: 2) {
                Text("FRIENDSHIP")
                    .font(SoftTheme.body(10, .bold)).foregroundColor(SoftTheme.coral).kerning(1.3)
                Text("+\(shown) xp with Buddy")
                    .font(SoftTheme.display(18)).foregroundColor(SoftTheme.ink)
            }
            Spacer()
            Text(Friendship.levelName(reward.newLevel))
                .font(SoftTheme.body(12, .semibold)).foregroundColor(SoftTheme.inkSoft)
        }
        .padding(14)
        .background(RoundedRectangle(cornerRadius: 18, style: .continuous)
            .fill(SoftTheme.card).shadow(color: SoftTheme.cardShadow, radius: 8, x: 0, y: 4))
        .onReceive(timer) { _ in
            if shown < reward.gained { shown += 1 }
        }
    }
}
```

`LevelUpCard` — coral-gradient card: sparkle row (3× `drawSparkle` via a tiny Canvas 60×20), "LEVEL \(newLevel)" kerned caps, `Friendship.levelName(newLevel)` display 22 white, reward line `RewardTable.reward(at: newLevel)?.title`, secondary line "…and something new appeared in Buddy's nook", white capsule button "Equip now" → `onEquip` (hide the button after tap via `@State var equipped`, replaced by "Equipped ✓-style text `"Equipped"` pill — plain text, no symbol). Gradient `LinearGradient(colors: [SoftTheme.coral, SoftTheme.rose], startPoint: .topLeading, endPoint: .bottomTrailing)`.

- [ ] **Step 4: Confetti v2.** In `SoftComponents.swift` replace the single rounded-rect particle body with a 3-shape mix keyed on `i % 3`: capsules (existing rect, cornerRadius 3), circles (`Path(ellipseIn:)`), and 4-point sparkle stars (reuse `drawSparkle`), 56 particles instead of 46. Keep everything else (deterministic seeds, fall phase). `drawSparkle` lives in BuddyView.swift at file scope — already visible target-wide.

- [ ] **Step 5: Also reset `companion.lastReward = nil` in `PlayerView.onDisappear`** so a later session starts clean.

- [ ] **Step 6: Build; commit** — `feat: xp tally, level-up ceremony, program chip, confetti v2`.

---

### Task 8: Progress tab — friendship card, month heatmap, body balance

**Files:**
- Modify: `Soft Stretch/StretchModels.swift` (`SessionRecord.areaSeconds`)
- Modify: `Soft Stretch/StretchStore.swift` (recordSession signature)
- Modify: `Soft Stretch/PlayerView.swift` (accumulate + pass areaSeconds)
- Modify: `Soft Stretch/ProgressTabView.swift` (new cards)

**Interfaces:**
- Produces:
  - `SessionRecord.areaSeconds: [String: Int]?` — optional; v1 records decode as nil.
  - `StretchStore.recordSession(routine:seconds:stretchCount:areaSeconds:)` — new **defaulted** param `areaSeconds: [String: Int]? = nil`.
  - `StretchStore.areaBalance() -> [(area: BodyArea, fraction: CGFloat)]` — share of seconds per non-fullBody area across all sessions that carry areaSeconds (fullBody seconds are attributed by the player per actual stretch area, so fullBody never appears as a key).
  - `StretchStore.minutes(on day: Date) -> Int`.

- [ ] **Step 1: Model.** `SessionRecord` gains `var areaSeconds: [String: Int]? = nil` (auto-synthesized Codable keeps it optional → tolerant for old data). `recordSession` stores it. In `PlayerView.tick()` accumulate per-area seconds where the active-time counter already increments:

```swift
        elapsedInSegment += 0.1
        totalActiveSeconds += 0.1
        if let seg = current {
            let key = seg.stretch.area.rawValue
            areaSecondsAcc[key, default: 0] += 0.1
        }
```

with `@State private var areaSecondsAcc: [String: Double] = [:]` and both record calls passing `areaSeconds: areaSecondsAcc.mapValues { Int($0) }`.

**Note:** for stretches whose own `area == .fullBody` there are none — every `Stretch.area` is a concrete area (fullBody is only a routine tint). Verify with `grep 'area: .fullBody' PoseLibrary.swift` → expect no stretch hits; if any stretch uses `.fullBody`, map it to `.backCore` in the accumulator.

- [ ] **Step 2: Store math** (append to StretchStore):

```swift
    func minutes(on day: Date) -> Int {
        sessions(on: day).reduce(0) { $0 + $1.seconds } / 60
    }

    // Share of stretched time per body area (sessions with area data only).
    func areaBalance() -> [(area: BodyArea, fraction: CGFloat)] {
        var totals: [String: Int] = [:]
        for s in sessions {
            for (k, v) in s.areaSeconds ?? [:] { totals[k, default: 0] += v }
        }
        let sum = totals.values.reduce(0, +)
        let areas: [BodyArea] = [.neckShoulders, .backCore, .hipsLegs, .armsChest]
        return areas.map { a in
            (a, sum > 0 ? CGFloat(totals[a.rawValue] ?? 0) / CGFloat(sum) : 0)
        }
    }
```

- [ ] **Step 3: Progress UI.** In `ProgressTabView` insert, top-down: friendship summary card (reuses `FriendshipChip` internals at larger scale — level ring 44, name, xp capsule bar, `NavigationLink` "Open Buddy Studio" chevron row), then existing tiles, then **month heatmap card**, then **body balance card**, then the existing two-week chart and badges.

Heatmap card (new structs in ProgressTabView.swift):

```swift
struct MonthHeatmap: View {
    @EnvironmentObject var store: StretchStore
    @State private var monthOffset = 0    // 0 = current month, negative = past

    private var monthStart: Date {
        let cal = Calendar.current
        let comps = cal.dateComponents([.year, .month], from: Date())
        let start = cal.date(from: comps) ?? Date()
        return cal.date(byAdding: .month, value: monthOffset, to: start) ?? start
    }

    var body: some View {
        SoftCard {
            VStack(spacing: 12) {
                HStack {
                    monthArrow(dir: -1)
                    Spacer()
                    Text(monthTitle)
                        .font(SoftTheme.body(15, .bold)).foregroundColor(SoftTheme.ink)
                    Spacer()
                    monthArrow(dir: 1)
                }
                grid
                HStack(spacing: 10) {
                    Text("less").font(SoftTheme.body(10)).foregroundColor(SoftTheme.inkSoft)
                    ForEach(0..<4, id: \.self) { i in
                        RoundedRectangle(cornerRadius: 3)
                            .fill(SoftTheme.coral.opacity(intensity(i)))
                            .frame(width: 14, height: 14)
                    }
                    Text("more").font(SoftTheme.body(10)).foregroundColor(SoftTheme.inkSoft)
                }
            }
        }
    }

    private func intensity(_ step: Int) -> Double { [0.08, 0.3, 0.55, 0.9][step] }

    private func stepFor(minutes: Int) -> Int {
        switch minutes {
        case 0: return 0
        case 1..<5: return 1
        case 5..<12: return 2
        default: return 3
        }
    }

    private var monthTitle: String {
        let f = DateFormatter(); f.locale = Locale(identifier: "en_US")
        f.dateFormat = "MMMM yyyy"
        return f.string(from: monthStart)
    }

    private func monthArrow(dir: Int) -> some View {
        let cal = Calendar.current
        let target = cal.date(byAdding: .month, value: monthOffset + dir, to: Date()) ?? Date()
        let disabled = dir > 0 ? monthOffset >= 0
            : (store.sessions.first.map { target < cal.date(byAdding: .month, value: -1, to: cal.startOfDay(for: $0.date))! } ?? true)
        return Button(action: { if !disabled { monthOffset += dir } }) {
            SoftIcon(kind: dir < 0 ? .chevronLeft : .chevronRight, size: 15,
                     color: disabled ? SoftTheme.ink.opacity(0.18) : SoftTheme.inkSoft)
                .frame(width: 30, height: 30)
        }
        .buttonStyle(SoftPressStyle())
        .disabled(disabled)
    }

    private var grid: some View {
        let cal = Calendar.current
        let range = cal.range(of: .day, in: .month, for: monthStart) ?? 1..<29
        let firstWeekday = cal.component(.weekday, from: monthStart)      // 1 = Sun
        let leading = (firstWeekday + 5) % 7                              // Mon-start blanks
        let cells: [Int?] = Array(repeating: nil, count: leading) + range.map { $0 }
        let columns = Array(repeating: GridItem(.flexible(), spacing: 5), count: 7)
        return LazyVGrid(columns: columns, spacing: 5) {
            ForEach(["M", "T", "W", "T", "F", "S", "S"].indices, id: \.self) { i in
                Text(["M", "T", "W", "T", "F", "S", "S"][i])
                    .font(SoftTheme.body(10, .semibold)).foregroundColor(SoftTheme.inkSoft)
            }
            ForEach(cells.indices, id: \.self) { i in
                if let day = cells[i], let date = cal.date(byAdding: .day, value: day - 1, to: monthStart) {
                    let mins = date <= Date() ? store.minutes(on: date) : 0
                    RoundedRectangle(cornerRadius: 5)
                        .fill(SoftTheme.coral.opacity(intensity(stepFor(minutes: mins))))
                        .overlay(RoundedRectangle(cornerRadius: 5)
                            .stroke(cal.isDateInToday(date) ? SoftTheme.coral : .clear, lineWidth: 1.6))
                        .frame(height: 22)
                } else {
                    Color.clear.frame(height: 22)
                }
            }
        }
    }
}
```

Body balance card:

```swift
struct BodyBalanceCard: View {
    @EnvironmentObject var store: StretchStore

    var body: some View {
        SoftCard {
            VStack(alignment: .leading, spacing: 12) {
                Text("Body balance")
                    .font(SoftTheme.display(17)).foregroundColor(SoftTheme.ink)
                let balance = store.areaBalance()
                if balance.allSatisfy({ $0.fraction == 0 }) {
                    Text("Finish a session and Buddy will chart which areas you care for most.")
                        .font(SoftTheme.body(13)).foregroundColor(SoftTheme.inkSoft)
                } else {
                    ForEach(balance, id: \.area) { item in
                        HStack(spacing: 10) {
                            Text(item.area.title)
                                .font(SoftTheme.body(12, .semibold))
                                .foregroundColor(SoftTheme.inkSoft)
                                .frame(width: 118, alignment: .leading)
                            GeometryReader { g in
                                ZStack(alignment: .leading) {
                                    Capsule().fill(item.area.tint.opacity(0.14))
                                    Capsule().fill(item.area.tint)
                                        .frame(width: max(g.size.width * item.fraction, item.fraction > 0 ? 6 : 0))
                                }
                            }
                            .frame(height: 10)
                            Text("\(Int((item.fraction * 100).rounded()))%")
                                .font(SoftTheme.body(11, .bold))
                                .foregroundColor(SoftTheme.ink.opacity(0.6))
                                .frame(width: 34, alignment: .trailing)
                        }
                    }
                }
            }
        }
    }
}
```

(Note: `ForEach(balance, id: \.area)` needs `BodyArea: Hashable` — it is a String-raw enum, already Hashable.)

- [ ] **Step 4: Build; commit** — `feat: progress upgrades - friendship card, month heatmap, body balance`.

---

### Task 9: Style pass

**Files:**
- Modify: `Soft Stretch/StretchTheme.swift` (softAppear modifier)
- Modify: `Soft Stretch/RootView.swift` (tab bar active pill)
- Modify: `Soft Stretch/OnboardingView.swift` (copy refresh + animated dots)
- Modify: `Soft Stretch/HomeView.swift`, `RoutinesView.swift`, `LibraryView.swift`, `ProgressTabView.swift` (apply softAppear to top-level sections)

**Interfaces:**
- Produces: `extension View { func softAppear(delay: Double) -> some View }`.

- [ ] **Step 1: softAppear** (append to StretchTheme.swift):

```swift
// Gentle staggered entrance for list sections.
private struct SoftAppear: ViewModifier {
    let delay: Double
    @State private var shown = false

    func body(content: Content) -> some View {
        content
            .opacity(shown ? 1 : 0)
            .offset(y: shown ? 0 : 14)
            .onAppear {
                withAnimation(.spring(response: 0.5, dampingFraction: 0.85).delay(delay)) {
                    shown = true
                }
            }
    }
}

extension View {
    func softAppear(delay: Double = 0) -> some View { modifier(SoftAppear(delay: delay)) }
}
```

Apply on Home (`header.softAppear()`, `nookCard.softAppear(delay: 0.05)`, `todaysPickCard.softAppear(delay: 0.1)`, stats row `0.15`, quickRow `0.2`), Routines segments content (`0.05`), Library header/chips/grid (`0, 0.05, 0.1`), Progress cards (`0…0.25` stepped).

- [ ] **Step 2: Tab bar pill.** In `RootView.tabButton`, wrap the VStack in a ZStack with, when active, a `Capsule().fill(SoftTheme.coral.opacity(0.12)).frame(height: 46)` behind icon+label, `.padding(.horizontal, 2)`; animate with `.animation(.spring(response: 0.3, dampingFraction: 0.8), value: selectedTab)`.

- [ ] **Step 3: Onboarding.** Page 3 copy becomes: title "Grow a friendship" body "Every session earns friendship xp - Buddy unlocks new looks and fills the nook you share with keepsakes." (check `OnboardingView.swift` structure: it has a pages array — update the third entry's title/body strings; art stays `onb_daily`). Dots: replace static circles with capsules — active dot animates to width 18 (`RoundedRectangle(cornerRadius: 4)` width `page == i ? 18 : 8`, `.animation(.spring(...), value: page)`).

- [ ] **Step 4: Build; commit** — `polish: entrance animations, tab pill, onboarding refresh`.

---

### Task 10: Verification, screenshots, re-delivery

**Files:**
- Modify: `Soft Stretch/RootView.swift` (temporary debug hook, removed again before delivery)
- Modify: `/Volumes/ADATA SE880/работа/development/APP_TRACKER.md`, `/Volumes/ADATA SE880/работа/APP_DESCRIPTIONS.md`
- Move: app folder → `/Volumes/ADATA SE880/работа/for_human_review_apps/Soft Stretch`

**Steps:**

- [ ] **Step 1: Full review pass.** Re-read every modified file top to bottom hunting for: iOS-16 API, SF Symbols (`grep -rn systemName *.swift` → empty), emoji (`grep -rnP "[\x{1F300}-\x{1FAFF}\x{2600}-\x{27BF}]" *.swift` → empty), retain cycles in closures capturing stores (all stores are environment objects — fine), the `explorer` badge fix (Task 6), `lastReward` cleared on player disappear (Task 7).

- [ ] **Step 2: Clean build both configs.** Debug (standard command) AND `-configuration Release` build → both `** BUILD SUCCEEDED **`, zero `error:`; note .app size via `du -sh build/Build/Products/Debug-iphonesimulator/Soft Stretch.app` → must stay ≥ 18 MB, < 99 MB (art unchanged → ~28 MB expected).

- [ ] **Step 3: Screenshot pass** (one pass, no test loops): re-add the Task-10 debug hook to RootView `.onAppear` (`SOFT_DEBUG_TAB` int → selectedTab; `SOFT_DEBUG_PLAY` routine id → activePlayer; NEW `SOFT_DEBUG_XP` int → `companion.state.xp = value` for studio/nook-rich shots), build, boot sim `iPhone 17`, seed onboarded settings JSON (same hex-defaults trick as v1 delivery), capture: home nook (with `SOFT_DEBUG_XP=140` → level 7, several nook items), Buddy Studio, Programs list, Program detail, builder sheet (manual tap not needed — screenshot the My Own empty state instead), progress (heatmap+balance), player finish is optional (needs a full session — skip). Copy to `screenshots/` replacing stale v1 shots where superseded (keep names: `home.png`, `studio.png`, `programs.png`, `program_detail.png`, `my_own.png`, `progress.png`, keep v1 `onboarding.png`, `player.png`, `routines.png`, `stretch_library.png`, `more.png`). Then REMOVE the debug hook, final rebuild, verify `** BUILD SUCCEEDED **`.

- [ ] **Step 4: Version guard.** `grep -E "MARKETING_VERSION|CURRENT_PROJECT_VERSION" "Soft Stretch.xcodeproj/project.pbxproj"` → `1.0` / `1` (×2 each).

- [ ] **Step 5: Commit final state** in the app repo: `Soft Stretch v2 - Buddy Companion Edition`.

- [ ] **Step 6: Re-deliver.** `find . -name "._*" -delete`; `rm -rf build`; `mv` the folder to `for_human_review_apps/`; verify no copy remains in `development/`; update `APP_TRACKER.md` row (append v2 summary before the v1 text, mirroring the Sound Grove `|||` pattern) and refresh the `APP_DESCRIPTIONS.md` row description to mention the companion loop, programs and builder; report to Master.

---

## Self-review notes

- Spec §1–§9 → Tasks 1–9 one-to-one; spec §10 out-of-scope respected (no audio/notifications/PNG).
- Spec said `areaMinutes` — plan uses `areaSeconds` (second precision is strictly better and converts to minutes in UI); spec updated to say `areaSeconds`.
- Type check: `BuddyOutfit` lives in CompanionStore.swift (Task 1) and is consumed by Tasks 2–7 under that exact name; `renderBuddy` extraction happens in Task 3 and is consumed by `AccessoryMini` in Task 4; `SoftSegmented` created in Task 5 before Task 6 uses it; `PlayerLaunch` created in Task 5; `programID`/`programDay` params consumed by Task 7's finish chip.
- Segments ordering: Task 5 ships 2 segments, Task 6 flips to 3 — no dangling references at any commit point.
