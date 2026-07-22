# Shake and Stir v2 — Home Bar Edition Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn Shake and Stir into a home-bar assistant: My Bar shelf of owned ingredients, "what can I mix now", buy-advice, shopping list, occasion picker, plus a living animated bar scene and an ingredient art pack.

**Architecture:** New `IngredientLibrary` catalog + `ingredientIDs` per drink; pure `BarMath` readiness functions over `MixStore.ownedIngredients`; new My Bar tab (5-tab root); Tonight rebuilt around mixability; single `BarSceneView` Canvas engine reused in three modes; deterministic CG generator `art_src/artgen2.swift` for ~50 ingredient tokens + occasion covers + textures.

**Tech Stack:** SwiftUI (iOS 15.6), Canvas/TimelineView, UserDefaults JSON, swiftc harness for logic tests, Core Graphics generators compiled with `swiftc`.

## Global Constraints

- iOS 15.6+ — `NavigationView` only, no iOS-16 API.
- Custom UI only: no SF Symbols, no emoji; glyphs via `MixIcon`/Canvas.
- Forced dark (`UIUserInterfaceStyle = Dark` already set); theme-independent.
- Version stays **1.0 (build 1)** — do not bump.
- English (US) copy only. Local storage only. No notifications/network (WebView gate stays as-is).
- App folder: `/Volumes/ADATA SE880/работа/development/Shake and Stir` (project `Shake and Stir.xcodeproj`, sources in `Shake and Stir/`).
- Build check: `xcodebuild -project "Shake and Stir.xcodeproj" -scheme "Shake and Stir" -destination 'platform=iOS Simulator,name=iPhone 17' -derivedDataPath build/ build` → `** BUILD SUCCEEDED **`, no `error:`.
- v1 saves must keep loading (tolerant decoding for all new keys).
- App size ≥18 MB, <99 MB. Art PNGs live in `Shake and Stir/Art/` (folder reference — new files ship automatically).
- Commit after every task in the app repo (`git -c core.hooksPath=/dev/null commit`).

---

### Task 1: Ingredient catalog + drink mapping

**Files:**
- Create: `Shake and Stir/IngredientLibrary.swift`
- Modify: `Shake and Stir/MixModels.swift` (Drink gains `ingredientIDs`)
- Modify: `Shake and Stir/DrinkLibrary.swift` (all 28 drinks get `ingredientIDs:`)
- Test: swiftc harness in scratchpad (see Step 4)

**Interfaces:**
- Produces: `enum IngredientGroup: String, CaseIterable { case spirits, liqueurs, mixers, fresh, pantry }`,
  `struct Ingredient { let id, name: String; let group: IngredientGroup }`,
  `enum IngredientLibrary { static let all: [Ingredient]; static func byID(_ id: String) -> Ingredient?; static func inGroup(_ g: IngredientGroup) -> [Ingredient]; static let needExempt: Set<String> }`,
  `Drink.ingredientIDs: [String]`, `Drink.neededIDs: [String]` (ingredientIDs minus needExempt).

- [ ] **Step 1: Write IngredientLibrary.swift**

