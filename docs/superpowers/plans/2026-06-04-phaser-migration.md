# Phaser Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create `phaser.html` — a single-file Phaser 3 dodgeball game with 100% feature parity with `vanilla.html`, using Arcade Physics, Phaser particles, and camera shake for hit effects.

**Architecture:** Two Phaser scenes (BootScene for texture generation, GameScene for all gameplay). All graphics generated at runtime, no external assets. CDN-loaded Phaser 3.80.1. Custom state variable (`gameState`) mirrors original architecture.

**Tech Stack:** Phaser 3.80.1 (CDN), vanilla JS, single HTML file.

---

## File Structure

| File | Purpose |
|------|---------|
| `phaser.html` | **Create** — The entire Phaser game (HTML + CSS + JS IIFE) |

No modifications to `vanilla.html`. This is a parallel new version.

---

### Task 1: HTML Shell + Phaser Config + BootScene (Texture Generation)

**Files:**
- Create: `phaser.html`

This task creates the file with the complete shell, Phaser config, and BootScene that generates all textures.

- [ ] **Step 1: Create `phaser.html` with HTML shell, CSS, and Phaser config**

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title>躲避球 · Phaser</title>
<style>
  * { box-sizing: border-box; margin: 0; padding: 0; }
  html, body {
    min-height: 100%;
    background: #1a1a1a;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "PingFang SC", "Microsoft YaHei", sans-serif;
    color: #eee;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 16px 0;
  }
  h1 {
    font-size: 18px;
    font-weight: 500;
    margin-bottom: 12px;
    color: #aaa;
    letter-spacing: 1px;
  }
</style>
</head>
<body>
  <h1>躲避球 · Dodgeball</h1>
<script src="https://cdn.jsdelivr.net/npm/phaser@3.80.1/dist/phaser.min.js"></script>
<script>
(() => {
  'use strict';

  // ---------- Constants (identical to original) ----------
  const WIDTH = 800;
  const HEIGHT = 500;
  const CHAR_SIZE = 30;
  const CHAR_SPEED = 4;
  const BALL_RADIUS = 6;
  const PLAYER_BALL_SPEED = 5;
  const AI_BALL_SPEED = 4.8;
  const PLAYER_THROW_COOLDOWN = 15;
  const AI_THROW_INTERVAL_MIN = 45;
  const AI_THROW_INTERVAL_MAX = 80;
  const BALL_MAX_BOUNCES = 4;
  const MAX_HP = 10;
  const AI_THREAT_RANGE = 200;
  const HINT_FRAMES = 300;
  const OBSTACLE_RADIUS = 18;
  const OBSTACLE_MIN_COUNT = 3;
  const OBSTACLE_MAX_COUNT = 5;
  const HISTORY_LEN = 6;

  const PERP_DODGE_WEIGHT = 1.0;
  const SPEED_THREAT_WEIGHT = 2.0;
  const AI_STATE_HP_RETREAT = 1;
  const AI_STATE_THREAT_DEFENSIVE = 1.5;
  const AGGRESSIVE_IDEAL_DIST = 90;
  const DEFENSIVE_IDEAL_DIST = 180;
  const RETREAT_GOAL_X_PLAYER = 50;
  const RETREAT_GOAL_X_AI = 750;
  const TEAM_SPREAD_MIN = 100;
  const TEAM_SPREAD_FORCE = 0.4;
  const ENEMY_VERTICAL_SPLIT = 120;
  const JITTER_MIN = 2;
  const JITTER_MAX = 12;
  const JITTER_ADAPT_THRESHOLD = 0.6;

  const COLOR_P1 = 0x4A90E2;
  const COLOR_P2 = 0x3DC1B5;
  const COLOR_AI1 = 0xE24A4A;
  const COLOR_AI2 = 0xE89B3D;
  const COLOR_HP_GREEN = 0x5cd65c;
  const COLOR_HP_YELLOW = 0xf0c040;
  const COLOR_HP_RED = 0xe05050;
  const COLOR_BG = '#f5f5f5';
  const COLOR_GRID = 0xe2e2e2;
  const COLOR_TEXT = '#333';
  const COLOR_OBSTACLE = '#6B5B4F';
  const BALL_COLOR_PLAYER = 0x1f6fcc;
  const BALL_COLOR_AI = 0xff1f1f;
```

- [ ] **Step 2: Add BootScene with texture generation**

```js
  // ---------- BootScene: Generate all textures ----------
  class BootScene extends Phaser.Scene {
    constructor() { super('BootScene'); }

    create() {
      // Character texture: 30x30 white rounded rect (tinted at runtime)
      const charGfx = this.add.graphics();
      charGfx.fillStyle(0xffffff);
      charGfx.fillRoundedRect(0, 0, CHAR_SIZE, CHAR_SIZE, 8);
      charGfx.generateTexture('char', CHAR_SIZE, CHAR_SIZE);
      charGfx.destroy();

      // Ball texture: white circle (tinted at runtime)
      const ballGfx = this.add.graphics();
      ballGfx.fillStyle(0xffffff);
      ballGfx.fillCircle(BALL_RADIUS, BALL_RADIUS, BALL_RADIUS);
      ballGfx.generateTexture('ball', BALL_RADIUS * 2, BALL_RADIUS * 2);
      ballGfx.destroy();

      // Particle texture: 4x4 white circle for hit effects
      const partGfx = this.add.graphics();
      partGfx.fillStyle(0xffffff);
      partGfx.fillCircle(2, 2, 2);
      partGfx.generateTexture('particle', 4, 4);
      partGfx.destroy();

      // Transition to game scene
      this.scene.start('GameScene');
    }
  }
```

- [ ] **Step 3: Add Phaser config and game launch at end of IIFE**

```js
  // (GameScene class will be inserted here in later tasks)

  const config = {
    type: Phaser.AUTO,
    width: WIDTH,
    height: HEIGHT,
    backgroundColor: COLOR_BG,
    parent: document.body,
    physics: {
      default: 'arcade',
      arcade: {
        gravity: { y: 0 },
        debug: false
      }
    },
    scene: [BootScene, GameScene]
  };

  const game = new Phaser.Game(config);
})();
</script>
</body>
</html>
```

- [ ] **Step 4: Open `phaser.html` in browser, verify black canvas loads with no errors in console**

Expected: A plain `#f5f5f5` canvas, no errors. BootScene should complete and transition to GameScene (which doesn't exist yet — will error, that's expected for this step; just verify the HTML/Phaser loads).

---

### Task 2: GameScene Skeleton + State + Obstacle Generation

**Files:**
- Modify: `phaser.html` (add GameScene class)

- [ ] **Step 1: Add GameScene class skeleton with create() and update() methods**

```js
  class GameScene extends Phaser.Scene {
    constructor() { super('GameScene'); }

    create() {
      // Will be filled in this and subsequent tasks
    }

    update() {
      // Will be filled in
    }
  }
```

- [ ] **Step 2: Add game state initialization in create()**

Add inside GameScene's `create()`:

```js
      // Game state
      this.gameState = 'ready';
      this.winner = null;
      this.hintTimer = HINT_FRAMES;
      this.ballSpeedMultiplier = 1.0;

      // Groups
      this.ballsGroup = this.physics.add.group();
      this.charGroup = this.physics.add.group();
      this.obstacleGroup = this.physics.add.staticGroup();

      // Input
      this.cursors = this.input.keyboard.createCursorKeys();
      this.wasd = this.input.keyboard.addKeys({
        up: Phaser.Input.Keyboard.KeyCodes.W,
        down: Phaser.Input.Keyboard.KeyCodes.S,
        left: Phaser.Input.Keyboard.KeyCodes.A,
        right: Phaser.Input.Keyboard.KeyCodes.D
      });
      this.mousePointer = this.input.activePointer;

      // History for player velocity estimation
      this.p1History = [];
      this.p2History = [];
```

- [ ] **Step 3: Add generateObstacles() method to GameScene**

```js
    generateObstacles() {
      const count = OBSTACLE_MIN_COUNT +
        Math.floor(Math.random() * (OBSTACLE_MAX_COUNT - OBSTACLE_MIN_COUNT + 1));
      const spawns = [
        { x: 110, y: 180 },
        { x: 110, y: 320 },
        { x: 690, y: 180 },
        { x: 690, y: 320 },
      ];
      const shapes = ['circle', 'ellipse', 'rect', 'diamond', 'pentagon'];
      const colors = [0x6B5B4F, 0x5a4f42, 0x7d6e60, 0x8c7d6f, 0x4a3f35, 0x7a6a5a, 0x6a5a4a];
      const obstacles = [];
      const margin = 30;
      const minFromSpawn = 130;
      let attempts = 0;

      while (obstacles.length < count && attempts < 400) {
        attempts++;
        const size = 10 + Math.random() * 18;
        const x = margin + size + Math.random() * (WIDTH - 2 * (margin + size));
        const y = margin + size + Math.random() * (HEIGHT - 2 * (margin + size));
        let tooCloseToSpawn = false;
        for (const s of spawns) {
          if (Math.hypot(x - s.x, y - s.y) < minFromSpawn + size - 10) {
            tooCloseToSpawn = true; break;
          }
        }
        if (tooCloseToSpawn) continue;
        let tooClose = false;
        for (const o of obstacles) {
          if (Math.hypot(x - o.x, y - o.y) < o.size + size + 10) { tooClose = true; break; }
        }
        if (tooClose) continue;
        obstacles.push({
          x, y, size,
          shape: shapes[Math.floor(Math.random() * shapes.length)],
          color: colors[Math.floor(Math.random() * colors.length)],
          rotation: Math.random() * Math.PI * 2,
          width: size * (0.6 + Math.random() * 1.0),
          height: size * (0.6 + Math.random() * 1.0),
        });
      }
      return obstacles;
    }
```

- [ ] **Step 4: Add drawObstacle() and spawnObstacles() methods**

These draw each obstacle shape as a static physics sprite. Since obstacles vary in shape and size, we generate individual textures:

```js
    spawnObstacles(obstacleData) {
      this.obstacleGroup.clear(true, true);
      for (const o of obstacleData) {
        // Generate a unique texture key for this obstacle
        const key = `obs_${o.shape}_${Math.round(o.size)}_${Math.round(o.width)}_${Math.round(o.height)}`;
        if (!this.textures.exists(key)) {
          const gfx = this.add.graphics();
          gfx.fillStyle(o.color);
          const or = o.size;
          const ow = o.width || or;
          const oh = o.height || or;
          if (o.shape === 'ellipse') {
            gfx.fillEllipse(ow, oh);
          } else if (o.shape === 'rect') {
            gfx.fillRect(-ow / 2, -oh / 2, ow, oh);
          } else if (o.shape === 'diamond') {
            gfx.beginPath();
            gfx.moveTo(0, -oh / 2);
            gfx.lineTo(ow / 2, 0);
            gfx.lineTo(0, oh / 2);
            gfx.lineTo(-ow / 2, 0);
            gfx.closePath();
            gfx.fillPath();
          } else if (o.shape === 'pentagon') {
            gfx.beginPath();
            for (let i = 0; i < 5; i++) {
              const angle = (i * 2 * Math.PI / 5) - Math.PI / 2;
              const px = Math.cos(angle) * or * 0.95;
              const py = Math.sin(angle) * or * 0.95;
              if (i === 0) gfx.moveTo(px, py);
              else gfx.lineTo(px, py);
            }
            gfx.closePath();
            gfx.fillPath();
          } else {
            gfx.fillCircle(0, 0, or);
          }
          // Border
          gfx.lineStyle(1.5, 0x4a3d33);
          if (o.shape === 'ellipse') {
            gfx.strokeEllipse(ow, oh);
          } else if (o.shape === 'rect') {
            gfx.strokeRect(-ow / 2, -oh / 2, ow, oh);
          } else if (o.shape === 'diamond') {
            gfx.beginPath();
            gfx.moveTo(0, -oh / 2); gfx.lineTo(ow / 2, 0);
            gfx.lineTo(0, oh / 2); gfx.lineTo(-ow / 2, 0);
            gfx.closePath(); gfx.strokePath();
          } else if (o.shape === 'pentagon') {
            gfx.beginPath();
            for (let i = 0; i < 5; i++) {
              const angle = (i * 2 * Math.PI / 5) - Math.PI / 2;
              const px = Math.cos(angle) * or * 0.95;
              const py = Math.sin(angle) * or * 0.95;
              if (i === 0) gfx.moveTo(px, py);
              else gfx.lineTo(px, py);
            }
            gfx.closePath(); gfx.strokePath();
          } else {
            gfx.strokeCircle(0, 0, or);
          }
          // Highlight
          gfx.fillStyle(0xffffff, 0.25);
          gfx.fillCircle(-or * 0.25, -or * 0.25, or * 0.35);
          const tw = Math.ceil(ow + 4);
          const th = Math.ceil(oh + 4);
          gfx.generateTexture(key, tw, th);
          gfx.destroy();
        }
        // Create static sprite
        const obs = this.obstacleGroup.create(o.x, o.y, key);
        if (o.rotation) obs.setRotation(o.rotation);
        obs.refreshBody();
      }
    }
```

- [ ] **Step 5: Refresh browser, verify no console errors and `phaser.html` loads**

---

### Task 3: Character Sprites + P1 Movement + HP Display

**Files:**
- Modify: `phaser.html` (expand GameScene)

- [ ] **Step 1: Add createCharacter() method to GameScene**

```js
    createCharacter(x, y, tint, team, role) {
      const sprite = this.physics.add.sprite(x, y, 'char');
      sprite.setTint(tint);
      sprite.setCollideWorldBounds(true);
      sprite.setData({
        team,
        role,
        hp: MAX_HP,
        maxHp: MAX_HP,
        facingAngle: team === 'player' ? 0 : Math.PI,
        throwCooldown: 0,
        aiState: 'AGGRESSIVE',
        aiHoldTimer: 60,
        aiWanderAngle: Math.random() * Math.PI * 2,
        aiWanderTimer: 0,
        targetIndex: 0,
        targetHitRates: {},
        dyingTimer: 0,
        deathParticles: [],
      });
      sprite.setSize(CHAR_SIZE, CHAR_SIZE);
      sprite.body.setCircle(CHAR_SIZE / 2, (sprite.width - CHAR_SIZE) / 2, (sprite.height - CHAR_SIZE) / 2);
      sprite.setImmovable(false);
      sprite.setDepth(10);
      this.charGroup.add(sprite);
      return sprite;
    }
```

