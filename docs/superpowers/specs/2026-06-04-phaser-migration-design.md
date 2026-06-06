# Phaser Migration Design

**Date:** 2026-06-04
**Status:** Approved

## Goal

Migrate the vanilla Canvas dodgeball game (`vanilla.html`, ~1300 lines) to a Phaser 3 version (`phaser.html`), keeping the single-file zero-config (CDN) format and 100% feature parity.

## Approach: Phaser Scene Refactoring (Option A)

Leverage Phaser's Arcade Physics, particle system, and scene management rather than just wrapping existing code.

## Project Structure

Single file `phaser.html`:

```html
<!DOCTYPE html>
<html>
<head><meta charset="UTF-8"><title>躲避球 - Phaser</title>
<style>/* minimal centering CSS */</style>
</head>
<body>
<script src="https://cdn.jsdelivr.net/npm/phaser@3.80.1/dist/phaser.min.js"></script>
<script>
// All game code in IIFE
(function() {
  // BootScene: generate textures
  // GameScene: full game logic
  // Phaser config + launch
})();
</script>
</body>
</html>
```

## Game Configuration

```js
const config = {
  type: Phaser.AUTO,
  width: 800,
  height: 500,
  backgroundColor: '#f5f5f5',
  physics: {
    default: 'arcade',
    arcade: { gravity: { y: 0 } }  // top-down, no gravity
  },
  scene: [BootScene, GameScene]
};
```

## Scenes

### BootScene

Generates all textures via `Graphics.generateTexture()`:
- `char` — 30×30 rounded rect (white, tinted per role at runtime via `setTint()`)
- `ball` — 12×12 circle (white, tinted by thrower team)
- `obstacle_circle` — circle obstacle
- `obstacle_ellipse` — ellipse obstacle
- `obstacle_rect` — rectangle obstacle
- `obstacle_diamond` — diamond obstacle
- `obstacle_pentagon` — pentagon obstacle

No external image files needed.

### GameScene

Full game logic. States managed by custom `gameState` variable ('ready' | 'playing' | 'gameover'), consistent with original.

## Characters

### Sprite Setup

Each character is an Arcade Physics Sprite using the `char` texture:

| Role | Tint Color | Team |
|------|-----------|------|
| p1 | 0x4A90E2 (blue) | player |
| ally | 0x3DC1B5 (teal) | player |
| ai1 | 0xE24A4A (red) | ai |
| ai2 | 0xE89B3D (orange) | ai |

Custom data stored via `sprite.setData()`:
- `hp`, `maxHp`, `role`, `team`, `facingAngle`
- `throwCooldown`, `aiWanderAngle`, `aiHoldTimer`, `aiState`
- `targetHitRates`, `dyingTimer`, `deathParticles`

Physics body: `setSize(30, 30)` with `setCircle(15)` offset to center.

### Movement

- P1: keyboard velocity (`setVelocity()`) + mouse aim for `facingAngle`
- Ally + AI: computed velocity via same algorithm as original, applied via `setVelocity()`
- All characters: `setCollideWorldBounds(true)`

### Obstacles

Static physics group. Characters `collider` with obstacles. Balls pass through obstacles (no collider).

## Balls

### Sprite Setup

Arcade Physics Sprite using `ball` texture, tinted by thrower's team color.

Physics:
- `setCircle(6)` matching radius
- `setBounce(1)` — perfect elastic bounce
- `setCollideWorldBounds(true)` — bounces off walls
- `gravity.y = 0`

### Bounce Counting

Listen to `physics.world.on('worldbounds', ...)` per ball. Track `bouncesLeft` via `setData()`. After 4 bounces or off-screen, `ball.destroy()`.

### Throwing

Original velocity + position calculation → `scene.physics.add.sprite(x, y, 'ball')` with `setVelocity(vx, vy)`.

## Collision Detection

### Ball vs Character (Hit)

```js
this.physics.add.overlap(ballGroup, charGroup, onBallHit, null, this);
```

In `onBallHit(ball, char)`:
1. Reduce `char.data.values.hp`
2. Destroy ball
3. Trigger hit effect (particles + flash ring + camera shake)
4. Check win/lose condition

### Character vs Obstacle

```js
this.physics.add.collider(chars, obstacles);
```

Balls intentionally NOT colliding with obstacles (matches original).

## Hit Effects

### Particle Burst

Use Phaser's particle system:

```js
scene.add.particles(hitX, hitY, 'particle', {
  speed: { min: 80, max: 250 },
  lifespan: 400,
  quantity: 16,
  scale: { start: 1, end: 0 },
  tint: [charColor, 0xffffff],  // mix character color + white
  gravityY: 200,
  emitting: false  // emit once via emitAt()
});
```

A small particle texture (`particle` — 4×4 white circle) generated in BootScene.

### Flash Ring

Transient Graphics object that expands and fades:

```js
const ring = scene.add.graphics();
// draw arc with expanding radius + fading alpha
// destroy after ~250ms via scene.tweens
```

### Camera Shake

On hit: `this.cameras.main.shake(200, 0.01)` — subtle 200ms shake for impact.

## AI Logic

Identical algorithm to original, adapted to read from `sprite.x/y` and write to `sprite.setVelocity()`:

1. Threat assessment (predict ball positions 24 frames ahead)
2. Center bias, team spread, enemy vertical split
3. Movement decision (dodge / seek / wander) → `setVelocity()`
4. Throw timing (same cooldowns, `pickTarget`, lead-targeting with jitter)

## HUD

All drawn with Phaser objects refreshed each frame in the `update()` loop:

- **HP bars**: `scene.add.graphics()` rectangles above each character
- **Ball speed controls**: Phaser Text + interactive rectangles for -/+ buttons
- **Hint text**: `scene.add.text()` with fade-out tween
- **Start/GameOver screens**: Phaser container with Text + interactive button rectangles

## Game Flow

- `ready`: show start screen, click start → `gameState = 'playing'`, hide start UI
- `playing`: full game loop, check win/lose on each hit
- `gameover`: show result + restart button, click → `resetGame()`

No Scene transitions needed — all within GameScene, matching original architecture.

## Feature Parity Checklist

All original features preserved:
- [x] 2v2 gameplay (P1 + ally vs 2 AI)
- [x] WASD movement + mouse aim/click throwing
- [x] AI threat assessment, movement, throwing with jitter adaptation
- [x] Obstacles (circle, ellipse, rect, diamond, pentagon with shadows)
- [x] Ball trails with glow
- [x] HP bars with color thresholds
- [x] Death animation + particles
- [x] Hit effects (flash ring + particle burst + camera shake)
- [x] Ball speed +/- controls
- [x] Start/Game over screens
- [x] Hint text fade-out
- [x] Ball wall bouncing with bounce limit