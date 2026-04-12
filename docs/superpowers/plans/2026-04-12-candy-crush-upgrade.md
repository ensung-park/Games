# Candy Crush World-Class Upgrade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Transform the basic match-3 prototype into a polished, addictive game with canvas rendering, special candies, level progression, obstacles, procedural audio, and full visual polish — all in a single self-contained HTML file.

**Architecture:** Complete rewrite of `candy-crush/index.html`. The game uses HTML5 Canvas for rendering with a requestAnimationFrame loop. Game state is managed by a central state object. The file is organized into clearly separated sections: constants/config, game state, audio engine, animation/particle system, rendering, game logic (matching/specials/blockers), level definitions, input handling, UI screens, and initialization.

**Tech Stack:** Vanilla JavaScript, HTML5 Canvas 2D, Web Audio API, localStorage. No external dependencies.

---

## File Structure

- **Rewrite:** `candy-crush/index.html` — the entire game in one file (~2500-3000 lines)

The file is organized into these `<script>` sections via comments:

```
// ═══════════════════════════════════════
// SECTION 1: CONSTANTS & CONFIGURATION
// ═══════════════════════════════════════
// SECTION 2: GAME STATE
// SECTION 3: AUDIO ENGINE (Web Audio API)
// SECTION 4: ANIMATION & PARTICLE SYSTEM
// SECTION 5: CANVAS RENDERING
// SECTION 6: GAME LOGIC (matching, specials, blockers, gravity)
// SECTION 7: LEVEL DEFINITIONS (30 levels)
// SECTION 8: INPUT HANDLING (mouse + touch)
// SECTION 9: UI SCREENS (level select, HUD, overlays)
// SECTION 10: GAME LOOP & INITIALIZATION
```

---

### Task 1: Canvas Bootstrap & Basic Candy Rendering

**Files:**
- Rewrite: `candy-crush/index.html`

Replace the entire file with the canvas-based foundation. This task creates the HTML shell, canvas element, constants, game state structure, and renders a static 8x8 grid of colored candies.

- [ ] **Step 1: Write the HTML shell with canvas**

Replace `candy-crush/index.html` with:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
<title>Candy Crush Match-3</title>
<style>
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
  background: #1a0533;
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
  font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
  -webkit-user-select: none;
  user-select: none;
  touch-action: none;
}
canvas {
  display: block;
  image-rendering: auto;
}
</style>
</head>
<body>
<canvas id="gc"></canvas>
<script>
'use strict';

// ═══════════════════════════════════════════════════════
// SECTION 1: CONSTANTS & CONFIGURATION
// ═══════════════════════════════════════════════════════

const ROWS = 8, COLS = 8;
const CANDY_TYPES = 6;
const EMPTY = -1;
const COLORS = [
  { main: '#ff4757', dark: '#c44569', light: '#ff6b81' }, // Red
  { main: '#ffa502', dark: '#e67e22', light: '#ffc048' }, // Orange
  { main: '#2ed573', dark: '#009432', light: '#7bed9f' }, // Green
  { main: '#1e90ff', dark: '#0652DD', light: '#70a1ff' }, // Blue
  { main: '#a55eea', dark: '#8854d0', light: '#c298f0' }, // Purple
  { main: '#ff6348', dark: '#ee5a24', light: '#ff9478' }, // Coral
];

// Special candy types
const SPECIAL = { NONE: 0, STRIPED_H: 1, STRIPED_V: 2, WRAPPED: 3, COLOR_BOMB: 4 };

// Blocker types
const BLOCKER = { NONE: 0, ICE1: 1, ICE2: 2, CHOCOLATE: 3, STONE: 4 };

// Candy shape silhouettes for accessibility
const SHAPES = ['circle', 'diamond', 'square', 'triangle', 'pentagon', 'heart'];

// Game screens
const SCREEN = { LEVEL_SELECT: 0, PLAYING: 1, LEVEL_COMPLETE: 2, LEVEL_FAILED: 3 };

const canvas = document.getElementById('gc');
const ctx = canvas.getContext('2d');

let W, H, cellSize, boardX, boardY, dpr;

function resize() {
  dpr = window.devicePixelRatio || 1;
  W = window.innerWidth;
  H = window.innerHeight;
  canvas.width = W * dpr;
  canvas.height = H * dpr;
  canvas.style.width = W + 'px';
  canvas.style.height = H + 'px';
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  cellSize = Math.floor(Math.min(W * 0.85, H * 0.55, 400) / COLS);
  boardX = Math.floor((W - cellSize * COLS) / 2);
  boardY = Math.floor(H * 0.22);
}

window.addEventListener('resize', resize);
resize();

// ═══════════════════════════════════════════════════════
// SECTION 2: GAME STATE
// ═══════════════════════════════════════════════════════

const state = {
  screen: SCREEN.LEVEL_SELECT,
  board: [],         // board[r][c] = { type: 0-5, special: SPECIAL.x, blocker: BLOCKER.x }
  score: 0,
  movesLeft: 0,
  totalMoves: 0,
  level: 1,
  cascade: 0,
  selected: null,    // { r, c } or null
  animating: false,
  hintTimer: 0,
  hintMove: null,
  particles: [],
  floatingTexts: [],
  animations: [],    // active tween animations
  shakeAmount: 0,
  levelData: null,
  stars: 0,
  progress: {},      // { levelNum: starsEarned } from localStorage
};

function makeCell(type, special, blocker) {
  return { type: type, special: special || SPECIAL.NONE, blocker: blocker || BLOCKER.NONE };
}

function loadProgress() {
  try {
    const saved = localStorage.getItem('candyCrushProgress');
    if (saved) state.progress = JSON.parse(saved);
  } catch(e) {}
}

function saveProgress() {
  try {
    localStorage.setItem('candyCrushProgress', JSON.stringify(state.progress));
  } catch(e) {}
}

loadProgress();

// ═══════════════════════════════════════════════════════
// SECTION 3: AUDIO ENGINE (Web Audio API)
// ═══════════════════════════════════════════════════════

let audioCtx = null;
let soundEnabled = true;

function ensureAudio() {
  if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  if (audioCtx.state === 'suspended') audioCtx.resume();
}

function playTone(freq, duration, type, vol, delay) {
  if (!soundEnabled || !audioCtx) return;
  const t = audioCtx.currentTime + (delay || 0);
  const osc = audioCtx.createOscillator();
  const gain = audioCtx.createGain();
  osc.type = type || 'sine';
  osc.frequency.setValueAtTime(freq, t);
  gain.gain.setValueAtTime(vol || 0.15, t);
  gain.gain.exponentialRampToValueAtTime(0.001, t + duration);
  osc.connect(gain);
  gain.connect(audioCtx.destination);
  osc.start(t);
  osc.stop(t + duration);
}

function playNoise(duration, vol, delay) {
  if (!soundEnabled || !audioCtx) return;
  const t = audioCtx.currentTime + (delay || 0);
  const bufferSize = audioCtx.sampleRate * duration;
  const buffer = audioCtx.createBuffer(1, bufferSize, audioCtx.sampleRate);
  const data = buffer.getChannelData(0);
  for (let i = 0; i < bufferSize; i++) data[i] = Math.random() * 2 - 1;
  const src = audioCtx.createBufferSource();
  src.buffer = buffer;
  const gain = audioCtx.createGain();
  gain.gain.setValueAtTime(vol || 0.1, t);
  gain.gain.exponentialRampToValueAtTime(0.001, t + duration);
  src.connect(gain);
  gain.connect(audioCtx.destination);
  src.start(t);
  src.stop(t + duration);
}

function sfxMatch(cascadeLevel) {
  const baseFreq = 400 + cascadeLevel * 80;
  playTone(baseFreq, 0.15, 'sine', 0.12);
  playTone(baseFreq * 1.25, 0.12, 'sine', 0.08, 0.05);
}

function sfxSpecialCreate() {
  playTone(500, 0.1, 'sine', 0.1);
  playTone(700, 0.1, 'sine', 0.1, 0.08);
  playTone(900, 0.15, 'sine', 0.1, 0.16);
}

function sfxExplosion() {
  playNoise(0.25, 0.15);
  playTone(120, 0.3, 'sine', 0.12);
}

function sfxSwap() {
  playTone(300, 0.08, 'sine', 0.08);
  playTone(400, 0.08, 'sine', 0.08, 0.04);
}

function sfxBadSwap() {
  playTone(200, 0.12, 'square', 0.06);
  playTone(150, 0.15, 'square', 0.06, 0.1);
}

function sfxWin() {
  [523, 659, 784, 1047].forEach((f, i) => playTone(f, 0.3, 'sine', 0.1, i * 0.12));
}

function sfxLose() {
  [400, 350, 300, 250].forEach((f, i) => playTone(f, 0.3, 'sine', 0.08, i * 0.15));
}

