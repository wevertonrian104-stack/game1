<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Floresta Protegida 🌲</title>
<style>
  @import url('https://fonts.googleapis.com/css2?family=Fredoka+One&family=Nunito:wght@400;700;900&display=swap');

  :root {
    --sky1: #87CEEB;
    --sky2: #b8e4f5;
    --ground: #5a3e1b;
    --grass: #2d7a1f;
    --grass2: #3a9e28;
    --trunk: #6b3a1f;
    --leaf1: #1a6b0a;
    --leaf2: #2d9e1a;
    --leaf3: #3abf24;
    --paddle: #f5c842;
    --paddle2: #e0a800;
    --saw: #c0c0c0;
    --saw-blade: #888;
    --score: #fff;
  }

  * { margin: 0; padding: 0; box-sizing: border-box; }

  body {
    background: #1a1a2e;
    display: flex;
    align-items: center;
    justify-content: center;
    height: 100vh;
    font-family: 'Fredoka One', cursive;
    overflow: hidden;
  }

  #wrapper {
    position: relative;
    width: 600px;
    height: 700px;
  }

  #gameCanvas {
    border-radius: 18px;
    box-shadow: 0 0 60px #00ff9944, 0 0 120px #00804455;
    display: block;
    cursor: none;
  }

  #ui {
    position: absolute;
    top: 14px;
    left: 0; right: 0;
    display: flex;
    justify-content: space-between;
    padding: 0 20px;
    pointer-events: none;
    z-index: 10;
  }

  .badge {
    background: rgba(0,0,0,0.45);
    backdrop-filter: blur(6px);
    border-radius: 30px;
    padding: 7px 20px;
    color: #fff;
    font-size: 22px;
    letter-spacing: 1px;
    border: 1.5px solid rgba(255,255,255,0.15);
    text-shadow: 0 1px 4px #000a;
  }

  #overlay {
    position: absolute;
    inset: 0;
    border-radius: 18px;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    background: rgba(0,0,0,0.72);
    backdrop-filter: blur(6px);
    z-index: 20;
    gap: 18px;
    transition: opacity 0.4s;
  }

  #overlay.hidden { display: none; }

  #overlay h1 {
    font-size: 58px;
    color: #2ecc71;
    text-shadow: 0 0 30px #2ecc71, 0 3px 8px #000;
    letter-spacing: 2px;
    animation: pulse 1.8s infinite alternate;
  }

  #overlay.gameover h1 {
    color: #e74c3c;
    text-shadow: 0 0 30px #e74c3c, 0 3px 8px #000;
  }

  @keyframes pulse {
    from { transform: scale(1); } to { transform: scale(1.06); }
  }

  #overlay p {
    font-family: 'Nunito', sans-serif;
    font-size: 19px;
    color: #cde;
    text-align: center;
    max-width: 380px;
    line-height: 1.5;
  }

  #overlay button {
    margin-top: 8px;
    background: linear-gradient(135deg, #2ecc71, #27ae60);
    color: #fff;
    border: none;
    border-radius: 50px;
    padding: 14px 48px;
    font-family: 'Fredoka One', cursive;
    font-size: 26px;
    cursor: pointer;
    box-shadow: 0 6px 24px #2ecc7166;
    transition: transform 0.15s, box-shadow 0.15s;
    letter-spacing: 1px;
  }

  #overlay button:hover {
    transform: scale(1.07);
    box-shadow: 0 10px 32px #2ecc7199;
  }

  #cutscene {
    position: absolute;
    inset: 0;
    border-radius: 18px;
    z-index: 25;
    display: none;
    overflow: hidden;
    background: #0a1a0a;
    flex-direction: column;
    align-items: center;
    justify-content: center;
  }

  #cutscene.show { display: flex; }

  #cutscene canvas {
    border-radius: 14px;
  }

  #cutscene .msg {
    position: absolute;
    bottom: 40px;
    font-size: 22px;
    color: #e74c3c;
    text-shadow: 0 0 18px #e74c3c;
    animation: blink 1s infinite;
  }

  @keyframes blink {
    0%,100%{opacity:1} 50%{opacity:0.3}
  }

  #lives {
    display: flex;
    gap: 5px;
  }
