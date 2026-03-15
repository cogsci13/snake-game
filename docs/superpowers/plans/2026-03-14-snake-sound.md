# Snake Sound System Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Web Audio API–synthesized background music and sound effects to `snake.html`, with a mute toggle in the HUD.

**Architecture:** All audio is synthesized inline via the Web Audio API — no external files. A single `AudioContext` instance is created on the first arrow key press (satisfying the browser autoplay policy) and reused for everything. Background music loops per-stage using scheduled oscillators and a single `musicTimeout` handle; one-shot sound effects are fire-and-forget oscillator nodes. A `state.muted` boolean gates all sound output without suspending the AudioContext, so in-flight effects always complete naturally.

**Tech Stack:** Web Audio API (`AudioContext`, `OscillatorNode`, `GainNode`), vanilla JS, single `snake.html` file.

**Spec:** `docs/superpowers/specs/2026-03-14-snake-sound-design.md`

---

## Chunk 1: Audio Infrastructure + Background Music

### Task 1: Add audio globals, `state.muted`, and AudioContext initialization

**Files:**
- Modify: `snake.html`

**Context:** Two module-level variables track the audio context and the single pending music timer. `state.muted` defaults to `false` and is reset on `fullReset`. The `AudioContext` is created on the first arrow key press to satisfy the browser autoplay policy.

- [ ] **Step 1: Add module-level audio globals**

  Locate the line `let decoFrame = 0;` (currently at line ~129). Insert the two new globals **above** it:

  ```js
  let audioCtx = null;
  let musicTimeout = null;
  ```

- [ ] **Step 2: Add `muted: false` to the state object**

  In `initState()`, the state object ends with `resetCount: 0,`. Add `muted: false,` as the line **before** `resetCount: 0,`:

  ```js
  muted: false,
  resetCount: 0,
  ```

- [ ] **Step 3: Create `AudioContext` on first arrow key press**

  In the `keydown` handler, the `state.screen === 'start'` branch currently reads:
  ```js
  if (state.screen === 'start' && DIRS[e.key]) {
    e.preventDefault();
    startGame();
    return;
  }
  ```

  Add AudioContext creation **before** `startGame()`:
  ```js
  if (state.screen === 'start' && DIRS[e.key]) {
    e.preventDefault();
    if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    startGame();
    return;
  }
  ```

- [ ] **Step 4: Verify in browser — no errors, game still works**

  Open `snake.html` in a browser. Open the browser DevTools console. Press an arrow key to start the game. Verify:
  - No JavaScript errors in the console
  - Game starts normally (snake appears, moves)
  - `audioCtx` is created (can check in DevTools: `audioCtx` in the console will be an `AudioContext` object)

- [ ] **Step 5: Commit**

  ```bash
  cd /Users/builder/Projects/games
  git add snake.html
  git commit -m "feat(sound): add audio globals and AudioContext initialization"
  ```

---

### Task 2: Implement `startMusic()` / `stopMusic()` and integrate into game flow

**Files:**
- Modify: `snake.html`

**Context:** `startMusic(stageIdx)` schedules all notes for a melody phrase using Web Audio API's timing system, then uses a single `musicTimeout` setTimeout to re-trigger the phrase when it ends (creating a seamless loop). `stopMusic()` cancels this timer. The critical invariant: `musicTimeout` covers ALL pending audio timers, including the 2-second stage-clear delay — so `stopMusic()` can cancel a pending `startMusic` that hasn't fired yet.

Note sequences (from spec):
- Jungle (square): `[523,0.18],[659,0.18],[784,0.18],[659,0.18],[523,0.18],[392,0.18],[659,0.18],[523,0.36]`
- Ocean (sine): `[262,0.3],[330,0.3],[392,0.3],[440,0.3],[392,0.3],[330,0.6]`
- Space (sawtooth): `[220,0.23],[330,0.23],[440,0.23],[523,0.23],[659,0.23],[440,0.23],[330,0.23],[220,0.46]`

