# AstroXR

An AR planetarium prototype for Meta Quest 3 (WebXR). Real survey imagery of deep-sky objects is placed on the
sky at its **true angular size**, over passthrough, with the Moon and Sun tracked live so you can align the
overlay by looking at the real Moon and pulling the trigger.

Single page: `index.html` + `assets/`. Runs in the Quest Browser (passthrough AR or VR) and in any desktop
browser (mouse-look) for development. Two small tools rebuild the object library (below).

## Vision

AstroXR and **ScopeControl** are two ends of one system. ScopeControl already runs the rig: mount GoTo,
plate-solve → sync → centre, capture, calibration, PHD2 guiding, EAF autofocus and server-side live stacking,
all behind one JSON service on the capture PC. AstroXR puts the sky itself in front of the observer. Joined up:

- **Point with your eyes.** Look at a target in the headset, confirm, and the mount slews there; the scope's
  current field is drawn on the real sky as a box (ScopeControl's blue scope box, lifted off the chart).
- **Frame in place.** The imaging FOV box rides on the sky at true scale; drag it onto a nebula and send GoTo.
- **Watch the stack build.** ScopeControl's live-stack preview appears in the headset beside the target, at
  the same angular scale as the sky around it, updating as subs arrive — the picture emerging *where it is*.
- **One service, two clients.** AstroXR needs nothing new from the capture PC beyond what the Tk client
  already uses: `ascom_status`, `ascom_goto`, `ascom_sync`, `ascom_stop`, `optics`, the solve/centre loop and
  the stack preview endpoint. A `token` and the LAN address are the only configuration.

The pieces this prototype already shares with ScopeControl: the OpenNGC catalogue, hips2fits imagery, the
named-star and constellation data, the label-culling rules, and the alignment problem — which ScopeControl
solves on the mount with plate solving and AstroXR will solve on the headset with a camera.

## What's in it

- **Sky base**: Mellinger Milky Way panorama (CDS HiPS `CDS/P/Mellinger/color`) rendered by hips2fits as a
  4096×2048 equirectangular map, sampled by direction in a shader and drawn additively so passthrough shows
  through the black sky.
- **210 deep-sky objects**: the 13 featured cut-outs from `../DSO-vs-Moon` (5 px/arcmin, curated fields) plus a
  library of ~300 — every Messier object, every sizeable NGC/IC nebula, a curated Sharpless set (Clamshell,
  Flying Bat, Cave, Lobster Claw, Tulip, Spaghetti, Barnard's Loop…) and the brightest clusters/galaxies — selected from
  ScopeControl's OpenNGC catalogue by `tools/build_library.py` and fetched from hips2fits (DSS2 colour) with a
  field of 1.4× the catalogued major axis. Each is a quad whose angular size equals its field, north-up/east-left,
  labelled with name, designation, type and size. Search box and "up now" filter in the strip.
- **Streamed DSS2 sky (HiPS)**: the whole sky is DSS2 colour, streamed as the HEALPix tiles Aladin uses, at
  **orders 2 to 8** (103″/px down to 1.6″/px) with the active orders following the field of view — so zooming in
  resolves finer rather than magnifying. Only three levels are ever live: the order the view justifies, one below
  it as backing while tiles arrive, and order 2 for the rest of the sky; a tile hides once all four of its
  children are in. Deep tiles are discovered by walking the tree down from order 3 with a cone cull at each step
  (order 8 has 786k tiles, so no registry is built, and subdividing only *loaded* parents would stall because the
  intermediate orders are skipped). Load cones follow the field of view, and a click or zoom step prefetches at
  once with a wider in-flight limit. Tile geometry is the HEALPix nested pixel → (face, x, y) → vector mapping,
  an 8×8 curved mesh per tile; the image orientation (column = HEALPix y, row = x) was calibrated against M31,
  M42 and M13.
- **Plate artefacts**: DSS2 is photographic, so it carries satellite trails, emulsion scratches and dust. Its
  colour comes from blue, red and near-IR plates exposed at *different times*, so an artefact lands in a single
  channel and renders as a vividly saturated streak while real sky stays desaturated — the shader pulls extreme
  chroma toward luminance, which suppresses them without any detection and without touching luminance. Toggle in
  Settings. (Reliable trail detection needs a trained network or several exposures of the same sky; neither suits
  a browser streaming tiles.)
- **Regional survey plates**: 18 large DSS2 colour fields (Cygnus, Cepheus, Cassiopeia, Auriga, California,
  Orion, Monoceros, Sagittarius, Ophiuchus, Carina, the Magellanic Clouds…) at 1.5 px/arcmin, drawn under the
  cut-outs so nebulosity is continuous across the rich areas (`tools/build_regions.py`). Every plate and cut-out
  is drawn through one shader: its own sky level subtracted, a mild stretch, feathered edges — so imagery melts
  into the base sky rather than sitting on it as boxes. Field outlines are off by default.
- **Stars and constellations**: ScopeControl's 493 named stars and the 88 constellation figures (d3-celestial).
- **Labels** follow what planetarium and map engines do: a budget tied to the field of view (wide views name only
  the famous things; narrow views open up), library names only near the reticle, biggest first; collision boxes
  on the text itself, placed in priority order with three candidate anchors (below, right, above); recently shown
  labels keep priority and everything fades in/out, so nothing flickers; whatever the reticle is on is always
  named. Both the text size **and every offset** scale with the field of view — a fixed angular offset is a few
  pixels at 70° but most of the screen at 1°, which leaves zoomed-in names stranded away from their objects.
- **Moon and Sun**: dashed 30′ rings at the topocentric position from
  [astronomy-engine](https://github.com/cosinekitty/astronomy); the Sun ring hides below the horizon.
- **Bright stars and planets**: ~50 named stars (Hipparcos, J2000) and the five naked-eye planets as markers,
  so there is always something to align on and the sky is recognisable.
- **Alignment on Polaris**: the Quest has no compass, so the sky's yaw is unknown. Polaris is the one reference:
  it never moves, so one alignment holds all night, and its altitude equals your latitude. It carries a standing
  orange ring; put the reticle on the real star and pull the trigger / pinch / press Align, and the sky rotates
  so the computed Polaris lands on it. The top bar shows "Aligned on Polaris · 3 min ago" and nags after 10 min.
- **Selection and progressive zoom**: pick an object in the strip, by search, by clicking it, or by clicking its
  **name** — every route behaves the same. The view turns to it and zooms to frame it; **+ / − / Return** (and the
  same keys, plus the wheel) drive the zoom from there, with the current field shown between them. The hit test
  picks the *smallest* object whose own extent contains the click, not the first quad the ray meets, so a click on
  empty sky near M31 does not select M31; clicking an object already chosen steps the zoom in, and empty sky,
  Return or Esc clears. A chevron at the edge of the view points the way to an off-screen target.
- **The sky freezes past a 15° field**, because beyond that the view is magnified past real scale and the sky's
  15°/hour rotation only shows as the picture sliding off screen. Button (and `F`) overrides; the clock says when
  it is frozen; never applies in AR, where the overlay must keep matching the real sky.
- **Imagery is natural colour**, since the point is to show the sky as it looks: PanSTARRS DR1 where it is
  sharper (objects under 16′, north of dec −29) and DSS2 colour otherwise, each candidate checked for usable
  pixels first because hips2fits returns a blank frame outside a survey's footprint. Researched and rejected:
  **DESI Legacy DR10** is deeper but its g/r/z composite renders galaxies teal; the **NSNS narrowband survey**
  (65% of sky, CC BY-NC-SA) shows far more nebula structure and M82's Hα outflow, but in an [OIII]/Hα/[SII]
  palette rather than true colour. M82's red cannot appear in any broadband survey — it is line emission.
  Curated press imagery from the **AAS WorldWide Telescope** study collections was also tried and reverted:
  coverage and framing varied too much object to object (builder in git history; note WWT's amateur
  "astrophoto" collection is All Rights Reserved).
- **Horizon and N/E/S/W** cue in the gravity-aligned frame; sky-brightness slider for light-polluted passthrough.
- **Desktop UI**: slim top bar (Enter AR, alignment state), collapsible settings drawer, object strip along the
  bottom (dimmed when below the horizon), drag to look, wheel to zoom, `A` to align, `?look=`/`?hold=`/`?fov=` URL
  params for testing.

## Phones (no WebXR)

iOS Safari and most phone browsers have no WebXR, so **Enter AR** cannot work there. The **Phone AR** button
(shown on touch devices) does what Star Walk / Sky Guide do instead: the gyroscope and compass steer the view
("magic window"), the rear camera shows behind the sky, and the compass gives a first north — put Polaris in the
reticle and tap Align to refine. iOS asks for motion permission on the tap. Yaw is driven by the gyro; the
magnetometer only sets north once (circular mean of its first ~1.5 s), because it wobbles several degrees
indoors. Needs HTTPS **and a top-level page**: inside a viewer iframe (e.g. the claude.ai artifact) motion,
camera and WebXR are blocked — for testing, `cloudflared tunnel --url http://localhost:8765` gives a
throwaway HTTPS URL; for real use enable GitHub Pages on the repo.

## Coordinates

The `sky` group's local frame is J2000 equatorial (x → RA 0°, z → NCP). Each frame it is rotated by
`Astronomy.Rotation_EQJ_HOR(time, observer)` (HOR: x north, y west, z zenith) mapped onto three.js world
(x east, y up, −z north), then by the user's yaw offset about the vertical. Everything is drawn at 50 m so head
movement is irrelevant; the sky and horizon follow the eye position.

## Live site

**https://jr06410.github.io/astroxr/** — GitHub Pages from the deploy-only repo `JR06410/astroxr`
(this folder is the source; `python tools/deploy_pages.py` mirrors it there and pushes). Top-level HTTPS, so
Phone AR, Enter AR and the camera all work. The claude.ai artifact is a desktop preview only.

## Running

- **Headset**: open the published HTTPS URL in the Quest Browser and press **Enter AR**. WebXR needs HTTPS.
  Locally: `npx serve` here plus an HTTPS tunnel (e.g. `ngrok http 3000`), or enable GitHub Pages on the repo.
- **Desktop**: any static server (`npx serve`), open `index.html`.

Dependencies load from CDN: three.js 0.160 (cdnjs) and astronomy-engine 2.1.19 (jsdelivr).

## Next steps

1. Passthrough Camera API: IMU-constrained bright-object solve (Moon/planets/mag-3 stars) to replace manual alignment.
2. Hand-tracked "pull forward" interaction: select an object, bring it close and large with its data.
3. Stream higher-order HiPS tiles for the gaze direction instead of a single all-sky texture.
4. Port to Unity + Meta XR SDK for the store build; the content and astronomy layer carry over unchanged.

## Rebuilding the library

```
python tools/build_library.py 220   # select from ../ScopeControl/.../dsos.json, fetch assets/library/*.jpg
python tools/inline_data.py         # inline library + stars + constellations into index.html
```

## Credits

Mellinger Milky Way panorama (Axel Mellinger) and DSS2 (© STScI, AAO/UKSTU, Palomar/Caltech) via CDS hips2fits
("This research made use of hips2fits, a service provided by CDS"). Object catalogue: OpenNGC by Mattia Verga
(CC-BY-SA-4.0) via ScopeControl. Stars and constellations: d3-celestial datasets (BSD). Astronomy by astronomy-engine (MIT). Rendering by three.js (MIT).
