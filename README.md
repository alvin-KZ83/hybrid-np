# proto-v4 — Notebook App

A live-captioning prototype for lectures. You capture moments from the captions by pressing
**Space**, and the captures become a **notebook** you can annotate, link, cluster, refine with AI,
save to a file, and reopen later. Every interaction is logged for analysis.

Everything lives in one self-contained file, **`notebook-app.html`**, with inline CSS and JS and
no build step.

---

## 1. Running it

1. **Serve the folder** instead of opening the file directly. Browsers block `fetch()` from
   `file://`, and the AI features need to fetch `ai-key.json`. Either of these works:
   - VS Code **Live Server**, or
   - `python -m http.server`, then open `http://localhost:8000/notebook-app.html`.
2. Use **Chrome or Edge**. The Web Speech API and the Translator API are Chromium-only.
3. Allow microphone access when prompted.

The published copy runs on GitHub Pages, with `ai-key.json` deployed alongside it.

### Files

| File | Committed | Purpose |
|---|---|---|
| `notebook-app.html` | yes | The whole app. |
| `ai-key.json` | yes (deployed) | Encoded Claude API key for "Refine with AI". Treat it as public (see §5). |
| `local-config.json` | **no** (gitignored) | Developer-only settings, read only on `localhost`. See §3. |
| `key.txt` | **no** (gitignored) | Raw API key, used only to generate `ai-key.json`. |

---

## 2. Features

### 2.1 Capture screen

- **Live captions.** Speech recognition runs in **English** or **Japanese**; choose with the
  *Speech* switch, which locks while recognition is running.
- **Live translation (optional).** English → Japanese or Japanese → English, using Chrome's
  on-device Translator API. A small built-in dictionary covers it if the API is unavailable. The
  first use may download a language pack, shown with a progress bar.
- **Two caption boxes**: original and translation, fixed at the top of the screen. Click a box to
  make it the **primary** pane (larger). The primary pane decides which text a capture records.
- **Space to capture.** Pressing Space while recognition is running records a capture:
  - **Raw:** what was on screen at the moment of the press.
  - **Delay-adjusted:** what was on screen **500 ms** later (`CAPTURE_DELAY_MS`), to allow for
    recognition lag.
  - The **previous and current sentence** at the moment of capture, from the full transcript.
  - Whether the raw and delayed text contain the session's **target phrase** ("blue triangle" /
    "青い三角形").
- **Space handling.** Space only captures while the Capture screen is showing and the cursor
  isn't in a text field. It never activates a focused button or checkbox and never scrolls the
  page. Holding it down counts once. A status line under the button confirms each capture
  ("Capture registered").
- **Fits the window.** During a session the page is locked to the window height and the caption
  boxes shrink to fit.

### 2.2 Results

Stopping recognition shows a **results** panel listing each capture's raw and delay-adjusted text,
target-phrase detection, and sentence context. You can export it with **Copy as JSON** (falls back
to a download if the clipboard is blocked) or **Download CSV** (UTF-8 with a BOM, so Excel shows
Japanese correctly). Results cover the current session only. The notebook file (§2.5) keeps every
capture permanently.

### 2.3 Notebook screen

Each capture becomes a note titled *Capture N*. Until the first real capture, a set of sample notes
from a Microeconomics 101 lecture is shown; the first capture replaces them. New recording sessions
**add to** the open notebook, so numbering and the timeline continue from the last note.

The page is locked to the window height. The main panel and the details panel scroll
independently.

**Six views** (tabs):

| View | Shows |
|---|---|
| All notes | Every note as a chip, coloured by cluster. |
| Chronological | Notes on a timeline. |
| Density | 30-second bins shaded by how many notes fall in each; click a bin to list its notes. |
| Clusters | Notes grouped into clusters (see §2.4), with each cluster's name. |
| Connections | The selected note at the centre with up to 6 connected notes around it (see below). |
| Breakdown | The selected note's text, the earlier notes it **builds on**, and its **key terms** with definitions. |

**Search** filters notes by any text on them: title, captured text, or annotations.

**Details panel** (selected note):

- Editable **title** and **captured text**, with *Revert to captured text*.
- **Sentence context.** The previous and current sentence at the moment of capture (read-only).
- **Annotations.** Free-form fields with presets (*Keyword, Idea, Example, Question,
  Definition*), any custom label, and up to 3 levels of nested sub-items. A field labelled
  *Keyword(s)* or *Tag(s)* holds comma-separated keywords. Annotations added by the AI carry an
  **AI** badge until you edit them; once edited they're yours and are kept on the next refine.
