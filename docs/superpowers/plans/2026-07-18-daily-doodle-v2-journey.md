# Daily Doodle v2.0 Journey Edition — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn Daily Doodle from a lesson catalog into a guided sketchbook journey: a winding path through all 33 lessons with warmups, chapter stickers, animated stroke guides, artwork replay, and a deep hand-crafted visual polish pass.

**Architecture:** All new features derive from existing state (`completions`) plus three small new persisted pieces (warmup stars, earned stickers, per-artwork stroke JSON). New UI is 5 new Swift files; existing drawing engine (`DoodleCanvasModel`, `DrawingCanvasView`) is reused untouched by warmups and extended (not rewritten) for guide animation. Art generator gains one section producing stickers/tiles/ribbons/grain in the same deterministic style.

**Tech Stack:** SwiftUI (iOS 15.6 floor), Canvas + TimelineView(.animation), UIKit haptics, Core Graphics offline generator (`swiftc`).

## Global Constraints

- iOS 15.6+ only; no NavigationStack, no Charts, no iOS-16-only APIs.
- No SF Symbols, no emoji, custom Path icons only; English (US) only.
- Forced light appearance; theme-independent colors from `Doodle` palette.
- Local storage only (UserDefaults + files in Documents); no notifications.
- App icon stays as-is (opaque). Non-icon PNGs may use alpha.
- App size must stay ≥ 18 MB and < 99 MB.
- WebView gate (`ScribbleRedirectScout` etc.) must not be touched.
- Verification per task = `xcodebuild` compile (this project has no test target by design; the ios-builder delivery rules forbid extra targets). Pure-logic tasks additionally get a `swiftc` script check run once and deleted (not committed).
- Build command (run from repo root `development/Daily Doodle`):
  `xcodebuild -project "Daily Doodle.xcodeproj" -scheme "Daily Doodle" -destination 'id=2C2671A3-E3B2-4ED3-BF1F-31025525B4C1' -derivedDataPath build/ build`
- Commit after every task with a `feat:`/`chore:` message + Claude co-author line.

---

### Task 1: JourneyMap — curated path data + warmup drills + accuracy scoring

**Files:**
- Create: `Daily Doodle/JourneyMap.swift`
- Modify: `Daily Doodle.xcodeproj/project.pbxproj` (add file ref `DD0000000000000000000023`, build file `DD0000000000000000000123`, group + sources entries next to the existing ones)

**Interfaces:**
- Consumes: `SketchPath`, `SketchCmd`, path builders from `DoodlePathKit.swift`; `DoodleLibrary.lessons`.
- Produces: `struct WarmupDrill { let id, name, blurb: String; let guides: [SketchPath] }`, `struct JourneyChapter { let id, name, motto: String; let lessonIds: [String]; let warmup: WarmupDrill }`, `enum JourneyMap { static let chapters: [JourneyChapter]; static func samplePoints(_ paths: [SketchPath], step: CGFloat) -> [CGPoint]; static func warmupStars(strokes: [[CGPoint]], guides: [SketchPath]) -> Int }`.

- [ ] **Step 1: Write JourneyMap.swift**

Six chapters covering all 33 lesson ids exactly once, easy → tricky:

```swift
import Foundation
import CoreGraphics

struct WarmupDrill {
    let id: String
    let name: String
    let blurb: String
    let guides: [SketchPath]
}

struct JourneyChapter {
    let id: String
    let name: String
    let motto: String
    let lessonIds: [String]
    let warmup: WarmupDrill
}

enum JourneyMap {

    static let chapters: [JourneyChapter] = [
        JourneyChapter(id: "ch1", name: "First Lines", motto: "Every artist starts with a single stroke.",
            lessonIds: ["sun", "crown", "kite", "apple", "ladybug"],
            warmup: WarmupDrill(id: "w_lines", name: "Steady Lines",
                blurb: "Trace three straight lines without lifting your finger mid-stroke.",
                guides: [seg(0.15, 0.30, 0.85, 0.30), seg(0.15, 0.50, 0.85, 0.50), seg(0.15, 0.70, 0.85, 0.70)])),
        JourneyChapter(id: "ch2", name: "Garden Sketches", motto: "Round shapes grow into living things.",
            lessonIds: ["flower", "mushroom", "tree", "bunny", "jellyfish"],
            warmup: WarmupDrill(id: "w_circles", name: "Perfect Circles",
                blurb: "Three calm circles — small, medium, large. Slow is smooth.",
                guides: [cir(0.5, 0.5, 0.12), cir(0.5, 0.5, 0.21), cir(0.5, 0.5, 0.30)])),
        JourneyChapter(id: "ch3", name: "Sweet Break", motto: "Waves and curls make everything delicious.",
            lessonIds: ["donut", "ice_cream", "cupcake", "mug", "fish"],
            warmup: WarmupDrill(id: "w_waves", name: "Gentle Waves",
                blurb: "Ride two smooth waves from edge to edge.",
                guides: [
                    qchain((0.12, 0.40), [((0.245, 0.26), (0.37, 0.40)), ((0.495, 0.54), (0.62, 0.40)), ((0.745, 0.26), (0.87, 0.40))]),
                    qchain((0.12, 0.65), [((0.245, 0.51), (0.37, 0.65)), ((0.495, 0.79), (0.62, 0.65)), ((0.745, 0.51), (0.87, 0.65))])
                ])),
        JourneyChapter(id: "ch4", name: "Out to Sea", motto: "Rain or waves, your hand stays calm.",
            lessonIds: ["sailboat", "umbrella", "rain_cloud", "whale", "crab", "moon"],
            warmup: WarmupDrill(id: "w_loops", name: "Bouncy Loops",
                blurb: "Loop-de-loop across the page like a happy bee.",
                guides: [loopChain(y: 0.5, count: 4)])),
        JourneyChapter(id: "ch5", name: "Furry Friends", motto: "Ears, whiskers and wings — details with love.",
            lessonIds: ["cat_face", "house", "puppy", "owl", "bee", "butterfly"],
            warmup: WarmupDrill(id: "w_zigzag", name: "Zigzag Peaks",
                blurb: "Sharp mountain peaks — crisp turns, straight slopes.",
                guides: [pl([(0.10, 0.62), (0.235, 0.34), (0.37, 0.62), (0.505, 0.34), (0.64, 0.62), (0.775, 0.34), (0.90, 0.62)])])),
        JourneyChapter(id: "ch6", name: "Big Adventures", motto: "Rockets and robots — you are ready for anything.",
            lessonIds: ["rainbow", "car", "balloon", "cactus", "rocket", "robot"],
            warmup: WarmupDrill(id: "w_spiral", name: "Calm Spiral",
                blurb: "One long spiral from the outside in. Breathe out, draw slow.",
                guides: [spiral(cx: 0.5, cy: 0.5, turns: 3, rMax: 0.34)]))
    ]

    static var allWarmups: [WarmupDrill] { chapters.map { $0.warmup } }

    static func chapterIndex(containing lessonId: String) -> Int? {
        chapters.firstIndex { $0.lessonIds.contains(lessonId) }
    }

    // MARK: - Warmup helpers (pure geometry)

    /// Cursive loop chain across the page.
    static func loopChain(y: CGFloat, count: Int) -> SketchPath {
        var cmds: [SketchCmd] = [.move(0.10, y + 0.10)]
        let w = 0.80 / CGFloat(count)
        for i in 0..<count {
            let x0 = 0.10 + CGFloat(i) * w
            cmds.append(.curve(x0 + w * 0.55, y - 0.22, x0 - w * 0.25, y - 0.22, x0 + w * 0.15, y + 0.02))
            cmds.append(.curve(x0 + w * 0.45, y + 0.20, x0 + w * 0.85, y + 0.16, x0 + w, y + 0.10))
        }
        return SketchPath(cmds: cmds)
    }

    /// Inward archimedean spiral sampled as short segments.
    static func spiral(cx: CGFloat, cy: CGFloat, turns: CGFloat, rMax: CGFloat) -> SketchPath {
        var cmds: [SketchCmd] = []
        let steps = Int(turns * 90)
        for i in 0...steps {
            let t = CGFloat(i) / CGFloat(steps)
            let a = t * turns * 2 * .pi
            let r = rMax * (1 - t * 0.92)
            let p = (cx + r * cos(a), cy + r * sin(a))
            cmds.append(i == 0 ? .move(p.0, p.1) : .line(p.0, p.1))
        }
        return SketchPath(cmds: cmds)
    }

    // MARK: - Accuracy scoring

    /// Densely samples paths in the unit square (bezier subdivision).
    static func samplePoints(_ paths: [SketchPath], step: CGFloat = 0.01) -> [CGPoint] {
        var pts: [CGPoint] = []
        var cur = CGPoint.zero
        var start = CGPoint.zero
        func emitLine(_ a: CGPoint, _ b: CGPoint) {
            let d = hypot(b.x - a.x, b.y - a.y)
            let n = max(1, Int(d / step))
            for i in 0...n {
                let t = CGFloat(i) / CGFloat(n)
                pts.append(CGPoint(x: a.x + (b.x - a.x) * t, y: a.y + (b.y - a.y) * t))
            }
        }
        func emitQuad(_ a: CGPoint, _ c: CGPoint, _ b: CGPoint) {
            let n = 24
            var prev = a
            for i in 1...n {
                let t = CGFloat(i) / CGFloat(n)
                let mt = 1 - t
                let p = CGPoint(x: mt * mt * a.x + 2 * mt * t * c.x + t * t * b.x,
                                y: mt * mt * a.y + 2 * mt * t * c.y + t * t * b.y)
                emitLine(prev, p)
                prev = p
            }
        }
        func emitCubic(_ a: CGPoint, _ c1: CGPoint, _ c2: CGPoint, _ b: CGPoint) {
            let n = 30
            var prev = a
            for i in 1...n {
                let t = CGFloat(i) / CGFloat(n)
                let mt = 1 - t
                let p = CGPoint(
                    x: mt * mt * mt * a.x + 3 * mt * mt * t * c1.x + 3 * mt * t * t * c2.x + t * t * t * b.x,
                    y: mt * mt * mt * a.y + 3 * mt * mt * t * c1.y + 3 * mt * t * t * c2.y + t * t * t * b.y)
                emitLine(prev, p)
                prev = p
            }
        }
        for path in paths {
            for cmd in path.cmds {
                switch cmd {
                case .move(let x, let y): cur = CGPoint(x: x, y: y); start = cur; pts.append(cur)
                case .line(let x, let y): let p = CGPoint(x: x, y: y); emitLine(cur, p); cur = p
                case .quad(let cx, let cy, let x, let y):
                    emitQuad(cur, CGPoint(x: cx, y: cy), CGPoint(x: x, y: y)); cur = CGPoint(x: x, y: y)
                case .curve(let c1x, let c1y, let c2x, let c2y, let x, let y):
                    emitCubic(cur, CGPoint(x: c1x, y: c1y), CGPoint(x: c2x, y: c2y), CGPoint(x: x, y: y))
                    cur = CGPoint(x: x, y: y)
                case .close: emitLine(cur, start); cur = start
                }
            }
        }
        return pts
    }

    /// 1–3 stars from mean distance of user points to the nearest guide point.
    /// Returns 0 for an empty drawing.
    static func warmupStars(strokes: [[CGPoint]], guides: [SketchPath]) -> Int {
        let user = strokes.flatMap { $0 }
        guard !user.isEmpty else { return 0 }
        let target = samplePoints(guides)
        guard !target.isEmpty else { return 0 }
        var total: CGFloat = 0
        for p in user {
            var best = CGFloat.greatestFiniteMagnitude
            for t in target {
                let d = hypot(p.x - t.x, p.y - t.y)
                if d < best { best = d }
            }
            total += best
        }
        let mean = total / CGFloat(user.count)
        if mean < 0.020 { return 3 }
        if mean < 0.035 { return 2 }
        return 1
    }
}
```