- [ ] **Step 1: Add `stopMusic()` and `startMusic()` functions**

  Locate the comment `// 9. Initial setup — show start screen` near the bottom of the `<script>`. Insert the new audio section **before** it:

  ```js
  // 10. Audio system
  function stopMusic() {
    clearTimeout(musicTimeout);
    musicTimeout = null;
  }

  function startMusic(stageIdx) {
    if (!audioCtx || state.muted) return;
    stopMusic(); // cancel any pending loop or stage-clear timer
    const melodies = [
      // Jungle: square wave, 160 BPM (~0.375s/beat → 0.18s notes)
      { type: 'square',   notes: [[523,0.18],[659,0.18],[784,0.18],[659,0.18],[523,0.18],[392,0.18],[659,0.18],[523,0.36]] },
      // Ocean: sine wave, 100 BPM (~0.6s/beat → 0.3s notes)
      { type: 'sine',     notes: [[262,0.3],[330,0.3],[392,0.3],[440,0.3],[392,0.3],[330,0.6]] },
      // Space: sawtooth wave, 130 BPM (~0.46s/beat → 0.23s notes)
      { type: 'sawtooth', notes: [[220,0.23],[330,0.23],[440,0.23],[523,0.23],[659,0.23],[440,0.23],[330,0.23],[220,0.46]] },
    ];
    const melody = melodies[stageIdx];

    function scheduleMelody() {
      if (!audioCtx || state.muted) return;
      let t = audioCtx.currentTime + 0.05; // small scheduling lookahead
      let total = 0;
      melody.notes.forEach(([freq, dur]) => {
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.type = melody.type;
        osc.frequency.value = freq;
        gain.gain.setValueAtTime(0.08, t);
        gain.gain.linearRampToValueAtTime(0, t + dur - 0.01);
        osc.connect(gain);
        gain.connect(audioCtx.destination);
        osc.start(t);
        osc.stop(t + dur);
        t += dur;
        total += dur;
      });
      musicTimeout = setTimeout(scheduleMelody, total * 1000);
    }
    scheduleMelody();
  }
  ```

- [ ] **Step 2: Call `startMusic(0)` at the end of `startGame()`**

  `startGame()` currently ends with `startLoop()`. Add `startMusic(state.stageIdx)` **after** `startLoop()`:

  ```js
  function startGame() {
    state.screen = 'play';
    stopDecoLoop();
    document.getElementById('hud').style.visibility = 'visible';
    document.getElementById('food-counter').style.visibility = 'visible';
    updateHUD();
    startLoop();
    startMusic(state.stageIdx);  // ← ADD THIS LINE
  }
  ```

- [ ] **Step 3: Add `stopMusic()` and `startMusic(0)` to `fullReset()`**

  `fullReset()` currently calls `stopLoop()` near the top and `startLoop()` near the bottom. Add `stopMusic()` right after `stopLoop()`, and `startMusic(0)` right after `startLoop()`:

  ```js
  function fullReset() {
    if (state.rafId) { cancelAnimationFrame(state.rafId); state.rafId = null; }
    stopLoop();
    stopMusic();   // ← ADD: safety stop before reinit
    clearOverlayBtn();
    initState();
    state.resetCount = (state.resetCount || 0) + 1;
    state.screen = 'play';
    document.getElementById('hud').style.visibility = 'visible';
    document.getElementById('food-counter').style.visibility = 'visible';
    updateHUD();
    render();
    startLoop();
    startMusic(0);  // ← ADD: restart Jungle theme from beginning
  }
  ```

- [ ] **Step 4: Add `stopMusic()` to `showGameOver()`**

  `showGameOver()` currently starts with `state.screen = 'gameover'`. Add `stopMusic()` as the very first line:

  ```js
  function showGameOver() {
    stopMusic();  // ← ADD THIS LINE
    state.screen = 'gameover';
    render();
    // ... rest unchanged
  }
  ```