function sfxClick() {
  playTone(600, 0.05, 'sine', 0.06);
}

// ═══════════════════════════════════════════════════════
// SECTION 4: ANIMATION & PARTICLE SYSTEM
// ═══════════════════════════════════════════════════════

function addParticle(x, y, color, count) {
  for (let i = 0; i < (count || 8); i++) {
    const angle = Math.random() * Math.PI * 2;
    const speed = 50 + Math.random() * 150;
    state.particles.push({
      x, y,
      vx: Math.cos(angle) * speed,
      vy: Math.sin(angle) * speed,
      life: 0.5 + Math.random() * 0.5,
      maxLife: 0.5 + Math.random() * 0.5,
      size: 2 + Math.random() * 4,
      color: color,
    });
  }
}

function addFloatingText(x, y, text, color) {
  state.floatingTexts.push({
    x, y, text, color: color || '#fff',
    life: 1.0, maxLife: 1.0,
    vy: -60,
  });
}

function updateParticles(dt) {
  for (let i = state.particles.length - 1; i >= 0; i--) {
    const p = state.particles[i];
    p.x += p.vx * dt;
    p.y += p.vy * dt;
    p.vy += 200 * dt; // gravity
    p.life -= dt;
    if (p.life <= 0) state.particles.splice(i, 1);
  }
  for (let i = state.floatingTexts.length - 1; i >= 0; i--) {
    const ft = state.floatingTexts[i];
    ft.y += ft.vy * dt;
    ft.life -= dt;
    if (ft.life <= 0) state.floatingTexts.splice(i, 1);
  }
  if (state.shakeAmount > 0) state.shakeAmount *= 0.9;
  if (state.shakeAmount < 0.5) state.shakeAmount = 0;
}

// Tween system
function addAnimation(obj, props, duration, easing, onDone) {
  const anim = { obj, props: {}, duration, elapsed: 0, easing: easing || 'easeOut', onDone };
  for (const key in props) {
    anim.props[key] = { from: obj[key], to: props[key] };
  }
  state.animations.push(anim);
  return anim;
}

function easeOut(t) { return 1 - (1 - t) * (1 - t); }
function easeIn(t) { return t * t; }
function easeInOut(t) { return t < 0.5 ? 2*t*t : 1-(-2*t+2)*(-2*t+2)/2; }
function easeBounce(t) {
  if (t < 0.5) return 4*t*t;
  const b = (t - 0.75) * 4;
  return 1 - b * b * 0.25;
}

function updateAnimations(dt) {
  for (let i = state.animations.length - 1; i >= 0; i--) {
    const a = state.animations[i];
    a.elapsed += dt;
    let t = Math.min(a.elapsed / a.duration, 1);
    const ease = a.easing === 'easeIn' ? easeIn : a.easing === 'easeInOut' ? easeInOut : a.easing === 'bounce' ? easeBounce : easeOut;
    const et = ease(t);
    for (const key in a.props) {
      const p = a.props[key];
      a.obj[key] = p.from + (p.to - p.from) * et;
    }
    if (t >= 1) {
      state.animations.splice(i, 1);
      if (a.onDone) a.onDone();
    }
  }
}

// ═══════════════════════════════════════════════════════
// SECTION 5: CANVAS RENDERING
// ═══════════════════════════════════════════════════════

function cellToScreen(r, c) {
  return {
    x: boardX + c * cellSize + cellSize / 2,
    y: boardY + r * cellSize + cellSize / 2,
  };
}

function drawRoundedRect(x, y, w, h, radius) {
  ctx.beginPath();
  ctx.moveTo(x + radius, y);
  ctx.lineTo(x + w - radius, y);
  ctx.quadraticCurveTo(x + w, y, x + w, y + radius);
  ctx.lineTo(x + w, y + h - radius);
  ctx.quadraticCurveTo(x + w, y + h, x + w - radius, y + h);
  ctx.lineTo(x + radius, y + h);
  ctx.quadraticCurveTo(x, y + h, x, y + h - radius);
  ctx.lineTo(x, y + radius);
  ctx.quadraticCurveTo(x, y, x + radius, y);
  ctx.closePath();
}

function drawCandy(x, y, size, type, special) {
  if (type < 0 || type >= CANDY_TYPES) return;
  const col = COLORS[type];
  const r = size * 0.42;

  // Shadow
  ctx.fillStyle = 'rgba(0,0,0,0.25)';
  ctx.beginPath();
  ctx.arc(x + 2, y + 2, r, 0, Math.PI * 2);
  ctx.fill();

  // Main body with shape
  const grad = ctx.createRadialGradient(x - r*0.3, y - r*0.3, 0, x, y, r);
  grad.addColorStop(0, col.light);
  grad.addColorStop(0.6, col.main);
  grad.addColorStop(1, col.dark);
  ctx.fillStyle = grad;

  drawCandyShape(x, y, r, type);
  ctx.fill();

  // Glossy highlight
  ctx.fillStyle = 'rgba(255,255,255,0.35)';
  ctx.beginPath();
  ctx.ellipse(x - r*0.15, y - r*0.25, r*0.55, r*0.35, -0.3, 0, Math.PI * 2);
  ctx.fill();

  // Special indicators
  if (special === SPECIAL.STRIPED_H) {
    ctx.strokeStyle = 'rgba(255,255,255,0.7)';
    ctx.lineWidth = 2;
    for (let i = -2; i <= 2; i++) {
      ctx.beginPath();
      ctx.moveTo(x - r * 0.7, y + i * 3);
      ctx.lineTo(x + r * 0.7, y + i * 3);
      ctx.stroke();
    }
  } else if (special === SPECIAL.STRIPED_V) {
    ctx.strokeStyle = 'rgba(255,255,255,0.7)';
    ctx.lineWidth = 2;
    for (let i = -2; i <= 2; i++) {
      ctx.beginPath();
      ctx.moveTo(x + i * 3, y - r * 0.7);
      ctx.lineTo(x + i * 3, y + r * 0.7);
      ctx.stroke();
    }
  } else if (special === SPECIAL.WRAPPED) {
    ctx.strokeStyle = 'rgba(255,255,255,0.8)';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.arc(x, y, r * 0.55, 0, Math.PI * 2);
    ctx.stroke();
    ctx.beginPath();
    ctx.arc(x, y, r * 0.3, 0, Math.PI * 2);
    ctx.stroke();
  }
}

function drawCandyShape(x, y, r, type) {
  ctx.beginPath();
  switch(SHAPES[type]) {
    case 'circle':
      ctx.arc(x, y, r, 0, Math.PI * 2);
      break;
    case 'diamond':
      ctx.moveTo(x, y - r);
      ctx.lineTo(x + r * 0.85, y);
      ctx.lineTo(x, y + r);
      ctx.lineTo(x - r * 0.85, y);
      ctx.closePath();
      break;
    case 'square':
      drawRoundedRect(x - r * 0.78, y - r * 0.78, r * 1.56, r * 1.56, r * 0.2);
      break;
    case 'triangle':
      ctx.moveTo(x, y - r);
      ctx.lineTo(x + r * 0.9, y + r * 0.7);
      ctx.lineTo(x - r * 0.9, y + r * 0.7);
      ctx.closePath();
      break;
    case 'pentagon': {
      for (let i = 0; i < 5; i++) {
        const angle = -Math.PI/2 + i * Math.PI * 2 / 5;
        const px = x + Math.cos(angle) * r;
        const py = y + Math.sin(angle) * r;
        if (i === 0) ctx.moveTo(px, py); else ctx.lineTo(px, py);
      }
      ctx.closePath();
      break;
    }
    case 'heart': {
      ctx.moveTo(x, y + r * 0.7);
      ctx.bezierCurveTo(x - r * 1.2, y - r * 0.2, x - r * 0.6, y - r * 1.0, x, y - r * 0.4);
      ctx.bezierCurveTo(x + r * 0.6, y - r * 1.0, x + r * 1.2, y - r * 0.2, x, y + r * 0.7);
      break;
    }
  }
}

function drawColorBomb(x, y, size) {
  const r = size * 0.42;
  // Rainbow swirl
  for (let i = 0; i < 6; i++) {
    const angle = (i / 6) * Math.PI * 2 + performance.now() * 0.002;
    ctx.fillStyle = COLORS[i].main;
    ctx.beginPath();
    ctx.moveTo(x, y);
    ctx.arc(x, y, r, angle, angle + Math.PI / 3);
    ctx.closePath();
    ctx.fill();
  }
  // White center
  ctx.fillStyle = '#fff';
  ctx.beginPath();
  ctx.arc(x, y, r * 0.3, 0, Math.PI * 2);
  ctx.fill();
  // Sparkle
  ctx.fillStyle = 'rgba(255,255,255,0.6)';
  ctx.beginPath();
  ctx.ellipse(x - r*0.15, y - r*0.2, r*0.4, r*0.25, -0.3, 0, Math.PI * 2);
  ctx.fill();
}