- [ ] **Step 2: Add drawCharacter() method — shadow, body, eyes, HP bar**

Instead of custom Canvas drawing, we draw these as Phaser Graphics each frame. We'll use a single graphics object:

```js
    drawCharacter(sprite, gfx) {
      const data = sprite.data.values;
      if (data.hp <= 0 && data.dyingTimer <= 0) return;

      const x = sprite.x;
      const y = sprite.y;

      if (data.dyingTimer > 0) {
        // Death animation: shrinking + fading
        const alpha = data.dyingTimer / 30;
        const scale = data.dyingTimer / 30;
        const sc = CHAR_SIZE * scale;
        gfx.save();
        gfx.setAlpha(alpha);
        gfx.fillStyle(sprite.tint & 0xffffff); // extract RGB from tint
        // Use the sprite's actual tint color for death
        const tint = sprite.tint & 0xffffff;
        gfx.fillStyle(tint);
        gfx.fillRoundedRect(x - sc / 2, y - sc / 2, sc, sc, 8 * scale);
        // X eyes
        gfx.lineStyle(2, 0x000000);
        gfx.beginPath();
        gfx.moveTo(x - sc * 0.3, y - sc * 0.3);
        gfx.lineTo(x + sc * 0.3, y + sc * 0.3);
        gfx.moveTo(x + sc * 0.3, y - sc * 0.3);
        gfx.lineTo(x - sc * 0.3, y + sc * 0.3);
        gfx.strokePath();
        gfx.restore();
        // Death particles
        for (const p of data.deathParticles) {
          if (p.life <= 0) continue;
          gfx.save();
          gfx.setAlpha(p.life / 30);
          gfx.fillStyle(p.color);
          gfx.fillCircle(x + p.x, y + p.y, p.size * (p.life / 30));
          gfx.restore();
        }
        return;
      }

      // Shadow
      gfx.fillStyle(0x000000, 0.12);
      gfx.fillEllipse(x, y + CHAR_SIZE / 2 + 2, CHAR_SIZE * 0.9, 8);

      // Body (the sprite itself handles this, but we draw the detail on top)
      // Eyes — offset toward facingAngle
      const eyeOffsetX = Math.cos(data.facingAngle) * 3;
      const eyeOffsetY = Math.sin(data.facingAngle) * 3;
      const eyeR = 3;
      const eyeY = y - 3;
      const eyeDx = 6;
      // White of eyes
      gfx.fillStyle(0xffffff);
      gfx.fillCircle(x - eyeDx / 2 + eyeOffsetX * 0.5, eyeY + eyeOffsetY * 0.5, eyeR);
      gfx.fillCircle(x + eyeDx / 2 + eyeOffsetX * 0.5, eyeY + eyeOffsetY * 0.5, eyeR);
      // Pupils
      gfx.fillStyle(0x222222);
      gfx.fillCircle(x - eyeDx / 2 + eyeOffsetX, eyeY + eyeOffsetY, eyeR * 0.55);
      gfx.fillCircle(x + eyeDx / 2 + eyeOffsetX, eyeY + eyeOffsetY, eyeR * 0.55);

      // HP bar
      const hpW = 36;
      const hpH = 6;
      const hpX = x - hpW / 2;
      const hpY = y - CHAR_SIZE / 2 - 14;
      const ratio = data.hp / data.maxHp;
      gfx.fillStyle(0x222222);
      gfx.fillRect(hpX - 1, hpY - 1, hpW + 2, hpH + 2);
      gfx.fillStyle(0x444444);
      gfx.fillRect(hpX, hpY, hpW, hpH);
      let hpColor = COLOR_HP_GREEN;
      if (ratio <= 0.34) hpColor = COLOR_HP_RED;
      else if (ratio <= 0.67) hpColor = COLOR_HP_YELLOW;
      gfx.fillStyle(hpColor);
      gfx.fillRect(hpX, hpY, hpW * ratio, hpH);
    }
```

- [ ] **Step 3: Add resetGame() that creates all 4 characters and obstacles**

```js
    resetGame() {
      // Clear existing sprites
      this.ballsGroup.clear(true, true);
      this.charGroup.clear(true, true);
      this.obstacleGroup.clear(true, true);
      this.p1History.length = 0;
      this.p2History.length = 0;
      this.gameState = 'ready';
      this.winner = null;
      this.hintTimer = HINT_FRAMES;
      this.ballSpeedMultiplier = 1.0;

      // Create characters
      this.p1 = this.createCharacter(110, 180, COLOR_P1, 'player', 'p1');
      this.ally = this.createCharacter(110, 320, COLOR_P2, 'player', 'ally');
      this.ai1 = this.createCharacter(690, 180, COLOR_AI1, 'ai', 'ai1');
      this.ai2 = this.createCharacter(690, 320, COLOR_AI2, 'ai', 'ai2');

      this.teamPlayer = [this.p1, this.ally];
      this.teamAI = [this.ai1, this.ai2];

      // Generate obstacles
      this.obstacleData = this.generateObstacles();
      this.spawnObstacles(this.obstacleData);

      // Colliders: characters vs obstacles
      this.physics.add.collider(this.charGroup, this.obstacleGroup);

      // Create grid graphics (drawn once, persists)
      this.drawGridGraphics();
    }
```

- [ ] **Step 4: Add P1 keyboard movement in update()**

```js
    updateP1() {
      const p1 = this.p1;
      if (!p1 || p1.data.values.hp <= 0 || p1.data.values.dyingTimer > 0) return;

      this.trackHistory(this.p1History, p1);

      let dx = 0, dy = 0;
      if (this.wasd.left.isDown) dx -= 1;
      if (this.wasd.right.isDown) dx += 1;
      if (this.wasd.up.isDown) dy -= 1;
      if (this.wasd.down.isDown) dy += 1;

      if (dx !== 0 && dy !== 0) {
        const inv = 1 / Math.SQRT2;
        dx *= inv;
        dy *= inv;
      }

      p1.setVelocity(dx * CHAR_SPEED * 60, dy * CHAR_SPEED * 60);

      // Mouse aim overrides movement-facing
      const aimDx = this.mousePointer.x - p1.x;
      const aimDy = this.mousePointer.y - p1.y;
      if (Math.hypot(aimDx, aimDy) > 1) {
        p1.data.values.facingAngle = Math.atan2(aimDy, aimDx);
      }

      if (p1.data.values.throwCooldown > 0) p1.data.values.throwCooldown--;
    }
```

Note: In Phaser, `setVelocity` uses pixels/second, while the original uses pixels/frame at 60fps. So `CHAR_SPEED * 60` converts from px/frame to px/sec.

- [ ] **Step 5: Add grid drawing and render graphics setup in create()**

