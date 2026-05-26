# Skill.md — Part 2: Machine Learning and AI Enhancements

> **App Context:** Astanga — Electron desktop app (Vite 5 · React 18 · TypeScript · Tailwind 3 · Framer Motion 11 · Zustand · Electron 33 · SQLite WAL · Kokoro 82M TTS · Human.js + MoveNet · MCP server)

---

## Installed but Unused Infrastructure

Before writing a single line of code, internalize what is **already wired in**:

| Package | Version | Status |
|---|---|---|
| `@huggingface/transformers` | v3.0.0 | Installed, unused |
| `@diffusionstudio/vits-web` | v1.0.3 | Installed, unused |
| `@vladmandic/human` | latest | Active (face/emotion/body) |
| `kokoro-js` | active | Active (TTS) |
| `@modelcontextprotocol/sdk` | active | Active (MCP server) |

Human.js already classifies **7 emotions per frame** and provides **468-point face mesh**, shoulder positions, head yaw/pitch/roll — all collected but never surfaced in UI.

---

## Sub-Feature Skills

---

### 2.1 Journal Sentiment + Insight Analysis

**Goal:** Run on-device NLP on journal text to surface mood trends and practice correlations.

**Pattern to follow:**
```
pipeline('sentiment-analysis', 'Xenova/distilbert-base-uncased-finetuned-sst-2-english')
```

**New file:** `src/lib/journalAnalysis.ts`