function drawBlocker(x, y, size, blocker) {
  const r = size * 0.45;
  if (blocker === BLOCKER.ICE1 || blocker === BLOCKER.ICE2) {
    ctx.strokeStyle = blocker === BLOCKER.ICE2 ? 'rgba(150,220,255,0.8)' : 'rgba(150,220,255,0.4)';
    ctx.lineWidth = blocker === BLOCKER.ICE2 ? 3 : 2;
    drawRoundedRect(x - r, y - r, r*2, r*2, 6);
    ctx.stroke();
    if (blocker === BLOCKER.ICE2) {
      ctx.fillStyle = 'rgba(180,230,255,0.15)';
      drawRoundedRect(x - r, y - r, r*2, r*2, 6);
      ctx.fill();
    }
  } else if (blocker === BLOCKER.CHOCOLATE) {
    ctx.fillStyle = '#5c3317';
    drawRoundedRect(x - r, y - r, r*2, r*2, 6);
    ctx.fill();
    ctx.fillStyle = '#3e210e';
    for (let i = 0; i < 4; i++) {
      const cx = x - r*0.4 + Math.random()*r*0.8;
      const cy = y - r*0.4 + Math.random()*r*0.8;
      ctx.beginPath();
      ctx.arc(cx, cy, 2 + Math.random()*3, 0, Math.PI*2);
      ctx.fill();
    }
  } else if (blocker === BLOCKER.STONE) {
    ctx.fillStyle = '#666';
    drawRoundedRect(x - r, y - r, r*2, r*2, 4);
    ctx.fill();
    ctx.strokeStyle = '#888';
    ctx.lineWidth = 1;
    drawRoundedRect(x - r, y - r, r*2, r*2, 4);
    ctx.stroke();
    // crack lines
    ctx.strokeStyle = '#555';
    ctx.beginPath();
    ctx.moveTo(x - r*0.5, y - r*0.3);
    ctx.lineTo(x + r*0.2, y + r*0.1);
    ctx.lineTo(x + r*0.5, y + r*0.5);
    ctx.stroke();
  }
}

function drawBoard() {
  if (state.screen !== SCREEN.PLAYING) return;
  const shk = state.shakeAmount;
  const sx = shk ? (Math.random() - 0.5) * shk : 0;
  const sy = shk ? (Math.random() - 0.5) * shk : 0;

  // Board background
  ctx.fillStyle = 'rgba(0,0,0,0.3)';
  drawRoundedRect(boardX - 8 + sx, boardY - 8 + sy, cellSize * COLS + 16, cellSize * ROWS + 16, 16);
  ctx.fill();

  // Grid cells
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      const { x, y } = cellToScreen(r, c);
      const cx = x + sx, cy = y + sy;

      // Cell background
      ctx.fillStyle = (r + c) % 2 === 0 ? 'rgba(255,255,255,0.04)' : 'rgba(255,255,255,0.02)';
      drawRoundedRect(cx - cellSize/2 + 1, cy - cellSize/2 + 1, cellSize - 2, cellSize - 2, 6);
      ctx.fill();

      const cell = state.board[r] && state.board[r][c];
      if (!cell) continue;

      // Draw blocker background
      if (cell.blocker === BLOCKER.STONE) {
        drawBlocker(cx, cy, cellSize, BLOCKER.STONE);
        continue;
      }
      if (cell.blocker === BLOCKER.CHOCOLATE) {
        drawBlocker(cx, cy, cellSize, BLOCKER.CHOCOLATE);
        continue;
      }

      // Draw candy
      if (cell.type >= 0) {
        const drawSize = cell.scale !== undefined ? cellSize * cell.scale : cellSize;
        const drawX = cell.drawX !== undefined ? cell.drawX + sx : cx;
        const drawY = cell.drawY !== undefined ? cell.drawY + sy : cy;

        if (cell.special === SPECIAL.COLOR_BOMB) {
          drawColorBomb(drawX, drawY, drawSize);
        } else {
          drawCandy(drawX, drawY, drawSize, cell.type, cell.special);
        }
      }

      // Draw ice overlay
      if (cell.blocker === BLOCKER.ICE1 || cell.blocker === BLOCKER.ICE2) {
        drawBlocker(cx, cy, cellSize, cell.blocker);
      }
    }
  }

  // Selected cell highlight
  if (state.selected) {
    const { x, y } = cellToScreen(state.selected.r, state.selected.c);
    ctx.strokeStyle = '#fff';
    ctx.lineWidth = 3;
    ctx.shadowColor = '#fff';
    ctx.shadowBlur = 12;
    drawRoundedRect(x - cellSize/2 + 2 + sx, y - cellSize/2 + 2 + sy, cellSize - 4, cellSize - 4, 8);
    ctx.stroke();
    ctx.shadowBlur = 0;
  }

  // Hint highlight
  if (state.hintMove && !state.selected && !state.animating) {
    const pulse = 0.5 + 0.5 * Math.sin(performance.now() * 0.005);
    ctx.strokeStyle = `rgba(255,255,100,${0.3 + pulse * 0.4})`;
    ctx.lineWidth = 2;
    const { x: hx, y: hy } = cellToScreen(state.hintMove.r, state.hintMove.c);
    drawRoundedRect(hx - cellSize/2 + 3 + sx, hy - cellSize/2 + 3 + sy, cellSize - 6, cellSize - 6, 8);
    ctx.stroke();
  }
}

function drawParticles() {
  for (const p of state.particles) {
    const alpha = p.life / p.maxLife;
    ctx.globalAlpha = alpha;
    ctx.fillStyle = p.color;
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.size * alpha, 0, Math.PI * 2);
    ctx.fill();
  }
  ctx.globalAlpha = 1;
}

function drawFloatingTexts() {
  for (const ft of state.floatingTexts) {
    const alpha = ft.life / ft.maxLife;
    ctx.globalAlpha = alpha;
    ctx.fillStyle = ft.color;
    ctx.font = `bold ${Math.floor(cellSize * 0.4)}px 'Segoe UI', system-ui, sans-serif`;
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(ft.text, ft.x, ft.y);
  }
  ctx.globalAlpha = 1;
}

function drawHUD() {
  if (state.screen !== SCREEN.PLAYING || !state.levelData) return;

  // Level title
  ctx.fillStyle = '#fff';
  ctx.font = `bold ${Math.floor(cellSize * 0.45)}px 'Segoe UI', system-ui, sans-serif`;
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText(`Level ${state.level}`, W / 2, boardY * 0.25);

  // Score
  ctx.font = `bold ${Math.floor(cellSize * 0.5)}px 'Segoe UI', system-ui, sans-serif`;
  ctx.fillText(state.score.toLocaleString(), W / 2, boardY * 0.55);

  // Moves left
  ctx.font = `${Math.floor(cellSize * 0.32)}px 'Segoe UI', system-ui, sans-serif`;
  ctx.fillStyle = state.movesLeft <= 5 ? '#ff4757' : '#ccc';
  ctx.fillText(`${state.movesLeft} moves left`, W / 2, boardY * 0.78);

  // Star thresholds progress bar
  const ld = state.levelData;
  const barW = cellSize * COLS;
  const barH = 8;
  const barX = boardX;
  const barY = boardY - 16;

  // Background
  ctx.fillStyle = 'rgba(255,255,255,0.1)';
  drawRoundedRect(barX, barY, barW, barH, 4);
  ctx.fill();

  // Fill
  const maxScore = ld.stars[2];
  const fillRatio = Math.min(state.score / maxScore, 1);
  const grad = ctx.createLinearGradient(barX, 0, barX + barW, 0);
  grad.addColorStop(0, '#ff6bca');
  grad.addColorStop(1, '#ffb347');
  ctx.fillStyle = grad;
  drawRoundedRect(barX, barY, barW * fillRatio, barH, 4);
  ctx.fill();

  // Star markers
  const starColors = ['#ccc', '#ffa502', '#ffa502'];
  for (let i = 0; i < 3; i++) {
    const sx = barX + (ld.stars[i] / maxScore) * barW;
    ctx.fillStyle = state.score >= ld.stars[i] ? '#ffa502' : '#666';
    ctx.font = `${Math.floor(cellSize * 0.3)}px sans-serif`;
    ctx.textAlign = 'center';
    ctx.fillText('★', sx, barY - 4);
  }

  // Sound toggle
  ctx.font = `${Math.floor(cellSize * 0.4)}px sans-serif`;
  ctx.textAlign = 'right';
  ctx.fillStyle = '#fff';
  ctx.fillText(soundEnabled ? '🔊' : '🔇', W - 16, 28);
}