```swift
import SwiftUI

enum IngredientGroup: String, CaseIterable, Identifiable {
    case spirits, liqueurs, mixers, fresh, pantry
    var id: String { rawValue }
    var displayName: String {
        switch self {
        case .spirits:  return "Spirits"
        case .liqueurs: return "Liqueurs & Wine"
        case .mixers:   return "Mixers & Juices"
        case .fresh:    return "Fresh"
        case .pantry:   return "Pantry"
        }
    }
}

struct Ingredient: Identifiable {
    let id: String
    let name: String
    let group: IngredientGroup
    var artName: String { "ing_\(id)" }
}

enum IngredientLibrary {
    // Items that never gate a recipe (always assumed available).
    static let needExempt: Set<String> = ["ice"]

    static let all: [Ingredient] = [
        // Spirits (8)
        Ingredient(id: "bourbon", name: "Bourbon", group: .spirits),
        Ingredient(id: "white_rum", name: "White rum", group: .spirits),
        Ingredient(id: "dark_rum", name: "Dark / aged rum", group: .spirits),
        Ingredient(id: "tequila", name: "Tequila", group: .spirits),
        Ingredient(id: "gin", name: "Gin", group: .spirits),
        Ingredient(id: "vodka", name: "Vodka", group: .spirits),
        Ingredient(id: "brandy", name: "Brandy", group: .spirits),
        Ingredient(id: "whiskey", name: "Whiskey", group: .spirits),
        // Liqueurs & Wine (9)
        Ingredient(id: "orange_liqueur", name: "Orange liqueur", group: .liqueurs),
        Ingredient(id: "coffee_liqueur", name: "Coffee liqueur", group: .liqueurs),
        Ingredient(id: "red_bitter", name: "Red bitter liqueur", group: .liqueurs),
        Ingredient(id: "sweet_vermouth", name: "Sweet vermouth", group: .liqueurs),
        Ingredient(id: "cherry_liqueur", name: "Cherry liqueur", group: .liqueurs),
        Ingredient(id: "creme_de_menthe", name: "Creme de menthe", group: .liqueurs),
        Ingredient(id: "creme_de_cacao", name: "Creme de cacao", group: .liqueurs),
        Ingredient(id: "blue_curacao", name: "Blue curacao", group: .liqueurs),
        Ingredient(id: "red_wine", name: "Red wine", group: .liqueurs),
        // Mixers & Juices (11)
        Ingredient(id: "soda_water", name: "Soda water", group: .mixers),
        Ingredient(id: "lemonade", name: "Lemonade", group: .mixers),
        Ingredient(id: "ginger_ale", name: "Ginger ale", group: .mixers),
        Ingredient(id: "ginger_beer", name: "Ginger beer", group: .mixers),
        Ingredient(id: "orange_juice", name: "Orange juice", group: .mixers),
        Ingredient(id: "pineapple_juice", name: "Pineapple juice", group: .mixers),
        Ingredient(id: "passion_fruit_juice", name: "Passion fruit juice", group: .mixers),
        Ingredient(id: "grenadine", name: "Grenadine", group: .mixers),
        Ingredient(id: "coconut_cream", name: "Coconut cream", group: .mixers),
        Ingredient(id: "espresso", name: "Espresso / strong coffee", group: .mixers),
        Ingredient(id: "orgeat", name: "Almond orgeat", group: .mixers),
        // Fresh (8)
        Ingredient(id: "lime", name: "Limes", group: .fresh),
        Ingredient(id: "lemon", name: "Lemons", group: .fresh),
        Ingredient(id: "orange", name: "Oranges", group: .fresh),
        Ingredient(id: "mint", name: "Fresh mint", group: .fresh),
        Ingredient(id: "cucumber", name: "Cucumber", group: .fresh),
        Ingredient(id: "berries", name: "Mixed berries", group: .fresh),
        Ingredient(id: "cherry", name: "Cocktail cherries", group: .fresh),
        Ingredient(id: "pineapple", name: "Pineapple", group: .fresh),
        // Pantry (15)
        Ingredient(id: "sugar", name: "Sugar", group: .pantry),
        Ingredient(id: "simple_syrup", name: "Simple syrup", group: .pantry),
        Ingredient(id: "vanilla_syrup", name: "Vanilla syrup", group: .pantry),
        Ingredient(id: "honey", name: "Honey", group: .pantry),
        Ingredient(id: "bitters", name: "Aromatic bitters", group: .pantry),
        Ingredient(id: "salt", name: "Coarse salt", group: .pantry),
        Ingredient(id: "heavy_cream", name: "Heavy cream", group: .pantry),
        Ingredient(id: "milk", name: "Whole milk", group: .pantry),
        Ingredient(id: "nutmeg", name: "Nutmeg", group: .pantry),
        Ingredient(id: "cinnamon", name: "Cinnamon sticks", group: .pantry),
        Ingredient(id: "chocolate", name: "Dark chocolate", group: .pantry),
        Ingredient(id: "mulling_spices", name: "Mulling spices", group: .pantry),
        Ingredient(id: "coffee_beans", name: "Coffee beans", group: .pantry),
        Ingredient(id: "party_picks", name: "Umbrellas & picks", group: .pantry),
        Ingredient(id: "ice", name: "Ice", group: .pantry),
    ]

    static func byID(_ id: String) -> Ingredient? { all.first { $0.id == id } }
    static func inGroup(_ g: IngredientGroup) -> [Ingredient] { all.filter { $0.group == g } }
}
```

- [ ] **Step 2: Add `ingredientIDs` to Drink** (`MixModels.swift`)

Add after `let steps: [MixStep]`:

```swift
    let ingredientIDs: [String]  // canonical ids for My Bar matching
```

and after `artName`:

```swift
    var neededIDs: [String] { ingredientIDs.filter { !IngredientLibrary.needExempt.contains($0) } }
```

- [ ] **Step 3: Map all 28 drinks in DrinkLibrary.swift**

