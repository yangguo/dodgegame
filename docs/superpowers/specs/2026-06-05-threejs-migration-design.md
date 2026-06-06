# Three.js Migration Design

**Date:** 2026-06-05
**Status:** Approved

## Goal

Migrate the vanilla Canvas dodgeball game (`vanilla.html`, ~1300 lines) to a Three.js + Cannon.js 3D version (`three.html`), keeping the single-file zero-config (CDN) format and 100% feature parity. The game plays as 2.5D top-down — same WASD+mouse controls — but rendered with 3D models, lighting, shadows, and particles.

## Approach: Three.js 3D Rendering + Cannon.js Physics (Option B)

Use Three.js for rendering (3D meshes, lighting, shadows, particles) and Cannon.js for physics (collision detection, ball bouncing, character-obstacle pushback). Game logic (AI, movement, throwing) remains custom code, matching the original algorithms.

## CDN Dependencies

```html
<script src="https://cdn.jsdelivr.net/npm/three@0.170.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/cannon@0.6.2/build/cannon.min.js"></script>
```

Note: Using original cannon.js (v0.6.2) instead of cannon-es because cannon-es only ships ESM/CJS modules without a browser global build. The original cannon.js has a proper UMD build that exposes `window.CANNON`, which is required for single-file HTML CDN loading. The APIs are functionally identical.

Single file `three.html` with all code in an IIFE.

## Project Structure

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>躲避球 Dodgeball (Three.js)</title>
  <style>/* minimal centering + HUD overlay CSS */</style>
</head>
<body>
  <div id="game-container">
    <canvas id="game-canvas"></canvas>
    <div id="hud"><!-- speed controls, hint text --></div>
    <div id="start-screen"><!-- start overlay --></div>
    <div id="gameover-screen"><!-- game over overlay --></div>
  </div>
  <script src="...three.min.js"></script>
  <script src="...cannon.min.js"></script>
  <script>
  (() => {
    'use strict';
    // All game code
  })();
  </script>
</body>
</html>
```

## Game Configuration

```js
const WIDTH = 800;
const HEIGHT = 500;
const CHAR_RADIUS = 15;        // sphere radius (was 30px diameter / 2)
const BALL_RADIUS = 6;
const CHAR_SPEED = 4;          // px/frame at 60fps → 240 px/sec
const PLAYER_BALL_SPEED = 5;   // px/frame → 300 px/sec
const AI_BALL_SPEED = 4.8;     // px/frame → 288 px/sec
// ... all other constants from original unchanged
```

## Scene & Rendering

### Camera

- `OrthographicCamera` — top-down view with slight isometric tilt for depth perception
- Camera frustum matches 800×500 game area
- No rotation/zoom — fixed viewpoint matching original
- Camera shake on hit: offset camera position for 200ms

### Lighting

- `DirectionalLight` — positioned above and slightly to the side, casts shadows
- `AmbientLight` — soft fill light (`0x404040` intensity)
- Hit flash: brief `PointLight` at impact point, fades over 200ms

### Ground

- `PlaneGeometry(800, 500)` — flat ground plane at y=0
- MeshBasicMaterial with grid texture (procedurally generated via Canvas → `CanvasTexture`)
- Grid matches original: `#f5f5f5` background, `#e2e2e2` grid lines every 50px

### Shadows

- Renderer: `shadowMap.enabled = true`, `shadowMap.type = THREE.PCFSoftShadowMap`
- DirectionalLight shadow camera covers the full 800×500 area
- Characters cast shadows, obstacles cast shadows
- Balls: shadow optional (small, may be noisy) — skip if performance concern

## Characters

### 3D Model (Procedural)

Each character is a group of meshes:

| Part | Geometry | Material | Notes |
|------|----------|----------|-------|
| Body | `SphereGeometry(CHAR_RADIUS)` | `MeshPhongMaterial` with team color | Main body |
| Left Eye White | `SphereGeometry(3)` | `MeshPhongMaterial(0xffffff)` | Front-top of head |
| Left Eye Pupil | `SphereGeometry(1.5)` | `MeshPhongMaterial(0x111111)` | Inside white, tracks facingAngle |
| Right Eye White | `SphereGeometry(3)` | Same | Symmetric |
| Right Eye Pupil | `SphereGeometry(1.5)` | Same | Symmetric |
| Shadow | `CircleGeometry(CHAR_RADIUS)` | `MeshBasicMaterial(0x000000, transparent, opacity 0.15)` | On ground, y=0.01 |