</style>
</head>
<body>
<div id="wrapper">
  <canvas id="gameCanvas" width="600" height="700"></canvas>
  <div id="ui">
    <div class="badge">🌲 <span id="scoreDisplay">0</span></div>
    <div class="badge" id="livesDisplay">❤️ ❤️ ❤️</div>
  </div>

  <div id="overlay">
    <h1>🌲 FLORESTA<br>PROTEGIDA</h1>
    <p>Mova o <b>protetor</b> para bloquear as motosserras!<br>Não deixe elas chegarem às árvores!</p>
    <button id="startBtn">▶ JOGAR</button>
  </div>

  <div id="cutscene">
    <canvas id="cutCanvas" width="600" height="700"></canvas>
    <div class="msg">🪚 AS ÁRVORES FORAM CORTADAS...</div>
  </div>
</div>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const W = canvas.width, H = canvas.height;

const cutscene = document.getElementById('cutscene');
const cutCanvas = document.getElementById('cutCanvas');
const cutCtx = cutCanvas.getContext('2d');

const overlay = document.getElementById('overlay');
const startBtn = document.getElementById('startBtn');
const scoreDisplay = document.getElementById('scoreDisplay');
const livesDisplay = document.getElementById('livesDisplay');

// Game state
let gameRunning = false;
let score = 0;
let lives = 3;
let paddleX = W / 2;
let chainsaws = [];
let particles = [];
let clouds = [];
let birds = [];
let frameCount = 0;
let difficulty = 1;
let hiScore = 0;
let bgOffset = 0;

// Trees definition (positions)
const TREES = [
  { x: 80,  h: 110, w: 44 },
  { x: 175, h: 130, w: 50 },
  { x: 280, h: 95,  w: 40 },
  { x: 380, h: 120, w: 48 },
  { x: 475, h: 105, w: 42 },
  { x: 545, h: 85,  w: 36 },
];

const GROUND_Y = 560;
const PADDLE_Y = GROUND_Y - 28;
const PADDLE_W = 110;
const PADDLE_H = 14;
const TREE_TOP_Y = GROUND_Y - 30;

// Mouse / touch
canvas.addEventListener('mousemove', e => {
  const rect = canvas.getBoundingClientRect();
  paddleX = e.clientX - rect.left;
});
canvas.addEventListener('touchmove', e => {
  e.preventDefault();
  const rect = canvas.getBoundingClientRect();
  paddleX = e.touches[0].clientX - rect.left;
}, { passive: false });

startBtn.addEventListener('click', startGame);

function startGame() {
  overlay.classList.add('hidden');
  score = 0;
  lives = 3;
  chainsaws = [];
  particles = [];
  frameCount = 0;
  difficulty = 1;
  updateUI();
  gameRunning = true;
  requestAnimationFrame(gameLoop);
}

function updateUI() {
  scoreDisplay.textContent = score;
  let h = '';
  for (let i = 0; i < 3; i++) h += i < lives ? '❤️ ' : '🖤 ';
  livesDisplay.textContent = h.trim();
}

// Chainsaw object
function spawnChainsaw() {
  const x = 40 + Math.random() * (W - 80);
  const speed = (2.2 + difficulty * 0.35) * (0.8 + Math.random() * 0.5);
  const angle = (Math.random() - 0.5) * 0.4;
  chainsaws.push({ x, y: -60, vy: speed, vx: angle, angle: 0, spin: (Math.random() - 0.5) * 0.18, alive: true });
}

function spawnCloud() {
  clouds.push({ x: W + 80, y: 40 + Math.random() * 120, w: 80 + Math.random() * 80, speed: 0.3 + Math.random() * 0.3 });
}

function spawnBird() {
  birds.push({ x: -30, y: 60 + Math.random() * 100, speed: 1.2 + Math.random() * 1.2, flap: 0 });
}