function drawBackground() {
  const grad = ctx.createLinearGradient(0, 0, W, H);
  grad.addColorStop(0, '#1a0533');
  grad.addColorStop(0.4, '#2d1b69');
  grad.addColorStop(0.7, '#4a1a6b');
  grad.addColorStop(1, '#1a0533');
  ctx.fillStyle = grad;
  ctx.fillRect(0, 0, W, H);
}

// ═══════════════════════════════════════════════════════
// SECTION 6: GAME LOGIC
// ═══════════════════════════════════════════════════════

function randomCandy() {
  return Math.floor(Math.random() * CANDY_TYPES);
}

function createBoard(levelData) {
  state.board = [];
  for (let r = 0; r < ROWS; r++) {
    state.board[r] = [];
    for (let c = 0; c < COLS; c++) {
      const blocker = (levelData && levelData.blockers && levelData.blockers[r] && levelData.blockers[r][c]) || BLOCKER.NONE;
      if (blocker === BLOCKER.STONE) {
        state.board[r][c] = makeCell(EMPTY, SPECIAL.NONE, BLOCKER.STONE);
        continue;
      }
      if (blocker === BLOCKER.CHOCOLATE) {
        state.board[r][c] = makeCell(EMPTY, SPECIAL.NONE, BLOCKER.CHOCOLATE);
        continue;
      }
      let type;
      do {
        type = randomCandy();
      } while (
        (c >= 2 && state.board[r][c-1].type === type && state.board[r][c-2].type === type) ||
        (r >= 2 && state.board[r-1][c].type === type && state.board[r-2][c].type === type)
      );
      state.board[r][c] = makeCell(type, SPECIAL.NONE, blocker);
    }
  }
}

function isPlayable(r, c) {
  if (r < 0 || r >= ROWS || c < 0 || c >= COLS) return false;
  const cell = state.board[r][c];
  return cell && cell.type >= 0 && cell.blocker !== BLOCKER.STONE && cell.blocker !== BLOCKER.CHOCOLATE;
}

function findMatchGroups() {
  const groups = [];
  // Horizontal
  for (let r = 0; r < ROWS; r++) {
    let c = 0;
    while (c < COLS) {
      if (!isPlayable(r, c)) { c++; continue; }
      let end = c;
      while (end + 1 < COLS && isPlayable(r, end+1) && state.board[r][end+1].type === state.board[r][c].type) end++;
      if (end - c >= 2) {
        groups.push({ cells: Array.from({length: end-c+1}, (_,i) => ({r, c: c+i})), dir: 'h' });
      }
      c = end + 1;
    }
  }
  // Vertical
  for (let c = 0; c < COLS; c++) {
    let r = 0;
    while (r < ROWS) {
      if (!isPlayable(r, c)) { r++; continue; }
      let end = r;
      while (end + 1 < ROWS && isPlayable(end+1, c) && state.board[end+1][c].type === state.board[r][c].type) end++;
      if (end - r >= 2) {
        groups.push({ cells: Array.from({length: end-r+1}, (_,i) => ({r: r+i, c})), dir: 'v' });
      }
      r = end + 1;
    }
  }
  return groups;
}

function findAllMatchedCells() {
  const groups = findMatchGroups();
  const set = new Set();
  for (const g of groups) {
    for (const cell of g.cells) set.add(cell.r + ',' + cell.c);
  }
  return [...set];
}

// Detect special candy creation from match patterns
function detectSpecials(groups) {
  const specials = []; // { r, c, special, type }
  const allMatched = new Set();
  for (const g of groups) for (const cell of g.cells) allMatched.add(cell.r+','+cell.c);

  // Check for L/T shapes (intersecting groups)
  for (let i = 0; i < groups.length; i++) {
    for (let j = i+1; j < groups.length; j++) {
      if (groups[i].dir === groups[j].dir) continue;
      // Find intersection
      for (const c1 of groups[i].cells) {
        for (const c2 of groups[j].cells) {
          if (c1.r === c2.r && c1.c === c2.c) {
            specials.push({ r: c1.r, c: c1.c, special: SPECIAL.WRAPPED, type: state.board[c1.r][c1.c].type });
          }
        }
      }
    }
  }

  for (const g of groups) {
    if (g.cells.length === 5) {
      // Color bomb
      const mid = g.cells[2];
      if (!specials.some(s => s.r === mid.r && s.c === mid.c)) {
        specials.push({ r: mid.r, c: mid.c, special: SPECIAL.COLOR_BOMB, type: state.board[mid.r][mid.c].type });
      }
    } else if (g.cells.length === 4) {
      // Striped
      const mid = g.cells[1]; // Place striped at second position
      if (!specials.some(s => s.r === mid.r && s.c === mid.c)) {
        specials.push({
          r: mid.r, c: mid.c,
          special: g.dir === 'h' ? SPECIAL.STRIPED_V : SPECIAL.STRIPED_H, // perpendicular
          type: state.board[mid.r][mid.c].type,
        });
      }
    }
  }

  return specials;
}

function activateSpecial(r, c, board) {
  const cell = board[r][c];
  const toRemove = new Set();
  if (cell.special === SPECIAL.STRIPED_H) {
    for (let cc = 0; cc < COLS; cc++) toRemove.add(r + ',' + cc);
  } else if (cell.special === SPECIAL.STRIPED_V) {
    for (let rr = 0; rr < ROWS; rr++) toRemove.add(rr + ',' + c);
  } else if (cell.special === SPECIAL.WRAPPED) {
    for (let dr = -1; dr <= 1; dr++) {
      for (let dc = -1; dc <= 1; dc++) {
        const nr = r + dr, nc = c + dc;
        if (nr >= 0 && nr < ROWS && nc >= 0 && nc < COLS) toRemove.add(nr + ',' + nc);
      }
    }
  } else if (cell.special === SPECIAL.COLOR_BOMB) {
    // Will be handled during swap with target type
  }
  return toRemove;
}

function activateColorBomb(r, c, targetType) {
  const toRemove = new Set();
  toRemove.add(r + ',' + c);
  for (let rr = 0; rr < ROWS; rr++) {
    for (let cc = 0; cc < COLS; cc++) {
      if (state.board[rr][cc].type === targetType) {
        toRemove.add(rr + ',' + cc);
      }
    }
  }
  return toRemove;
}

// Handle special+special combo swaps
function handleSpecialCombo(r1, c1, r2, c2) {
  const s1 = state.board[r1][c1].special;
  const s2 = state.board[r2][c2].special;
  if (s1 === SPECIAL.NONE && s2 === SPECIAL.NONE) return null;

  const toRemove = new Set();

  // Color bomb + Color bomb = clear entire board
  if (s1 === SPECIAL.COLOR_BOMB && s2 === SPECIAL.COLOR_BOMB) {
    for (let r = 0; r < ROWS; r++)
      for (let c = 0; c < COLS; c++)
        if (state.board[r][c].type >= 0) toRemove.add(r + ',' + c);
    return toRemove;
  }

  // Color bomb + anything
  if (s1 === SPECIAL.COLOR_BOMB) {
    const targetType = state.board[r2][c2].type;
    const cells = activateColorBomb(r1, c1, targetType);
    // If other is striped, make all of that color striped then detonate
    if (s2 === SPECIAL.STRIPED_H || s2 === SPECIAL.STRIPED_V) {
      for (const key of cells) {
        const [rr, cc] = key.split(',').map(Number);
        state.board[rr][cc].special = Math.random() > 0.5 ? SPECIAL.STRIPED_H : SPECIAL.STRIPED_V;
        const extra = activateSpecial(rr, cc, state.board);
        for (const e of extra) toRemove.add(e);
      }
    }
    for (const c of cells) toRemove.add(c);
    return toRemove;
  }
  if (s2 === SPECIAL.COLOR_BOMB) {
    const targetType = state.board[r1][c1].type;
    const cells = activateColorBomb(r2, c2, targetType);
    if (s1 === SPECIAL.STRIPED_H || s1 === SPECIAL.STRIPED_V) {
      for (const key of cells) {
        const [rr, cc] = key.split(',').map(Number);
        state.board[rr][cc].special = Math.random() > 0.5 ? SPECIAL.STRIPED_H : SPECIAL.STRIPED_V;
        const extra = activateSpecial(rr, cc, state.board);
        for (const e of extra) toRemove.add(e);
      }
    }
    for (const c of cells) toRemove.add(c);
    return toRemove;
  }

  // Striped + Striped = cross
  const isStriped = s => s === SPECIAL.STRIPED_H || s === SPECIAL.STRIPED_V;
  if (isStriped(s1) && isStriped(s2)) {
    for (let cc = 0; cc < COLS; cc++) toRemove.add(r1 + ',' + cc);
    for (let rr = 0; rr < ROWS; rr++) toRemove.add(rr + ',' + c1);
    for (let cc = 0; cc < COLS; cc++) toRemove.add(r2 + ',' + cc);
    for (let rr = 0; rr < ROWS; rr++) toRemove.add(rr + ',' + c2);
    return toRemove;
  }

  // Wrapped + Wrapped = 5x5
  if (s1 === SPECIAL.WRAPPED && s2 === SPECIAL.WRAPPED) {
    const cr = Math.floor((r1 + r2) / 2), cc = Math.floor((c1 + c2) / 2);
    for (let dr = -2; dr <= 2; dr++)
      for (let dc = -2; dc <= 2; dc++) {
        const nr = cr + dr, nc = cc + dc;
        if (nr >= 0 && nr < ROWS && nc >= 0 && nc < COLS) toRemove.add(nr + ',' + nc);
      }
    return toRemove;
  }

  // Striped + Wrapped = 3 rows + 3 cols
  if ((isStriped(s1) && s2 === SPECIAL.WRAPPED) || (s1 === SPECIAL.WRAPPED && isStriped(s2))) {
    const cr = Math.floor((r1 + r2) / 2), cc = Math.floor((c1 + c2) / 2);
    for (let d = -1; d <= 1; d++) {
      for (let i = 0; i < COLS; i++) { const nr = cr + d; if (nr >= 0 && nr < ROWS) toRemove.add(nr + ',' + i); }
      for (let i = 0; i < ROWS; i++) { const nc = cc + d; if (nc >= 0 && nc < COLS) toRemove.add(i + ',' + nc); }
    }
    return toRemove;
  }

  return null; // No special combo, handle normally
}