Character colors (from original):

| Role | Color (hex) | Team |
|------|------------|------|
| p1 | 0x4A90E2 (blue) | player |
| ally | 0x3DC1B5 (teal) | player |
| ai1 | 0xE24A4A (red) | ai |
| ai2 | 0xE89B3D (orange) | ai |

### Eye Tracking

Pupils offset from eye center in the direction of `facingAngle`:

```js
const pupilOffset = 1.5;
const px = Math.cos(facingAngle) * pupilOffset;
const pz = Math.sin(facingAngle) * pupilOffset;
leftPupil.position.set(leftEyeX + px, eyeY, leftEyeZ + pz);
rightPupil.position.set(rightEyeX + px, eyeY, rightEyeZ + pz);
```

### Cannnon.js Body

```js
const charBody = new CANNON.Body({
  mass: 1,
  shape: new CANNON.Sphere(CHAR_RADIUS),
  linearDamping: 0.99,  // slight friction
});
charBody.type = CANNON.Body.KINEMATIC; // velocity set by code each frame
```

Characters use kinematic bodies — velocity is set each frame by game logic (player input or AI), not driven by physics forces. Cannon.js handles collision response (pushback from obstacles) automatically.

### Hit Flash

On hit, character body material `emissive` set to `0xffffff`, then tween back to `0x000000` over 200ms.

### Death Animation

Three phases (matching original):
1. **Shrink** (frames 0-15): body scales from 1.0 → 0.3, rotation speeds up
2. **Particle burst** (frame 15): spawn 12-16 small sphere particles flying outward
3. **Fade** (frames 15-30): opacity 1.0 → 0.0, then remove from scene

## Balls

### 3D Model

- `SphereGeometry(BALL_RADIUS)` with `MeshPhongMaterial`
- Team-colored emissive glow (player team: blue/teal tint, AI team: red/orange tint)
- White base color with emissive team color

### Trail Effect

Maintain last N frames of ball positions. Render as:
- 6-8 `Mesh` objects (small spheres) at previous positions
- Progressive fade: alpha 0.5 → 0.0, scale 1.0 → 0.3
- Or use `Points` system for better performance

### Bounce Squash

On wall bounce, briefly scale ball:
- Y-axis compressed to 0.7, then spring back (100ms)
- Simulated using mesh.scale animation

### Cannon.js Body

```js
const ballBody = new CANNON.Body({
  mass: 0.1,
  shape: new CANNON.Sphere(BALL_RADIUS),
  linearDamping: 0,
});
ballBody.restitution = 1.0;  // perfect bounce
ballBody.friction = 0;
```

Balls are dynamic — Cannon.js handles wall bounces and movement. Initial velocity set by `tryThrowToward()`.

### Bounce Counting

Listen to `ballBody.addEventListener('collide', ...)` events. Count contacts with wall bodies. After 4 wall bounces, mark for removal.

Alternatively: maintain a boundary contact counter per ball. Walls are 4 static `CANNON.Body` planes. On each contact between ball and wall body, increment counter. At 4, destroy ball.

## Obstacles

### 3D Model

Each obstacle shape is extruded from 2D profile to 3D:

| Original Shape | 3D Geometry | Height |
|---------------|-------------|--------|
| Circle (r=18) | `CylinderGeometry(18, 18, 20)` | 20 units tall |
| Ellipse | `CylinderGeometry` with scale | 20 units tall |
| Rectangle | `BoxGeometry(w, 20, h)` | 20 units tall |
| Diamond | Custom `ExtrudeGeometry` from diamond shape | 20 units tall |
| Pentagon | Custom `ExtrudeGeometry` from pentagon shape | 20 units tall |

Material:
- Top face: `MeshPhongMaterial({ color: COLOR_OBSTACLE_HI })` — lighter brown
- Side faces: `MeshPhongMaterial({ color: COLOR_OBSTACLE })` — darker brown
- Casts shadows

