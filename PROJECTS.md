# Plane Projects — 4 demo audio/vision/3D per il portfolio

## Context

Hai ~30h offline (2 voli da 3h + 2 voli da 12h, primavera 2026) e un **MacBook Pro M1 Pro**. Vuoi 4 progetti incrementali che culminano in un visualizer 3D audio-reattivo controllato con le mani — "qualcosa che faccia rimanere sorpresi i recruiter".

Stato attuale: `~/Desktop/planeProjects/handMouse/` contiene `.git`, `.gitignore`, un `.venv` rotto e un `package.json` spurio. I file `.py` del vecchio handMouse sono cancellati dal working dir ma esistono ancora in `HEAD` del repo git — li recuperiamo prima di pulire.

**Vincoli che governano tutto il piano:**
- 100% offline a runtime durante i voli
- Codice 100% verificato (zero "vedi docs", zero "se non funziona modifica X")
- M1 Pro / Apple Silicon (ARM): tutte le dipendenze devono avere wheels arm64
- Macos richiede permessi espliciti per Webcam (OpenCV) e Accessibility (pyautogui/pynput) — da concedere a terra
- **Gibber è il motore audio in tutti e 3 i progetti che ne hanno bisogno (P2, P3, P4)** per mantenere l'aesthetic live-coding (massimo impact da portfolio). Limiti del Gibber (docs scarse, API che varia tra fork, bundle UMD) sono mitigati con: probing difensivo dell'AudioContext in P4, e **Tone.js pre-installato come Plan B** swap-in-1-riga se Gibber dà problemi specifici nell'embed Vite (`src/audio_tonejs.js` già pronto)

**Repo strategy:**
- **P1 = riuso del vecchio repo `CarloFanelli/handMouse`** già su GitHub — il commit history mostra l'evoluzione del progetto (valore portfolio: si vede che è un lavoro in corso da tempo, non buttato giù in un weekend). Taggiamo `v1-monolithic` la versione attuale, poi pushiamo i nuovi commit del refactor → tag `v2-modular` alla fine.
- **P2, P3, P4 = 3 nuove repo GitHub** create pre-volo (vuote remote + initial scaffold pushato → poi in volo committi local-only → al ritorno solo `git push`)
- **P4 è il flagship "finale"**, completamente self-contained (vendora il modulo handmouse di P1)
- I README cross-linkano la "saga": P1 → P2 → P3 → P4
- Opzionale: un 5° repo "umbrella" `creative-coding-portfolio` con thumbnail + link a tutti
- Tutte le remote sono già configurate pre-volo → in aereo solo `git commit`, al ritorno `git push`

**Allocazione tempo target:**
- Volo 1 (3h): Pre-flight di emergenza se hai dimenticato qualcosa + P1 completo
- Volo 2 (3h): P2 completo + commit/polish P1
- Volo 3 (12h): P3 completo + polish P1/P2 + setup P4
- Volo 4 (12h): P4 (il pezzo forte) + registrazioni demo

---

## PRE-FLIGHT — Da fare a terra con internet e tempo (60-90 min + 1-2h download in background)

### 0. Permessi macOS (da fare UNA volta sola, sopravvivono ai reboot)

Su macOS M1, prima di lanciare i demo Python:
1. **System Settings → Privacy & Security → Accessibility** → aggiungi e abilita: Terminal (o iTerm/Warp), Visual Studio Code (se usi il debugger). Serve a `pyautogui` e `pynput` per simulare input.
2. **System Settings → Privacy & Security → Camera** → al primo run di OpenCV macOS chiede automaticamente, dai OK.
3. Ricorda: se rilanci da un terminale "nuovo" (es. tmux dentro un altro term) potrebbe non ereditare i permessi.

### 1. Recovery dei file vecchi + RIUSO della repo originale

> Strategia: **NON cancelliamo `.git`** — il repo `github.com/CarloFanelli/handMouse` resta il repo di P1. Tagghiamo lo stato attuale come `v1-monolithic`, poi il refactor sarà una nuova serie di commit. Bonus: il recruiter vedrà 30+ commit storici + il nuovo lavoro.

```bash
cd ~/Desktop/planeProjects/handMouse

# 1.a — Recupera i file cancellati (sono ancora in HEAD)
git checkout HEAD -- README.md gesture_recognizer.task hand_recognition.py package-lock.json package.json
ls -la   # verifica: README.md, gesture_recognizer.task, hand_recognition.py, .gitignore, .venv, .git

# 1.b — Verifica remote
git remote -v
# Aspettati: origin  git@github.com:CarloFanelli/handMouse.git (fetch)
#            origin  git@github.com:CarloFanelli/handMouse.git (push)
# Se manca o è https, settalo:
# git remote set-url origin git@github.com:CarloFanelli/handMouse.git

# 1.c — Tag della milestone "v1 monolite" PRIMA del refactor
git status   # deve essere "nothing to commit, working tree clean"
git tag -a v1-monolithic -m "v1: single-file gesture/mouse demo (pre-refactor baseline)"

# 1.d — Sposta il modello in zona "shared" del workspace
mkdir -p ~/Desktop/planeProjects/_assets
cp gesture_recognizer.task ~/Desktop/planeProjects/_assets/

# 1.e — Cleanup venv rotto (pyvenv.cfg mancante, deps parziali) e package.json spurio
rm -rf .venv
rm -f package.json package-lock.json
git add -A
git commit -m "chore: remove broken venv and spurious package.json (pre-refactor cleanup)"
```

### 2. Rinomina handMouse → p1_handmouse e crea le altre cartelle

```bash
cd ~/Desktop/planeProjects

# Rinomina la cartella (porta con sé il .git → la repo CarloFanelli/handMouse resta legata)
mv handMouse p1_handmouse

# Crea le cartelle per gli altri progetti + zone shared
mkdir -p p2_gibber p3_handgibber p4_synesthesia _docs _vendor
```

**Stato dopo step 1+2:** `p1_handmouse/` mantiene il vecchio `.git` (repo `CarloFanelli/handMouse` su GitHub, tag `v1-monolithic` creato). I file recovered (`hand_recognition.py`, `README.md`, `gesture_recognizer.task`) sono nel working dir e li userai come reference / da riorganizzare nel refactor di P1.

Albero finale del workspace pre-volo:
```
~/Desktop/planeProjects/
├── _assets/                        # gesture_recognizer.task + futuri sample
├── _docs/                          # docs scaricate (Three.js, Tone.js, MDN)
├── _vendor/                        # Gibber, Three.js cache, npm cache, pip wheels
├── p1_handmouse/                   # handmouse module + 3 demo (vecchia repo riusata)
├── p2_gibber/                      # 3 sketch Gibber + mini runner
├── p3_handgibber/                  # bridge Python WS → Gibber playground
└── p4_synesthesia/                 # Three.js + Gibber audio-reactive flagship
```

### 3. Python venv (condiviso, posizionato a livello workspace)

```bash
cd ~/Desktop/planeProjects

python3.11 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip wheel setuptools

# Install (mediapipe ha wheels arm64 native da v0.10.x, funziona su M1 con Python 3.11)
pip install \
  "mediapipe>=0.10.14" \
  "opencv-python>=4.9" \
  "pyautogui>=0.9.54" \
  "numpy>=1.26,<2.0" \
  "websockets>=12" \
  "python-osc>=1.8" \
  "pynput>=1.7" \
  "rich>=13"

# Safety net: cache wheels offline
mkdir -p _vendor/wheels
pip download \
  "mediapipe>=0.10.14" "opencv-python>=4.9" "pyautogui>=0.9.54" \
  "numpy>=1.26,<2.0" "websockets>=12" "python-osc>=1.8" "pynput>=1.7" "rich>=13" \
  -d _vendor/wheels --platform macosx_11_0_arm64 --only-binary=:all: --python-version 311 || \
  echo "WARN: alcune wheels arm64 mancanti, ma installazione corrente OK"

pip freeze > requirements.txt
deactivate
```

> Se l'install di mediapipe fallisce: forza `pip install mediapipe==0.10.18 --platform macosx_11_0_arm64 --only-binary=:all:`. Su Python 3.11 + M1 il wheel arm64 è ufficiale.

### 4. Clone Gibber (per P2 e P3 — live coding in browser)

```bash
cd ~/Desktop/planeProjects/_vendor
git clone --depth 1 https://github.com/gibber-cc/gibber.git
cd gibber
npm install              # ~500MB-1GB, deve avere internet
npm run build            # genera playground/bundle.js
# Verifica che parta
npm start &              # serve http://127.0.0.1:9080
sleep 3
curl -s http://127.0.0.1:9080 | head -5
kill %1
```

> Note Gibber:
> - In **P2 e P3** lo usi via il suo playground (paste-and-run). Niente embed nostro.
> - In **P4** copieremo `playground/bundle.js` + `playground/jsdsp.js` dentro `p4_synesthesia/public/gibber/` e li caricheremo via `<script>` tag (vedi step 4.0 e 4.2). L'API esatta del master node Gibber non è perfettamente documentata, quindi `src/audio.js` di P4 fa probing difensivo + esiste `src/audio_tonejs.js` come Plan B per swap in 1 riga.
> - Esempi verificati di codice Gibber stanno in `_vendor/gibber/playground/examples/` — se in volo ti serve la sintassi esatta di un'API, guarda lì.

### 5. Installa Three.js + Tone.js + Vite + cache npm globale (per P4)

> Three.js = visualizer 3D. Tone.js = **Plan B fallback** se Gibber non collabora nell'embed. Vite = dev server. Gibber stesso non è un pacchetto npm: lo copieremo dal `_vendor/gibber/playground/bundle.js` dentro `p4_synesthesia/public/gibber/` durante lo step 4.0.

```bash
cd ~/Desktop/planeProjects/p4_synesthesia
npm init -y
npm install three tone
npm install -D vite

# CACHE npm: tarball locali per reinstallare se i node_modules si rompono
mkdir -p ~/Desktop/planeProjects/_vendor/npm-cache
cd ~/Desktop/planeProjects/_vendor/npm-cache
npm pack three tone vite   # crea three-*.tgz, tone-*.tgz, vite-*.tgz
```

