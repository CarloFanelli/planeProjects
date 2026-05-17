# WORKBOOK — costruisci i 4 progetti in volo, step by step

> **Stato di partenza**: i 4 repo GitHub esistono ma sono **scaffold vuoti** (solo README + .gitignore).
> Le dipendenze (Python venv, node_modules di P4, Gibber bundle, Three.js docs) sono **già scaricate** in `~/Desktop/planeProjects/_vendor/` e `_docs/`.
> Tu **scrivi tutto il codice in volo**.
> Se ti blocchi: il codice completo "di riferimento" è in [`PROJECTS.html`](PROJECTS.html) (apri con `open PROJECTS.html`). Usalo per sbloccarti, non come copy-paste — la skill è scriverlo tu.

---

## Convention

- `- [ ]` = task da spuntare (la HTML version ha checkbox cliccabili persistenti)
- Ogni step ha: **cosa creare**, **perché**, **hint/struttura**, **commit message pronto** da copia-incollare
- Tutti i comandi assumono `~/Desktop/planeProjects/` come root del workspace
- ESC nelle finestre OpenCV per chiudere
- `Ctrl+C` per killare i dev server
- **Tutto è offline-ready**: Python deps installate nel venv condiviso, npm cache + tarballs in `_vendor/npm-cache/`, Gibber bundle in `_vendor/gibber/playground/`, three.js/tone.js source in `_docs/`

---

## STAGE 0 — Pre-takeoff (10 min a terra)

- [ ] Permessi macOS: System Settings → Privacy & Security → **Accessibility** ON per il tuo terminale
- [ ] Test camera: `cd ~/Desktop/planeProjects && source .venv/bin/activate && python -c "import cv2; c=cv2.VideoCapture(0); print('cam OK' if c.read()[0] else 'NO CAM'); c.release()"`
- [ ] Verifica stato repo (devono essere tutti puliti, allineati a origin):
  ```bash
  for d in ~/Desktop/planeProjects/p{1_handmouse,2_gibber,3_handgibber,4_synesthesia}; do
    echo "=== $(basename $d) ==="; (cd $d && git status -s && git log --oneline -1); done
  ```
- [ ] **Disattiva wifi**, riapri il terminale, ripeti il test camera per essere sicuro che tutto giri offline
- [ ] Apri **WORKBOOK.html** in Safari (`open ~/Desktop/planeProjects/WORKBOOK.html`) — i checkbox sono cliccabili e persistono
- [ ] Apri **PROJECTS.html** in una seconda finestra (reference rapido se ti blocchi)

---

## VOLO 1 — 3h — Build `handmouse` (refactor v1 monolith → v2 modular)

**Target**: trasformare `hand_recognition.py` (monolite con globals e tutto nel main loop) in un pacchetto Python con classe `HandTracker` callback-based + 3 demo applicative (mouse, presenter, air keyboard).

```bash
cd ~/Desktop/planeProjects/p1_handmouse
source ../.venv/bin/activate
ls -la                         # vedi: README.md, hand_recognition.py, gesture_recognizer.task
```

### Step 1.0 — Leggi il v1 monolite (5 min)
- [ ] `cat hand_recognition.py | less` (q per uscire)
- [ ] Identifica le parti: classe `HandPoint`, funzioni `checkPinch/click/update_hand_points/draw_landmarks/process_frame/display_info/drawSquare`, main loop
- [ ] Idea del refactor: split in `geometry.py` (HandPoint+filter+calibration) / `gestures.py` (pinch/click/swipe) / `tracker.py` (cv2 loop + event bus)

### Step 1.1 — Crea la struttura
- [ ] `mkdir handmouse demos`
- [ ] `touch handmouse/__init__.py demos/__init__.py`

### Step 1.2 — `handmouse/geometry.py`
**Cosa scrivere**:
- `@dataclass HandPoint` con x, y, z float
  - `distance_to(other)` → euclidean 3D
  - `midpoint(other)` → nuovo HandPoint a metà
- `class OneEuroFilter(min_cutoff=1.0, beta=0.02, d_cutoff=1.0)`: callable `__call__(x, t)` con smoothing adattivo. Mantiene `_x_prev, _dx_prev, _t_prev`. Helper `_alpha(cutoff, dt) = 1/(1+tau/dt)` con `tau = 1/(2*pi*cutoff)`.
- `class Calibration(x_min, x_max, y_min, y_max)`: `map_point(x, y) -> (nx, ny)` mappato in [0,1] e clampato.

**Perché OneEuro**: il cursore mouse ha jitter percepibile coi landmarks raw. OneEuro è low-pass adattivo: smussa a riposo ma non lagga sul movimento veloce.

