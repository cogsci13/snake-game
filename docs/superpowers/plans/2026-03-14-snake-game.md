# Snake Game Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single self-contained `snake.html` kids' snake game with 3 themed stages, 3 lives, cute visuals, and a scoring system.

**Architecture:** All game code (HTML + CSS + JS) lives inline in `snake.html`. The game is driven by a `setInterval` loop. Game state is a plain JS object. Rendering is done entirely on an HTML5 Canvas using the 2D context. The HUD (score, lives, food counter) is HTML above the canvas.

**Tech Stack:** Vanilla HTML5, CSS3, JavaScript (ES6+). Google Fonts `Fredoka One` via CDN. No libraries, no build step.

**Spec:** `docs/superpowers/specs/2026-03-14-snake-game-design.md`

---

## Chunk 1: Foundation

### Task 1: HTML Skeleton + CSS Layout

**Files:**
- Create: `snake.html`

- [ ] **Step 1: Create the HTML skeleton**

Write `snake.html` with this structure:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>🐍 Snake!</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link href="https://fonts.googleapis.com/css2?family=Fredoka+One&display=swap" rel="stylesheet">
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }

    body {
      background: #111;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      min-height: 100vh;
      font-family: 'Fredoka One', 'Comic Sans MS', cursive;
      color: #fff;
      user-select: none;
    }

    #game-wrapper {
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 8px;
    }

    #hud {
      width: 480px;
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 6px 10px;
      background: rgba(255,255,255,0.1);
      border-radius: 12px;
      font-size: 18px;
    }

    #hud-score { flex: 1; text-align: left; }
    #hud-stage { flex: 1; text-align: center; }
    #hud-lives { flex: 1; text-align: right; }

    #food-counter {
      font-size: 16px;
      opacity: 0.85;
      letter-spacing: 1px;
    }

    #canvas-container {
      position: relative;
      display: inline-block;
    }

    #canvas {
      display: block;
      border-radius: 8px;
    }

    #overlay-btn-container {
      position: absolute;
      bottom: -60px;
      left: 50%;
      transform: translateX(-50%);
      white-space: nowrap;
    }

    .overlay-btn {
      padding: 12px 32px;
      font-family: 'Fredoka One', 'Comic Sans MS', cursive;
      font-size: 22px;
      border: none;
      border-radius: 50px;
      background: #ffd166;
      color: #333;
      cursor: pointer;
      transition: transform 0.1s;
    }
    .overlay-btn:hover { transform: scale(1.07); }
    .overlay-btn:active { transform: scale(0.96); }
  </style>
</head>
<body>
  <div id="game-wrapper">
    <div id="hud">
      <span id="hud-score">Score: 0</span>
      <span id="hud-stage">🌿 Jungle</span>
      <span id="hud-lives">❤️❤️❤️</span>
    </div>
    <div id="food-counter">🍎 0 / 10</div>
    <div id="canvas-container">
      <canvas id="canvas" width="480" height="480"></canvas>
      <div id="overlay-btn-container"></div>
    </div>
  </div>
  <script>
    // Game code goes here
  </script>
</body>
</html>
```

- [ ] **Step 2: Verify layout in browser**

Open `snake.html` in a browser. You should see:
- Dark background, centered layout
- HUD bar (480px wide) with Score / Stage / Lives
- Food counter text below HUD
- Black canvas (480×480px)

- [ ] **Step 3: Init git and commit**

```bash
cd /Users/builder/Projects/games
git init
git add snake.html docs/
git commit -m "chore: initial project setup with HTML skeleton"
```

---

### Task 2: Game Constants + State

**Files:**
- Modify: `snake.html` (inside `<script>`)

- [ ] **Step 1: Add constants and stage config**

Inside `<script>`, add:

```js
const CELL = 24;
const COLS = 20;
const ROWS = 20;
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');

const STAGES = [
  { name: '🌿 Jungle', bg: '#1a472a', snake: '#6bcb77', food: '#ff4d4d', foodEmoji: '🍎', speed: 200, borderStyle: 'leaf' },
  { name: '🌊 Ocean',  bg: '#0d3b6e', snake: '#00b4d8', food: '#ffd166', foodEmoji: '🐟', speed: 150, borderStyle: 'wave' },
  { name: '🚀 Space',  bg: '#0d0221', snake: '#c77dff', food: '#ffd700', foodEmoji: '⭐', speed: 100, borderStyle: 'stars' },
];

