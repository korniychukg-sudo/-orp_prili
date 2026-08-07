# Hebrew Year — Living Companion Edition

**Date:** 2026-08-07
**App:** `for_human_review_apps/Hebrew Year` (worked in place; canonical location)
**Constraint:** marketing version and build stay **1.0 (1)**. Fully offline, iOS 15.6+, no new permissions, no notifications, custom UI only.

## Problem

v1 works and is correct, but reads like a tidy reference book: static art header, flat
cards, nothing that moves with the real world, nothing personal. The goal of this pass
is that opening the app feels like checking a living instrument — the sky, the moon,
the week, your own dates — not opening a handbook.

## Approaches considered

1. **Visual reskin only** (grain, shadows, haptics, animation). Cheap, but the app
   would still be an impersonal list — rejected as insufficient.
2. **Living-calendar + personal stake** — make the calendar itself alive (real sky,
   real moon phase, live countdowns, the Omer) and let the user put their own dates
   into it. All computable offline from engines we already validated. **Chosen.**
3. **Gamified almanac** (badges/XP/streaks — the house pattern). Rejected: scoring
   points on a religious calendar reads tone-deaf; richness must come from utility.

Weekly Torah-portion schedule was considered and **deliberately excluded**: the
keviyah-based parsha algorithm cannot be validated offline against trusted anchors in
this session, and shipping guessable religious data is worse than omitting it.

## Design

### 1. Living Sky header (Today)

Replace the static month plate with a `SkyScene` Canvas that renders the actual sky
for the selected city at the current minute: day / golden hour / dusk / night bands
from the existing solar engine (sunrise/sunset ± civil-twilight offsets), a drawn sun
whose arc position follows the day fraction, stars and the **real moon phase** at
night, a silhouette skyline strip at the bottom (drawn, not photographic — matches the
ink plates). Hebrew date + gematria year are overlaid on the scene in the serif face.
`TimelineView(.everyMinute)` drives it; reduce-motion not needed (no continuous
animation, one redraw a minute).

### 2. Moon engine, moon everywhere

`MoonEngine`: synodic phase from the astronomical epoch (JD 2451550.1, synodic month
29.530588853 d) → phase fraction, illumination, waxing flag; `MoonDiscView` draws the
lit limb with two arcs (terminator ellipse), craters stippled from a seeded RNG.
Usage: night sky in SkyScene; a "Tonight's moon" chip on Today (name of phase +
Hebrew day-of-month note, e.g. "15 Av — full moon of Tu B'Av"); moon column in the
Candles weekly table. Validator: across Hebrew years 5700–5800, 1st of month ⇒ phase
≥ 0.88 or ≤ 0.15, 15th ⇒ phase 0.35–0.65 (the fixed calendar tracks the moon within
~1.5 days).

### 3. Erev-Shabbat / erev-festival live countdown

When the next candle-lighting for the selected city is under 24 h away, the Shabbat
card on Today grows a live "Candles in 3 h 12 m" line (minute-driven), and after
sunset on Friday flips to "Shabbat shalom — havdalah at HH:MM". Same treatment for a
festival eve (uses the same countdown component).

### 4. Year Wheel

A full-screen circular map of the current Hebrew year, opened from a new Today module
and from the Holidays header: ring of 12/13 month segments (leap year shows Adar I/II),
each holiday a colored tick at its true angular date position, today's pointer, moon
midpoints (15th) as small discs, tap a tick → holiday detail. Pure Canvas — no new
art. This is the signature "not a handbook" visual.

### 5. Personal Dates (the sticky feature)

Store entries `{name, gregorian date, kind: birthday | anniversary | memorial}`
(Codable JSON in UserDefaults, tolerant decoding). For each: the Hebrew date of the
original event, the **next Hebrew-anniversary** (same Hebrew month/day, with the
standard Adar/30th fallback rules: Adar → Adar II in leap years handled as Adar for
non-leap originals; 30 Cheshvan/Kislev → 1 Kislev/Tevet in short years), countdown
pill, and the count of Hebrew years. Today shows the next two upcoming; a managed
list screen (add/edit/delete, custom sheet pickers reused from Converter) lives
behind it. Memorial entries render with the night palette and a candle icon instead
of festive gold.

### 6. Omer counter card

During the 49 days (16 Nisan – 5 Sivan): a Today card with "Day N of the Omer",
weeks-and-days formula text, tonight's blessing link, and a 7×7 dot grid filled to N.
Already computed in Converter; this surfaces it where it's useful.

### 7. Sun times card (Candles)

"The Sun in <city> today": sunrise, sunset, day length, plus candle/havdalah context.
One card, data already available from `SolarTimes`.

### 8. Checklist progress on Today

The next candle-eve holiday's checklist state ("Rosh Hashanah: 4 of 10 ready") with a
mini progress ring, deep-linking to the guide's checklist. Makes the checklists (v1's
best practical feature) impossible to miss.

### 9. Texture & touch polish

- Laid-paper grain tile (new generated `texture_grain` plate) overlaid app-wide at low
  opacity via a tiled resizable Image on the parchment background.
- Haptics: light impact on checklist ticks, quiz answers, wheel taps, personal-date
  saves (UIKit generators, wrapped).
- Cards get a subtle two-layer shadow + hairline highlight for depth; hero images in
  list cards crop bottom-aligned so still-life subjects stay in frame.
- Springy `ScalePressStyle` stays; add gentle appear-transitions on Today modules.

## Components

| Unit | Purpose | Depends on |
|---|---|---|
| `MoonEngine.swift` | phase math + `MoonDiscView` | Foundation |
| `SkyScene.swift` | living Today header | SolarTimes, MoonEngine, HYCities |
| `YearWheel.swift` | wheel view + full-screen sheet | HebrewCalendar, HolidayDates |
| `PersonalDates.swift` | model, next-anniversary math, list/add UI | HebrewCalendar, HYStore |
| `HYHaptics.swift` | impact/selection wrappers | UIKit |
| Store additions | `personalDates` JSON persistence | HYStore |
| View updates | Today, Candles, Holidays, Components | above |

## Error handling

- Personal-date decode failures fall back to empty list (never crash on old data).
- Anniversary edge rules (Adar, 30th) covered explicitly in the validator.
- Polar-city nil sun times: SkyScene falls back to a fixed dusk gradient; countdown
  hides rather than showing "--".

## Testing

Extend the existing offline swiftc harness: moon-phase vs Hebrew-day bounds over 100
years; personal-date next-anniversary over synthetic cases (leap Adar, 30 Cheshvan,
today-is-the-day) and monotonicity (next ≥ today, within 385 days); Omer window
arithmetic; SkyScene pure helpers (band selection) by direct call. Then: clean sim
Debug + device Release builds, on-sim screenshots of every changed screen
(re-shot per the stale-screenshot rule), otool/strings network scan, size ≥ 18 MB,
version check 1.0 (1).