// Init clouds
for (let i = 0; i < 5; i++) clouds.push({ x: Math.random() * W, y: 30 + Math.random() * 120, w: 80 + Math.random() * 90, speed: 0.3 + Math.random() * 0.3 });

// DRAW FUNCTIONS
function drawSky() {
  const grad = ctx.createLinearGradient(0, 0, 0, GROUND_Y);
  grad.addColorStop(0, '#3a7bd5');
  grad.addColorStop(0.5, '#74b9ff');
  grad.addColorStop(1, '#b8e4f5');
  ctx.fillStyle = grad;
  ctx.fillRect(0, 0, W, GROUND_Y);
}

function drawSun() {
  // Sun glow
  const sx = 510, sy = 70;
  const glow = ctx.createRadialGradient(sx, sy, 10, sx, sy, 70);
  glow.addColorStop(0, 'rgba(255,240,80,0.9)');
  glow.addColorStop(0.4, 'rgba(255,200,50,0.4)');
  glow.addColorStop(1, 'rgba(255,160,0,0)');
  ctx.fillStyle = glow;
  ctx.beginPath(); ctx.arc(sx, sy, 70, 0, Math.PI*2); ctx.fill();
  ctx.fillStyle = '#ffe066';
  ctx.beginPath(); ctx.arc(sx, sy, 28, 0, Math.PI*2); ctx.fill();
}

function drawCloud(c) {
  ctx.save();
  ctx.globalAlpha = 0.88;
  ctx.fillStyle = '#fff';
  const r = c.w * 0.28;
  ctx.beginPath();
  ctx.ellipse(c.x, c.y, c.w * 0.5, r * 0.7, 0, 0, Math.PI*2); ctx.fill();
  ctx.beginPath();
  ctx.ellipse(c.x - c.w*0.2, c.y + 6, r*0.7, r*0.6, 0, 0, Math.PI*2); ctx.fill();
  ctx.beginPath();
  ctx.ellipse(c.x + c.w*0.2, c.y + 4, r*0.65, r*0.55, 0, 0, Math.PI*2); ctx.fill();
  ctx.restore();
}

function drawBird(b) {
  ctx.save();
  ctx.strokeStyle = '#1a1a2e';
  ctx.lineWidth = 2;
  ctx.beginPath();
  const wing = Math.sin(b.flap) * 8;
  ctx.moveTo(b.x - 10, b.y);
  ctx.quadraticCurveTo(b.x - 5, b.y - wing, b.x, b.y);
  ctx.quadraticCurveTo(b.x + 5, b.y - wing, b.x + 10, b.y);
  ctx.stroke();
  ctx.restore();
}

function drawGround() {
  // Dirt
  const dirt = ctx.createLinearGradient(0, GROUND_Y, 0, H);
  dirt.addColorStop(0, '#5a3e1b');
  dirt.addColorStop(1, '#3a2508');
  ctx.fillStyle = dirt;
  ctx.fillRect(0, GROUND_Y, W, H - GROUND_Y);

  // Grass strip
  const grass = ctx.createLinearGradient(0, GROUND_Y - 14, 0, GROUND_Y + 10);
  grass.addColorStop(0, '#3abf24');
  grass.addColorStop(1, '#2d7a1f');
  ctx.fillStyle = grass;
  ctx.fillRect(0, GROUND_Y - 14, W, 24);

  // Grass tufts
  ctx.fillStyle = '#4cd62e';
  for (let i = 0; i < W; i += 18) {
    const h2 = 5 + Math.sin(i * 0.3) * 3;
    ctx.beginPath();
    ctx.moveTo(i, GROUND_Y - 10);
    ctx.lineTo(i + 6, GROUND_Y - 10 - h2);
    ctx.lineTo(i + 12, GROUND_Y - 10);
    ctx.fill();
  }
}