function swap(r1, c1, r2, c2) {
  const tmp = state.board[r1][c1];
  state.board[r1][c1] = state.board[r2][c2];
  state.board[r2][c2] = tmp;
}

function gravity() {
  const newTiles = [];
  for (let c = 0; c < COLS; c++) {
    // Compact downward
    let writePos = ROWS - 1;
    for (let r = ROWS - 1; r >= 0; r--) {
      const cell = state.board[r][c];
      if (cell.blocker === BLOCKER.STONE || cell.blocker === BLOCKER.CHOCOLATE) {
        writePos = r - 1;
        continue;
      }
      if (cell.type >= 0) {
        if (r !== writePos) {
          state.board[writePos][c] = cell;
          state.board[r][c] = makeCell(EMPTY);
        }
        writePos--;
      }
    }
    // Fill empties from top
    for (let r = writePos; r >= 0; r--) {
      if (state.board[r][c].type === EMPTY && state.board[r][c].blocker === BLOCKER.NONE) {
        state.board[r][c] = makeCell(randomCandy());
        newTiles.push({ r, c });
      }
    }
  }
  return newTiles;
}

function spreadChocolate() {
  const toSpread = [];
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      if (state.board[r][c].blocker === BLOCKER.CHOCOLATE) {
        const dirs = [[0,1],[0,-1],[1,0],[-1,0]];
        for (const [dr, dc] of dirs) {
          const nr = r+dr, nc = c+dc;
          if (nr >= 0 && nr < ROWS && nc >= 0 && nc < COLS && state.board[nr][nc].type >= 0 && state.board[nr][nc].blocker === BLOCKER.NONE) {
            toSpread.push({r: nr, c: nc});
          }
        }
      }
    }
  }
  if (toSpread.length > 0) {
    const target = toSpread[Math.floor(Math.random() * toSpread.length)];
    state.board[target.r][target.c] = makeCell(EMPTY, SPECIAL.NONE, BLOCKER.CHOCOLATE);
  }
}

function clearAdjacentChocolate(matchedSet) {
  const toRemove = [];
  for (const key of matchedSet) {
    const [r, c] = key.split(',').map(Number);
    const dirs = [[0,1],[0,-1],[1,0],[-1,0]];
    for (const [dr, dc] of dirs) {
      const nr = r+dr, nc = c+dc;
      if (nr >= 0 && nr < ROWS && nc >= 0 && nc < COLS && state.board[nr][nc].blocker === BLOCKER.CHOCOLATE) {
        toRemove.push({r: nr, c: nc});
      }
    }
  }
  for (const {r, c} of toRemove) {
    state.board[r][c] = makeCell(EMPTY);
    const pos = cellToScreen(r, c);
    addParticle(pos.x, pos.y, '#5c3317', 6);
  }
}

function clearIceAt(r, c) {
  const cell = state.board[r][c];
  if (cell.blocker === BLOCKER.ICE2) {
    cell.blocker = BLOCKER.ICE1;
    return true;
  } else if (cell.blocker === BLOCKER.ICE1) {
    cell.blocker = BLOCKER.NONE;
    return true;
  }
  return false;
}

async function processMatches() {
  let cascade = 0;
  let clearedChocolate = false;

  while (true) {
    const groups = findMatchGroups();
    const allMatched = findAllMatchedCells();
    if (allMatched.length === 0) break;

    cascade++;
    state.cascade = cascade;

    // Detect and place specials
    const specials = detectSpecials(groups);

    // Score
    let pts = 0;
    for (const g of groups) {
      if (g.cells.length === 3) pts += 30;
      else if (g.cells.length === 4) pts += 60;
      else pts += 50 * g.cells.length;
    }
    pts += specials.length * 50;
    const multiplier = cascade > 1 ? cascade : 1;
    state.score += pts * multiplier;

    sfxMatch(cascade);
    if (cascade > 1) {
      state.shakeAmount = Math.min(cascade * 3, 15);
    }

    // Activate specials that are being matched
    const matchedSet = new Set(allMatched);
    const extraRemove = new Set();
    for (const key of matchedSet) {
      const [r, c] = key.split(',').map(Number);
      const cell = state.board[r][c];
      if (cell.special !== SPECIAL.NONE) {
        sfxExplosion();
        const extra = activateSpecial(r, c, state.board);
        for (const e of extra) extraRemove.add(e);
      }
    }
    for (const e of extraRemove) matchedSet.add(e);

    // Clear adjacent chocolate
    clearAdjacentChocolate(matchedSet);

    // Particles and floating text for matches
    for (const key of matchedSet) {
      const [r, c] = key.split(',').map(Number);
      const pos = cellToScreen(r, c);
      if (state.board[r][c].type >= 0) {
        addParticle(pos.x, pos.y, COLORS[state.board[r][c].type % CANDY_TYPES].main, 6);
      }
    }

    // Show score popup
    if (groups.length > 0) {
      const midGroup = groups[0].cells[Math.floor(groups[0].cells.length / 2)];
      const pos = cellToScreen(midGroup.r, midGroup.c);
      addFloatingText(pos.x, pos.y, '+' + (pts * multiplier), cascade > 1 ? '#ffb347' : '#fff');
    }

    if (cascade > 1) {
      addFloatingText(W/2, boardY - 30, cascade + 'x COMBO!', '#ff6bca');
    }

    // Remove matched cells (but keep specials that were just created)
    const specialPositions = new Set(specials.map(s => s.r + ',' + s.c));
    for (const key of matchedSet) {
      const [r, c] = key.split(',').map(Number);
      if (specialPositions.has(key)) continue;
      // Clear ice first
      clearIceAt(r, c);
      state.board[r][c] = makeCell(EMPTY);
    }

    // Place special candies
    for (const s of specials) {
      state.board[s.r][s.c] = makeCell(s.type, s.special);
      sfxSpecialCreate();
    }

    await sleep(250);

    // Gravity
    gravity();
    await sleep(200);
  }

  // Spread chocolate if no chocolate was cleared this turn
  if (!clearedChocolate) {
    spreadChocolate();
  }

  state.cascade = 0;
}

async function trySwap(r, c) {
  if (state.animating) return;
  state.animating = true;
  const sr = state.selected.r, sc = state.selected.c;
  state.selected = null;

  ensureAudio();

  // Check for special combo
  const comboResult = handleSpecialCombo(sr, sc, r, c);
  if (comboResult) {
    sfxSwap();
    swap(sr, sc, r, c);
    state.movesLeft--;

    // Remove all cells from combo
    sfxExplosion();
    state.shakeAmount = 12;
    for (const key of comboResult) {
      const [rr, cc] = key.split(',').map(Number);
      if (state.board[rr][cc].blocker === BLOCKER.STONE) continue;
      clearIceAt(rr, cc);
      const pos = cellToScreen(rr, cc);
      if (state.board[rr][cc].type >= 0) {
        addParticle(pos.x, pos.y, COLORS[state.board[rr][cc].type % CANDY_TYPES].main, 8);
      }
      state.board[rr][cc] = makeCell(EMPTY);
    }
    state.score += comboResult.size * 30;

    await sleep(300);
    gravity();
    await sleep(200);
    await processMatches();
    state.animating = false;
    checkLevelEnd();
    return;
  }

  // Normal swap
  swap(sr, sc, r, c);
  const matches = findAllMatchedCells();

  if (matches.length === 0) {
    sfxBadSwap();
    swap(sr, sc, r, c);
    await sleep(200);
    state.animating = false;
    return;
  }

  sfxSwap();
  state.movesLeft--;

  await processMatches();
  state.animating = false;

  // Reset hint timer
  state.hintTimer = 0;
  state.hintMove = null;

  checkLevelEnd();
}

