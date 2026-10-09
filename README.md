# VEVOR EH-720TS (E-cut / "Cutter TT-720") — how to drive it without SignMaster

Practical notes from reverse-engineering a VEVOR EH-720TS desktop cutting/creasing plotter (with ARMS camera)
on Windows, October 2026. Written so other users (and AI assistants) can find the path quickly.
Everything below was observed on one machine, firmware "HW CAM V1.4.5". Test small first.

## Connection
- USB printer class, VID `0483` PID `5750`, driver `usbprint.sys`. No COM port needed.
- Open the device path (`\\?\usb#vid_0483&pid_5750#...#{28d78fad-5a12-11d1-ae5b-0000f803a8c2}`) with
  `CreateFile` + `WriteFile` (overlapped, with a timeout). The machine **never answers** — no status, no ack.
- If the panel is in **Offline** mode or after **Reset**, writes block / fail (Windows error 31). Go back to the
  main screen first.
- SignMaster / Vinyl Spooler must be closed (exclusive access).

## Language: HPGL, 40 units/mm
- Coordinates: **X = paper feed direction, Y = carriage**. Origin = where you set it on the panel
  (Offline → arrows → «+» → X=0,Y=0 → Offline). Origin is front-right; the design arrives rotated 180°.
- Never send negative coordinates.
- Header used by SignMaster: `IN;PU0,0;PD1,1;PU0,0;` (tiny pre-cut at origin).
- Footer: `PU0,0;` then ` @ @` followed by 101 spaces.
- Tools: `SP1` = blade (head 1, blue holder), `SP2` = second head (we use it for **creasing**).
- **Speed `VS` is in mm/s** (20–400). `VS30` → panel shows "Speed 030". (Not cm/s.)
- **Force `FS` in grams** (machine accepts up to 500; we cap at 400). FS only works when VS is also sent:
  `SP1;VS80;FS200;`. The sent values are written to the panel for that head (Force1 / Force2).
- Example (crease then cut a rectangle):
  `IN;SP2;VS80;FS300;PU0,0;PD1,1;PU0,0;PU400,400;PD1200,400;SP1;VS80;FS200;PU...;PD...;PU0,0; @ @` + spaces.
- The touch screen has a speed AND force per head; tap the round icon top-left to switch Force1/Force2.

## Paper handling tips
- Use a **cutting mat (carrier)** longer than the sheet: if the job goes far from the origin the machine pushes
  the bare sheet out of the pinch rollers and cannot pull it back.
- Short creases are slow mainly because the head lifts/lowers for each line; order lines to minimise travel.
- 100 g paper: cut 200 g, crease 300 g, 80 mm/s, blade barely protruding (holder ring positions 1–3).
  Holder: 10 positions × 0.065 mm. Cardstock ≈ positions 4–6, 250–350 g.

## Print-and-cut with the ARMS camera (verified)
Format captured from SignMaster (driver file `ECut_TSTools_base.dat` in SignMaster's `System\Data`):

```
CR;<1025 spaces>SC1;MN4;TB25;<X>,<Y>;CT1;<1025 spaces>IN;PA;[SP2;FSf;VSv;<creases>]SP1;[FSf;VSv;]<cut>PU<X>,0;!PG;
```
- 4 round black marks **Ø 5 mm**, 15 mm outside the artwork, plus a small triangle on top (orientation).
- `<X>,<Y>` = distance between mark centres in units (40/mm): X along paper, Y along carriage.
- Contour coordinates are relative to the **bottom-right mark** (no +1 offset).
- Send the first part (up to `CT1;` + spaces), wait ~0.5 s, then the rest (that is what SignMaster does).
- Procedure: print at **100 %** (no "fit to page", no borderless), sheet face up, **triangle pointing into the
  machine**, put the blade tip over the **front-right mark**, set origin, leave Offline, send.
- On the camera screen (Setting → Edge Calibration, also shown during the search) the **ARMS switch must be OFF**
  for automatic reading. ON = manual mode: the machine stops at each mark and waits for «+».
- Do not change Main X / Main Y on that screen (camera-to-blade offset; ours: +46.525 / −02.875 mm).
- "Limit reminder" on screen = carriage tried to go past its travel; check sheet position/sizes.

## Things that broke our marks (and the fixes)
1. Printer shrank the page ~0.8–0.9 % (Epson ET-8550). Over 185 mm that is ~1.5 mm → camera misses the mark.
   Print a test with known distances, measure with a caliper, and pre-scale the print.
2. Marks too close to the paper edge: camera sees the edge/mat. Centre the layout on the page.
3. Resizing the design **after** printing → marks no longer match. Lock the size once printed.
4. Art and cut scaled from different reference points → offset. Scale both from the same point.
5. Using the normal "cut" command on a printed sheet (no camera) → cut lands centimetres off.
6. The crease head sits a few mm from the blade: if creases land beside the printed lines while cuts are
   correct, apply a fixed X/Y offset to the crease paths.

## Panel notes
- Setting → Factory needs a factory password (do not guess).
- Reset forgets the origin; set it again on the new sheet.
- Manual (VEVOR): 20–400 mm/s, 20–500 g, 0.0245 mm/step, HPGL, 16 MB buffer. Same USB ID is used by the VEVOR
  375/720/870/1350 family (DMPL/HPGL), so these notes may help on sibling models.

## SoftMelt Cutter — a complete replacement for SignMaster (available for licensing / sale)

Built by **Paulo Knopp** for this machine and tested on it. Windows desktop app (Python + Qt), Portuguese UI,
no subscription, no dongle. Main features, all working on the real EH-720TS:

- **Direct USB driver** (no SignMaster, no spooler), with safety checks on every job (no negative coordinates,
  force/speed limits, command whitelist) and a send history.
- **Opens SVG, DXF, PDF, PNG/JPG**; text tool (cake toppers); real-size import.
- **Cut + crease in one job** (two heads), automatic force/speed per material or manual per tool.
- **Print-and-cut with the ARMS camera** (verified): prints the art with the 4 marks on the user's printer
  (with printer scale correction, centred page, direct choice of paper tray / photo paper / quality),
  then "test marks", "crease only", "cut only" or "crease + cut" around the print.
- **Box tools**: detects the cut contour of a box artwork, deduces folds from the shape, optional **AI**
  (local Claude) to pick the real crease lines, click-by-click drawing of cut and crease lines with snapping,
  crease-head offset calibration.
- **Box generator**: choose a model (mailer box without glue, straight tuck box), type the measurements and get an
  exact die-line (cut + crease) with a live 3D preview (open/closed).
- Interactive preview (drag, resize, ruler, undo/redo, live path animation), A4/A3/custom sheet in portrait or
  landscape, paper-loss warning, test-area trace, eject, job library ("collection") with folders, autosave.
- 150 automated tests.

**VEVOR, resellers or other manufacturers** using this machine family (USB 0483:5750): the software is
available for **licensing, white-label or outright purchase**. Contact Paulo Knopp through this GitHub page
(open an "Issue") to talk.

Author: **Paulo Knopp** (tests on the real machine), with AI assistance (Claude) for the reverse-engineering.
Please credit "Paulo Knopp" when reusing these notes.

Licence: CC BY 4.0 (free to share and adapt, with credit). No warranty — you are responsible for your machine.