- [ ] **Step 5: Add `stopMusic()` + `startMusic()` to `showStageClear()`**

  The stage clear flow per spec:
  1. `stopMusic()` — stop current stage's music immediately (called in `handleFoodEaten` before `showStageClear()`, see Step 6)
  2. Stage clear overlay is shown
  3. The 2s `setTimeout` is assigned to `musicTimeout` so it's cancellable
  4. Inside the 2s callback: `startMusic(state.stageIdx)` after `state.stageIdx` is incremented

  Modify `showStageClear()` — change the `setTimeout` to assign to `musicTimeout` and add `startMusic` at the end of the callback:

  Find this block at the bottom of `showStageClear()`:
  ```js
  const snapshotReset = state.resetCount;
  setTimeout(() => {
    if (state.resetCount !== snapshotReset) return;
    if (state.stageIdx >= STAGES.length - 1) return;
    state.stageIdx += 1;
    state.foodEaten = 0;
    state.snake = createSnake();
    state.dir = { x: 1, y: 0 };
    state.nextDir = { x: 1, y: 0 };
    state.currentDirKey = 'ArrowRight';
    state.food = spawnFood(state.snake);
    state.screen = 'play';
    updateHUD();
    startLoop();
  }, 2000);
  ```

  Replace it with:
  ```js
  const snapshotReset = state.resetCount;
  musicTimeout = setTimeout(() => {  // ← assigned to musicTimeout so stopMusic() can cancel it
    if (state.resetCount !== snapshotReset) return;
    if (state.stageIdx >= STAGES.length - 1) return;
    state.stageIdx += 1;
    state.foodEaten = 0;
    state.snake = createSnake();
    state.dir = { x: 1, y: 0 };
    state.nextDir = { x: 1, y: 0 };
    state.currentDirKey = 'ArrowRight';
    state.food = spawnFood(state.snake);
    state.screen = 'play';
    updateHUD();
    startLoop();
    startMusic(state.stageIdx);  // ← start the new stage's theme
  }, 2000);
  ```

- [ ] **Step 6: Call `stopMusic()` in `handleFoodEaten()` before `showStageClear()`**

  In `handleFoodEaten()`, the stage-complete branch currently reads:
  ```js
  if (state.foodEaten >= FOOD_PER_STAGE) {
    stopLoop();
    if (state.stageIdx === STAGES.length - 1) showVictory();
    else showStageClear();
  }
  ```

  Replace with:
  ```js
  if (state.foodEaten >= FOOD_PER_STAGE) {
    stopLoop();
    if (state.stageIdx === STAGES.length - 1) {
      showVictory();
    } else {
      stopMusic();  // ← stop current stage's music immediately
      showStageClear();
    }
  }
  ```

  (Note: `showVictory()` will call `stopMusic()` internally in Task 3.)

- [ ] **Step 7: Verify background music in browser**

  Open `snake.html`. Press arrow key to start. Listen for:
  - Stage 1 (Jungle): bouncy square wave melody
  - Eat 10 foods: music stops → stage clear overlay → 2s pause → Stage 2 music starts (smooth sine wave)
  - Stage 2 (Ocean): slower, smooth sine wave melody
  - Eat 10 more: same transition → Stage 3 (Space): arpeggiated sawtooth

  Also verify:
  - Hitting a wall: music continues playing through the flash
  - Clicking "Try Again": music stops and Jungle theme restarts
  - No console errors

- [ ] **Step 8: Commit**

  ```bash
  cd /Users/builder/Projects/games
  git add snake.html
  git commit -m "feat(sound): add background music with per-stage melodies"
  ```

---

## Chunk 2: Sound Effects + Mute Toggle

### Task 3: Implement `playEatSound()` and `playSound()`, integrate at event sites

**Files:**
- Modify: `snake.html`