### 6. Scarica documentazione offline

```bash
cd ~/Desktop/planeProjects/_docs

# Three.js: il repo contiene docs/ ed examples/ navigabili con un http server locale
git clone --depth 1 https://github.com/mrdoob/three.js.git threejs-source

# Tone.js docs: clone repo (ha sia src che esempi)
git clone --depth 1 https://github.com/Tonejs/Tone.js.git tonejs-source

# MediaPipe cheatsheet (salva queste pagine come PDF dal browser con "Stampa → Salva PDF"):
# https://ai.google.dev/edge/mediapipe/solutions/vision/gesture_recognizer/python
# https://ai.google.dev/edge/mediapipe/solutions/vision/hand_landmarker
# Salvale come _docs/mediapipe_gesture_recognizer.pdf e _docs/mediapipe_hand_landmarker.pdf

# (Opzionale ma utile) DevDocs desktop https://devdocs.io/ → abilita: JavaScript, DOM, Web Audio API, WebGL, Three.js, Tone.js
```

### 7. Git init + creazione repo GitHub pre-volo (P2, P3, P4) + push iniziale

> P1 ha già la sua repo (riusata dal vecchio handMouse). P2/P3/P4 vanno create + linkate pre-volo per poter fare solo `git push` all'atterraggio.

**7.a — Installa `gh` CLI se non l'hai:**
```bash
which gh || brew install gh
gh auth status || gh auth login   # segui prompt: GitHub.com → HTTPS → login via browser
```

**7.b — Per P2, P3, P4: git init + scaffold + crea repo remote + push:**
```bash
cd ~/Desktop/planeProjects

# Template gitignore comune
read -r -d '' GITIGNORE <<'EOF'
.venv/
node_modules/
dist/
.vite/
.DS_Store
__pycache__/
*.pyc
EOF

# Mapping cartella locale → nome repo GitHub + descrizione
declare -a PROJECTS=(
  "p2_gibber|gibber-sketches|Live-coding audio sketches for the Gibber playground"
  "p3_handgibber|handgibber|Bridge: hand gestures → Gibber audio synth via WebSocket"
  "p4_synesthesia|synesthesia|Audio-reactive 3D particle visualizer controlled by hand gestures"
)

for entry in "${PROJECTS[@]}"; do
  IFS='|' read -r dir reponame desc <<< "$entry"
  cd ~/Desktop/planeProjects/$dir
  git init -q -b main
  echo "$GITIGNORE" > .gitignore
  cat > README.md <<EOF
# $reponame

$desc

> Work in progress — scaffold created pre-flight, implementation during a 30-hour offline flight stretch.
EOF
  git add .gitignore README.md
  git commit -q -m "chore: initial scaffold"

  # Crea repo su GitHub e aggiunge remote (NON pusha file ancora, lo facciamo in 7.c)
  gh repo create "$reponame" --public --source=. --remote=origin --description "$desc" --disable-issues=false
done
```

**7.c — Push iniziale (così la remote è popolata e i commit del volo si pushano puliti al ritorno):**
```bash
for entry in "${PROJECTS[@]}"; do
  IFS='|' read -r dir _ _ <<< "$entry"
  cd ~/Desktop/planeProjects/$dir
  git push -u origin main
done
```

**7.d — Per P1: verifica che sei pronto a pushare il nuovo commit di cleanup + il tag**

Non pushiamo ancora (lo fai al ritorno per evitare ondate parziali), ma verifica che la connection funzioni:
```bash
cd ~/Desktop/planeProjects/p1_handmouse
git fetch origin   # deve completare senza errori
git log --oneline -5
git tag -l         # deve mostrare v1-monolithic
# Quando vorrai pushare:
#   git push origin main
#   git push origin v1-monolithic
```

**Riepilogo repo GitHub dopo questa fase:**
- `github.com/CarloFanelli/handMouse` (esistente, riusato per P1) — locale ha 1 commit nuovo + tag pronto, push al ritorno
- `github.com/CarloFanelli/gibber-sketches` (P2, creato e pushato scaffold)
- `github.com/CarloFanelli/handgibber` (P3, creato e pushato scaffold)
- `github.com/CarloFanelli/synesthesia` (P4, creato e pushato scaffold)

### 8. Verifica finale offline (CRITICA, fai questa)

```bash
# Disattiva il wifi dalla menu bar
cd ~/Desktop/planeProjects
source .venv/bin/activate

# Test Python
python -c "import mediapipe, cv2, pyautogui, numpy, websockets, pynput; print('OK Python')"

# Test webcam (apre finestra, ESC per chiudere)
python -c "
import cv2
cap = cv2.VideoCapture(0)
ok, f = cap.read()
print('camera OK' if ok else 'CAMERA FAIL')
cap.release()
"

# Test Gibber offline
cd _vendor/gibber && npm start &
sleep 3
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:9080
kill %1

# Test Tone+Three+Vite offline
cd ~/Desktop/planeProjects/p4_synesthesia
npm run dev -- --host 127.0.0.1 &
sleep 5
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:5173
kill %1
```

Se tutti i test rispondono `200` o "OK" senza errori di rete → puoi imbarcarti.

### 9. Copia questo piano nel workspace

```bash
cp /Users/carlofanelli/.claude/plans/allora-ti-spiego-andr-dapper-pumpkin.md \
   ~/Desktop/planeProjects/PROJECTS.md
```

---

## Progetto 1 — `p1_handmouse`: HandMouse Pro

### Vision
Trasformare il vecchio monolite `hand_recognition.py` in un **modulo Python riusabile** con API a callback. 3 demo dimostrano l'utilità: mouse-control, slide-presenter, air-keyboard. Il modulo verrà importato da P3 e P4.

Differenza chiave: il vecchio file aveva tutto nel main loop con variabili globali. Qui `HandTracker` è una **classe libreria** che emette eventi (`on_landmarks`, `on_pinch`, `on_click`, `on_swipe`, `on_gesture`); le demo attaccano listener e basta.

### Struttura finale (dopo il refactor)
```
p1_handmouse/                       # = vecchio repo CarloFanelli/handMouse riusato
├── .git/                           # repo storica preservata, tag v1-monolithic già fatto
├── handmouse/                      # nuovo modulo
│   ├── __init__.py
│   ├── tracker.py                  # HandTracker + cv2 loop
│   ├── gestures.py                 # pinch, click, swipe + thresholds
│   └── geometry.py                 # HandPoint, OneEuroFilter, Calibration
├── demos/                          # nuove demo
│   ├── __init__.py
│   ├── mouse.py
│   ├── presenter.py
│   └── air_keyboard.py
├── legacy/                         # archivio v1 (per chi vuole guardare l'originale)
│   ├── hand_recognition.py
│   └── README.md
├── gesture_recognizer.task         # symlink → ../_assets/gesture_recognizer.task
├── requirements.txt                # symlink → ../requirements.txt
└── README.md                       # riscritto per la v2
```

### Step 1.0 — Archivia il vecchio codice (dentro lo stesso repo)

```bash
cd ~/Desktop/planeProjects/p1_handmouse
mkdir -p legacy
git mv hand_recognition.py legacy/hand_recognition.py
git mv README.md            legacy/README.md
# Il file gesture_recognizer.task lo abbiamo già copiato in ../_assets/
# Rimuoviamo la copia locale e mettiamo un symlink
git rm gesture_recognizer.task
ln -sf ../_assets/gesture_recognizer.task .
ln -sf ../requirements.txt .
git add gesture_recognizer.task requirements.txt
git commit -m "refactor: archive v1 monolith into legacy/, switch model to shared symlink"
```

### Step 1.1 — Crea le directory del nuovo modulo

```bash
cd ~/Desktop/planeProjects/p1_handmouse
mkdir -p handmouse demos
```

### Step 1.2 — `handmouse/geometry.py`

```python
"""Punti 3D, low-pass adattivo (OneEuro), calibration camera→screen."""
from dataclasses import dataclass
import math


@dataclass
class HandPoint:
    x: float = 0.0
    y: float = 0.0
    z: float = 0.0

    def distance_to(self, other: "HandPoint") -> float:
        return math.sqrt(
            (self.x - other.x) ** 2
            + (self.y - other.y) ** 2
            + (self.z - other.z) ** 2
        )

    def midpoint(self, other: "HandPoint") -> "HandPoint":
        return HandPoint(
            (self.x + other.x) / 2,
            (self.y + other.y) / 2,
            (self.z + other.z) / 2,
        )


class OneEuroFilter:
    """Casiez et al. 2012. min_cutoff bassa = più smooth a riposo,
    beta alta = meno lag su movimenti veloci."""

    def __init__(self, min_cutoff: float = 1.0, beta: float = 0.02, d_cutoff: float = 1.0):
        self.min_cutoff = min_cutoff
        self.beta = beta
        self.d_cutoff = d_cutoff
        self._x_prev: float | None = None
        self._dx_prev: float = 0.0
        self._t_prev: float | None = None

    @staticmethod
    def _alpha(cutoff: float, dt: float) -> float:
        tau = 1.0 / (2.0 * math.pi * cutoff)
        return 1.0 / (1.0 + tau / dt)

    def __call__(self, x: float, t: float) -> float:
        if self._x_prev is None or self._t_prev is None:
            self._x_prev = x
            self._t_prev = t
            return x
        dt = max(t - self._t_prev, 1e-6)
        dx = (x - self._x_prev) / dt
        a_d = self._alpha(self.d_cutoff, dt)
        dx_hat = a_d * dx + (1.0 - a_d) * self._dx_prev
        cutoff = self.min_cutoff + self.beta * abs(dx_hat)
        a = self._alpha(cutoff, dt)
        x_hat = a * x + (1.0 - a) * self._x_prev
        self._x_prev = x_hat
        self._dx_prev = dx_hat
        self._t_prev = t
        return x_hat


class Calibration:
    """Mappa una "zona comfort" della camera (es. quadrato 0.2..0.8) sull'intero schermo."""

    def __init__(self, x_min: float = 0.2, x_max: float = 0.8,
                 y_min: float = 0.2, y_max: float = 0.8):
        self.x_min, self.x_max = x_min, x_max
        self.y_min, self.y_max = y_min, y_max

    def map_point(self, x: float, y: float) -> tuple[float, float]:
        nx = (x - self.x_min) / (self.x_max - self.x_min)
        ny = (y - self.y_min) / (self.y_max - self.y_min)
        return max(0.0, min(1.0, nx)), max(0.0, min(1.0, ny))
```

