# Deepa — Bilingual Wake Word Listener

> **Author:** Santa Basnet  
> **Company:** Wiseyak  
> **Version:** v2  

A browser-native, bilingual wake word detection engine that listens for **"Namaste Deepa"** (Nepali) and **"Hey Deepa"** (English) entirely in the browser — no backend, no machine learning model, no external APIs.

---

## Table of Contents

- [Overview](#overview)
- [Wake Phrases](#wake-phrases)
- [Usage](#usage)
- [How It Works](#how-it-works)
- [Implementation Summary](#implementation-summary)
  - [Fuzzy Matching Engine](#fuzzy-matching-engine)
  - [Scoring Pipeline](#scoring-pipeline)
  - [Session-Persistent Microphone](#session-persistent-microphone)
  - [Silence Watchdog](#silence-watchdog)
  - [LRU Detection Log](#lru-detection-log)
  - [Audio Waveform & Confidence Meters](#audio-waveform--confidence-meters)
- [Configuration Constants](#configuration-constants)
- [Browser Compatibility](#browser-compatibility)

---

## Overview

Deepa is a single-file HTML application (`deepa-wake-word.html`) that uses the browser's built-in **Web Speech API** (`SpeechRecognition`) for continuous speech-to-text, combined with a custom **fuzzy keyword matching** engine to detect wake words across accent and phonetic variations.

---

## Wake Phrases

| Language | Phrase          | Phonetic        |
|----------|-----------------|-----------------|
| Nepali   | Namaste Deepa   | na-mas-te dee-pah |
| English  | Hey Deepa       | hay dee-pah      |

The engine also recognises common phonetic variants such as:  
`dipa`, `deepaa`, `deepha`, `diipa`, `namashte`, `namasthe`, `namastay`, `hi deepa`, `hello deepa`, `hei deepa`, `hay deepa`, `okay deepa`, and more.

---

## Usage

### Running the App

1. Open `deepa-wake-word.html` directly in **Google Chrome** or **Microsoft Edge**.
2. The browser will request microphone access — click **Allow**.
3. Click the **"Start Listening"** button (or the orb).
4. Speak one of the wake phrases.
5. A detection card shows the recognised phrase, language, timestamp, and confidence score.
6. Click **"Stop Listening"** to pause. The microphone permission is retained for the session so you will **not** be prompted again.

> **Tip:** Use **Chrome** for best results with Nepali phoneme coverage.

### Clearing the Log

Click the **Clear** button in the Detection Log panel to wipe all entries.

---

## How It Works

```
Microphone → Web Speech API → transcript → Normalise → Fuzzy Score → Threshold Check → Detection
```

1. **Continuous STT** — `SpeechRecognition` runs with `continuous: true` and `interimResults: true`, so it streams partial results in real time without stopping.
2. **Multi-alternative results** — Up to **3 recognition hypotheses** per utterance are scored independently, capturing accent variation across alternatives.
3. **Fuzzy keyword matching** — Levenshtein edit-distance is run against every keyword variant. A token-level bigram scan catches split-word recognition artefacts.
4. **Threshold + cooldown** — A score ≥ **0.70** fires a detection. A **2.2 s cooldown** per language prevents double-fires.

---

## Implementation Summary

### Fuzzy Matching Engine

The core scoring function `scoreTranscript(raw)` operates in three passes:

| Pass | Technique | Score |
|------|-----------|-------|
| 1 | Direct substring match against all keyword variants | 1.0 (full) |
| 2 | Token-level fuzzy match using Levenshtein distance | 0.35 – 0.92 |
| 3 | Sliding window bigram scan with edit-distance penalty | variable |

**Regular expressions** used during normalisation are defined as named constants:

```js
const RE_STRIP_NON_ALPHA  = /[^a-z\s]/g;  // strip punctuation/digits
const RE_WHITESPACE_SPLIT = /\s+/;         // tokenise on whitespace
```

### Scoring Pipeline

- Raw transcript is **lowercased** and stripped of non-alpha characters via `RE_STRIP_NON_ALPHA`.
- Tokens are split using `RE_WHITESPACE_SPLIT`.
- Each token is checked against three token lists — `DEEPA_TOKENS`, `NAMASTE_TOKENS`, `HEY_TOKENS` — using `fuzzyInList()`.
- Scores from all three passes are combined with `Math.max()` to keep the highest confidence.
- A short **exponential moving average (EMA)** smooths the rolling score displayed on the confidence meters:
  ```
  rollingScore = oldScore × 0.4 + newScore × 0.6
  ```

### Session-Persistent Microphone

The microphone stream (`sessionStream`) is obtained **once per page load** via `navigator.mediaDevices.getUserMedia()` and reused across Start/Stop cycles. This avoids repeated browser permission prompts during a session.

- On `DOMContentLoaded`, if the permission is already `granted`, the stream is silently pre-warmed.
- On `stopListening()`, the stream tracks are **not** stopped — only the audio context and recogniser are torn down.
- The stream is released cleanly on `beforeunload`.

### Silence Watchdog

A `setTimeout`-based watchdog (`silenceTimer`) runs while listening is active. If no speech result arrives within **8 seconds**, the recogniser is aborted and automatically restarted via its `onend` handler. This prevents the STT engine from silently dying after long pauses.

```
Any speech result → resetSilenceTimer() → clears and restarts the 8 s timer
8 s elapses with no result → abort recogniser → onend → restart
```

### LRU Detection Log

Detections are stored in a fixed-capacity array (`detectionLRU`, capacity = **10**). When full, the oldest entry is evicted with `Array.shift()` before pushing the new one — a simple LRU eviction pattern. The log is re-rendered on every detection, displaying the newest entry at the top.

### Audio Waveform & Confidence Meters

- A **Web Audio API** `AnalyserNode` reads raw float time-domain data from the microphone stream every animation frame.
- The waveform is drawn on a `<canvas>` element with energy-reactive opacity (brighter when louder).
- Two confidence meter bars (Nepali / English) decay smoothly toward zero between detections using a per-frame `METER_DECAY_STEP`.

---

## Configuration Constants

All tunable values are top-level `const` declarations — no magic numbers in logic.

| Constant | Default | Description |
|---|---|---|
| `THRESHOLD` | `0.70` | Minimum score to fire a detection |
| `COOLDOWN_MS` | `2200` | ms between detections per language |
| `SILENCE_TIMEOUT_MS` | `8000` | ms of silence before recogniser restart |
| `REMEMBER_LOGS` | `10` | LRU log capacity |
| `ORB_RESET_DELAY_MS` | `900` | ms before orb returns to listening state |
| `FUZZY_DIST_DEEPA` | `2` | Max edit distance for deepa token |
| `FUZZY_DIST_NAMASTE` | `2` | Max edit distance for namaste token |
| `FUZZY_DIST_HEY` | `1` | Max edit distance for hey token |
| `BIGRAM_MAX_DIST` | `3` | Max edit distance for bigram scan |
| `STT_MAX_ALTERNATIVES` | `3` | Recognition hypotheses per utterance |
| `EMA_OLD_WEIGHT` | `0.4` | Weight of previous rolling score |
| `EMA_NEW_WEIGHT` | `0.6` | Weight of new score |
| `TONE_FREQ_NE` | `528 Hz` | Detection tone for Nepali |
| `TONE_FREQ_EN` | `440 Hz` | Detection tone for English |

---

## Browser Compatibility

| Browser | Support | Notes |
|---|---|---|
| Chrome | ✅ Best | Full support, best Nepali phoneme coverage |
| Edge | ✅ Good | Full support |
| Safari | ⚠️ Partial | Limited SpeechRecognition support |
| Firefox | ❌ No | `SpeechRecognition` not supported |