**Context:** All effects are fire-and-forget — create oscillator, connect, start, stop at `currentTime + duration`. No references kept. The eat sound varies by stage; death/stageClear/victory are universal. Music continues through the death flash (not stopped — only stopped at Game Over).

Sound specs:
- Eat Jungle: square 400→800 Hz, 0.12s
- Eat Ocean: sine 600→200 Hz, 0.15s
- Eat Space: sawtooth 300→1200 Hz, 0.08s (distinctly higher/faster than others)
- Death: sine 400→100 Hz, 0.4s (plays over music during flash)
- Stage clear: square, C5(523)→E5(659)→G5(784), each 0.15s
- Victory: square, C5(523)→E5(659)→G5(784)→C6(1047)→E6(1319), each 0.12s

- [ ] **Step 1: Add `playEatSound()` and `playSound()` to the audio section**

  After the `startMusic()` function (in the `// 10. Audio system` section), add:

  ```js
  function playEatSound(stageIdx) {
    if (!audioCtx || state.muted) return;
    const configs = [
      { type: 'square',   startF: 400, endF: 800,  dur: 0.12 }, // Jungle: bright chirp
      { type: 'sine',     startF: 600, endF: 200,  dur: 0.15 }, // Ocean: bubble pop
      { type: 'sawtooth', startF: 300, endF: 1200, dur: 0.08 }, // Space: laser zap
    ];
    const { type, startF, endF, dur } = configs[stageIdx];
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    osc.type = type;
    osc.frequency.setValueAtTime(startF, audioCtx.currentTime);
    osc.frequency.linearRampToValueAtTime(endF, audioCtx.currentTime + dur);
    gain.gain.setValueAtTime(0.15, audioCtx.currentTime);
    gain.gain.linearRampToValueAtTime(0, audioCtx.currentTime + dur);
    osc.connect(gain);
    gain.connect(audioCtx.destination);
    osc.start();
    osc.stop(audioCtx.currentTime + dur);
  }

  function playSound(name) {
    if (!audioCtx || state.muted) return;

    if (name === 'death') {
      // Low descending wah: 400→100 Hz over 0.4s
      const osc = audioCtx.createOscillator();
      const gain = audioCtx.createGain();
      osc.type = 'sine';
      osc.frequency.setValueAtTime(400, audioCtx.currentTime);
      osc.frequency.linearRampToValueAtTime(100, audioCtx.currentTime + 0.4);
      gain.gain.setValueAtTime(0.2, audioCtx.currentTime);
      gain.gain.linearRampToValueAtTime(0, audioCtx.currentTime + 0.4);
      osc.connect(gain);
      gain.connect(audioCtx.destination);
      osc.start();
      osc.stop(audioCtx.currentTime + 0.4);
    }

    if (name === 'stageClear') {
      // 3-note ascending fanfare: C5→E5→G5, each 0.15s
      [[523, 0], [659, 0.15], [784, 0.30]].forEach(([freq, offset]) => {
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.type = 'square';
        osc.frequency.value = freq;
        const t = audioCtx.currentTime + offset;
        gain.gain.setValueAtTime(0.12, t);
        gain.gain.linearRampToValueAtTime(0, t + 0.13);
        osc.connect(gain);
        gain.connect(audioCtx.destination);
        osc.start(t);
        osc.stop(t + 0.15);
      });
    }

    if (name === 'victory') {
      // 5-note arpeggio: C5→E5→G5→C6→E6, each 0.12s
      [[523, 0], [659, 0.12], [784, 0.24], [1047, 0.36], [1319, 0.48]].forEach(([freq, offset]) => {
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.type = 'square';
        osc.frequency.value = freq;
        const t = audioCtx.currentTime + offset;
        gain.gain.setValueAtTime(0.12, t);
        gain.gain.linearRampToValueAtTime(0, t + 0.10);
        osc.connect(gain);
        gain.connect(audioCtx.destination);
        osc.start(t);
        osc.stop(t + 0.12);
      });
    }
  }
  ```