### Step 1.3 — `handmouse/gestures.py`

```python
"""Detector pinch/click/swipe. Thresholds tarati sui valori normalizzati MediaPipe."""
from collections import deque
from .geometry import HandPoint

PINCH_THRESHOLD = 0.06   # indice ↔ pollice
CLICK_THRESHOLD = 0.045  # medio  ↔ pollice


def is_pinching(index_tip: HandPoint, thumb_tip: HandPoint) -> tuple[bool, float]:
    d = index_tip.distance_to(thumb_tip)
    return d < PINCH_THRESHOLD, d


def is_clicking(middle_tip: HandPoint, thumb_tip: HandPoint) -> tuple[bool, float]:
    d = middle_tip.distance_to(thumb_tip)
    return d < CLICK_THRESHOLD, d


class SwipeDetector:
    """Swipe orizzontale sulla traiettoria dell'ultimo handful di frame."""

    def __init__(self, history: int = 10, min_dx: float = 0.25, max_dy: float = 0.10):
        self.buf: deque[tuple[float, float]] = deque(maxlen=history)
        self.min_dx = min_dx
        self.max_dy = max_dy
        self._cooldown = 0

    def update(self, x: float, y: float) -> str | None:
        self.buf.append((x, y))
        if self._cooldown > 0:
            self._cooldown -= 1
            return None
        if len(self.buf) < self.buf.maxlen:
            return None
        xs = [p[0] for p in self.buf]
        ys = [p[1] for p in self.buf]
        dx = xs[-1] - xs[0]
        dy = max(ys) - min(ys)
        if dy > self.max_dy:
            return None
        if dx > self.min_dx:
            self._cooldown = 15
            self.buf.clear()
            return "right"
        if dx < -self.min_dx:
            self._cooldown = 15
            self.buf.clear()
            return "left"
        return None
```

### Step 1.4 — `handmouse/tracker.py` (cuore)

```python
"""HandTracker: wrap MediaPipe Tasks GestureRecognizer + cv2 loop + event bus.

Esempio uso:
    tracker = HandTracker(model_path="gesture_recognizer.task")
    tracker.on("landmarks", lambda pts, res: print(pts["index_tip"]))
    tracker.on("pinch", lambda center, dist: print("pinch", dist))
    tracker.run()  # bloccante, ESC per uscire
"""
from __future__ import annotations
from collections import defaultdict
from typing import Callable

import cv2
import mediapipe as mp
from mediapipe.tasks import python as mp_python
from mediapipe.tasks.python import vision as mp_vision

from .geometry import HandPoint
from .gestures import is_pinching, is_clicking, SwipeDetector


# Indici dei landmark MediaPipe Hands
LM_THUMB_TIP = 4
LM_INDEX_TIP = 8
LM_INDEX_BASE = 5
LM_MIDDLE_TIP = 12


class HandTracker:
    def __init__(
        self,
        model_path: str = "gesture_recognizer.task",
        num_hands: int = 1,
        camera_index: int = 0,
        mirror: bool = True,
        show_window: bool = True,
        window_name: str = "HandTracker",
    ):
        self.mirror = mirror
        self.show_window = show_window
        self.window_name = window_name

        opts = mp_vision.GestureRecognizerOptions(
            base_options=mp_python.BaseOptions(model_asset_path=model_path),
            num_hands=num_hands,
            min_hand_detection_confidence=0.5,
            min_hand_presence_confidence=0.5,
        )
        self.recognizer = mp_vision.GestureRecognizer.create_from_options(opts)
        self.cap = cv2.VideoCapture(camera_index)
        if not self.cap.isOpened():
            raise RuntimeError(f"Cannot open camera index {camera_index}")

        self.swipe = SwipeDetector()
        self._was_pinching = False
        self._was_clicking = False

        self._handlers: dict[str, list[Callable]] = defaultdict(list)

    # ---------- event bus ----------
    def on(self, event: str, handler: Callable) -> "HandTracker":
        """event ∈ {landmarks, pinch, pinch_release, click, click_release,
                    swipe, gesture, frame}"""
        self._handlers[event].append(handler)
        return self

    def _emit(self, event: str, *args) -> None:
        for h in self._handlers[event]:
            try:
                h(*args)
            except Exception as e:  # listener non deve killare il loop
                print(f"[HandTracker] handler '{event}' error: {e}")

    # ---------- landmark extraction ----------
    def _extract(self, hand_landmarks) -> dict[str, HandPoint]:
        def pt(i: int) -> HandPoint:
            lm = hand_landmarks[i]
            x = (1.0 - lm.x) if self.mirror else lm.x
            return HandPoint(x, lm.y, lm.z)
        return {
            "thumb_tip":  pt(LM_THUMB_TIP),
            "index_tip":  pt(LM_INDEX_TIP),
            "index_base": pt(LM_INDEX_BASE),
            "middle_tip": pt(LM_MIDDLE_TIP),
        }

    # ---------- main loop ----------
    def run(self) -> None:
        try:
            while True:
                ok, img = self.cap.read()
                if not ok:
                    continue

                img_rgb = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
                mp_img = mp.Image(image_format=mp.ImageFormat.SRGB, data=img_rgb)
                result = self.recognizer.recognize(mp_img)

                self._emit("frame", img, result)

                if not result.hand_landmarks:
                    if self._was_pinching:
                        self._emit("pinch_release"); self._was_pinching = False
                    if self._was_clicking:
                        self._emit("click_release"); self._was_clicking = False
                else:
                    pts = self._extract(result.hand_landmarks[0])
                    self._emit("landmarks", pts, result)

                    if result.gestures and result.gestures[0]:
                        g = result.gestures[0][0]
                        self._emit("gesture", g.category_name, g.score)

                    pinching, pdist = is_pinching(pts["index_tip"], pts["thumb_tip"])
                    if pinching and not self._was_pinching:
                        self._emit("pinch", pts["index_tip"].midpoint(pts["thumb_tip"]), pdist)
                        self._was_pinching = True
                    elif not pinching and self._was_pinching:
                        self._emit("pinch_release"); self._was_pinching = False

                    clicking, _ = is_clicking(pts["middle_tip"], pts["thumb_tip"])
                    if clicking and not self._was_clicking:
                        self._emit("click", pts["middle_tip"].midpoint(pts["thumb_tip"]))
                        self._was_clicking = True
                    elif not clicking and self._was_clicking:
                        self._emit("click_release"); self._was_clicking = False

                    swipe = self.swipe.update(pts["index_base"].x, pts["index_base"].y)
                    if swipe:
                        self._emit("swipe", swipe)

                if self.show_window:
                    disp = cv2.flip(img, 1) if self.mirror else img
                    cv2.imshow(self.window_name, disp)
                    if cv2.waitKey(2) & 0xFF == 27:  # ESC
                        break
        finally:
            self.cap.release()
            cv2.destroyAllWindows()
```

### Step 1.5 — `handmouse/__init__.py`

```python
from .tracker import HandTracker
from .geometry import HandPoint, OneEuroFilter, Calibration
from .gestures import SwipeDetector, is_pinching, is_clicking

__all__ = [
    "HandTracker",
    "HandPoint",
    "OneEuroFilter",
    "Calibration",
    "SwipeDetector",
    "is_pinching",
    "is_clicking",
]
```

### Step 1.6 — `demos/__init__.py`

```python
# empty: lo serve solo a python -m demos.X
```

### Step 1.7 — `demos/mouse.py`

```python
"""Demo 1: muovi il mouse con la punta dell'indice, click con pinch medio↔pollice."""
import time
import pyautogui
from handmouse import HandTracker, OneEuroFilter, Calibration

pyautogui.FAILSAFE = False
pyautogui.PAUSE = 0.0
SCREEN_W, SCREEN_H = pyautogui.size()

tracker = HandTracker()
calib = Calibration(0.20, 0.80, 0.20, 0.80)
fx, fy = OneEuroFilter(min_cutoff=1.5, beta=0.05), OneEuroFilter(min_cutoff=1.5, beta=0.05)


def on_landmarks(pts, _result):
    t = time.time()
    nx, ny = calib.map_point(pts["index_tip"].x, pts["index_tip"].y)
    sx = fx(nx * SCREEN_W, t)
    sy = fy(ny * SCREEN_H, t)
    pyautogui.moveTo(sx, sy, duration=0)


tracker.on("landmarks", on_landmarks)
tracker.on("click", lambda _center: pyautogui.click())
print("HandMouse demo — ESC nella finestra video per uscire.")
tracker.run()
```

### Step 1.8 — `demos/presenter.py`

```python
"""Demo 2: telecomando slide.
- Swipe destra/sinistra → freccia destra/sinistra (cambia slide)
- Closed_Fist → F5 (avvia presentazione)
- Open_Palm  → Esc (esce)"""
from pynput.keyboard import Controller, Key
from handmouse import HandTracker

kb = Controller()
tracker = HandTracker()

GESTURE_KEYS = {
    "Closed_Fist": Key.f5,
    "Open_Palm":   Key.esc,
}


def tap(key) -> None:
    kb.press(key); kb.release(key)


tracker.on("swipe",   lambda d: tap(Key.right if d == "right" else Key.left))
def on_gesture(name, score):
    if score < 0.7:
        return
    if name in GESTURE_KEYS:
        tap(GESTURE_KEYS[name])
tracker.on("gesture", on_gesture)

print("Presenter — apri Keynote/PowerPoint a fianco, ESC nel video per uscire.")
tracker.run()
```