function drawTree(t, ctx2 = ctx, cut = 0) {
  const bx = t.x, by = TREE_TOP_Y;
  const trunkH = 55 + t.h * 0.3;
  const trunkW = 12;

  // Shadow
  ctx2.save();
  ctx2.globalAlpha = 0.18;
  ctx2.fillStyle = '#000';
  ctx2.beginPath();
  ctx2.ellipse(bx + 10, by + trunkH + 4, t.w * 0.7, 8, 0, 0, Math.PI*2);
  ctx2.fill();
  ctx2.restore();

  // Trunk
  const trunkGrad = ctx2.createLinearGradient(bx - trunkW/2, 0, bx + trunkW/2, 0);
  trunkGrad.addColorStop(0, '#7a4a22');
  trunkGrad.addColorStop(0.5, '#a0622a');
  trunkGrad.addColorStop(1, '#5a3010');
  ctx2.fillStyle = trunkGrad;

  if (cut > 0) {
    // Cut trunk - falls
    ctx2.save();
    ctx2.translate(bx, by + trunkH);
    ctx2.rotate(cut);
    ctx2.fillRect(-trunkW/2, -trunkH, trunkW, trunkH * (1 - cut));
    ctx2.restore();
    // Stump
    ctx2.fillStyle = '#a0622a';
    ctx2.fillRect(bx - trunkW/2, by + trunkH * 0.7, trunkW, trunkH * 0.3);
    // Stump rings
    ctx2.strokeStyle = '#7a4a22';
    ctx2.lineWidth = 1;
    ctx2.beginPath();
    ctx2.arc(bx, by + trunkH, 5, 0, Math.PI*2); ctx2.stroke();
    return;
  }

  ctx2.fillRect(bx - trunkW/2, by + 20, trunkW, trunkH);

  // Layered foliage (3 triangles)
  const leafColors = ['#1a6b0a', '#2d9e1a', '#3abf24'];
  for (let layer = 0; layer < 3; layer++) {
    const ly = by - t.h * 0.5 + layer * (t.h * 0.28);
    const lw = t.w * (1 - layer * 0.18);
    const lh = t.h * 0.55;
    const grad = ctx2.createLinearGradient(bx - lw, ly, bx + lw, ly + lh);
    grad.addColorStop(0, leafColors[layer]);
    grad.addColorStop(0.5, layer === 2 ? '#5eea3a' : '#3abf24');
    grad.addColorStop(1, '#1a6b0a');
    ctx2.fillStyle = grad;
    ctx2.beginPath();
    ctx2.moveTo(bx, ly);
    ctx2.lineTo(bx + lw, ly + lh);
    ctx2.lineTo(bx - lw, ly + lh);
    ctx2.closePath();
    ctx2.fill();

    // Highlight
    ctx2.fillStyle = 'rgba(255,255,255,0.07)';
    ctx2.beginPath();
    ctx2.moveTo(bx, ly);
    ctx2.lineTo(bx + lw * 0.3, ly + lh * 0.4);
    ctx2.lineTo(bx - lw * 0.1, ly + lh * 0.5);
    ctx2.closePath();
    ctx2.fill();
  }
}

function drawChainsaw(cs) {
  ctx.save();
  ctx.translate(cs.x, cs.y);
  ctx.rotate(cs.angle);

  // Body
  const bg = ctx.createLinearGradient(-20, -12, 20, 12);
  bg.addColorStop(0, '#e0e0e0');
  bg.addColorStop(0.5, '#a0a0a0');
  bg.addColorStop(1, '#606060');
  ctx.fillStyle = bg;
  ctx.beginPath();
  ctx.roundRect(-18, -10, 36, 20, 6);
  ctx.fill();
  ctx.strokeStyle = '#555';
  ctx.lineWidth = 1.5;
  ctx.stroke();

  // Engine detail
  ctx.fillStyle = '#ff5533';
  ctx.fillRect(-8, -7, 16, 5);
  ctx.fillStyle = '#222';
  ctx.fillRect(-4, -7, 3, 5);
  ctx.fillRect(2, -7, 3, 5);

  // Blade bar
  ctx.fillStyle = '#888';
  ctx.fillRect(18, -4, 26, 8);
  ctx.strokeStyle = '#555';
  ctx.lineWidth = 1;
  ctx.stroke();

  // Blade teeth (spinning)
  ctx.fillStyle = '#c8c8c8';
  for (let i = 0; i < 8; i++) {
    const tx = 20 + i * 3.2;
    const ty = (Math.floor((cs.angle * 5 + i) % 2) === 0) ? -6 : 5;
    ctx.fillRect(tx, ty, 2.5, 3);
  }

  // Handle
  ctx.fillStyle = '#333';
  ctx.beginPath();
  ctx.roundRect(-20, -14, 10, 6, 3);
  ctx.fill();

  // Motion blur lines
  ctx.strokeStyle = 'rgba(200,200,200,0.3)';
  ctx.lineWidth = 2;
  for (let i = 1; i <= 4; i++) {
    ctx.beginPath();
    ctx.moveTo(-22, -i * 5);
    ctx.lineTo(22, -i * 5);
    ctx.stroke();
  }

  ctx.restore();
}

