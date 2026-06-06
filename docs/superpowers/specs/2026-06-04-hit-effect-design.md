# Hit Effect Design: Screen Flash + Particle Explosion

**Date:** 2026-06-04
**Status:** Approved

## Problem

Hits in the dodgeball game have almost no visual feedback — only the HP bar changes.
Death has a particle effect, but ordinary hits feel weightless.

## Solution

Add a global `effects[]` array to `state` and create a `HitEffect` on every ball-character collision.

Each `HitEffect` contains:

1. **Expanding ring** — a white semi-transparent circle that grows from the hit point outward (~120px radius) while fading, creating a shockwave feel. Duration: ~15 frames, starting alpha 0.6, radius grows from 0 to `ringMaxRadius`.

2. **Particle burst** — 14-18 small circle particles flying outward from the hit point. Colors come from the hit character's `color` property, plus 3-4 white highlight particles. Each particle has random angle, speed (2-6), size (2-5px), and a slight downward gravity (0.1/frame). Duration: 25 frames with linear alpha fade.

### Data Structure

```js
// In state
effects: []

// HitEffect object
{
  x: Number,           // hit point x
  y: Number,           // hit point y
  color: String,       // hit character's color (hex)
  ringRadius: Number,  // current ring radius, starts at 0
  ringAlpha: Number,   // current ring alpha, starts at 0.6
  ringMaxRadius: 120,  // max radius the ring expands to
  particles: [{
    x: Number, y: Number,
    vx: Number, vy: Number,
    size: Number,
    life: Number,
    maxLife: Number,
    color: String
  }],
  age: Number,        // frames elapsed
  maxAge: 25          // total duration in frames
}
```

### Where Effects Are Created

In `checkCollisions()`, when `circleHitsCharacter(b, target)` is true (before `balls.splice`):

```js
spawnHitEffect(target.x, target.y, target.color);
```

### Effect Lifecycle

- **Update**: `updateEffects()` called in `update()` after collision detection
  - Ring: radius grows linearly, alpha fades
  - Particles: position += velocity, vy += 0.1 (gravity), life decreases
  - Remove effect when age >= maxAge

- **Render**: `drawEffects()` called in `render()` after drawBalls, before UI
  - Ring: `ctx.arc` with `globalAlpha`
  - Particles: `ctx.arc` with `globalAlpha` based on life/maxLife

### Files Modified

Only `vanilla.html` — all changes are additions within the existing IIFE:

1. Add `effects: []` to state object (~line 106)
2. Add `spawnHitEffect(x, y, color)` function (after `checkCollisions`)
3. Add `updateEffects()` function
4. Add `drawEffects()` function
5. Call `spawnHitEffect()` in `checkCollisions()` ~line 793
6. Call `updateEffects()` in `update()` ~line 841
7. Call `drawEffects()` in `render()` ~line 1163

### Constants

No new top-level constants needed — tuning values (ring max radius, particle count, durations) are inline in `spawnHitEffect()`.