### Step 1.9 — `demos/air_keyboard.py`

```python
"""Demo 3: tastiera virtuale 3x3.
Layout (mirror, come ti vedi nella webcam):
   q w e
   a s d
   z x c
Hover una zona con l'indice, pinch indice↔pollice = scrive il carattere."""
import time
from pynput.keyboard import Controller
from handmouse import HandTracker, Calibration

kb = Controller()
tracker = HandTracker()
calib = Calibration(0.15, 0.85, 0.15, 0.85)

LAYOUT = [
    ["q", "w", "e"],
    ["a", "s", "d"],
    ["z", "x", "c"],
]

state = {"cell": None, "last": 0.0}


def cell_at(nx: float, ny: float):
    if not (0.0 <= nx <= 1.0 and 0.0 <= ny <= 1.0):
        return None
    col = min(2, int(nx * 3))
    row = min(2, int(ny * 3))
    return row, col


def on_landmarks(pts, _r):
    nx, ny = calib.map_point(pts["index_tip"].x, pts["index_tip"].y)
    state["cell"] = cell_at(nx, ny)


def on_pinch(_center, _dist):
    if state["cell"] is None:
        return
    if time.time() - state["last"] < 0.4:
        return
    r, c = state["cell"]
    ch = LAYOUT[r][c]
    print("type", ch)
    kb.type(ch)
    state["last"] = time.time()


tracker.on("landmarks", on_landmarks)
tracker.on("pinch", on_pinch)
print("Air Keyboard — apri TextEdit, ESC nel video per uscire.")
tracker.run()
```

### Step 1.10 — Riscrivi il README (heredoc così non devi escapare i backtick)

```bash
cd ~/Desktop/planeProjects/p1_handmouse
cat > README.md <<'README_EOF'
# handmouse

Reusable hand-tracking module for Python (MediaPipe + OpenCV) with a callback-based API.
Three included demos: mouse-control, presenter remote, air keyboard 3x3.

> v2 modular refactor — see [`legacy/`](legacy/) for the original single-file v1.

## Demo
![mouse demo](docs/mouse.gif)

## Quickstart
```bash
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python -m demos.mouse        # ESC in the video window to quit
python -m demos.presenter
python -m demos.air_keyboard
```

## API
```python
from handmouse import HandTracker
tracker = HandTracker()
tracker.on("pinch",  lambda center, dist: print("pinch", dist))
tracker.on("swipe",  lambda direction:    print("swipe", direction))
tracker.on("gesture", lambda name, score: print("gesture", name, score))
tracker.run()  # ESC to exit
```

## Stack
Python 3.11 · MediaPipe Tasks (gesture_recognizer) · OpenCV · pyautogui · pynput

## Part of the planeProjects portfolio
[handmouse](https://github.com/CarloFanelli/handMouse) → [gibber-sketches](https://github.com/CarloFanelli/gibber-sketches) → [handgibber](https://github.com/CarloFanelli/handgibber) → [synesthesia](https://github.com/CarloFanelli/synesthesia)
README_EOF
```

### Verifica P1
```bash
cd ~/Desktop/planeProjects
source .venv/bin/activate
cd p1_handmouse
python -m demos.mouse
python -m demos.presenter
python -m demos.air_keyboard
git add . && git commit -m "feat: handmouse module + 3 demos (v2 modular refactor)"
git tag -a v2-modular -m "v2: modular refactor (HandTracker module + callback API + 3 demos)"
# push al ritorno: git push origin main && git push origin v2-modular
```

---

## Progetto 2 — `p2_gibber`: Sketches per il playground

### Vision
**Gibber è ottimo come live-coding environment** (scrivi codice in browser, lo esegui inline, senti il risultato) ma non è una libreria che embedi in app proprie (bundle UMD globale, API non perfettamente documentata). Quindi P2 è una **collezione di sketch** che incolli nel playground locale di Gibber e suoni.

Il "deliverable" è un piccolo repo con:
- `sketches/*.js` — file pronti da copia-incollare nel playground
- `runner.html` — paginetta che li lista con bottone "Copy" per sketch
- README con GIF/screenshot

> Verità importante: useremo **soltanto pattern Gibber documentati nel playground stesso** (i bottoni "Tutorials/Examples" nel playground locale ti danno snippet già verificati). I 3 sketch che seguono sono volutamente minimali e usano solo API che la mia memoria conferma stabili (`Synth`, `Drums`, `FX.add`, `Clock.bpm`, `note.seq`). Se uno sketch dovesse non funzionare, hai i `_vendor/gibber/playground/examples/` come backup di esempi verificati.

### Struttura
```
p2_gibber/
├── sketches/
│   ├── 01_groove.js
│   ├── 02_acid.js
│   └── 03_drone.js
├── runner/
│   ├── index.html
│   └── runner.js
└── README.md
```

### Sketch 1 — `sketches/01_groove.js`

```javascript
// Un groove minimale four-on-the-floor + hat + bass
Clock.bpm = 124

kick = Drums('x*x*x*x*').gain(.9)
hat  = Drums('-*-*-*-*').gain(.4).fx.add( Reverb(.3) )

bass = Synth('bleep').gain(.5).fx.add( Filter24({ cutoff: .35, Q: .8 }) )
bass.note.seq([0, 0, 7, 5], 1/4)

// Stop tutto con: Gibber.clear()
```

### Sketch 2 — `sketches/02_acid.js`

```javascript
// Acid bassline con cutoff che modula
Clock.bpm = 132

bass = Synth('bleep').gain(.6).fx.add( Filter24({ cutoff: .25, Q: .95 }) )
bass.note.seq([0, 3, 5, 7, 5, 3, 0, -5], 1/8)

// Filter sweep ogni 2 misure
bass.fx[0].cutoff.seq([.2, .45, .7, .9, .5, .3], Measure(.5))

kick = Drums('x*x*x*x*').gain(.8)
```

### Sketch 3 — `sketches/03_drone.js`

```javascript
// Ambient drone con due voci FM
Clock.bpm = 60

pad1 = FM({ cmRatio: 1.5, index: 4 }).gain(.4).fx.add( Reverb(.85), Delay(1/4) )
pad1.note.seq([0, 3, 7, 10, 5], Measure(2))

pad2 = FM({ cmRatio: .5, index: 2 }).gain(.3)
pad2.note.seq([-12, -7, -5], Measure(3))
```