function drawPaddle() {
  const px = Math.max(PADDLE_W/2 + 5, Math.min(W - PADDLE_W/2 - 5, paddleX));
  const py = PADDLE_Y;

  // Glow
  ctx.save();
  ctx.shadowColor = '#ffe066';
  ctx.shadowBlur = 18;

  // Wood planks effect
  const grad = ctx.createLinearGradient(px - PADDLE_W/2, py - PADDLE_H/2, px - PADDLE_W/2, py + PADDLE_H/2);
  grad.addColorStop(0, '#f5c842');
  grad.addColorStop(0.5, '#e6b800');
  grad.addColorStop(1, '#c49200');
  ctx.fillStyle = grad;
  ctx.beginPath();
  ctx.roundRect(px - PADDLE_W/2, py - PADDLE_H/2, PADDLE_W, PADDLE_H, 8);
  ctx.fill();

  // Shine
  ctx.fillStyle = 'rgba(255,255,255,0.35)';
  ctx.beginPath();
  ctx.roundRect(px - PADDLE_W/2 + 4, py - PADDLE_H/2 + 2, PADDLE_W - 8, 4, 4);
  ctx.fill();

  // Border
  ctx.strokeStyle = '#a07800';
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.roundRect(px - PADDLE_W/2, py - PADDLE_H/2, PADDLE_W, PADDLE_H, 8);
  ctx.stroke();

  // Metal bolts
  ctx.fillStyle = '#888';
  [px - PADDLE_W/2 + 10, px + PADDLE_W/2 - 10].forEach(bx => {
    ctx.beginPath();
    ctx.arc(bx, py, 3.5, 0, Math.PI*2);
    ctx.fill();
  });

  ctx.restore();
}

function drawParticles() {
  particles = particles.filter(p => p.life > 0);
  particles.forEach(p => {
    ctx.save();
    ctx.globalAlpha = p.life / p.maxLife;
    ctx.fillStyle = p.color;
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.r, 0, Math.PI*2);
    ctx.fill();
    ctx.restore();
    p.x += p.vx;
    p.y += p.vy;
    p.vy += 0.15;
    p.life--;
  });
}

function spawnParticles(x, y, color, count = 12) {
  for (let i = 0; i < count; i++) {
    const a = Math.random() * Math.PI * 2;
    const spd = 1.5 + Math.random() * 3;
    particles.push({
      x, y,
      vx: Math.cos(a) * spd,
      vy: Math.sin(a) * spd - 2,
      r: 2 + Math.random() * 4,
      color,
      life: 30 + Math.random() * 20,
      maxLife: 50
    });
  }
}

function drawHUD() {
  // Score glow
  ctx.save();
  ctx.font = 'bold 20px Fredoka One';
  ctx.fillStyle = '#fff';
  ctx.textAlign = 'left';
  ctx.restore();
}