**Hint**: import `dataclass`, `math`.

If stuck → [PROJECTS.html → Step 1.2](PROJECTS.html#step-12-handmousegeometrypy)

- [ ] File scritto e gira (test): `python -c "from handmouse.geometry import HandPoint, OneEuroFilter, Calibration; print(HandPoint(1,2,3).distance_to(HandPoint(4,5,6)))"`
- [ ] Commit:
  ```
  git add handmouse/__init__.py handmouse/geometry.py
  git commit -m "feat(handmouse): geometry primitives (HandPoint, OneEuroFilter, Calibration)"
  ```

### Step 1.3 — `handmouse/gestures.py`
**Cosa scrivere**:
- Costanti `PINCH_THRESHOLD = 0.06`, `CLICK_THRESHOLD = 0.045` (in coordinate normalizzate MediaPipe)
- `is_pinching(index_tip, thumb_tip) -> (bool, dist)` — usa `distance_to`
- `is_clicking(middle_tip, thumb_tip) -> (bool, dist)`
- `class SwipeDetector(history=10, min_dx=0.25, max_dy=0.10)`:
  - `update(x, y) -> 'left' | 'right' | None`
  - usa una `deque(maxlen=history)`, calcola dx = `xs[-1] - xs[0]`, dy = `max(ys) - min(ys)`
  - se dy troppo grande → non è swipe (mano si muove anche verticalmente)
  - cooldown di 15 frame dopo uno swipe per evitare doppi trigger

**Hint**: `from collections import deque`, `from .geometry import HandPoint`.

If stuck → [PROJECTS.html → Step 1.3](PROJECTS.html#step-13-handmousegesturespy)

- [ ] Commit:
  ```
  git add handmouse/gestures.py
  git commit -m "feat(handmouse): pinch/click/swipe detection helpers"
  ```

### Step 1.4 — `handmouse/tracker.py` (cuore del modulo)
**Cosa scrivere**:
- `class HandTracker(model_path, num_hands=1, camera_index=0, mirror=True, show_window=True)`
- Costruttore: crea `mp_vision.GestureRecognizer.create_from_options(...)`, apre `cv2.VideoCapture(camera_index)`, raise se non si apre
- Costanti landmark MediaPipe: `LM_THUMB_TIP=4, LM_INDEX_TIP=8, LM_INDEX_BASE=5, LM_MIDDLE_TIP=12`
- Event bus: `_handlers: dict[str, list]`, metodi `on(event, fn)` chainable, `_emit(event, *args)` con try/except per non killare il loop
- Estrazione punti: `_extract(hand_landmarks)` ritorna dict `{thumb_tip, index_tip, index_base, middle_tip}` di HandPoint (applica `1 - x` se mirror)
- Main loop `run()`: ciclo `while True`:
  - `cap.read()`, `cv2.cvtColor` BGR→RGB, `mp.Image(...)`, `recognizer.recognize`
  - emit `frame`, `landmarks`, `gesture`
  - Detect pinch/click rising edges (track `_was_pinching`, `_was_clicking`)
  - Emit `swipe` da `SwipeDetector` su `index_base.x/y`
  - Se `show_window`: `cv2.imshow`, ESC per uscire
  - `finally`: `cap.release()`, `cv2.destroyAllWindows()`

**Eventi totali**: `landmarks(pts, result)`, `pinch(center, dist)`, `pinch_release()`, `click(center)`, `click_release()`, `swipe(dir)`, `gesture(name, score)`, `frame(img, result)`.

**Perché**: il vecchio main loop era monolitico con variabili globali. Qui le demo attaccano listener; logica pulita.

If stuck → [PROJECTS.html → Step 1.4](PROJECTS.html#step-14-handmousetrackerpy-cuore)

- [ ] Commit:
  ```
  git add handmouse/tracker.py
  git commit -m "feat(handmouse): HandTracker class with callback-based event bus"
  ```

### Step 1.5 — `handmouse/__init__.py`
- [ ] Esporta `HandTracker, HandPoint, OneEuroFilter, Calibration, SwipeDetector, is_pinching, is_clicking`
- [ ] `__all__ = [...]`
- [ ] Commit:
  ```
  git add handmouse/__init__.py
  git commit -m "feat(handmouse): public API exports"
  ```

### Step 1.6 — Smoke test del modulo
- [ ] Test rapido senza demo: `python -c "
from handmouse import HandTracker
t = HandTracker()
t.on('landmarks', lambda pts, _: print('hand:', pts['index_tip']))
t.on('pinch', lambda c, d: print('PINCH', d))
t.run()
"`
- [ ] Vedi finestra webcam? Quando pinchi vedi `PINCH` printato? ESC per uscire.
- [ ] Se errore "Cannot open camera": permessi macOS (Settings → Privacy → Camera)

### Step 1.7 — Archive del v1 in `legacy/`
- [ ] `mkdir legacy && git mv hand_recognition.py legacy/ && git mv README.md legacy/`
- [ ] Commit:
  ```
  git commit -m "refactor: archive v1 monolith into legacy/"
  ```

### Step 1.8 — `demos/mouse.py`
**Cosa scrivere**:
- `import pyautogui`; setta `FAILSAFE = False`, `PAUSE = 0.0`; ottieni screen size
- Crea `tracker = HandTracker()`, `calib = Calibration(0.2, 0.8, 0.2, 0.8)`, due `OneEuroFilter(min_cutoff=1.5, beta=0.05)` (uno per x uno per y)
- `on_landmarks(pts, _)`: mappa `index_tip.x/y` via calib, filtra via OneEuro, `pyautogui.moveTo(sx, sy, duration=0)`
- `on_click`: `pyautogui.click()` (è triggered dal pinch medio↔pollice)
- `tracker.on('landmarks', on_landmarks).on('click', lambda c: pyautogui.click()).run()`

**Test**: `python -m demos.mouse` → muovi mano → vedi cursore? Pinch medio→pollice = click?

If stuck → [PROJECTS.html → Step 1.7](PROJECTS.html#step-17-demosmousepy)

- [ ] Commit:
  ```
  git add demos/__init__.py demos/mouse.py
  git commit -m "feat(demos): mouse-control demo (move + click via pinch)"
  ```

### Step 1.9 — `demos/presenter.py`
**Cosa scrivere**:
- `from pynput.keyboard import Controller, Key`
- `tracker.on('swipe', lambda d: tap(Key.right if d=='right' else Key.left))`
- `on_gesture(name, score)`: se score>0.7 e name in `{'Closed_Fist':F5, 'Open_Palm':Esc}` → tap

**Test**: apri Keynote/Preview con qualche pagina, swipe per cambiare; closed_fist per F5 (presentation mode).

- [ ] Commit:
  ```
  git add demos/presenter.py
  git commit -m "feat(demos): slide presenter remote (swipe + gestures)"
  ```

### Step 1.10 — `demos/air_keyboard.py`
**Cosa scrivere**:
- Layout 3x3 di caratteri `[['q','w','e'],['a','s','d'],['z','x','c']]`
- `on_landmarks` aggiorna stato `cell = cell_at(nx, ny)` (0..2, 0..2)
- `on_pinch` con cooldown 0.4s: se cella valida, `kb.type(LAYOUT[r][c])`

**Test**: apri TextEdit, prova a scrivere "ciao".

- [ ] Commit:
  ```
  git add demos/air_keyboard.py
  git commit -m "feat(demos): air keyboard 3x3 (hover + pinch to type)"
  ```

### Step 1.11 — `requirements.txt` + nuovo README
- [ ] Crea `requirements.txt` con: mediapipe, opencv-python, pyautogui, numpy, pynput
- [ ] Riscrivi `README.md` con: descrizione modulo, quickstart, API esempio, link agli altri 3 progetti del portfolio (https://github.com/CarloFanelli/gibber-sketches etc.)
- [ ] Commit:
  ```
  git add requirements.txt README.md
  git commit -m "docs: requirements + new README for v2 modular API"
  ```

### Step 1.12 — Tag v2-modular
- [ ] `git tag -a v2-modular -m "v2: modular refactor (HandTracker module + 3 demos)"`
- [ ] Verifica: `git log --oneline && git tag -l` (devi vedere v1-monolithic e v2-modular)

### 🎉 P1 DONE
Push lo fai a terra: `git push && git push --tags`.

---

## VOLO 2 — 3h — Build `gibber-sketches`

**Target**: collection di sketch live-coding per il playground Gibber + mini runner HTML.

```bash
cd ~/Desktop/planeProjects/p2_gibber
mkdir -p sketches runner
```

### Step 2.0 — Setup
- [ ] Lancia Gibber locale in un terminale (resterà acceso tutto il volo):
  ```bash
  cd ~/Desktop/planeProjects/_vendor/gibber && npm start
  # http://127.0.0.1:9080 (apri nel browser)
  ```
- [ ] Esplora i [tutorials/examples nel menu del playground] per familiarizzare con la sintassi
- [ ] Spegni Gibber con `Gibber.clear()` o `Cmd+.`

### Step 2.1 — sketch `01_groove.js`
**Target**: groove minimale four-on-the-floor + hat + bass.
**API Gibber usate**: `Clock.bpm`, `Drums('pattern')` (x=hit, *=rest, -=quiet hit), `.gain()`, `.fx.add()`, `Reverb`, `Synth('bleep')`, `Filter24`, `.note.seq([...], 1/4)`.

**Struttura**:
```js
Clock.bpm = 124
kick = Drums('x*x*x*x*').gain(.9)
hat  = Drums('-*-*-*-*').gain(.4).fx.add( Reverb(.3) )
bass = Synth('bleep').gain(.5).fx.add( Filter24({cutoff:.35, Q:.8}) )
bass.note.seq([0, 0, 7, 5], 1/4)
```

- [ ] Scrivi `sketches/01_groove.js`
- [ ] Test: incolla in Gibber, `Cmd+Shift+Enter`
- [ ] Senti groove? `Gibber.clear()` per stoppare.
- [ ] Commit:
  ```
  git add sketches/01_groove.js
  git commit -m "feat(sketches): 01 groove (four-on-the-floor + hat + bass)"
  ```

### Step 2.2 — sketch `02_acid.js`
**Target**: acid bassline che gira con filter sweep su 0.5 measure.
**Aggiunta**: `.fx[0].cutoff.seq([...], Measure(.5))` per modulare il cutoff.

- [ ] Scrivi `sketches/02_acid.js` (BPM 132, pattern di note tipo `[0,3,5,7,5,3,0,-5]` 1/8)
- [ ] Test in Gibber
- [ ] Commit:
  ```
  git add sketches/02_acid.js
  git commit -m "feat(sketches): 02 acid (bassline + modulating filter sweep)"
  ```

### Step 2.3 — sketch `03_drone.js`
**Target**: ambient FM drone con 2 voci.
**API nuove**: `FM({cmRatio: 1.5, index: 4})`, `Measure(2)` per pattern lenti.

- [ ] Scrivi `sketches/03_drone.js` (BPM 60, due FM con note progressioni lente)
- [ ] Test in Gibber
- [ ] Commit:
  ```
  git add sketches/03_drone.js
  git commit -m "feat(sketches): 03 drone (ambient FM with reverb/delay)"
  ```

### Step 2.4 — Mini runner HTML
**Target**: pagina che lista gli sketch con bottone Copy.

- [ ] `runner/index.html`: HTML scheletro con `<div id="list"></div>`, CSS dark, `<script type="module" src="./runner.js"></script>`
- [ ] `runner/runner.js`: array `files = ['01_groove.js', '02_acid.js', '03_drone.js']`, per ognuno `fetch('../sketches/'+f).then(r=>r.text())`, crea `<section>` con `<h2>` + bottone Copy (`navigator.clipboard.writeText`) + `<pre>` col codice
- [ ] Test: secondo terminale `python3 -m http.server 8000`, apri http://127.0.0.1:8000/runner/
- [ ] Vedi 3 sketch listate? Copy funziona?
- [ ] Commit:
  ```
  git add runner/
  git commit -m "feat(runner): browser playlist with copy buttons"
  ```

### Step 2.5 — README
- [ ] Riscrivi `README.md` con: link a Gibber, come avviare, descrizione di ogni sketch
- [ ] Commit:
  ```
  git add README.md
  git commit -m "docs: enrich README"
  ```

### Polish opzionale (resto del volo)
- [ ] sketch 04 polyrhythm 3v4 (vedi PROJECTS.html → Polish P2)
- [ ] sketch 05 glitch BitCrusher
- [ ] sketch 06 visual con `Marching()` (Gibber ha anche grafica generativa)

### 🎉 P2 DONE

---

## VOLO 3 — 12h — Build `handgibber` + polish P1/P2 + setup P4

**Target**: bridge Python WebSocket → Gibber playground. Le mani modulano synth live.

```bash
cd ~/Desktop/planeProjects/p3_handgibber
mkdir -p server sketches
```

### Step 3.0 — Architettura
```
[bridge.py: HandTracker]
       │ ws://127.0.0.1:8765 (JSON)
       ▼
[Browser: Gibber playground (paste-and-run sketch)]
```
Messages: `{type:'hand', x, y, ...}` continuous, `{type:'event', name:'pinch|swipe|gesture', ...}` discrete.

### Step 3.1 — `server/bridge.py`
**Cosa scrivere**:
- Import `asyncio, json, sys, pathlib, threading, websockets`
- Aggiungi `~/Desktop/planeProjects/p1_handmouse` al `sys.path` per importare handmouse
- `async def _amain()`:
  - `clients = set()`
  - `async def handler(ws)`: aggiungi a `clients`, loop `async for _ in ws: pass`, `finally` discard
  - `loop = asyncio.get_running_loop()`
  - `def broadcast_sync(msg: dict)`: serializza JSON, schedula `asyncio.run_coroutine_threadsafe(send_all(), loop)`
  - `def run_tracker()` (per thread):
    - `tracker = HandTracker(model_path=MODEL_PATH)`
    - attacca i listener che chiamano `broadcast_sync(...)`
    - `tracker.run()` bloccante
  - Thread daemon che esegue `run_tracker`
  - `async with websockets.serve(handler, '127.0.0.1', 8765): await asyncio.Future()`
- `if __name__ == "__main__": asyncio.run(_amain())`

**Pattern cruciale**: HandTracker.run è bloccante (cv2 loop) → thread separato. Le sue callbacks devono schedulare sull'asyncio loop con `run_coroutine_threadsafe` (non possono chiamare `await` direttamente).

If stuck → [PROJECTS.html → Step 3.1](PROJECTS.html#step-31-serverbridgepy)

- [ ] Test prima di committare:
  ```bash
  python server/bridge.py
  # in un altro terminale:
  # printf "" | websocat ws://127.0.0.1:8765 -1 2>/dev/null
  # oppure usa browser console: new WebSocket('ws://127.0.0.1:8765').onmessage=e=>console.log(JSON.parse(e.data))
  ```
- [ ] Muovi la mano davanti webcam → vedi JSON arrivare nel browser?
- [ ] Commit:
  ```
  git add server/bridge.py
  git commit -m "feat(server): WebSocket bridge from HandTracker to browser clients"
  ```

### Step 3.2 — sketch `01_theremin.js`
**Target**: X mano → pitch midi 48..84, Y → volume.

**Struttura**:
```js
osc = Synth('sine').gain(0).fx.add( Reverb(.4), Delay(1/4) )
if (window._hgWs) window._hgWs.close()
window._hgWs = new WebSocket('ws://127.0.0.1:8765')
window._hgWs.onmessage = e => {
  const m = JSON.parse(e.data)
  if (m.type === 'hand') {
    osc.note( 48 + Math.floor(m.x * 36) )
    osc.gain( Math.max(0, 1 - m.y) )
  }
}
// Stop: window._hgWs.close(); Gibber.clear()
```

- [ ] Scrivi, testa: bridge attivo + paste in Gibber playground
- [ ] Senti theremin che segue la mano in X (pitch) e Y (volume)?
- [ ] Commit:
  ```
  git add sketches/01_theremin.js
  git commit -m "feat(sketches): 01 theremin (X=pitch, Y=volume)"
  ```

### Step 3.3 — sketch `02_drumpad.js`
**Target**: pinch in 3 zone X → kick/snare/hat.

**Struttura**: 3 `Synth('kick'|'snare'|'hat').gain(...).fx.add(...)`. Su `event:pinch` trigger uno dei 3 in base a `m.x`.

- [ ] Scrivi, testa
- [ ] Commit:
  ```
  git add sketches/02_drumpad.js
  git commit -m "feat(sketches): 02 drumpad (pinch + zone -> drum trigger)"
  ```

### Step 3.4 — sketch `03_acid_lab.js`
**Target**: bassline che gira sempre, Y → cutoff, X → Q, swipe → cambia pattern.

**Pattern array**: 3 array di note diversi, indice incrementato su swipe.

- [ ] Scrivi, testa
- [ ] Commit:
  ```
  git add sketches/03_acid_lab.js
  git commit -m "feat(sketches): 03 acid lab (hand modulates filter, swipe cycles pattern)"
  ```

### Step 3.5 — README
- [ ] README con: architettura ASCII, quickstart in 3 step (bridge + gibber + paste sketch), descrizione delle 3 patch, link cross-progetti
- [ ] Commit:
  ```
  git add README.md
  git commit -m "docs: complete README with architecture and quickstart"
  ```

### Polish + Setup P4 (resto del volo)

Polish P1/P2 con quello che hai imparato dall'usarli + commit nei rispettivi repo.

Setup P4 prep (per essere pronto al volo 4):
- [ ] `cd ~/Desktop/planeProjects/p4_synesthesia`
- [ ] Verifica `node_modules` esiste (era pre-installato): `ls node_modules/three node_modules/tone node_modules/vite` (devi vedere directory)
- [ ] Crea symlink al Gibber bundle: `mkdir -p public && ln -s ../../_vendor/gibber/playground public/gibber && ls -la public/gibber/bundle.js`
- [ ] Smoke test prima del volo 4: `npm run dev` (errore? OK, manca package.json — lo crei nel volo 4)

### 🎉 P3 DONE

---

## VOLO 4 — 12h — Build `synesthesia` (FLAGSHIP)

**Target**: visualizer 3D audio-reattivo + hand control. Three.js particelle + Gibber audio + bridge WS.

```bash
cd ~/Desktop/planeProjects/p4_synesthesia
mkdir -p src/shaders server vendor
```

### Step 4.0 — Architettura
```
[bridge.py: HandTracker] ── ws ──► [Browser Vite :5173]
                                        ├─ <script> gibber/bundle.js (UMD global)
                                        └─ <script type=module> src/main.js
                                            ├─ src/audio.js (Gibber synth + AnalyserNode)
                                            ├─ src/hands.js (WS client)
                                            └─ src/particles.js (Three.js shader)
```

### Step 4.1 — `package.json`
```json
{
  "name": "synesthesia",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": { "dev": "vite", "build": "vite build", "preview": "vite preview" },
  "dependencies": { "three": "^0.184.0", "tone": "^15.1.22" },
  "devDependencies": { "vite": "^8.0.13" }
}
```

- [ ] Scrivi `package.json`. NO `npm install` (offline): `node_modules` è già installato dal pre-flight.
- [ ] Verifica: `npm run dev --help 2>&1 | head -1` → mostra help di Vite? OK.
- [ ] Commit:
  ```
  git add package.json
  git commit -m "chore: package.json (three, tone, vite already in node_modules)"
  ```

### Step 4.2 — `vite.config.js`
```js
export default {
  server: { host: '127.0.0.1', port: 5173 },
  optimizeDeps: { include: ['three', 'tone'] },
}
```
- [ ] Scrivi, commit:
  ```
  git add vite.config.js
  git commit -m "chore: vite config (port 5173, optimize three+tone)"
  ```

### Step 4.3 — `index.html`
**Cosa**: HTML scaffold con:
- `<div id="start">click to start</div>` (autoplay policy: audio dev partire dopo gesture utente)
- `<div id="hud">` con 4 span: fps, hand, energy, palette
- `<script src="/gibber/jsdsp.js"></script>` E `<script src="/gibber/bundle.js"></script>` (UMD, deve caricare PRIMA dei moduli)
- `<script type="module" src="/src/main.js"></script>`
- CSS inline: body nero, hud fisso top-left, start overlay full screen

If stuck → [PROJECTS.html → Step 4.2](PROJECTS.html#step-42-indexhtml)

- [ ] Scrivi, commit:
  ```
  git add index.html
  git commit -m "feat(p4): index.html scaffold with HUD + start overlay"
  ```

### Step 4.4 — Setup Gibber bundle localmente
- [ ] Se non l'hai già fatto nel volo 3: `ln -s ../../_vendor/gibber/playground public/gibber`
- [ ] Verifica: `ls -la public/gibber/bundle.js` (3.7MB)
- [ ] **Nessun commit** — `public/gibber/` è gitignored

### Step 4.5 — `src/hands.js` (WS client)
**Cosa scrivere**: export `hands` object `{x:.5, y:.5, pinch:false, lastPinchAt:0, swipe:null, connected:false}` + `startHands(url='ws://127.0.0.1:8765')` che apre WS e aggiorna `hands` sui messaggi.

- [ ] Scrivi, commit:
  ```
  git add src/hands.js
  git commit -m "feat(p4): WebSocket client (shared hands state)"
  ```

### Step 4.6 — `src/audio.js` (Gibber primary)
**Cosa scrivere**:
- Helper `findGibberOutputNode()`: prova `G.Audio.Master.output`, `G.Audio.Master`, `G.Master.output`, `G.Master`, `G.Audio.output` (probing difensivo perché l'API esatta varia tra build)
- Helper `findGibberAudioContext()`: `G.Audio.ctx || G.Audio.context || G.ctx`
- `audio.start()`: chiama `Gibber.export(window)`, resume AudioContext, crea `Drums()`, `Synth('bleep')` (bass), `FM()` (lead silenzioso), avvia `Clock.bpm = 124`
- Connetti `AnalyserNode` (fftSize 256, smoothing .7) al master node trovato
- `update()`: `getByteFrequencyData(buf)`, calcola bass/mid/high/energy in 0..1
- `setHandIntensity(x, y)`: lead.gain((1-y)*.55), lead.note(48 + x*36)
- `bang()`: trigger nota stab alta

**Fallback**: se master node non trovato, log warning e collega analyser ai singoli synth via `s.output.connect(analyser)`.

If stuck → [PROJECTS.html → Step 4.4](PROJECTS.html#step-44-srcaudiojs-gibber-primary)

- [ ] Scrivi, commit:
  ```
  git add src/audio.js
  git commit -m "feat(p4): Gibber audio engine + AnalyserNode (defensive probing)"
  ```

### Step 4.7 — `src/audio_tonejs.js` (Plan B)
Stesso API (`audio.start/update/setHandIntensity/bang`), implementato con `Tone.js` (MembraneSynth + MetalSynth + FMSynth + MonoSynth + Sequence). Usa `Tone.FFT(64)` per l'analyser. **Da usare solo se Gibber non collabora** (swap-in 1 riga in main.js).

If stuck → [PROJECTS.html → Step 4.4-bis](PROJECTS.html#step-44-bis-srcaudio_tonejsjs-plan-b)

- [ ] Scrivi, commit:
  ```
  git add src/audio_tonejs.js
  git commit -m "feat(p4): Tone.js Plan B fallback (rock-solid alternative)"
  ```

### Step 4.8 — Shader vertex (`src/shaders/particle.vert.glsl`)
**Cosa**: input `uniform float uTime, uEnergy, uBass, uNoiseScale; attribute float aSeed; varying float vSeed, vEnergy;`
Calcola displacement con sin/cos di `t = uTime*.4 + aSeed*10`, applica a `position`. PointSize = `(1.5 + uEnergy*8)*(300/-mv.z)`.

- [ ] Scrivi, commit:
  ```
  git add src/shaders/particle.vert.glsl
  git commit -m "feat(p4): vertex shader (noise displacement + audio-pulse point size)"
  ```

### Step 4.9 — Shader fragment (`src/shaders/particle.frag.glsl`)
**Cosa**: gl_PointCoord per fare cerchio sfumato (`smoothstep(.5, 0, d)`), color = `uPalette * (.55 + vEnergy*.9 + vSeed*.3)`, alpha additive blending.

- [ ] Scrivi, commit:
  ```
  git add src/shaders/particle.frag.glsl
  git commit -m "feat(p4): fragment shader (additive glowing dots)"
  ```

### Step 4.10 — `src/particles.js`
**Cosa scrivere**: import THREE, `import vert from './shaders/particle.vert.glsl?raw'` (Vite `?raw` built-in), idem frag.
`export createParticles(count=40000)`: BufferGeometry, position con cubic root spherical distribution, aSeed random.
ShaderMaterial con vert/frag e uniforms `{uTime, uEnergy, uBass, uNoiseScale, uPalette: new THREE.Color('#6cf')}`.
`new THREE.Points(geo, mat)`.

If stuck → [PROJECTS.html → Step 4.7](PROJECTS.html#step-47-srcparticlesjs)

- [ ] Scrivi, commit:
  ```
  git add src/particles.js
  git commit -m "feat(p4): GPU particle system (40k points, shader-displaced)"
  ```

### Step 4.11 — `src/main.js` (render loop)
**Cosa scrivere**:
- Scene + FogExp2 + PerspectiveCamera(60deg) + WebGLRenderer (antialias, highperf)
- `particles = createParticles(40000); scene.add(particles)`
- Palette array, `setPalette(i)` ruota
- HUD spans references
- `frame()`: `audio.update`, `audio.setHandIntensity(hands.x, hands.y)`, camera orbit con hand X/Y (formule sferiche), update uniforms (uTime+=dt, uEnergy smooth, uBass smooth, uNoiseScale dipende da y), if pinch recente → uEnergy=1, swipe → cambia palette, `renderer.render()`
- Click su "start" → remove overlay, `await audio.start()`, `startHands()`, parte `frame()`

If stuck → [PROJECTS.html → Step 4.8](PROJECTS.html#step-48-srcmainjs)

- [ ] Scrivi, smoke test (`npm run dev`, http://127.0.0.1:5173, click start, vedi particelle?), commit:
  ```
  git add src/main.js
  git commit -m "feat(p4): Three.js render loop with audio + hand reactive uniforms"
  ```

### Step 4.12 — `server/bridge.py` (vendored handmouse)
- [ ] Crea `mkdir vendor && cp -r ../p1_handmouse/handmouse vendor/handmouse && cp ../p1_handmouse/gesture_recognizer.task vendor/`
- [ ] `cp ../p3_handgibber/server/bridge.py server/bridge.py`
- [ ] Modifica le 2 righe nel bridge.py:
  - `sys.path.insert(0, str(HERE.parent / "vendor"))`
  - `MODEL = str(HERE.parent / "vendor" / "gesture_recognizer.task")`
- [ ] **NO commit di vendor/** (è in .gitignore — terzi-parti). Solo `server/bridge.py`:
  ```
  git add server/bridge.py
  git commit -m "feat(p4): bridge.py (vendored handmouse path; vendor/ gitignored)"
  ```

### Step 4.13 — Test end-to-end
- [ ] Terminal A: `python server/bridge.py` → vedi webcam window?
- [ ] Terminal B: `npm run dev` → http://127.0.0.1:5173
- [ ] Click start → console: cerca `[audio] AudioContext state: running` e `[audio] analyser connesso a master`
- [ ] Muovi mano → camera orbita?
- [ ] Pinch → boost energia (particelle esplodono)?
- [ ] Swipe → cambia palette?
- [ ] HUD energy > 0 quando audio suona?

**Se energy resta 0 o vedi `Master node non trovato`**:
- Apri `src/main.js`, cambia `import { audio } from './audio.js'` in `import { audio } from './audio_tonejs.js'`
- Ricarica → ora usa Tone.js (sempre funziona)
- Commit: `git add src/main.js && git commit -m "fix(p4): switch to Tone.js audio engine (Gibber master API mismatch)"`

### Step 4.14 — Polish (resto del volo)

Scegli 2-3 effetti per il wow factor:

**A) Bloom postprocessing** (highest impact):
- `import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js'`
- `import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js'`
- `import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js'`
- Setup composer, sostituisci `renderer.render(scene, camera)` con `composer.render()`
- Commit: `feat(p4): UnrealBloomPass for glowing particles`

**B) Beat detection**: scala particelle pulsa sul kick
- Mantieni `bassBaseline = bassBaseline * .99 + audio.bass * .01`
- Se `audio.bass > bassBaseline * 1.6` → `particles.scale.setScalar(1.2)`, altrimenti `multiplyScalar(.95)`
- Commit: `feat(p4): beat-reactive scale pulse on bass`

**C) Geometry toggle**: tasto `g` cambia tra particelle e icosahedron wireframe deformato

**D) Color picker**: tasto `c` genera palette random invece di lista fissa

### Step 4.15 — Record demo
- [ ] `Cmd+Shift+5` (Mac) → registra 30-60s del browser → salva come `~/Desktop/synesthesia.mov`
- [ ] (Al ritorno con internet: converti a GIF con `ffmpeg -i input.mov -vf "fps=20,scale=720:-1" -loop 0 docs/demo.gif`)

### Step 4.16 — README
- [ ] Riscrivi README con: descrizione + demo placeholder + quickstart + architettura ASCII + controlli + stack + setup Gibber local
- [ ] Commit:
  ```
  git add README.md
  git commit -m "docs: complete README for synesthesia"
  ```

### 🎉 P4 DONE — il flagship è finito

---

## POST-VOLO — A terra con internet (15 min)

### Push tutto
```bash
for d in ~/Desktop/planeProjects/p{1_handmouse,2_gibber,3_handgibber,4_synesthesia}; do
  echo "=== pushing $(basename $d) ==="
  (cd $d && git push && git push --tags)
done
```

### Demo GIF (se hai registrato .mov)
```bash
# Richiede ffmpeg: brew install ffmpeg
cd ~/Desktop/planeProjects/p4_synesthesia && mkdir -p docs
ffmpeg -i ~/Desktop/synesthesia.mov -vf "fps=20,scale=720:-1" -loop 0 docs/demo.gif
git add docs/demo.gif
git commit -m "docs: add demo GIF"
git push
```

(Idem per p1, p2, p3 se hai registrato)

### Aggiorna umbrella `planeProjects` con riepilogo finale
- [ ] Eventualmente aggiorna README umbrella con cosa è effettivamente venuto fuori
- [ ] Push

---

## Cheat sheet

```bash
# Activate venv
source ~/Desktop/planeProjects/.venv/bin/activate

# Status di tutti i repo
for d in ~/Desktop/planeProjects/p{1_handmouse,2_gibber,3_handgibber,4_synesthesia}; do
  echo "=== $(basename $d) ==="; (cd $d && git status -s); done

# Lancia Gibber playground (per P2, P3)
cd ~/Desktop/planeProjects/_vendor/gibber && npm start          # http://127.0.0.1:9080

# Bridge handmouse->WS (per P3 e P4)
python ~/Desktop/planeProjects/p3_handgibber/server/bridge.py   # quando p3 esiste
# oppure (per p4):
python ~/Desktop/planeProjects/p4_synesthesia/server/bridge.py  # quando p4 esiste

# Vite dev server (per P4)
cd ~/Desktop/planeProjects/p4_synesthesia && npm run dev        # http://127.0.0.1:5173

# Apri reference solutions (PROJECTS.md rendered)
open ~/Desktop/planeProjects/PROJECTS.html

# Kill server orfani
lsof -ti:9080 | xargs kill -9 2>/dev/null
lsof -ti:5173 | xargs kill -9 2>/dev/null
lsof -ti:8765 | xargs kill -9 2>/dev/null
lsof -ti:8000 | xargs kill -9 2>/dev/null
```

---

## ✈️ Buon volo
