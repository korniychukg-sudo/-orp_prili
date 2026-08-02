# Salt Road — ink illustration system and march decisions

Date: 2026-08-02
Status: approved by Master, ready for implementation

## Problem

Two things are wrong with the current build.

**The art is bad.** All 98 plates are flat filled shapes with hard vector edges. There is no
line, no texture, no material, no light direction, and no composition — objects are scattered
by a random number generator rather than placed in a frame. The result reads as a wireframe
mock-up, not an illustration.

**The march is passive.** The core loop on the road is a single "Walk on" button pressed
repeatedly until an event fires. The player makes no decisions between towns.

## Root cause of the art failure

The generator only fills shapes. Filling is the thing Core Graphics does *worst* in terms of
apparent craft — a filled polygon always looks like a filled polygon. Drawing is the thing it
does *best*: a hatched shadow or a tapered contour is genuinely the same operation a pen makes.
The fix is not "better colours"; it is to stop painting and start drawing.

## Design

### 1. Ink-and-wash rendering system

Every plate is built in three layers, identical in construction across the whole set so the
98 images read as one book.

**Paper layer.** Toned laid paper: warm base, faint horizontal laid lines, fibre flecks,
uneven tone toward the edges, one or two pale water stains. The same paper recipe everywhere.

**Wash layer.** Two or three muted washes only. Edges are irregular and pool darker at the
boundary, the way watercolour dries. Washes deliberately overrun the ink outline by a small
random margin — misregistration is what makes a wash read as hand-laid rather than vector.

**Ink layer.** Everything is drawn:

- Contour with varying weight: heavier on the shadow side, lighter on the lit side, tapered ends
- Cross-hatching for shadow, density driven by one light direction per plate; two or three
  passes at different angles for the darkest passages
- Stipple for sand, short flicks for foliage and fur, long parallel strokes for water and sky
- A drawn horizon and drawn ground texture instead of a flat fill

Per plate, additionally:

- **One light direction**, applied consistently, with a cast shadow under every camel, figure
  and wall
- **Staged composition**: a near framing element at one edge, the subject in the middle
  distance, a silhouette far off. Three depth planes, each with its own ink weight and hatch
  density — the depth comes from this, not from colour
- **Atmospheric perspective expressed in line**: far things get thin sparse line, near things
  thick dense line
- **Individual variation** for camels and figures — posture, load, blanket pattern — instead
  of one repeated stamp

### 2. March decisions

The march screen stops being one button. Three standing settings, changed whenever the player
wants, applied to each day walked.

| Setting | Options | Trade |
|---|---|---|
| Pace | Easy · Steady · Forced | Forced covers about 1.6 days of ground per day for double camel wear and a morale cost. Easy mends camels and spends the season. |
| Hours | Day march · Night march | Night saves roughly 35% of water and camel condition, costs morale, and risks losing the way without a star-reader. |
| Water | Full ration · Short ration | Short ration cuts water use by about 30% at a steady morale cost. |

Rules:

- Leg progress becomes fractional rather than whole days, so pace genuinely matters
- Night marching raises the chance of navigation events and lowers the chance of heat events;
  day marching does the reverse
- Short rations compound — three consecutive days and the camp reacts
- Companions change the *options*, not merely the numbers: Adala removes the night-march risk,
  Nawa reduces the morale cost of short rations, Mira caps the morale bleed from a forced pace,
  Ferouz reveals the day's hazard before the player commits
- Settings persist between days, so this is a decision revisited on changing conditions rather
  than three taps every morning

### 3. iPad support

`TARGETED_DEVICE_FAMILY = "1,2"` in both Debug and Release. The `~ipad` orientation array
already carries `PortraitUpsideDown`, which is what ITMS-90474 requires. Town and market
screens gain a two-column layout at regular width; prose keeps its existing reading measure.

### 4. Version

`MARKETING_VERSION = 1.0` and `CURRENT_PROJECT_VERSION = 1` stay as they are.

### 5. Cloaking

Salt Road has never contained a WebView, a launch check, a redirect tracker, an
`NSAppTransportSecurity` entry, or any remote URL. Verified by grep over the source. Nothing to
remove; the app is fully offline and honest as built.

## Risk and control

The art rebuild is the bulk of the work and it is the part that already failed once. Control:
write the ink toolkit first, render exactly three plates — a town, a dune event, a portrait —
and show them to Master. Only proceed to the remaining 95 after explicit approval. No further
blind batches of 98.

## Out of scope

Explicitly rejected during brainstorming: a persistent trading house between runs, an animated
living road scene, and a caravan-book artefact. These may be revisited later; they are not part
of this change.