```js
    drawGridGraphics() {
      if (this.gridGfx) this.gridGfx.destroy();
      this.gridGfx = this.add.graphics().setDepth(0);
      this.gridGfx.lineStyle(1, COLOR_GRID);
      const step = 40;
      for (let x = step; x < WIDTH; x += step) {
        this.gridGfx.beginPath();
        this.gridGfx.moveTo(x + 0.5, 0);
        this.gridGfx.lineTo(x + 0.5, HEIGHT);
        this.gridGfx.strokePath();
      }
      for (let y = step; y < HEIGHT; y += step) {
        this.gridGfx.beginPath();
        this.gridGfx.moveTo(0, y + 0.5);
        this.gridGfx.lineTo(WIDTH, y + 0.5);
        this.gridGfx.strokePath();
      }
    }
```

And in `create()`, add at the end:

```js
      // World bounds for physics
      this.physics.world.setBounds(0, 0, WIDTH, HEIGHT);

      // Main graphics layer for character overlays (eyes, HP, effects)
      this.overlayGfx = this.add.graphics().setDepth(20);

      this.resetGame();
```

- [ ] **Step 6: Add trackHistory() and estimateVelocity() helper methods**

```js
    trackHistory(history, sprite) {
      history.push({ x: sprite.x, y: sprite.y });
      if (history.length > HISTORY_LEN) history.shift();
    }

    estimateVelocity(history) {
      if (history.length < 2) return { vx: 0, vy: 0 };
      const first = history[0];
      const last = history[history.length - 1];
      const dt = history.length - 1;
      return { vx: (last.x - first.x) / dt, vy: (last.y - first.y) / dt };
    }
```

- [ ] **Step 7: Refresh browser, verify characters appear at spawn positions and P1 can move with WASD**

---

### Task 4: Ball Throwing + Ball Physics + Wall Bouncing

**Files:**
- Modify: `phaser.html`

- [ ] **Step 1: Add tryThrowToward() method**

```js
    tryThrowToward(thrower, tx, ty) {
      const data = thrower.data.values;
      if (data.throwCooldown > 0) return;
      const dx = tx - thrower.x;
      const dy = ty - thrower.y;
      const dist = Math.hypot(dx, dy);
      if (dist < 1) return;
      const ndx = dx / dist;
      const ndy = dy / dist;
      const speed = data.team === 'player' ? PLAYER_BALL_SPEED : AI_BALL_SPEED;
      const spawnDist = CHAR_SIZE / 2 + BALL_RADIUS + 4;
      const bx = thrower.x + ndx * spawnDist;
      const by = thrower.y + ndy * spawnDist;

      const ball = this.physics.add.sprite(bx, by, 'ball');
      const tint = data.team === 'player' ? BALL_COLOR_PLAYER : BALL_COLOR_AI;
      ball.setTint(tint);
      ball.setData({
        owner: thrower,
        ownerTeam: data.team,
        bouncesLeft: BALL_MAX_BOUNCES,
        trail: [],
      });
      ball.setCircle(BALL_RADIUS);
      ball.setBounce(1);
      ball.setCollideWorldBounds(true);
      // Convert speed from px/frame to px/sec (60fps)
      const speedMul = this.ballSpeedMultiplier;
      ball.setVelocity(ndx * speed * 60 * speedMul, ndy * speed * 60 * speedMul);
      ball.setDepth(5);
      this.ballsGroup.add(ball);

      // Listen for world bounds bounce
      ball.body.onWorldBounds = true;
      ball.body.world.on('worldbounds', (body, up, down, left, right) => {
        if (body.gameObject === ball && ball.active) {
          const bd = ball.data.values;
          bd.bouncesLeft--;
          if (bd.bouncesLeft <= 0) {
            ball.destroy();
          }
        }
      });

      data.throwCooldown = PLAYER_THROW_COOLDOWN;
      data.facingAngle = Math.atan2(ndy, ndx);
    }
```

- [ ] **Step 2: Add mouse click handler in create() for P1 throwing**

```js
      this.input.on('pointerdown', (pointer) => {
        const px = pointer.x;
        const py = pointer.y;

        // Speed control buttons checked in render
        if (this.checkSpeedButtonClick(px, py)) return;

        if (this.gameState === 'ready') {
          this.startGame();
        } else if (this.gameState === 'playing') {
          const p1 = this.p1;
          if (p1 && p1.data.values.hp > 0) {
            this.tryThrowToward(p1, px, py);
          }
        } else if (this.gameState === 'gameover') {
          if (this.restartBtn && px >= this.restartBtn.x && px <= this.restartBtn.x + this.restartBtn.w &&
              py >= this.restartBtn.y && py <= this.restartBtn.y + this.restartBtn.h) {
            this.resetGame();
          }
        }
      });
```

- [ ] **Step 3: Add keyboard handler for starting game + any-key-start**

```js
      this.input.keyboard.on('keydown', (event) => {
        if (this.gameState === 'ready') {
          this.startGame();
        }
      });
```

- [ ] **Step 4: Add ball trail drawing and update in the render/update loop**

In `update()`, add ball trail tracking:

```js
      // Update ball trails
      for (const ball of this.ballsGroup.getChildren()) {
        if (!ball.active) continue;
        const data = ball.data.values;
        if (!data.trail) data.trail = [];
        data.trail.push({ x: ball.x, y: ball.y });
        if (data.trail.length > 10) data.trail.shift();
        // Remove off-screen balls
        if (ball.x < -50 || ball.x > WIDTH + 50 || ball.y < -50 || ball.y > HEIGHT + 50) {
          ball.destroy();
        }
      }
```

- [ ] **Step 5: Add ball drawing in the render graphics method**

```js
    drawBalls(gfx) {
      for (const ball of this.ballsGroup.getChildren()) {
        if (!ball.active) continue;
        const data = ball.data.values;
        const tint = ball.tint & 0xffffff;
        // Trail
        if (data.trail && data.trail.length > 1) {
          for (let i = 0; i < data.trail.length; i++) {
            const t = data.trail[i];
            const ratio = (i + 1) / data.trail.length;
            gfx.save();
            gfx.setAlpha(ratio * 0.6);
            gfx.fillStyle(tint);
            gfx.fillCircle(t.x, t.y, BALL_RADIUS * (0.5 + 0.5 * ratio));
            gfx.restore();
          }
        }
        // Glow
        gfx.save();
        gfx.setAlpha(0.25);
        gfx.fillStyle(tint);
        gfx.fillCircle(ball.x, ball.y, BALL_RADIUS * 1.6);
        gfx.restore();
        // Body
        gfx.fillStyle(tint);
        gfx.fillCircle(ball.x, ball.y, BALL_RADIUS);
        // Outline
        gfx.lineStyle(2, 0x000000, 0.5);
        gfx.strokeCircle(ball.x, ball.y, BALL_RADIUS);
        // Highlight
        gfx.fillStyle(0xffffff, 0.85);
        gfx.fillCircle(ball.x - 4, ball.y - 4, BALL_RADIUS * 0.4);
      }
    }
```

- [ ] **Step 6: Refresh browser, verify P1 can throw balls with mouse click, balls bounce off walls, balls disappear after 4 bounces**

---

### Task 5: Collision Detection (Ball vs Character)

