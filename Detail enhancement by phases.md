# Detail enhancement by phases.md

# Part 2 — Machine Learning and AI Enhancements

This document converts Part 2 of the roadmap into a practical development plan for implementation in Astanga.

---

## Phase 2.1 — Journal Sentiment + Insight Analysis

### Objective
Analyze journal entries locally using HuggingFace Transformers and surface emotional trends plus practice correlations.

### Scope
- Sentiment analysis for each journal entry
- Mood trend chart by week
- Insight cards such as “more peace this week”
- Correlation between journal sentiment and practice consistency

### Files
- New: `src/lib/journalAnalysis.ts`
- Update: `src/pages/Habits.tsx`
- Update: journal data fetch layer / IPC bridge if needed

### Technical tasks
- Create lazy-loaded sentiment pipeline using `Xenova/distilbert-base-uncased-finetuned-sst-2-english`
- Normalize model output into `positive | neutral | negative`
- Aggregate sentiment history by day and week
- Join journal sentiment with `sessions` table on date
- Add chart-ready output formatter
- Add UI cards for trend summaries

### Data model additions
Suggested derived shape:
```ts
interface JournalSentimentResult {
  entryId: string;
  date: string;
  label: 'positive' | 'neutral' | 'negative';
  score: number;
}
```

### Acceptance criteria
- New journal entries can be analyzed without freezing the UI
- Habits page shows mood trend history
- At least one insight card is generated from journal data
- If model loading fails, journaling still works normally

### Risks
- First model load may be heavy
- Renderer blocking if inference is done on the main thread
- Sentiment labels may feel simplistic for spiritual journaling

---

## Phase 2.2 — Personalized Practice Recommendations

### Objective
Generate relevant practice suggestions from session history and badge progress.

### Scope
- Rule-based recommendations first
- Show suggestions on Today, Meditation, and Pranayama pages
- Prepare design for future MCP/ML recommendation engine

### Files
- New: `src/lib/recommendations.ts`
- Update: `src/pages/Today.tsx`
- Update: `src/pages/Pranayama.tsx`
- Update: `src/pages/Meditation.tsx`

### Rule examples
- You have not done Nāda meditation in 7 days
- You used the same preset 10 times in a row
- You are close to unlocking a badge
- Your highest focus scores happen at a certain time of day

### Technical tasks
- Build query helpers over `sessions` table
- Add recommendation scoring and deduping
- Prioritize only one or two recommendations at a time
- Add UI recommendation card component
- Add telemetry / logging for accepted recommendations later

### Acceptance criteria
- Recommendations appear from real user history
- Recommendations change over time
- Empty-state fallback exists for new users
- Recommendation generation completes quickly on page load

### Risks
- Too many recommendations can feel noisy
- Weak recommendations reduce trust
- Needs clean session taxonomy to avoid poor suggestions

---

## Phase 2.3 — Pose Correction Coaching

### Objective
Use Human.js body landmarks to provide live posture guidance and a post-session posture report.

### Scope
- Shoulder symmetry score
- Spine alignment proxy
- Forward head detection during kumbhaka
- Voice cues during session
- Post-session posture summary

### Files
- Update: `src/lib/attentionMonitor.ts`
- Update: `src/components/pranayama/PranayamaPlayer.tsx`
- Optional update: session summary component

### Technical tasks
- Derive posture metrics from existing Human.js landmarks
- Define thresholds for each posture issue
- Throttle cue generation to avoid repeated voice interruptions
- Integrate with Kokoro voice engine
- Save summarized posture compliance in session metadata

### Example derived metrics
```ts
interface PostureMetrics {
  shoulderDiffPx: number;
  headPitchDeg: number;
  spineLeanDeg: number;
  alignedPct: number;
}
```

### Acceptance criteria
- Live cues trigger only when posture issue persists beyond threshold
- Post-session report shows percentage of aligned time
- No major CPU spike beyond existing camera analysis baseline
- User can disable posture coaching

### Risks
- False positives from camera angle changes
- Too many cues may disturb meditation flow
- Device performance differences can affect reliability

---

## Phase 2.4 — Heart Rate Estimation from Video (rPPG)

### Objective
Estimate BPM from forehead color changes using the existing live camera feed.

### Scope
- Forehead ROI extraction
- Green-channel signal tracking
- Bandpass filtering and FFT frequency estimation
- Live BPM display when confidence is high
- Post-session BPM trend summary

### Files
- New: `src/lib/rppgEngine.ts`
- Update: meditation player HUD / session overlay
- Update: analytics/session summary components

### Technical tasks
- Identify stable forehead region from face mesh
- Maintain rolling signal buffer
- Filter the signal for physiological frequencies
- Estimate BPM and confidence score
- Hide output when motion/light conditions are poor
- Store session start/end BPM and trend data

### Acceptance criteria
- BPM appears only under stable, high-confidence conditions
- No BPM shown when face tracking is weak
- Session summary can show a start-to-end trend when enough data exists
- Feature can be toggled off globally

