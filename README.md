# Palm Slash

A little Fruit Ninja–style game I built that uses your **webcam and your hands** instead of a mouse or a touchscreen. Point at the screen, swipe through the fruit, and watch it split apart. No controller, no keyboard — just your hands in the air.

It's one single `index.html` file. That's the whole game. Open it and play.

**▶️ Play it here:** https://shivamvkapadia.github.io/PalmSlash/
*(your browser will ask for camera permission — that's the whole point, say yes)*

---

## 🆕 What's new in v1.1

- **Two-handed play** — bring up both hands and you get **two independent blades**. Twice the carnage.
- **Power slices** — swipe *fast* and you land a ⚡ power slice: bonus point, bigger splash, more juice.
- **Pinch for precision** — close your thumb and index finger for a tight, deliberate cut when things get crowded.
- **Difficulty ramp** — past 30 points the game steps up every 10 points: faster fruit, more on screen, more bombs.
- **Way smoother tracking** — rebuilt the whole tracking pipeline to be much more fluid and a lot less laggy (see below).

---

## What it does

- Tracks your **index fingertip(s)** in real time and turns them into glowing blades.
- Fruit fall from the top — 🍊 oranges, 🍅 tomatoes, 🍉 watermelons, 🍌 bananas, 🍇 grapes, 🍓 strawberries, 🍋 lemons. Slice them for points; they burst into juice and fly apart in two halves.
- You get **3 lives**. Let three pieces of fruit hit the floor and it's over.
- Past **30 points**, **💣 bombs** start dropping in with the fruit, with an angry red glow. Slice one and you're instantly done — for those, just keep your hands away and let them fall.
- Your **high score** is saved locally, so you've always got a number to beat.

## How to play

1. Open the link above (or run it locally — see below).
2. Allow the camera.
3. Stand back far enough that your hands are comfortably in frame, with decent lighting.
4. **Swipe through the fruit.** Use both hands for two blades, swipe fast for power slices, pinch for precise cuts — and dodge the bombs once they appear.

Works best when the room isn't too dark and your hands aren't lost against a busy background.

## How it's built

Nothing heavy — it's deliberately a single file with no build step.

- **[p5.js](https://p5js.org/)** for the canvas, the fruit, the juice particles, and the blade trails.
- **[MediaPipe Tasks – Hand Landmarker](https://ai.google.dev/edge/mediapipe)** (running on the GPU) for the hand tracking. It returns 21 points per hand every frame; I use the index fingertips, and read pinch from the thumb–index distance.
- A **One-Euro filter** smooths each cursor — it kills jitter when your hand is still but stays snappy when you swipe fast.

### Making it smooth (the v1.1 fight)
Getting two-handed tracking to feel fluid took some doing:
- The heaviest thing on the page is the model inference, and it blocks the render loop while it runs — so I **throttle detection** to a sane rate and let the **60fps One-Euro smoothing fill the gaps**. Looks just as smooth, far less lag.
- The webcam feed is cropped to fill the screen, so raw landmark coordinates don't line up with what you see — everything goes through the same crop/mirror transform or the blade drifts off your finger.
- I expand the reach a bit toward the screen edges so the bottom is comfortably reachable without your hand slipping out of the camera's view.
- A touch of velocity prediction trims the perceived latency so the blade stays glued to your fingertip.

The look is **8-bit arcade**: bitmap type, chunky beveled panels, hard black outlines and
buttons that clunk down when you press them — all plain CSS, no images. The frames use
un-blurred shadows and stepped animations on purpose, so the chrome costs the browser
almost nothing and every cycle stays where it matters: the hand tracking.

## Heads up

- It's a webcam toy, so the experience depends on your camera, lighting, and machine. On a decent laptop it runs smooth; on something older it might chug a little (two hands is heavier than one).
- First load pulls the tracking model (a few MB) from a CDN, so the very first "loading hand tracking…" takes a moment.

---

Built for fun. If you get a stupidly high score, I don't want to know about it. 🍉
