# Inner Tides v3 — Shore Life & Field Guides (Design Spec)

**Date:** 2026-07-28
**App:** Inner Tides (`com.innertides.app`)
**Version:** stays 1.0 (build 1) — never bumped before delivery
**Goal:** Make the app substantially fuller and higher-quality: an illustrated
guide to intertidal wildlife tied to the tide mechanic, long-form field guides,
and full iPad support — without duplicating other shipped apps' core loops.

## Approved scope (Master picked)

1. **Shore Life** — illustrated intertidal-wildlife guide with a Zone Explorer.
2. **Guides** — 7 long-form illustrated guides with interactive checklists.
3. **iPad support** — mandatory.

Explicitly out of scope: bulk "more facts everywhere" additions (not selected).
Quiz/glossary/lessons stay as-is from v2.

## 1. Shore Life

### Zone Explorer (unique mechanic — ties wildlife to the tide)
`ZoneExplorerView`: a Canvas cross-section of a rocky shore showing the five
intertidal bands — Splash, High, Mid, Low, Subtidal fringe. A **draggable water
level**: pull the tide up/down with a finger; bands flood/drain visually and the
species visible at that water level highlight in a chip row. A "set to now"
button snaps the level to the current modelled tide fraction of the home coast.
Full-flood + full-drain interaction awards a badge.

### Species catalog
~30 species in `ShoreLifeModels.swift` (struct `ShoreSpecies`): id, name,
common-name style English, zone (enum of the 5 bands), sizeText, whenToLook
("Best at low tide" / "Visible at any tide" / "Watch the water's edge"),
where ("On open rock", "In pools", "Under weed"...), 3 facts, art name.
Species mix: barnacle, limpet, periwinkle, dog whelk, blue mussel, shore crab,
hermit crab, beadlet anemone, snakelocks anemone, common starfish, brittle star,
sea urchin, blenny/rockpool fish, goby, prawn, sea lettuce, bladder wrack, kelp,
coralline algae, sponge, chiton, cockle, razor clam, lugworm, sand hopper,
oystercatcher, curlew, herring gull, harbour seal, common octopus. (Final list
may vary slightly; count stays ≈30.)

### Species UI
- `ShoreLifeView`: intro card linking to Zone Explorer, then species grouped by
  zone (adaptive grid; 2-3 columns on iPad).
- `SpeciesDetailView`: portrait hero, zone + when-to-look chips, size/where rows,
  facts, subtle "Spotted" toggle (a Set<String> in the store; feeds badges/XP
  only — deliberately NO notes/counts/date logging, that belongs to Tide Log and
  keeps this distinct from Birds Next Door's life-list loop).
- Coast detail gets a small "Shore life" link card into ShoreLifeView.

## 2. Guides

`GuideCatalog` in `ShoreLifeModels.swift` (or its own `GuideModels` section):
7 guides, each: id, title, subtitle, art, intro, sections (title + paragraphs),
optional checklist items, safety callout. Guides:
1. Exploring Tide Pools Safely (safety callout heavy)
2. What to Bring to the Shore (interactive checklist)
3. How to Read a Beach (sections on wrack line, zonation, wet sand)
4. Tidepooling Etiquette (leave-no-trace rules)
5. Beachcombing Finds (what shells/egg cases/driftwood mean)
6. Photographing the Shore (golden hour + tide windows tie-in)
7. Planning a Shore Day Around the Tide (uses the app's own windows)

UI: `GuidesView` list with hero cards + read progress; `GuideDetailView` with
hero art, sections, checklist rows (persisted toggles in store), safety callout
style. Reading a guide awards XP; finishing all → badge.

## 3. Discover hub (Learn tab evolves)

Tab label changes `Learn` → `Discover`. `LearnView` becomes a hub of feature
cards: Shore Life, Guides, Lessons (existing 18), Tide Quiz, Glossary.
Lessons list moves one level deeper (`LessonsListView`) to keep the hub clean.
All v2 lesson/quiz/glossary functionality is preserved unchanged.

## 4. iPad support

- `TARGETED_DEVICE_FAMILY = "1,2"` in both Debug and Release.
- A reusable `ContentColumn` wrapper: centers scroll content at
  `maxWidth: 760` on wide layouts (applied to the main scroll views).
- Adaptive grids (`GridItem(.adaptive(...))`) for coasts list, species grid,
  hub cards, badges — 2-3 columns on iPad naturally.
- Living scene, Tide Machine, charts already scale with GeometryReader/aspect
  ratios; verify by iPad simulator screenshots (Today, Machine, Coasts,
  Discover, a guide, Zone Explorer).
- Info.plist iPad orientations already present (incl. upside down). No other
  plist changes.

## 5. Badges & store additions

+6 badges (total 30): `first_species` (spot 1), `ten_species` (10),
`all_species` (all), `zone_master` (flood + drain the Zone Explorer fully),
`guide_read` (finish first guide), `guides_all` (finish all 7).
Store: `spottedSpecies: Set<String>`, `readGuides: Set<String>`,
`checkedItems: Set<String>` (guide checklist persistence), award methods, XP
(+4 per species spotted, +8 per guide finished).

## 6. Art generation

Extend `art_src/gen_art.swift`:
- ~30 species portraits at 1024×1024: same muted deep-sea palette family,
  procedural stylised creature silhouettes/shapes on soft radial grounds with
  grain (consistent with existing art; recognisable, not photoreal).
- 7 guide heroes at 1500×1140 reusing the coast/motif machinery (new motifs:
  pool, checklist satchel, wrack line, hands/etiquette, finds flat-lay, camera
  at sunset, planning map).
Size budget: current Art ≈46 MB; +species ≈20 MB; +guides ≈8 MB → ≈74 MB art,
app well under the 99 MB cap (and far above the 18 MB floor). Verify with `du`
after generation; if over ~85 MB, re-render species at 900².

## Files

New: `ShoreLifeModels.swift`, `ZoneExplorerView.swift`, `ShoreLifeViews.swift`
(list + detail), `GuidesViews.swift` (list + detail), `LessonsListView` (inside
LearnView file is fine).
Changed: `LearnView.swift` (hub), `RootView.swift` (tab label Discover),
`CoastDetailView.swift` (shore-life link), `TideStore.swift`, `TideModels.swift`
(badges), `TideTheme.swift` (ContentColumn), `TideIcons.swift` (new glyphs:
crab/shell for Shore Life, backpack/list for Guides, zone bands), pbxproj
(new files + TARGETED_DEVICE_FAMILY), `art_src/gen_art.swift`.

## Constraints (unchanged)

Version **1.0 build 1** — do not bump. iOS 15.6+, forced light, custom
Canvas/Shape icons only (no SF Symbols/emoji), offline, no notifications,
WebView gate untouched, English (US). App size < 99 MB.

## Success criteria

- Clean build, 0 code warnings; no crashes on all tabs (iPhone + iPad sim).
- Zone Explorer drag floods/drains bands and filters species live.
- ~30 species with individual portraits; 7 guides with working checklists.
- Discover hub navigates to all five sections; v2 features intact.
- iPad renders all key screens with centered columns/adaptive grids.
- App size within 18–99 MB. Trackers + APP_DESCRIPTIONS updated.
