# Scribe Stone — visual overhaul and commission bench

Date: 2026-08-02
Status: approved
App: `for_human_review_apps/Scribe Stone` — stays at version 1.0, build 1

## Why

The delivered v1 works and is technically clean, but the generated art reads as
placeholder output. Three systemic faults, not a matter of taste:

1. **No composition.** Signs float centred in an empty field. No foreground,
   middle ground or background; nothing for the eye to land on.
2. **No light model.** Everything is flat fill. Objects cast no shadow, have no
   volume, and do not meet the surface they sit on.
3. **Naive motifs instead of illustrations.** "The Alphabet's Family Tree" is
   five straight lines radiating from a point. "Why Writing Was Invented" is
   scattered circles. This is debug output.

The app also reads as a browsable reference. It needs one through-line that
turns looking things up into doing something.

## Art direction

**Museum engraving plus cinematic light.** Chapter plates and banners are tipped
-in plates from an old archaeological atlas: aged paper, ink hatching, ruled
borders, brass captions. Hero art (workshop scene, script banners) is lit
cinematically — lamp light, depth, haze, cast shadows.

### Engine changes (`art_src`)

Three capabilities the current generator does not have at all:

- **Unified key light** from upper left across every asset. Incisions, objects
  and edges cast consistent shadows; contact shadows darken where objects meet.
- **Ink hatching.** Form is modelled with cross-hatching and stipple rather than
  flat fill. This is the single largest quality lever — it is what makes an
  engraving read as an engraving.
- **Plate furniture.** Double ruled border, corner rules, caption band, plate
  number, faint age foxing, deckled paper edge.
- **Layered composition helpers** so no subject is ever drawn alone on a field:
  background plane → mid objects → subject → foreground framing → vignette.

### Real subjects

| Plate | Subject |
|---|---|
| Why writing was invented | Broken clay bulla spilling counting tokens, beside the flattened tablet it became |
| How a picture becomes a letter | Ox head above, three rotation steps down, letter A below |
| The alphabet's family tree | Real genealogy with actual letterforms at each node: proto-Sinaitic → Phoenician → Greek and Aramaic → Latin |
| Which way do you read | Boustrophedon field: ox and plough, furrows, arrows turning at the ends |
| What scribes wrote on | Cutaway shelf: clay tablet, papyrus roll, wax diptych, birch bark, stone stele, each labelled |
| How a script dies | Time ribbon with the last dated inscription marked |
| How a script is deciphered | Rosetta's three registers with tie-lines between matching cartouches |
| Names in a foreign script | One name written six times, once per script, substitutions circled |

**Script banners become places**: hieroglyphs on a temple wall with a column
edge and raking light; cuneiform on a tablet on a scribe's table with a stylus;
Phoenician on a bronze plaque nailed to a ship's timber; Greek on a stele with a
pediment; runes on a plank with a knife stuck in it; ogham on the corner of a
standing stone against grass and sky.

**Workshop scenes get real depth**: floor, back wall with a window opening,
shelving with rolls and jars, a bench with thickness and cast shadow, tools with
material, dust in the light beam. Four genuinely different times of day.

**New: 12 inscription plates** — each real inscription gets the actual object
drawn (comb, slab, cliff face, sarcophagus, jug, cup), not glyphs on a texture.

Target: ~55 assets, 90–120 MB.

## The hook: the commission bench

A client arrives with a job. 40+ authored commissions across the six cultures,
each with who they are, what they want written, and **why**.

> A widow of Memphis. "My husband's name. He was a scribe at the temple. Let
> them read it when I am gone."

The player picks a script and a material and cuts it. Grading is soft and
delivered in character:

- **Fitting** — the client is delighted, pays in full, the piece stays in the
  workshop.
- **Plausible but wrong** — accepted with a remark that teaches: *"You have cut
  my father's name in ink on reed. It will not outlive me."* Reduced pay.
- **There is no fail state.**

Pay buys workshop upgrades.

## The workshop that grows

The Bench home screen stops being a picture and becomes a composite whose layers
switch on with progress: a shelf of finished pieces, a rack of unlocked tools,
pigment jars that fill, a window, an apprentice's stool, a cat. Each kept piece
leans against the wall, up to eight visible.

## iPad

`TARGETED_DEVICE_FAMILY = "1,2"`, and `UIInterfaceOrientationPortraitUpsideDown`
added to the `~ipad` orientation array — without it App Store upload fails with
ITMS-90474. Two-column layouts: bench = slab plus tools; signs = list plus
detail; almanac = list plus reader; gallery = wall plus selected piece. Art at
higher resolution.

## Unchanged

- Version 1.0, build 1.
- No network of any kind: no WebView, no ATS key, no URL in the binary, no
  permissions. Verified on the Release binary.
- All existing content: 143 signs, 12 inscriptions, 8 chapters, 30-term
  glossary, generated quiz, script drill, flashcards, ranks, awards.

## Risks

- Build times on this machine are long and another session competes for it;
  every verification cycle is expensive. Batch changes before rebuilding.
- The hatching engine is the main unknown. If hatching at asset resolution is
  too slow or too noisy, fall back to fewer, larger hatch strokes with the same
  light model — the light model alone already fixes most of the flatness.