**Key skills needed:**
- `@huggingface/transformers` pipeline API (lazy-load, don't block app startup)
- Aggregate per-entry sentiment scores into weekly mood arrays
- Query `journal` SQLite table via existing IPC bridge
- Correlate `sessions` table (practice days) with positive-entry days using simple Pearson or rank correlation
- Surface results in `src/pages/Habits.tsx` as a mood trend line chart (Recharts or existing chart lib)

**Data flow:**
```
journal SQLite rows → journalAnalysis.ts → sentiment scores → Zustand slice → Habits page chart
```

**Gotchas:**
- HuggingFace Transformers v3 uses ONNX Runtime Web — first run downloads model (~67 MB). Cache in `userData` via Electron's `app.getPath('userData')`.
- Run inference in a **Web Worker** or **Electron utility process** to avoid blocking the renderer thread.
- Debounce: only re-run when journal entries are added/updated, not on every render.

---

### 2.2 Personalized Practice Recommendations

**Goal:** Show contextual nudges on Today page and practice landing pages.

**New file:** `src/lib/recommendations.ts`

**Phase A — Rule Engine (build first):**

```typescript
// Rules to implement:
// 1. "You haven't done Nāda meditation in 7 days"
// 2. "Same preset 10× in a row — try something else"
// 3. "You're 3 days from Prāṇa badge"
// 4. "Your best attention scores are at 6 AM — practice now"
```

**Data sources available:**
- `sessions` SQLite table — `type`, `preset`, `started_at`, `attention.focusPercent`
- Zustand challenge store — badge progress
- Zustand routine store — current routine

**Phase B — MCP / ML upgrade (after rule engine works):**
- Pass session history JSON to Claude via MCP tool call
- Or use HuggingFace collaborative filtering (lightweight k-NN on session vectors)

**Surface recommendations:**
- `src/pages/Today.tsx` — card above routine list
- `src/pages/Pranayama.tsx` — subtitle under page heading
- `src/pages/Meditation.tsx` — subtitle under page heading

---

### 2.3 Pose Correction Coaching

**Goal:** Use already-available Human.js skeletal data to give real-time voice posture cues.

**Files to modify:** `src/lib/attentionMonitor.ts` · `src/components/pranayama/PranayamaPlayer.tsx`

**Metrics to derive from existing Human.js output:**

| Metric | Derivation | Cue trigger |
|---|---|---|
| Shoulder symmetry | `left.shoulder.y - right.shoulder.y` > threshold | "Relax right shoulder" |
| Spine alignment proxy | Nose x vs shoulders midpoint x | "Sit upright — lean detected" |
| Forward head (kumbhaka) | `face.rotation.pitch` > +15° | "Tuck chin slightly" |
| Head drop (drowsiness) | `face.rotation.pitch` < -20° | "Lift your gaze gently" |

**Voice cue delivery:** Pipe through existing `kokoroEngine.ts` — do not add a second TTS engine for this.

**Post-session posture report:** Store per-second alignment snapshots compressed (% time in range) in `sessions` table as a new JSON column `postureReport`.

**Skill note:** Throttle Human.js pose reads to **2–4 Hz** for posture (not 30 fps) — CPU cost is high on laptops without GPU.

---

### 2.4 Heart Rate Estimation from Video (rPPG)

**Goal:** Estimate BPM from forehead color fluctuation using the existing camera feed.

**New file:** `src/lib/rppgEngine.ts`

**Algorithm — Green Channel rPPG:**
1. Define a 40×40 px ROI over detected forehead (use `face.mesh` landmark ~10 = forehead center)
2. Extract mean green channel value per frame → 30 fps signal buffer
3. Bandpass filter: 0.7–3.5 Hz (42–210 BPM physiological range) using a simple IIR filter
4. Find dominant frequency via FFT (use `fft.js` or implement lightweight DFT for short buffers)
5. Convert dominant frequency → BPM

**Display:**
- Live BPM badge during meditation session (bottom-right HUD, subtle)
- Post-session: "Heart rate: 78 → 62 bpm over 10 min" in session summary
- Monthly trend in Analytics dashboard

**Accuracy notes:**
- Works well in stable indoor light (typical meditation setup)
- Fails under flickering lights or strong motion — gate output: only show BPM when `face.box` movement < threshold for last 5s
- Do not show BPM if confidence < 0.6

**HRV proxy:** Compute beat-to-beat interval variance from the peak intervals → parasympathetic activation proxy (high HRV = calm).

---

### 2.5 Emotion-Aware Session Adaptation

**Goal:** Use Human.js emotion classifications (already computed) to adapt session start and auto-suggest practices.

**Files:** `src/lib/attentionMonitor.ts` · `src/store/attentionMonitorStore.ts`

**Pre-session snapshot logic:**
```typescript
// Sample emotions over 3s (90 frames) → dominant emotion
const dominantEmotion = getDominantEmotion(emotionBuffer); 

const suggestion = {
  angry:    { type: 'pranayama', preset: 'Extended Exhale' },
  fearful:  { type: 'pranayama', preset: 'Cyclic Sigh' },
  sad:      { type: 'meditation', practice: 'Nāda Anusandhāna' },
  happy:    null, // no change
  neutral:  null,
  calm:     null,
};
```

**Post-session delta:**
- Store `emotionAtStart` and `emotionAtEnd` in `sessions` table
- Surface: "Your expression shifted from tense → neutral at minute 4"

**Opt-in gate:**
```typescript
attentionPreferences.emotionAdaptation: boolean  // add to store + Settings page toggle
```

**Do not** show emotion labels to the user — show only gentle, supportive language ("You look a little tense today — here's a calming start").

---

### 2.6 VITS Emotion-Driven TTS

**Goal:** Replace or supplement Kokoro with VITS for emotion-tagged prosody.

**New file:** `src/lib/vitsEngine.ts` — mirror the pattern of `src/lib/kokoroEngine.ts`

**`@diffusionstudio/vits-web` API pattern:**
```typescript
import { PiperTTSEngine } from '@diffusionstudio/vits-web';

const engine = new PiperTTSEngine();
await engine.load('en_US-lessac-medium'); // or similar model

const audio = await engine.synthesize(text, {
  speakingRate: 0.75,  // slow for Yoga Nidrā
  pitch: -2,           // deeper for meditation
});
```

**Practice-specific voice profiles:**

| Practice | speakingRate | pitch | notes |
|---|---|---|---|
| Vipassana | 0.85 | 0 | Detached, equanimous |
| Yoga Nidrā | 0.65 | -3 | Hypnotic, slow |
| Ātma Vichāra | 0.70 | -1 | Quiet, inward |
| Celebrations / badges | 1.1 | +2 | Warm, uplifting |

**Integration:** Add `voiceEngine: 'kokoro' | 'vits'` to `voicePreferences` store. Settings page lets user toggle. Fall back to Kokoro if VITS model fails to load.

---

### 2.7 Claude MCP — Active Coaching (Phase E+)

**Goal:** Extend the MCP server to receive session lifecycle events and push coaching actions back.

**Files:** `electron/ipc.ts` · `mcp-server/index.ts` · `mcp-server/tools/coaching.ts` (new)

**IPC event to add:**
```typescript
// electron/ipc.ts
ipcMain.on('session:complete', (event, sessionData) => {
  mcpServer.emit('session:complete', sessionData);
});
```

**New MCP tools to register in `coaching.ts`:**

| Tool name | Input | Action |
|---|---|---|
| `get_weekly_review` | `{ userId, weekStart }` | Returns practice summary for Claude to narrate |
| `get_journal_coach_prompt` | `{ recentEntries[] }` | Claude returns a deepening question |
| `generate_challenge` | `{ quizWeakSpots[], sessionHistory[] }` | Claude returns custom 7-day plan |
| `push_reflection_prompt` | `{ prompt: string, sessionId }` | Writes prompt to `journal` table |

**Flow:**
```
Session ends → renderer emits IPC → main process → MCP server event → 
Claude Desktop receives → calls coaching tool → result written to journal/challenge store
```

---

## Cross-Cutting Rules for All 2.x Features

1. **Never block the main/renderer thread** — all ML inference (HuggingFace, rPPG FFT, VITS) must run in Web Workers or Electron utility processes.
2. **Lazy-load all models** — load on first use, cache in `userData`. Never load on app startup.
3. **Gate every ML feature behind a settings toggle** — camera features need explicit opt-in.
4. **Graceful degradation** — if a model fails to load, the feature silently hides itself; the app never crashes.
5. **No audio stored, no video stored** — rPPG and emotion use live frames only; do not persist raw media.
6. **Throttle Human.js reads** — use existing `attentionMonitor.ts` tick loop; do not add separate intervals.
7. **Single source of truth** — all ML outputs flow into Zustand stores before reaching UI components.

---

## Recommended Build Order

| Step | Feature | Why first |
|---|---|---|
| 1 | 2.1 Journal Sentiment | Lowest risk — no camera, pure text pipeline, HF already installed |
| 2 | 2.2 Recommendations (Rule Engine) | No ML needed yet, high user value, fast to ship |
| 3 | 2.5 Emotion Adaptation | Human.js data already flowing, just need to surface it |
| 4 | 2.3 Pose Correction | Human.js data already flowing, adds real session value |
| 5 | 2.4 rPPG Heart Rate | Most complex signal processing, needs stable camera baseline |
| 6 | 2.6 VITS TTS | Low priority unless voice quality is a user complaint |
| 7 | 2.7 MCP Coaching | Depends on Phase E (Settings/MCP) being complete first |
