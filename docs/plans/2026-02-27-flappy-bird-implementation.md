# Flappy Bird - Level Edition: Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a complete Flappy Bird game with 9 visual levels, deployed as a single HTML file to play.fun.

**Architecture:** Single `index.html` with inline JS/CSS. HTML5 Canvas renders all game screens (loading, home, gameplay, game over). Game state machine drives transitions. Web Audio API generates all sounds programmatically. Assets loaded from `/assets/` folder.

**Tech Stack:** Vanilla HTML5/JS/CSS, Canvas 2D API, Web Audio API, localStorage

**Design doc:** `docs/plans/2026-02-27-flappy-bird-design.md`

---

## Asset Reference

- **Bird frames:** 64x16 PNG, RGBA. Contains 4 animation frames at 16x16 each (left to right). Individual files: `Bird{style}-{color}.png`
- **Backgrounds:** 256x256 PNG. Tile/scale to fill canvas.
- **Pipe spritesheets:** PipeStyle1=128x160, PipeStyle2-5=128x96. Grid of colored pipe body segments. Each cell ~32px wide.
- **Ground tiles:** SimpleStyle 128x80-112, TileStyle 400x112-208. Ground/platform tile strips.

---

### Task 1: Project Setup + Asset Copy

**Files:**
- Create: `assets/` folder structure
- Create: `index.html` (skeleton)

**Step 1: Initialize git repo**

```bash
cd "C:\Users\alokp\OneDrive\Desktop\flappy bird game"
git init
```

**Step 2: Copy assets into project**

Copy from `C:\Users\alokp\Downloads\Flappy Bird Assets 1.6 (Zip)\Flappy Bird Assets\` into `assets/`:
```
assets/
  backgrounds/   -> Background1.png through Background9.png
  birds/
    style1/      -> Bird1-1.png through Bird1-7.png
    style2/      -> Bird2-1.png through Bird2-7.png
  tiles/
    style1/      -> PipeStyle1.png, SimpleStyle1.png, TileStyle1.png
    style2/      -> PipeStyle2.png, SimpleStyle2.png, TileStyle2.png
    style3/      -> PipeStyle3.png, SimpleStyle3.png, TileStyle3.png
    style4/      -> PipeStyle4.png, SimpleStyle4.png, TileStyle4.png
    style5/      -> PipeStyle5.png, SimpleStyle5.png, TileStyle5.png
```

**Step 3: Create skeleton `index.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
  <title>Flappy Bird</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      background: #000;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      overflow: hidden;
      touch-action: none;
    }
    canvas {
      image-rendering: pixelated;
      image-rendering: crisp-edges;
    }
  </style>
</head>
<body>
  <canvas id="game"></canvas>
  <script>
    // Game code goes here
    const canvas = document.getElementById('game');
    const ctx = canvas.getContext('2d');
    ctx.imageSmoothingEnabled = false;

    // === CONSTANTS ===
    const GAME_WIDTH = 288;
    const GAME_HEIGHT = 512;

    // === RESIZE HANDLER ===
    function resize() {
      const ratio = GAME_WIDTH / GAME_HEIGHT;
      let w = window.innerWidth;
      let h = window.innerHeight;
      if (w / h > ratio) {
        canvas.style.height = h + 'px';
        canvas.style.width = (h * ratio) + 'px';
      } else {
        canvas.style.width = w + 'px';
        canvas.style.height = (w / ratio) + 'px';
      }
      canvas.width = GAME_WIDTH;
      canvas.height = GAME_HEIGHT;
      ctx.imageSmoothingEnabled = false;
    }
    window.addEventListener('resize', resize);
    resize();

    // Placeholder: black screen
    ctx.fillStyle = '#000';
    ctx.fillRect(0, 0, GAME_WIDTH, GAME_HEIGHT);
  </script>
</body>
</html>
```

**Step 4: Verify skeleton renders**

Open `index.html` in browser. Should see a black canvas centered on screen, responsive to window resizing.

**Step 5: Commit**

```bash
git add -A
git commit -m "feat: project setup with assets and HTML skeleton"
```

---

### Task 2: Asset Loader + Loading Screen

**Files:**
- Modify: `index.html` (add asset loading system and loading screen rendering)

**Step 1: Add asset loading system**

Add after the resize handler in `<script>`:

```javascript
// === ASSET LOADER ===
const assets = {};
const assetList = [];
let assetsLoaded = 0;

function addAsset(key, src) {
  assetList.push({ key, src });
}

function loadAssets() {
  return new Promise((resolve) => {
    if (assetList.length === 0) { resolve(); return; }
    assetList.forEach(({ key, src }) => {
      const img = new Image();
      img.onload = () => {
        assets[key] = img;
        assetsLoaded++;
        if (assetsLoaded === assetList.length) resolve();
      };
      img.onerror = () => {
        console.warn('Failed to load:', src);
        assetsLoaded++;
        if (assetsLoaded === assetList.length) resolve();
      };
      img.src = src;
    });
  });
}