- [ ] **Step 2: Verify logic with a throwaway swiftc script**

Write `/tmp`-scratchpad `journeycheck.swift` (NOT committed):

```swift
// swiftc "Daily Doodle/DoodlePathKit.swift" "Daily Doodle/LessonsAnimals.swift" \
//   "Daily Doodle/LessonsNature.swift" "Daily Doodle/LessonsFoodThings.swift" \
//   "Daily Doodle/DoodleLibrary.swift" "Daily Doodle/JourneyMap.swift" journeycheck.swift -o jc && ./jc
let ids = JourneyMap.chapters.flatMap { $0.lessonIds }
assert(ids.count == 33, "expected 33, got \(ids.count)")
assert(Set(ids).count == 33, "duplicate lesson ids on path")
for id in ids { assert(DoodleLibrary.lesson(id) != nil, "unknown id \(id)") }
let perfect = JourneyMap.samplePoints([seg(0.15, 0.30, 0.85, 0.30)]).map { [$0] }
// tracing exactly on the line → 3 stars
assert(JourneyMap.warmupStars(strokes: [JourneyMap.samplePoints([seg(0.15, 0.30, 0.85, 0.30)])],
                              guides: [seg(0.15, 0.30, 0.85, 0.30)]) == 3)
// far away scribble → 1 star
assert(JourneyMap.warmupStars(strokes: [[CGPoint(x: 0.1, y: 0.95)]],
                              guides: [seg(0.15, 0.30, 0.85, 0.30)]) == 1)
// empty → 0
assert(JourneyMap.warmupStars(strokes: [], guides: [seg(0, 0, 1, 1)]) == 0)
_ = perfect
print("journey map OK")
```