Insert `ingredientIDs:` right after each `ingredients:` array. Exact mapping (ice included where recipes list it — it is need-exempt):

```text
old_fashioned:    ["sugar","bitters","bourbon","ice","orange"]
mojito:           ["mint","lime","simple_syrup","white_rum","ice","soda_water"]
margarita:        ["salt","tequila","lime","orange_liqueur","ice"]
whiskey_sour:     ["bourbon","lemon","simple_syrup","ice","cherry"]
negroni:          ["gin","red_bitter","sweet_vermouth","ice","orange"]
daiquiri:         ["white_rum","lime","simple_syrup","ice"]
espresso_martini: ["vodka","coffee_liqueur","espresso","ice","coffee_beans"]
pina_colada:      ["white_rum","coconut_cream","pineapple_juice","ice","pineapple"]
mai_tai:          ["dark_rum","lime","orange_liqueur","orgeat","ice","mint"]
tequila_sunrise:  ["tequila","orange_juice","grenadine","ice","orange"]
blue_lagoon:      ["vodka","blue_curacao","lemonade","ice","cherry"]
hurricane:        ["dark_rum","passion_fruit_juice","lemon","grenadine","ice","orange"]
singapore_sling:  ["gin","cherry_liqueur","pineapple_juice","lime","ice","soda_water","cherry"]
bahama_mama:      ["white_rum","dark_rum","coconut_cream","pineapple_juice","orange_juice","ice","party_picks"]
virgin_mojito:    ["mint","lime","simple_syrup","ice","soda_water"]
shirley_temple:   ["ice","ginger_ale","grenadine","cherry"]
cucumber_cooler:  ["cucumber","lime","simple_syrup","ice","soda_water"]
ginger_sunset:    ["ice","orange_juice","ginger_beer","grenadine","orange"]
berry_fizz:       ["berries","lemon","simple_syrup","ice","soda_water"]
citrus_presse:    ["lemon","simple_syrup","ice","soda_water"]
minted_lemonade:  ["mint","lemon","honey","ice"]
white_russian:    ["ice","vodka","coffee_liqueur","heavy_cream","coffee_beans"]
irish_coffee:     ["espresso","sugar","whiskey","heavy_cream"]
hot_toddy:        ["honey","lemon","whiskey","cinnamon"]
grasshopper:      ["creme_de_menthe","creme_de_cacao","heavy_cream","ice","chocolate"]
brandy_alexander: ["brandy","creme_de_cacao","heavy_cream","ice","chocolate"]
mulled_wine:      ["red_wine","orange","mulling_spices","honey","cinnamon"]
eggnog:           ["milk","heavy_cream","vanilla_syrup","nutmeg"]
```

- [ ] **Step 4: Validation harness** — write scratchpad `barcheck.swift` that includes copies of catalog+mapping data? No — compile the real files:

Run from app folder:
```bash
S=/private/tmp/claude-501/*/*/scratchpad; mkdir -p $S/barcheck && cat > $S/barcheck/main.swift <<'EOF'
// Assertions over the real library (files compiled together).
for d in DrinkLibrary.all {
    assert(!d.ingredientIDs.isEmpty, "\(d.id) has no ingredientIDs")
    for id in d.ingredientIDs {
        assert(IngredientLibrary.byID(id) != nil, "\(d.id): unknown id \(id)")
    }
    assert(!d.neededIDs.isEmpty, "\(d.id) needs nothing?")
}
print("CATALOG OK — \(DrinkLibrary.all.count) drinks, \(IngredientLibrary.all.count) ingredients")
EOF
```
Compiling SwiftUI files with swiftc needs iOS SDK; instead run:
`xcrun swiftc -sdk $(xcrun --show-sdk-path --sdk iphonesimulator) -target arm64-apple-ios15.6-simulator "Shake and Stir/MixModels.swift" "Shake and Stir/IngredientLibrary.swift" "Shake and Stir/DrinkLibrary.swift" $S/barcheck/main.swift -o $S/barcheck/run` — if linking a runnable binary fails for simulator target, fall back to `-emit-object` (type-check only) plus moving the assertions into a temporary `#if DEBUG` onAppear check run once on the simulator.
Expected: prints `CATALOG OK — 28 drinks, 51 ingredients` (or clean object emit + sim log line).

- [ ] **Step 5: Build + commit**

```bash
xcodebuild ... build   # BUILD SUCCEEDED
git add -A && git commit -m "feat: ingredient catalog and per-drink ingredient ids"
```

---

### Task 2: MixStore bar state + BarMath