// Register all assets
// Backgrounds
for (let i = 1; i <= 9; i++) addAsset(`bg${i}`, `assets/backgrounds/Background${i}.png`);
// Birds Style 1
for (let i = 1; i <= 7; i++) addAsset(`bird1_${i}`, `assets/birds/style1/Bird1-${i}.png`);
// Birds Style 2
for (let i = 1; i <= 7; i++) addAsset(`bird2_${i}`, `assets/birds/style2/Bird2-${i}.png`);
// Tiles
for (let i = 1; i <= 5; i++) {
  addAsset(`pipe${i}`, `assets/tiles/style${i}/PipeStyle${i}.png`);
  addAsset(`simple${i}`, `assets/tiles/style${i}/SimpleStyle${i}.png`);
  addAsset(`tile${i}`, `assets/tiles/style${i}/TileStyle${i}.png`);
}

// Loading progress: 0.0 to 1.0
function getLoadProgress() {
  return assetList.length === 0 ? 1 : assetsLoaded / assetList.length;
}
```

**Step 2: Add loading screen rendering**

```javascript
// === GAME STATE ===
let gameState = 'loading'; // loading | home | playing | gameover
let loadingBirdFrame = 0;
let loadingBirdTimer = 0;
let loadingBobY = 0;
let loadingBobDir = 1;

function drawLoadingScreen(dt) {
  ctx.fillStyle = '#000';
  ctx.fillRect(0, 0, GAME_WIDTH, GAME_HEIGHT);

  // Title text
  ctx.fillStyle = '#FFD700';
  ctx.font = 'bold 28px monospace';
  ctx.textAlign = 'center';
  ctx.fillText('FLAPPY BIRD', GAME_WIDTH / 2, 140);

  // Animated bird (use a simple colored rectangle until assets load, then real sprite)
  loadingBirdTimer += dt;
  if (loadingBirdTimer > 150) {
    loadingBirdTimer = 0;
    loadingBirdFrame = (loadingBirdFrame + 1) % 4;
  }
  loadingBobY += loadingBobDir * dt * 0.03;
  if (Math.abs(loadingBobY) > 10) loadingBobDir *= -1;

  const birdY = 210 + loadingBobY;
  // Draw placeholder bird (yellow rect) - will be replaced with sprite in Task 5
  ctx.fillStyle = '#FFD700';
  ctx.fillRect(GAME_WIDTH / 2 - 16, birdY, 32, 32);

  // Progress bar
  const progress = getLoadProgress();
  const barW = 180;
  const barH = 16;
  const barX = (GAME_WIDTH - barW) / 2;
  const barY = 320;

  // Bar background
  ctx.fillStyle = '#333';
  ctx.fillRect(barX, barY, barW, barH);
  // Bar fill
  ctx.fillStyle = '#4CAF50';
  ctx.fillRect(barX, barY, barW * progress, barH);
  // Bar border
  ctx.strokeStyle = '#FFF';
  ctx.lineWidth = 2;
  ctx.strokeRect(barX, barY, barW, barH);

  // Percentage text
  ctx.fillStyle = '#FFF';
  ctx.font = '14px monospace';
  ctx.fillText(Math.floor(progress * 100) + '%', GAME_WIDTH / 2, barY + barH + 24);
}
```

**Step 3: Add game loop**

```javascript
// === GAME LOOP ===
let lastTime = 0;

function gameLoop(timestamp) {
  const dt = lastTime ? timestamp - lastTime : 16;
  lastTime = timestamp;

  switch (gameState) {
    case 'loading':
      drawLoadingScreen(dt);
      break;
    case 'home':
      // TODO Task 3
      break;
    case 'playing':
      // TODO Task 5+
      break;
    case 'gameover':
      // TODO Task 10
      break;
  }

  requestAnimationFrame(gameLoop);
}

