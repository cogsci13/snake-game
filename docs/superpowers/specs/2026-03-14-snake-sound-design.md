# Snake Game — Sound Design Spec
**Date:** 2026-03-14
**Status:** Approved
**Scope:** Adding background music + sound effects to the existing `snake.html`

---

## Overview

Add Web Audio API–synthesized sound to the kids' snake game. All sound is generated inline in JS — no external files, no libraries. Sounds include a looping background melody per stage (3 different themes) and distinct sound effects for eating food (per stage), collision/death, stage clear, and victory. A mute toggle button (🔊/🔇) sits in the HUD.

---

## Tech

- **API:** Web Audio API (`AudioContext`, `OscillatorNode`, `GainNode`)
- **No external dependencies or audio files**
- `AudioContext` created on first arrow key press (browser autoplay policy requires user gesture)
- Single shared `AudioContext` instance reused for all sounds

---

## Background Music

Each stage loops a short melodic phrase (6–8 notes) continuously during gameplay. Music stops during overlays (stage clear, game over, victory) and restarts from the beginning of the current stage's phrase when gameplay resumes.

| Stage | Waveform | Tempo | Character |
|-------|----------|-------|-----------|
| 🌿 Jungle | `square` | 160 BPM | Bouncy, upbeat chiptune — adventure feel |
| 🌊 Ocean | `sine` | 100 BPM | Smooth, flowing — calm and wavy |
| 🚀 Space | `sawtooth` | 130 BPM | Arpeggiated, mysterious — electronic sci-fi |

**Implementation:**
- Each stage defines a `notes` array of `[frequency (Hz), duration (seconds)]` pairs
- Music loop: schedule next note using `AudioContext.currentTime` + cumulative offset, then `setTimeout` to re-trigger the phrase when it ends
- On stage change: cancel current music loop, start the new stage's phrase from the beginning
- On mute: stop the loop; on unmute: restart from beginning of current phrase

**Note sequences with Hz values:**

Jungle (square, 160 BPM → ~0.375s per beat):
```
[523, 0.18], [659, 0.18], [784, 0.18], [659, 0.18],
[523, 0.18], [392, 0.18], [659, 0.18], [523, 0.36]
```
(C5=523, E5=659, G5=784, G4=392)

Ocean (sine, 100 BPM → ~0.6s per beat):
```
[262, 0.3], [330, 0.3], [392, 0.3], [440, 0.3],
[392, 0.3], [330, 0.6]
```
(C4=262, E4=330, G4=392, A4=440)

Space (sawtooth, 130 BPM → ~0.46s per beat):
```
[220, 0.23], [330, 0.23], [440, 0.23], [523, 0.23],
[659, 0.23], [440, 0.23], [330, 0.23], [220, 0.46]
```
(A3=220, E4=330, A4=440, C5=523, E5=659)

---

## Sound Effects

All effects are one-shot: create oscillator + gain, play, then auto-disconnect.

### Eat Food (per stage)

| Stage | Sound | Waveform | Description |
|-------|-------|----------|-------------|
| 🌿 Jungle | Bright chirp | `square` | Quick ascending sweep 400→800 Hz, 0.12s |
| 🌊 Ocean | Bubble pop | `sine` | Short descending 600→200 Hz, 0.15s |
| 🚀 Space | Laser zap | `sawtooth` | Sharp ascending 300→1200 Hz with fast decay, 0.08s (distinctly higher/faster than Jungle) |

### Other Events

| Event | Sound | Waveform | Description |
|-------|-------|----------|-------------|
| Collision/death | Low wah | `sine` | Descending 400→100 Hz, 0.4s, with vibrato feel |
| Stage clear | Fanfare | `square` | 3-note ascending: C5→E5→G5, each 0.15s |
| Victory | Arpeggio | `square` | 5-note: C5→E5→G5→C6→E6, each 0.12s |

---

## Mute Toggle

- **Button:** Small text button added as a 4th element in the HUD row, right-aligned
- **Labels:** 🔊 (unmuted) / 🔇 (muted)
- **State:** `state.muted` boolean (default: `false`)
- **On mute:** Stop background music loop immediately; suppress all future sound effects
- **On unmute:** Restart current stage's background music from beginning
- **Style:** Matches existing HUD font, transparent background, no border — consistent with HUD aesthetic

---

## Audio Lifecycle

```
First arrow key press
  → AudioContext created (satisfies browser autoplay policy)
  → startGame() called
  → startMusic(0) called immediately

Stage advance (handleFoodEaten detects stage complete)
  → stopMusic()           ← stops background music immediately
  → playSound('stageClear')  ← 3-note jingle plays (0.45s total)
  → showStageClear() draws overlay
  → 2s later (setTimeout): startMusic(newStageIdx) ← new theme begins

Life lost (handleCollision)
  → stopMusic() is NOT called — music continues through flash
  → playSound('death') plays over the music during the 0.5s flash
  → after flash: music resumes on same phrase position

Game Over (showGameOver)
  → stopMusic()
  → death sound already played at collision moment

Victory (showVictory)
  → stopMusic()
  → playSound('victory') ← 5-note arpeggio plays

fullReset (Try Again / Play Again button clicked)
  → stopMusic()           ← safety stop
  → initState()
  → startMusic(0)         ← called immediately inside fullReset(), same call site as startLoop()

Mute toggled ON
  → clearTimeout(musicTimeout); musicTimeout = null
  → In-flight one-shot effects (eat/death sounds < 0.4s) play to completion — no cancellation
  → All subsequent sound calls return immediately (state.muted check at entry)

Mute toggled OFF
  → startMusic(state.stageIdx) restarts current stage theme from beginning
```

---

## Implementation Notes

- **AudioContext creation:** Create once on first keydown (before `startGame()`), store as module-level `let audioCtx = null`. All sound functions guard against null audioCtx with an early return: `if (!audioCtx || state.muted) return;`

- **Music loop:** Use a single `let musicTimeout = null` handle for ALL pending music timers (both the note-to-note scheduling and the 2s stage-clear delay). `stopMusic()` calls `clearTimeout(musicTimeout)` and sets `musicTimeout = null`. This ensures that even a pending `startMusic()` scheduled inside `showStageClear`'s 2s delay is cancellable if `fullReset()` fires during that window.

- **Stage clear timer:** The 2s delay in `showStageClear` must be assigned to `musicTimeout`:
  ```js
  musicTimeout = setTimeout(() => { startMusic(newStageIdx); }, 2000);
  ```
  This makes it cancellable by `stopMusic()`.

- **One-shot effects:** Create a fresh oscillator + gain node per effect call; connect → start → stop at `currentTime + duration`; no reference kept. Do NOT use `audioCtx.suspend()` for muting — use only the `state.muted` flag. This ensures in-flight effects (already scheduled) always play to completion naturally.

- **Mute mechanism:** Exclusively use the `state.muted` flag + `clearTimeout(musicTimeout)`. Never call `audioCtx.suspend()` or `audioCtx.close()`. This guarantees in-flight one-shot nodes complete without abrupt cutoff.

- **Unmute handler:** Must call `clearTimeout(musicTimeout)` before `startMusic(state.stageIdx)` to cancel any pending stage-clear `startMusic` timer, preventing a double-start if the user mutes then unmutes within the 2s stage-clear window.

- **fullReset:** Calls `stopMusic()` then `startMusic(0)`. Since `audioCtx` exists by the time `fullReset` can be called (requires prior arrow key press), `startMusic` will not encounter a null context in practice. The null guard in `startMusic` is still required as defensive code.

---

## Out of Scope

- Volume slider
- Per-sound volume control
- Sound on the start screen
- Mobile vibration
