# Project Context — Palm Slash

Technical context for anyone (human or AI assistant) picking up this codebase. The
[README.md](README.md) is written for players; this file is written for whoever has to
change the code.

---

## 1. What this is

A webcam-controlled Fruit Ninja clone. MediaPipe tracks the player's hands, the index
fingertips become blades, and fruit falling down the screen get sliced by swiping through
them in the air. Runs entirely client-side in the browser — no server, no accounts, no
data leaves the machine.

- **Live:** https://shivamvkapadia.github.io/PalmSlash/ (GitHub Pages, served from `main`)
- **Current version:** v1.1 — two-handed play, power slices, pinch precision, difficulty ramp
- **License:** MIT ([LICENSE](LICENSE))

## 2. Repository layout

```
index.html   the entire game — markup, CSS, and JS in one file (~1100 lines)
README.md    player-facing docs: how to play, how to run, how it's built
CONTEXT.md   this file
LICENSE      MIT
```

There is no build step, no `package.json`, no dependencies installed locally. Both
libraries load from a CDN at runtime.

## 3. Hard constraints (read before changing anything)

- **Single file, no build.** `index.html` is deliberately self-contained so it can be
  dropped on any static host and opened. Don't introduce a bundler, a framework, or a
  module split without a strong reason — it breaks the one property the project is
  designed around.
- **Camera requires a secure context.** `getUserMedia` is blocked on `file://`. The game
  must be served over `http://localhost` or `https://`. Double-clicking the file gives a
  black background and no tracking.
- **Everything runs on the main thread.** Model inference and rendering compete for the
  same thread. Any work added to the draw loop directly costs frame rate, and frame rate
  is the whole feel of the game.

## 4. Runtime dependencies

| Dependency | Version | Loaded as | Used for |
|---|---|---|---|
| p5.js | 1.9.0 | `<script>` from jsDelivr | canvas, draw loop, fruit/particle/blade rendering |
| MediaPipe Tasks Vision | 0.10.14 | dynamic `import()` from jsDelivr | `HandLandmarker` (GPU delegate), + its WASM fileset |
| `hand_landmarker.task` | float16/1 | Google Cloud Storage | the tracking model itself (a few MB, first-load cost) |
| Google Fonts | — | stylesheet | Press Start 2P + VT323 (the pixel UI) |

All four are external. The game degrades rather than crashes: if the model fails to load
the video still shows and fruit still fall, you just can't slice.

## 5. Code map (`index.html`)

| Lines | Section | Notes |
|---|---|---|
| 1–357 | `<head>` + CSS | pixel design system: palette tokens in `:root`, frames, HUD, overlays, buttons |
| 359–421 | DOM | canvas mount `#wrap`, `#cam` video (hidden, texture source only), `#hud`, start/over overlays, `#status` |
| 425–500 | Global state & tuning | game state machine, entity arrays, `FRUITS` table, `BOMB` config |
| 513–572 | p5 sketch | `setup` / `windowResized` / `draw` — draw order lives here |
| 582–762 | Game logic | difficulty, spawn, slice, lives, blade-vs-fruit collision |
| 767–941 | Rendering | fruits, particles, popups, splashes, blade trails, cursors |
| 944–1007 | HUD | score/lives DOM updates, the pixel-heart sprite, screen-flash effects |
| 1010–1036 | Game flow | `startGame` / `endGame` / `setStatus` |
| 1040–1223 | Vision | coordinate mapping, One-Euro filter, pinch, landmark handling, detect loop, `boot()` |
| 1226–1237 | Wiring | button listeners |

### The pixel UI layer
Everything outside the canvas — HUD, overlays, scoreboard, buttons — is 8-bit arcade
chrome, and it follows three rules that are worth keeping:

- **Nothing blurs.** Every frame is built from *un-blurred* `box-shadow` rings: a bright
  bezel (`--panel-2`), then a black outline (`--ink`), over a dark well. Blur radii and
  `backdrop-filter` are what the old theme used, and both cost the compositor real time on
  a page that is already fighting for the main thread. Hard shadows are free and they're
  the only way the edges stay pixel-crisp.
- **Nothing eases.** Transitions and animations use `steps()`, so state changes snap the
  way a sprite does. The old full-screen animated gradient mesh is gone for the same
  reason as the blur — it repainted the whole viewport forever, for decoration.
- **Two fonts, both bitmap.** `Press Start 2P` for anything numeric or label-like (score,
  best, lives, buttons, titles, the in-canvas "+N" popups) and `VT323` for body copy,
  which stays readable at paragraph length. Press Start 2P only covers ASCII — that's why
  the `#status` strings and `#newHi` are plain ASCII rather than `…`, `⚠` or `★`, which
  would drop to a fallback font mid-word.

The camera feed, fruit, particles and blades are drawn by p5 and were deliberately left
alone — the pixel work stops at the canvas edge.

### State machine
`State = { START, PLAYING, PAUSED, OVER }` in `gameState`. `PAUSED` is defined but not
currently reachable — no pause is wired up. `draw()` always renders the world so the
game-over screen sits over a frozen frame; only `updateGame()` is gated on `PLAYING`.

### Entities
Four flat arrays — `fruits`, `particles`, `splashes`, `popups` — each iterated backwards
and spliced in place. A sliced fruit becomes a fading `isGhost` husk plus two `isHalf`
pieces that fly apart; only whole, unsliced, non-ghost fruit count toward the on-screen
cap.

