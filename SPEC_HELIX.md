# ButterflyDreaming Helix Module — SPEC_HELIX.md
*Started: May 2026*

## Overview

This module is a new BD media module that renders L-system grammars as 
three-dimensional helical structures using Three.js. It shares the same 
%%bd_ directive system, BD messaging protocol, and score block format as 
the 2D visual module (VISUAL_MODULE_SPEC.md) and music module 
(MUSIC_MODULE_SPEC.md). Claude Code should read all three spec files 
before beginning any implementation work.

The key architectural concept is that the same L-system grammar that 
produces a 2D Kolam pattern in the visual module here drives a 3D helical 
structure — the turtle path gains a vertical displacement component so that 
each forward step rises by a small increment dZ. Rotational symmetry stamps 
stack vertically rather than resetting to zero, producing a corkscrew or 
helix form. The camera is positioned on the central vertical axis looking 
outward through the structure, so the user experiences the pattern from 
inside the helix.

The module is designed with WebXR in mind from the outset so that VR 
support can be added in a later phase without architectural changes.

---

## BD Protocol

This module implements the standard BD messaging protocol as defined in 
VISUAL_MODULE_SPEC.md:

- **BD_READY** — posted to parent when module is loaded and initialised
- **BD_INIT** — received from parent, contains node text with %%bd_ 
  directives and score block
- **BD_UPDATE** — posted to parent in response to BD_REQUEST_UPDATE, 
  contains current node text with all directive values written back from 
  current control state
- **BD_REQUEST_UPDATE** — received from parent, triggers BD_UPDATE response
- **BD_STOP** — received from parent, clears the scene
- **BD_ERROR** — posted to parent if rendering fails

---

## Directives

### Inherited from visual module (same syntax, same defaults)
- `%%bd_depth` — L-system iteration depth (1–5, default 4)
- `%%bd_step` — base forward step size (10–200, default 40)
- `%%bd_angle` — turn angle in degrees (5–90, default 45)
- `%%bd_angle_minutes` — fine angle adjustment in minutes of arc (0–59, 
  default 0)
- `%%bd_colour_speed` — hue cycling rate relative to accumulated angle 
  (0.25–16, default 4)
- `%%bd_saturation` — HSL saturation 0–100 (default 100)
- `%%bd_lightness` — HSL lightness 0–100 (default 65)
- `%%bd_symmetry` — rotational symmetry count (1,2,3,4,6,8,10,12,16, 
  default 8)
- `%%bd_background` — background colour (default #0a0a0f)

### New directives specific to helix module
- `%%bd_vertical_step` — vertical rise per forward turtle step, as a 
  fraction of the horizontal step size (0.0–1.0, default 0.1)
- `%%bd_tube_radius` — radius of the tube geometry along the turtle path 
  (0.01–0.5, default 0.05)
- `%%bd_tube_segments` — radial segments on tube cross section (3–12, 
  default 6) — lower values give angular faceted look, higher gives smooth
- `%%bd_camera_height` — vertical position of camera on central axis as 
  fraction of total helix height (0.0–1.0, default 0.5) — 0 is base, 
  1 is top
- `%%bd_camera_radius` — distance of camera from central axis (0.0–10.0, 
  default 0.0) — 0 places camera exactly on axis
- `%%bd_rotation_speed` — slow auto-rotation speed of the scene around 
  vertical axis (0.0–2.0, default 0.3)
- `%%bd_stack_mode` — how symmetry stamps accumulate vertically: `stack` 
  (each stamp continues from where previous ended) or `reset` (each stamp 
  returns to vertical zero). Default: `stack`

---

## Score Block

Identical format to the visual module. The same L-system grammars work 
without modification. Example default score:
%%bd_module helix_module.html
%%bd_depth 3
%%bd_step 40
%%bd_angle 25
%%bd_angle_minutes 0
%%bd_vertical_step 0.15
%%bd_tube_radius 0.05
%%bd_tube_segments 6
%%bd_symmetry 8
%%bd_colour_speed 4
%%bd_saturation 100
%%bd_lightness 65
%%bd_background #0a0a0f
%%bd_camera_height 0.5
%%bd_camera_radius 0.0
%%bd_rotation_speed 0.3
%%bd_stack_mode stack
%%bd_score [
axiom: F+F+F+F+F+F+F+F
F: F+F-F-F+F+F+F-F
%%bd_]

---

## Rendering Architecture

### Three.js
Use Three.js (r128 or later) loaded from CDN. Do not use OrbitControls 
from the Three.js examples as these are not reliably hosted on CDN — 
implement camera control directly.

### Turtle in 3D
The turtle operates in 3D space. Each symbol is interpreted as follows:

- **F** — move forward by effective step in XY plane, rise by 
  `vertical_step × effective_step` in Z. Draw tube segment.
- **f** — move forward without drawing
- **+** — turn left by effective angle (yaw in XY plane)
- **-** — turn right by effective angle (yaw in XY plane)
- **[** — push turtle state (position, angle, Z height)
- **]** — pop turtle state
- **|** — turn 180 degrees

### Tube geometry
Collect all turtle path points into an array, then build a 
`THREE.TubeGeometry` along a `THREE.CatmullRomCurve3` through those points. 
Apply colour per vertex based on accumulated angle at each point, matching 
the colour_speed hue cycling logic from the visual module.

### Symmetry stacking
Render the turtle path once to get a set of 3D points and colours. Then 
stamp n copies of that geometry, each rotated by `(2π/n × i)` around the 
Z axis. In `stack` mode each stamp is offset vertically by 
`(i × total_Z_rise_of_single_stamp)`. In `reset` mode all stamps share 
Z=0 as their base.

### Camera
Position the camera at `(camera_radius, 0, camera_height × total_height)` 
looking outward (away from central axis) or upward depending on 
`camera_radius`. Auto-rotate the entire scene around the Z axis at 
`rotation_speed` degrees per second. Allow mouse drag to tilt the camera 
vertically.

### WebXR
Include a WebXR session request button from the outset. In non-VR mode 
it is hidden or greyed out. When WebXR is available the button activates 
and enters an immersive-vr session. Camera position in VR follows the 
same parameters but head tracking overrides tilt.

---

## File Structure
butterflydreaming_helix_1/
index.html              — host page with textarea, Send, Receive,
Copy, Copy Link buttons
helix_module.html       — Three.js renderer, BD protocol handler
SPEC_HELIX.md           — this file
VISUAL_MODULE_SPEC.md   — reference (copied from 2D module)
MUSIC_MODULE_SPEC.md    — reference (copied from music module)

---

## Implementation Notes

- Claude Code should read VISUAL_MODULE_SPEC.md and MUSIC_MODULE_SPEC.md 
  before beginning — the BD protocol, directive parsing pattern, and 
  messaging architecture should be inherited directly, not reimplemented
- The %%bd_ parser from visual_module.html can be copied verbatim
- The index.html should follow the same layout as the 2D visual index — 
  textarea left, player right, with Copy/Copy Link/Receive from Player 
  buttons
- Depth should default to 3 not 4 for the helix — the 3D geometry is 
  more complex and higher depths will be slow to render
- Start with line geometry first (THREE.Line) to verify the algorithm 
  before switching to tube geometry — tubes are more expensive and harder 
  to debug

---

## Amendments

*(Amendments to be added here as development proceeds)*