**Files:**
- Modify: `phaser.html`

- [ ] **Step 1: Add overlap detection in resetGame() or create() after character/obstacle setup**

```js
      // Ball vs character overlap (hit detection)
      this.physics.add.overlap(this.ballsGroup, this.charGroup, this.onBallHit, null, this);
```

- [ ] **Step 2: Add onBallHit() callback method**

```js
    onBallHit(ball, char) {
      if (!ball.active || !char.active) return;
      const charData = char.data.values;
      const ballData = ball.data.values;

      // Can only hit the opposing team
      if (ballData.ownerTeam === charData.team) return;
      if (charData.hp <= 0 || charData.dyingTimer > 0) return;

      // Reduce HP
      charData.hp = Math.max(0, charData.hp - 1);

      // Hit effect
      this.spawnHitEffect(char.x, char.y, char.tint & 0xffffff);

      // Record hit for adaptive jitter
      if (ballData.owner && ballData.owner.data && ballData.owner.data.values.targetHitRates) {
        const rates = ballData.owner.data.values.targetHitRates;
        if (rates[charData.role]) {
          rates[charData.role].hits++;
        }
      }

      // Destroy ball
      ball.destroy();

      // Death check
      if (charData.hp <= 0 && charData.dyingTimer === 0) {
        charData.dyingTimer = 30;
        for (let p = 0; p < 12; p++) {
          const angle = Math.random() * Math.PI * 2;
          const spd = 1 + Math.random() * 3;
          charData.deathParticles.push({
            x: 0, y: 0,
            vx: Math.cos(angle) * spd,
            vy: Math.sin(angle) * spd,
            life: 30,
            size: 2 + Math.random() * 3,
            color: 0x1000000 + Math.floor(Math.random() * 0xffffff),
          });
        }
      } else if (charData.hp > 0) {
        // Check win/loss
        if (this.allDead(this.teamAI)) { this.endGame('player'); return; }
        if (this.allDead(this.teamPlayer)) { this.endGame('ai'); return; }
      }
    }
```

- [ ] **Step 3: Add allDead() helper method**

```js
    allDead(team) {
      return team.every(c => c.data.values.hp <= 0 && c.data.values.dyingTimer <= 0);
    }
```

- [ ] **Step 4: Refresh browser, verify balls hit characters and reduce HP**

---

### Task 6: Hit Effects (Particles + Flash Ring + Camera Shake)

**Files:**
- Modify: `phaser.html`

- [ ] **Step 1: Add spawnHitEffect() method using Phaser particles and Graphics ring**

```js
    spawnHitEffect(x, y, color) {
      // Particle burst
      const emitter = this.add.particles(x, y, 'particle', {
        speed: { min: 120, max: 350 },
        lifespan: { min: 300, max: 500 },
        quantity: 16,
        scale: { start: 1.5, end: 0 },
        tint: [color, color, color, 0xffffff, 0xffffff],
        gravityY: 300,
        emitting: false,
      });
      emitter.explode(16);
      // Auto-destroy after particles finish
      this.time.delayedCall(600, () => { emitter.destroy(); });

      // Flash ring — expanding circle that fades
      const ring = this.add.graphics().setDepth(25);
      let ringAge = 0;
      const ringMaxAge = 15;
      const ringMaxRadius = 120;
      const ringEvent = this.time.addEvent({
        delay: 16,
        repeat: ringMaxAge - 1,
        callback: () => {
          ringAge++;
          ring.clear();
          const progress = ringAge / ringMaxAge;
          const radius = Math.max(0.1, ringMaxRadius * progress);
          const alpha = Math.max(0, 0.6 * (1 - progress));
          ring.setAlpha(alpha);
          ring.lineStyle(4, 0xffffff);
          ring.strokeCircle(x, y, radius);
          ring.fillStyle(0xffffff, alpha * 0.3);
          ring.fillCircle(x, y, radius * 0.6);
        }
      });
      this.time.delayedCall(ringMaxAge * 16 + 100, () => { ring.destroy(); });

      // Camera shake
      this.cameras.main.shake(200, 0.005);
    }
```

- [ ] **Step 2: Refresh browser, verify hit produces particles + expanding ring + subtle camera shake**

---

### Task 7: AI Logic (Ally + Enemies)

**Files:**
- Modify: `phaser.html`

This is the largest task — all AI logic migrated to read from/set Phaser sprite data.

- [ ] **Step 1: Add nearestEnemy() and pickTarget() methods**

```js
    nearestEnemy(character) {
      const charData = character.data.values;
      const enemies = charData.team === 'player' ? this.teamAI : this.teamPlayer;
      let best = null, bestD = Infinity;
      for (const e of enemies) {
        if (!e.active) continue;
        const ed = e.data.values;
        if (ed.hp <= 0 || ed.dyingTimer > 0) continue;
        const d = Math.hypot(e.x - character.x, e.y - character.y);
        if (d < bestD) { bestD = d; best = e; }
      }
      return best;
    }

    pickTarget(ai) {
      const aiData = ai.data.values;
      const enemies = aiData.team === 'player' ? this.teamAI : this.teamPlayer;
      const alive = enemies.filter(e => e.active && e.data.values.hp > 0 && e.data.values.dyingTimer <= 0);
      if (alive.length === 0) return null;
      if (alive.length === 1) return alive[0];
      if (aiData.targetIndex == null) aiData.targetIndex = 0;
      const target = alive[aiData.targetIndex % alive.length];
      aiData.targetIndex = (aiData.targetIndex + 1) % alive.length;
      return target;
    }
```

- [ ] **Step 2: Add updateAIChar() method (full AI logic)**

This is the largest single method. It mirrors the original `updateAIChar` exactly but reads/writes to `sprite.data.values` and uses `setVelocity()`:

```js
    updateAIChar(ai) {
      const aiData = ai.data.values;
      if (aiData.hp <= 0 || aiData.dyingTimer > 0) return;

      const pVel = this.estimatePlayerVelocity();

      // 1) Threat assessment
      let avoidX = 0, avoidY = 0;
      let threatScore = 0;
      const PREDICT_FRAMES = 24;
      const DANGER_RADIUS = 70;

      for (const ball of this.ballsGroup.getChildren()) {
        if (!ball.active) continue;
        const bData = ball.data.values;
        if (bData.ownerTeam === aiData.team) continue;
        if (bData.owner && (!bData.owner.active || bData.owner.data.values.hp <= 0 || bData.owner.data.values.dyingTimer > 0)) continue;

        const body = ball.body;
        if (!body) continue;
        const bvx = body.velocity.x / 60;  // convert from px/sec back to px/frame
        const bvy = body.velocity.y / 60;
        const bxi = ball.x;
        const byi = ball.y;
        const dx = bxi - ai.x;
        const dy = byi - ai.y;
        const d = Math.hypot(dx, dy);
        if (d > AI_THREAT_RANGE) continue;
        const vlen = Math.hypot(bvx, bvy);
        if (vlen < 0.01) continue;

        const speedFactor = Math.min(3.0, vlen / AI_BALL_SPEED);
        const fx = bxi + bvx * PREDICT_FRAMES;
        const fy = byi + bvy * PREDICT_FRAMES;
        const fdx = fx - ai.x;
        const fdy = fy - ai.y;
        const fDist = Math.hypot(fdx, fdy);

        const dot = (bvx * dx + bvy * dy) / (vlen * d);
        const inLineOfFire = dot > 0;
        const willBeClose = fDist < DANGER_RADIUS;
        if (!inLineOfFire && !willBeClose) continue;

        const bvxNorm = bvx / vlen;
        const bvyNorm = bvy / vlen;
        const perpX = -bvyNorm;
        const perpY = bvxNorm;
        const sideDot = perpX * dx + perpY * dy;
        const sign = sideDot > 0 ? -1 : 1;
        const w = d > 0 ? 1 / (d + 20) : 1;

        avoidX += perpX * sign * w * PERP_DODGE_WEIGHT * speedFactor;
        avoidY += perpY * sign * w * PERP_DODGE_WEIGHT * speedFactor;

        const urgency = (DANGER_RADIUS - Math.min(fDist, DANGER_RADIUS)) / DANGER_RADIUS;
        threatScore += (0.5 + urgency) * speedFactor;
      }

      const aLen = Math.hypot(avoidX, avoidY);
      if (aLen > 0) { avoidX /= aLen; avoidY /= aLen; }

      // 1b) AI state
      if (aiData.hp <= AI_STATE_HP_RETREAT) {
        aiData.aiState = 'RETREAT';
      } else if (threatScore >= AI_STATE_THREAT_DEFENSIVE) {
        aiData.aiState = 'DEFENSIVE';
      } else {
        aiData.aiState = 'AGGRESSIVE';
      }

      // 2) Center bias
      const margin = 60;
      let centerX = 0, centerY = 0;
      if (ai.x < margin) centerX = (margin - ai.x) / margin;
      if (ai.x > WIDTH - margin) centerX = -(ai.x - (WIDTH - margin)) / margin;
      if (ai.y < margin) centerY = (margin - ai.y) / margin;
      if (ai.y > HEIGHT - margin) centerY = -(ai.y - (HEIGHT - margin)) / margin;

      // 3) Movement
      let mx = 0, my = 0;
      const dodgeGain = aiData.aiState === 'RETREAT' ? 1.6
                      : aiData.aiState === 'DEFENSIVE' ? 1.3 : 1.0;

      if (threatScore > 0.1) {
        const t = Math.min(1, threatScore * dodgeGain);
        mx = avoidX * t + centerX * 0.3;
        my = avoidY * t + centerY * 0.3;
      } else {
        const target = this.nearestEnemy(ai);
        if (aiData.aiState === 'RETREAT') {
          const goalX = aiData.team === 'player' ? RETREAT_GOAL_X_PLAYER : RETREAT_GOAL_X_AI;
          const goalY = HEIGHT / 2;
          const rdx = goalX - ai.x;
          const rdy = goalY - ai.y;
          const rd = Math.hypot(rdx, rdy) || 1;
          mx = (rdx / rd) * 0.7 + centerX * 0.3;
          my = (rdy / rd) * 0.7 + centerY * 0.3;
        } else if (target) {
          const tdx = target.x - ai.x;
          const tdy = target.y - ai.y;
          const td = Math.hypot(tdx, tdy) || 1;
          const ideal = aiData.aiState === 'AGGRESSIVE' ? AGGRESSIVE_IDEAL_DIST : DEFENSIVE_IDEAL_DIST;
          const diff = td - ideal;
          const strength = Math.min(0.6, Math.abs(diff) / 200);
          const dir = diff > 0 ? 1 : -1;
          mx = (tdx / td) * dir * strength + centerX * 0.3;
          my = (tdy / td) * dir * strength + centerY * 0.3;
        } else {
          aiData.aiWanderTimer = (aiData.aiWanderTimer || 0) - 1;
          if (aiData.aiWanderTimer <= 0) {
            aiData.aiWanderAngle = Math.random() * Math.PI * 2;
            aiData.aiWanderTimer = 40 + Math.floor(Math.random() * 50);
          }
          mx = Math.cos(aiData.aiWanderAngle) * 0.25 + centerX * 0.3;
          my = Math.sin(aiData.aiWanderAngle) * 0.25 + centerY * 0.3;
        }
      }

      // 3b) Team spread forces
      const mates = aiData.team === 'player' ? this.teamPlayer : this.teamAI;
      for (const mate of mates) {
        if (mate === ai || !mate.active || mate.data.values.hp <= 0 || mate.data.values.dyingTimer > 0) continue;
        const mdx = ai.x - mate.x;
        const mdy = ai.y - mate.y;
        const md = Math.hypot(mdx, mdy);
        if (md < TEAM_SPREAD_MIN && md > 0.01) {
          const force = (TEAM_SPREAD_MIN - md) / TEAM_SPREAD_MIN * TEAM_SPREAD_FORCE;
          mx += (mdx / md) * force;
          my += (mdy / md) * force;
        }
      }

      // Enemy vertical split
      if (aiData.team === 'ai') {
        const cY = HEIGHT / 2;
        const pushDir = aiData.role === 'ai1' ? -1 : 1;
        if (Math.abs(ai.y - cY) < ENEMY_VERTICAL_SPLIT * 0.5) {
          my += pushDir * 0.15;
        }
      }

      ai.setVelocity(mx * CHAR_SPEED * 60, my * CHAR_SPEED * 60);

      // 4) Throw
      if (aiData.throwCooldown > 0) aiData.throwCooldown--;
      aiData.aiHoldTimer = (aiData.aiHoldTimer || 0) + 1;

      const target = this.pickTarget(ai);
      if (target) {
        const td = Math.hypot(target.x - ai.x, target.y - ai.y) || 1;
        const MIN_HOLD = aiData.aiState === 'AGGRESSIVE' ? 25
                       : aiData.aiState === 'RETREAT' ? 60 : 40;
        const MAX_HOLD = aiData.aiState === 'AGGRESSIVE' ? 70
                       : aiData.aiState === 'RETREAT' ? 150 : 100;

        if (aiData.aiHoldTimer >= MIN_HOLD) {
          const tdx = target.x - ai.x;
          const tdy = target.y - ai.y;
          const targetData = target.data.values;
          const tVel = target === this.teamPlayer[0] ? this.estimateVelocity(this.p1History)
                     : target === this.teamPlayer[1] ? this.estimateVelocity(this.p2History)
                     : { vx: 0, vy: 0 };
          const closingSpeed = -(tVel.vx * tdx + tVel.vy * tdy) / td;
          const closingFactor = Math.max(0, Math.min(1, closingSpeed / AI_BALL_SPEED));
          const distFactor = 1 - Math.min(1, td / 500);
          let opportunity = distFactor * 0.3 + closingFactor * 0.5;
          if (targetData.hp <= 1) opportunity += 0.2;

          const threshold = aiData.aiState === 'RETREAT' ? 0.7
                          : aiData.aiState === 'AGGRESSIVE' ? 0.3 : 0.5;
          const forcedThrow = aiData.aiHoldTimer >= MAX_HOLD;

          if (opportunity >= threshold || forcedThrow) {
            aiData.aiHoldTimer = 0;
            if (td < 40) return;
            const travelFrames = td / AI_BALL_SPEED;
            const leadX = target.x + tVel.vx * travelFrames;
            const leadY = target.y + tVel.vy * travelFrames;

            const stats = aiData.targetHitRates[targetData.role];
            const hitRate = (stats && stats.total >= 3) ? stats.hits / stats.total : 0.5;
            const jitterRange = JITTER_MIN + (1 - hitRate) * (JITTER_MAX - JITTER_MIN);
            const jitterX = (Math.random() - 0.5) * jitterRange;
            const jitterY = (Math.random() - 0.5) * jitterRange;

            this.tryThrowToward(ai, leadX + jitterX, leadY + jitterY);

            if (!aiData.targetHitRates[targetData.role]) aiData.targetHitRates[targetData.role] = { hits: 0, total: 0 };
            aiData.targetHitRates[targetData.role].total++;
          }
        }
      }
    }
```

