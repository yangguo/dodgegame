# Three.js Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate the vanilla Canvas dodgeball game to Three.js + Cannon.js as a single-file `three.html` with 2.5D top-down rendering, physics-based collision, and 100% feature parity.

**Architecture:** Single HTML file loaded from CDN (three.js + cannon.js). Three.js handles 3D rendering (scene, camera, lights, meshes). Cannon.js handles collision detection and physics response. Game logic (AI, movement, throwing) is custom code ported from the original. HUD is HTML overlay projected from 3D coordinates.

**Tech Stack:** Three.js r170 (CDN), Cannon.js 0.6.2 (CDN), vanilla JS in IIFE.

**Design Spec:** `docs/superpowers/specs/2026-06-05-threejs-migration-design.md`

**Reference:** `vanilla.html` (original, ~1300 lines) — the authoritative source for all game logic.

---

## File Structure

| File | Purpose |
|------|---------|
| `three.html` | Single-file game — HTML/CSS + CDN scripts + all JS (create) |

All code in one IIFE inside `three.html`. No other files needed.

---

## Coordinate System

Original uses 2D canvas (x right, y down, origin top-left). Three.js uses 3D (x right, y up, z toward viewer, origin center).

Mapping used throughout:
```
threeX = canvasX - 400   (canvasX - WIDTH/2)
threeZ = canvasY - 250   (canvasY - HEIGHT/2)
threeY = height above ground (positive up)

canvasX = threeX + 400
canvasY = threeZ + 250
```

Camera: `OrthographicCamera(-400, 400, 250, -250, 0.1, 1000)` at position `(0, 400, 200)` looking at `(0, 0, 0)`.

---

### Task 1: HTML Skeleton + Three.js/Cannon.js Boot

**Files:**
- Create: `three.html`

- [ ] **Step 1: Create `three.html` with HTML structure, CDN scripts, and render loop**

Create the file with complete HTML skeleton, CSS for game container and HUD overlay, CDN script tags, and a minimal IIFE that initializes Three.js renderer, OrthographicCamera, scene, Cannon.js world, and starts the animation loop. Verify by opening in browser — should show 800×500 canvas with light background and start screen overlay visible.

The full code for this step is the entire HTML file with:
- CSS for `#game-container`, `#game-canvas`, HP bars, speed controls, start/gameover screens
- `<script>` tags for three.min.js and cannon.min.js
- IIFE with all constants (WIDTH, HEIGHT, CHAR_RADIUS, speeds, AI constants, colors)
- Three.js renderer setup (WebGLRenderer with shadows)
- OrthographicCamera at (0, 400, 200) looking at origin
- DirectionalLight + AmbientLight
- Cannon.js World (zero gravity, NaiveBroadphase)
- Ground plane with procedural grid texture
- 4 Wall bodies (CANNON.Box, static)
- Game state object
- Stub functions for all game systems (updateP1, updateAlly, updateAIChar, syncMeshes, etc.)
- Animation loop with `requestAnimationFrame`
- All HTML overlay elements (start screen, gameover screen, hint, speed controls)

- [ ] **Step 2: Open `three.html` in browser and verify**

Expected: dark page with "躲避球 · Dodgeball (Three.js)" header, 800×500 canvas showing grid ground, start screen overlay visible, speed controls visible. No console errors.

- [ ] **Step 3: Commit**

```bash
git add three.html
git commit -m "feat: Three.js migration — HTML skeleton with boot, renderer, camera, world"
```

---

### Task 2: Character Factory + Cannon.js Bodies

**Files:**
- Modify: `three.html`

- [ ] **Step 1: Add shared geometries and materials**

Before game state, add reusable geometry/material constants: `charBodyGeo`, `charEyeWhiteGeo`, `charEyePupilGeo`, `charShadowGeo`, `ballGeo`, `particleGeo`, `ROLE_COLORS` map, `ROLE_COLOR_STR` map.

- [ ] **Step 2: Implement `createCharacter()` function**

