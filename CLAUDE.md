# Diwali Mela

A single-page Diwali crackers web app built for HEAPS colleagues. Pick a
cracker from childhood, light it, watch it burst with real particle
physics and synthesized sound, then get a "Happy Diwali" wish.

## Stack

Plain HTML/CSS/JS. No framework, no bundler, no npm dependencies, no
build step. Everything lives in `index.html`:

- Rendering: `<canvas>` + 2D context, hand-rolled particle engine.
- Audio: Web Audio API only — every sound (hiss, bang, crackle, whirr,
  whistle) is synthesized at runtime from oscillators and filtered noise
  buffers. No audio files are loaded.
- Fonts: Google Fonts (`Baloo 2` for display, `Poppins` for body),
  loaded via a `<link>` in `<head>`.

## Running locally

No install, no build. Just serve the file and open it:

```bash
python3 -m http.server 8080
# then open http://localhost:8080/index.html
```

Opening the file directly via `file://` also works in most browsers,
but a local server is safer (some browsers restrict things like
`AudioContext` autoplay or canvas sizing on `file://`).

## Deploying

This is a static site — one HTML file, nothing else required.

**Vercel via GitHub**: push this repo to GitHub, import it in Vercel,
no framework preset needed ("Other" / static). Vercel serves
`index.html` at the root automatically — no `vercel.json` needed.

Because the file declares its own `<!DOCTYPE html>`, `<meta charset="UTF-8">`,
and full `<head>`/`<body>` structure, it renders correctly on any static
host without special server configuration. (Earlier versions of this
file were written to be wrapped by Claude's Artifact viewer, which
injects that boilerplate automatically — if you ever diff against an
artifact-hosted copy, don't be surprised that one is missing those tags.)

## App structure

Three full-screen states, crossfaded via CSS (`opacity` + `visibility`,
not `display`, so transitions are smooth):

1. **`#screen-select`** — grid of 7 cracker cards, each tagged "on the
   ground" or "goes up high".
2. **`#screen-ignite`** — a single `<canvas>` showing a night rooftop
   scene (procedurally drawn: gradient sky, stars, moon, a distant
   skyline silhouette, a foreground parapet with a water tank and
   string lights). The selected cracker sits in a fixed spot; a "Light
   it" button (real `<button>`, so Enter/Space work natively) triggers
   its animation + sound.
3. **`#screen-burst`** — "Happy Diwali!" + a randomly picked wish +
   "Light another cracker".

All three screens occupy the same absolutely-positioned box so the
crossfade doesn't cause layout jumps.

## The seven crackers

Each has its own physics and sound profile, tuned against reference
footage of the real thing (not copied code/assets — rebuilt from
scratch based on observed behavior):

| Cracker | Kind | Behavior |
|---|---|---|
| Anar | ground | Tall narrow cone; fountain height is capped to just above the skyline (not unbounded); sound is a thin, airy "whoosh" (light body + bright high-frequency hiss, intentionally no sub-bass so it doesn't overlap with Chakri). |
| Chakri | ground | Flat round disc with alternating color wedges (not thin blade lines — reads as "round" at any rotation); wanders slightly while spinning; sound has AM "flutter" modulation to read as a spinning wheel rather than a siren tone. |
| Phuljhari | ground | Handheld sparkler; continuous fine white sparks + crackle. |
| Aloo Bomb | ground | Single heavy bang — uses the `heavy` mode of the bang synth (lower sub-frequency, longer decay, higher gain ceiling) so it actually lands instead of fizzling. |
| Ladi Mirchi Bomb | ground | A short string of chili-shaped bombs that burst in sequence, each a heavy bang. |
| Rocket | sky | Launches and bursts into a randomly chosen "shell" (chrysanthemum / willow / ring / palm) — different look and particle physics every time. |
| Seven Shots | sky | A repeater: 7 staggered mini-launches at varying heights/colors. |

## Engine notes (if you're extending this)

- **Particle system**: a single `particles` array updated each frame
  (`updatePhysics`) and drawn each frame (`render`). Particles draw a
  two-segment tapering tail (current + previous two positions) rather
  than a single-frame line, so fast sparks read as comet streaks
  instead of choppy dashes.
- **Sound self-termination**: `hiss()` and `whirr()` schedule their own
  fade-out and stop directly on the Web Audio clock (`AudioParam`
  automation + `.stop(when)`), not via a JS timer or animation-loop
  callback. This was a real bug fix — the original version relied on
  the `requestAnimationFrame` loop to call `.stop()`, which meant a
  stalled tab or a slow frame could leave a sound playing forever.
  Don't reintroduce that pattern.
- **Reverb + panning**: `ensureBus()` builds one shared `ConvolverNode`
  (impulse = decaying noise) that every sound sends a portion of its
  signal to, for a sense of outdoor space. `out(node, wetAmount, pan)`
  is the single chokepoint every sound function routes through —
  extend it there, not per-sound.
- **`onDone` callbacks**: every ignite function (`igniteAnar`,
  `igniteChakri`, `launchRocket`, etc.) takes an `onDone` callback,
  invoked once its full animation (including any afterglow/fade) has
  settled. `runIgnite()` passes `finishToMessage` as `onDone` so the
  crossfade to the wish screen only happens once the effect actually
  looks finished, not the instant the physics loop says "done".
- **Accessibility**: `prefers-reduced-motion` is respected — it skips
  straight to a quick bang + message instead of the full animation, and
  also drives the default state of the in-app "Reduce flashes" toggle
  (which halves particle counts, dims brightness, disables camera
  shake, and rate-limits the white bomb-flash to twice a second).

## Things that were tried and removed

- **A "suggest a cracker" feature** backed by Claude's Artifact `db`
  capability (`window.claude.use('db')`) was built, then removed. That
  API only exists inside Claude's Artifact viewer — it does not exist
  on a self-hosted static deploy (Vercel, GitHub Pages, etc.), so it
  would have silently done nothing once deployed here. If this feature
  comes back, it needs a real backend (Supabase was the recommended
  option — free Postgres + a JS client that can run directly from this
  static page with no server code, and unlike a plain form-to-email
  service it supports reading back a public list of past suggestions).
- **A fully free-tap "simulator" mode** (pick a cracker, tap anywhere on
  an open scene to place it, with a multiplier and a persistent
  "crackers lit" counter) exists as a separate build, inspired by
  Diwali firework simulator apps/sites. It is a different interaction
  model from this file's "select → light → message" flow and was kept
  as a separate artifact rather than merged in, since the two flows
  don't coexist cleanly in one file.