const FOOD_PER_STAGE = 10;
const DIRS = {
  ArrowUp:    { x: 0, y: -1 },
  ArrowDown:  { x: 0, y:  1 },
  ArrowLeft:  { x: -1, y: 0 },
  ArrowRight: { x:  1, y: 0 },
};
const OPPOSITE = {
  ArrowUp: 'ArrowDown', ArrowDown: 'ArrowUp',
  ArrowLeft: 'ArrowRight', ArrowRight: 'ArrowLeft',
};
```

- [ ] **Step 2: Add game state object and init function**

```js
let state = {};

function createSnake() {
  return [
    { x: 10, y: 10 },
    { x:  9, y: 10 },
    { x:  8, y: 10 },
  ];
}

function spawnFood(snake) {
  const occupied = new Set(snake.map(s => `${s.x},${s.y}`));
  let cell, attempts = 0;
  do {
    cell = {
      x: Math.floor(Math.random() * COLS),
      y: Math.floor(Math.random() * ROWS),
    };
    attempts++;
  } while (occupied.has(`${cell.x},${cell.y}`) && attempts < 400);
  // If board is nearly full (extremely unlikely with 400-cell grid), returns
  // last attempted cell. Game remains playable.
  return cell;
}

function initState() {
  const snake = createSnake();
  state = {
    screen: 'start',
    stageIdx: 0,
    score: 0,
    lives: 3,
    snake,
    dir: { x: 1, y: 0 },
    nextDir: { x: 1, y: 0 },
    currentDirKey: 'ArrowRight',
    food: spawnFood(snake),
    foodEaten: 0,
    loopId: null,
    rafId: null,
    flashing: false,
  };
}

initState();
```

- [ ] **Step 3: Verify in browser (no crash)**

Open DevTools console → no errors. `state` is accessible.

- [ ] **Step 4: Commit**

```bash
git add snake.html
git commit -m "feat: add game constants, stage config, and state init"
```

---

### Task 3: Basic Canvas Rendering

**Files:**
- Modify: `snake.html`

- [ ] **Step 1: Add helper + basic draw functions**

```js
function lighten(hex, amount) {
  const num = parseInt(hex.slice(1), 16);
  const r = Math.min(255, (num >> 16) + amount);
  const g = Math.min(255, ((num >> 8) & 0xff) + amount);
  const b = Math.min(255, (num & 0xff) + amount);
  return `rgb(${r},${g},${b})`;
}

function drawBackground() {
  ctx.fillStyle = STAGES[state.stageIdx].bg;
  ctx.fillRect(0, 0, canvas.width, canvas.height);
}

function drawSnake() {
  const stage = STAGES[state.stageIdx];
  state.snake.forEach((seg, i) => {
    const isHead = i === 0;
    const isTail = i === state.snake.length - 1;
    const size = isTail ? Math.round(CELL * 0.7) : CELL - 6;
    const offset = (CELL - size) / 2;
    ctx.fillStyle = isHead ? lighten(stage.snake, 20) : stage.snake;
    ctx.fillRect(seg.x * CELL + offset, seg.y * CELL + offset, size, size);
  });
}

function drawFood() {
  const stage = STAGES[state.stageIdx];
  const cx = state.food.x * CELL + CELL / 2;
  const cy = state.food.y * CELL + CELL / 2;
  ctx.shadowBlur = 10;
  ctx.shadowColor = stage.food;
  ctx.fillStyle = stage.food;
  ctx.beginPath();
  ctx.arc(cx, cy, 9, 0, Math.PI * 2);
  ctx.fill();
  ctx.shadowBlur = 0;
}

function render() {
  drawBackground();
  drawSnake();
  drawFood();
}

