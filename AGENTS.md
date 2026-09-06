# xuefeng-runner

Static website: Chrome T-Rex runner game with custom BGM, character reskin, and enhanced audio system. No build system, no package manager, no tests.

## Structure

- `index.html` — entry point, loads `index.js` and `index.css`, contains base64-encoded sound FX in `<template>` tag
- `index.js` — all game logic in a single IIFE, vanilla JS, no modules/imports
- `index.css` — game styles
- `assets/default_200_percent/` — sprite sheets and error icon (2x only, `IS_HIDPI` is hardcoded `true`)
- `assets/music/` — audio files:
  - `bgm.mp3` — background music, looped during gameplay
  - `end-bgm.mp3` — game over music, looped until restart
  - `start.mp3` — game start sound (0.5s delay)
  - `Qiaolezi.mp3` — coin collection sound (first coin only)
  - `sound/` — audio pool:
    - `fate.mp3`, `Qiaolezi.mp3`, `run.mp3` — random playback every 30s interval

## Key Quirks

- `IS_HIDPI` is forced `true` at `index.js:95` — always uses 2x sprite sheet; do not restore dynamic detection
- `IS_IOS` disables Web Audio sound FX entirely (`index.js:318`), BGM still plays via `<audio>` element
- Game auto-enters "arcade mode" (fullscreen scaling) on first jump via `setArcadeMode()`
- BGM volume hardcoded to 0.3 at `index.js:920`
- Sprite coordinates in `Runner.spriteDefinition` must match the sprite sheet PNGs exactly

## Audio System

### BGM
- `bgm.mp3` — loops during gameplay, volume 0.3
- `end-bgm.mp3` — loops after game over, volume 0.3
- `start.mp3` — plays once at game start (0.5s delay), volume 0.25

### Audio Pool
- Triggers after 10s, then every 30s interval (random time within interval)
- Pool: `fate.mp3` (weight 1), `Qiaolezi.mp3` (weight 0.25), `run.mp3` (weight 1)
- `run.mp3` weight doubles at distance 2100 and 4200
- Volume: 0.75
- Checks `paused` state before playing to avoid overlap

### Game Over Behavior
- `stopAllAudio()` immediately stops all audio
- Then plays `end-bgm.mp3` in loop

## Controls

- **Jump**: ↑, Space, W
- **Duck**: ↓, S
- **Restart**: Enter, click canvas, or jump key after 750ms

## Visual Effects

- Game over: entire page turns grayscale (CSS `filter: grayscale(100%)`)
- Restart: restores normal colors

## Coin System

- `coinScore` starts at -1, increments by 1 per coin collected
- First coin (score 0) triggers `Qiaolezi.mp3` sound
- Each coin adds 200 to distance

## How to Run

Open `index.html` in a browser. For local dev, any static file server works:
```
python3 -m http.server 8000
```

## Deployment

GitHub Pages at `https://hychyssan.github.io/xuefeng-runner/` — served directly from repo root.
