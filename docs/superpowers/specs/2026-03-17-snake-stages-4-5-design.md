# Snake Game — Stages 4 & 5 Design Spec

**Date:** 2026-03-17
**Status:** Approved

---

## Overview

Add two new stages to the existing 3-stage snake game:

- **Stage 4: Cookie Land 🍪** — eat candy (🍬), avoid a wandering dog
- **Stage 5: Doll Land 🧸** — eat teddy bears (🧸), avoid a wandering dog

Both stages keep the same speed as Stage 3 (100ms tick interval). Each stage introduces one dog obstacle that wanders the grid; the snake head touching the dog's cell costs one life (same as wall/self collision).

---

## Stage Configurations

Append two entries to the `STAGES` array (indices 3 and 4).

### Stage 4: Cookie Land 🍪

| Property      | Value                         |
|---------------|-------------------------------|
| `name`        | `'🍪 Cookie Land'`            |
| `bg`          | `'#3d1a00'` (dark chocolate)  |
| `snake`       | `'#ff9a3c'` (caramel orange)  |
| `food`        | `'#ff6eb4'` (hot pink)        |
| `foodEmoji`   | `'🍬'`                        |
| `speed`       | `100`                         |
| `borderStyle` | `'cookies'`                   |
| `hasDog`      | `true`                        |

**Food shape — `drawCandy(cx, cy)`:**
- Stick: fillRect, width 4px, height 10px, centered above cx, dark pink `#cc3380`
- Body: arc radius 9px centered at (cx, cy+2), filled hot pink `#ff6eb4`

**Border — `drawCookieBorder()`:**
- Draw small filled circles, radius 5px, color `#a0522d`, every 24px along all 4 edges at y=10 (top), y=470 (bottom), x=10 (left), x=470 (right)

---

### Stage 5: Doll Land 🧸

| Property      | Value                         |
|---------------|-------------------------------|
| `name`        | `'🧸 Doll Land'`              |
| `bg`          | `'#2d1b4e'` (deep lavender)   |
| `snake`       | `'#ffb3d9'` (baby pink)       |
| `food`        | `'#cd853f'` (warm brown)      |
| `foodEmoji`   | `'🧸'`                        |
| `speed`       | `100`                         |
| `borderStyle` | `'hearts'`                    |
| `hasDog`      | `true`                        |

**Food shape — `drawTeddyBear(cx, cy)`:**
- Head: arc radius 8px at (cx, cy+1), filled `#cd853f`
- Left ear: arc radius 4px at (cx-8, cy-7), filled `#b8732a`
- Right ear: arc radius 4px at (cx+8, cy-7), filled `#b8732a`
- Eyes: two filled circles radius 2px, at (cx-3, cy) and (cx+3, cy), color `#333`
- Nose: filled circle radius 2px at (cx, cy+4), color `#333`

**Border — `drawHeartsBorder()`:**
- Draw small hearts, size ~8px, color `#ff69b4`, every 28px along all 4 edges at y=10 (top), y=470 (bottom), x=10 (left), x=470 (right)
- Heart drawn with two arcs + lineTo path (standard heart path scaled to ~8px)

---

## Dog Obstacle Mechanic

### Spawn

`spawnDog(snake, food)`:
- Build occupied set from all snake cells + food cell
- Pick random cell not in occupied, with max 400 attempts (same guard as `spawnFood`)
- If no cell found after 400 attempts, place dog at (0, 0) as fallback
- Called: in `showStageClear` timeout before `startLoop()`, in `showStageClear` for each new stage

**Dog does NOT reset on snake death.** It stays in its current position. `spawnDog` is NOT called in `handleCollision`.

### Movement

`moveDog()`:
- Build a shuffled list of all 4 directions: `[{0,-1},{0,1},{-1,0},{1,0}]` randomized (Fisher-Yates or sort by random)
- Iterate through the shuffled list; pick the first direction where `{x+dx, y+dy}` is within grid bounds (0 ≤ x < COLS, 0 ≤ y < ROWS)
- If all 4 directions hit a wall (impossible unless COLS or ROWS < 2, not a concern here), stay in place
- Update `state.dog` to the new position
- Dog does NOT avoid snake body or food

### Collision Detection

In `tick()`, after computing `newHead` and before self-collision check:
```
if (stage.hasDog && state.dog && newHead.x === state.dog.x && newHead.y === state.dog.y) {
  handleCollision(); return;
}
```

### Visual — `drawDog(gx, gy)`

Draws a cute dog face centered in grid cell (gx, gy). Canvas coordinates: `px = gx * CELL`, `py = gy * CELL`. Center: `cx = px + CELL/2`, `cy = py + CELL/2`.