function checkCollision(cs) {
  const px = Math.max(PADDLE_W/2 + 5, Math.min(W - PADDLE_W/2 - 5, paddleX));
  const py = PADDLE_Y;

  // Hit paddle
  if (cs.y + 14 >= py - PADDLE_H/2 &&
      cs.y - 14 <= py + PADDLE_H/2 &&
      cs.x >= px - PADDLE_W/2 - 12 &&
      cs.x <= px + PADDLE_W/2 + 12) {
    spawnParticles(cs.x, py, '#ffe066', 14);
    spawnParticles(cs.x, py, '#fff', 8);
    score += 10;
    updateUI();
    return 'paddle';
  }

  // Hit ground / trees zone
  if (cs.y >= GROUND_Y - 20) {
    spawnParticles(cs.x, GROUND_Y - 10, '#e74c3c', 18);
    spawnParticles(cs.x, GROUND_Y - 10, '#ff8c00', 10);
    return 'ground';
  }

  return null;
}

function gameLoop() {
  if (!gameRunning) return;
  frameCount++;

  // Background
  ctx.clearRect(0, 0, W, H);
  drawSky();
  drawSun();

  // Clouds
  clouds.forEach(c => { drawCloud(c); c.x -= c.speed; });
  clouds = clouds.filter(c => c.x > -150);
  if (Math.random() < 0.005) spawnCloud();

  // Birds
  birds.forEach(b => { drawBird(b); b.x += b.speed; b.flap += 0.18; });
  birds = birds.filter(b => b.x < W + 50);
  if (Math.random() < 0.004) spawnBird();

  // Trees
  TREES.forEach(t => drawTree(t));

  // Ground
  drawGround();

  // Spawn chainsaws
  difficulty = 1 + score / 100;
  const spawnRate = Math.max(55 - difficulty * 4, 22);
  if (frameCount % Math.round(spawnRate) === 0) spawnChainsaw();

  // Chainsaws
  chainsaws = chainsaws.filter(cs => cs.alive);
  chainsaws.forEach(cs => {
    cs.y += cs.vy;
    cs.x += cs.vx;
    cs.angle += cs.spin;

    const hit = checkCollision(cs);
    if (hit === 'paddle') {
      cs.alive = false;
    } else if (hit === 'ground') {
      cs.alive = false;
      lives--;
      updateUI();
      if (lives <= 0) {
        gameRunning = false;
        triggerCutscene();
        return;
      }
      // Flash screen red
      ctx.fillStyle = 'rgba(255,0,0,0.18)';
      ctx.fillRect(0, 0, W, H);
    }

    if (cs.alive) {
      drawChainsaw(cs);
    }
  });

  // Paddle
  drawPaddle();

  // Particles
  drawParticles();

  requestAnimationFrame(gameLoop);
}