- [ ] **Step 2: Call `playEatSound()` at the top of `handleFoodEaten()`**

  `handleFoodEaten()` currently starts with `state.score += ...`. Add `playEatSound(state.stageIdx)` as the **first** line:

  ```js
  function handleFoodEaten() {
    playEatSound(state.stageIdx);  // ← ADD THIS LINE
    state.score += (state.stageIdx + 1) * 10;
    // ... rest unchanged
  }
  ```

- [ ] **Step 3: Call `playSound('death')` in `handleCollision()` and `stopMusic()` on game over path**

  In `handleCollision()`, `state.lives -= 1` is the second line. Add `playSound('death')` **after** it. Also, the game over path (`state.lives <= 0`) calls `showGameOver()` which already calls `stopMusic()` — no change needed there.

  Find:
  ```js
  function handleCollision() {
    stopLoop();
    state.lives -= 1;
    state.flashing = true;
  ```

  Replace with:
  ```js
  function handleCollision() {
    stopLoop();
    state.lives -= 1;
    playSound('death');  // ← ADD: plays over background music during flash
    state.flashing = true;
  ```

- [ ] **Step 4: Call `stopMusic()` and `playSound('victory')` at the top of `showVictory()`**

  `showVictory()` currently starts with `state.screen = 'victory'`. Add two lines before it:

  ```js
  function showVictory() {
    stopMusic();           // ← ADD
    playSound('victory');  // ← ADD: 5-note arpeggio
    state.screen = 'victory';
    stopLoop();
    stopDecoLoop();
    // ... rest unchanged
  }
  ```

- [ ] **Step 5: Call `playSound('stageClear')` in `handleFoodEaten()` before `showStageClear()`**

  From Task 2 Step 6, the else branch now reads:
  ```js
  } else {
    stopMusic();
    showStageClear();
  }
  ```

  Add `playSound('stageClear')` between `stopMusic()` and `showStageClear()`:
  ```js
  } else {
    stopMusic();
    playSound('stageClear');  // ← ADD: 3-note jingle
    showStageClear();
  }
  ```

- [ ] **Step 6: Verify sound effects in browser**

  Open `snake.html`. Test each sound:
  - **Eat sound (Jungle):** Start game, let snake eat apple → should hear a quick ascending chirp
  - **Eat sound (Ocean):** Advance to Stage 2, eat fish → descending bubble pop
  - **Eat sound (Space):** Advance to Stage 3, eat star → sharp laser zap
  - **Death:** Hit a wall → low descending wah sound; music should continue during flash, then resume
  - **Stage clear:** Eat 10 foods in Stage 1 → 3-note ascending fanfare before the overlay
  - **Victory:** Complete all 3 stages → 5-note ascending arpeggio

  No console errors expected.

- [ ] **Step 7: Commit**

  ```bash
  cd /Users/builder/Projects/games
  git add snake.html
  git commit -m "feat(sound): add eat, death, stage-clear, and victory sound effects"
  ```

---

### Task 4: Add mute toggle button to HUD

**Files:**
- Modify: `snake.html`

**Context:** A mute button (🔊/🔇) sits in the HUD row as a 4th element. It uses `state.muted` to gate all sound. On mute: stops background music loop; on unmute: restarts current stage's music from beginning (only during 'play' screen). In-flight one-shot effects always play to completion — no cancellation needed since we only use `state.muted` flag + `clearTimeout(musicTimeout)`.

- [ ] **Step 1: Add `#mute-btn` CSS**

  In the `<style>` block, after `#hud-lives { flex: 1; text-align: right; }`, add:

  ```css
  #mute-btn {
    background: none;
    border: none;
    font-size: 18px;
    cursor: pointer;
    padding: 0 4px;
    color: #fff;
    font-family: inherit;
    line-height: 1;
    flex-shrink: 0;
  }
  ```