Creates a Three.js Group (body sphere + 2 eye whites + 2 eye pupils + ground shadow) and a CANNON.KINEMATIC body with CANNON.Sphere shape. Returns an object with `group`, `physBody`, `bodyMesh`, `leftPupil`, `rightPupil`, `shadowMesh`, `role`, `team`, `color`, `hp`, `maxHp`, `throwCooldown`, `facingAngle`, `aiState`, `aiHoldTimer`, `aiWanderAngle`, `targetIndex`, `targetHitRates`, `dyingTimer`, `deathParticles`, `hitFlashTimer`, `hpBar`, `hpBarFill`. All character body materials use `transparent: true, opacity: 1, depthWrite: true` so they can be animated during death without needing material property changes later.

Spawn positions: P1=(110,180), Ally=(110,320), AI1=(690,180), AI2=(690,320) in canvas coords, mapped to Three.js coords (subtract 400 from X, subtract 250 from Y → Z).

- [ ] **Step 3: Add `spawnCharacters()` and `syncMeshes()`**

`spawnCharacters()`: clears old chars, creates 4 characters, populates `teamPlayer` and `teamAI` arrays.
`syncMeshes()`: copies `physBody.position.x/z` to `group.position.x/z`, updates eye pupil positions based on `facingAngle`, updates ball group positions from ball physBody positions, updates ball trail meshes.

- [ ] **Step 4: Verify**

Remove stub `spawnCharacters()` call, add real one. Should see 4 colored spheres with eyes standing on the ground grid. No errors.

- [ ] **Step 5: Commit**

```bash
git add three.html
git commit -m "feat: character factory with 3D models, eyes, shadows, Cannon.js bodies"
```

---

### Task 3: Ball Creation + Throwing + Bouncing

**Files:**
- Modify: `three.html`

- [ ] **Step 1: Implement `createBall()` and `tryThrowToward()`**

`createBall(thrower, dx, dy)`: Creates a Three.js Group with ball sphere mesh (white with emissive team color), Cannon.js dynamic Body (CANNON.Sphere, mass 0.1, restitution 1.0, friction 0), sets velocity based on direction and speed (multiplied by `ballSpeedMultiplier` and 60 for px/sec). Adds collide event listener for bounce counting against wall bodies. Returns ball object with `group`, `mesh`, `physBody`, `owner`, `ownerTeam`, `color`, `bouncesLeft`, `bounceListener`, `shouldRemove`, `trailMeshes`, `trailPositions`.

`tryThrowToward(thrower, targetX, targetY)`: Computes normalized direction from thrower to target, calls `createBall()`, sets thrower cooldown, updates facing angle.

- [ ] **Step 2: Implement `removeDeadBalls()`**

Iterate `balls` in reverse, remove balls where `shouldRemove === true` or `bouncesLeft <= 0` from scene, world, and array. Also call `physBody.removeEventListener('collide', bounceListener)` for cleanup.

- [ ] **Step 3: Verify**

Manually trigger a throw from console. Ball should appear, bounce off walls, disappear after 4 wall bounces.

- [ ] **Step 4: Commit**

```bash
git add three.html
git commit -m "feat: ball creation, throwing, wall bouncing with Cannon.js"
```

---

### Task 4: Obstacle Generation + Collision Groups

**Files:**
- Modify: `three.html`

- [ ] **Step 1: Implement `generateObstacles()` with 5 shape types**

Port obstacle generation logic from original (lines 150-200). For each obstacle, create:
- Three.js mesh: CylinderGeometry for circle/ellipse, BoxGeometry for rect, ExtrudeGeometry for diamond/pentagon. All extruded to height=20 units. Two materials (lighter top, darker sides).
- CANNON.Body: Static, matching shape (CANNON.Cylinder for circle/ellipse, CANNON.Box for rect, CANNON.Cylinder for pentagon, CANNON.Box for diamond).