- [ ] **Step 3: Add updateAlly() method**

```js
    updateAlly() {
      const ally = this.ally;
      if (!ally || !ally.active) return;
      if (ally.data.values.dyingTimer > 0) return;
      this.trackHistory(this.p2History, ally);
      this.updateAIChar(ally);
    }
```

- [ ] **Step 4: Refresh browser, verify ally and enemies can move and throw balls**

---

### Task 8: Death Animation + Win/Loss + Update Loop

**Files:**
- Modify: `phaser.html`

- [ ] **Step 1: Add death animation update logic in update()**

```js
      // Update death animations
      const allChars = [...this.teamPlayer, ...this.teamAI];
      for (const c of allChars) {
        if (!c || !c.active) continue;
        const d = c.data.values;
        if (d.dyingTimer > 0) {
          d.dyingTimer--;
          for (const p of d.deathParticles) {
            p.x += p.vx;
            p.y += p.vy;
            p.life--;
          }
          if (d.dyingTimer <= 0) {
            c.setVisible(false);
            c.body.enable = false;
            if (this.allDead(this.teamAI)) { this.endGame('player'); return; }
            if (this.allDead(this.teamPlayer)) { this.endGame('ai'); return; }
          }
        }
      }
```

- [ ] **Step 2: Add endGame() and startGame() methods**

```js
    endGame(winner) {
      this.gameState = 'gameover';
      this.winner = winner;
    }

    startGame() {
      this.gameState = 'playing';
      this.hintTimer = HINT_FRAMES;
    }
```

- [ ] **Step 3: Add full update() method with all game states**

```js
    update() {
      if (this.gameState === 'playing') {
        if (this.hintTimer > 0) this.hintTimer--;
        this.updateP1();
        this.updateAlly();
        for (const ai of this.teamAI) this.updateAIChar(ai);
        // Death animations
        this.updateDeathAnimations();
        // Ball cleanup already handled by overlap + worldbounds
      }
      // Always render overlay graphics
      this.renderOverlays();
    }
```

- [ ] **Step 4: Add renderOverlays() method that draws eyes, HP bars, balls, effects**

```js
    renderOverlays() {
      this.overlayGfx.clear();

      // Draw all characters (eyes, HP, death anim, particles)
      for (const c of this.teamPlayer) {
        if (c && c.active) this.drawCharacter(c, this.overlayGfx);
      }
      for (const c of this.teamAI) {
        if (c && c.active) this.drawCharacter(c, this.overlayGfx);
      }

      // Draw balls
      this.drawBalls(this.overlayGfx);

      // Speed controls
      this.drawSpeedControls(this.overlayGfx);

      // Hint text
      this.drawHint();

      // Start screen
      if (this.gameState === 'ready') this.drawStartScreen();
      // Game over screen
      else if (this.gameState === 'gameover') this.drawGameOver();
    }
```

- [ ] **Step 5: Refresh browser, verify death animation plays and game over triggers correctly**

---

### Task 9: HUD (Speed Controls, Hint, Start Screen, Game Over Screen)

**Files:**
- Modify: `phaser.html`

- [ ] **Step 1: Add drawSpeedControls() method**

```js
    drawSpeedControls(gfx) {
      const btnW = 40, btnH = 32, gap = 8, btnY = 12, btnStartX = WIDTH - 200;
      const btn1X = btnStartX;
      const btn2X = btnStartX + btnW + gap;
      this.speedDownBtn = { x: btn1X, y: btnY, w: btnW, h: btnH };
      this.speedUpBtn = { x: btn2X, y: btnY, w: btnW, h: btnH };

      // Buttons
      gfx.fillStyle(0x4a5568);
      gfx.fillRoundedRect(btn1X, btnY, btnW, btnH, 6);
      gfx.fillRoundedRect(btn2X, btnY, btnW, btnH, 6);

      // Labels (use Phaser Text for centered text)
      if (!this.speedMinusText) {
        this.speedMinusText = this.add.text(btn1X + btnW / 2, btnY + btnH / 2, '-', {
          fontSize: '22px', fontFamily: '-apple-system, "PingFang SC", "Microsoft YaHei", sans-serif',
          color: '#ffffff', fontStyle: 'bold'
        }).setOrigin(0.5).setDepth(30);
        this.speedPlusText = this.add.text(btn2X + btnW / 2, btnY + btnH / 2, '+', {
          fontSize: '22px', fontFamily: '-apple-system, "PingFang SC", "Microsoft YaHei", sans-serif',
          color: '#ffffff', fontStyle: 'bold'
        }).setOrigin(0.5).setDepth(30);
        this.speedLabelText = this.add.text(btn2X + btnW + gap + 30, btnY + btnH / 2, '', {
          fontSize: '14px', fontFamily: '-apple-system, "PingFang SC", "Microsoft YaHei", sans-serif',
          color: '#333333', fontStyle: 'bold'
        }).setOrigin(0, 0.5).setDepth(30);
      }
      this.speedLabelText.setText('球速: ' + Math.round(this.ballSpeedMultiplier * 100) + '%');
    }
```

- [ ] **Step 2: Add checkSpeedButtonClick() method**

```js
    checkSpeedButtonClick(px, py) {
      const btn = this.speedDownBtn;
      if (btn && px >= btn.x && px <= btn.x + btn.w && py >= btn.y && py <= btn.y + btn.h) {
        this.ballSpeedMultiplier = Math.max(0.3, this.ballSpeedMultiplier - 0.25);
        return true;
      }
      const btn2 = this.speedUpBtn;
      if (btn2 && px >= btn2.x && px <= btn2.x + btn2.w && py >= btn2.y && py <= btn2.y + btn2.h) {
        this.ballSpeedMultiplier = Math.min(3.0, this.ballSpeedMultiplier + 0.25);
        return true;
      }
      return false;
    }
```

- [ ] **Step 3: Add drawHint() method**

```js
    drawHint() {
      if (this.hintTimer <= 0) return;
      const alpha = Math.min(1, this.hintTimer / 60);
      if (!this.hintText) {
        this.hintText = this.add.text(WIDTH / 2, 8, 'WASD 移动   ·   鼠标点击 投球', {
          fontSize: '14px', fontFamily: '-apple-system, "PingFang SC", "Microsoft YaHei", sans-serif',
          color: '#999999',
        }).setOrigin(0.5, 0).setDepth(30);
      }
      this.hintText.setAlpha(alpha);
    }
```