### Runner — `runner/index.html`

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>p2_gibber sketches</title>
  <style>
    body { font-family: ui-monospace, monospace; background:#0f0f0f; color:#eaeaea; padding: 2rem; max-width: 800px; margin: auto; }
    h1 { color: #6cf; }
    pre { background:#1a1a1a; padding: 1rem; border-left: 3px solid #6cf; overflow:auto; }
    button { background:#6cf; color:#000; border:0; padding:.5rem 1rem; font-family: inherit; cursor:pointer; margin-bottom: 2rem; }
    a { color: #6cf; }
  </style>
</head>
<body>
  <h1>p2_gibber — Gibber sketches</h1>
  <p>1. Apri Gibber locale: <a href="http://127.0.0.1:9080" target="_blank">http://127.0.0.1:9080</a><br>
     2. Copia uno sketch qui sotto, incolla nell'editor, <kbd>Cmd</kbd>+<kbd>Enter</kbd> per eseguire la riga / <kbd>Cmd</kbd>+<kbd>Shift</kbd>+<kbd>Enter</kbd> per eseguire tutto.<br>
     3. Stop: <code>Gibber.clear()</code>.</p>
  <div id="list"></div>
  <script type="module" src="./runner.js"></script>
</body>
</html>
```

### Runner — `runner/runner.js`

```javascript
const files = ['01_groove.js', '02_acid.js', '03_drone.js']
const list = document.getElementById('list')

for (const f of files) {
  const src = await fetch(`../sketches/${f}`).then(r => r.text())
  const wrap = document.createElement('section')
  const pre  = document.createElement('pre')
  pre.textContent = src
  const h2 = document.createElement('h2'); h2.textContent = f
  const btn = document.createElement('button')
  btn.textContent = 'Copy'
  btn.onclick = () => navigator.clipboard.writeText(src).then(
    () => { btn.textContent = 'Copied ✓'; setTimeout(() => btn.textContent = 'Copy', 1500) }
  )
  wrap.append(h2, btn, pre)
  list.appendChild(wrap)
}
```

### Come provarlo

```bash
# Terminal A: Gibber playground
cd ~/Desktop/planeProjects/_vendor/gibber && npm start
# → http://127.0.0.1:9080

# Terminal B: il tuo runner
cd ~/Desktop/planeProjects/p2_gibber
python3 -m http.server 8000
# → http://127.0.0.1:8000/runner/
```

### Polish (se avanza tempo)
- 2 sketch extra esplorando `Marching()` (visual generativo di Gibber)
- Registra MP4 di 30s per sketch con QuickTime (`Cmd+Shift+5`)

### Commit
```bash
cd ~/Desktop/planeProjects/p2_gibber
git add . && git commit -m "feat(p2): 3 gibber sketches + runner"
```

---

## Progetto 3 — `p3_handgibber`: Mani che suonano (via Gibber playground)

### Vision
Bridge tra HandTracker Python e Gibber. **Architettura volutamente semplice**: un server Python apre WebSocket e broadcasta i dati mano; il "client" è una sketch che incolli nel playground locale di Gibber, che apre una WebSocket dalla stessa origin (localhost) e modula i parametri del synth in tempo reale.

```
[Python: bridge.py]
   HandTracker → emit landmarks/pinch/swipe
   WS server on :8765 → broadcast JSON
              │
              ▼  ws://127.0.0.1:8765
[Browser: Gibber playground (localhost:9080)]
   sketch incollata:
     const ws = new WebSocket(...)
     ws.onmessage = m => { /* modula synth Gibber */ }
```

Vantaggi di questa architettura:
- Zero custom HTML/bundle: usi Gibber così com'è
- Il bridge Python è riusabile in P4
- Il messaggio JSON è semplice (`{type, x, y, ...}`)

### Struttura
```
p3_handgibber/
├── server/
│   ├── bridge.py
│   └── README.md
├── sketches/
│   ├── 01_theremin.js       # x→pitch, y→volume
│   ├── 02_drumpad.js        # pinch in zona → trigger kick/snare/hat
│   └── 03_acid_lab.js       # y→cutoff, x→Q, swipe→cambia pattern
└── README.md
```

### Step 3.1 — `server/bridge.py`

```python
"""Bridge: HandTracker → WebSocket broadcast.

Run:
    cd ~/Desktop/planeProjects && source .venv/bin/activate
    python p3_handgibber/server/bridge.py

Poi nel browser apri http://127.0.0.1:9080 (Gibber) e incolla una sketch da ../sketches/.
"""
from __future__ import annotations
import asyncio
import json
import sys
import pathlib
import threading

import websockets

# Aggiungi p1_handmouse al path
HERE = pathlib.Path(__file__).resolve().parent
ROOT = HERE.parent.parent
sys.path.insert(0, str(ROOT / "p1_handmouse"))

from handmouse import HandTracker  # noqa: E402

HOST, PORT = "127.0.0.1", 8765
MODEL = str(ROOT / "_assets" / "gesture_recognizer.task")


async def _amain() -> None:
    clients: set[websockets.WebSocketServerProtocol] = set()

    async def handler(ws):
        clients.add(ws)
        print(f"[bridge] client connected ({len(clients)} total)")
        try:
            async for _ in ws:  # ignora messaggi dal client
                pass
        except websockets.ConnectionClosed:
            pass
        finally:
            clients.discard(ws)
            print(f"[bridge] client disconnected ({len(clients)} total)")

    loop = asyncio.get_running_loop()

    def broadcast_sync(msg: dict) -> None:
        if not clients:
            return
        data = json.dumps(msg)
        async def send_all():
            await asyncio.gather(
                *(c.send(data) for c in list(clients)),
                return_exceptions=True,
            )
        asyncio.run_coroutine_threadsafe(send_all(), loop)

    # --- Avvia HandTracker in thread separato (cv2 è bloccante) ---
    def run_tracker() -> None:
        tracker = HandTracker(model_path=MODEL, show_window=True)
        tracker.on("landmarks", lambda pts, _r: broadcast_sync({
            "type": "hand",
            "x": pts["index_tip"].x,
            "y": pts["index_tip"].y,
            "thumb_x": pts["thumb_tip"].x,
            "thumb_y": pts["thumb_tip"].y,
            "mid_x":   pts["middle_tip"].x,
            "mid_y":   pts["middle_tip"].y,
        }))
        tracker.on("pinch",         lambda c, d: broadcast_sync({"type": "event", "name": "pinch", "x": c.x, "y": c.y}))
        tracker.on("pinch_release", lambda:      broadcast_sync({"type": "event", "name": "pinch_release"}))
        tracker.on("click",         lambda c:    broadcast_sync({"type": "event", "name": "click", "x": c.x, "y": c.y}))
        tracker.on("swipe",         lambda d:    broadcast_sync({"type": "event", "name": "swipe", "dir": d}))
        tracker.on("gesture",       lambda n, s: broadcast_sync({"type": "event", "name": "gesture", "label": n, "score": s}))
        tracker.run()

    t = threading.Thread(target=run_tracker, daemon=True)
    t.start()

    print(f"[bridge] WS listening on ws://{HOST}:{PORT}")
    async with websockets.serve(handler, HOST, PORT):
        await asyncio.Future()  # run forever


def main() -> None:
    try:
        asyncio.run(_amain())
    except KeyboardInterrupt:
        pass


if __name__ == "__main__":
    main()
```

### Step 3.2 — Sketch Gibber lato client

**`sketches/01_theremin.js`** — copia in Gibber playground (Cmd+Shift+Enter per run-all):
```javascript
// Theremin: indice X → pitch midi 48..84, Y → volume
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

**`sketches/02_drumpad.js`**:
```javascript
// Drum pad: pinch + zona X → kick/snare/hat
kick  = Drums('x').gain(.9)   // pattern singolo da triggerare via .play()/.note?
// Nota: l'API esatta di "trigger one-shot" in gibber varia.
// Pattern semplice e affidabile: usa Synth con .note() per i drum, percussivi:
kk = Synth('kick').gain(.9)
sn = Synth('snare').gain(.7).fx.add( Reverb(.3) )
hh = Synth('hat').gain(.4)

if (window._hgWs) window._hgWs.close()
window._hgWs = new WebSocket('ws://127.0.0.1:8765')
window._hgWs.onmessage = e => {
  const m = JSON.parse(e.data)
  if (m.type === 'event' && m.name === 'pinch') {
    if      (m.x < .33) kk.note(36)
    else if (m.x < .66) sn.note(60)
    else                hh.note(72)
  }
}
// Stop: window._hgWs.close(); Gibber.clear()
```

> Se `Synth('kick'|'snare'|'hat')` non esiste nel tuo build di Gibber, alternativa: usa un Synth percussivo con env breve, es. `Synth('square.perc').note(36)` per il kick. Gli esempi nel playground locale `_vendor/gibber/playground/examples/` ti danno la sintassi esatta del tuo build.

**`sketches/03_acid_lab.js`**:
```javascript
// Acid lab: bassline che gira sempre, Y modula cutoff filter, X modula Q, swipe cambia pattern
Clock.bpm = 130

bass = Synth('bleep').gain(.55).fx.add( Filter24({ cutoff: .3, Q: .9 }) )
const PATTERNS = [
  [0, 3, 5, 7, 5, 3, 0, -5],
  [0, 7, 0, 5, 0, 3, 0, 12],
  [0, -5, 3, -5, 5, -5, 7, -5],
]
let pIdx = 0
bass.note.seq(PATTERNS[pIdx], 1/8)
kick = Drums('x*x*x*x*').gain(.8)

if (window._hgWs) window._hgWs.close()
window._hgWs = new WebSocket('ws://127.0.0.1:8765')
window._hgWs.onmessage = e => {
  const m = JSON.parse(e.data)
  if (m.type === 'hand') {
    bass.fx[0].cutoff = .15 + (1 - m.y) * .8
    bass.fx[0].Q      = .4 + m.x * .55
  }
  if (m.type === 'event' && m.name === 'swipe') {
    pIdx = (pIdx + (m.dir === 'right' ? 1 : -1) + PATTERNS.length) % PATTERNS.length
    bass.note.seq(PATTERNS[pIdx], 1/8)
    console.log('pattern', pIdx)
  }
}
// Stop: window._hgWs.close(); Gibber.clear()
```

### Come provarlo

```bash
# Terminal A: bridge Python
cd ~/Desktop/planeProjects && source .venv/bin/activate
python p3_handgibber/server/bridge.py

# Terminal B: Gibber playground
cd ~/Desktop/planeProjects/_vendor/gibber && npm start

# Browser:
# 1. http://127.0.0.1:9080
# 2. Copia il contenuto di p3_handgibber/sketches/01_theremin.js
# 3. Cmd+Shift+Enter → muovi la mano davanti alla webcam
# 4. Per cambiare sketch: Gibber.clear(), incolla un altro file
```

### Commit
```bash
cd ~/Desktop/planeProjects/p3_handgibber
git add . && git commit -m "feat(p3): WS bridge + 3 Gibber playground sketches"
```

---

## Progetto 4 — `p4_synesthesia`: il flagship audio-reattivo 3D

### Vision
Un'esperienza completa, **clonabile da chiunque + `npm install + npm run dev`** che funziona. Three.js per il 3D + **Gibber per l'audio** (mantieni l'aesthetic live-coding) + Web Audio AnalyserNode per la reattività. Le mani controllano sia il visualizer sia il synth Gibber. Tone.js è pre-installato come Plan B (vedi "Plan B" più sotto) se Gibber dà problemi specifici nell'embed.

Effetto: una galassia di particelle deformata da un noise field, pulsante sul kick, color palette che cambia su swipe, esplosione su pinch. Quando suoni con le mani, vedi anche il risultato 3D.

```
[bridge.py Python]
     │ WS :8765
     ▼
[Vite dev server :5173]
   ├─ index.html (carica gibber bundle via <script src="/gibber/bundle.js">)
   ├─ src/main.js (Three.js render loop)
   ├─ src/audio.js (Gibber synth + AnalyserNode su AudioContext)
   ├─ src/hands.js (WS client)
   └─ src/particles.js (shader Three.js)
```

### Struttura
```
p4_synesthesia/
├── index.html
├── package.json
├── vite.config.js
├── public/
│   └── gibber/                # bundle.js + jsdsp.js copiati da _vendor/gibber/playground/
├── src/
│   ├── main.js
│   ├── audio.js               # Gibber primary
│   ├── audio_tonejs.js        # Plan B fallback già pronto
│   ├── hands.js
│   ├── particles.js
│   └── shaders/
│       ├── particle.vert.glsl
│       └── particle.frag.glsl
├── vendor/
│   ├── handmouse/             # copia (non symlink) di p1_handmouse/handmouse
│   └── gesture_recognizer.task
├── server/
│   └── bridge.py              # copia di p3_handgibber/server/bridge.py (path adattati)
└── README.md
```

### Step 4.0 — Vendor handmouse + bridge + Gibber bundle (self-contained per il flagship)

```bash
cd ~/Desktop/planeProjects/p4_synesthesia

# 4.0.a — Vendora handmouse + modello + bridge
mkdir -p vendor server public/gibber
cp -r ../p1_handmouse/handmouse ./vendor/handmouse
cp ../_assets/gesture_recognizer.task ./vendor/
cp ../p3_handgibber/server/bridge.py ./server/bridge.py

# 4.0.b — Copia il bundle Gibber dentro public/ (Vite serve public/* a root URL)
cp ../_vendor/gibber/playground/bundle.js  ./public/gibber/bundle.js
cp ../_vendor/gibber/playground/jsdsp.js   ./public/gibber/jsdsp.js
# Verifica che entrambi i file ci siano e non siano vuoti
ls -la public/gibber/
```

Apri `server/bridge.py` con un editor e applica due modifiche esatte (le righe da cambiare sono nelle prime ~20 righe):

```python
# RIGA DA CAMBIARE (era):
#   sys.path.insert(0, str(ROOT / "p1_handmouse"))
# DIVENTA:
sys.path.insert(0, str(HERE.parent / "vendor"))

# RIGA DA CAMBIARE (era):
#   MODEL = str(ROOT / "_assets" / "gesture_recognizer.task")
# DIVENTA:
MODEL = str(HERE.parent / "vendor" / "gesture_recognizer.task")
```

> Le variabili `HERE` e `ROOT` sono definite all'inizio del file. Dopo l'edit, `bridge.py` cerca handmouse e il modello dentro `vendor/` invece che nei path del workspace condiviso — così la repo `synesthesia` è 100% self-contained per chiunque la cloni.

### Step 4.1 — `package.json` e `vite.config.js`

`package.json` (già creato in pre-flight, conferma il contenuto):
```json
{
  "name": "p4_synesthesia",
  "private": true,
  "type": "module",
  "scripts": {
    "dev":   "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "three": "^0.166.0",
    "tone":  "^15.0.0"
  },
  "devDependencies": {
    "vite": "^5.4.0"
  }
}
```

`vite.config.js`:
```javascript
export default {
  server: { host: '127.0.0.1', port: 5173 },
  optimizeDeps: { include: ['three', 'tone'] },
}
```

### Step 4.2 — `index.html`

> Nota chiave: carichiamo Gibber via `<script>` tag **prima** del nostro modulo, così `window.Gibber` è disponibile.

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>synesthesia</title>
  <style>
    html, body { margin: 0; padding: 0; background: #000; overflow: hidden;
                 font-family: ui-monospace, monospace; color: #fff; }
    canvas { display: block; }
    #hud {
      position: fixed; top: 1em; left: 1em; padding: .6em .9em;
      background: rgba(0,0,0,.55); border: 1px solid #333; font-size: 12px; z-index: 10;
    }
    #start {
      position: fixed; inset: 0; display: flex; align-items: center; justify-content: center;
      background: #000; color: #6cf; font-size: 2em; cursor: pointer; z-index: 100;
      text-align: center; padding: 2em;
    }
  </style>
</head>
<body>
  <div id="start">click to start<br><span style="font-size:.5em;color:#888">(audio + connect to hand bridge)</span></div>
  <div id="hud">
    <div>fps:    <span id="fps">-</span></div>
    <div>hand:   <span id="hand">-</span></div>
    <div>energy: <span id="energy">-</span></div>
    <div>palette:<span id="palette">-</span></div>
  </div>

  <!-- Gibber bundle UMD: deve caricarsi PRIMA dei nostri moduli ES -->
  <script src="/gibber/jsdsp.js"></script>
  <script src="/gibber/bundle.js"></script>

  <script type="module" src="/src/main.js"></script>
</body>
</html>
```

### Step 4.3 — `src/hands.js`

```javascript
// Stato condiviso popolato dal WebSocket bridge.
export const hands = {
  x: 0.5, y: 0.5,
  pinch: false,
  lastPinchAt: 0,
  swipe: null,    // 'left' | 'right' | null (consumato dal main loop)
  connected: false,
}

export function startHands(url = 'ws://127.0.0.1:8765') {
  const ws = new WebSocket(url)
  ws.onopen    = () => { hands.connected = true;  console.log('[hands] connected') }
  ws.onclose   = () => { hands.connected = false; console.log('[hands] disconnected') }
  ws.onerror   = (e) => console.warn('[hands] error', e)
  ws.onmessage = (e) => {
    const m = JSON.parse(e.data)
    if (m.type === 'hand') {
      hands.x = m.x
      hands.y = m.y
    } else if (m.type === 'event') {
      if (m.name === 'pinch')         { hands.pinch = true;  hands.lastPinchAt = performance.now() }
      if (m.name === 'pinch_release') { hands.pinch = false }
      if (m.name === 'swipe')         { hands.swipe = m.dir }
    }
  }
  return ws
}
```

### Step 4.4 — `src/audio.js` (Gibber primary)

> **Approccio difensivo**: Gibber expone l'AudioContext, ma il path esatto del "master output" varia tra build. Sondiamo i path noti e prendiamo il primo che risponde. Diagnostic console.log per debug offline.

```javascript
// Gibber è caricato come global via <script> in index.html.
// window.Gibber è disponibile dopo che bundle.js esegue.

/** Trova un AudioNode collegabile (per inserire un AnalyserNode in parallelo). */
function findGibberOutputNode() {
  const G = window.Gibber
  // Path noti, in ordine di probabilità basato su gibber-cc/gibber e gibber.audio.lib
  const candidates = [
    G?.Audio?.Master?.output,
    G?.Audio?.Master,
    G?.Master?.output,
    G?.Master,
    G?.Audio?.output,
  ]
  for (const c of candidates) {
    if (c && typeof c.connect === 'function') return c
  }
  return null
}

/** Recupera l'AudioContext di Gibber. */
function findGibberAudioContext() {
  const G = window.Gibber
  return G?.Audio?.ctx || G?.Audio?.context || G?.ctx || G?.context || null
}

export const audio = {
  analyser: null,
  fftBuf: null,
  bass: 0, mid: 0, high: 0, energy: 0,
  ready: false,

  async start() {
    if (!window.Gibber) {
      console.error('[audio] window.Gibber non trovato — bundle non caricato?')
      return
    }

    // 1) Esporta tutta l'API Gibber su window per scrivere codice "playground-style"
    window.Gibber.export(window)

    // 2) AudioContext lock-screen: alcuni browser richiedono resume() dopo user gesture
    const ctx = findGibberAudioContext()
    if (ctx && ctx.state === 'suspended') await ctx.resume()
    console.log('[audio] AudioContext state:', ctx?.state)

    // 3) Setup audio: una piccola scena ritmica + lead controllabile
    Clock.bpm = 124

    // Kick + hat — pattern Gibber tipico
    this.kick = Drums('x*x*x*x*').gain(.9)
    this.hat  = Drums('-*-*-*-*').gain(.35).fx.add( Reverb(.3) )

    // Bassline che gira sempre
    this.bass = Synth('bleep').gain(.5).fx.add( Filter24({ cutoff: .35, Q: .8 }) )
    this.bass.note.seq([0, 0, 7, 5, 3, 0, -5, 0], 1/8)

    // Lead controllato dalle mani (parte silenzioso)
    this.lead = FM({ cmRatio: 1.5, index: 3 }).gain(0).fx.add( Reverb(.5), Delay(1/4) )

    // 4) Collega un AnalyserNode al master di Gibber (NON destination — non rompe l'audio)
    const masterNode = findGibberOutputNode()
    if (masterNode && ctx) {
      this.analyser = ctx.createAnalyser()
      this.analyser.fftSize = 256
      this.analyser.smoothingTimeConstant = 0.7
      this.fftBuf = new Uint8Array(this.analyser.frequencyBinCount)
      masterNode.connect(this.analyser)   // tap non destructive
      console.log('[audio] analyser connesso a master')
      this.ready = true
    } else {
      console.warn('[audio] Master node non trovato. Diagnostica: window.Gibber =', window.Gibber)
      console.warn('[audio] Caduta su tap-per-istruzione: collego analyser a ctx.destination via stub')
      // Fallback estremo: analyser su nodo destinato a ctx.destination (silent tap)
      if (ctx) {
        this.analyser = ctx.createAnalyser()
        this.analyser.fftSize = 256
        this.fftBuf = new Uint8Array(this.analyser.frequencyBinCount)
        // Connetti i singoli synth (workaround se master non c'è):
        for (const s of [this.kick, this.hat, this.bass, this.lead]) {
          // Heuristica: ogni instrument Gibber espone .output o .gain o .ugen
          const node = s?.output || s?.gain?.node || s?.ugen?.output
          if (node?.connect) node.connect(this.analyser)
        }
        this.ready = true
      }
    }
  },

  update() {
    if (!this.ready || !this.analyser) return
    this.analyser.getByteFrequencyData(this.fftBuf)
    const N = this.fftBuf.length
    const sum = (a, b) => {
      let s = 0
      for (let i = a; i < b; i++) s += this.fftBuf[i]
      return s / Math.max(1, (b - a)) / 255
    }
    this.bass   = sum(0, Math.floor(N * .1))
    this.mid    = sum(Math.floor(N * .1), Math.floor(N * .4))
    this.high   = sum(Math.floor(N * .4), N)
    this.energy = (this.bass + this.mid + this.high) / 3
  },

  setHandIntensity(x, y) {
    if (!this.lead) return
    // y bassa = forte, y alta = silenzio
    this.lead.gain( Math.max(0, (1 - y)) * 0.55 )
    // x → midi note 48..84 (3 ottave)
    const midi = 48 + Math.floor(x * 36)
    if (this._lastMidi !== midi) {
      this.lead.note(midi)
      this._lastMidi = midi
    }
  },

  bang() {
    // Pinch: una nota "stab" alta
    if (!this.lead) return
    this.lead.note(72 + Math.floor(Math.random() * 12))
    this.lead.gain(0.7)
  },
}
```

### Step 4.4-bis — `src/audio_tonejs.js` (Plan B — pronto in 1 swap se Gibber dà problemi)

> **Quando usare il Plan B**: se in `console` vedi `[audio] Master node non trovato` E i log mostrano `window.Gibber` malformato (es. niente `.Audio.Master`), oppure se l'analyser non riceve dati nonostante l'audio si senta. In quel caso, rimpiazza in `main.js` l'import di `./audio.js` con `./audio_tonejs.js` e in `index.html` puoi anche RIMUOVERE i due script tag di Gibber (non servono più).

```javascript
import * as Tone from 'tone'

export const audio = {
  fft: null,
  bass: 0, mid: 0, high: 0, energy: 0,
  ready: false,

  async start() {
    await Tone.start()

    const kick = new Tone.MembraneSynth().toDestination()
    const hat  = new Tone.MetalSynth({
      envelope: { attack: 0.001, decay: 0.05, release: 0.01 },
      harmonicity: 5.1, modulationIndex: 32, resonance: 4000, octaves: 1.5,
    }).toDestination()
    hat.volume.value = -18

    new Tone.Loop(t => kick.triggerAttackRelease('C1', '8n', t), '4n').start(0)
    new Tone.Loop(t => hat.triggerAttackRelease('32n', t),       '8n').start('8n')

    this.lead = new Tone.FMSynth({
      harmonicity: 2,
      modulationIndex: 4,
      envelope: { attack: 0.02, decay: 0.3, sustain: 0.5, release: 0.8 },
    }).toDestination()
    this.lead.volume.value = -28

    const bass = new Tone.MonoSynth({
      oscillator: { type: 'sawtooth' },
      filter: { type: 'lowpass', rolloff: -24, Q: 4 },
      envelope: { attack: 0.01, decay: 0.2, sustain: 0.4, release: 0.2 },
      filterEnvelope: { attack: 0.01, decay: 0.2, sustain: 0.2, release: 0.3,
                        baseFrequency: 200, octaves: 3 },
    }).toDestination()
    bass.volume.value = -14
    const bassNotes = ['C2', 'C2', 'G2', 'D#2', 'F2', 'D#2', 'G2', 'C2']
    new Tone.Sequence((t, n) => bass.triggerAttackRelease(n, '16n', t), bassNotes, '8n').start(0)

    Tone.Transport.bpm.value = 124
    Tone.Transport.start()

    this.fft = new Tone.FFT(64)
    Tone.getDestination().connect(this.fft)
    this.ready = true
  },

  update() {
    if (!this.fft) return
    const data = this.fft.getValue()
    const norm = db => Math.max(0, Math.min(1, (db + 100) / 100))
    let b = 0, m = 0, h = 0
    for (let i = 0; i < 8;  i++) b += norm(data[i])
    for (let i = 8; i < 32; i++) m += norm(data[i])
    for (let i = 32; i < data.length; i++) h += norm(data[i])
    this.bass = b / 8
    this.mid  = m / 24
    this.high = h / (data.length - 32)
    this.energy = (this.bass + this.mid + this.high) / 3
  },

  setHandIntensity(x, y) {
    if (!this.lead) return
    this.lead.volume.rampTo(-30 + y * 22, 0.1)
    const midi = 60 + Math.floor((1 - y) * 24)
    if (this._lastMidi !== midi) {
      this.lead.triggerAttackRelease(Tone.Frequency(midi, 'midi'), '8n')
      this._lastMidi = midi
    }
  },

  bang() {
    if (!this.lead) return
    this.lead.triggerAttackRelease('C5', '4n')
  },
}
```

**Come switchare al Plan B (Tone.js) se Gibber dà problemi:**
```diff
// src/main.js, riga di import:
- import { audio } from './audio.js'
+ import { audio } from './audio_tonejs.js'
```
E (opzionale) rimuovi da `index.html` i due `<script src="/gibber/...">` perché non più necessari.

### Step 4.5 — `src/shaders/particle.vert.glsl`

```glsl
uniform float uTime;
uniform float uEnergy;
uniform float uBass;
uniform float uNoiseScale;
attribute float aSeed;
varying float vSeed;
varying float vEnergy;

void main() {
  vSeed = aSeed;
  vEnergy = uEnergy;

  vec3 p = position;
  float t = uTime * 0.4 + aSeed * 10.0;
  vec3 disp = vec3(
    sin(t + p.y * uNoiseScale),
    cos(t * 1.2 + p.z * uNoiseScale),
    sin(t * 0.7 + p.x * uNoiseScale)
  );
  p += disp * (0.4 + uBass * 2.5);

  vec4 mv = modelViewMatrix * vec4(p, 1.0);
  gl_Position = projectionMatrix * mv;
  gl_PointSize = (1.5 + uEnergy * 8.0) * (300.0 / -mv.z);
}
```

### Step 4.6 — `src/shaders/particle.frag.glsl`

```glsl
uniform vec3 uPalette;
varying float vSeed;
varying float vEnergy;
void main() {
  vec2 c = gl_PointCoord - 0.5;
  float d = length(c);
  if (d > 0.5) discard;
  float a = smoothstep(0.5, 0.0, d);
  vec3 col = uPalette * (0.55 + vEnergy * 0.9 + vSeed * 0.3);
  gl_FragColor = vec4(col, a * 0.9);
}
```

### Step 4.7 — `src/particles.js`

```javascript
import * as THREE from 'three'
import vert from './shaders/particle.vert.glsl?raw'
import frag from './shaders/particle.frag.glsl?raw'

export function createParticles(count = 40000) {
  const geo = new THREE.BufferGeometry()
  const pos  = new Float32Array(count * 3)
  const seed = new Float32Array(count)

  for (let i = 0; i < count; i++) {
    // Distribuzione sferica uniforme
    const r  = Math.cbrt(Math.random()) * 8
    const th = Math.random() * Math.PI * 2
    const ph = Math.acos(2 * Math.random() - 1)
    pos[i*3+0] = r * Math.sin(ph) * Math.cos(th)
    pos[i*3+1] = r * Math.sin(ph) * Math.sin(th)
    pos[i*3+2] = r * Math.cos(ph)
    seed[i] = Math.random()
  }

  geo.setAttribute('position', new THREE.BufferAttribute(pos, 3))
  geo.setAttribute('aSeed',    new THREE.BufferAttribute(seed, 1))

  const mat = new THREE.ShaderMaterial({
    vertexShader:   vert,
    fragmentShader: frag,
    uniforms: {
      uTime:       { value: 0 },
      uEnergy:     { value: 0 },
      uBass:       { value: 0 },
      uNoiseScale: { value: 0.3 },
      uPalette:    { value: new THREE.Color('#6cf') },
    },
    transparent: true,
    depthWrite:  false,
    blending:    THREE.AdditiveBlending,
  })

  return new THREE.Points(geo, mat)
}
```

### Step 4.8 — `src/main.js`

```javascript
import * as THREE from 'three'
import { audio } from './audio.js'
import { hands, startHands } from './hands.js'
import { createParticles } from './particles.js'

// --- Scene ---
const scene = new THREE.Scene()
scene.fog = new THREE.FogExp2(0x000000, 0.04)

const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 100)
camera.position.set(0, 0, 18)

const renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: 'high-performance' })
renderer.setPixelRatio(Math.min(devicePixelRatio, 2))
renderer.setSize(innerWidth, innerHeight)
document.body.appendChild(renderer.domElement)

const particles = createParticles(40000)
scene.add(particles)

// --- Palette ---
const PALETTES = ['#6cf', '#f6c', '#cf6', '#fc6', '#9f6', '#c6f']
let paletteIdx = 0
const setPalette = (i) => {
  paletteIdx = ((i % PALETTES.length) + PALETTES.length) % PALETTES.length
  particles.material.uniforms.uPalette.value.set(PALETTES[paletteIdx])
  document.getElementById('palette').textContent = PALETTES[paletteIdx]
}
setPalette(0)

// --- HUD ---
const hud = {
  fps: document.getElementById('fps'),
  hand: document.getElementById('hand'),
  energy: document.getElementById('energy'),
}
let lastT = performance.now(), frames = 0, fpsAcc = 0

// --- Loop ---
function frame() {
  const t = performance.now()
  const dt = (t - lastT) / 1000
  lastT = t

  audio.update()
  audio.setHandIntensity(hands.x, hands.y)

  // Camera orbit con mano
  const azim = (hands.x - 0.5) * Math.PI
  const elev = (hands.y - 0.5) * Math.PI * 0.6
  const R = 18
  camera.position.x = R * Math.cos(elev) * Math.sin(azim)
  camera.position.y = R * Math.sin(elev)
  camera.position.z = R * Math.cos(elev) * Math.cos(azim)
  camera.lookAt(0, 0, 0)

  const u = particles.material.uniforms
  u.uTime.value      += dt
  u.uEnergy.value     = u.uEnergy.value * 0.85 + audio.energy * 0.15
  u.uBass.value       = u.uBass.value   * 0.80 + audio.bass   * 0.20
  u.uNoiseScale.value = 0.15 + hands.y * 0.8

  // Pinch = boost transient
  if (performance.now() - hands.lastPinchAt < 250) {
    u.uEnergy.value = 1.0
  }

  // Consuma swipe
  if (hands.swipe === 'right') { setPalette(paletteIdx + 1); hands.swipe = null }
  if (hands.swipe === 'left')  { setPalette(paletteIdx - 1); hands.swipe = null }

  renderer.render(scene, camera)

  fpsAcc += dt; frames++
  if (fpsAcc > 1) {
    hud.fps.textContent    = (frames / fpsAcc).toFixed(0)
    hud.hand.textContent   = `${hands.x.toFixed(2)},${hands.y.toFixed(2)}${hands.connected ? '' : ' (off)'}`
    hud.energy.textContent = audio.energy.toFixed(2)
    fpsAcc = 0; frames = 0
  }
  requestAnimationFrame(frame)
}

addEventListener('resize', () => {
  camera.aspect = innerWidth / innerHeight
  camera.updateProjectionMatrix()
  renderer.setSize(innerWidth, innerHeight)
})

document.getElementById('start').addEventListener('click', async () => {
  document.getElementById('start').remove()
  await audio.start()
  startHands()
  frame()
})
```

> **Nota Vite + GLSL come stringa**: `import vert from './x.glsl?raw'` funziona in Vite out-of-the-box senza plugin (è una feature built-in: `?raw` carica come stringa).

### Step 4.9 — README del flagship

```bash
cd ~/Desktop/planeProjects/p4_synesthesia
cat > README.md <<'README_EOF'
# synesthesia

Audio-reactive 3D particle visualizer controlled by your hands.
Built with MediaPipe (hand tracking) + Gibber (live-coded audio) + Three.js (3D).

## Demo
![demo](docs/demo.gif)

## Quickstart
```bash
# 1. Hand-tracking bridge (Python)
python3.11 -m venv .venv && source .venv/bin/activate
pip install mediapipe opencv-python numpy websockets
python server/bridge.py

# 2. Web app (separate terminal)
npm install
npm run dev
# → open http://127.0.0.1:5173, click to start
```

## Controls
- **Move your hand**: orbit the camera + change the lead synth pitch
- **Pinch (index + thumb)**: trigger a note + energy burst
- **Swipe left/right**: change the color palette

## Stack
Python 3.11 · MediaPipe Tasks · WebSockets · Gibber (audio) · Three.js · GLSL shaders

## Architecture
```
[bridge.py: HandTracker]
       │ ws://127.0.0.1:8765
       ▼
[Vite: index.html]
   ├─ <script> gibber bundle (UMD global)
   └─ <script type=module> src/main.js
       ├─ src/audio.js     → Gibber synth + AnalyserNode
       ├─ src/hands.js     → WS client
       └─ src/particles.js → Three.js shader
```

## Part of the planeProjects portfolio
[handmouse](https://github.com/CarloFanelli/handMouse) · [gibber-sketches](https://github.com/CarloFanelli/gibber-sketches) · [handgibber](https://github.com/CarloFanelli/handgibber) · **synesthesia** (you are here)

Built across 4 flights (~30h offline coding) — see commit history for the timeline.
README_EOF
```

### Come provarlo (in volo)

```bash
# Terminal A
cd ~/Desktop/planeProjects/p4_synesthesia
source ../.venv/bin/activate    # riusa venv del workspace
python server/bridge.py

# Terminal B
cd ~/Desktop/planeProjects/p4_synesthesia
npm run dev        # http://127.0.0.1:5173
```

### Polish (l'effetto wow vero)
- **Bloom post-process**: aggiungi `import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js'` + `UnrealBloomPass`. Tutto già nel repo `_docs/threejs-source/examples/jsm/postprocessing/`.
- **Beat detection più aggressivo**: salva un baseline di `audio.bass` e fai pulse di `particles.scale` quando `bass > baseline * 1.5`.
- **Geometry alternativa**: prova `THREE.IcosahedronGeometry(5, 4)` con `wireframe: true` deformata in vertex shader: meno particelle ma più "Tron".
- **Recording**: in `main.js` aggiungi `MediaRecorder` per registrare il canvas come MP4 (utile per la GIF demo).

### Commit
```bash
cd ~/Desktop/planeProjects/p4_synesthesia
git add . && git commit -m "feat(p4): synesthesia flagship (Three.js + Tone.js + hand bridge)"
```

---

## Verifica end-to-end (checklist)

**P1:**
- [ ] `python -m demos.mouse` muove il cursore senza jitter percepibile
- [ ] Pinch indice↔pollice → mouse fermo; pinch medio↔pollice → click
- [ ] `presenter.py` cambia slide in Keynote/PowerPoint reali
- [ ] `air_keyboard.py` riesce a scrivere "ciao" (5+ caratteri consecutivi)

**P2:**
- [ ] Tutti gli sketch suonano nel playground Gibber locale
- [ ] Il runner HTML li mostra e Copy funziona

**P3:**
- [ ] `bridge.py` mostra "client connected" quando apri la sketch in Gibber
- [ ] Theremin segue smoothly la mano in X
- [ ] Drumpad fa partire 3 suoni distinti per le 3 zone
- [ ] Acid lab cambia pattern con lo swipe

**P4:**
- [ ] 30+ fps a 40k particelle su M1 Pro
- [ ] Audio Gibber parte dopo il click iniziale (in console: "AudioContext state: running" + "analyser connesso a master")
- [ ] HUD mostra hand X/Y, FPS, energy > 0 quando l'audio suona (se energy resta 0, l'analyser non riceve dati → switcha a Plan B Tone.js)
- [ ] Pinch fa burst di energia (vedi le particelle esplodere)
- [ ] Swipe cambia palette
- [ ] Camera orbita coerentemente con la mano

---

## Post-volo (a terra, con internet)

### Push delle 4 repo (le remote sono già configurate pre-volo)

```bash
# P1 — repo riusata + tag v2-modular
cd ~/Desktop/planeProjects/p1_handmouse
git push origin main
git push origin v1-monolithic v2-modular

# P2, P3, P4 — scaffold già pushato pre-volo, ora pushiamo tutto il lavoro in volo
for d in p2_gibber p3_handgibber p4_synesthesia; do
  cd ~/Desktop/planeProjects/$d
  git push origin main
done
```

### README "flagship" da curare con attenzione

Per **p4_synesthesia** (è il pezzo principale del portfolio):
- GIF demo in alto (registrata in volo con QuickTime, convertita con `ffmpeg -i input.mov -vf "fps=20,scale=720:-1" -loop 0 demo.gif` quando torni a terra)
- "What you'll see in 30 seconds" — descrizione user-facing
- Sezione "How it works" con diagramma ASCII dell'architettura
- "Try it yourself" con i 2 comandi (bridge + npm run dev)
- "Built during a 30-hour flight stretch — see also [p1](#) [p2](#) [p3](#)"

### Repo "umbrella" opzionale (consigliato per il colloquio)

```bash
cd ~/Desktop/planeProjects
mkdir creative-coding-portfolio && cd creative-coding-portfolio
git init
# README con 4 thumbnail (le GIF da P1..P4) + breve descrizione + link
# Si compila come una mini "landing" del tuo portfolio creativo
gh repo create creative-coding-portfolio --public --source=. --push
```

---

## Mapping vecchio handMouse → nuovo modulo

Cosa del vecchio `hand_recognition.py` migra dove:

| Vecchio | Nuovo |
|---|---|
| classe `HandPoint` (3 attr) | `geometry.HandPoint` (dataclass) |
| `checkPinch()` | `gestures.is_pinching()` |
| `click()` | `gestures.is_clicking()` |
| `update_hand_points()` | `HandTracker._extract()` |
| `process_frame()` | inline in `HandTracker.run()` |
| main loop monolitico | `HandTracker.run()` + event bus |
| `drawSquare()` HUD debug | opzionale: `visualizer.py` (polish, salta) |
| variabili globali `indexFingerTip` etc. | locali in `_extract()` |

Il file originale resta in `p1_handmouse/legacy/hand_recognition.py` come riferimento (visibile sia in `git log` della repo che nel filesystem).

---

## File critici (riepilogo)

| Path | Cosa fa |
|------|---------|
| `~/Desktop/planeProjects/_assets/gesture_recognizer.task` | Modello MediaPipe, condiviso |
| `~/Desktop/planeProjects/.venv/` | Python venv condiviso (mediapipe, opencv, etc.) |
| `~/Desktop/planeProjects/_vendor/gibber/` | Gibber clonato + `npm install` (offline) |
| `~/Desktop/planeProjects/_vendor/wheels/` | pip wheels arm64 cached (safety net) |
| `~/Desktop/planeProjects/_vendor/npm-cache/` | tarball `three`, `tone`, `vite` cached |
| `~/Desktop/planeProjects/_docs/threejs-source/` | Three.js source + docs + examples offline |
| `~/Desktop/planeProjects/_docs/tonejs-source/` | Tone.js source + docs offline |
| `~/Desktop/planeProjects/p1_handmouse/handmouse/tracker.py` | Cuore HandTracker (callback-based) |
| `~/Desktop/planeProjects/p3_handgibber/server/bridge.py` | WebSocket bridge condiviso |
| `~/Desktop/planeProjects/p4_synesthesia/src/main.js` | Render loop Three.js |
| `~/Desktop/planeProjects/p4_synesthesia/src/audio.js` | Gibber synth + AnalyserNode (primario) |
| `~/Desktop/planeProjects/p4_synesthesia/src/audio_tonejs.js` | Plan B fallback Tone.js (swap-in 1 riga) |
| `~/Desktop/planeProjects/p4_synesthesia/public/gibber/bundle.js` | Gibber UMD bundle copiato da _vendor |
| `~/Desktop/planeProjects/p4_synesthesia/src/particles.js` | Particle system + shader |
| `~/Desktop/planeProjects/PROJECTS.md` | Copia di questo piano |

---

## Decisioni di design importanti (rationale)

1. **Gibber ovunque (P2/P3/P4) per l'aesthetic live-coding, con Tone.js come Plan B in P4**: Gibber è l'aesthetic giusta per portfolio creative-coding; il rischio "API meno documentata + bundle UMD" lo mitighiamo in 3 modi: (a) script tag in `index.html` invece di import ESM, (b) probing difensivo del master node + AudioContext con fallback su tap per-istruzione, (c) Tone.js pre-installato e `src/audio_tonejs.js` pronto per swap in 1 riga se Gibber dovesse non collaborare. Costo del Plan B: ~2 minuti per applicare lo swap.

2. **Bridge WebSocket, non OSC**: il browser parla WebSocket nativamente, OSC richiederebbe un proxy aggiuntivo. JSON keep-it-simple.

3. **Thread separato per cv2 + asyncio loop**: il loop OpenCV è bloccante; lo metto in thread daemon e schedulo le `broadcast()` sul loop asyncio con `run_coroutine_threadsafe`. Pattern standard, robusto.

4. **OneEuroFilter su mouse, non su tutto**: filtrare i landmark grezzi smussa anche eventi rapidi tipo pinch. Filtro solo la coordinata mouse, lascio gli eventi raw.

5. **4 repo + 1 umbrella, P4 self-contained**: massimizza valore portfolio. Un recruiter che clicca su `synesthesia` può clonare e provare in 2 comandi senza setup di submodule.

6. **Vendor invece di submodule per il flagship**: la copia di `handmouse/` in `p4_synesthesia/vendor/` aggiunge ~200 righe ma toglie ogni attrito di onboarding per chi clona.