// === START ===
loadAssets().then(() => {
  setTimeout(() => { gameState = 'home'; }, 500); // brief pause after 100%
});
requestAnimationFrame(gameLoop);
```

**Step 4: Verify loading screen**

Open in browser. Should see:
- Black background
- "FLAPPY BIRD" gold text
- Yellow rectangle bobbing up/down (placeholder bird)
- Green progress bar filling to 100%
- Transitions (gameState becomes 'home') after loading

**Step 5: Commit**

```bash
git add index.html
git commit -m "feat: asset loader and loading screen with progress bar"
```

---

### Task 3: Home Screen

**Files:**
- Modify: `index.html`

**Step 1: Add home screen state variables**

```javascript
let homeGroundX = 0;
let homeBirdFrame = 0;
let homeBirdTimer = 0;
let homeBirdBobY = 0;
let homeBirdBobDir = 1;
```

**Step 2: Add home screen draw function**

```javascript
function drawHomeScreen(dt) {
  // Draw background (BG1 scaled to fill)
  const bg = assets.bg1;
  if (bg) {
    ctx.drawImage(bg, 0, 0, GAME_WIDTH, GAME_HEIGHT);
  } else {
    ctx.fillStyle = '#4EC0CA';
    ctx.fillRect(0, 0, GAME_WIDTH, GAME_HEIGHT);
  }

  // Scrolling ground
  homeGroundX -= 1.5;
  const ground = assets.simple1;
  if (ground) {
    const gH = 56; // ground height on screen
    const gY = GAME_HEIGHT - gH;
    // tile ground across bottom
    const gW = ground.width * (gH / ground.height);
    if (homeGroundX <= -gW) homeGroundX += gW;
    for (let x = homeGroundX; x < GAME_WIDTH; x += gW) {
      ctx.drawImage(ground, x, gY, gW, gH);
    }
  }

  // Title
  ctx.fillStyle = '#FFF';
  ctx.strokeStyle = '#000';
  ctx.lineWidth = 4;
  ctx.font = 'bold 36px monospace';
  ctx.textAlign = 'center';
  ctx.strokeText('FLAPPY BIRD', GAME_WIDTH / 2, 120);
  ctx.fillText('FLAPPY BIRD', GAME_WIDTH / 2, 120);

  // Animated bird
  homeBirdTimer += dt;
  if (homeBirdTimer > 150) {
    homeBirdTimer = 0;
    homeBirdFrame = (homeBirdFrame + 1) % 4;
  }
  homeBirdBobY += homeBirdBobDir * dt * 0.04;
  if (Math.abs(homeBirdBobY) > 12) homeBirdBobDir *= -1;

  const birdImg = assets.bird1_1;
  if (birdImg) {
    const frameW = 16;
    const frameH = 16;
    const scale = 3;
    const dx = GAME_WIDTH / 2 - (frameW * scale) / 2;
    const dy = 200 + homeBirdBobY;
    ctx.drawImage(birdImg, homeBirdFrame * frameW, 0, frameW, frameH,
                  dx, dy, frameW * scale, frameH * scale);
  }

  // Play button
  const btnW = 120;
  const btnH = 50;
  const btnX = (GAME_WIDTH - btnW) / 2;
  const btnY = 330;

  ctx.fillStyle = '#4CAF50';
  ctx.fillRect(btnX, btnY, btnW, btnH);
  ctx.strokeStyle = '#2E7D32';
  ctx.lineWidth = 3;
  ctx.strokeRect(btnX, btnY, btnW, btnH);
  ctx.fillStyle = '#FFF';
  ctx.font = 'bold 24px monospace';
  ctx.fillText('PLAY', GAME_WIDTH / 2, btnY + 34);

  // Best score
  const best = localStorage.getItem('flappyBest');
  if (best) {
    ctx.fillStyle = '#FFF';
    ctx.font = '16px monospace';
    ctx.fillText('BEST: ' + best, GAME_WIDTH / 2, 420);
  }

  // Store button bounds for click detection
  window._playBtn = { x: btnX, y: btnY, w: btnW, h: btnH };
}
```

**Step 3: Add input handler for home screen**

```javascript
// === INPUT ===
function handleInput(e) {
  e.preventDefault();
  const rect = canvas.getBoundingClientRect();
  const scaleX = GAME_WIDTH / rect.width;
  const scaleY = GAME_HEIGHT / rect.height;

  let clickX, clickY;
  if (e.type === 'touchstart') {
    clickX = (e.touches[0].clientX - rect.left) * scaleX;
    clickY = (e.touches[0].clientY - rect.top) * scaleY;
  } else {
    clickX = (e.clientX - rect.left) * scaleX;
    clickY = (e.clientY - rect.top) * scaleY;
  }

  if (gameState === 'home') {
    const btn = window._playBtn;
    if (btn && clickX >= btn.x && clickX <= btn.x + btn.w &&
        clickY >= btn.y && clickY <= btn.y + btn.h) {
      startGame();
    }
  } else if (gameState === 'playing') {
    flap();
  } else if (gameState === 'gameover') {
    handleGameOverClick(clickX, clickY);
  }
}

canvas.addEventListener('click', handleInput);
canvas.addEventListener('touchstart', handleInput, { passive: false });
document.addEventListener('keydown', (e) => {
  if (e.code === 'Space') {
    e.preventDefault();
    if (gameState === 'home') startGame();
    else if (gameState === 'playing') flap();
  }
});

function startGame() {
  gameState = 'playing';
  resetGame();
}

function resetGame() {
  // Will be filled in Task 5
}

function flap() {
  // Will be filled in Task 5
}

function handleGameOverClick(x, y) {
  // Will be filled in Task 10
}
```

**Step 4: Wire home screen into game loop**

Update the `case 'home':` in gameLoop:
```javascript
case 'home':
  drawHomeScreen(dt);
  break;
```

**Step 5: Verify**

Open in browser. After loading completes, should see:
- Background1 image filling screen
- Scrolling ground tiles at bottom
- "FLAPPY BIRD" title with bird bobbing
- Green PLAY button
- Clicking PLAY changes gameState to 'playing'

**Step 6: Commit**

```bash
git add index.html
git commit -m "feat: home screen with animated bird, play button, and scrolling ground"
```

---

### Task 4: Level Configuration System

**Files:**
- Modify: `index.html`

**Step 1: Add level definitions**

```javascript
// === LEVEL CONFIG ===
const LEVELS = [
  { bg: 'bg1', tileStyle: 1, pipeColor: 0, bird: 'bird1_1', speed: 2.0, gap: 130, spacing: 220 },
  { bg: 'bg2', tileStyle: 2, pipeColor: 1, bird: 'bird1_2', speed: 2.2, gap: 125, spacing: 210 },
  { bg: 'bg3', tileStyle: 3, pipeColor: 2, bird: 'bird1_3', speed: 2.4, gap: 120, spacing: 200 },
  { bg: 'bg4', tileStyle: 4, pipeColor: 3, bird: 'bird2_1', speed: 2.6, gap: 115, spacing: 195 },
  { bg: 'bg5', tileStyle: 5, pipeColor: 4, bird: 'bird2_2', speed: 2.8, gap: 110, spacing: 190 },
  { bg: 'bg6', tileStyle: 1, pipeColor: 5, bird: 'bird2_3', speed: 3.0, gap: 105, spacing: 185 },
  { bg: 'bg7', tileStyle: 2, pipeColor: 6, bird: 'bird1_4', speed: 3.2, gap: 100, spacing: 180 },
  { bg: 'bg8', tileStyle: 3, pipeColor: 7, bird: 'bird2_4', speed: 3.4, gap: 95,  spacing: 175 },
  { bg: 'bg9', tileStyle: 4, pipeColor: 0, bird: 'bird1_5', speed: 3.6, gap: 90,  spacing: 170 },
];