### Cannon.js Body

Static bodies matching the mesh shape:
- Circle: `CANNON.Cylinder(18, 18, 20)` (or `CANNON.Sphere(18)` for 2D collision)
- Others: `CANNON.Box(halfExtents)` or `CANNON.ConvexPolyhedron`

Since this is 2.5D (top-down), obstacles only need to block XZ movement. A simple approach: use `CANNON.Box` or `CANNON.Cylinder` for all shapes, matching their footprint.

### Generation

Same random seed logic as original. On each game reset:
1. Remove old obstacle meshes + bodies from scene/world
2. Generate new random set (3-5 obstacles)
3. Create Three.js meshes + Cannon.js bodies
4. Add to scene and physics world

## Collision Detection

### Ball vs Character (Hit)

Cannon.js collision event between ball body and character body. Filter by team:

```js
world.addEventListener('endContact', ...);  // or per-body 'collide'
// Check: ball.ownerTeam !== character.team
// Reduce HP, trigger hit effect
// Destroy ball
```

In `onBallHit(ball, character)`:
1. Verify ball team ≠ character team (friendly fire disabled)
2. Reduce character HP by 1
3. Trigger hit effect (particles + flash + camera shake)
4. Update HP bar
5. If HP ≤ 0: start death animation, check win condition
6. Destroy ball

### Character vs Obstacle

Cannon.js automatic — kinematic character bodies collide with static obstacle bodies. Characters get pushed back naturally.

### Ball vs Obstacle

Balls pass through obstacles (same as original). To achieve this:
- Set collision groups: balls and obstacles are in different groups
- Or manually skip obstacle contacts in the ball's collision handler

### Ball vs Wall

Wall bodies bound the arena (4 planes). Balls bounce off with `restitution: 1.0`. Each wall contact increments `bouncesLeft`. At 0, ball is destroyed.

## Hit Effects

### Particle Burst

On ball hit character:
```js
// Create particle group at hit position
const particles = [];
for (let i = 0; i < 16; i++) {
  const angle = (Math.PI * 2 / 16) * i + Math.random() * 0.3;
  const speed = 80 + Math.random() * 170; // px/sec
  const p = {
    mesh: new THREE.Mesh(particleGeo, material),
    vx: Math.cos(angle) * speed,
    vz: Math.sin(angle) * speed,
    vy: 60 + Math.random() * 100, // slight upward arc
    life: 0.4, // seconds
    maxLife: 0.4,
  };
  particles.push(p);
  scene.add(p.mesh);
}
```

Use `MeshBasicMaterial` with team color for 12 particles + white for 4 particles.

### Flash Ring

Expanding ring at hit point:
```js
const ring = new THREE.Mesh(
  new THREE.RingGeometry(0, CHAR_RADIUS * 2, 32),
  new THREE.MeshBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0.8 })
);
// Scale up + fade out over 250ms
```

### Camera Shake

```js
// Offset camera position randomly for 200ms
const shakeOffset = new THREE.Vector3(
  (Math.random() - 0.5) * 4,
  0,
  (Math.random() - 0.5) * 4
);
// Tween back to original position
```

## AI Logic

Identical algorithm to original, with adaptations:

1. Ball positions read from `ball.mesh.position` / `ballBody.position`
2. Character positions read from `charBody.position`
3. AI velocity set via `charBody.velocity.set(vx, 0, vz)`
4. Throw direction computed same way, ball body initialized with `ballBody.velocity.set(dx * speed, 0, dz * speed)`

No physics-driven AI movement — all AI decisions produce direct velocity values, matching original behavior.

## HUD (HTML Overlay)

All HUD elements are HTML elements overlaid on the 3D canvas:

### HP Bars

Position HP bars above characters by projecting 3D position to 2D:

```js
const vec = new THREE.Vector3(char.x, char.y + 20, char.z);
vec.project(camera);
const screenX = (vec.x * 0.5 + 0.5) * canvas.clientWidth;
const screenY = (-vec.y * 0.5 + 0.5) * canvas.clientHeight;
hpBarElement.style.left = screenX + 'px';
hpBarElement.style.top = screenY + 'px';
```

