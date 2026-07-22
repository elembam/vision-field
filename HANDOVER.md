# Copilot Handover — Foveal/Peripheral Vision Field Tools

## What this project is

A small suite of standalone HTML/JS tools (no build step, no framework, no
backend) that model **human visual acuity falloff with eccentricity** and
apply it to real-world scenarios: sizing a painting on a wall, comparing it
to a printed reference, and — the newest piece — **actually filtering an
image** so it looks the way it would if you were fixating on one point of it,
sharp at the center and progressively blurred outward.

The end goal (this handover exists to move toward it) is a **mobile web app
that does this live, accurately, and useably** — something a person can
point at a painting, a room, or a photo and get a scientifically-grounded
visualization of what their visual system actually resolves at each point,
calibrated to the real camera/lens rather than a guessed field of view.

Three files exist today, in increasing order of ambition:

| File | What it does | State |
|---|---|---|
| `vision-canvas/index.html` | Calculator: sightline/projection diagrams comparing a near plane (e.g. a printed page) and a far plane (e.g. a canvas on a wall), plus face-on zone-circle overlays | Done, working |
| `vision-camera/index.html` | Live camera feed with acuity-zone rings overlaid, manual distance slider, tap-to-fixate, one-time calibration against a known-size object | Done, working, needs HTTPS hosting to run (camera API requirement) |
| `vision-foveate/index.html` | **Static image foveation filter** — take/choose a photo, set fixation, calibrate, and render an actual blurred-per-zone output image | Done, working, this is the one to build out further |

This handover focuses on `vision-foveate/index.html`, since that's the one
the user wants developed into "a powerful image filter web app for
foveal/peripheral field of vision analysis on a mobile phone."

---

## The core science (same in all three files — don't reinvent it)