Run it; expected output: `journey map OK`. Delete the script and binary.

- [ ] **Step 3: Add JourneyMap.swift to pbxproj** (file ref + build file + group child + sources entry, same DD-hex id pattern as existing files).

- [ ] **Step 4: Build**

Run the Global-Constraints xcodebuild command. Expected: `** BUILD SUCCEEDED **`.

- [ ] **Step 5: Commit** — `feat: add journey map, warmup drills and accuracy scoring`

---

### Task 2: DoodleEffects — wobbly borders, confetti, haptics, paper texture, press style

**Files:**
- Create: `Daily Doodle/DoodleEffects.swift`
- Modify: `Daily Doodle.xcodeproj/project.pbxproj` (add refs like Task 1)

**Interfaces:**
- Produces:
  - `struct WobblyRoundedRect: Shape { init(cornerRadius: CGFloat, amplitude: CGFloat = 1.6, seed: Int = 1) }`
  - `struct ConfettiBurst: View { init(trigger: Int, accent: Color) }` — replays its ~1.6 s burst every time `trigger` increments; passthrough (no hit testing).
  - `enum Haptic { static func success(); static func light(); static func medium() }`
  - `extension View { func paperTextured() -> some View }` — grain overlay via `Art/paper_grain.png` (multiply, tiled); no-op if asset missing.
  - `struct SpringyButtonStyle: ButtonStyle`
  - `func stableTilt(_ id: String, maxDegrees: Double) -> Double` — deterministic per-id tilt.

- [ ] **Step 1: Write DoodleEffects.swift**

