# Snake Stages 4 & 5 Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Cookie Land (stage 4) and Doll Land (stage 5) to the snake game, each with a wandering dog obstacle that costs a life on contact.

**Architecture:** All changes are in a single file (`snake.html`). Two new entries are appended to the `STAGES` array. A `dog` property is added to `state` (`null` for stages 1–3, `{x,y}` for 4–5). Dog movement is driven by a `tickCount` counter inside the existing `setInterval` tick — every 3 ticks, `moveDog()` shuffles the 4 directions and moves to the first valid one.

**Tech Stack:** Vanilla JS, Canvas 2D API, Web Audio API — no external dependencies.

**Spec:** `docs/superpowers/specs/2026-03-17-snake-stages-4-5-design.md`

---

> **Chunk order matters.** Chunks 1–2 add data/helpers; chunks 3–4 wire them in. After chunk 2, the dog feature is defined but inert (dog never spawns or renders). It becomes fully active only after chunk 4.

## Chunk 1: Data & State Foundation

### Task 1: Add `hasDog` to all stages and append stages 4–5

**Files:**
- Modify: `snake.html` (STAGES array, lines ~111–115)

- [ ] **Step 1.1: Add `hasDog: false` to existing three stage objects**

In `snake.html`, find the `STAGES` array. Add `hasDog: false` to each of the three existing entries:

```js
const STAGES = [
  { name: '🌿 Jungle', bg: '#1a472a', snake: '#6bcb77', food: '#ff4d4d', foodEmoji: '🍎', speed: 200, borderStyle: 'leaf',  hasDog: false },
  { name: '🌊 Ocean',  bg: '#0d3b6e', snake: '#00b4d8', food: '#ffd166', foodEmoji: '🐟', speed: 150, borderStyle: 'wave',  hasDog: false },
  { name: '🚀 Space',  bg: '#0d0221', snake: '#c77dff', food: '#ffd700', foodEmoji: '⭐', speed: 100, borderStyle: 'stars', hasDog: false },
  { name: '🍪 Cookie Land', bg: '#3d1a00', snake: '#ff9a3c', food: '#ff6eb4', foodEmoji: '🍬', speed: 100, borderStyle: 'cookies', hasDog: true },
  { name: '🧸 Doll Land',   bg: '#2d1b4e', snake: '#ffb3d9', food: '#cd853f', foodEmoji: '🧸', speed: 100, borderStyle: 'hearts',  hasDog: true },
];
```

- [ ] **Step 1.2: Verify stage count in browser console**

Open `snake.html` in browser, open DevTools console, run:
```js
console.log(STAGES.length); // expect: 5
console.log(STAGES[3].name); // expect: '🍪 Cookie Land'
console.log(STAGES[4].name); // expect: '🧸 Doll Land'
```

- [ ] **Step 1.3: Commit**

```bash
git add snake.html
git commit -m "feat(stages): add Cookie Land and Doll Land stage configs"
```

---

### Task 2: Add `dog` and `tickCount` to state

**Files:**
- Modify: `snake.html` (`initState` function, lines ~170–189)

- [ ] **Step 2.1: Add `dog` and `tickCount` to the `state` object in `initState()`**

Find `initState()`. Add `dog: null` and `tickCount: 0` to the returned `state` object:

```js
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
    food: spawnFood(snake, null),
    foodEaten: 0,
    loopId: null,
    rafId: null,
    flashing: false,
    muted: false,
    resetCount: 0,
    dog: null,
    tickCount: 0,
  };
}
```