### Risks
- Light flicker and movement reduce accuracy
- Hard to validate without reference device
- May require heavy tuning across webcams and skin tones

---

## Phase 2.5 — Emotion-Aware Session Adaptation

### Objective
Use Human.js emotion inference to adapt the opening of a session and suggest a more suitable practice.

### Scope
- Pre-session emotion snapshot
- Emotion-driven practice suggestion
- Start-vs-end emotion comparison
- Settings toggle for opt-in use

### Files
- Update: `src/lib/attentionMonitor.ts`
- Update: `src/store/attentionMonitorStore.ts`
- Update: Settings page once available

### Technical tasks
- Buffer emotion readings over a short pre-session window
- Derive dominant emotional state with smoothing
- Map emotional states to a small set of practice suggestions
- Save start/end emotion labels in session metadata
- Write supportive copy instead of explicit emotional labeling

### Acceptance criteria
- Suggestion appears before the session without blocking start
- Suggestions feel calm and non-judgmental
- Start/end emotional change can be shown in summary when confidence is sufficient
- Feature is disabled by default unless user opts in

### Risks
- Emotion inference can be inaccurate
- Direct emotional feedback may feel invasive
- Needs careful tone design to preserve user trust

---

## Phase 2.6 — VITS Voice Synthesis (Emotion-Driven TTS)

### Objective
Introduce a second TTS engine with stronger control over delivery style for specific practices and celebratory moments.

### Scope
- Add VITS engine wrapper
- Support voice style presets by practice type
- Allow switching between Kokoro and VITS
- Fall back safely if VITS fails

### Files
- New: `src/lib/vitsEngine.ts`
- Update: voice preference store
- Update: Settings page voice section
- Optional update: session narration layer

### Technical tasks
- Implement VITS engine abstraction similar to Kokoro engine
- Add model loading and caching
- Add practice-to-voice profile mapping
- Add engine switch in preferences
- Ensure narration timing remains aligned with guided sessions

### Acceptance criteria
- User can choose a voice engine
- Guided playback still works if VITS is unavailable
- Different practice profiles sound noticeably distinct
- No regression in existing Kokoro-based playback

### Risks
- Added model weight and load time
- Voice mismatch with existing session pacing
- More maintenance for dual-engine support

---

## Phase 2.7 — Claude MCP Active Coaching

### Objective
Connect session completion and journal context to the MCP server so Claude can provide reflective coaching and adaptive challenges.

### Scope
- Session-end IPC event
- Weekly review tool
- Journal coaching prompt tool
- Adaptive challenge generator
- Reflection prompt write-back to journal

### Files
- Update: `electron/ipc.ts`
- Update: `mcp-server/index.ts`
- New: `mcp-server/tools/coaching.ts`

### Technical tasks
- Emit `session:complete` event from renderer/main pipeline
- Register new MCP coaching tools
- Design tool contracts for weekly review and challenge generation
- Persist Claude-generated prompts to journal or challenge state
- Add error handling if MCP/Claude is unavailable

### Dependencies
- Best implemented after Settings/MCP activation work is complete
- Requires stable IPC event channel design

### Acceptance criteria
- Session completion can trigger an MCP coaching workflow
- Coaching output can be stored back into the app
- Weekly review can summarize recent patterns
- Failure in MCP flow does not affect session completion UX

### Risks
- Strong dependency on external Claude/MCP workflow quality
- Needs careful boundaries to avoid over-coaching
- Debugging event-driven flows across renderer/main/MCP may be time-consuming

---

## Suggested implementation order

| Order | Enhancement | Reason |
|---|---|---|
| 1 | 2.1 Journal Sentiment | Lowest integration risk, uses already installed package |
| 2 | 2.2 Recommendations | High visible value, mostly business logic |
| 3 | 2.5 Emotion Adaptation | Existing Human.js data can be surfaced quickly |
| 4 | 2.3 Pose Correction | Reuses Human.js and existing session flow |
| 5 | 2.4 rPPG | Highest technical complexity |
| 6 | 2.6 VITS TTS | Nice upgrade but not core to phase value |
| 7 | 2.7 MCP Coaching | Depends on broader MCP settings work |

---

## Recommended delivery strategy

### Milestone A — Fast wins
- 2.1 Journal Sentiment
- 2.2 Recommendation Engine
- 2.5 Emotion Adaptation

### Milestone B — Camera intelligence
- 2.3 Pose Correction
- 2.4 rPPG

### Milestone C — Voice and coaching
- 2.6 VITS TTS
- 2.7 Claude MCP Coaching

---

## Engineering notes

- All ML inference should be lazy-loaded
- Never block the renderer during model loading or inference
- All camera-based intelligence should be opt-in
- Raw video/audio should not be stored
- Every feature should degrade gracefully when unavailable
- Keep feature flags for each enhancement during rollout