// Pipe color extraction from spritesheets
// PipeStyle1: 128x160, 4 cols x 8 rows => each cell 32x20
// PipeStyle2-5: 128x96, 4 cols x 4-6 rows => each cell 32x16-24
// We extract a single colored pipe body strip for each color index
const PIPE_LAYOUTS = {
  1: { cols: 4, cellW: 32, cellH: 20, sheetW: 128, sheetH: 160 },
  2: { cols: 4, cellW: 32, cellH: 16, sheetW: 128, sheetH: 96 },
  3: { cols: 4, cellW: 32, cellH: 16, sheetW: 128, sheetH: 96 },
  4: { cols: 4, cellW: 32, cellH: 16, sheetW: 128, sheetH: 96 },
  5: { cols: 4, cellW: 32, cellH: 16, sheetW: 128, sheetH: 96 },
};

function getCurrentLevel() {
  const levelIndex = Math.floor(score / 15) % LEVELS.length;
  return LEVELS[levelIndex];
}

function getDifficulty() {
  const totalLevel = Math.floor(score / 15);
  if (totalLevel < LEVELS.length) return LEVELS[totalLevel];
  // On loops, cap difficulty at level 9 values
  return { ...getCurrentLevel(), speed: 3.6, gap: 90, spacing: 170 };
}
```

**Step 2: Verify**

Console test: `getCurrentLevel()` should return level 1 config when score=0, level 2 when score=15, etc.

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: level configuration system with 9 levels and difficulty scaling"
```

---

### Task 5: Bird Physics + Rendering

**Files:**
- Modify: `index.html`

**Step 1: Add bird state and physics**

```javascript
// === BIRD STATE ===
let bird = {
  x: 80,
  y: GAME_HEIGHT / 2,
  velocity: 0,
  frame: 0,
  frameTimer: 0,
  rotation: 0,
  width: 16,
  height: 16,
  scale: 2.5,
};

const GRAVITY = 0.4;
const FLAP_FORCE = -6.5;
const MAX_FALL_SPEED = 8;
const BIRD_FRAME_COUNT = 4;
const BIRD_FRAME_INTERVAL = 100; // ms

function flap() {
  bird.velocity = FLAP_FORCE;
  playSound('flap');
}

function updateBird(dt) {
  bird.velocity += GRAVITY;
  if (bird.velocity > MAX_FALL_SPEED) bird.velocity = MAX_FALL_SPEED;
  bird.y += bird.velocity;

  // Rotation based on velocity
  bird.rotation = Math.min(bird.velocity * 4, 90);
  if (bird.velocity < 0) bird.rotation = Math.max(bird.velocity * 6, -30);

  // Animation
  bird.frameTimer += dt;
  if (bird.frameTimer > BIRD_FRAME_INTERVAL) {
    bird.frameTimer = 0;
    bird.frame = (bird.frame + 1) % BIRD_FRAME_COUNT;
  }
}

function drawBird() {
  const level = getCurrentLevel();
  const birdImg = assets[level.bird];
  if (!birdImg) return;

  const fw = bird.width;
  const fh = bird.height;
  const s = bird.scale;
  const dx = bird.x - (fw * s) / 2;
  const dy = bird.y - (fh * s) / 2;

  ctx.save();
  ctx.translate(bird.x, bird.y);
  ctx.rotate((bird.rotation * Math.PI) / 180);
  ctx.drawImage(birdImg, bird.frame * fw, 0, fw, fh,
                -(fw * s) / 2, -(fh * s) / 2, fw * s, fh * s);
  ctx.restore();
}
```

**Step 2: Implement resetGame**

```javascript
function resetGame() {
  bird.y = GAME_HEIGHT / 2;
  bird.velocity = 0;
  bird.rotation = 0;
  bird.frame = 0;
  pipes = [];
  score = 0;
  pipesPassed = 0;
  groundX = 0;
  pipeSpawnTimer = 0;
  gameStarted = false; // wait for first tap
}
```

**Step 3: Verify**

Set gameState to 'playing' temporarily, see bird appear and fall with gravity. Press space to flap upward.

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: bird physics with gravity, flap, rotation, and sprite animation"
```

---

### Task 6: Pipe System

**Files:**
- Modify: `index.html`

**Step 1: Add pipe state and spawning**

```javascript
// === PIPES ===
let pipes = [];
let pipeSpawnTimer = 0;
let gameStarted = false;