Human visual acuity is not uniform across the visual field. It falls off
with eccentricity (angular distance from the point you're fixating on) in a
well-characterized way, sometimes called **cortical magnification**:

```
cutoff_spatial_frequency(e) = foveal_acuity / (1 + e / E2)
```

- `e` = eccentricity in degrees from the fixation point
- `E2` ≈ 2.3° (a commonly cited constant in foveated-rendering literature —
  the eccentricity at which acuity has halved)
- `foveal_acuity` ≈ 60 cycles/degree (roughly 20/20 vision)

This is the same falloff curve used in real foveated-rendering research (VR
headset GPUs render less detail in the periphery using this exact model).
Currently `vision-foveate` **discretizes** this continuous curve into six
bands (fovea / parafovea / macula / near-peripheral / comfort / beyond),
matching the same zone boundaries used in the other two tools:

```js
const BANDS = [
  {inner:0,  outer:1,    key:'fovea'},
  {inner:1,  outer:2.5,  key:'para'},
  {inner:2.5,outer:9,    key:'macula'},
  {inner:9,  outer:15,   key:'peri'},
  {inner:15, outer:20,   key:'comfort'},
  {inner:20, outer:9999, key:'beyond'},
];
```

Each band gets a single blur radius (`sigma`, in pixels) computed from its
midpoint eccentricity, and bands are composited from most-blurred (outer) to
sharpest (fovea) with a soft radial-gradient feather between them so the
transitions aren't hard edges. This is a reasonable **approximation** — the
real thing is a continuous gradient, not six discrete rings — and that's the
single biggest thing worth improving (see Priority 1 below).

## The calibration approach (this is the clever bit, keep it)

The web platform gives no access to a camera's real focal length or field of
view. Instead of guessing, the tool uses a **classic pinhole-camera
calibration**: the user marks the two edges of something with a known
real-world width (an A4 sheet, a credit card) at a known distance, and:

```js
focalLengthPx = pixelWidthOnScreen * knownDistanceCm / knownWidthCm
```

That `focalLengthPx` is then the single number that converts any angle to a
pixel radius for that specific photo:

```js
radiusPx = focalLengthPx * Math.tan(angleRadians)
```

This is exact pinhole-camera geometry, not an approximation, and it's the
right primitive to build everything else on top of. **Do not replace this
with a hardcoded "iPhone FOV" constant** — different lenses (main/ultrawide/
telephoto), different phones, and even different photo resolutions all
change this number, and calibration is what makes the tool trustworthy
rather than illustrative.

---

## Current file structure

```
vision-foveate/
  index.html    — everything: HTML, CSS, and JS in one file, no dependencies
                  besides a Google Fonts import (IBM Plex Mono/Sans Condensed)
```

Inside `index.html`, the flow is:

1. **Load an image** — `<input type=file accept=image/* capture=environment>`
   for camera capture, or a second input without `capture` for the photo
   library. Both funnel into `loadImageFile()`, which downsizes to a working
   resolution (max 1200px wide) for performance and stores it in an offscreen
   `srcCanvas`.
2. **Fixation** — tap anywhere on the displayed canvas sets `fixation = {x,y}`
   in working-resolution pixel coordinates.
3. **Calibration** (optional but recommended) — drag two on-screen handles
   onto a reference object's edges, enter its real width + distance, compute
   `focalLengthPx`, store in memory (not persisted — see Priority 3).
4. **Render** — `applyBtn` click handler builds one blurred copy of the
   source per band (`blurredCopy()`, using `ctx.filter = 'blur(Npx)'`, which
   is well-supported in Safari), masks each with a radial gradient
   (`createRadialGradient` + `destination-in` compositing) centered on the
   fixation point, and composites from outside in.
5. **Save** — `canvas.toBlob()` → object URL → synthetic `<a download>` click.

No frameworks, no build tools, no bundler. This was a deliberate choice for
portability (it's meant to be pasted directly into GitHub's web-based file
editor and deployed via GitHub Pages with zero tooling) — **that constraint
may or may not still apply going forward; ask the user whether they want to
keep the zero-build-step property or whether introducing a real toolchain
(Vite, npm, etc.) is now acceptable**, since several of the improvements
below get meaningfully easier with one.

---

## Roadmap: turning this into a real mobile image-filter app

Roughly in priority order — earlier items unlock or clarify later ones.

### 1. Replace discrete bands with a continuous per-pixel blur

This is the single highest-value change. Right now the filter looks like
concentric rings of blur, softened at the edges but still visibly banded.
The real visual system doesn't have six steps — it's a smooth gradient.

The clean way to do this on the web is a **WebGL fragment shader**: pass the
image in as a texture, compute each pixel's eccentricity from the fixation
point (using the same `focalLengthPx`/`atan` math already in the codebase),
look up the cortical-magnification cutoff frequency for that eccentricity,
and apply a variable-radius blur — most efficiently via a **mipmap-based
approach** (sample from a blurred mip level chosen per-pixel by eccentricity,
similar to how foveated renderers in VR actually do this) rather than a true
spatially-varying convolution, which would be far too slow per-pixel.

A pragmatic middle ground if a full shader rewrite is too much right now:
increase the band count from 6 to ~16–20 with smaller steps between them —
cheap to do (just extend the `BANDS` array and tune `sigmaForBand`), and the
visual banding becomes much less noticeable even though it's still discrete
under the hood.

### 2. Live video, not just still photos

The user's stated goal is analysis "on a mobile phone" in a way that feels
immediate — the static-photo version was a deliberate first step (see prior
conversation: static was chosen over live specifically because live
performance was uncertain), but live video is the natural next target now
that the static approach is proven out.

This needs the WebGL approach from #1 rather than the current Canvas2D
`ctx.filter` compositing loop, which re-blurs and re-composites the entire
image per band per frame — fine once, not fine at 30-60fps. A single
fragment shader evaluated once per pixel per frame is the only realistic way
to hit real-time rates on a phone GPU.

There's already a **live camera overlay** tool in this project
(`vision-camera/index.html`) that draws *rings* over live video — it does
NOT currently filter the video itself, just annotates it. Merging that
project's camera-handling code (`getUserMedia`, HUD, calibration UI pattern)
with this project's *filtering* logic is the natural integration path.

### 3. Persist calibration (and fixation defaults) across sessions

Right now, calibration is in-memory only — closing the tab loses it. Since
a given phone's camera focal length doesn't change, this should be a
**one-time setup step**, not a per-session one. Use `localStorage` (this is
a real standalone web app the user hosts themselves, not an embedded
preview — `localStorage` is fully appropriate here, unlike in sandboxed
artifact previews). Store per-camera-mode profiles (main/ultrawide lens
produce different focal lengths) if the app grows to support lens
switching.

### 4. True PWA — manifest + service worker

The current meta-tag approach (`apple-mobile-web-app-capable`, etc.) gets
"Add to Home Screen" full-screen behavior on iOS, but a real
`manifest.json` + a service worker caching the app shell would make it:
- Installable with a proper name/icon prompt (not just Safari's generic
  behavior)
- Fully reliable offline (currently the Google Fonts `@import` requires a
  network hit on first load — self-host the two font files instead so the
  app works from a dead network on first launch too)
- Update-checkable (the service worker can detect a new deployed version
  and prompt the user to refresh)

### 5. Auto-detect the fixation point (optional, bigger lift)

Currently fixation is always manual (tap to set). A meaningfully more
"powerful" version could use the front-facing camera with a lightweight
on-device gaze/face-orientation model (e.g. via **TensorFlow.js** or
**MediaPipe's Face Landmarker**, both of which run client-side in-browser
with no server round-trip) to estimate roughly where on the rear-camera
image the user is actually looking, rather than assuming center or requiring
a manual tap every time. This is a real research-grade feature, not just
polish — flag it to the user as a distinct, larger scope item rather than
bundling it into a "quick improvement."

### 6. Side-by-side / before-after comparison export

For the user's stated purpose (visual-field *analysis*, not just a fun
filter), being able to export or view original-vs-foveated side by side —
or as a slider/wipe comparison — would make the tool useful for actually
presenting findings (e.g. to explain why a panoramic canvas reads as
immersive). This is comparatively low effort: duplicate the canvas, lay out
two `<canvas>` elements or one canvas with a draggable clip-path divider.

