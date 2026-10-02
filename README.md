# proto-v4 — Speech Capture Pilot

## 1. Entry point

Three runnable entry points, all self-contained HTML files (inline CSS/JS, no build step):

- **`notebook-app.html`** — the most complete and current build. Start here.
- **`index.html`** / **`unlimited.html`** — simpler baselines that isolate the core
  capture/delay mechanic without the translation or notebook layers.
- **`capture-sample.html`** — a static, non-interactive doc page ("Anatomy of a Capture"),
  not an entry point.

Open a file directly in the browser, or serve the folder (e.g. `python -m http.server`) and
navigate to it. Chrome or Edge only — Web Speech API and the Translator API are Chromium-only.
Grant microphone permission when prompted.

## 2. Main features and functions

- Live speech-to-text captioning with a **Capture** button that snapshots the on-screen
  caption immediately ("raw") and again after a fixed post-press delay ("delay-adjusted"),
  then checks both against a target phrase.
- `index.html` runs a capped set of delay conditions per session; `unlimited.html` fixes the
  delay at 1000 ms with unlimited captures; `notebook-app.html` fixes it at 500 ms.
- `notebook-app.html` also adds a live English → Japanese **translation pane** with a
  per-capture label field, and a second **Notebook** screen that turns captures into notes,
  visualized six ways: chip grid, chronological timeline, time-density heatmap, keyword
  clusters (union-find), an SVG ego-graph of related notes, and a breakdown view with an
  optional "Ask AI to define" lookup.

## 3. Libraries or packages used

None — no external JS libraries or frameworks, only native browser APIs:

- **Web Speech API** (`SpeechRecognition`) for live transcription.
- **Translator API** (Chrome/Edge on-device, experimental) for the translation pane, with a
  small hand-written word dictionary as a synchronous fallback.
- **Claude API** — called directly from the browser in `notebook-app.html`'s "Ask AI to
  define" feature, using a user-supplied API key stored only in `localStorage`.

## 4. Potential pros and cons

**Pros**
- Zero dependencies, zero build step — easy to run and inspect.
- Real calibration data (`analysis-report.md`, `calibration-result.csv`) backs the delay
  choice instead of guessing.
- Graceful degradation: translation and AI-definition features fail safe to local fallbacks.

**Cons**
- Chromium-only (Web Speech API and Translator API aren't supported elsewhere).
- The calibration report recommends a **1500 ms** delay for 100% detection of the target
  phrase, but the shipped prototypes use less: 1000 ms in `unlimited.html` and 500 ms in
  `notebook-app.html` — both trade measured accuracy for a snappier feel.
- Small calibration sample (5 participants, 35 observations).
- "Ask AI to define" needs a user-supplied API key with no server-side proxy.

## 5. Where it builds from the previous version

proto-v5 is a **standalone research branch**, not a direct continuation of proto-v1/v2/v3's
shared transcription-tool codebase (`core.js`/`main.js`/`nlp.js`). It shares no code with
those folders — it's an independent line of work built around the capture-delay calibration
study. The internal lineage within proto-v5 is:

`index.html` → `unlimited.html` → `notebook-app.html`

Each is a superset of the previous one's capture/delay logic: `unlimited.html` uncaps the
capture count and fixes the delay; `notebook-app.html` keeps that mechanic and adds the
translation pane and Notebook screen on top.