- **Related notes**, each with the reason it's connected.

**Connections view legend:**

- **Solid circle:** the selected note, filled with its cluster's colour.
- **Ringed circle:** a connected note, in its own cluster's colour. A different colour means a
  link into a different cluster.
- **Line thickness:** how strongly the two notes are related.

### 2.4 How connections and clusters are made

**Before any AI refine:** connections and clusters come from word matching.

- Each note gets a set of words taken from its title, captured text and annotations: lowercase,
  at least 4 letters, common words removed, plurals roughly stripped.
- Keywords from *Keyword* or *Tag* annotations are added as whole phrases.
- Link strength is the number of words two notes share, with each shared keyword counting double.
- Notes with a link strength of 2 or more are chained into clusters.
- Limitation: only English letters are kept, so Japanese notes link only through keywords.

**After "Refine with AI":**

- **Clusters are topics.** The AI assigns each note one topic, reusing topic names across notes.
  Notes with the same topic form a cluster named after it.
- **Links are judged by the AI.** For each note it returns related notes from the whole notebook,
  each with a strength (1–3) and a short **reason**. The reason replaces the list of shared words
  wherever the link is shown.
- **Mixed notebooks.** A pair of notes uses the AI's judgement only when **both** have been
  refined. Notes captured since the last refine fall back to word matching, and have no topic,
  until the next refine.

### 2.5 Refine with AI

One click sends every note to **Claude Haiku 4.5** (`claude-haiku-4-5`) in batches of 20, using a
structured JSON response. Each request includes:

- each note's title, captured text, **previous and current sentence**, and your own annotations;
- a short **index of the whole notebook**, including every note's sentence context, so the AI can
  link across batches;
- the keywords and topics already in use, so it reuses the same wording.

For each note the AI returns:

| Returned | Becomes |
|---|---|
| Suggested title | The note's title, but only while it's still the default *Capture N*. |
| Keywords | A *Keyword* annotation (AI). |
| 1–3 ideas | *Idea* annotations (AI). |
| Explanation | An *Explanation* annotation (AI). |
| Term definitions | Definitions of every keyword as used in that note, shown in Breakdown → Key terms. |
| Topic | The note's cluster. |
| Links | AI-judged connections, each with a strength and reason. |

**Limits.** Refine is capped at **5 runs per browser**. The count is kept in that browser's
storage. Only a request that succeeds uses up a run. Setting `unlimitedRefines` in
`local-config.json` lifts the cap on your own machine (§3).

**Cost** (Haiku 4.5: $1 input / $5 output per million tokens). These are estimates; the browser
console prints the exact tokens and cost of each request.

| Notebook size | Approx. cost per refine |
|---|---|
| 10 notes | ~$0.03 |
| 20 notes | ~$0.055 |
| 50 notes | ~$0.15 |
| 100 notes | ~$0.33 |

Every refine reprocesses the whole notebook. Output tokens (about 450 per note) make up most of
the cost.

### 2.6 Saving and reopening a notebook

- **⬇ Download notebook** saves everything to `notebook-<timestamp>.json`.
- **⬆ Open notebook** loads such a file back at any time, with no capture session needed. It
  checks the file and fills in missing fields first, and asks before replacing unsaved notes. It's
  unavailable while recognition is running.
- **Unsaved warning.** The browser warns before closing or reloading the page if the notes have
  changed since the last download or open. This tracks note edits only. Interactions that change
  no notes don't trigger it, so they're lost unless you download.

**File format** (`format: "proto-v4-notebook"`, `version: 2`; version 1 files still open):

| Field | Contents |
|---|---|
| `savedAt` | When the file was downloaded. |
| `notes[]` | `id`, `concept` (title), `phrase` and `originalPhrase`, `annotations` (tree), `previousSentence` and `currentSentence`, `elapsedS` and `timeLabel`, `capturedAt`, `pane`, `targetPhrase`, `topic`, `aiLinks`, `termDefinitions`, and **`capture`**: the full capture record (raw vs. delay-adjusted text and sentences, target detection, `snapshotErrorMs`, `offsetMs`, `trial`, `speechLanguage`, `session`). |
| `interactionSummary` | Total events, a count per event type, and each session's first event, last event and event count. |
| `interactionLog[]` | Every logged interaction (§2.7). |

### 2.7 Interaction logging

Every meaningful interaction is logged with `t` (ISO time), `ms` (since page load), `session`
(one id per page load), `screen`, `type`, and details. The log belongs to the notebook: it's saved
in the downloaded file and restored and appended to when the file is reopened. A notebook used over
several days therefore holds its whole history; group by `session` and order by `t`.