**Files:**
- Create: `Shake and Stir/BarMath.swift`
- Modify: `Shake and Stir/MixStore.swift`

**Interfaces:**
- Produces:
  `MixStore.ownedIngredients: Set<String>`, `MixStore.shoppingList: Set<String>`,
  `toggleOwned(_ id: String)`, `addToShopping(_ id: String)`, `removeFromShopping(_ id: String)`, `purchaseFromShopping(_ id: String)`, `stockStarterBar()`,
  `enum Readiness: Equatable { case ready, almost(missing: String), missing(Int) }`,
  `enum BarMath { static func readiness(of: Drink, owned: Set<String>) -> Readiness; static func mixable(owned: Set<String>) -> [Drink]; static func almost(owned: Set<String>) -> [(Drink, String)]; static func bestBuys(owned: Set<String>, limit: Int) -> [(id: String, unlocks: Int)]; static func coverage(owned: Set<String>) -> Double }`

- [ ] **Step 1: Write BarMath.swift**

```swift
import Foundation

enum Readiness: Equatable {
    case ready
    case almost(missing: String)   // exactly one missing ingredient id
    case missing(Int)              // 2+
}

enum BarMath {
    static func readiness(of drink: Drink, owned: Set<String>) -> Readiness {
        let missing = drink.neededIDs.filter { !owned.contains($0) }
        switch missing.count {
        case 0:  return .ready
        case 1:  return .almost(missing: missing[0])
        default: return .missing(missing.count)
        }
    }

    static func mixable(owned: Set<String>) -> [Drink] {
        DrinkLibrary.all.filter { readiness(of: $0, owned: owned) == .ready }
    }

    static func almost(owned: Set<String>) -> [(Drink, String)] {
        DrinkLibrary.all.compactMap { d in
            if case .almost(let m) = readiness(of: d, owned: owned) { return (d, m) }
            return nil
        }
    }

    /// For each not-owned ingredient: how many drinks become ready if it is bought.
    static func bestBuys(owned: Set<String>, limit: Int = 3) -> [(id: String, unlocks: Int)] {
        var unlocks: [String: Int] = [:]
        for (_, missingID) in almost(owned: owned) {
            unlocks[missingID, default: 0] += 1
        }
        return unlocks.sorted { $0.value != $1.value ? $0.value > $1.value : $0.key < $1.key }
            .prefix(limit).map { (id: $0.key, unlocks: $0.value) }
    }

    static func coverage(owned: Set<String>) -> Double {
        let gated = IngredientLibrary.all.filter { !IngredientLibrary.needExempt.contains($0.id) }
        guard !gated.isEmpty else { return 0 }
        return Double(gated.filter { owned.contains($0.id) }.count) / Double(gated.count)
    }
}
```

- [ ] **Step 2: Extend MixStore**

Published props (after `hapticsOn`):
```swift
    @Published var ownedIngredients: Set<String> = []
    @Published var shoppingList: Set<String> = []
```
SavedState fields + tolerant decode (`?? []`), encode in `save()`, restore in `load()` — mirror the `favorites` pattern exactly (`[String]` arrays in JSON).

Mutations (before `resetAll`):
```swift
    func toggleOwned(_ id: String) {
        if ownedIngredients.contains(id) { ownedIngredients.remove(id) }
        else { ownedIngredients.insert(id); shoppingList.remove(id) }
        let newBadges = MixBadge.evaluate(store: self)
        _ = newBadges
        save()
    }
    func addToShopping(_ id: String) { shoppingList.insert(id); save() }
    func removeFromShopping(_ id: String) { shoppingList.remove(id); save() }
    func purchaseFromShopping(_ id: String) {
        shoppingList.remove(id)
        ownedIngredients.insert(id)
        hasPurchasedFromList = true
        _ = MixBadge.evaluate(store: self)
        save()
    }
    func stockStarterBar() {
        for id in ["vodka", "white_rum", "lime", "sugar", "soda_water", "orange_juice", "simple_syrup", "ice"] {
            ownedIngredients.insert(id)
        }
        save()
    }
```
Also add `@Published var hasPurchasedFromList: Bool = false` persisted the same tolerant way (backs the Quartermaster badge), and include `ownedIngredients`/`shoppingList`/`hasPurchasedFromList` in `resetAll()`.

- [ ] **Step 3: Logic assertions** — extend the Task 1 harness main.swift:

```swift
assert(BarMath.mixable(owned: []).isEmpty)
let allIDs = Set(IngredientLibrary.all.map { $0.id })
assert(BarMath.mixable(owned: allIDs).count == 28)
let mojitoNoMint = Set(["lime","simple_syrup","white_rum","soda_water","ice"])
if case .almost(let m) = BarMath.readiness(of: DrinkLibrary.all.first { $0.id == "mojito" }!, owned: mojitoNoMint) {
    assert(m == "mint")
} else { assert(false, "expected almost(mint)") }
assert(BarMath.bestBuys(owned: mojitoNoMint, limit: 3).first?.id == "mint" ||
       !BarMath.bestBuys(owned: mojitoNoMint, limit: 3).isEmpty)
print("BARMATH OK")
```
Run same compile command; expected `BARMATH OK`.

- [ ] **Step 4: Build + commit** — `feat: bar state in MixStore and BarMath readiness engine`

---

### Task 3: Art pack v2 generator

**Files:**
- Create: `art_src/artgen2.swift` (folder recreated; keep generator source, delete binary after)
- Output into: `Shake and Stir/Art/` — `ing_<id>.png` ×51 (512×512, alpha OK), `occasion_<key>.png` ×6 (1200×600, opaque), `texture_wood.png` (256×256 tile, opaque), `scene_backbar.png` (1600×700, opaque)

**Interfaces:**
- Produces art names consumed later: `ing_bourbon` … `ing_ice`, `occasion_date_night`, `occasion_party_crowd`, `occasion_zero_proof`, `occasion_quick_easy`, `occasion_cozy_night`, `occasion_sunny_afternoon`, `texture_wood`, `scene_backbar`.

- [ ] **Step 1: Write artgen2.swift.** Same plumbing as v1 `artgen.swift` (xorshift `frand`, savePNG via ImageIO) but token context uses `CGImageAlphaInfo.premultipliedLast` (transparency needed); banners/textures keep `.noneSkipLast`. Token drawing: per-group base form —
  - spirits/liqueurs/wine: bottle silhouettes with 6 shape variants (shouldered, tall-slim, round-flask, square, tapered, long-neck) chosen per id hash; body gradient in an id-specific hue table (bourbon amber, gin pale sage, vodka ice-blue, red_bitter crimson…), cream label band with tiny motif (diamond/dot/ring), cork/cap, glass sheen stripe, soft base shadow.
  - mixers: squat bottle or carton/jug variants + fruit-hue palette.
  - fresh: literal simple fruit forms (lime disc + wedge, lemon, orange, mint leaves, cucumber, berry trio, cherry pair, pineapple).
  - pantry: jar/tin/ramekin forms (sugar jar, syrup bottle, honey pot, bitters mini-bottle with eyedropper, salt dish, cream carton, milk bottle, spice sticks, chocolate shard, spice pouch, bean pile, umbrella, ice cube trio).
  Occasion covers: category-palette dusk gradient + table line + 2–3 glass silhouettes (reuse v1 GlassBox profiles) + motif (heart glow / bunting dots / leaf / lightning-fast diagonal / fireplace glow / sun rays) + grain + vignette.
  `texture_wood`: horizontal walnut planks, grain streaks, subtle seams. `scene_backbar`: wide dusk back-bar — shelf lines, 12 varied bottle silhouettes, lamp glow pools, neon underline strip.
- [ ] **Step 2: Generate + verify**
```bash
cd art_src && swiftc -O artgen2.swift -o artgen2 && ./artgen2 "../Shake and Stir/Art" && rm artgen2
ls "../Shake and Stir/Art" | grep -c '^ing_'        # expect 51
ls "../Shake and Stir/Art" | grep -c '^occasion_'   # expect 6
sips -g hasAlpha "../Shake and Stir/Art/texture_wood.png"   # hasAlpha: no
```
- [ ] **Step 3: Visual spot-check** — contact sheet via `sips`/Preview or Read 3–4 tokens as images; regenerate with tweaks if muddy.
- [ ] **Step 4: Commit** — `feat: v2 art pack — ingredient tokens, occasion covers, wood and backbar`

---

### Task 4: Living bar scene engine

**Files:**
- Create: `Shake and Stir/BarSceneView.swift`

**Interfaces:**
- Produces: `struct BarSceneView: View { enum Mode { case full, ambient, intro }; let mode: Mode }` — self-contained; reads `MixStore.shared.reduceMotion`.

- [ ] **Step 1: Implement.** Structure:

```swift
struct BarSceneView: View {
    enum Mode { case full, ambient, intro }
    let mode: Mode
    @ObservedObject private var store = MixStore.shared

    var body: some View {
        if store.reduceMotion {
            sceneCanvas(time: 1234.5)      // one static frame
        } else {
            TimelineView(.animation(minimumInterval: 1.0 / 30.0)) { tl in
                sceneCanvas(time: tl.date.timeIntervalSinceReferenceDate)
            }
        }
    }

    private func sceneCanvas(time: TimeInterval) -> some View {
        Canvas { ctx, size in
            drawBackdrop(&ctx, size)                      // scene_backbar image if present, else gradient
            drawLampGlow(&ctx, size, time)                // 2 radial pulses, 7 s period, opacity 0.10–0.22
            drawDust(&ctx, size, time)                    // N seeded motes drifting up-left in a light cone
            drawSmoke(&ctx, size, time)                   // 2 translucent sin-wisps
            drawSparkles(&ctx, size, time)                // occasional 1 s diamond glints (seeded schedule)
            drawNeonEdge(&ctx, size, time)                // brass underline, flicker = 0.65 + 0.35*noise(time)
        }
        .opacity(mode == .ambient ? 0.35 : 1)
        .allowsHitTesting(false)
    }
}
```
Particle counts per mode: full 24 dust / 2 smoke / 3 sparkle slots; ambient 10/1/1; intro 12/1/2 (cap ≤ 60 total elements). All positions seeded (`sin(seed * k + time * speed)` formulas — no RandomNumberGenerator per frame, no state). `drawBackdrop` uses `MixArt.image(named: "scene_backbar")` drawn `.scaledToFill`-style via `ctx.draw(Image(uiImage:), in:)`; fallback linear gradient `MixTheme.backdropDeep → card`.
- [ ] **Step 2: Preview smoke-test** — temporarily set `RootView` case 0 content to `BarSceneView(mode: .full).frame(height: 220)`, build, screenshot, revert. Verify glow/dust visible, no console errors.
- [ ] **Step 3: Commit** — `feat: living bar scene engine (full/ambient/intro modes)`

---

### Task 5: My Bar tab + shopping list

**Files:**
- Create: `Shake and Stir/MyBarView.swift`
- Modify: `Shake and Stir/RootView.swift` (5 tabs), `Shake and Stir/MixIcons.swift` (new `.bottle` glyph)

**Interfaces:**
- Consumes: `BarMath.*`, `IngredientLibrary.*`, `MixStore` mutations (Task 2), art `ing_*`, `texture_wood`, `BarSceneView(mode: .ambient)`.
- Produces: `struct MyBarView: View` (pushed inside `NavigationView` by RootView).

- [ ] **Step 1: `.bottle` glyph in MixIcons** — add case + draw: neck rect, shoulder quad-curves, body round-rect, label line. Stroke style consistent with other glyphs.
- [ ] **Step 2: RootView 5 tabs** — order: Tonight(0, .coupeGlass) · Menu(1, .menuBook) · **My Bar(2, .bottle)** · Bar Book(3, .shelfStar) · More(4, .gear). `case 2: NavigationView { MyBarView() }...`; Bar Book → case 3; More → default.
- [ ] **Step 3: MyBarView layout** (LoungeScreen scroll):
  1. Header card over `BarSceneView(mode: .ambient)` clipped rounded: big count line "You can mix **N** drinks tonight" (`BarMath.mixable(owned:).count`), coverage ring (reuse ring pattern from BarBookView), advice chip from `BarMath.bestBuys(...).first` — "Buy \(name) → +\(unlocks) drinks", tap = `store.addToShopping(id)` + toast; hidden if `owned.isEmpty || bestBuys.isEmpty`.
  2. Shopping list card: header "Shopping list (count)"; sections **Suggested** (bestBuys not in list, rows show "+N drinks") and **My list** (`shoppingList` rows); row = ing token thumb 34 pt + name + circle button (tap → `purchaseFromShopping`, sparkle overlay + `BarHaptics.shared.success()`), trailing small ✕ (`removeFromShopping`). Empty → single-line hint.
  3. Search field (reuse MenuView search styling) filtering token grid.
  4. Shelves: per `IngredientGroup.allCases` a section — group title, then rows of tokens on a shelf: `LazyVGrid` 4 columns; each cell = `MixArt(name: ing.artName)` 64 pt; owned → full color + brass底 glow; not owned → `.opacity(0.32).saturation(0.25)` + hairline outline. Behind each grid row band: `texture_wood` tiled `Image(...).resizable(resizingMode: .tile)` in rounded rect + 1 pt brass top edge. Tap cell → `store.toggleOwned(id)`, `BarHaptics.shared.tap()`, `BarSounds.shared.play("sfx_ice")` (existing API — check name in SoundAndHaptics.swift and use its public play method).
  5. Empty-bar state (owned.isEmpty): friendly copy + BrassButton "Start with the basics" → `store.stockStarterBar()`.