- [ ] **Step 2: Add mute button to HUD HTML**

  In the `<div id="hud">`, after `<span id="hud-lives">❤️❤️❤️</span>`, add:

  ```html
  <button id="mute-btn">🔊</button>
  ```

  The HUD now has 4 elements: score (left), stage (center), lives (right), mute button.

- [ ] **Step 3: Add `toggleMute()` function**

  In the `// 10. Audio system` section, after `playSound()`, add:

  ```js
  function toggleMute() {
    state.muted = !state.muted;
    document.getElementById('mute-btn').textContent = state.muted ? '🔇' : '🔊';
    if (state.muted) {
      stopMusic();
    } else {
      // clearTimeout first to prevent double-start if unmuting during 2s stage-clear window
      clearTimeout(musicTimeout);
      if (state.screen === 'play') startMusic(state.stageIdx);
    }
  }
  ```

- [ ] **Step 4: Wire mute button click**

  Add the event listener in the `// 9. Initial setup` section, alongside `drawStartScreen()` and `startDecoLoop()`:

  ```js
  document.getElementById('mute-btn').addEventListener('click', toggleMute);
  ```

- [ ] **Step 4b: Reset mute button label in `fullReset()`**

  `fullReset()` calls `initState()` which sets `state.muted = false`, but the `#mute-btn` DOM element keeps its last label (`🔇` if the player muted before clicking Try Again / Play Again). Add a label reset in `fullReset()` after `initState()`:

  ```js
  function fullReset() {
    if (state.rafId) { cancelAnimationFrame(state.rafId); state.rafId = null; }
    stopLoop();
    stopMusic();
    clearOverlayBtn();
    initState();
    document.getElementById('mute-btn').textContent = '🔊';  // ← ADD: sync label with reset muted=false
    state.resetCount = (state.resetCount || 0) + 1;
    // ... rest unchanged
  }
  ```

- [ ] **Step 5: Verify mute toggle in browser**

  Open `snake.html`. Test:
  - Press arrow key to start — Jungle music plays, button shows 🔊
  - Click 🔊 → music stops immediately, button shows 🔇
  - Eat food — no eat sound (muted)
  - Click 🔇 → music restarts from beginning of Jungle phrase, button shows 🔊
  - Mute during stage clear 2s window (mute → music stops), then unmute → music should restart for new stage only if you're in the 2s wait period and then the stage starts
  - Game over → click "Try Again" → music should restart at 🔊 state (muted resets to false on fullReset)
  - No console errors

- [ ] **Step 6: Final browser smoke test — full game run**

  Play through the full game:
  1. Start screen → press arrow key
  2. Play Stage 1 — Jungle square wave music playing
  3. Eat apple → chirp sound
  4. Hit wall → death sound, music continues, respawn, music still playing
  5. Eat 10 apples → stage clear fanfare, music stops, 2s delay, Ocean music starts
  6. Play Stage 2 — Ocean sine wave music
  7. Eat fish → bubble pop sound
  8. Eat 10 fish → stage clear fanfare, 2s delay, Space music starts
  9. Play Stage 3 — Space sawtooth music
  10. Eat star → laser zap sound
  11. Eat 10 stars → victory arpeggio, confetti, music stops
  12. Click "Play Again" → Jungle music restarts immediately

  Verify no console errors throughout.

- [ ] **Step 7: Commit**

  ```bash
  cd /Users/builder/Projects/games
  git add snake.html
  git commit -m "feat(sound): add mute toggle button to HUD"
  ```

---

## Implementation Complete

After all tasks are committed, the sound system is fully implemented:
- ✅ Per-stage looping background music (Jungle/Ocean/Space themes)
- ✅ Per-stage eat sound effects (chirp/bubble/laser)
- ✅ Death sound over music during flash
- ✅ Stage clear fanfare + victory arpeggio
- ✅ Mute toggle (🔊/🔇) in HUD
- ✅ `musicTimeout` cancellability prevents race conditions on rapid reset
- ✅ No external audio files — all synthesized via Web Audio API