function spawnPipe() {
  const diff = getDifficulty();
  const minY = 80;
  const maxY = GAME_HEIGHT - 56 - diff.gap - minY; // 56 = ground height
  const gapY = minY + Math.random() * maxY;

  pipes.push({
    x: GAME_WIDTH + 10,
    gapY: gapY,
    gapH: diff.gap,
    width: 52,
    scored: false,
  });
}

function updatePipes(dt) {
  if (!gameStarted) return;

  const diff = getDifficulty();
  pipeSpawnTimer += diff.speed;
  if (pipeSpawnTimer >= diff.spacing) {
    pipeSpawnTimer = 0;
    spawnPipe();
  }

  for (let i = pipes.length - 1; i >= 0; i--) {
    pipes[i].x -= diff.speed;

    // Score when bird passes pipe
    if (!pipes[i].scored && pipes[i].x + pipes[i].width < bird.x) {
      pipes[i].scored = true;
      score++;
      pipesPassed++;
      playSound('score');
      // Check level change
      if (score % 15 === 0) {
        playSound('levelUp');
      }
    }

    // Remove off-screen pipes
    if (pipes[i].x + pipes[i].width < -10) {
      pipes.splice(i, 1);
    }
  }
}

function drawPipes() {
  const level = getCurrentLevel();
  const pipeSheet = assets[`pipe${level.tileStyle}`];

  pipes.forEach(pipe => {
    if (pipeSheet) {
      // Extract pipe color from spritesheet
      const layout = PIPE_LAYOUTS[level.tileStyle];
      const colorIdx = level.pipeColor;
      const col = colorIdx % layout.cols;
      const row = Math.floor(colorIdx / layout.cols);
      const sx = col * layout.cellW;
      const sy = row * layout.cellH;

      // Draw top pipe (flipped)
      const topH = pipe.gapY;
      for (let y = 0; y < topH; y += layout.cellH * 2) {
        const drawH = Math.min(layout.cellH * 2, topH - y);
        ctx.drawImage(pipeSheet, sx, sy, layout.cellW, layout.cellH,
                      pipe.x, y, pipe.width, drawH);
      }
      // Pipe cap (top, wider)
      ctx.fillStyle = '#333';
      ctx.fillRect(pipe.x - 3, pipe.gapY - 12, pipe.width + 6, 12);

      // Draw bottom pipe
      const bottomY = pipe.gapY + pipe.gapH;
      for (let y = bottomY; y < GAME_HEIGHT - 56; y += layout.cellH * 2) {
        const drawH = Math.min(layout.cellH * 2, GAME_HEIGHT - 56 - y);
        ctx.drawImage(pipeSheet, sx, sy, layout.cellW, layout.cellH,
                      pipe.x, y, pipe.width, drawH);
      }
      // Pipe cap (bottom)
      ctx.fillRect(pipe.x - 3, bottomY, pipe.width + 6, 12);
    } else {
      // Fallback: green rectangles
      ctx.fillStyle = '#2E7D32';
      ctx.fillRect(pipe.x, 0, pipe.width, pipe.gapY);
      ctx.fillRect(pipe.x, pipe.gapY + pipe.gapH, pipe.width, GAME_HEIGHT);
    }
  });
}
```

**Step 2: Wire first-tap start into flap**

Update `flap()`:
```javascript
function flap() {
  if (!gameStarted) {
    gameStarted = true;
    spawnPipe(); // first pipe immediately
  }
  bird.velocity = FLAP_FORCE;
  playSound('flap');
}
```

**Step 3: Verify**

Pipes should spawn from right, move left at level speed. Passing a pipe increments score.

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: pipe spawning, movement, scoring, and sprite rendering"
```

---

### Task 7: Ground Rendering + Background

**Files:**
- Modify: `index.html`

**Step 1: Add ground and background rendering**

```javascript
// === GROUND ===
let groundX = 0;
const GROUND_HEIGHT = 56;

function drawBackground() {
  const level = getCurrentLevel();
  const bg = assets[level.bg];
  if (bg) {
    ctx.drawImage(bg, 0, 0, GAME_WIDTH, GAME_HEIGHT);
  } else {
    ctx.fillStyle = '#4EC0CA';
    ctx.fillRect(0, 0, GAME_WIDTH, GAME_HEIGHT);
  }
}

function drawGround() {
  const level = getCurrentLevel();
  const ground = assets[`simple${level.tileStyle}`];
  const gY = GAME_HEIGHT - GROUND_HEIGHT;

  if (ground) {
    const gW = ground.width * (GROUND_HEIGHT / ground.height);
    if (groundX <= -gW) groundX += gW;
    for (let x = groundX; x < GAME_WIDTH + gW; x += gW) {
      ctx.drawImage(ground, x, gY, gW, GROUND_HEIGHT);
    }
  } else {
    ctx.fillStyle = '#8B4513';
    ctx.fillRect(0, gY, GAME_WIDTH, GROUND_HEIGHT);
  }
}

function updateGround() {
  if (!gameStarted) return;
  const diff = getDifficulty();
  groundX -= diff.speed;
}
```

**Step 2: Verify**