Apply collision filter groups: `GROUP_OBSTACLE` for obstacle bodies, `GROUP_CHARACTER` as mask (NOT `GROUP_BALL` — balls pass through).

- [ ] **Step 2: Define collision filter groups**

At top of file, define collision group constants and apply to all body types:
- Walls: `GROUP_WALL`, mask = `GROUP_CHARACTER | GROUP_BALL`
- Characters: `GROUP_CHARACTER`, mask = `GROUP_WALL | GROUP_CHARACTER | GROUP_OBSTACLE`
- Balls: `GROUP_BALL`, mask = `GROUP_WALL | GROUP_CHARACTER`
- Obstacles: `GROUP_OBSTACLE`, mask = `GROUP_CHARACTER`

- [ ] **Step 3: Verify**

Start game. Should see 3-5 3D obstacles with shadows. Characters blocked by obstacles, balls pass through.

- [ ] **Step 4: Commit**

```bash
git add three.html
git commit -m "feat: 3D obstacle generation with collision groups (balls pass through)"
```

---

### Task 5: Player Input (WASD + Mouse Aim/Throw) + Screen Logic

**Files:**
- Modify: `three.html`

- [ ] **Step 1: Implement keyboard and mouse input**

Add `keys` object tracking WASD state. Add `mousemove` handler using `Raycaster` to project mouse to ground plane (Y=0). Add `click` handler: if `ready` state → `startGame()`, if `playing` → `tryThrowToward(p1, targetX, targetY)`.

- [ ] **Step 2: Implement `updateP1()`**

Read WASD keys, compute normalized direction, set `p1.physBody.velocity` to `dx * CHAR_SPEED * 60, 0, dz * CHAR_SPEED * 60`. Update `facingAngle` toward `aimTarget`. Decrement `throwCooldown`.

- [ ] **Step 3: Wire start/restart/speed controls**

Add click handlers for start button, restart button, speed +/- buttons. `startGame()`: set `gameState = 'playing'`, hide start screen. `endGame(winner)`: set `gameState = 'gameover'`, show game over screen. `resetGame()`: remove all balls and effects, respawn characters/obstacles, reset state, show start screen.

- [ ] **Step 4: Verify movement**

Open in browser, click start. WASD should move P1. Mouse position should update eye pupil direction. Click should throw a ball. Speed +/- should work.

- [ ] **Step 5: Commit**

```bash
git add three.html
git commit -m "feat: player WASD movement, mouse aim/throw, start/restart/speed controls"
```

---

### Task 6: AI Logic + Ally + Hit Detection

**Files:**
- Modify: `three.html`

- [ ] **Step 1: Port `updateAIChar()` from original**

Port the full AI logic from `vanilla.html` lines 467-547. Key adaptations:
- Read positions from `physBody.position.x/z` (add HW/HH for canvas coords where needed)
- Write velocities via `physBody.velocity.set(vx, 0, vz)`
- All constants (PERP_DODGE_WEIGHT, AI_STATE_HP_RETREAT, etc.) are already defined

`updateAlly()` calls `updateAIChar(ally, delta)`.

- [ ] **Step 2: Implement `onBallHit()` and `setupBallCharCollision()`**

`setupBallCharCollision(ball)`: For each character, add a `collide` event listener on the ball's body. On collision with a character: check team (skip friendly fire), reduce HP, trigger `spawnHitEffect()`, set `shouldRemove` on ball, check death/win conditions.

`onBallHit(ball, char)`: Reduce char HP by 1. Set `hitFlashTimer = 0.2`. Track hit rates on owner. If HP ≤ 0: set `dyingTimer = 0.5`, call `spawnDeathParticles()`, check `allDead()` for win condition.

`allDead(team)`: Check if all characters in team have `hp ≤ 0 && dyingTimer ≤ 0`.

- [ ] **Step 3: Wire `setupBallCharCollision()` into `createBall()`**

Call at the end of ball creation.

- [ ] **Step 4: Verify AI**