function hasValidMoves() {
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      if (!isPlayable(r, c)) continue;
      if (c + 1 < COLS && isPlayable(r, c+1)) {
        swap(r, c, r, c+1);
        const has = findAllMatchedCells().length > 0;
        swap(r, c, r, c+1);
        if (has) return { r, c, r2: r, c2: c+1 };
      }
      if (r + 1 < ROWS && isPlayable(r+1, c)) {
        swap(r, c, r+1, c);
        const has = findAllMatchedCells().length > 0;
        swap(r, c, r+1, c);
        if (has) return { r, c, r2: r+1, c2: c };
      }
    }
  }
  return null;
}

function shuffleBoard() {
  const playable = [];
  for (let r = 0; r < ROWS; r++)
    for (let c = 0; c < COLS; c++)
      if (isPlayable(r, c)) playable.push({r, c});

  // Fisher-Yates shuffle of candy types
  for (let i = playable.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    const a = playable[i], b = playable[j];
    const tmp = state.board[a.r][a.c].type;
    state.board[a.r][a.c].type = state.board[b.r][b.c].type;
    state.board[b.r][b.c].type = tmp;
  }

  // Remove any initial matches
  let attempts = 0;
  while (findAllMatchedCells().length > 0 && attempts < 100) {
    for (let r = 0; r < ROWS; r++)
      for (let c = 0; c < COLS; c++)
        if (isPlayable(r, c)) state.board[r][c].type = randomCandy();
    attempts++;
  }
}

function checkLevelEnd() {
  if (state.movesLeft <= 0) {
    const ld = state.levelData;
    if (state.score >= ld.stars[0]) {
      // Win!
      const earnedStars = state.score >= ld.stars[2] ? 3 : state.score >= ld.stars[1] ? 2 : 1;
      state.stars = earnedStars;
      const prev = state.progress[state.level] || 0;
      if (earnedStars > prev) {
        state.progress[state.level] = earnedStars;
        saveProgress();
      }
      state.screen = SCREEN.LEVEL_COMPLETE;
      sfxWin();
    } else {
      state.screen = SCREEN.LEVEL_FAILED;
      sfxLose();
    }
    return;
  }

  // Auto-shuffle if no valid moves
  const validMove = hasValidMoves();
  if (!validMove) {
    shuffleBoard();
    addFloatingText(W/2, H/2, 'SHUFFLE!', '#ffb347');
  }
}

function sleep(ms) { return new Promise(r => setTimeout(r, ms)); }

// ═══════════════════════════════════════════════════════
// SECTION 7: LEVEL DEFINITIONS
// ═══════════════════════════════════════════════════════

function generateLevels() {
  const levels = [];

  for (let i = 1; i <= 30; i++) {
    const tier = i <= 10 ? 'easy' : i <= 20 ? 'medium' : 'hard';
    const baseMoves = tier === 'easy' ? 30 : tier === 'medium' ? 25 : 20;
    const movesVariance = Math.floor(i * 0.3);
    const totalMoves = baseMoves - movesVariance;

    const baseScore = 500 + i * 300;
    const stars = [
      baseScore,
      Math.floor(baseScore * 1.8),
      Math.floor(baseScore * 2.8),
    ];

    const blockers = Array.from({length: ROWS}, () => Array(COLS).fill(BLOCKER.NONE));

    // Add blockers based on tier and level
    if (i >= 8 && i < 14) {
      // Ice levels
      const iceCount = 3 + Math.floor((i - 8) * 1.5);
      placeRandomBlockers(blockers, BLOCKER.ICE1, iceCount);
    } else if (i >= 14 && i < 20) {
      // Chocolate levels
      const chocCount = 2 + Math.floor((i - 14) * 0.8);
      const iceCount = 2 + Math.floor((i - 14) * 0.5);
      placeRandomBlockers(blockers, BLOCKER.CHOCOLATE, chocCount);
      placeRandomBlockers(blockers, BLOCKER.ICE1, iceCount);
    } else if (i >= 20) {
      // Mixed blocker levels
      const stoneCount = 2 + Math.floor((i - 20) * 0.4);
      const chocCount = 2 + Math.floor((i - 20) * 0.5);
      const iceCount = 3 + Math.floor((i - 20) * 0.3);
      placeRandomBlockers(blockers, BLOCKER.STONE, stoneCount);
      placeRandomBlockers(blockers, BLOCKER.CHOCOLATE, chocCount);
      placeRandomBlockers(blockers, BLOCKER.ICE2, iceCount);
    }

    levels.push({ level: i, moves: Math.max(totalMoves, 10), stars, blockers });
  }

  return levels;
}

function placeRandomBlockers(blockers, type, count) {
  let placed = 0;
  let attempts = 0;
  while (placed < count && attempts < 200) {
    const r = 1 + Math.floor(Math.random() * (ROWS - 2));
    const c = 1 + Math.floor(Math.random() * (COLS - 2));
    if (blockers[r][c] === BLOCKER.NONE) {
      blockers[r][c] = type;
      placed++;
    }
    attempts++;
  }
}

const ALL_LEVELS = generateLevels();

// ═══════════════════════════════════════════════════════
// SECTION 8: INPUT HANDLING
// ═══════════════════════════════════════════════════════

let touchStartX = null, touchStartY = null;

function getClickedCell(px, py) {
  const c = Math.floor((px - boardX) / cellSize);
  const r = Math.floor((py - boardY) / cellSize);
  if (r >= 0 && r < ROWS && c >= 0 && c < COLS) return { r, c };
  return null;
}

function handleBoardClick(px, py) {
  if (state.animating || state.screen !== SCREEN.PLAYING) return;
  const cell = getClickedCell(px, py);
  if (!cell || !isPlayable(cell.r, cell.c)) return;

  if (!state.selected) {
    state.selected = cell;
    sfxClick();
    return;
  }

  if (state.selected.r === cell.r && state.selected.c === cell.c) {
    state.selected = null;
    return;
  }

  const dr = Math.abs(state.selected.r - cell.r);
  const dc = Math.abs(state.selected.c - cell.c);
  if ((dr === 1 && dc === 0) || (dr === 0 && dc === 1)) {
    trySwap(cell.r, cell.c);
  } else {
    state.selected = cell;
    sfxClick();
  }
}

function handleSwipe(startX, startY, endX, endY) {
  if (state.animating || state.screen !== SCREEN.PLAYING) return;
  const cell = getClickedCell(startX, startY);
  if (!cell || !isPlayable(cell.r, cell.c)) return;

  const dx = endX - startX;
  const dy = endY - startY;
  if (Math.abs(dx) < 10 && Math.abs(dy) < 10) {
    handleBoardClick(startX, startY);
    return;
  }

  let tr, tc;
  if (Math.abs(dx) > Math.abs(dy)) {
    tr = cell.r;
    tc = cell.c + (dx > 0 ? 1 : -1);
  } else {
    tr = cell.r + (dy > 0 ? 1 : -1);
    tc = cell.c;
  }

  if (tr >= 0 && tr < ROWS && tc >= 0 && tc < COLS && isPlayable(tr, tc)) {
    state.selected = cell;
    trySwap(tr, tc);
  }
}

// Sound toggle area (top right)
function isSoundToggle(px, py) {
  return px > W - 50 && py < 50;
}

// Level select click handling
function handleLevelSelectClick(px, py) {
  const gridCols = 5;
  const gridRows = 6;
  const btnSize = Math.min(cellSize * 1.2, 60);
  const gap = 12;
  const totalW = gridCols * (btnSize + gap) - gap;
  const startX = (W - totalW) / 2;
  const startY = H * 0.2;

  for (let i = 0; i < 30; i++) {
    const row = Math.floor(i / gridCols);
    const col = i % gridCols;
    const bx = startX + col * (btnSize + gap);
    const by = startY + row * (btnSize + gap + 8);

    if (px >= bx && px <= bx + btnSize && py >= by && py <= by + btnSize) {
      const levelNum = i + 1;
      const unlocked = levelNum === 1 || state.progress[levelNum - 1] >= 1;
      if (unlocked) {
        startLevel(levelNum);
        sfxClick();
      }
      return;
    }
  }
}