Note: `spawnFood(snake)` is updated to `spawnFood(snake, null)` here (the function signature change happens in Task 3; add the `null` argument now so it's ready).

- [ ] **Step 2.2: Verify in browser console**

```js
// After page load, open console:
console.log(state.dog);       // expect: null
console.log(state.tickCount); // expect: 0
```

- [ ] **Step 2.3: Commit**

```bash
git add snake.html
git commit -m "feat(state): add dog and tickCount fields to game state"
```

---

## Chunk 2: Dog Logic Functions

### Task 3: `spawnDog`, `moveDog`, update `spawnFood` signature

**Files:**
- Modify: `snake.html` (`spawnFood` function ~line 157, add new functions after it)

- [ ] **Step 3.1: Update `spawnFood` signature to accept a `dog` parameter**

Find `spawnFood(snake)` and replace the function:

```js
function spawnFood(snake, dog) {
  const occupied = new Set(snake.map(s => `${s.x},${s.y}`));
  if (dog) occupied.add(`${dog.x},${dog.y}`);
  let cell, attempts = 0;
  do {
    cell = {
      x: Math.floor(Math.random() * COLS),
      y: Math.floor(Math.random() * ROWS),
    };
    attempts++;
  } while (occupied.has(`${cell.x},${cell.y}`) && attempts < 400);
  return cell;
}
```

- [ ] **Step 3.2: Add `spawnDog` function after `spawnFood`**

```js
function spawnDog(snake, food) {
  const occupied = new Set(snake.map(s => `${s.x},${s.y}`));
  occupied.add(`${food.x},${food.y}`);
  let cell, attempts = 0;
  do {
    cell = {
      x: Math.floor(Math.random() * COLS),
      y: Math.floor(Math.random() * ROWS),
    };
    attempts++;
  } while (occupied.has(`${cell.x},${cell.y}`) && attempts < 400);
  return cell; // falls back to last random cell after 400 attempts (grid never full in practice)
}
```

- [ ] **Step 3.3: Add `moveDog` function after `spawnDog`**

```js
function moveDog() {
  const dirs = [{ x: 0, y: -1 }, { x: 0, y: 1 }, { x: -1, y: 0 }, { x: 1, y: 0 }];
  // Fisher-Yates shuffle
  for (let i = dirs.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [dirs[i], dirs[j]] = [dirs[j], dirs[i]];
  }
  for (const d of dirs) {
    const nx = state.dog.x + d.x;
    const ny = state.dog.y + d.y;
    if (nx >= 0 && nx < COLS && ny >= 0 && ny < ROWS) {
      state.dog = { x: nx, y: ny };
      return;
    }
  }
  // All directions hit wall — stay in place (cannot happen on 20×20 grid)
}
```

- [ ] **Step 3.4: Verify in browser console**

```js
// Simulate spawnDog:
const testSnake = [{ x: 10, y: 10 }, { x: 9, y: 10 }];
const testFood  = { x: 5, y: 5 };
const dog = spawnDog(testSnake, testFood);
console.log(dog); // expect: { x: <number 0-19>, y: <number 0-19> }, not (10,10), (9,10), or (5,5)

// Simulate moveDog:
state.dog = { x: 10, y: 10 };
moveDog();
console.log(state.dog); // expect: adjacent cell (9,10), (11,10), (10,9), or (10,11)

// Verify spawnFood still works with dog arg:
const food = spawnFood(testSnake, dog);
console.log(food); // expect: not on dog position or snake cells
```

- [ ] **Step 3.5: Commit**

```bash
git add snake.html
git commit -m "feat(dog): add spawnDog, moveDog; update spawnFood to exclude dog cell"
```

---

### Task 4: `drawDog`

**Files:**
- Modify: `snake.html` (add after `drawStar` function, ~line 384)

- [ ] **Step 4.1: Add `drawDog(gx, gy)` function**

Add after the `drawStar` function:

```js
function drawDog(gx, gy) {
  const px = gx * CELL;
  const py = gy * CELL;
  const cx = px + CELL / 2;
  const cy = py + CELL / 2;

  ctx.save();

  // Ears (draw behind head)
  ctx.fillStyle = '#8b6914';
  ctx.beginPath(); ctx.arc(cx - 6, cy - 7, 5, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath(); ctx.arc(cx + 6, cy - 7, 5, 0, Math.PI * 2); ctx.fill();

  // Head
  ctx.fillStyle = '#d4a55a';
  ctx.beginPath(); ctx.arc(cx, cy + 1, 9, 0, Math.PI * 2); ctx.fill();

  // Eyes
  ctx.fillStyle = '#333';
  ctx.beginPath(); ctx.arc(cx - 3, cy - 1, 2, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath(); ctx.arc(cx + 3, cy - 1, 2, 0, Math.PI * 2); ctx.fill();

  // Nose
  ctx.beginPath(); ctx.arc(cx, cy + 3, 2, 0, Math.PI * 2); ctx.fill();

  ctx.restore();
}
```

- [ ] **Step 4.2: Quick visual test in browser console**

```js
// Temporarily call drawDog to verify it renders correctly:
drawDog(5, 5);
// Look at canvas: expect a small tan circle (head) with brown ears, two dark eyes, and a dark nose, in cell (5,5).
```

- [ ] **Step 4.3: Commit**

```bash
git add snake.html
git commit -m "feat(dog): add drawDog canvas rendering function"
```

---

## Chunk 3: Visual Rendering

### Task 5: New food shapes + update `drawFood`

**Files:**
- Modify: `snake.html` (add `drawCandy` and `drawTeddyBear` after `drawStar`; update `drawFood`)

- [ ] **Step 5.1: Add `drawCandy(cx, cy)` function** (add after `drawStar`)

```js
function drawCandy(cx, cy) {
  // Stick (dark pink)
  ctx.fillStyle = '#cc3380';
  ctx.fillRect(cx - 2, cy - 18, 4, 10);

  // Body (hot pink — must set fillStyle again; ctx.fillStyle is still '#cc3380' from stick)
  ctx.fillStyle = '#ff6eb4';
  ctx.beginPath();
  ctx.arc(cx, cy + 2, 9, 0, Math.PI * 2);
  ctx.fill();
}
```

- [ ] **Step 5.2: Add `drawTeddyBear(cx, cy)` function** (add after `drawCandy`)

```js
function drawTeddyBear(cx, cy) {
  // Ears (behind head)
  ctx.fillStyle = '#b8732a';
  ctx.beginPath(); ctx.arc(cx - 8, cy - 7, 4, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath(); ctx.arc(cx + 8, cy - 7, 4, 0, Math.PI * 2); ctx.fill();

  // Head
  ctx.fillStyle = '#cd853f';
  ctx.beginPath(); ctx.arc(cx, cy + 1, 8, 0, Math.PI * 2); ctx.fill();

  // Eyes
  ctx.fillStyle = '#333';
  ctx.beginPath(); ctx.arc(cx - 3, cy, 2, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath(); ctx.arc(cx + 3, cy, 2, 0, Math.PI * 2); ctx.fill();

  // Nose
  ctx.beginPath(); ctx.arc(cx, cy + 4, 2, 0, Math.PI * 2); ctx.fill();
}
```

- [ ] **Step 5.3: Update `drawFood()` to branch for stages 4 and 5**

Find `drawFood()` and replace the if/else chain:

```js
function drawFood() {
  const stage = STAGES[state.stageIdx];
  const cx = state.food.x * CELL + CELL / 2;
  const cy = state.food.y * CELL + CELL / 2;
  ctx.save();
  ctx.shadowBlur = 10;
  ctx.shadowColor = stage.food;
  ctx.fillStyle = stage.food;
  if (state.stageIdx === 0)      drawApple(cx, cy);
  else if (state.stageIdx === 1) drawFish(cx, cy);
  else if (state.stageIdx === 2) drawStar(cx, cy);
  else if (state.stageIdx === 3) drawCandy(cx, cy);
  else                           drawTeddyBear(cx, cy);
  ctx.restore();
}
```

- [ ] **Step 5.4: Visual test in browser console**

```js
// Test candy:
state.stageIdx = 3;
ctx.fillStyle = '#ff6eb4';
drawCandy(240, 240);
// Expect: pink circle with a darker stick above it

// Test teddy bear:
state.stageIdx = 4;
ctx.fillStyle = '#cd853f';
drawTeddyBear(240, 240);
// Expect: brown circle head with two smaller ear circles, tiny eyes and nose
```

- [ ] **Step 5.5: Commit**

```bash
git add snake.html
git commit -m "feat(visual): add drawCandy and drawTeddyBear food shapes for stages 4-5"
```

---

### Task 6: New border styles + update `drawBackground`

**Files:**
- Modify: `snake.html` (add `drawCookieBorder` and `drawHeartsBorder`; update `drawBackground`)

- [ ] **Step 6.1: Add `drawCookieBorder()` function** (add after `drawStarsBorder`)

```js
function drawCookieBorder() {
  ctx.save();
  ctx.fillStyle = '#a0522d';
  for (let x = 0; x < canvas.width; x += 24) {
    ctx.beginPath(); ctx.arc(x + 12, 10, 5, 0, Math.PI * 2); ctx.fill();
    ctx.beginPath(); ctx.arc(x + 12, 470, 5, 0, Math.PI * 2); ctx.fill();
  }
  for (let y = 24; y < canvas.height - 24; y += 24) {
    ctx.beginPath(); ctx.arc(10, y + 12, 5, 0, Math.PI * 2); ctx.fill();
    ctx.beginPath(); ctx.arc(470, y + 12, 5, 0, Math.PI * 2); ctx.fill();
  }
  ctx.restore();
}
```

- [ ] **Step 6.2: Add `drawHeartsBorder()` function** (add after `drawCookieBorder`)

```js
function drawHeartsBorder() {
  ctx.save();
  ctx.fillStyle = '#ff69b4';

  function drawHeart(hx, hy) {
    ctx.beginPath();
    ctx.moveTo(hx, hy + 3);
    ctx.bezierCurveTo(hx, hy,     hx - 6, hy,     hx - 6, hy + 3);
    ctx.bezierCurveTo(hx - 6, hy + 7, hx, hy + 10, hx, hy + 12);
    ctx.bezierCurveTo(hx, hy + 10, hx + 6, hy + 7, hx + 6, hy + 3);
    ctx.bezierCurveTo(hx + 6, hy,     hx, hy,     hx, hy + 3);
    ctx.fill();
  }

  for (let x = 0; x < canvas.width; x += 28) {
    drawHeart(x + 8, 3);
    drawHeart(x + 8, 462);
  }
  for (let y = 28; y < canvas.height - 28; y += 28) {
    drawHeart(3, y);
    drawHeart(462, y);
  }
  ctx.restore();
}
```

- [ ] **Step 6.3: Update `drawBackground()` to call new border functions**

Find `drawBackground()` and add two lines after the existing `if` checks:

```js
function drawBackground() {
  const stage = STAGES[state.stageIdx];
  ctx.fillStyle = stage.bg;
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  if (stage.borderStyle === 'leaf')    drawLeafBorder();
  if (stage.borderStyle === 'wave')    drawWaveBorder();
  if (stage.borderStyle === 'stars')   drawStarsBorder();
  if (stage.borderStyle === 'cookies') drawCookieBorder();
  if (stage.borderStyle === 'hearts')  drawHeartsBorder();
}
```

- [ ] **Step 6.4: Visual test in browser console**

```js
// Test cookie border:
state.stageIdx = 3;
drawBackground();
// Expect: dark chocolate background with small brown circles along all edges

// Test hearts border:
state.stageIdx = 4;
drawBackground();
// Expect: deep lavender background with pink heart shapes along all edges
```

- [ ] **Step 6.5: Commit**

```bash
git add snake.html
git commit -m "feat(visual): add cookie and hearts border styles for stages 4-5"
```

---

## Chunk 4: Integration

### Task 7: Update `tick()` for dog movement and collision

**Files:**
- Modify: `snake.html` (`tick` function, lines ~471–492)

- [ ] **Step 7.1: Add `tickCount` increment and `moveDog` call at top of `tick()`**

Find `tick()` and add three lines at the very beginning of the function body:

```js
function tick() {
  state.tickCount++;
  const stage = STAGES[state.stageIdx];
  if (stage.hasDog && state.dog && state.tickCount % 3 === 0) moveDog();

  state.dir = state.nextDir;
  // ... rest of existing tick unchanged
```

- [ ] **Step 7.2: Add dog collision check after `newHead` is computed**

After the line `const newHead = { x: head.x + state.dir.x, y: head.y + state.dir.y };`, add the wall-collision check (already exists), then before the self-collision check, insert:

```js
  if (stage.hasDog && state.dog && newHead.x === state.dog.x && newHead.y === state.dog.y) {
    handleCollision(); return;
  }
```

The full updated `tick()` function should look like:

```js
function tick() {
  state.tickCount++;
  const stage = STAGES[state.stageIdx];
  if (stage.hasDog && state.dog && state.tickCount % 3 === 0) moveDog();

  state.dir = state.nextDir;
  const head = state.snake[0];
  const newHead = { x: head.x + state.dir.x, y: head.y + state.dir.y };

  if (newHead.x < 0 || newHead.x >= COLS || newHead.y < 0 || newHead.y >= ROWS) {
    handleCollision(); return;
  }

  // Food check comes BEFORE dog check: if dog walks onto a food cell, the snake can
  // still eat the food. Dog collision is only lethal if there's no food there.
  const ateFood = newHead.x === state.food.x && newHead.y === state.food.y;

  if (!ateFood && stage.hasDog && state.dog && newHead.x === state.dog.x && newHead.y === state.dog.y) {
    handleCollision(); return;
  }

  const bodyToCheck = ateFood ? state.snake : state.snake.slice(0, -1);
  if (bodyToCheck.some(s => s.x === newHead.x && s.y === newHead.y)) {
    handleCollision(); return;
  }

  state.snake.unshift(newHead);
  if (!ateFood) {
    state.snake.pop();
  } else {
    handleFoodEaten();
  }
  render();
}
```

- [ ] **Step 7.3: Verify dog moves in browser**

Skip ahead to stage 4 in console and watch the dog move:
```js
// Force stage 4 (after starting game):
stopLoop();
state.stageIdx = 3;
state.dog = spawnDog(state.snake, state.food);
render();
startLoop();
// Expect: dog appears on canvas and visibly moves every 3 snake steps
```

- [ ] **Step 7.4: Commit**

```bash
git add snake.html
git commit -m "feat(dog): integrate dog movement and collision into game tick"
```

---

### Task 8: Update `render()`, `spawnFood` call sites, and `showStageClear`

**Files:**
- Modify: `snake.html` (`render`, `handleFoodEaten`, `showStageClear`)

- [ ] **Step 8.1: Update `handleCollision()` to re-place dog if it overlaps the new snake spawn**

The snake always resets to cells `(10,10), (9,10), (8,10)`. If the dog happens to be on one of those cells, the snake dies instantly on the next tick. Add a dog re-spawn check inside the `setTimeout` callback in `handleCollision`, right after the snake/dir reset:

Find this block inside the `setTimeout` in `handleCollision`:
```js
state.snake = createSnake();
state.dir = { x: 1, y: 0 };
state.nextDir = { x: 1, y: 0 };
state.currentDirKey = 'ArrowRight';
updateHUD();
render();
startLoop();
```

Replace with:
```js
state.snake = createSnake();
state.dir = { x: 1, y: 0 };
state.nextDir = { x: 1, y: 0 };
state.currentDirKey = 'ArrowRight';
// If dog overlaps new snake spawn, re-place it
const stg = STAGES[state.stageIdx];
if (stg.hasDog && state.dog) {
  const snakeSet = new Set(state.snake.map(s => `${s.x},${s.y}`));
  if (snakeSet.has(`${state.dog.x},${state.dog.y}`)) {
    state.dog = spawnDog(state.snake, state.food);
  }
}
updateHUD();
render();
startLoop();
```

- [ ] **Step 8.2: Update `render()` to draw the dog**

Find `render()` and add the dog draw call after `drawFood()`:

```js
function render() {
  drawBackground();
  drawSnake();
  drawFood();
  if (state.dog) drawDog(state.dog.x, state.dog.y);
}
```

- [ ] **Step 8.3: Update `handleFoodEaten()` — pass `state.dog` to `spawnFood`**

Find this line in `handleFoodEaten`:
```js
state.food = spawnFood(state.snake);
```
Replace with:
```js
state.food = spawnFood(state.snake, state.dog);
```

- [ ] **Step 8.4: Update `showStageClear()` — pass `state.dog` to `spawnFood` and spawn new dog**

Find the `showStageClear` timeout body. Update the `spawnFood` call and add dog spawn:

Find:
```js
state.food = spawnFood(state.snake);
```
Replace with:
```js
state.food = spawnFood(state.snake, state.dog);
state.dog = STAGES[state.stageIdx].hasDog ? spawnDog(state.snake, state.food) : null;
```

Note: at this point in `showStageClear`, `state.stageIdx` has already been incremented to the new stage — so `STAGES[state.stageIdx].hasDog` correctly checks the *incoming* stage.

- [ ] **Step 8.5: End-to-end browser test — play through to stage 4**

1. Open `snake.html`, press arrow key to start
2. Play through stage 1 (10 apples) → stage 2 (10 fish) → stage 3 (10 stars)
3. On stage 4: confirm brown dog appears on the chocolate background
4. Move snake into dog → confirm red flash + life lost, dog stays in position
5. Confirm dog visibly wanders around while playing
6. Confirm food never spawns on dog's cell

- [ ] **Step 8.6: Commit**

```bash
git add snake.html
git commit -m "feat(dog): wire dog into render and stage transition; fix spawnFood call sites"
```

---

### Task 9: Add sounds for stages 4 and 5

**Files:**
- Modify: `snake.html` (`startMusic`, `playEatSound`)

- [ ] **Step 9.1: Extend `melodies` array in `startMusic`**

Find the `melodies` array inside `startMusic`. It currently has 3 entries (indices 0, 1, 2). Add two more:

```js
const melodies = [
  // Jungle: square wave, 160 BPM
  { type: 'square',   notes: [[523,0.18],[659,0.18],[784,0.18],[659,0.18],[523,0.18],[392,0.18],[659,0.18],[523,0.36]] },
  // Ocean: sine wave, 100 BPM
  { type: 'sine',     notes: [[262,0.3],[330,0.3],[392,0.3],[440,0.3],[392,0.3],[330,0.6]] },
  // Space: sawtooth wave, 130 BPM
  { type: 'sawtooth', notes: [[220,0.23],[330,0.23],[440,0.23],[523,0.23],[659,0.23],[440,0.23],[330,0.23],[220,0.46]] },
  // Cookie Land: triangle wave, upbeat ~130 BPM
  { type: 'triangle', notes: [[523,0.23],[659,0.23],[784,0.23],[880,0.23],[784,0.23],[659,0.23],[523,0.23],[523,0.46]] },
  // Doll Land: sine wave, gentle ~90 BPM
  { type: 'sine',     notes: [[392,0.33],[440,0.33],[494,0.33],[523,0.33],[494,0.33],[440,0.66]] },
];
```

- [ ] **Step 9.2: Extend `configs` array in `playEatSound`**

Find the `configs` array inside `playEatSound`. It has 3 entries. Add two more:

```js
const configs = [
  { type: 'square',   startF: 400, endF: 800,  dur: 0.12 }, // Jungle
  { type: 'sine',     startF: 600, endF: 200,  dur: 0.15 }, // Ocean
  { type: 'sawtooth', startF: 300, endF: 1200, dur: 0.08 }, // Space
  { type: 'sine',     startF: 500, endF: 900,  dur: 0.10 }, // Cookie: sweet pop
  { type: 'triangle', startF: 800, endF: 600,  dur: 0.18 }, // Doll: soft chime
];
```

- [ ] **Step 9.3: Verify sound in browser**

1. Skip to stage 4 in DevTools:
```js
stopLoop(); stopMusic();
state.stageIdx = 3;
startMusic(3);
```
Expected: hear an upbeat triangle-wave melody (different from the sawtooth Space music)

2. Test eat sound:
```js
playEatSound(3); // expect: short rising sweet pop (sine 500→900Hz)
playEatSound(4); // expect: gentle descending chime (triangle 800→600Hz)
```

- [ ] **Step 9.4: Commit**

```bash
git add snake.html
git commit -m "feat(sound): add music and eat sounds for Cookie Land and Doll Land stages"
```

---

## Final Integration Test

- [ ] **Full playthrough test**

1. Open `snake.html` fresh in browser
2. Press arrow key — Stage 1 (Jungle) starts with leaf border ✓
3. Eat 10 apples → Stage Clear → Stage 2 (Ocean) with wave border ✓
4. Eat 10 fish → Stage Clear → Stage 3 (Space) with star border ✓
5. Eat 10 stars → Stage Clear → **Stage 4 (Cookie Land)**:
   - Dark chocolate background with cookie dot border ✓
   - Caramel snake ✓
   - Pink candy food ✓
   - Dog appears and wanders ✓
   - Triangle wave music plays ✓
   - Eating candy plays sweet pop sound ✓
   - Touching dog → life lost, dog stays in place ✓
6. Eat 10 candies → Stage Clear → **Stage 5 (Doll Land)**:
   - Deep lavender background with pink heart border ✓
   - Baby pink snake ✓
   - Brown teddy bear food ✓
   - Dog appears and wanders ✓
   - Gentle sine wave music plays ✓
   - Eating teddy bear plays soft chime sound ✓
   - Touching dog → life lost ✓
7. Eat 10 teddy bears → **Victory screen** with confetti ✓
8. Click "Play Again" → resets to Stage 1 with no dog ✓
9. Lose all 3 lives → Game Over screen → Try Again works ✓

- [ ] **Final commit**

```bash
git add snake.html
git commit -m "feat: complete stages 4-5 (Cookie Land + Doll Land) with dog obstacle"
```