Start game. All four characters should move. AI should dodge, throw balls, and respond to threats.

- [ ] **Step 5: Commit**

```bash
git add three.html
git commit -m "feat: AI logic, ally behavior, ball-character hit detection"
```

---

### Task 7: Hit Effects + Death Animation + Camera Shake

**Files:**
- Modify: `three.html`

- [ ] **Step 1: Implement `spawnHitEffect()` and `spawnDeathParticles()`**

Hit effect: 16 particles (12 team color + 4 white) in a radial burst from hit point, flying outward with gravity. Plus an expanding/fading `RingGeometry` flash. Plus camera shake (random offset for 200ms).

Death particles: 12 particles in the character's color, flying outward from character position.

- [ ] **Step 2: Implement `updateEffects()`**

Update all effect particles: move by velocity, apply gravity, fade opacity, scale down. Remove when life <= 0. Update camera shake: offset camera randomly for duration, then restore.

- [ ] **Step 3: Implement `updateDeathAnimations()`**

For characters with `dyingTimer > 0`: scale down group, rotate, fade body mesh opacity. When timer reaches 0, set `group.visible = false`. Also handle `hitFlashTimer`: set body emissive to white during flash, then restore to black.

- [ ] **Step 4: Verify effects**

Throw ball at enemy. On hit: particles fly, ring expands, camera shakes. On death: character shrinks and disappears with particle burst.

- [ ] **Step 5: Commit**

```bash
git add three.html
git commit -m "feat: hit effects, death animation, camera shake"
```

---

### Task 8: HUD (HP Bars, Hint, Ball Trails)

**Files:**
- Modify: `three.html`

- [ ] **Step 1: Implement HP bars as HTML overlay**

Create `<div>` elements for each character in `createCharacter()`. In `updateHUDPositions()`, project 3D character positions to 2D screen coordinates using `Vector3.project(camera)`, position each HP bar above its character. Update fill width and color based on HP percentage (green >60%, yellow >30%, red ≤30%).

- [ ] **Step 2: Implement ball trail effect**

In `createBall()`, create 6 trail meshes (small spheres) added to the ball group. In `syncMeshes()`, maintain `trailPositions` array. On each frame, shift positions and update trail mesh positions/visibility. Trail meshes fade with decreasing opacity and scale.

- [ ] **Step 3: Implement hint text fade-out**

`updateHint(delta)`: Decrement `hintTimer`, fade hint element opacity to 0 when timer expires.

- [ ] **Step 4: Verify HUD**

Start game. HP bars should follow characters. Ball trails visible. Hint fades after ~5 seconds.

- [ ] **Step 5: Commit**

```bash
git add three.html
git commit -m "feat: HP bars, ball trails, hint text fade-out"
```

---

### Task 9: Full Integration + Bug Fix Pass

**Files:**
- Modify: `three.html`

- [ ] **Step 1: Complete game lifecycle**

Ensure `resetGame()` properly clears all balls and effects, removes/respawns characters and obstacles, resets all state variables. Verify `startGame()` hides start screen. Verify `endGame()` shows game over screen with correct message.

- [ ] **Step 2: Fix death animation rendering**

During death animation, when opacity drops below 1, set `bodyMesh.material.depthWrite = false` to prevent transparent-object rendering artifacts. Restore `depthWrite = true` after animation completes (or when resetting). This is needed because character body materials use `transparent: true` (set during creation in Task 2).

- [ ] **Step 3: Verify complete game round**

Play through: start → WASD movement → mouse aim → throw balls → ally & enemy AI active → ball bouncing → ball-character hit → HP loss → hit effects → death animation → game over → restart. All features present and working.

- [ ] **Step 4: Add ball bounce squash animation**

On wall bounce, briefly scale ball mesh Y-axis to 0.7 and back. Track `squashTimer` per ball, update in animation loop.

- [ ] **Step 5: Final commit**

```bash
git add three.html
git commit -m "feat: full Three.js game integration, bug fixes, playtest pass complete"
```