Background fills screen with current level's image. Ground tiles scroll at pipe speed along the bottom.

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: scrolling ground tiles and level-specific backgrounds"
```

---

### Task 8: Collision Detection

**Files:**
- Modify: `index.html`

**Step 1: Add collision checks**

```javascript
// === COLLISION ===
function checkCollision() {
  const bw = bird.width * bird.scale * 0.7; // slightly smaller hitbox for fairness
  const bh = bird.height * bird.scale * 0.7;
  const bx = bird.x - bw / 2;
  const by = bird.y - bh / 2;

  // Ground collision
  if (bird.y + bh / 2 >= GAME_HEIGHT - GROUND_HEIGHT) return true;

  // Ceiling collision
  if (bird.y - bh / 2 <= 0) return true;

  // Pipe collision
  for (const pipe of pipes) {
    // Top pipe
    if (bx + bw > pipe.x && bx < pipe.x + pipe.width) {
      if (by < pipe.gapY || by + bh > pipe.gapY + pipe.gapH) {
        return true;
      }
    }
  }

  return false;
}

function die() {
  gameState = 'gameover';
  playSound('hit');

  // Save best score
  const best = parseInt(localStorage.getItem('flappyBest') || '0');
  if (score > best) {
    localStorage.setItem('flappyBest', score.toString());
  }
}
```

**Step 2: Verify**

Bird should die on hitting ground, ceiling, or pipes. gameState switches to 'gameover'.

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: collision detection for pipes, ground, and ceiling"
```

---

### Task 9: Main Gameplay Draw Loop

**Files:**
- Modify: `index.html`

**Step 1: Add score/level HUD and assemble gameplay draw**

```javascript
// === SCORE ===
let score = 0;
let pipesPassed = 0;

function drawHUD() {
  // Score (center top)
  ctx.fillStyle = '#FFF';
  ctx.strokeStyle = '#000';
  ctx.lineWidth = 3;
  ctx.font = 'bold 32px monospace';
  ctx.textAlign = 'center';
  ctx.strokeText(score.toString(), GAME_WIDTH / 2, 50);
  ctx.fillText(score.toString(), GAME_WIDTH / 2, 50);

  // Level indicator (top left)
  const lvl = Math.floor(score / 15) + 1;
  ctx.font = 'bold 14px monospace';
  ctx.textAlign = 'left';
  ctx.strokeText('LVL ' + lvl, 10, 25);
  ctx.fillText('LVL ' + lvl, 10, 25);
}

function drawPlaying(dt) {
  // Update
  updateBird(dt);
  updatePipes(dt);
  updateGround();

  // Check death
  if (gameStarted && checkCollision()) {
    die();
    return;
  }

  // Draw layers (back to front)
  drawBackground();
  drawPipes();
  drawGround();
  drawBird();
  drawHUD();

  // "Tap to start" prompt before first flap
  if (!gameStarted) {
    ctx.fillStyle = 'rgba(255,255,255,0.8)';
    ctx.font = '18px monospace';
    ctx.textAlign = 'center';
    ctx.fillText('TAP TO START', GAME_WIDTH / 2, GAME_HEIGHT / 2 + 60);
  }
}
```

**Step 2: Wire into game loop**

Update gameLoop `case 'playing':`:
```javascript
case 'playing':
  drawPlaying(dt);
  break;
```

**Step 3: Verify full gameplay**

Play through: tap to start, bird flaps, pipes scroll, score increments, level changes at 15 points (background/tiles/bird swap instantly), collision triggers game over.

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: complete gameplay loop with HUD, scoring, and level transitions"
```

---

### Task 10: Game Over Screen

**Files:**
- Modify: `index.html`

**Step 1: Add game over screen**

```javascript
function drawGameOver(dt) {
  // Draw frozen game state underneath
  drawBackground();
  drawPipes();
  drawGround();
  drawBird();

  // Dark overlay
  ctx.fillStyle = 'rgba(0, 0, 0, 0.6)';
  ctx.fillRect(0, 0, GAME_WIDTH, GAME_HEIGHT);

  // Stats panel
  const panelW = 220;
  const panelH = 260;
  const panelX = (GAME_WIDTH - panelW) / 2;
  const panelY = 80;

  // Panel background
  ctx.fillStyle = '#DEB887';
  ctx.fillRect(panelX, panelY, panelW, panelH);
  ctx.strokeStyle = '#8B4513';
  ctx.lineWidth = 4;
  ctx.strokeRect(panelX, panelY, panelW, panelH);

  // "GAME OVER" title
  ctx.fillStyle = '#FF0000';
  ctx.font = 'bold 24px monospace';
  ctx.textAlign = 'center';
  ctx.fillText('GAME OVER', GAME_WIDTH / 2, panelY + 35);

  // Stats
  ctx.fillStyle = '#333';
  ctx.font = '16px monospace';
  const best = parseInt(localStorage.getItem('flappyBest') || '0');
  const lvl = Math.floor(score / 15) + 1;

  ctx.textAlign = 'left';
  const sx = panelX + 20;
  ctx.fillText('Score:', sx, panelY + 75);
  ctx.fillText('Best:', sx, panelY + 105);
  ctx.fillText('Level:', sx, panelY + 135);
  ctx.fillText('Pipes:', sx, panelY + 165);

  ctx.textAlign = 'right';
  const rx = panelX + panelW - 20;
  ctx.font = 'bold 16px monospace';
  ctx.fillText(score.toString(), rx, panelY + 75);
  ctx.fillText(best.toString(), rx, panelY + 105);
  ctx.fillText(lvl.toString(), rx, panelY + 135);
  ctx.fillText(pipesPassed.toString(), rx, panelY + 165);

  // Retry button
  const retryBtn = { x: panelX + 10, y: panelY + 190, w: 95, h: 45 };
  ctx.fillStyle = '#4CAF50';
  ctx.fillRect(retryBtn.x, retryBtn.y, retryBtn.w, retryBtn.h);
  ctx.strokeStyle = '#2E7D32';
  ctx.lineWidth = 2;
  ctx.strokeRect(retryBtn.x, retryBtn.y, retryBtn.w, retryBtn.h);
  ctx.fillStyle = '#FFF';
  ctx.font = 'bold 16px monospace';
  ctx.textAlign = 'center';
  ctx.fillText('RETRY', retryBtn.x + retryBtn.w / 2, retryBtn.y + 30);

  // Home button
  const homeBtn = { x: panelX + panelW - 105, y: panelY + 190, w: 95, h: 45 };
  ctx.fillStyle = '#2196F3';
  ctx.fillRect(homeBtn.x, homeBtn.y, homeBtn.w, homeBtn.h);
  ctx.strokeStyle = '#1565C0';
  ctx.strokeRect(homeBtn.x, homeBtn.y, homeBtn.w, homeBtn.h);
  ctx.fillStyle = '#FFF';
  ctx.fillText('HOME', homeBtn.x + homeBtn.w / 2, homeBtn.y + 30);

  window._retryBtn = retryBtn;
  window._homeBtn = homeBtn;
}