// ===== CUTSCENE =====
function triggerCutscene() {
  cutscene.classList.add('show');
  let cutFrame = 0;
  const duration = 260;
  const treeStates = TREES.map(() => 0); // 0=intact, grows to ~1.2

  function cutLoop() {
    cutCtx.clearRect(0, 0, W, H);

    // Dark dramatic sky
    const dgrad = cutCtx.createLinearGradient(0, 0, 0, GROUND_Y);
    dgrad.addColorStop(0, '#1a0a00');
    dgrad.addColorStop(0.5, '#3a1800');
    dgrad.addColorStop(1, '#200e00');
    cutCtx.fillStyle = dgrad;
    cutCtx.fillRect(0, 0, W, GROUND_Y);

    // Red glow on horizon
    const hg = cutCtx.createRadialGradient(W/2, GROUND_Y, 10, W/2, GROUND_Y, 300);
    hg.addColorStop(0, 'rgba(220,60,0,0.5)');
    hg.addColorStop(1, 'rgba(0,0,0,0)');
    cutCtx.fillStyle = hg;
    cutCtx.fillRect(0, 0, W, H);

    // Embers particles
    for (let i = 0; i < 8; i++) {
      const ex = (cutFrame * 3 + i * 77) % W;
      const ey = GROUND_Y - (cutFrame * 1.5 + i * 33) % (GROUND_Y - 40);
      cutCtx.save();
      cutCtx.fillStyle = `hsl(${20 + i * 10}, 100%, 60%)`;
      cutCtx.globalAlpha = 0.7;
      cutCtx.beginPath();
      cutCtx.arc(ex, ey, 2 + Math.sin(cutFrame * 0.3 + i) * 1.5, 0, Math.PI*2);
      cutCtx.fill();
      cutCtx.restore();
    }

    // Ground
    const dirtG = cutCtx.createLinearGradient(0, GROUND_Y, 0, H);
    dirtG.addColorStop(0, '#3a1a00');
    dirtG.addColorStop(1, '#1a0800');
    cutCtx.fillStyle = dirtG;
    cutCtx.fillRect(0, GROUND_Y, W, H - GROUND_Y);
    cutCtx.fillStyle = '#4a2800';
    cutCtx.fillRect(0, GROUND_Y - 14, W, 24);

    // Draw trees falling
    TREES.forEach((t, i) => {
      const delay = i * 30;
      if (cutFrame > delay) {
        treeStates[i] = Math.min(1.4, (cutFrame - delay) / 80);
      }
      drawTree(t, cutCtx, treeStates[i]);

      // Chainsaw passing over
      if (cutFrame > delay && cutFrame < delay + 50) {
        const cx = t.x + (cutFrame - delay) * 3;
        const cy = GROUND_Y - 40;
        cutCtx.save();
        cutCtx.translate(cx, cy);
        cutCtx.rotate(0.2);
        // Simple chainsaw icon
        cutCtx.fillStyle = '#aaa';
        cutCtx.beginPath();
        cutCtx.roundRect(-15, -8, 30, 16, 5);
        cutCtx.fill();
        cutCtx.fillStyle = '#e74c3c';
        cutCtx.fillRect(-6, -6, 12, 5);
        cutCtx.restore();
      }
    });

    // Smoke puffs
    for (let i = 0; i < 5; i++) {
      const sx = 80 + i * 110;
      const sy = GROUND_Y - 30 - (cutFrame * 0.8 + i * 20) % 120;
      cutCtx.save();
      cutCtx.globalAlpha = 0.25;
      cutCtx.fillStyle = '#888';
      cutCtx.beginPath();
      cutCtx.arc(sx, sy, 18 + i * 4, 0, Math.PI*2);
      cutCtx.fill();
      cutCtx.restore();
    }

    // GAME OVER text
    if (cutFrame > 60) {
      const alpha = Math.min(1, (cutFrame - 60) / 40);
      cutCtx.save();
      cutCtx.globalAlpha = alpha;
      cutCtx.font = 'bold 72px Fredoka One';
      cutCtx.textAlign = 'center';
      cutCtx.fillStyle = '#e74c3c';
      cutCtx.shadowColor = '#e74c3c';
      cutCtx.shadowBlur = 30;
      cutCtx.fillText('GAME OVER', W/2, H/2 - 40);
      cutCtx.font = '26px Nunito';
      cutCtx.fillStyle = '#ffd';
      cutCtx.shadowBlur = 10;
      cutCtx.fillText(`Pontuação: ${score}`, W/2, H/2 + 10);
      if (score > hiScore) {
        hiScore = score;
        cutCtx.fillStyle = '#ffe066';
        cutCtx.fillText('🏆 Novo Recorde!', W/2, H/2 + 45);
      } else {
        cutCtx.fillStyle = '#aaa';
        cutCtx.fillText(`Recorde: ${hiScore}`, W/2, H/2 + 45);
      }
      cutCtx.restore();
    }

    cutFrame++;
    if (cutFrame < duration) {
      requestAnimationFrame(cutLoop);
    } else {
      // Show gameover overlay
      cutscene.classList.remove('show');
      overlay.classList.remove('hidden');
      overlay.classList.add('gameover');
      overlay.querySelector('h1').textContent = '💀 GAME OVER';
      overlay.querySelector('p').innerHTML = `Sua pontuação: <b>${score}</b><br>As florestas precisam de você! Tente novamente!`;
      overlay.querySelector('button').textContent = '🔄 Jogar Novamente';
    }
  }

  cutLoop();
}
</script>
</body>
</html>