Each HP bar is a `<div>` with colored fill matching original thresholds (green > 60%, yellow > 30%, red ≤ 30%).

### Speed Controls

HTML buttons positioned in top-right corner, matching original:
- `[-]` button to decrease speed
- `[+]` button to increase speed
- Speed multiplier display text

### Hint Text

HTML `<span>` with CSS `opacity` transition for fade-out.

### Start Screen

HTML overlay with semi-transparent background, title, instructions, and "点击开始" button.

### Game Over Screen

HTML overlay with result text and "再来一局" button.

## Mouse Input

### Raycasting for Aim

```js
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

canvas.addEventListener('mousemove', (e) => {
  const rect = canvas.getBoundingClientRect();
  mouse.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
  mouse.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
  raycaster.setFromCamera(mouse, camera);
  // Intersect with ground plane (y=0)
  const groundPlane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0);
  const target = new THREE.Vector3();
  raycaster.ray.intersectPlane(groundPlane, target);
  // target.x, target.z = aim position in game coordinates
});
```

### Click to Throw

```js
canvas.addEventListener('click', (e) => {
  if (gameState === 'playing') tryThrowToward(p1, aimTarget.x, aimTarget.z);
  if (gameState === 'ready') startGame();
  // etc.
});
```

### Start/Restart Buttons

Start and gameover clicks handled via HTML button click events (not raycasting). The 3D canvas handles aiming and throwing; HTML overlays handle UI buttons.

## Game Flow

Identical to original:

| State | Behavior |
|-------|----------|
| `ready` | Show start screen overlay. Click start → `gameState = 'playing'`, hide overlay. |
| `playing` | Full game loop. Update P1 input, ally AI, enemy AI. Check collisions via Cannon.js. Trigger effects. |
| `gameover` | Show game over overlay with result. Click restart → `resetGame()`. |

No Three.js scene transitions — single scene, state managed by variable.

## Coordinate System Mapping

Original uses 2D canvas coordinates (x right, y down, origin top-left).
Three.js uses 3D (x right, y up, z toward camera, origin center).

Mapping:
```
threeX = canvasX - WIDTH/2
threeZ = canvasY - HEIGHT/2
threeY = 0 (ground) or +height (elevated)

canvasX = threeX + WIDTH/2
canvasY = threeZ + HEIGHT/2
```

Camera: `OrthographicCamera(left=-400, right=400, top=250, bottom=-250, near=0.1, far=1000)`
Position: `camera.position.set(0, 500, 0)` looking down at `camera.lookAt(0, 0, 0)`

Or slight tilt for depth: `camera.position.set(0, 400, 200)` with `lookAt(0, 0, 0)` — adjustable.

## Animation Loop

```js
const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  const delta = clock.getDelta(); // seconds since last frame

  if (gameState === 'playing') {
    updateP1(delta);
    updateAlly(delta);
    for (const ai of teamAI) updateAIChar(ai, delta);

    // Cannon.js step
    world.step(1/60, delta, 3);

    // Sync Three.js meshes to Cannon.js bodies
    syncMeshes();

    updateBallObstacleCollisions();
    updateEffects(delta);
    updateDeathAnimations(delta);
    removeDeadBalls();
  }

  // Update HP bar positions
  updateHUDPositions();

  renderer.render(scene, camera);
}
```

## Feature Parity Checklist

All original features preserved:

- [x] 2v2 gameplay (P1 + ally vs 2 AI)
- [x] WASD movement + mouse aim/click throwing
- [x] AI threat assessment, movement, throwing with jitter adaptation
- [x] Obstacles (circle, ellipse, rect, diamond, pentagon) — 3D extruded
- [x] Ball trails with glow — mesh-based or Points
- [x] HP bars with color thresholds — HTML overlay projected from 3D
- [x] Death animation (shrink + particles) — 3D version
- [x] Hit effects (particle burst + flash ring + camera shake)
- [x] Ball speed +/- controls — HTML buttons
- [x] Start/Game over screens — HTML overlays
- [x] Hint text fade-out — HTML overlay
- [x] Ball wall bouncing with bounce limit — Cannon.js `restitution: 1.0` + bounce counter
- [x] Speed multiplier (0.5×–3×) — applied to ball velocity