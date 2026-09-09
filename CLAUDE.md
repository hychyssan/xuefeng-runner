# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Chrome T-Rex runner game fork with custom character reskin, BGM, coin system, and enhanced audio. Static website — no build system, no package manager, no tests. See `AGENTS.md` for file structure, quirks, audio system details, controls, and how to run.

## How to Run

```
python3 -m http.server 8000
```

Open `http://localhost:8000` in a browser. Press Space, ↑, or W to start.

## Architecture

All game logic lives in a single `index.js` IIFE (~2400 lines). The top-level function `Runner` is set on `window.Runner` and invoked at the bottom of the file. It's a singleton — a second call returns the existing instance.

### Class/Prototype Hierarchy

- **`Runner`** — top-level controller: owns the game loop (`update()` via `requestAnimationFrame`), keyboard/touch input routing, state transitions, audio orchestration, and canvas sizing. Not a "class" in the ES6 sense; methods are assigned to `Runner.prototype` as an object literal.
- **`Trex`** — player character: jump physics, ducking, sprite animation frames, collision box, blink invincibility after collision.
- **`Horizon`** — manages the scrolling ground line, clouds, obstacles, coins, and night mode. Each is a sub-object:
  - `HorizonLine` — scrolling ground texture
  - `Cloud` — decorative clouds
  - `Obstacle` — cacti/pterodactyls, with `typeConfig` for sizing/collision/multiple-obstacle grouping
  - `Coin` — collectible coins with random spawn intervals
  - `NightMode` — day/night toggle at distance thresholds (700, 1400, …), inverts colors
- **`DistanceMeter`** — score display: digits rendered from a sprite, achievement flash at 100-point milestones.
- **`GameOverPanel`** — game-over sprite overlay with restart button.
- **`CollisionBox`** — AABB used by `checkForCollision()` and `boxCompare()`.

Helpers: `createCanvas()`, `getTimeStamp()`, `decodeBase64ToArrayBuffer()`, `vibrate()`.

### Game State Machine

States are tracked via boolean flags on `Runner`:

```
[page load] → activated=false, playing=false, crashed=false
  ↓ (first jump/start key while !crashed)
activated=true, playing=true → game loop runs
  ↓ (collision)
playing=false, crashed=true, paused=true → game-over panel shown
  ↓ (restart: Enter, click, or jump key after 750ms)
playing=true, crashed=false, paused=false → new game
```

- `playingIntro` (on `Runner`) and `Trex.playingIntro` gate whether the horizon scrolls — stays static during the first-jump intro animation.
- `inverted` toggles night mode colors; `invertTimer` tracks fade duration.
- Visibility/blur handlers pause audio when the tab loses focus.

### Game Loop (`Runner.update`)

1. Compute `deltaTime` from last frame timestamp.
2. If `playing`: clear canvas, update Trex jump physics, advance `runningTime`.
3. If past `CLEAR_TIME` (3s): scroll horizon, spawn obstacles/coins.
4. Check collision with first obstacle → `gameOver()` if hit.
5. Draw and check coin collection (AABB vs Trex); coin adds +200 distance and increments `coinScore`.
6. Update `DistanceMeter`; play score achievement sound at 100-pt milestones.
7. Handle night-mode toggle at `INVERT_DISTANCE` (700) intervals.
8. Update Trex animation frame.
9. Schedule next frame via `requestAnimationFrame`.

### Audio: Two Separate Systems

1. **Web Audio API** (`audioContext`) — for short SFX (jump, hit, score). Base64-encoded in `<template id="audio-resources">`, decoded async into `this.soundFx` map. iOS skips this entirely (`IS_IOS` check).

2. **`<audio>` elements** — for BGM and music clips. Referenced from DOM by ID:
   - `bgm-audio` — gameplay BGM (looped, volume 0.3)
   - `end-bgm-audio` — game-over BGM (looped, volume 0.3, 250ms delay)
   - `start-audio` — start sound (500ms delay, volume 0.25)
   - `qiaolezi-audio` — coin collection (volume 0.625)
   - `fate-audio`, `run-audio` — used by sound pool (volume 0.75)

3. **Sound pool** (`startSoundPool` / `scheduleNextSoundPool`) — timer-based system:
   - First trigger at 10s + random(0–30s), then every 30s + random(0–30s)
   - Weighted random selection from pool: fate (1x), Qiaolezi (0.25x), run (1x)
   - `run.mp3` weight doubles at distance 2100 (2x) and 4200 (4x)
   - Checks `audio.paused` before playing to avoid overlap

### Key Constants Worth Knowing

| Constant | Value | Effect |
|---|---|---|
| `IS_HIDPI` | hardcoded `true` | Always loads 2x sprite sheet |
| `IS_IOS` | platform regex | Disables Web Audio SFX |
| `FPS` | 60 | Frame rate target |
| `DEFAULT_WIDTH` | 600 | Canvas max width |
| `Runner.config.SPEED` | 6 | Starting speed |
| `Runner.config.MAX_SPEED` | 13 | Terminal speed |
| `Runner.config.CLEAR_TIME` | 3000 | Obstacles start after 3s |
| `Runner.config.GAP_COEFFICIENT` | 0.6 | Gap between obstacles |

### Custom Modifications vs Upstream Chromium

- `IS_HIDPI` forced to `true` with 2x sprite sheet (custom character art)
- `coinScore` counter and coin collection mechanics (not in original)
- BGM system with `<audio>` elements (original only had Web Audio SFX)
- Sound pool for periodic random audio clips
- `stopAllAudio()` on game over, `endBgmAudio` looping
- Page turns grayscale on crash: `.offline.crashed { filter: grayscale(100%) }`
- W key = jump, S key = duck (original only had ↑/↓/Space)
- `playStartSound()` with 500ms delay, `playEndBgm()` with 250ms delay
- Volume hardcoded per audio source (not configurable)

### Editing Guidelines

- The entire game is one IIFE in `index.js`. Be careful with scope — variables declared with `var` at the top of the IIFE are module-private; properties on `Runner`/`Runner.prototype` are public.
- Sprite coordinates in `Runner.spriteDefinition.HDPI` must exactly match the PNG in `assets/default_200_percent/200-offline-sprite.png`. If you replace the sprite sheet, update the JSON coordinates too.
- Audio file paths in `index.html` `<audio>` elements must match actual files in `assets/music/`.
- BGM is controlled via `Runner` methods (`playBgm`, `stopBgm`, `playEndBgm`, etc.), not through `Horizon` or `Trex`.
- The `Runner` singleton pattern means there's no clean teardown — refreshing the page is the only way to fully reset state besides `restart()`.