### 7. Multiple fixation points / scan-path simulation

Real viewing isn't one static fixation — the eye saccades across a scene
sampling several fixation points in sequence (this was actually discussed
explicitly with the user earlier in this project, re: how a painting is
"read" through eye movements, not one static gaze). A future version could
let the user place several fixation points with dwell-time weights and
render a composite that approximates cumulative perceived sharpness across
a realistic scan path, rather than a single frozen gaze. This is a research
feature, likely lower priority than 1–4, but worth keeping in mind since it
connects directly to the motivating idea behind the whole project.

---

## Things NOT to change without discussion

- **The pinhole calibration formula.** It's correct and it's the backbone
  of every other tool in this project. If a future feature needs a
  different focal-length estimate (e.g. reading EXIF data from a photo,
  which sometimes contains real focal length + sensor size), that should
  *supplement* the manual calibration as a convenience default, not replace
  it — EXIF focal length is in mm, not px, and still needs the sensor's
  physical size (rarely present in EXIF) to convert, so it's not a drop-in
  replacement for the on-screen calibration.
- **The E2 = 2.3° / 60 cycles-per-degree constants.** These are standard
  literature values, not arbitrary tuning knobs — if they need to change,
  it should be because of a specific citation, not visual taste.
- **Zero-build-step architecture** — don't introduce npm/bundlers/frameworks
  without first checking whether the user still wants to be able to
  copy-paste the file directly into GitHub's web editor (their current
  deployment method, chosen specifically because file-upload tooling was
  unavailable in an earlier session).

## Related files in this project (context, not in scope for this task)

- `vision-canvas/index.html` — the projection/sightline calculator. Shares
  the same `ZONES` array and angle-to-pixel math conventions; if you touch
  those constants in one file, check whether they've drifted from the
  others.
- `vision-camera/index.html` — live camera + ring overlay (not filtering).
  This is the file to reference for `getUserMedia` handling, HUD/panel UI
  patterns, and the calibration-drag-handle UI, all of which are more
  mature there than the copy currently in `vision-foveate`.