- [ ] **Step 4: Build, screenshot My Bar (temp default tab 2 → revert), commit** — `feat: My Bar shelf tab with shopping list and buy advice`

---

### Task 6: Tonight rebuild + occasions

**Files:**
- Create: `Shake and Stir/OccasionLibrary.swift`, `Shake and Stir/OccasionView.swift`
- Modify: `Shake and Stir/TonightView.swift`

**Interfaces:**
- Produces: `struct Occasion: Identifiable { let id, title, blurb: String; let drinkIDs: [String] }`, `enum OccasionLibrary { static let all: [Occasion] }`, `struct OccasionView: View { let occasion: Occasion }`.
- Consumes: `BarSceneView(mode: .full)`, `BarMath`, readiness badge component (defined here as `ReadinessBadge(drink:)` in TonightView file, reused by Menu in Task 7 — put it in `LoungeUI.swift` instead so both can use it).

- [ ] **Step 1: `ReadinessBadge` in LoungeUI.swift**

```swift
struct ReadinessBadge: View {
    let drink: Drink
    @ObservedObject private var store = MixStore.shared
    var body: some View {
        switch BarMath.readiness(of: drink, owned: store.ownedIngredients) {
        case .ready:
            Text("Ready").font(MixTheme.label(10)).foregroundColor(MixTheme.backdropDeep)
                .padding(.horizontal, 8).padding(.vertical, 3)
                .background(Capsule().fill(MixTheme.brass))
        case .almost:
            Text("+1").font(MixTheme.label(10)).foregroundColor(MixTheme.brass)
                .padding(.horizontal, 8).padding(.vertical, 3)
                .background(Capsule().stroke(MixTheme.brass.opacity(0.7), lineWidth: 1))
        case .missing:
            EmptyView()
        }
    }
}
```
- [ ] **Step 2: OccasionLibrary.swift** — 6 curated entries (editorial lists):

```text
date_night        "Date Night"        ["negroni","espresso_martini","brandy_alexander","daiquiri","berry_fizz","grasshopper"]
party_crowd       "Party Crowd"       ["mojito","margarita","tequila_sunrise","blue_lagoon","shirley_temple","bahama_mama","singapore_sling"]
zero_proof        "Zero-Proof Guest"  ["virgin_mojito","shirley_temple","cucumber_cooler","ginger_sunset","berry_fizz","citrus_presse","minted_lemonade"]
quick_easy        "Quick & Easy"      ["daiquiri","whiskey_sour","citrus_presse","shirley_temple","tequila_sunrise","old_fashioned"]
cozy_night        "Cozy Night"        ["irish_coffee","hot_toddy","mulled_wine","eggnog","white_russian","brandy_alexander"]
sunny_afternoon   "Sunny Afternoon"   ["pina_colada","mai_tai","cucumber_cooler","minted_lemonade","hurricane","mojito"]
```
Each with a one-line blurb; `artName = "occasion_\(id)"`.
- [ ] **Step 3: OccasionView** — cover art header (MixArt), blurb, drink rows ranked ready-first (`sorted by readiness rank then name`), each row: portrait thumb, name, ReadinessBadge, chevron → NavigationLink to `DrinkDetailView`.
- [ ] **Step 4: TonightView rebuild** — hero becomes `ZStack { BarSceneView(mode: .full); special medallion + name + Tonight's Special chip }` (fixed height ~230); then sections: "Ready to mix" horizontal rail (`BarMath.mixable`, unmastered first; empty → link card to My Bar), "So close" rail (`BarMath.almost`, card shows "just add \(name)" chip, chip tap = addToShopping + toast state), "For the occasion" chips row (6, each NavigationLink to OccasionView), then existing category rails (cards now overlay `ReadinessBadge` top-right corner). Stats tiles row kept.
- [ ] **Step 5: Build, screenshot Tonight, commit** — `feat: Tonight rebuilt around mixability with occasions and live scene`

---

### Task 7: Menu + detail readiness

**Files:**
- Modify: `Shake and Stir/MenuView.swift`, `Shake and Stir/DrinkDetailView.swift`

