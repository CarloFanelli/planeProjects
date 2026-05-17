# planeProjects

A small portfolio of creative-coding experiments built across a 30-hour flight stretch
(2×3h + 2×12h, spring 2026): computer vision, browser audio synthesis, real-time
WebSocket bridging, and GPU-shader 3D visualization — all running offline on an M1 Pro.

The four repos build on each other:

```
   handmouse  ──►  gibber-sketches  ──►  handgibber  ──►  synesthesia
   (vision)        (audio)              (vision+audio)    (everything + 3D)
```

---

## 1. [handmouse](https://github.com/CarloFanelli/handMouse) → hand-tracking module

Reusable Python module wrapping MediaPipe Gesture Recognizer with a callback-based API.
A `HandTracker` class emits events (`landmarks`, `pinch`, `click`, `swipe`, `gesture`) and
ships with three demo apps:

- **mouse-control** — move the macOS cursor with your index finger, click with a thumb↔middle pinch (smoothed via OneEuroFilter)
- **presenter remote** — swipe to change slides, closed-fist to start, open-palm to exit
- **air-keyboard 3x3** — hover a virtual key zone, pinch to type

> v2 modular refactor of a single-file v1 — both versions tagged in the repo.

**Stack:** Python 3.11 · MediaPipe Tasks · OpenCV · pyautogui · pynput

---

## 2. [gibber-sketches](https://github.com/CarloFanelli/gibber-sketches) → live-coding audio

A small collection of patterns I wrote to explore [Gibber](https://gibber.cc), the
browser-based live-coding environment for music. Each sketch is a `.js` snippet you
paste into the Gibber playground and run with `Cmd+Shift+Enter`:

- **01_groove** — four-on-the-floor + hat + bass
- **02_acid** — acid bassline with a modulating filter sweep
- **03_drone** — ambient FM drone with reverb and delay

Plus a tiny HTML browser to list the sketches with a one-click copy button.

**Stack:** JavaScript · Gibber (Web Audio live-coding)

---

## 3. [handgibber](https://github.com/CarloFanelli/handgibber) → play sound with your hands

The bridge between #1 and #2. A Python WebSocket server runs `HandTracker` and
broadcasts hand events as JSON; the "client" is a Gibber sketch you paste in the
playground that listens to the WebSocket and modulates synth parameters live.

Three patches included:
- **theremin** — index X → pitch (3 octaves), Y → volume
- **drumpad** — pinch in 3 X-zones → trigger kick/snare/hat
- **acid lab** — Y → filter cutoff, X → resonance Q, swipe → cycle pattern

**Stack:** Python 3.11 · MediaPipe · WebSockets · Gibber

---

## 4. [synesthesia](https://github.com/CarloFanelli/synesthesia) → flagship 🎆

Audio-reactive 3D particle visualizer controlled by your hands.
Three.js shader-based galaxy (40k GPU-instanced points), deformed by a noise field and
pulsed by audio FFT data. The same hand bridge from #3 also orbits the camera, picks
the lead synth note, triggers energy bursts on pinch, and cycles the color palette on
swipe.

This one is fully self-contained: `git clone + npm install + python server/bridge.py
+ npm run dev` and it just runs.

**Stack:** Python 3.11 · MediaPipe · WebSockets · Gibber (audio) · Three.js · GLSL shaders

---

## How it was built

All four projects were scaffolded pre-flight and implemented offline during four
flights with no internet access. The setup required pre-caching all npm/pip
dependencies, downloading Gibber's bundle locally (and patching its CDN refs), and
downloading the entire three.js + Tone.js sources for offline reference.

Notes on the constraints, pre-flight checklist, architecture decisions, and per-project
implementation code are in [`PROJECTS.md`](PROJECTS.md) (the full implementation plan).

---

Built by [Carlo Fanelli](https://github.com/CarloFanelli) · spring 2026