// Overlay button click handling
function handleOverlayClick(px, py) {
  const btnW = 160, btnH = 48;
  const btnX = (W - btnW) / 2;

  if (state.screen === SCREEN.LEVEL_COMPLETE) {
    // Next level button
    const btnY = H * 0.62;
    if (px >= btnX && px <= btnX + btnW && py >= btnY && py <= btnY + btnH) {
      if (state.level < 30) {
        startLevel(state.level + 1);
      } else {
        state.screen = SCREEN.LEVEL_SELECT;
      }
      sfxClick();
      return;
    }
    // Level select button
    const btn2Y = H * 0.72;
    if (px >= btnX && px <= btnX + btnW && py >= btn2Y && py <= btn2Y + btnH) {
      state.screen = SCREEN.LEVEL_SELECT;
      sfxClick();
      return;
    }
  } else if (state.screen === SCREEN.LEVEL_FAILED) {
    // Retry button
    const btnY = H * 0.58;
    if (px >= btnX && px <= btnX + btnW && py >= btnY && py <= btnY + btnH) {
      startLevel(state.level);
      sfxClick();
      return;
    }
    // Level select button
    const btn2Y = H * 0.68;
    if (px >= btnX && px <= btnX + btnW && py >= btn2Y && py <= btn2Y + btnH) {
      state.screen = SCREEN.LEVEL_SELECT;
      sfxClick();
      return;
    }
  }
}

function onPointerDown(e) {
  const px = e.clientX, py = e.clientY;
  ensureAudio();

  if (isSoundToggle(px, py)) {
    soundEnabled = !soundEnabled;
    sfxClick();
    return;
  }

  if (state.screen === SCREEN.LEVEL_SELECT) {
    handleLevelSelectClick(px, py);
    return;
  }

  if (state.screen === SCREEN.LEVEL_COMPLETE || state.screen === SCREEN.LEVEL_FAILED) {
    handleOverlayClick(px, py);
    return;
  }

  touchStartX = px;
  touchStartY = py;
}

function onPointerUp(e) {
  if (touchStartX === null) return;
  const px = e.clientX, py = e.clientY;

  if (state.screen === SCREEN.PLAYING) {
    handleSwipe(touchStartX, touchStartY, px, py);
  }

  touchStartX = null;
  touchStartY = null;
}

canvas.addEventListener('pointerdown', onPointerDown);
canvas.addEventListener('pointerup', onPointerUp);

// ═══════════════════════════════════════════════════════
// SECTION 9: UI SCREENS
// ═══════════════════════════════════════════════════════

function drawLevelSelect() {
  // Title
  ctx.fillStyle = '#fff';
  ctx.font = `bold ${Math.floor(cellSize * 0.65)}px 'Segoe UI', system-ui, sans-serif`;
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';

  // Gradient text effect
  const titleGrad = ctx.createLinearGradient(W/2 - 100, 0, W/2 + 100, 0);
  titleGrad.addColorStop(0, '#ff6bca');
  titleGrad.addColorStop(0.5, '#ffb347');
  titleGrad.addColorStop(1, '#ff6bca');
  ctx.fillStyle = titleGrad;
  ctx.fillText('Candy Crush', W / 2, H * 0.08);

  ctx.font = `${Math.floor(cellSize * 0.35)}px 'Segoe UI', system-ui, sans-serif`;
  ctx.fillStyle = '#ccc';
  ctx.fillText('Select a Level', W / 2, H * 0.14);

  // Level grid
  const gridCols = 5;
  const btnSize = Math.min(cellSize * 1.2, 60);
  const gap = 12;
  const totalW = gridCols * (btnSize + gap) - gap;
  const startX = (W - totalW) / 2;
  const startY = H * 0.2;

  for (let i = 0; i < 30; i++) {
    const row = Math.floor(i / gridCols);
    const col = i % gridCols;
    const bx = startX + col * (btnSize + gap);
    const by = startY + row * (btnSize + gap + 8);
    const levelNum = i + 1;
    const starsEarned = state.progress[levelNum] || 0;
    const unlocked = levelNum === 1 || state.progress[levelNum - 1] >= 1;

    // Button background
    if (unlocked) {
      const tier = levelNum <= 10 ? 0 : levelNum <= 20 ? 1 : 2;
      const tierColors = [
        ['#2ed573', '#009432'],
        ['#ffa502', '#e67e22'],
        ['#ff4757', '#c44569'],
      ];
      const btnGrad = ctx.createLinearGradient(bx, by, bx + btnSize, by + btnSize);
      btnGrad.addColorStop(0, tierColors[tier][0]);
      btnGrad.addColorStop(1, tierColors[tier][1]);
      ctx.fillStyle = btnGrad;
    } else {
      ctx.fillStyle = '#444';
    }
    drawRoundedRect(bx, by, btnSize, btnSize, 10);
    ctx.fill();

    // Level number
    ctx.fillStyle = unlocked ? '#fff' : '#888';
    ctx.font = `bold ${Math.floor(btnSize * 0.4)}px 'Segoe UI', system-ui, sans-serif`;
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(unlocked ? levelNum : '🔒', bx + btnSize/2, by + btnSize/2);

    // Stars
    if (starsEarned > 0) {
      ctx.font = `${Math.floor(btnSize * 0.25)}px sans-serif`;
      const starStr = '★'.repeat(starsEarned) + '☆'.repeat(3 - starsEarned);
      ctx.fillStyle = '#ffa502';
      ctx.fillText(starStr, bx + btnSize/2, by + btnSize + 6);
    }
  }
}

function drawLevelComplete() {
  // Dim background
  ctx.fillStyle = 'rgba(0,0,0,0.7)';
  ctx.fillRect(0, 0, W, H);

  // Box
  const boxW = Math.min(W * 0.8, 350);
  const boxH = 280;
  const boxX = (W - boxW) / 2;
  const boxY = H * 0.25;

  const grad = ctx.createLinearGradient(boxX, boxY, boxX + boxW, boxY + boxH);
  grad.addColorStop(0, '#2d1b69');
  grad.addColorStop(1, '#4a1a6b');
  ctx.fillStyle = grad;
  drawRoundedRect(boxX, boxY, boxW, boxH, 20);
  ctx.fill();
  ctx.strokeStyle = 'rgba(255,255,255,0.2)';
  ctx.lineWidth = 2;
  drawRoundedRect(boxX, boxY, boxW, boxH, 20);
  ctx.stroke();

  // Title
  ctx.fillStyle = '#2ed573';
  ctx.font = `bold ${Math.floor(cellSize * 0.55)}px 'Segoe UI', system-ui, sans-serif`;
  ctx.textAlign = 'center';
  ctx.fillText('Level Complete!', W/2, boxY + 45);

  // Score
  ctx.fillStyle = '#fff';
  ctx.font = `bold ${Math.floor(cellSize * 0.5)}px 'Segoe UI', system-ui, sans-serif`;
  ctx.fillText(state.score.toLocaleString(), W/2, boxY + 90);

  // Stars
  ctx.font = `${Math.floor(cellSize * 0.7)}px sans-serif`;
  const starStr = '★'.repeat(state.stars) + '☆'.repeat(3 - state.stars);
  ctx.fillStyle = '#ffa502';
  ctx.fillText(starStr, W/2, boxY + 140);

  // Next Level button
  const btnW = 160, btnH = 48;
  const btnX = (W - btnW) / 2;
  const btnY = H * 0.62;

  const btnGrad = ctx.createLinearGradient(btnX, btnY, btnX + btnW, btnY + btnH);
  btnGrad.addColorStop(0, '#ff6bca');
  btnGrad.addColorStop(1, '#ff9a56');
  ctx.fillStyle = btnGrad;
  drawRoundedRect(btnX, btnY, btnW, btnH, 24);
  ctx.fill();
  ctx.fillStyle = '#fff';
  ctx.font = `bold ${Math.floor(cellSize * 0.35)}px 'Segoe UI', system-ui, sans-serif`;
  ctx.fillText(state.level < 30 ? 'Next Level' : 'Done!', W/2, btnY + btnH/2);

  // Level Select button
  const btn2Y = H * 0.72;
  ctx.fillStyle = 'rgba(255,255,255,0.15)';
  drawRoundedRect(btnX, btn2Y, btnW, btnH, 24);
  ctx.fill();
  ctx.strokeStyle = 'rgba(255,255,255,0.3)';
  ctx.lineWidth = 1;
  drawRoundedRect(btnX, btn2Y, btnW, btnH, 24);
  ctx.stroke();
  ctx.fillStyle = '#fff';
  ctx.fillText('Level Select', W/2, btn2Y + btnH/2);
}

