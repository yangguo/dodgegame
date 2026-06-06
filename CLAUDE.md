# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Two versions of a 2D dodgeball game — both single-file, zero-dependency HTML:

- **`vanilla.html`** (~1300 lines): Vanilla JS on HTML5 Canvas. Original version.
- **`phaser.html`** (~1360 lines): Phaser 3.80.1 (CDN) with Arcade Physics. Feature-parity migration.
- **`three.html`** (~835 lines): Three.js r160 (CDN) + Cannon.js 0.6.2. 3D top-down version.

Both are 2v2: player (blue, WASD+mouse) + teal AI ally vs two red/orange AI enemies. No build tools, no server.

## How to Run

Open either file directly in a browser. No server, no build step.

## Architecture

### `vanilla.html` — Vanilla Canvas

All logic in one IIFE. Key sections:

| Lines | Section |
|-------|---------|
| 42-74 | Constants (speeds, cooldowns, sizes, colors) |
| 92-122 | Canvas setup, game state, factories |
| 150-200 | Obstacle generation, game reset/start |
| 221-340 | Input, throwing, velocity tracking, collision helpers |
| 341-547 | Player movement, AI logic (`updateAIChar`) |
| 549-627 | Ball physics, hit detection, HP, win/loss |
| 628-920 | Hit effects (ring + particles), update loop |
| 921-1300 | Rendering (characters, balls, HUD, overlays) |
| 1288-1295 | Main loop (`requestAnimationFrame`) |

### `phaser.html` — Phaser 3

Two scene classes. Phaser Sprites use `setData()`/`getData()` for custom properties instead of plain JS objects.

**BootScene** (line 93): Generates runtime textures via `Graphics.generateTexture()` — `char`, `ball`, `particle`.

**GameScene** (line 119): All gameplay. Key methods:

| Method | Line | Purpose |
|--------|------|---------|
| `create()` | 122 | Sets up physics groups, input, worldbounds handler, characters |
| `resetGame()` | 256 | Resets all state, recreates obstacles, restores sprites |
| `startGame()` | 323 | Hides start screen, starts playing (`setAlpha(0)`) |
| `endGame()` | 570 | Shows game over (`setAlpha(1)`) |
| `tryThrowToward()` | 454 | Ball creation + physics velocity (`speed * 60` for px/frame→px/sec) |
| `onBallHit()` | 515 | Overlap callback — damage, death, hit effects |
| `spawnHitEffect()` | 579 | Particle explosion + camera shake |
| `updateP1()` | 635 | Player movement, aim, cooldown |
| `updateAIChar()` | 678 | Full AI: threat assessment, dodge, throw, lead-targeting |
| `resolveCharObstacles()` | 872 | Manual circle-vs-obstacle push-out |
| `update()` | 1020 | Main loop dispatch |
| `renderOverlays()` | 1038 | Draws all HUD/effects via single Graphics object |
| `drawCharacter()` | 1081 | Character rendering (body, eyes, HP bar, markers) |
| `drawStartScreen()` / `drawGameOver()` | 1257/1293 | UI overlays with Text objects (alpha-toggled) |

**Critical pattern — UI text visibility**: Start/go screens use Phaser Text objects toggled via `setAlpha()`. `startGame()` must hide start texts (alpha 0), `endGame()` must show gameover texts (alpha 1), `resetGame()` must show start texts (alpha 1) and hide gameover texts (alpha 0). Missing any of these causes persistent text on screen.

**Velocity conversion**: Original game uses px/frame at 60fps. Phaser uses px/sec. All `setVelocity()` calls multiply by 60 (e.g., `CHAR_SPEED * 60`, `PLAYER_BALL_SPEED * 60`).

**Ball lifecycle**: Balls are Arcade sprites with `body.onWorldBounds = true`. The `worldbounds` event decrements `bouncesLeft`; at 0, `shouldRemove` flag is set. `removeDeadBalls()` cleans up flagged balls each frame.

## AI Behavior (`updateAIChar`)

Shared by ally and both enemies — priority-based reactive system:

1. **Threat assessment** — predict ball positions 24 frames ahead, build avoidance vector
2. **Center bias** — gentle push away from walls
3. **Movement** — dodge if threatened (>0.1 score), else seek combat distance, else wander
4. **Throw** — cooldown timer, `pickTarget` rotates targets, lead-targeting with jitter

## Key Constants

| Constant | Value | Note |
|----------|-------|------|
| `AI_THROW_INTERVAL_MIN/MAX` | 45-80 | Frames between AI throws |
| `AI_THREAT_RANGE` | 200px | Ball threat detection range |
| `PLAYER_THROW_COOLDOWN` | 15 frames | After any throw |
| `AI_BALL_SPEED` | 4.8 | vs `PLAYER_BALL_SPEED`: 5.0 |
| `BALL_MAX_BOUNCES` | 4 | Then auto-removed |
| `MAX_HP` | 10 | Per character |

## Known Pitfalls

- **AI `throwCooldown` must be decremented** in `updateAIChar()` — it doesn't use `updatePlayerCharacter()`. If AI stops throwing, check cooldown decrement.
- AI team has 2 characters vs 1 ally, so AI throws 2× as often — `AI_THROW_INTERVAL` is longer to compensate.
- Balls pass through obstacles (only characters collide with obstacles).
- In Phaser version, UI text visibility is managed via `setAlpha()` — missing a toggle in `startGame()`/`endGame()`/`resetGame()` causes ghost text on screen.
- `worldbounds` handler must guard against `bouncesLeft === undefined` (destroyed balls can still fire the event).
- Phaser input cannot be triggered via DOM-dispatched events — browser automation can't click "start" or simulate keypresses. Test gameplay manually.