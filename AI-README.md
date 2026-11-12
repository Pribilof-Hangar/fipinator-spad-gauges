# AI-README: how these gauges work in SPAD.neXt (and what had to be worked around)

For you and your AI assistant, if you want to change these gauges or make new ones. These are the things that
didn't work the obvious way in SPAD.neXt with MSFS 2024, and what does work. Each was found by trying it on real
FIPs, reading SPAD.neXt's log, or reading the sim's values in flight.

The gauges here are generated from the FIPinator project (gauge files in a JSON format, turned into SPAD.neXt
gauges by an exporter). Changing a gauge's XML here by hand works, but the next generated version would replace it;
fixes in the FIPinator project reach everyone.

## A SPAD.neXt FIP gauge, in short

- One `.zip` per gauge: an XML file at the top (`<Gauge>`) and the pictures in a `1024` folder inside the zip. The
  XML names pictures without the folder.
- SPAD.neXt follows the gauge schema `gauge.xsd` (published by SPAD.neXt in github.com/SPAD-neXt/DemoCode).
- Elements are drawn in file order, first at the back. A `<GaugeSettings><Category>` names the group in SPAD's list
  ("FIPinator: DC-6"). SPAD's list doesn't show sub-folders, so the category does that job.
- Formulas are FS-style RPN, e.g. `(A:INDICATED ALTITUDE, feet) 1000 /`.

## Knobs and buttons

- **Knobs: use `<Mouse>` blocks.** One block per knob, right knob first: `<Mouse><Area><Area>` with
  `<Cursor Type="DownArrow"/>` and `<Click Event="KOHLSMAN_DEC" Repeat="Yes"/>`, then an `<Area>` with `UpArrow` and
  the `_INC` event. DownArrow = turning anticlockwise. Plain sim event names (no prefix).
- **`<GaugeSettings><Buttons><Button BoundTo="FIP_RIGHTDIAL_CLOCKWISE" ...>` does nothing for knobs** (neither does
  the `FIP_DIAL_RIGHT_...` spelling); SPAD's log doesn't even mention them.
- S buttons go in as `<Button BoundTo="FIP_BUTTON_S1" Event="..."/>` (only a few template gauges use them; lightly
  tested).
- Page up/down: SPAD.neXt sets these itself for every gauge on a FIP (next/previous gauge). Gauges must not set them.
- A user's own button settings in their SPAD.neXt profile win over the gauge's.
- SPAD sends one event per knob click: there's no "faster knob" (in FIPinator, the left knob often steps 10x).

## Formulas (RPN)

- **SPAD.neXt ignores `near` (round).** A picture number worked out with `near` came out as 37.77 and SPAD asked for
  `slide1_37.77.png`, which doesn't exist (it then shows the element's first picture). Round with `0.5 + flr`.
- `flr`, `abs`, `min`, `max`, `+ - * /`, `<`, `>`, `if{ } els{ }` all work.
- Avoid `%`: use `x - span * flr(x / span)`.

## Moving parts

- **Needles:** an `Element` with an `Image` drawn pointing right (`PointsTo="East"`), its `Axis` at the pivot, and a
  `Rotate` with a `Nonlinearity` table of value -> point on a circle around the pivot (points at most 10 degrees
  apart, so the needle follows bent or uneven scales).
- **Turning cards** (compass card, heading bug, RMI card): the same, the card drawn level, the angle as the value.
  A card on a card (heading bug) turns by both angles added up. A card on an uneven scale (radar altimeter DH bug)
  gets a table of points like a needle.
- **Numbers:** one picture per digit place, picked by value with a Dynamic image (`Image Dynamic="name_{VALUE}.png"`
  with a `<Value>`): pictures 0-9, a blank one (no leading zeros), a minus. Signs and words around the digits ("+",
  "%", "°", "KT") are their own pictures; a sign sits in the slot left of the first digit. "%" means value x 100.
- **Sliding parts** (ILS bars, flags): **don't use `MaskImage`**: SPAD drew our PNG masks as white circles over the
  gauge. Instead each sliding part is drawn at every position it reaches (about 2 pixels apart), each picture already
  cut to the dial window, and SPAD picks one by value (a Dynamic image). A part that hides inside a sliding part gets
  an extra blank picture.
- **Attitude horizon, HSI CDI bar:** one element with both `Shift` (value x `Scale` pixels, in the element's own
  level frame) and `Rotate` (value in **radians**, positive = clockwise, no `Nonlinearity`), around the element's
  `Position`. **A positive `Scale Y` slides the picture up**, not down, so a horizon that should go down for nose up
  needs a negative `Scale Y`. `Scale X` slides right as expected. SPAD has no
  clipping, so right after the horizon come copies of everything drawn under it with the round window cut out: they
  hide what slides outside the window.
- Growing arcs (engine "pie" dials): 60 pictures from empty to full, picked by value.

## The sim's directions (checked in flight, MSFS 2024)

- **Pitch** (`ATTITUDE INDICATOR PITCH DEGREES`, `PLANE PITCH DEGREES`): **nose up is negative.**
- **Course needle** (`HSI CDI NEEDLE`, `NAV CDI`): a positive value means the needle goes **right** (fly right).
  Whether SPAD.neXt slides the HSI's course bar the right way (`Shift` X) hasn't been checked on a FIP.
- **Attitude bars knob:** the events are `ATTITUDE_BARS_POSITION_UP` / `_DOWN`. `..._INC` / `_DEC` are only the
  SDK's internal names; the sim answers "name unrecognized".
- **Decision height knob:** `INCREASE_DECISION_HEIGHT` / `DECREASE_DECISION_HEIGHT` (1 m per click). Like the
  attitude bars, `DECISION_HEIGHT_INC` / `_DEC` are only internal names.
- **Bank** (`ATTITUDE INDICATOR BANK DEGREES`, `PLANE BANK DEGREES`): **a left bank is positive.**
- **N1/N2:** the Asobo 737 MAX display shows `TURB ENG CORRECTED N1` / `N2`; the Asobo 727 shows the plain `TURB ENG N1`.

## Finding problems

- SPAD.neXt's log: `%APPDATA%\SPAD.neXt\logs\spad.log` (its clock is UTC). It shows which gauges loaded, every event
  sent (`TransmitClientEvent ...`), events the sim rejects (`NAME_UNRECOGNIZED`) and missing pictures
  (`... missing in archive`).
- A missing picture is logged every frame: worth fixing, it also makes the gauge show the wrong picture.
- Study-level planes (PMDG and others) often use their own knob commands instead of the standard sim events.

## The 737s: where we stopped (for whoever picks it up)

- **PMDG 737:** no gauges in this set. It keeps engine numbers, autopilot panel, knobs and autobrake in its own
  data, so gauges never matched the cockpit. Known limit; not being worked on.
- **Asobo 737 MAX:** standard values work; its display shows corrected N1/N2 (see above); its knobs use its own
  commands (e.g. `B:FCC_HEADING_SEL`), not the standard heading-bug events. Not worked out.
- **SPAD.neXt percent numbers:** N1 41.5 showed as 41.0 on the FIP (decimals cut off). Not solved; first test idea:
  a number with and without `abs`, and `percent over 100` x 100 instead of `percent`.

## Change a gauge with your AI

Give your AI this file and the gauge's `.zip`. Typical asks: "the right knob should set X, here's the sim event",
"this needle reads backwards", "make the digits bigger". For bigger changes (new gauges), the FIPinator project has
the gauge maker and the exporter that made all of these.
