# Snake Game for Kids — Design Spec
**Date:** 2026-03-14
**Status:** Approved
**Delivery:** Single HTML file (`snake.html`)

---

## Overview

A simple, kid-friendly snake game running entirely in a single HTML file using an HTML Canvas for rendering. Players guide a snake to eat food across 3 themed stages, with increasing speed and a scoring system. 3 lives, cute rounded visuals, and a victory screen on completion.

---

## Tech Stack

- **Platform:** Single self-contained `snake.html` file
- **Rendering:** HTML5 Canvas (2D context)
- **Font:** Google Fonts `Fredoka One` loaded via CDN `<link>` tag; fallback to `'Comic Sans MS', cursive` if offline
- **No JS libraries, no build step, no sound**

---

## Stages & Themes

| Stage | Theme | Background | Snake Color | Food | Food Emoji | Speed |
|-------|-------|-----------|-------------|------|------------|-------|
| 1 | 🌿 Jungle | Deep green (`#1a472a`) with leaf-pattern border | Lime green (`#6bcb77`) | Red apple (`#ff4d4d`) | 🍎 | 200ms |
| 2 | 🌊 Ocean | Deep blue (`#0d3b6e`) with wave-pattern border | Teal/cyan (`#00b4d8`) | Yellow fish (`#ffd166`) | 🐟 | 150ms |
| 3 | 🚀 Space | Dark purple (`#0d0221`) with scattered white star dots | Purple/pink (`#c77dff`) | Gold star (`#ffd700`) | ⭐ | 100ms |

- Each stage requires eating **10 food items** to advance.

---

## Grid & Movement

- **Grid size:** 20×20 cells
- **Cell size:** 24px → canvas is 480×480px
- **Movement:** One cell per tick (interval varies by stage)
- **Controls:** Arrow keys only; pressing opposite of current direction is ignored
- **Starting state (each life):** Snake length = 3 segments, positioned at grid center (cell 10,10), facing right

---

## Scoring

| Event | Points |
|-------|--------|
| Eat food in Stage 1 | +10 pts |
| Eat food in Stage 2 | +20 pts |
| Eat food in Stage 3 | +30 pts |

- Score accumulates across all stages
- Score is **NOT reset on life loss** — retained through the stage
- Score resets to 0 on "Try Again" or "Play Again" (full restart)

---

## Lives System

- Player starts with **3 lives** (shown as ❤️❤️❤️ in HUD)
- Lose a life when: hitting a wall, or hitting own body
- On life lost:
  - Old snake body is **immediately cleared** from canvas
  - **Game loop is paused** (no movement, no input processed) for the entire flash duration
  - Brief red flash overlay for 0.5s
  - After flash: snake resets to center (cell 10,10), length 3, facing right; game loop resumes
  - Stage and score are retained
  - No invincibility window after respawn (game loop resumes immediately after reset)
- **Game over** when all 3 lives are gone

---

## Snake Appearance

- Segments: rounded rectangles, 18×18px (6px total gap) centered in each 24px cell
- Head: same rounded rect + two 3px radius white dot eyes
- Last segment (tail): drawn at 70% size (≈13×13px), centered in its cell
- Color: themed per stage

---

## Food Appearance

- Stage 1: red filled circle (r=9px) + small green stem line (2px wide, 5px tall)
- Stage 2: yellow filled ellipse (14px wide × 9px tall)
- Stage 3: gold 5-point star (10px outer radius, 5px inner radius)
- All food: soft `shadowBlur: 10` glow in matching color
- Spawns on a random cell not occupied by any snake segment (including at reset, the starting cells (8,10), (9,10), (10,10) are excluded); one food item at a time

---

## UI Screens

### 1. Start Screen
- Background: **always Jungle theme** (`#1a472a`), regardless of restart state
- "Try Again" and "Play Again" buttons skip the start screen entirely and jump directly to Stage 1 gameplay
- Big centered title: "🐍 Snake!"
- Subtitle: "🌿 Jungle"
- Prompt below: "Press any arrow key to start"
- Animation: a 5-segment snake traces a fixed rectangular loop around a **4×4 cell area** (96×96px, perimeter = 12 cells) centered horizontally, 80px above the title. The snake moves one cell per 200ms, looping indefinitely counterclockwise. Segments may overlap the tail during the loop (purely decorative, no collision logic).

### 2. HUD (During Play)
Rendered as HTML `<div>` elements above the canvas:
- **Left:** "Score: 240"
- **Center:** Stage name (e.g. "🌊 Ocean")
- **Right:** ❤️❤️❤️ (heart emoji × remaining lives)
- **Below HUD, above canvas:** Food counter: `[emoji] 3 / 10` where emoji is the current stage's food emoji (🍎 / 🐟 / ⭐)

### 3. Stage Clear Screen
- Semi-transparent black canvas overlay
- Text: "Stage Complete! 🎉"
- Current score shown below
- **Auto-advances after 2 seconds** — no button
- Used for Stage 1→2 and Stage 2→3 transitions only

### 4. Victory Screen (after Stage 3 clear)
- Replaces Stage Clear screen entirely for Stage 3 (no "Stage Complete" shown)
- Full canvas overlay: "You Win! 🏆"
- Final score displayed prominently
- **Confetti animation:**
  - 80 particles, each 8×8px filled squares
  - Colors chosen randomly from: `['#ff6b6b','#ffd166','#06d6a0','#118ab2','#c77dff','#ff9f43','#48dbfb']`
  - Each particle starts at a random X position along the top of the canvas, Y = -8
  - Falls at speed 2–4 px/frame (randomized per particle), horizontal drift –1 to +1 px/frame (randomized)
  - When a particle reaches the bottom of the canvas, it resets to Y = -8 at a new random X (infinite loop)
- "Play Again" button → full reset: score to 0, lives to 3, Stage 1; goes **directly into gameplay** — the start screen is never shown again after the first play

### 5. Game Over Screen
- Full canvas overlay: "Game Over 💀"
- Final score shown
- "Try Again" button → full reset: score to 0, lives to 3, Stage 1; goes **directly into gameplay** — the start screen is never shown again after the first play

---

## File Structure

```
games/
  snake.html        ← entire game (HTML + inline CSS + inline JS)
  docs/
    superpowers/
      specs/
        2026-03-14-snake-game-design.md
```

---

## Implementation Notes

- **Game loop:** Use `setInterval` keyed to stage speed. On life loss: call `clearInterval`, show flash overlay (setTimeout 500ms), then call `startGameLoop()` again. This prevents ghost intervals.
- **Input handling:** Buffer one pending direction per tick. On keydown, if the pressed direction is not opposite to current direction, store it as `nextDir`. At the start of each tick, apply `nextDir` as `currentDir`. This prevents self-collision on fast double-tap.
- **Food on life loss:** Food position persists unchanged across life loss. No new food is spawned. New food only spawns when the previous food is eaten.
- **Confetti loop:** Track the `requestAnimationFrame` ID in a variable. Cancel it (`cancelAnimationFrame`) when "Play Again" is clicked or any screen transition occurs.
- **Start screen animation position:** The 4×4 cell decorative snake loop is centered horizontally on the canvas, starting 60px from the top of the canvas.

---

## Out of Scope

- Sound effects
- Mobile/touch controls
- High score persistence (localStorage)
- More than 3 stages
- Multiplayer
- Stage select screen