```swift
import SwiftUI

// Shared "hand-made sketchbook" effects: wobbly borders, confetti,
// haptics, paper grain and springy buttons.

struct WobblyRoundedRect: Shape {
    var cornerRadius: CGFloat
    var amplitude: CGFloat = 1.6
    var seed: Int = 1

    func path(in rect: CGRect) -> Path {
        let base = Path(roundedRect: rect, cornerRadius: cornerRadius, style: .continuous)
        var out = Path()
        var lastEmit: CGPoint?
        let phase = CGFloat(seed % 17)
        var idx: CGFloat = 0
        // Sample the rounded rect outline and jitter it with layered sines.
        let samples = 140
        for i in 0...samples {
            let t = CGFloat(i) / CGFloat(samples)
            guard let p = base.trimmedPath(from: 0, to: max(t, 0.0001)).currentPoint else { continue }
            idx += 1
            let wob = sin(idx * 0.9 + phase) * 0.6 + sin(idx * 0.23 + phase * 1.7) * 0.4
            let q = CGPoint(x: p.x + wob * amplitude, y: p.y + cos(idx * 0.7 + phase) * amplitude * 0.8)
            if lastEmit == nil { out.move(to: q) } else { out.addLine(to: q) }
            lastEmit = q
        }
        out.closeSubpath()
        return out
    }
}

func stableTilt(_ id: String, maxDegrees: Double) -> Double {
    var h = 0
    for u in id.unicodeScalars { h = (h &* 31 &+ Int(u.value)) & 0xFFFF }
    return (Double(h) / Double(0xFFFF) * 2 - 1) * maxDegrees
}

// MARK: - Haptics

enum Haptic {
    static func success() { UINotificationFeedbackGenerator().notificationOccurred(.success) }
    static func light() { UIImpactFeedbackGenerator(style: .light).impactOccurred() }
    static func medium() { UIImpactFeedbackGenerator(style: .medium).impactOccurred() }
}

// MARK: - Confetti

struct ConfettiBurst: View {
    let trigger: Int
    var accent: Color = Doodle.coral
    @State private var startDate: Date?

    private struct Piece {
        let angle: Double; let speed: Double; let spin: Double
        let size: Double; let colorIndex: Int; let drift: Double
    }

    private func pieces(seed: Int) -> [Piece] {
        var h = UInt64(truncatingIfNeeded: seed &* 2654435761)
        func rnd() -> Double {
            h ^= h << 13; h ^= h >> 7; h ^= h << 17
            return Double(h % 10_000) / 10_000
        }
        return (0..<26).map { _ in
            Piece(angle: rnd() * 2 * .pi, speed: 0.45 + rnd() * 0.85, spin: rnd() * 8 - 4,
                  size: 5 + rnd() * 7, colorIndex: Int(rnd() * 7.99), drift: rnd() * 0.5 - 0.25)
        }
    }

    var body: some View {
        Group {
            if trigger > 0 {
                TimelineView(.animation) { tl in
                    Canvas { ctx, size in
                        guard let start = startDate else { return }
                        let t = tl.date.timeIntervalSince(start)
                        guard t >= 0 && t < 1.6 else { return }
                        let progress = t / 1.6
                        let c = CGPoint(x: size.width / 2, y: size.height * 0.38)
                        for p in pieces(seed: trigger) {
                            let dist = p.speed * progress * Double(min(size.width, size.height)) * 0.85
                            let x = c.x + CGFloat(cos(p.angle) * dist + p.drift * t * 120)
                            let y = c.y + CGFloat(sin(p.angle) * dist + 190 * t * t)
                            let alpha = 1 - progress
                            var rect = ctx
                            rect.translateBy(x: x, y: y)
                            rect.rotate(by: .radians(p.spin * t))
                            let color = Doodle.inkPalette[p.colorIndex].opacity(alpha)
                            rect.fill(Path(CGRect(x: -p.size / 2, y: -p.size / 3, width: p.size, height: p.size * 0.66)),
                                      with: .color(color))
                        }
                    }
                }
                .allowsHitTesting(false)
            }
        }
        .onChange(of: trigger) { _ in startDate = Date() }
    }
}

// MARK: - Paper texture

struct PaperTexture: ViewModifier {
    func body(content: Content) -> some View {
        content.overlay(
            Group {
                if let img = DoodleArtStore.image("paper_grain") {
                    Image(uiImage: img)
                        .resizable(resizingMode: .tile)
                        .blendMode(.multiply)
                        .opacity(0.55)
                        .ignoresSafeArea()
                        .allowsHitTesting(false)
                }
            }
        )
    }
}

extension View {
    func paperTextured() -> some View { modifier(PaperTexture()) }
}

// MARK: - Springy buttons

struct SpringyButtonStyle: ButtonStyle {
    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .scaleEffect(configuration.isPressed ? 0.96 : 1)
            .animation(.spring(response: 0.25, dampingFraction: 0.6), value: configuration.isPressed)
    }
}
```

- [ ] **Step 2: pbxproj entry, build** — expected `** BUILD SUCCEEDED **`.
- [ ] **Step 3: Commit** — `feat: add sketchbook effects kit (wobble, confetti, haptics, grain, springy buttons)`

---

### Task 3: DoodleStore v2 — warmups, stickers, stroke JSON, 14 badges

**Files:**
- Modify: `Daily Doodle/DoodleStore.swift`, `Daily Doodle/DoodleModels.swift`, `Daily Doodle/LessonPlayerView.swift` (call-site signature), `Daily Doodle/FreeDrawView.swift` (call-site signature)

**Interfaces:**
- Produces on `DoodleStore`:
  - `@Published var warmupStars: [String: Int]`, `@Published var earnedStickers: [String]`
  - `func recordWarmup(_ drillId: String, stars: Int)` (keeps best)
  - `func completedChapters() -> [JourneyChapter]` (all lessons done)
  - `func claimNewStickers() -> [JourneyChapter]` — chapters complete but sticker not yet earned; marks them earned, saves, returns them (empty when none).
  - `completeLesson(_:image:strokes:)` and `completeFreeDraw(image:strokes:)` — same as v1 plus `strokes: [DoodleStroke]`, which are serialized to `Artworks/<id>.strokes.json`.
  - `func artworkStrokes(_ meta: ArtworkMeta) -> [DoodleStroke]?` — nil when JSON absent (pre-v2 art).
- New Codable DTO in `DoodleModels.swift`:

```swift
struct StrokeDTO: Codable {
    let pts: [[CGFloat]]        // [x, y] pairs, normalized
    let color: Int
    let width: CGFloat
    let eraser: Bool

    init(_ s: DoodleStroke) {
        pts = s.points.map { [$0.x, $0.y] }
        color = s.colorIndex; width = s.width; eraser = s.isEraser
    }
    var stroke: DoodleStroke {
        DoodleStroke(points: pts.map { CGPoint(x: $0[0], y: $0[1]) },
                     colorIndex: min(max(color, 0), Doodle.inkPalette.count - 1),
                     width: width, isEraser: eraser)
    }
}
```

- [ ] **Step 1: Store changes** — new keys `dd.warmupStars` (JSON dict), `dd.earnedStickers` (JSON array), loaded in `init`, written in `save()`; delete `<id>.strokes.json` inside `deleteArtwork` and `resetAllProgress` (reset also clears warmups/stickers). `saveArtwork` gains `strokes:` param and writes the JSON via `JSONEncoder` on `[StrokeDTO]`.
- [ ] **Step 2: Badges** — extend `BadgeCatalog.all` with:

```swift
DoodleBadge(id: "limber_wrist", name: "Limber Wrist", detail: "Finish every warm-up drill on the journey.", goal: 6,
            progress: { s in JourneyMap.allWarmups.filter { (s.warmupStars[$0.id] ?? 0) > 0 }.count }),
DoodleBadge(id: "sticker_collector", name: "Sticker Collector", detail: "Earn all six chapter stickers.", goal: 6,
            progress: { $0.earnedStickers.count })
```

- [ ] **Step 3: Update the two call sites** to pass `strokes: canvas.strokes`.
- [ ] **Step 4: Build** — expected `** BUILD SUCCEEDED **`.
- [ ] **Step 5: Commit** — `feat: persist warmups, chapter stickers and artwork stroke data; 14 badges`

---

### Task 4: Art generator — stickers, warmup tiles, ribbons, paper grain

**Files:**
- Modify: `art_src/main.swift` (new section before the Run block), plus a temporary copy of `JourneyMap.swift` is already compiled in (add it to the swiftc invocation).