| Area | Event types |
|---|---|
| App | `app_loaded` (browser, viewport, API support, host), `page_hidden` / `page_visible`, `screen_changed` |
| Recognition | `recognition_start_requested`, `recognition_started` (`restart` marks the browser's automatic restarts), `recognition_ended`, `recognition_error`, `recognition_stop_requested` |
| Capture settings | `speech_language_changed`, `translation_toggled`, `primary_pane_changed` |
| Capturing | `capture_pressed`, `capture_ignored` (Space pressed while recognition wasn't running), `capture_completed` (resulting note, target detection, timing error) |
| Results | `results_copied_json`, `results_downloaded_csv` |
| Navigation | `notebook_view_changed`, `note_selected` (`via`: chip, timeline dot, connections node, related row), `notebook_searched` (logged once typing pauses), `density_bin_toggled` |
| Editing | `note_title_edited`, `captured_text_edited`, `captured_text_reverted`, `annotation_added`, `annotation_label_edited`, `annotation_text_edited`, `annotation_removed`, `ai_annotation_claimed` |
| AI | `ai_refine_started`, `ai_refine_request` (tokens and cost per request), `ai_refine_completed` (links and clusters before and after), `ai_refine_failed`, `ai_refine_blocked` |
| File | `notebook_downloaded`, `notebook_opened`, `notebook_open_failed` |

The log records **lengths** of edited captured text and annotation text, not the text itself,
which is already in `notes`. Titles, annotation labels and search queries are recorded verbatim.
Hovering and scrolling are not logged.

---

## 3. Configuration

### API key (`ai-key.json`)

```json
{ "version": 1, "key": "<encoded>" }
```

`<encoded>` is the key XOR'd with a secret in the page, then base64-encoded. To generate it, serve
the page, open DevTools, run `nbEncodeApiKey("sk-ant-...")`, and save the printed JSON as
`ai-key.json`.

### Local developer config (`local-config.json`)

```json
{ "unlimitedRefines": true }
```

- `unlimitedRefines`: removes the 5-refines-per-browser cap. Local runs also don't use up the
  browser's normal allowance.
- The file is gitignored, so it never reaches GitHub Pages.
- The page only requests it when served from `localhost`, `127.0.0.1` or `[::1]`.
- When it's active, the notebook's status line reads *unlimited refines (local-config.json)*.

---

## 4. Libraries and APIs

No external JS libraries or frameworks, only browser APIs:

- **Web Speech API** (`SpeechRecognition`) for live transcription.
- **Translator API** (Chrome/Edge, on-device, experimental) for live translation, with a built-in
  word dictionary as a fallback.
- **Claude API** (Messages API with structured outputs), called directly from the browser for
  "Refine with AI".

---

## 5. Pros and cons

**Pros**

- Zero dependencies and no build step; one file to run and inspect.
- Calibration data backs the capture delay (`../latency calibrator/analysis-report.md`).
- Notebooks are portable JSON files that carry their full capture records and interaction history,
  ready for analysis.
- AI suggestions stay separate from the user's own work (AI badge) and are only kept once adopted.

**Cons**

- Chromium-only (Web Speech API, Translator API).
- The capture delay is **500 ms**, below the calibration report's recommended **1500 ms** (the
  smallest delay with 100% target detection in a small sample: 5 participants, 35 observations).
  This trades measured accuracy for a snappier feel.
- **The API key is effectively public.** `ai-key.json` is deployed with the page and decoded in
  the browser, and the per-browser refine cap can be reset by clearing site data. Spending must be
  capped on Anthropic's side: prepaid credits with auto-reload off, and the key in its own
  workspace with a spend limit.
- Before the first AI refine, connections rely on English word matching (see §2.4).
- Notebooks live only in the page until downloaded. Closing without downloading loses unsaved
  work, though the page warns about unsaved note edits.

---

## 6. Where it builds from the previous version

Related files sit in the parent `proto-v4/` folder:

- `latency calibrator/`: the calibration study that measured caption lag and informed the capture
  delay (`analysis-report.md`, `calibration-result.csv`).
- `unlimited.html`: the earlier baseline, the capture and delay mechanic with unlimited
  captures, without translation or the notebook. Its title says 1000 ms, but its code uses
  500 ms.
- `sample.html`: "Anatomy of a Capture", a static explainer page, not an entry point.

`notebook-app.html` keeps the capture and delay mechanic and adds live translation, the notebook,
AI refinement, saving and reopening, and interaction logging on top.