## 6. The tracking pipeline (the part that's easy to break)

```
webcam frame
  └─ HandLandmarker.detectForVideo()   throttled to ~22Hz (DETECT_INTERVAL = 45ms)
       └─ onLandmarks()                landmark 8 (index tip) → raw target per hand
            └─ videoToScreen()         reach expansion → object-fit:cover → mirror
                 └─ updateHands()      60fps: One-Euro smoothing + velocity prediction
                      └─ blade trail + slice tests
```

Four things here are load-bearing, and each exists because of a specific failure:

1. **Detection is throttled, smoothing is not.** Inference blocks the render loop while it
   runs. Running it every frame tanks the frame rate; running it at ~22Hz and letting the
   60fps One-Euro filter interpolate between samples looks identical and leaves room to
   render. Raising the detection rate is the fastest way to make the game feel worse.
2. **`videoToScreen()` must mirror the canvas transform exactly.** The feed is drawn with
   `object-fit: cover` and flipped horizontally. Landmarks are normalized to the *source*
   frame, so they only line up after the same crop, scale, and mirror are applied. Change
   how the video is drawn in `draw()` and you must change this function to match, or the
   blade drifts off the finger.
3. **`REACH_X` / `REACH_Y` expand the usable area.** Landmark coords are pushed outward
   from center (1.22× / 1.42×) so the player can reach the screen edges — especially the
   bottom — without their hand leaving the camera's view, which is exactly where tracking
   gets unreliable.
4. **Per-hand filter continuity.** `HANDS` is keyed by MediaPipe's handedness label so each
   physical hand keeps its own One-Euro state. On a label collision the second hand is
   flipped to the other key. When a hand is lost for >220ms it deactivates, its filters
   reset, and its blade clears — so re-acquiring snaps cleanly instead of drawing a
   phantom slash across the screen.

### Slicing
A cut is the segment from the previous rendered position to the current one, tested
against each fruit's circle (`segCircleHit`, radius ×1.05). It only counts if the hand is
moving at `MIN_SLICE_SPEED` (6 px/frame) or better — otherwise a resting finger would
shred everything it touched. A pinch bypasses the speed gate for deliberate slow cuts.
At `POWER_SPEED` (34 px/frame) it becomes a power slice: +1 point, bigger splash, more
particles. Both hands are tested every frame; the first hit wins.

## 7. Game rules & tuning constants

| Constant | Value | Meaning |
|---|---|---|
| `MAX_LIVES` | 3 | missed fruit cost a life; missed bombs are free |
| `GRAVITY` | 0.16 | base fall speed, scaled up to 2× by difficulty |
| `BASE_MAX_ACTIVE` | 3 | whole fruit on screen at once, up to 6 |
| `BOMB_SCORE` | 30 | score at which bombs start spawning |
| `BOMB_CHANCE` | 0.28 | spawn chance once unlocked, creeping to 0.45 |
| `MIN_SLICE_SPEED` | 6 | px/frame floor for a cut to register |
| `POWER_SPEED` | 34 | px/frame threshold for a power slice |
| `DETECT_INTERVAL` | 45 | ms between inferences (~22Hz) |
| `PREDICT` | 0.9 | frames of velocity lead-ahead |
| `BLADE_MS` | 190 | blade trail lifetime |

Difficulty is `0` below `BOMB_SCORE`, then `+1` every 10 points after it. Each level
shortens the spawn interval (floor 480ms), increases gravity, raises the fruit cap, and
nudges the bomb chance up. Slicing a bomb is instant game over.

Fruit are drawn as **color emoji glyphs**, not sprites — which means fruit appearance
depends on the platform's emoji font (`EMOJI_FONT` lists the fallbacks). Point values and
particle colors live in the `FRUITS` table.

## 8. Persistence

One key: `localStorage['neonFruitHi']` — the high score, written in `endGame()`. That's
the only persisted state and the only storage the game touches.

## 9. Startup sequence

`startBtn` → `boot()` is fired **without `await`**, then `startGame()` runs immediately.
This is intentional: a pending or denied camera permission prompt must never block the
game from starting. Fruit begin falling on the dark background and the video fades in when
the stream is ready. Inside `boot()`, the camera is requested first (step 1) so the
background appears as soon as the player clicks Allow, and only then is the model loaded
(step 2). Failures at either step surface in `#status` and leave the game playable.

A global `error` handler swallows the detail-less `"Script error."` that browsers emit for
cross-origin scripts — MediaPipe's CDN worker triggers it and it isn't a real bug.

## 10. Testing & gotchas

There is no test suite. Verification is manual, in a browser, with a camera:

- Serve it (`npx http-server -p 8000`) and open `http://localhost:8000` — never `file://`.
- Check both hands independently, hand loss and re-acquisition, and that the blade stays
  glued to the fingertip near all four edges (that's the `videoToScreen` regression test).
- Check the bomb transition at 30 points and that a missed bomb costs nothing.
- Watch the frame rate with two hands up — that's the worst case.
- Lighting and background matter. Poor tracking is often the room, not the code.

Known rough edges: `State.PAUSED` is unused; there's no mobile/touch fallback; CDN pins are
hard-coded and unversioned locally, so an upstream removal would break the live build.