function handleGameOverClick(x, y) {
  const r = window._retryBtn;
  const h = window._homeBtn;
  if (r && x >= r.x && x <= r.x + r.w && y >= r.y && y <= r.y + r.h) {
    gameState = 'playing';
    resetGame();
  }
  if (h && x >= h.x && x <= h.x + h.w && y >= h.y && y <= h.y + h.h) {
    gameState = 'home';
  }
}
```

**Step 2: Wire into game loop**

```javascript
case 'gameover':
  drawGameOver(dt);
  break;
```

**Step 3: Verify**

Die -> see overlay with stats (score, best, level, pipes), retry restarts game, home goes to title.

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: game over screen with stats panel, retry and home buttons"
```

---

### Task 11: Audio System

**Files:**
- Modify: `index.html`

**Step 1: Add Web Audio API sound system**

```javascript
// === AUDIO ===
let audioCtx = null;
let musicOsc = null;
let musicGain = null;

function initAudio() {
  if (audioCtx) return;
  audioCtx = new (window.AudioContext || window.webkitAudioContext)();
}

function playSound(type) {
  if (!audioCtx) initAudio();
  if (!audioCtx) return;

  const now = audioCtx.currentTime;

  switch (type) {
    case 'flap': {
      const osc = audioCtx.createOscillator();
      const gain = audioCtx.createGain();
      osc.type = 'sine';
      osc.frequency.setValueAtTime(400, now);
      osc.frequency.exponentialRampToValueAtTime(800, now + 0.08);
      gain.gain.setValueAtTime(0.15, now);
      gain.gain.exponentialRampToValueAtTime(0.001, now + 0.1);
      osc.connect(gain).connect(audioCtx.destination);
      osc.start(now);
      osc.stop(now + 0.1);
      break;
    }
    case 'score': {
      const osc = audioCtx.createOscillator();
      const gain = audioCtx.createGain();
      osc.type = 'triangle';
      osc.frequency.setValueAtTime(880, now);
      osc.frequency.setValueAtTime(1100, now + 0.05);
      gain.gain.setValueAtTime(0.12, now);
      gain.gain.exponentialRampToValueAtTime(0.001, now + 0.15);
      osc.connect(gain).connect(audioCtx.destination);
      osc.start(now);
      osc.stop(now + 0.15);
      break;
    }
    case 'hit': {
      const osc = audioCtx.createOscillator();
      const gain = audioCtx.createGain();
      osc.type = 'sawtooth';
      osc.frequency.setValueAtTime(200, now);
      osc.frequency.exponentialRampToValueAtTime(50, now + 0.3);
      gain.gain.setValueAtTime(0.2, now);
      gain.gain.exponentialRampToValueAtTime(0.001, now + 0.3);
      osc.connect(gain).connect(audioCtx.destination);
      osc.start(now);
      osc.stop(now + 0.3);
      break;
    }
    case 'levelUp': {
      [523, 659, 784, 1047].forEach((freq, i) => {
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.type = 'square';
        osc.frequency.setValueAtTime(freq, now + i * 0.08);
        gain.gain.setValueAtTime(0.08, now + i * 0.08);
        gain.gain.exponentialRampToValueAtTime(0.001, now + i * 0.08 + 0.12);
        osc.connect(gain).connect(audioCtx.destination);
        osc.start(now + i * 0.08);
        osc.stop(now + i * 0.08 + 0.12);
      });
      break;
    }
  }
}

// Background music - simple 8-bit loop
function startMusic() {
  if (!audioCtx) initAudio();
  if (!audioCtx || musicOsc) return;

  // Simple repeating melody using a square wave
  const melody = [262, 294, 330, 294, 262, 330, 294, 262]; // C D E D C E D C
  const noteLen = 0.25;
  let noteIndex = 0;

  function playNote() {
    if (gameState !== 'playing' && gameState !== 'home') {
      musicOsc = null;
      return;
    }
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    osc.type = 'square';
    osc.frequency.setValueAtTime(melody[noteIndex % melody.length], audioCtx.currentTime);
    gain.gain.setValueAtTime(0.03, audioCtx.currentTime);
    gain.gain.setValueAtTime(0.001, audioCtx.currentTime + noteLen * 0.9);
    osc.connect(gain).connect(audioCtx.destination);
    osc.start();
    osc.stop(audioCtx.currentTime + noteLen);
    noteIndex++;
    musicOsc = setTimeout(playNote, noteLen * 1000);
  }
  playNote();
}

function stopMusic() {
  if (musicOsc) {
    clearTimeout(musicOsc);
    musicOsc = null;
  }
}
```