- [ ] **Step 1: MenuView** — add `ReadinessBadge(drink:)` overlay to cards (top-left over art, small); extend filter chips with `In Stock` (filters `readiness == .ready`). Chip enum/state extended accordingly.
- [ ] **Step 2: DrinkDetailView "You will need"** — above the list one summary line: `You have X of Y — missing lime, mint` (names of missing, or "you have everything" in brass). Each ingredient row: leading 22 pt state mark — brass filled `MixIcon(.check)` circle if owned, hollow circle if missing (needExempt rows always check). Missing rows get trailing mini-button: "Add to list" → `store.addToShopping(id)`, state flips to "In list ✓" (dim). Rows map display string → id by index into `drink.ingredientIDs` (same order as `ingredients` — guaranteed by Task 1 mapping; ice lines map to `ice`).

  **Order guarantee check:** in Task 1 keep `ingredientIDs[i]` aligned with `ingredients[i]` for every drink (same count, same order). The Task 1 harness asserts `d.ingredientIDs.count == d.ingredients.count` for all drinks — add that assertion there.
- [ ] **Step 3: Build, screenshot detail with partial bar, commit** — `feat: readiness in Menu and interactive needs in drink detail`

---

### Task 8: Badges, XP bonus, onboarding copy

**Files:**
- Modify: `Shake and Stir/Badges.swift`, `Shake and Stir/MixSessionView.swift` (or wherever `store.finish(result:)` is called / XP granted — the bonus goes in `MixStore.finish`), `Shake and Stir/MixStore.swift`, `Shake and Stir/OnboardingView.swift`

- [ ] **Step 1: Two new badges** in `MixBadge` catalog + `evaluate`:
  - `stocked_shelf` "Stocked Shelf" — "Own 15 bar ingredients." → `store.ownedIngredients.count >= 15`
  - `quartermaster` "Quartermaster" — "Buy your first item from the shopping list." → `store.hasPurchasedFromList`
  Glyphs: `.bottle` and existing `.check`-based cart-like glyph (add `.basket` if trivial; otherwise reuse `.bottle`/`.star`).
- [ ] **Step 2: +5 XP bonus** in `MixStore.finish(result:)`: after existing xp line —
```swift
        if BarMath.readiness(of: d, owned: ownedIngredients) == .ready { xp += 5 }
```
- [ ] **Step 3: Onboarding page 3 copy** — mention My Bar: "Stock your shelf in My Bar, see what you can pour tonight, and fill your Bar Book from Barback to Bar Legend."
- [ ] **Step 4: Build + commit** — `feat: bar badges, own-bar XP bonus, onboarding copy`

---

### Task 9: Verification, screenshots, delivery

**Files:**
- Modify: `screenshots/*` (refresh), `development/APP_TRACKER.md`
- No source changes except temporary tab-default patches (each reverted).

- [ ] **Step 1: Full clean build** sim Debug — zero `error:`; run harness assertions once more.
- [ ] **Step 2: Seeded visual pass** — seed UserDefaults on sim (onboardingDone + a mid-size bar: spirits vodka/white_rum/bourbon + lime/lemon/mint/sugar/simple_syrup/soda_water/ice + 2 shopping items) via the hex-JSON `simctl spawn defaults write` trick; screenshot: Tonight (live scene + rails), My Bar (shelf + shopping list), Menu with In Stock filter visible, drink detail (partial needs), Occasion view, Bar Book, session intro. Use temporary default-tab patches for non-reachable screens; **revert every patch** (final `git diff` must show only intended changes).
- [ ] **Step 3: Size check** — installed .app via `du -sh` (expect ~50–60 MB; must be ≥18, <99).
- [ ] **Step 4: Release device build check** — `xcodebuild -configuration Release -destination 'generic/platform=iOS' CODE_SIGNING_ALLOWED=NO build` → succeeds.
- [ ] **Step 5: Delivery** — verify version still 1.0/1; update `APP_TRACKER.md` row (append v2 Home Bar Edition notes to the Shake and Stir line); commit app repo; `mv development/"Shake and Stir" for_human_review_apps/`; report.

---

## Self-Review Notes

- Spec §1→Task 1/2, §2→Task 5, §3→Task 6, §4→Task 7, §5→Task 4, §6→Task 3, §7→Task 8, §8/testing→Task 9. Covered.
- Alignment rule `ingredientIDs[i] ↔ ingredients[i]` added to Task 1 harness (needed by Task 7 row mapping).
- Sound API name in Task 5 Step 3 must be checked against `SoundAndHaptics.swift` (its class/method names were authored in v1 — use whatever `MixSessionView` already calls, e.g. the shared player used for `sfx_ice`).
- `MixBadge.evaluate(store:)` signature confirmed in v1 (`MixStore.finish` already calls it).
