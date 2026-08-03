# Plate Passport v2 — art overhaul and enrichment

Date: 2026-08-03. Version stays 1.0 (build 1). App is already fully white (no WebView, no
network code — verified by grep and on the Release binary) and already universal
(iPhone + iPad, four iPad orientations). This rework is about the two real complaints:
the plates look like pale scribbles, and the app reads like a thin reference book.

## Why the v1 art failed

Looked at honestly, the v1 dish plates have five compounding problems:

1. **Translucent paint.** Everything was washed at 0.2–0.5 alpha. Overlapping food layers
   ghost through each other — the Japan atlas card shows a board with two see-through
   bowls floating over it, which reads as a compositing bug, not a drawing.
2. **Top-down view.** Every vessel was a flat ellipse seen from above; every dish became a
   specimen diagram. Food has no height, no heap, no appetite.
3. **Sepia everywhere.** The washes were so desaturated the food has no colour identity —
   a curry, a broth and a stew all read as beige.
4. **Food too small.** The vessel occupies ~55% of the frame and the food maybe half of
   that; the rest is empty paper and a thin sketchy frame.
5. **Multiply-composited scenes.** The cuisine table scenes stacked three translucent
   sub-renders; every overlap ghosts.

## v2 art direction: the gouache cookbook plate

Mid-century culinary illustration: **opaque gouache over an ink drawing** on warm paper.
Deep saturated paint laid back-to-front (painter's algorithm — overlaps occlude, never
ghost), ink contour and hatching on top for form, one warm light, a grounding cast
shadow, and a proper plate frame. Food fills ~70% of the frame, drawn in a low 3/4 view
so bowls have walls and heaps have height.

Generator changes (art_src):

- **Opaque paint primitives.** `gouache(region, tone)` fills a region with an opaque
  colour modulated by coarse noise (brush unevenness) and a darker pooled edge; three
  tones per palette (shadow / body / light) replace single translucent washes. Paper
  shows only where nothing was painted.
- **Palettes saturated.** The twelve palettes get body colours at roughly ×1.6 the
  saturation of v1, each with fixed shadow and highlight variants, plus vivid fixed
  accent colours (herb green, chili red, saffron, cream) that do not desaturate.
- **3/4 vessels.** Every vessel is rebuilt with a visible rim ellipse AND a side wall
  (bowls, pots, glasses, baskets), an interior in shadow tone, and food that sits *in*
  the vessel with its own height, cropping the far rim.
- **Bigger, staged food.** Main archetype fills the well; extras sit behind/beside with
  real occlusion (drawn first, painted over); garnish scattered on top last.
- **Composition.** A table line and a soft backdrop band behind the vessel (wall +
  table), the deep vignette kept, and a clean double-rule frame with a small diamond at
  each corner instead of the sketchy "plate mark".
- **Scenes on one canvas.** The cuisine table is drawn back-to-front on a single plate —
  three vessels at three depths, overlapping honestly, over a patterned cloth.
- Visas, guide boards, route maps, passport paper and the cover stay as-is (they were
  the strongest part), except the visa centre rosette keeps its darker v2 inking.

Sample gate: render one dish per vessel type (12 plates), eyeball, iterate, and only
then run the full 254-plate render. Plates ship as JPEG (q0.86), icon stays PNG.

## Feature enrichment

Three additions that make it a tool rather than a catalogue, all offline:

1. **Want-to-try list.** A flag on every unstamped dish ("Add to my list"). Home gets a
   "Next on your list" shelf; the atlas gets a "My list" filter; the stats row counts
   it. Stored in the save file (`wishlist: [String]`).
2. **Palate match.** For each cuisine, similarity between your palate vector and the
   cuisine's average dish profile, shown as "N% your kind of table" on cuisine pages
   and atlas rows once you have 5+ stamps. The journal gains "Your best-matched
   kitchens" (top 3) — turns the taste data into advice.
3. **Stamp aftermath.** After the strike, the press screen shows what the stamp just
   advanced: the visas it moved with their new counts ("Monsoon Transit — 2 to go"),
   pulled from the same progress logic. Makes every stamp feel consequential.

UI polish alongside: dish cards in the atlas plates-grid go full-bleed (art fills the
card, name on a paper band), the region leaves show the three stamp slots larger, and
the dish page opens on a full-width plate. No new dependencies, no version bump.

## Out of scope

New dish content, sounds beyond the existing feedback, trips/grouping of stamps,
account/cloud anything. TabView/SF Symbols/emoji remain forbidden; iOS 15.6 floor holds.

## Verification

- Offline validator re-run (now also checks wishlist invariants: no duplicate ids, no
  stamped dish in the wishlist after stamping).
- Debug + Release builds clean; true bundle size measured by byte-sum (target ≥ 18 MB,
  < 99 MB).
- Every screenshot re-shot on fresh dedicated simulators (old art must not linger).
- otool/strings re-audit of the Release binary stays clean of WebKit/CFNetwork/URLs.