**Step 2: Wire audio init to first user interaction**

```javascript
// Init audio on first interaction (required by browsers)
function ensureAudio() {
  initAudio();
  startMusic();
}
// Add to startGame and handleInput
```

**Step 3: Verify**

Play the game - hear flap chirp, score ding, hit thud, level-up arpeggio, background chiptune loop.

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: Web Audio API sound effects and background chiptune music"
```

---

### Task 12: Update Loading Screen with Real Bird Sprite

**Files:**
- Modify: `index.html`

**Step 1: Update loading screen to use real bird sprite once loaded**

Replace the yellow rectangle in `drawLoadingScreen` with actual bird sprite rendering when available:

```javascript
// In drawLoadingScreen, replace the placeholder rectangle:
const loadBird = assets.bird1_1;
if (loadBird) {
  const fw = 16, fh = 16, s = 3;
  ctx.drawImage(loadBird, loadingBirdFrame * fw, 0, fw, fh,
                GAME_WIDTH / 2 - (fw * s) / 2, birdY, fw * s, fh * s);
} else {
  ctx.fillStyle = '#FFD700';
  ctx.fillRect(GAME_WIDTH / 2 - 16, birdY, 32, 32);
}
```

**Step 2: Verify**

Loading screen should show the real bird sprite once it loads (may show yellow rect briefly at start).

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: loading screen uses real bird sprite when available"
```

---

### Task 13: Mobile Polish + Responsiveness

**Files:**
- Modify: `index.html`

**Step 1: Add mobile-specific CSS and meta tags**

Ensure these are in `<head>`:
```html
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
```

Add to CSS:
```css
html, body { height: 100%; width: 100%; }
canvas { display: block; }
body { -webkit-user-select: none; user-select: none; }
```

**Step 2: Prevent default touch behaviors**

```javascript
document.addEventListener('touchmove', (e) => e.preventDefault(), { passive: false });
document.addEventListener('contextmenu', (e) => e.preventDefault());
```

**Step 3: Verify on mobile**

Open on phone browser. Canvas fills screen in portrait. Tap works for flap. No scrolling or zooming interferes.

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: mobile responsiveness, touch handling, and PWA meta tags"
```

---

### Task 14: Final Polish + Bug Fixes

**Files:**
- Modify: `index.html`

**Step 1: Add visual polish**

- Ensure loading screen bird placeholder also bobs before first asset loads
- Pipe caps should use spritesheet coloring (not plain gray)
- Add slight screen shake on death (optional, 3-frame effect)
- Ensure level text flashes briefly on level change

**Step 2: Edge case fixes**

- Prevent double-tap on game over buttons
- Ensure `resetGame` fully resets all state
- Cap deltaTime to prevent physics jumps on tab-switch (`dt = Math.min(dt, 50)`)
- Ensure score doesn't increment after death

**Step 3: Full playtest**

Test complete flow: Loading -> Home -> Play -> Score 15+ (level change) -> Die -> Game Over -> Retry -> Home. Verify on both desktop and mobile.

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: visual polish, edge case fixes, and delta time capping"
```

---

### Task 15: Deploy to play.fun

**Files:**
- The entire project

**Step 1: Install playdotfun plugin**

```bash
claude plugin marketplace add opusgamelabs/skills
claude plugin install playdotfun
```

**Step 2: Build and deploy**

Follow the playdotfun plugin's deploy workflow to publish `index.html` + `assets/` to play.fun.

**Step 3: Verify live**

Open the play.fun URL. Confirm game loads, plays, and works on both desktop and mobile browsers.

**Step 4: Commit deploy config (if any)**

```bash
git add -A
git commit -m "feat: deploy configuration for play.fun"
```

---

## Summary

| Task | Description | Estimated Size |
|------|------------|---------------|
| 1 | Project setup + asset copy | Small |
| 2 | Asset loader + loading screen | Medium |
| 3 | Home screen | Medium |
| 4 | Level configuration system | Small |
| 5 | Bird physics + rendering | Medium |
| 6 | Pipe system | Large |
| 7 | Ground + background rendering | Small |
| 8 | Collision detection | Small |
| 9 | Main gameplay loop + HUD | Medium |
| 10 | Game over screen | Medium |
| 11 | Audio system | Medium |
| 12 | Loading screen real sprite | Small |
| 13 | Mobile polish | Small |
| 14 | Final polish + bug fixes | Medium |
| 15 | Deploy to play.fun | Small |