**Interfaces:**
- Produces PNGs in `Daily Doodle/Art/`: `sticker_ch1…ch6` (512², alpha-enabled round stickers: white die-cut ring, chapter-pastel fill, chapter's first lesson doodle inked + 3 tiny stars), `warmup_w_lines… w_spiral` (512² opaque tiles: paper card + drill guides in accent), `ribbon_ch1…ch6` (1024×300 opaque pastel ribbons with ghost doodles), `paper_grain` (256² opaque near-white noise tile).
- Sticker context uses `CGImageAlphaInfo.premultipliedLast` (alpha allowed for non-icon art); everything else stays `noneSkipLast`.

- [ ] **Step 1: Extend generator** — chapter → style mapping reuses `styles[...]` by the category of the chapter's first lesson; `makeAlphaContext(_:_:)` helper mirrors `makeContext` with premultipliedLast.
- [ ] **Step 2: Recompile with JourneyMap included**:

`swiftc -O "Daily Doodle/DoodlePathKit.swift" "Daily Doodle/LessonsAnimals.swift" "Daily Doodle/LessonsNature.swift" "Daily Doodle/LessonsFoodThings.swift" "Daily Doodle/DoodleLibrary.swift" "Daily Doodle/JourneyMap.swift" art_src/main.swift -o art_src/doodlegen && ./art_src/doodlegen "Daily Doodle/Art"`

Expected: 19 new PNGs (6+6+6+1), existing 44 regenerated identically (same seed).
- [ ] **Step 3: Visual check** — contact sheet of the 19 new files via the scratchpad `sheet` tool; inspect; fix geometry if any sticker/tile is off.
- [ ] **Step 4: Delete `art_src/doodlegen` binary; build app** (Art folder reference picks new files automatically). Expected `** BUILD SUCCEEDED **`.
- [ ] **Step 5: Commit** — `feat: generate journey art (stickers, warmup tiles, ribbons, paper grain)`

---

### Task 5: Animated stroke guides + celebratory finish in the player

**Files:**
- Modify: `Daily Doodle/DrawingKit.swift` (`DrawingCanvasView` gains `guidePhase: Double?`), `Daily Doodle/LessonPlayerView.swift`

**Interfaces:**
- `DrawingCanvasView` new optional param `guidePhase: Double? = nil`. When non-nil, the current step's guide paths animate: paths draw sequentially on a shared clock (1.3 s draw per path + 0.5 s hold after the last), the active path stroked with `guideColor.opacity(min(0.9, guideOpacity + 0.35))` via `Path(cg).trimmedPath(from: 0, to: f)`, finished-this-cycle paths shown at the same boosted opacity, and a pen dot (radius `s*0.014`, accent) rides at `trimmedPath(...).currentPoint`. Static guide underneath unchanged.
- Player wraps the canvas in `TimelineView(.animation)` only when `animateGuide && showGuide && finished == nil`; the toggle button (custom play/pause glyph added inline as `DoodleIcon.playPause(_:_:playing:)` in `DoodleIcons.swift`) sits beside the eye button; `@State var animateGuide = true`.
- Finish: `finishLesson()` calls `Haptic.success()` and increments `@State confettiTrigger` consumed by a full-screen `ConfettiBurst(trigger:accent:)` in the outer ZStack; step advance calls `Haptic.light()`.

- [ ] **Step 1: Implement canvas animation** (guard `guidePhase != nil`, compute cycle from `guidePhase!.truncatingRemainder`), **Step 2: player wiring + toggle icon**, **Step 3: build** (`** BUILD SUCCEEDED **`), **Step 4: commit** — `feat: animated stroke guides with pen dot, confetti and haptics on finish`.

---

### Task 6: WarmupPlayerView

**Files:**
- Create: `Daily Doodle/WarmupPlayerView.swift`; pbxproj entry.

**Interfaces:**
- `struct WarmupPlayerView: View { let drill: WarmupDrill; ... }` presented via `fullScreenCover`. Reuses `DoodleCanvasModel` + `DrawingCanvasView` (guides = `drill.guides`, fixed charcoal ink, medium brush, no eraser/palette — warmups are about line control, so tools are deliberately minimal: only undo + clear + guide always on at 0.45 opacity).
- "Check my lines" button → `JourneyMap.warmupStars(strokes: canvas.strokes.filter { !$0.isEraser }.map { $0.points }, guides: drill.guides)`; result overlay shows 1–3 large `DoodleIcon.starIcon` (filled per star, springy scale-in animation), encouraging copy per score (3: "Silky smooth!", 2: "Great control — one more pass?", 1: "Warmed up! Try tracing a little closer."), `store.recordWarmup`, `Haptic.success()` + `ConfettiBurst` when 3 stars, and Retry / Done buttons. Empty canvas keeps the check button disabled.

- [ ] Steps: implement, pbxproj, build (`** BUILD SUCCEEDED **`), commit — `feat: warmup drill player with star accuracy scoring`.

---

### Task 7: JourneyView + 5-tab RootView

**Files:**
- Create: `Daily Doodle/JourneyView.swift`; pbxproj entry.
- Modify: `Daily Doodle/RootView.swift` (insert Journey as tab index 1, shift others), `Daily Doodle/DoodleIcons.swift` (add `DoodleIcon.journey` — winding path with 3 node dots).

**Interfaces:**
- `JourneyView` structure: `ScrollView` of chapter sections. Each section = ribbon header card (`ribbon_<id>` art, wobbly border, name + motto + `done/total` chip) → warmup node row → lesson node rows laid on an alternating left/right serpentine: node x-offset cycles `[-1, 0, 1, 0]` * 27% width; consecutive nodes joined by a dashed quad-curve connector drawn in a background `Canvas` per section (connector points computed from fixed row height 92).
- Node states from `store`: done (solid accent ring + mini `LessonCoverView` clipped in circle + small check), current (first incomplete on the whole path; pulsing halo via `scaleEffect` on repeatForever animation), upcoming (dashed `Circle().stroke(style: StrokeStyle(dash:))`, faint `LessonSketchView` silhouette). Warmup node: square-ish tile with `warmup_<id>` art + earned stars row.
- Taps: lesson node → `fullScreenCover(LessonPlayerView)`; warmup node → `fullScreenCover(WarmupPlayerView)`.
- Sticker awards: `.onAppear` and on each fullScreenCover dismiss call `store.claimNewStickers()`; non-empty → sticker celebration overlay (sticker art scale-in + `ConfettiBurst` + `Haptic.success()` + "Chapter complete!" + Done). Multiple stickers queue one at a time.
- Header: "Your Journey" title + overall progress capsule ("12 of 33 doodles").

- [ ] Steps: implement JourneyView, journey icon, RootView 5 tabs (labels: Today, Journey, Lessons, Gallery, Studio), pbxproj, build (`** BUILD SUCCEEDED **`), commit — `feat: journey path tab with chapters, warmup nodes and sticker rewards`.

---

### Task 8: Artwork replay

**Files:**
- Create: `Daily Doodle/ReplayView.swift`; pbxproj entry.
- Modify: `Daily Doodle/GalleryView.swift` (`ArtworkViewer`).

**Interfaces:**
- `struct ReplayCanvas: View { let strokes: [DoodleStroke] }` — TimelineView(.animation) + Canvas on paper card. Cumulative point budget: `total = strokes.map(\.points.count).reduce(0,+)`; duration = `min(6, max(2.5, Double(total) / 220))`; at time t draws whole strokes while budget lasts and a `prefix` of the boundary stroke (eraser strokes replay too — same layer/clear logic as `DrawingCanvasView`). Loops with a 1.2 s hold at the end.
- `ArtworkViewer`: when `store.artworkStrokes(meta)` is non-nil shows a "Watch it draw" / "Show picture" toggle button (accent capsule, brush icon) switching between the static image and `ReplayCanvas`. Hidden for pre-v2 artworks.

- [ ] Steps: implement, wire viewer, pbxproj, build (`** BUILD SUCCEEDED **`), commit — `feat: artwork replay in gallery viewer`.

---

### Task 9: Visual polish pass (narrative-coherent, no new features)

**Files:**
- Modify: `TodayView.swift`, `GalleryView.swift`, `StudioView.swift`, `RootView.swift`, `OnboardingView.swift`, `LessonPlayerView.swift` (cosmetic only)

Changes (each small and checked in the sim screenshot at Task 10):
- [ ] Today: header becomes a hero card — `hero_today` art backdrop (height 150, rounded 26, wobbly border stroke), greeting/date/streak overlaid on a soft cream wash for readability; flame pulses (`repeatForever` scale 1↔1.12) when streak > 0; daily card keeps layout but gains a washi-tape strip (small tinted rounded rect rotated 8°) over its top edge.
- [ ] Root: `.paperTextured()` applied once around the tab content ZStack (single grain pass over every screen; canvases keep their own clean paper).
- [ ] Gallery: each grid card gets `rotationEffect(.degrees(stableTilt(meta.id, maxDegrees: 2.6)))` + a tape strip on top; section subtitle becomes "Your art wall".
- [ ] Studio: sticker shelf card between level card and stats — `LazyVGrid` 3 columns; earned → `sticker_<id>` art (springy), unearned → dashed circle + chapter name in soft ink; plus journey progress line in the level card ("Chapter 3 of 6 on your journey").
- [ ] All primary action buttons (`Start`, `Next Step`, `Finish`, `Save to Gallery`, onboarding CTA, tab buttons) get `.buttonStyle(SpringyButtonStyle())`; tool selects in the player call `Haptic.light()`.
- [ ] Build (`** BUILD SUCCEEDED **`), commit — `feat: sketchbook polish pass (hero header, art wall, sticker shelf, grain, springy buttons)`.

---

### Task 10: Verify, review, redeliver

- [ ] Sim Debug + device Release builds clean (0 real warnings); Release `.app` size ≥ 18 MB, < 99 MB.
- [ ] Sim smoke: launch → Journey tab render, warmup open, replay hidden on old art; screenshots of Today/Journey/warmup into `screenshots/` (replace stale ones).
- [ ] Code review vs delivery checklist (no SF Symbols/emoji, orientations, iPad-safe flexible layouts, no overlaps, English only).
- [ ] Bump `MARKETING_VERSION` to 2.0, `CURRENT_PROJECT_VERSION` to 2 (both Debug + Release).
- [ ] Update `APP_TRACKER.md` row and `APP_DESCRIPTIONS.md` entry to v2.0 Journey Edition.
- [ ] Commit `chore: release v2.0 Journey Edition`, `mv` the folder back to `for_human_review_apps/`, report.

## Self-review

- Spec coverage: journey tab (T7), warmups+scoring (T1, T6), stickers (T3, T4, T7, T9), animated guides (T5), replay (T3, T8), polish list (T2, T9), new art (T4), badges 14 (T3), 5 tabs (T7), verification (T10). No gaps.
- Placeholders: none — every step names exact behavior, code given for all non-trivial algorithms.
- Type consistency: `warmupStars(strokes:guides:)` (T1) matches T6 call; `claimNewStickers()` (T3) matches T7; `guidePhase` (T5) matches player wiring; `artworkStrokes(_:)` (T3) matches T8; `stableTilt` (T2) matches T9.