function drawLevelFailed() {
  ctx.fillStyle = 'rgba(0,0,0,0.7)';
  ctx.fillRect(0, 0, W, H);

  const boxW = Math.min(W * 0.8, 350);
  const boxH = 240;
  const boxX = (W - boxW) / 2;
  const boxY = H * 0.28;

  const grad = ctx.createLinearGradient(boxX, boxY, boxX + boxW, boxY + boxH);
  grad.addColorStop(0, '#2d1b69');
  grad.addColorStop(1, '#4a1a6b');
  ctx.fillStyle = grad;
  drawRoundedRect(boxX, boxY, boxW, boxH, 20);
  ctx.fill();
  ctx.strokeStyle = 'rgba(255,255,255,0.2)';
  ctx.lineWidth = 2;
  drawRoundedRect(boxX, boxY, boxW, boxH, 20);
  ctx.stroke();

  ctx.fillStyle = '#ff4757';
  ctx.font = `bold ${Math.floor(cellSize * 0.55)}px 'Segoe UI', system-ui, sans-serif`;
  ctx.textAlign = 'center';
  ctx.fillText('Out of Moves!', W/2, boxY + 45);

  ctx.fillStyle = '#ccc';
  ctx.font = `${Math.floor(cellSize * 0.35)}px 'Segoe UI', system-ui, sans-serif`;
  ctx.fillText(`Score: ${state.score.toLocaleString()}`, W/2, boxY + 85);
  ctx.fillText(`Needed: ${state.levelData.stars[0].toLocaleString()}`, W/2, boxY + 115);

  // Retry button
  const btnW = 160, btnH = 48;
  const btnX = (W - btnW) / 2;
  const btnY = H * 0.58;

  const btnGrad = ctx.createLinearGradient(btnX, btnY, btnX + btnW, btnY + btnH);
  btnGrad.addColorStop(0, '#ff6bca');
  btnGrad.addColorStop(1, '#ff9a56');
  ctx.fillStyle = btnGrad;
  drawRoundedRect(btnX, btnY, btnW, btnH, 24);
  ctx.fill();
  ctx.fillStyle = '#fff';
  ctx.font = `bold ${Math.floor(cellSize * 0.35)}px 'Segoe UI', system-ui, sans-serif`;
  ctx.fillText('Try Again', W/2, btnY + btnH/2);

  // Level Select button
  const btn2Y = H * 0.68;
  ctx.fillStyle = 'rgba(255,255,255,0.15)';
  drawRoundedRect(btnX, btn2Y, btnW, btnH, 24);
  ctx.fill();
  ctx.strokeStyle = 'rgba(255,255,255,0.3)';
  ctx.lineWidth = 1;
  drawRoundedRect(btnX, btn2Y, btnW, btnH, 24);
  ctx.stroke();
  ctx.fillStyle = '#fff';
  ctx.fillText('Level Select', W/2, btn2Y + btnH/2);
}

// ═══════════════════════════════════════════════════════
// SECTION 10: GAME LOOP & INITIALIZATION
// ═══════════════════════════════════════════════════════

function startLevel(levelNum) {
  const ld = ALL_LEVELS[levelNum - 1];
  state.level = levelNum;
  state.levelData = ld;
  state.score = 0;
  state.movesLeft = ld.moves;
  state.totalMoves = ld.moves;
  state.selected = null;
  state.animating = false;
  state.cascade = 0;
  state.hintTimer = 0;
  state.hintMove = null;
  state.particles = [];
  state.floatingTexts = [];
  state.animations = [];
  state.shakeAmount = 0;
  state.stars = 0;
  state.screen = SCREEN.PLAYING;

  createBoard(ld);

  // Ensure no initial matches
  let attempts = 0;
  while (findAllMatchedCells().length > 0 || !hasValidMoves()) {
    createBoard(ld);
    if (++attempts > 100) break;
  }
}

let lastTime = 0;

function gameLoop(time) {
  const dt = Math.min((time - lastTime) / 1000, 0.05);
  lastTime = time;

  // Update
  updateParticles(dt);
  updateAnimations(dt);

  // Hint timer
  if (state.screen === SCREEN.PLAYING && !state.animating && !state.selected) {
    state.hintTimer += dt;
    if (state.hintTimer > 5 && !state.hintMove) {
      state.hintMove = hasValidMoves();
    }
  }

  // Draw
  drawBackground();

  if (state.screen === SCREEN.LEVEL_SELECT) {
    drawLevelSelect();
  } else if (state.screen === SCREEN.PLAYING) {
    drawBoard();
    drawHUD();
    drawParticles();
    drawFloatingTexts();
  } else if (state.screen === SCREEN.LEVEL_COMPLETE) {
    drawBoard();
    drawHUD();
    drawParticles();
    drawFloatingTexts();
    drawLevelComplete();
  } else if (state.screen === SCREEN.LEVEL_FAILED) {
    drawBoard();
    drawHUD();
    drawLevelFailed();
  }

  // Sound toggle icon (always visible)
  ctx.font = `${Math.floor(cellSize * 0.4)}px sans-serif`;
  ctx.textAlign = 'right';
  ctx.fillStyle = '#fff';
  ctx.fillText(soundEnabled ? '🔊' : '🔇', W - 16, 28);

  requestAnimationFrame(gameLoop);
}

// Start
loadProgress();
requestAnimationFrame(gameLoop);

</script>
</body>
</html>
```

- [ ] **Step 2: Open in browser and verify**

Open `candy-crush/index.html` in a browser. You should see:
- Level select screen with 30 levels in a 5x6 grid
- Level 1 is clickable (green), rest are locked (grey)
- Clicking Level 1 shows the game board with colored candy shapes
- HUD shows level number, score, moves remaining, and star progress bar
- Sound toggle icon in top right
- Clicking/swiping candies triggers swap attempts with sound
- Matches clear with particles, cascades, and floating score text
- Special candies (striped, wrapped, color bomb) appear from 4+ matches
- Ice, chocolate, and stone blockers appear in higher levels
- Game over and level complete overlays with retry/next/level-select buttons
- Progress saved to localStorage

- [ ] **Step 3: Commit**

```bash
git add candy-crush/index.html
git commit -m "feat: complete Candy Crush rewrite with canvas rendering, specials, levels, audio, and polish"
```

---

### Task 2: Gameplay Polish & Bug Fixes

**Files:**
- Modify: `candy-crush/index.html`

Play through levels 1-5 and fix any issues found. Common things to verify and fix:

- [ ] **Step 1: Test and fix board initialization**

Verify no initial matches appear. Verify the board always has valid moves. If `createBoard` sometimes fails, increase the retry limit or improve the algorithm.

- [ ] **Step 2: Test special candy creation**

Make 4-in-a-row matches and verify striped candies appear. Make L/T shapes and verify wrapped candies appear. Make 5-in-a-row and verify color bombs appear. Fix detection logic if any don't trigger correctly.

- [ ] **Step 3: Test special candy activation and combos**

Verify striped clears a full row/column. Verify wrapped explodes 3x3. Verify color bomb clears all of target color. Test special+special combos. Fix activation logic for any that don't work correctly.

- [ ] **Step 4: Test level progression**

Complete level 1 and verify level 2 unlocks. Verify stars are saved to localStorage. Verify the level select shows earned stars. Retry a failed level and verify it resets properly.

- [ ] **Step 5: Test edge cases**

- Swapping into/near stone blockers
- Chocolate spreading behavior
- Ice layer removal
- Shuffle when no moves available
- Hint appearing after 5s idle

- [ ] **Step 6: Commit fixes**

```bash
git add candy-crush/index.html
git commit -m "fix: gameplay polish and bug fixes from playtesting"
```

---

### Task 3: End-of-Level Bonus & Final Polish

**Files:**
- Modify: `candy-crush/index.html`

- [ ] **Step 1: Add end-of-level bonus move conversion**

When a level is won, remaining moves should each spawn a random special candy that detonates for bonus points. Add this to `checkLevelEnd` before showing the complete screen:

```javascript
// Inside checkLevelEnd, after determining the player won:
async function endOfLevelBonus() {
  for (let i = 0; i < state.movesLeft; i++) {
    // Find a random playable cell
    const playable = [];
    for (let r = 0; r < ROWS; r++)
      for (let c = 0; c < COLS; c++)
        if (isPlayable(r, c)) playable.push({r, c});
    if (playable.length === 0) break;

    const target = playable[Math.floor(Math.random() * playable.length)];
    const specType = [SPECIAL.STRIPED_H, SPECIAL.STRIPED_V, SPECIAL.WRAPPED][Math.floor(Math.random() * 3)];
    state.board[target.r][target.c].special = specType;

    // Activate it
    const removed = activateSpecial(target.r, target.c, state.board);
    for (const key of removed) {
      const [rr, cc] = key.split(',').map(Number);
      if (state.board[rr][cc].type >= 0) {
        const pos = cellToScreen(rr, cc);
        addParticle(pos.x, pos.y, COLORS[state.board[rr][cc].type % CANDY_TYPES].main, 4);
        state.board[rr][cc] = makeCell(EMPTY);
      }
    }
    state.score += 50;
    sfxExplosion();
    await sleep(150);
    gravity();
    await sleep(100);
  }
}
```

- [ ] **Step 2: Verify end-of-level bonus works**

Win a level with remaining moves. Verify specials appear and detonate one by one. Verify score increases. Verify the complete screen shows the final score.

- [ ] **Step 3: Final commit**

```bash
git add candy-crush/index.html
git commit -m "feat: add end-of-level bonus and final polish"
```