| Part       | Shape       | Color      | Position / Size                              |
|------------|-------------|------------|----------------------------------------------|
| Head       | arc         | `#d4a55a`  | center (cx, cy+1), radius 9px                |
| Left ear   | arc         | `#8b6914`  | center (cx-6, cy-7), radius 5px              |
| Right ear  | arc         | `#8b6914`  | center (cx+6, cy-7), radius 5px              |
| Left eye   | arc         | `#333`     | center (cx-3, cy-1), radius 2px              |
| Right eye  | arc         | `#333`     | center (cx+3, cy-1), radius 2px              |
| Nose       | arc         | `#333`     | center (cx, cy+3), radius 2px                |

Draw order: ears first (behind head), then head, then eyes, then nose.

---

## State Changes

In `initState()`, add to the `state` object:
```js
dog: null,      // { x, y } when hasDog stage active; null otherwise
tickCount: 0,   // increments each tick; never reset (modulo math always correct)
```

**Important:** `tickCount` must be explicitly initialized to `0` inside `initState()`. If omitted, `undefined % 3` evaluates to `NaN`, and `NaN++` stays `NaN`, permanently breaking dog movement.

`tickCount` is never explicitly reset — `% 3` works indefinitely.

In `initState()` and `showStageClear` transition: after setting up `state.food`, call `spawnDog` if `STAGES[state.stageIdx].hasDog` is true; otherwise set `state.dog = null`.

---

## `spawnFood` Update

`spawnFood(snake)` must also exclude the dog's current position (if any):
```js
function spawnFood(snake, dog) {
  const occupied = new Set(snake.map(s => `${s.x},${s.y}`));
  if (dog) occupied.add(`${dog.x},${dog.y}`);
  // ... rest unchanged
}
```
Update **all three** call sites to pass `state.dog`:
1. `initState()` — `spawnFood(snake)` → `spawnFood(snake, null)` (no dog at start)
2. `handleFoodEaten()` — `spawnFood(state.snake)` → `spawnFood(state.snake, state.dog)`
3. `showStageClear` timeout — `spawnFood(state.snake)` → `spawnFood(state.snake, state.dog)`

---

## Game Tick Changes

In `tick()`, add at the top of the function body:
```js
state.tickCount++;
const stage = STAGES[state.stageIdx];
if (stage.hasDog && state.dog && state.tickCount % 3 === 0) moveDog();
```

Then after computing `newHead`, before the existing self-collision check, add:
```js
if (stage.hasDog && state.dog && newHead.x === state.dog.x && newHead.y === state.dog.y) {
  handleCollision(); return;
}
```

---

## Render Changes

In `render()`, add after `drawFood()`:
```js
if (state.dog) drawDog(state.dog.x, state.dog.y);
```

### `drawFood` — Add Branches for Stages 4 and 5

The existing `drawFood()` function:
```js
if (state.stageIdx === 0) drawApple(cx, cy);
else if (state.stageIdx === 1) drawFish(cx, cy);
else drawStar(cx, cy);
```

The `else` branch catches indices 2, 3, and 4. Extend it to handle new stages:
```js
if (state.stageIdx === 0)      drawApple(cx, cy);
else if (state.stageIdx === 1) drawFish(cx, cy);
else if (state.stageIdx === 2) drawStar(cx, cy);
else if (state.stageIdx === 3) drawCandy(cx, cy);
else                           drawTeddyBear(cx, cy);
```

---

## `drawBackground` Changes

Add two new branches:
```js
if (stage.borderStyle === 'cookies') drawCookieBorder();
if (stage.borderStyle === 'hearts')  drawHeartsBorder();
```

---

## Sound Changes

### `startMusic(stageIdx)` — append two entries to `melodies` array

```js
// Cookie Land: triangle wave, upbeat ~130 BPM
{ type: 'triangle', notes: [[523,0.23],[659,0.23],[784,0.23],[880,0.23],[784,0.23],[659,0.23],[523,0.23],[523,0.46]] },
// Doll Land: sine wave, gentle ~90 BPM
{ type: 'sine',     notes: [[392,0.33],[440,0.33],[494,0.33],[523,0.33],[494,0.33],[440,0.66]] },
```

### `playEatSound(stageIdx)` — append two entries to `configs` array

```js
{ type: 'sine',     startF: 500, endF: 900,  dur: 0.10 }, // Cookie: sweet pop
{ type: 'triangle', startF: 800, endF: 600,  dur: 0.18 }, // Doll: soft chime
```

---

## Victory Condition (No Change Needed)

`handleFoodEaten()` already checks `state.stageIdx === STAGES.length - 1`. With two new entries in `STAGES`, this naturally becomes index 4. No code change needed.

---

## `showStageClear` Changes

After `state.food = spawnFood(state.snake)`, add:
```js
state.dog = STAGES[state.stageIdx].hasDog ? spawnDog(state.snake, state.food) : null;
```

---

## `fullReset` — No Dog Spawn Needed

`fullReset` resets `stageIdx` to 0 (Jungle), which has `hasDog: false`. `state.dog` is set to `null` by `initState()`. No special dog handling required in `fullReset`.

---

## Files Changed

- `snake.html` — all changes in one file