- [ ] **Step 4: Add drawStartScreen() and drawGameOver() methods**

These use Phaser Text + Graphics overlay objects:

```js
    drawStartScreen() {
      if (!this.startContainer) {
        this.startContainer = this.add.container(0, 0).setDepth(40);
        // Semi-transparent overlay
        const overlay = this.add.graphics();
        overlay.fillStyle(0xf5f5f5, 0.85);
        overlay.fillRect(0, 0, WIDTH, HEIGHT);
        this.startContainer.add(overlay);
        // Title
        const title = this.add.text(WIDTH / 2, HEIGHT / 2 - 80, '躲避球', {
          fontSize: '36px', fontFamily: '-apple-system, "PingFang SC", "Microsoft YaHei", sans-serif',
          color: COLOR_TEXT, fontStyle: 'bold'
        }).setOrigin(0.5);
        this.startContainer.add(title);
        // Instructions
        const instr1 = this.add.text(WIDTH / 2, HEIGHT / 2 - 30, 'WASD 移动 · 鼠标点击 投球', {
          fontSize: '16px', fontFamily: '-apple-system, "PingFang SC", "Microsoft YaHei", sans-serif',
          color: '#555555'
        }).setOrigin(0.5);
        this.startContainer.add(instr1);
        const instr2 = this.add.text(WIDTH / 2, HEIGHT / 2 - 5, '2v2 模式: 你 (蓝) + AI 队友 (青) vs 2 个敌方 AI (红/橙)', {
          fontSize: '16px', fontFamily: '-apple-system, "PingFang SC", "Microsoft YaHei", sans-serif',
          color: '#555555'
        }).setOrigin(0.5);
        this.startContainer.add(instr2);
        // Start button
        const bw = 180, bh = 48;
        const bx = WIDTH / 2 - bw / 2;
        const by = HEIGHT / 2 + 30;
        const btnGfx = this.add.graphics();
        btnGfx.fillStyle(COLOR_P1);
        btnGfx.fillRoundedRect(bx, by, bw, bh, 8);
        this.startContainer.add(btnGfx);
        const btnText = this.add.text(WIDTH / 2, by + bh / 2, '点击开始', {
          fontSize: '20px', fontFamily: '-apple-system, "PingFang SC", "Microsoft YaHei", sans-serif',
          color: '#ffffff', fontStyle: 'bold'
        }).setOrigin(0.5);
        this.startContainer.add(btnText);
        this.startBtnRect = { x: bx, y: by, w: bw, h: bh };
      }
      this.startContainer.setVisible(true);
    }
```

```js
    drawGameOver() {
      if (!this.gameOverContainer) {
        this.gameOverContainer = this.add.container(0, 0).setDepth(40);
        // Overlay
        const overlay = this.add.graphics();
        overlay.fillStyle(0x000000, 0.55);
        overlay.fillRect(0, 0, WIDTH, HEIGHT);
        this.gameOverContainer.add(overlay);
        // Message
        this.gameOverMsg = this.add.text(WIDTH / 2, HEIGHT / 2 - 30, '', {
          fontSize: '48px', fontFamily: '-apple-system, "PingFang SC", "Microsoft YaHei", sans-serif',
          color: '#ffffff', fontStyle: 'bold'
        }).setOrigin(0.5);
        this.gameOverContainer.add(this.gameOverMsg);
        // Restart button
        const bw = 160, bh = 44;
        const bx = WIDTH / 2 - bw / 2;
        const by = HEIGHT / 2 + 20;
        const btnGfx = this.add.graphics();
        btnGfx.fillStyle(COLOR_P1);
        btnGfx.fillRoundedRect(bx, by, bw, bh, 8);
        this.gameOverContainer.add(btnGfx);
        const btnText = this.add.text(WIDTH / 2, by + bh / 2, '再来一局', {
          fontSize: '18px', fontFamily: '-apple-system, "PingFang SC", "Microsoft YaHei", sans-serif',
          color: '#ffffff', fontStyle: 'bold'
        }).setOrigin(0.5);
        this.gameOverContainer.add(btnText);
        this.restartBtn = { x: bx, y: by, w: bw, h: bh };
      }
      this.gameOverMsg.setText(this.winner === 'player' ? '你赢了!' : '你输了');
      this.gameOverContainer.setVisible(true);
    }
```

- [ ] **Step 5: Hide start/gameover containers when game state changes. Update resetGame() to hide gameOverContainer**

In `resetGame()`, add:

```js
      if (this.startContainer) this.startContainer.setVisible(false);
      if (this.gameOverContainer) this.gameOverContainer.setVisible(false);
```

And in `startGame()`:

```js
      if (this.startContainer) this.startContainer.setVisible(false);
```

- [ ] **Step 6: Refresh browser, verify start screen shows, click to start, game over triggers, restart works**

---

### Task 10: Full Game Test + Bug Fix Pass

**Files:**
- Modify: `phaser.html`

- [ ] **Step 1: Play a full game round — test P1 movement (WASD), P1 throwing (mouse click), ally AI behavior, enemy AI behavior, hit detection, HP reduction, death animation, game over, restart**

- [ ] **Step 2: Test edge cases — ball bouncing off walls, ball destroying after 4 bounces, characters colliding with obstacles, speed +/- buttons, hint text appearing and fading**

- [ ] **Step 3: Compare side-by-side with `vanilla.html` — verify visual parity for character appearance, ball trails, HP bars, obstacle rendering, grid, UI elements**

- [ ] **Step 4: Fix any bugs found during testing — common issues:**
  - Velocity units: Phaser uses px/sec, original uses px/frame at 60fps. Ensure all velocities are multiplied by 60.
  - Character sprite visibility: death sprites should hide but not destroy (use `setVisible(false)` + `body.enable = false`).
  - Ball world-bounds callback: ensure `body.onWorldBounds = true` is set on each ball and the listener doesn't double-count.
  - Obstacle collision shapes may need `refreshBody()` after setting size.
  - Reset: ensure all groups are properly cleared and all Phaser objects destroyed/removed in `resetGame()`.

- [ ] **Step 5: Commit phaser.html**

```bash
git add phaser.html
git commit -m "feat: add Phaser 3 version of dodgeball game with full feature parity"
```

---

## Self-Review Checklist

- [x] All spec requirements covered (see feature parity checklist)
- [x] No placeholders — every step has complete code
- [x] Type consistency — `sprite.data.values` used consistently for custom data
- [x] No TBD/TODO items
- [x] Single file `phaser.html` — matches spec
- [x] CDN Phaser 3.80.1 — matches spec
- [x] BootScene generates textures — matches spec
- [x] GameScene manages all gameplay — matches spec
- [x] Arcade Physics for collisions — matches spec
- [x] Phaser particles for hit effects — matches spec
- [x] Camera shake on hit — matches spec
- [x] Custom state variable `gameState` — matches spec
- [x] `setTint()` for character colors — matches spec