render();
```

- [ ] **Step 2: Verify in browser**

Canvas should show green background, a 3-segment snake near center, a red dot somewhere.

- [ ] **Step 3: Commit**

```bash
git add snake.html
git commit -m "feat: add basic canvas rendering"
```

---

### Task 4: Game Loop + Input + Movement

**Files:**
- Modify: `snake.html`

- [ ] **Step 1: Add input handler**

```js
document.addEventListener('keydown', (e) => {
  if (state.screen === 'start' && DIRS[e.key]) {
    e.preventDefault();
    startGame();
    return;
  }
  if (state.screen !== 'play' || state.flashing) return;
  if (!DIRS[e.key]) return;
  e.preventDefault();
  if (e.key === OPPOSITE[state.currentDirKey]) return;
  state.nextDir = DIRS[e.key];
  state.currentDirKey = e.key;
});
```

- [ ] **Step 2: Add tick + movement + game flow functions**

```js
function tick() {
  state.dir = state.nextDir;
  const head = state.snake[0];
  const newHead = { x: head.x + state.dir.x, y: head.y + state.dir.y };

  if (newHead.x < 0 || newHead.x >= COLS || newHead.y < 0 || newHead.y >= ROWS) {
    handleCollision(); return;
  }
  if (state.snake.some(s => s.x === newHead.x && s.y === newHead.y)) {
    handleCollision(); return;
  }

  const ateFood = newHead.x === state.food.x && newHead.y === state.food.y;
  state.snake.unshift(newHead);
  if (!ateFood) {
    state.snake.pop();
  } else {
    handleFoodEaten();
  }
  render();
}

function handleFoodEaten() {
  state.score += (state.stageIdx + 1) * 10;
  state.foodEaten += 1;
  updateHUD();
  if (state.foodEaten >= FOOD_PER_STAGE) {
    stopLoop();
    if (state.stageIdx === STAGES.length - 1) showVictory();
    else showStageClear();
  } else {
    state.food = spawnFood(state.snake);
  }
}

function startLoop() {
  state.loopId = setInterval(tick, STAGES[state.stageIdx].speed);
}

function stopLoop() {
  if (state.loopId) { clearInterval(state.loopId); state.loopId = null; }
}

function updateHUD() {
  const stage = STAGES[state.stageIdx];
  document.getElementById('hud-score').textContent = `Score: ${state.score}`;
  document.getElementById('hud-stage').textContent = stage.name;
  document.getElementById('hud-lives').textContent = '❤️'.repeat(state.lives);
  document.getElementById('food-counter').textContent =
    `${stage.foodEmoji} ${state.foodEaten} / ${FOOD_PER_STAGE}`;
}

function startGame() {
  state.screen = 'play';
  stopDecoLoop();
  document.getElementById('hud').style.visibility = 'visible';
  document.getElementById('food-counter').style.visibility = 'visible';
  updateHUD();
  startLoop();
}

// Stubs — implemented fully in later tasks
function handleCollision() {}
function showStageClear() {}
function showVictory() {}
function showGameOver() {}
```

- [ ] **Step 3: Verify movement in browser**

Press an arrow key → snake moves. Reaching a wall does nothing yet (stub).

- [ ] **Step 4: Commit**

```bash
git add snake.html
git commit -m "feat: add game loop, arrow key input, and snake movement"
```

---

### Task 5: Collision Detection + Lives System

**Files:**
- Modify: `snake.html` — replace stubs

- [ ] **Step 1: Implement handleCollision**

```js
function clearOverlayBtn() {
  const container = document.getElementById('overlay-btn-container');
  while (container.firstChild) container.removeChild(container.firstChild);
}

function showButton(id, label, onClick) {
  clearOverlayBtn();
  const btn = document.createElement('button');
  btn.id = id;
  btn.className = 'overlay-btn';
  btn.textContent = label;
  btn.addEventListener('click', () => { clearOverlayBtn(); onClick(); });
  document.getElementById('overlay-btn-container').appendChild(btn);
}

function fullReset() {
  if (state.rafId) { cancelAnimationFrame(state.rafId); state.rafId = null; }
  stopLoop();
  clearOverlayBtn();
  initState();              // sets screen:'start' — overridden on next line
  state.screen = 'play';   // skip start screen after first play
  document.getElementById('hud').style.visibility = 'visible';
  document.getElementById('food-counter').style.visibility = 'visible';
  updateHUD();
  render();
  startLoop();
}

function handleCollision() {
  stopLoop();               // clear setInterval — no ghost ticks
  state.lives -= 1;
  state.flashing = true;    // blocks keydown input during flash

  ctx.fillStyle = 'rgba(255,0,0,0.45)';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  setTimeout(() => {
    // Guard: if fullReset() ran before this timeout fired, do nothing
    if (state.screen !== 'play') return;
    state.flashing = false; // re-enable input
    if (state.lives <= 0) {
      showGameOver();
    } else {
      // Reset snake to starting position; food position is retained
      state.snake = createSnake();            // [{x:10,y:10},{x:9,y:10},{x:8,y:10}]
      state.dir = { x: 1, y: 0 };
      state.nextDir = { x: 1, y: 0 };
      state.currentDirKey = 'ArrowRight';
      updateHUD();
      render();
      startLoop();          // fresh interval with current stage speed
    }
  }, 500);
}
```

- [ ] **Step 2: Implement showGameOver**

```js
function showGameOver() {
  state.screen = 'gameover';
  render();
  ctx.fillStyle = 'rgba(0,0,0,0.72)';
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  ctx.textAlign = 'center';
  ctx.shadowBlur = 0;
  ctx.fillStyle = '#ff6b6b';
  ctx.font = '48px "Fredoka One", cursive';
  ctx.fillText('Game Over 💀', canvas.width / 2, canvas.height / 2 - 40);
  ctx.fillStyle = '#fff';
  ctx.font = '28px "Fredoka One", cursive';
  ctx.fillText(`Final Score: ${state.score}`, canvas.width / 2, canvas.height / 2 + 20);
  showButton('tryAgainBtn', 'Try Again 🔄', () => fullReset());
}
```

- [ ] **Step 3: Implement showStageClear**

```js
function showStageClear() {
  state.screen = 'stageclear';
  stopLoop(); // defensive — handleFoodEaten already called stopLoop, but prevent stacking
  render();
  ctx.fillStyle = 'rgba(0,0,0,0.65)';
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  ctx.textAlign = 'center';
  ctx.shadowBlur = 0;
  ctx.fillStyle = '#ffd166';
  ctx.font = '42px "Fredoka One", cursive';
  ctx.fillText('Stage Complete! 🎉', canvas.width / 2, canvas.height / 2 - 30);
  ctx.fillStyle = '#fff';
  ctx.font = '26px "Fredoka One", cursive';
  ctx.fillText(`Score: ${state.score}`, canvas.width / 2, canvas.height / 2 + 20);
  ctx.fillStyle = 'rgba(255,255,255,0.6)';
  ctx.font = '18px "Fredoka One", cursive';
  ctx.fillText('Get ready...', canvas.width / 2, canvas.height / 2 + 60);

  setTimeout(() => {
    // Guard: showStageClear is only called when stageIdx < STAGES.length - 1
    // (handleFoodEaten branches to showVictory for the last stage),
    // but cap here defensively.
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
}
```

- [ ] **Step 4: Implement showVictory (placeholder — full confetti in Task 10)**

```js
function showVictory() {
  state.screen = 'victory';
  render();
  ctx.fillStyle = 'rgba(0,0,0,0.72)';
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  ctx.textAlign = 'center';
  ctx.shadowBlur = 0;
  ctx.fillStyle = '#ffd700';
  ctx.font = '52px "Fredoka One", cursive';
  ctx.fillText('You Win! 🏆', canvas.width / 2, canvas.height / 2 - 40);
  ctx.fillStyle = '#fff';
  ctx.font = '28px "Fredoka One", cursive';
  ctx.fillText(`Final Score: ${state.score}`, canvas.width / 2, canvas.height / 2 + 20);
  showButton('playAgainBtn', 'Play Again ✨', () => fullReset());
}
```

- [ ] **Step 5: Verify full game loop in browser**

- Hit wall → red flash → snake resets → lives decrease in HUD
- Die 3 times → Game Over overlay + "Try Again" → game restarts
- Eat 10 food → Stage Clear → auto-advances to Stage 2
- Complete Stage 3 → Victory (placeholder)

- [ ] **Step 6: Commit**

```bash
git add snake.html
git commit -m "feat: add collision, lives system, and all overlay screens"
```

---

## Chunk 2: Visual Polish

### Task 6: Themed Backgrounds + Borders

**Files:**
- Modify: `snake.html` — expand `drawBackground()`

- [ ] **Step 1: Replace drawBackground with themed version**

```js
function drawBackground() {
  const stage = STAGES[state.stageIdx];
  ctx.fillStyle = stage.bg;
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  if (stage.borderStyle === 'leaf')  drawLeafBorder();
  if (stage.borderStyle === 'wave')  drawWaveBorder();
  if (stage.borderStyle === 'stars') drawStarsBorder();
}

function drawLeafBorder() {
  ctx.save();
  ctx.fillStyle = '#2d6a4f';
  for (let x = 0; x < COLS; x += 2) {
    drawLeaf(x * CELL + CELL / 2, 8);
    drawLeaf(x * CELL + CELL / 2, canvas.height - 8);
  }
  for (let y = 0; y < ROWS; y += 2) {
    drawLeaf(8, y * CELL + CELL / 2);
    drawLeaf(canvas.width - 8, y * CELL + CELL / 2);
  }
  ctx.restore();
}

function drawLeaf(x, y) {
  ctx.beginPath();
  ctx.ellipse(x, y, 10, 5, Math.PI / 4, 0, Math.PI * 2);
  ctx.fill();
}

function drawWaveBorder() {
  ctx.save();
  ctx.strokeStyle = '#48cae4';
  ctx.lineWidth = 3;
  ctx.setLineDash([8, 6]);
  // Top wave
  ctx.beginPath();
  for (let x = 0; x <= canvas.width; x += 12) {
    const y = 10 + Math.sin((x / canvas.width) * Math.PI * 4) * 4;
    x === 0 ? ctx.moveTo(x, y) : ctx.lineTo(x, y);
  }
  ctx.stroke();
  // Bottom wave
  ctx.beginPath();
  for (let x = 0; x <= canvas.width; x += 12) {
    const y = canvas.height - 10 + Math.sin((x / canvas.width) * Math.PI * 4) * 4;
    x === 0 ? ctx.moveTo(x, y) : ctx.lineTo(x, y);
  }
  ctx.stroke();
  ctx.setLineDash([]);
  ctx.restore();
}

function drawStarsBorder() {
  ctx.save();
  ctx.fillStyle = 'rgba(255,255,255,0.7)';
  const positions = [
    {x:20,y:20},{x:60,y:15},{x:110,y:25},{x:170,y:10},{x:240,y:20},
    {x:300,y:15},{x:360,y:22},{x:420,y:12},{x:470,y:20},
    {x:20,y:465},{x:80,y:470},{x:150,y:460},{x:220,y:468},
    {x:290,y:462},{x:350,y:470},{x:410,y:460},{x:460,y:468},
    {x:10,y:60},{x:15,y:130},{x:8,y:200},{x:14,y:270},{x:10,y:340},{x:16,y:410},
    {x:470,y:60},{x:465,y:140},{x:472,y:210},{x:468,y:280},{x:474,y:350},{x:470,y:420},
  ];
  positions.forEach(({ x, y }) => {
    ctx.beginPath();
    ctx.arc(x, y, 2, 0, Math.PI * 2);
    ctx.fill();
  });
  ctx.restore();
}
```

- [ ] **Step 2: Verify in browser**

- Stage 1: dark green bg + small leaf ellipses on edges ✓
- Stage 2: dark blue bg + dashed wave lines top/bottom ✓
- Stage 3: dark purple bg + white dot stars on edges ✓

- [ ] **Step 3: Commit**

```bash
git add snake.html
git commit -m "feat: add themed backgrounds with leaf, wave, and star borders"
```

---

### Task 7: Cute Snake Rendering

**Files:**
- Modify: `snake.html` — replace `drawSnake()`

- [ ] **Step 1: Add roundRect helper**

```js
function roundRect(x, y, w, h, r) {
  ctx.beginPath();
  ctx.moveTo(x + r, y);
  ctx.lineTo(x + w - r, y);
  ctx.quadraticCurveTo(x + w, y, x + w, y + r);
  ctx.lineTo(x + w, y + h - r);
  ctx.quadraticCurveTo(x + w, y + h, x + w - r, y + h);
  ctx.lineTo(x + r, y + h);
  ctx.quadraticCurveTo(x, y + h, x, y + h - r);
  ctx.lineTo(x, y + r);
  ctx.quadraticCurveTo(x, y, x + r, y);
  ctx.closePath();
}
```

- [ ] **Step 2: Replace drawSnake with cute version**

```js
function drawSnake() {
  const stage = STAGES[state.stageIdx];
  state.snake.forEach((seg, i) => {
    const isHead = i === 0;
    const isTail = i === state.snake.length - 1;
    // CELL-6 = 18px in a 24px cell → 6px gap between segments (intentional per spec)
    const size = isTail ? Math.round(CELL * 0.70) : CELL - 6;
    const offset = (CELL - size) / 2;
    const px = seg.x * CELL + offset;
    const py = seg.y * CELL + offset;
    const r = Math.round(size * 0.35);

    ctx.fillStyle = isHead ? lighten(stage.snake, 25) : stage.snake;
    roundRect(px, py, size, size, r);
    ctx.fill();

    if (isHead) drawSnakeEyes(px, py, size);
  });
}

function drawSnakeEyes(px, py, size) {
  ctx.fillStyle = '#fff';
  const d = state.dir;
  let e1, e2;
  if (d.x === 1) {
    e1 = { x: px + size - 7, y: py + 4 };
    e2 = { x: px + size - 7, y: py + size - 7 };
  } else if (d.x === -1) {
    e1 = { x: px + 4, y: py + 4 };
    e2 = { x: px + 4, y: py + size - 7 };
  } else if (d.y === -1) {
    e1 = { x: px + 4,        y: py + 4 };
    e2 = { x: px + size - 7, y: py + 4 };
  } else {
    e1 = { x: px + 4,        y: py + size - 7 };
    e2 = { x: px + size - 7, y: py + size - 7 };
  }
  ctx.beginPath(); ctx.arc(e1.x, e1.y, 3, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath(); ctx.arc(e2.x, e2.y, 3, 0, Math.PI * 2); ctx.fill();
}
```

- [ ] **Step 3: Verify in browser**

Segments: rounded rectangles with gaps. Head: slightly brighter, two white dot eyes that shift with direction. Tail: ~70% size.

- [ ] **Step 4: Commit**

```bash
git add snake.html
git commit -m "feat: cute rounded snake rendering with directional eyes"
```

---

### Task 8: Cute Food Rendering

**Files:**
- Modify: `snake.html` — replace `drawFood()`

- [ ] **Step 1: Replace drawFood with themed shapes**

```js
function drawFood() {
  const stage = STAGES[state.stageIdx];
  const cx = state.food.x * CELL + CELL / 2;
  const cy = state.food.y * CELL + CELL / 2;
  ctx.save();
  ctx.shadowBlur = 10;
  ctx.shadowColor = stage.food;
  ctx.fillStyle = stage.food;
  if (state.stageIdx === 0) drawApple(cx, cy);
  else if (state.stageIdx === 1) drawFish(cx, cy);
  else drawStar(cx, cy);
  ctx.restore();
}

function drawApple(cx, cy) {
  ctx.beginPath();
  ctx.arc(cx, cy + 1, 9, 0, Math.PI * 2);
  ctx.fill();
  ctx.shadowBlur = 0;
  ctx.strokeStyle = '#4caf50';
  ctx.lineWidth = 2;
  ctx.lineCap = 'round';
  ctx.beginPath();
  ctx.moveTo(cx, cy - 8);
  ctx.lineTo(cx, cy - 13);
  ctx.stroke();
}

function drawFish(cx, cy) {
  // Body ellipse (shifted right so tail has room)
  ctx.beginPath();
  ctx.ellipse(cx + 2, cy, 10, 6, 0, 0, Math.PI * 2);
  ctx.fill();
  // Tail: triangle pointing left — tip at body edge (cx-8), fins at (cx-16, cy±6)
  ctx.beginPath();
  ctx.moveTo(cx - 8, cy);        // tip (meets body)
  ctx.lineTo(cx - 16, cy - 6);  // top fin
  ctx.lineTo(cx - 16, cy + 6);  // bottom fin
  ctx.closePath();               // closes back to tip
  ctx.fill();
  // Eye — reset shadow so it doesn't glow
  ctx.shadowBlur = 0;
  ctx.fillStyle = '#333';
  ctx.beginPath();
  ctx.arc(cx + 7, cy - 1, 2, 0, Math.PI * 2);
  ctx.fill();
}

function drawStar(cx, cy) {
  const outer = 10, inner = 5, points = 5;
  ctx.beginPath();
  for (let i = 0; i < points * 2; i++) {
    const r = i % 2 === 0 ? outer : inner;
    const angle = (i * Math.PI) / points - Math.PI / 2;
    i === 0
      ? ctx.moveTo(cx + r * Math.cos(angle), cy + r * Math.sin(angle))
      : ctx.lineTo(cx + r * Math.cos(angle), cy + r * Math.sin(angle));
  }
  ctx.closePath();
  ctx.fill();
}
```

- [ ] **Step 2: Verify in browser**

- Stage 1: red circle + green stem (apple) with glow ✓
- Stage 2: yellow oval + tail triangle + dark eye (fish) with glow ✓
- Stage 3: gold 5-point star with glow ✓

- [ ] **Step 3: Commit**

```bash
git add snake.html
git commit -m "feat: add themed food shapes (apple, fish, star) with glow"
```

---

## Chunk 3: UI Screens + Final Polish

### Task 9: Start Screen + Decorative Snake Animation

**Files:**
- Modify: `snake.html`

- [ ] **Step 1: Build the decorative loop path**

Add this after the constants (before `initState`):

```js
// Decorative 4×4 cell loop path (12 cells, counterclockwise)
const DECO_LOOP = (() => {
  const sx = Math.round((COLS - 4) / 2); // column 8
  const sy = Math.round(60 / CELL);      // row 2
  const path = [];
  for (let x = sx; x < sx + 4; x++)      path.push({ x, y: sy });          // top L→R
  for (let y = sy + 1; y < sy + 4; y++)  path.push({ x: sx + 3, y });      // right T→B
  for (let x = sx + 2; x >= sx; x--)    path.push({ x, y: sy + 3 });      // bottom R→L
  for (let y = sy + 2; y > sy; y--)     path.push({ x: sx, y });          // left B→T
  return path; // 12 positions
})();

let decoFrame = 0;
let decoLoopId = null;
```

- [ ] **Step 2: Add drawStartScreen + loop control**

```js
function drawStartScreen() {
  ctx.fillStyle = STAGES[0].bg;
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  drawLeafBorder();

  // Decorative 5-segment snake — NOTE: parens required for correct modulo
  for (let i = 0; i < 5; i++) {
    const pos = DECO_LOOP[(decoFrame + i) % DECO_LOOP.length]; // NOT: decoFrame + i%12
    const size = CELL - 6;
    const off = (CELL - size) / 2;
    ctx.fillStyle = i === 0 ? lighten(STAGES[0].snake, 25) : STAGES[0].snake;
    roundRect(pos.x * CELL + off, pos.y * CELL + off, size, size, Math.round(size * 0.35));
    ctx.fill();
  }

  ctx.textAlign = 'center';
  ctx.shadowBlur = 0;
  ctx.fillStyle = '#ffd166';
  ctx.font = 'bold 56px "Fredoka One", cursive';
  ctx.fillText('🐍 Snake!', canvas.width / 2, 280);
  ctx.fillStyle = '#6bcb77';
  ctx.font = '28px "Fredoka One", cursive';
  ctx.fillText('🌿 Jungle', canvas.width / 2, 322);
  ctx.fillStyle = 'rgba(255,255,255,0.85)';
  ctx.font = '20px "Fredoka One", cursive';
  ctx.fillText('Press any arrow key to start', canvas.width / 2, 372);
}

function startDecoLoop() {
  decoFrame = 0;
  decoLoopId = setInterval(() => {
    // Use DECO_LOOP.length (not hardcoded 12) so draw and counter stay in sync
    decoFrame = (decoFrame + 1) % DECO_LOOP.length;
    if (state.screen === 'start') drawStartScreen();
    else stopDecoLoop();
  }, 200);
}

function stopDecoLoop() {
  if (decoLoopId) { clearInterval(decoLoopId); decoLoopId = null; }
}
```

- [ ] **Step 3: Wire up start screen on page load**

Replace the lone `render()` call at bottom of script with:

```js
// Initial setup
document.getElementById('hud').style.visibility = 'hidden';
document.getElementById('food-counter').style.visibility = 'hidden';
drawStartScreen();
startDecoLoop();
```

And in `startGame()` ensure `stopDecoLoop()` is called (already added in Task 4).

- [ ] **Step 4: Verify in browser**

- Page loads → Jungle green bg + title + subtitle + "Press any arrow key to start"
- Small 5-segment snake loops around rectangle near top of canvas
- Press arrow key → HUD appears, snake starts moving

- [ ] **Step 5: Commit**

```bash
git add snake.html
git commit -m "feat: add start screen with animated decorative snake loop"
```

---

### Task 10: Victory Screen with Confetti

**Files:**
- Modify: `snake.html` — replace stub `showVictory()`

- [ ] **Step 1: Replace showVictory with confetti version**

```js
const CONFETTI_COLORS = ['#ff6b6b','#ffd166','#06d6a0','#118ab2','#c77dff','#ff9f43','#48dbfb'];

function initConfetti() {
  return Array.from({ length: 80 }, () => ({
    x: Math.random() * canvas.width,
    y: -8 - Math.random() * canvas.height,
    speed: 2 + Math.random() * 2,
    drift: (Math.random() - 0.5) * 2,
    color: CONFETTI_COLORS[Math.floor(Math.random() * CONFETTI_COLORS.length)],
  }));
}

function showVictory() {
  state.screen = 'victory';
  stopLoop();
  stopDecoLoop(); // safety guard in case deco loop wasn't stopped earlier
  const particles = initConfetti();

  function victoryLoop() {
    if (state.screen !== 'victory') return;
    render();
    ctx.fillStyle = 'rgba(0,0,0,0.60)';
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    // Draw confetti
    particles.forEach(p => {
      ctx.fillStyle = p.color;
      ctx.fillRect(p.x, p.y, 8, 8);
      p.y += p.speed;
      p.x += p.drift;
      // Reset when off bottom OR drifted off left/right edge
      if (p.y > canvas.height || p.x < -8 || p.x > canvas.width + 8) {
        p.y = -8;
        p.x = Math.random() * canvas.width;
      }
    });

    ctx.textAlign = 'center';
    ctx.shadowBlur = 0;
    ctx.fillStyle = '#ffd700';
    ctx.font = '52px "Fredoka One", cursive';
    ctx.fillText('You Win! 🏆', canvas.width / 2, canvas.height / 2 - 50);
    ctx.fillStyle = '#fff';
    ctx.font = '28px "Fredoka One", cursive';
    ctx.fillText(`Final Score: ${state.score}`, canvas.width / 2, canvas.height / 2 + 10);
    ctx.fillStyle = 'rgba(255,255,255,0.7)';
    ctx.font = '20px "Fredoka One", cursive';
    ctx.fillText('Amazing! 🎊', canvas.width / 2, canvas.height / 2 + 48);

    state.rafId = requestAnimationFrame(victoryLoop);
  }
  state.rafId = requestAnimationFrame(victoryLoop);
  showButton('playAgainBtn', 'Play Again ✨', () => fullReset());
}
```

- [ ] **Step 2: Verify in browser**

Complete all 3 stages → Victory screen shows:
- Gold "You Win! 🏆" title
- Final score
- "Amazing! 🎊" subtitle
- Colorful confetti squares falling and looping
- "Play Again ✨" button → resets to Stage 1 gameplay immediately

- [ ] **Step 3: Commit**

```bash
git add snake.html
git commit -m "feat: victory screen with looping confetti animation"
```

---

### Task 11: Final Verification Checklist

**Files:**
- Read-only pass (no code changes unless a bug is found)

- [ ] **Snake rendering**
  - Segments: rounded rectangles with visible gaps ✓
  - Head: brighter color, two white dot eyes that update with direction ✓
  - Tail: visibly smaller (~70%) ✓

- [ ] **Food rendering**
  - Stage 1: red circle + green stem + glow ✓
  - Stage 2: yellow ellipse + fin + eye + glow ✓
  - Stage 3: gold 5-point star + glow ✓

- [ ] **Themes**
  - Stage 1: dark green + leaf shapes on border ✓
  - Stage 2: dark blue + dashed wave lines top/bottom ✓
  - Stage 3: dark purple + white dot stars on border ✓

- [ ] **HUD**
  - Score increments (+10/+20/+30) on each food eaten ✓
  - Stage name updates when stage changes ✓
  - Hearts (❤️) decrease on life loss ✓
  - Food counter emoji matches current stage ✓

- [ ] **Game flows**
  - Start screen → arrow key → play ✓
  - Wall collision → red flash → lives−1 → respawn ✓
  - 3 lives lost → Game Over → Try Again → play ✓
  - 10 food eaten → Stage Clear (2s) → next stage ✓
  - Stage 3 complete → Victory + confetti → Play Again → play ✓

- [ ] **Final commit**

```bash
git add snake.html
git commit -m "chore: complete kids snake game - all stages, lives, screens implemented"
```

---

## Summary

Single file: `snake.html`. Open in any browser. No server or build step needed.

**Max score:** Stage 1 (10×10=100) + Stage 2 (10×20=200) + Stage 3 (10×30=300) = **600 points**
