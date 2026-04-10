# Dinotmp2
'''


<!DOCTYPE html>

<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Dino Run!</title>
<style>
  @import url('https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap');

- { margin: 0; padding: 0; box-sizing: border-box; }

body {
background: #1a1a2e;
display: flex;
justify-content: center;
align-items: center;
min-height: 100vh;
font-family: ‘Press Start 2P’, monospace;
overflow: hidden;
}

#wrapper {
position: relative;
background: #0f0f23;
border: 3px solid #16213e;
border-radius: 12px;
box-shadow:
0 0 40px rgba(0, 200, 255, 0.08),
inset 0 0 60px rgba(0, 0, 0, 0.4);
overflow: hidden;
}

canvas {
display: block;
}

#ui-overlay {
position: absolute;
top: 16px;
right: 20px;
color: #e0e0e0;
font-size: 14px;
text-shadow: 0 0 8px rgba(0, 200, 255, 0.4);
z-index: 10;
text-align: right;
}

#ui-overlay .label {
font-size: 8px;
color: #607080;
margin-bottom: 2px;
}

#ui-overlay .hi-score {
font-size: 10px;
color: #505868;
margin-top: 4px;
}

#start-screen, #game-over-screen {
position: absolute;
inset: 0;
display: flex;
flex-direction: column;
justify-content: center;
align-items: center;
background: rgba(15, 15, 35, 0.85);
z-index: 20;
gap: 16px;
transition: opacity 0.3s;
}

#start-screen h1 {
font-size: 28px;
color: #00c8ff;
text-shadow: 0 0 20px rgba(0, 200, 255, 0.5);
letter-spacing: 4px;
}

#start-screen .sub, #game-over-screen .sub {
font-size: 10px;
color: #607080;
animation: blink 1.2s infinite;
}

#game-over-screen h2 {
font-size: 20px;
color: #ff6b6b;
text-shadow: 0 0 15px rgba(255, 107, 107, 0.4);
}

#game-over-screen .final-score {
font-size: 14px;
color: #e0e0e0;
}

@keyframes blink {
0%, 100% { opacity: 1; }
50% { opacity: 0.3; }
}
</style>

</head>
<body>

<div id="wrapper">
  <canvas id="game"></canvas>

  <div id="ui-overlay">
    <div class="label">SCORE</div>
    <div id="score-display">0</div>
    <div class="hi-score">HI <span id="hi-score-display">0</span></div>
  </div>

  <div id="start-screen">
    <h1>DINO RUN</h1>
    <div class="sub">SPACE / TAP TO START</div>
  </div>

  <div id="game-over-screen" style="display:none;">
    <h2>GAME OVER</h2>
    <div class="final-score">SCORE: <span id="final-score">0</span></div>
    <div class="sub">SPACE / TAP TO RETRY</div>
  </div>
</div>

<script>
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');

// responsive sizing
const W = Math.min(800, window.innerWidth - 32);
const H = 260;
canvas.width = W;
canvas.height = H;
document.getElementById('wrapper').style.width = W + 6 + 'px';

const GROUND_Y = H - 40;
const GRAVITY = 0.6;
const JUMP_FORCE = -12;
const MIN_OBSTACLE_GAP = 260;

let state = 'start'; // start | playing | over
let score = 0;
let hiScore = parseInt(localStorage.getItem('dinoHi') || '0');
let speed = 5;
let frameCount = 0;
let lastObstacleX = W;

document.getElementById('hi-score-display').textContent = hiScore;

// Dino
const dino = {
  x: 60, y: GROUND_Y, w: 32, h: 36,
  vy: 0, grounded: true, ducking: false,
  legFrame: 0, legTimer: 0,
};

// Obstacles
let obstacles = [];

// Particles (ground dust)
let particles = [];

// Stars background
const stars = Array.from({ length: 60 }, () => ({
  x: Math.random() * W,
  y: Math.random() * (GROUND_Y - 20),
  r: Math.random() * 1.5 + 0.3,
  speed: Math.random() * 0.3 + 0.1,
  brightness: Math.random(),
}));

// Clouds
const clouds = Array.from({ length: 4 }, () => ({
  x: Math.random() * W,
  y: 30 + Math.random() * 80,
  w: 40 + Math.random() * 50,
  speed: 0.3 + Math.random() * 0.4,
}));

// ---- Drawing helpers ----
function drawDino() {
  const d = dino;
  const cy = d.ducking ? GROUND_Y : d.y;
  const ch = d.ducking ? 20 : d.h;
  const cw = d.ducking ? 40 : d.w;
  const bx = d.x;
  const by = cy - ch;

  ctx.fillStyle = '#88ee88';
  ctx.shadowColor = 'rgba(100, 255, 100, 0.3)';
  ctx.shadowBlur = 8;

  // body
  ctx.fillRect(bx + 4, by + 4, cw - 8, ch - 8);
  // head
  if (!d.ducking) {
    ctx.fillRect(bx + 14, by - 6, 16, 12);
    // eye
    ctx.fillStyle = '#1a1a2e';
    ctx.fillRect(bx + 24, by - 3, 4, 4);
    ctx.fillStyle = '#88ee88';
  } else {
    ctx.fillRect(bx + cw - 18, by - 2, 16, 10);
    ctx.fillStyle = '#1a1a2e';
    ctx.fillRect(bx + cw - 6, by, 4, 4);
    ctx.fillStyle = '#88ee88';
  }

  // legs
  if (d.grounded) {
    d.legTimer++;
    if (d.legTimer > 6) { d.legTimer = 0; d.legFrame = 1 - d.legFrame; }
    if (d.ducking) {
      const off = d.legFrame === 0 ? 0 : 4;
      ctx.fillRect(bx + 6 + off, cy - 2, 5, 6);
      ctx.fillRect(bx + 18 - off, cy - 2, 5, 6);
    } else {
      const off = d.legFrame === 0 ? 0 : 4;
      ctx.fillRect(bx + 6 + off, cy - 4, 5, 8);
      ctx.fillRect(bx + 16 - off, cy - 4, 5, 8);
    }
  } else {
    ctx.fillRect(bx + 8, cy - 4, 5, 8);
    ctx.fillRect(bx + 18, cy - 4, 5, 8);
  }

  ctx.shadowBlur = 0;
}

function drawCactus(o) {
  ctx.fillStyle = '#ff6b6b';
  ctx.shadowColor = 'rgba(255, 107, 107, 0.3)';
  ctx.shadowBlur = 6;
  const bx = o.x, by = GROUND_Y - o.h;
  ctx.fillRect(bx + 2, by, o.w - 4, o.h);
  // arms
  if (o.h > 30) {
    ctx.fillRect(bx - 6, by + 10, 8, 4);
    ctx.fillRect(bx - 6, by + 10, 4, 14);
    ctx.fillRect(bx + o.w - 2, by + 16, 8, 4);
    ctx.fillRect(bx + o.w + 2, by + 16, 4, 12);
  }
  ctx.shadowBlur = 0;
}

function drawBird(o) {
  ctx.fillStyle = '#ffcc00';
  ctx.shadowColor = 'rgba(255, 204, 0, 0.3)';
  ctx.shadowBlur = 6;
  const bx = o.x, by = o.y;
  ctx.fillRect(bx, by + 4, 24, 8);
  // wing flap
  const wingY = Math.sin(frameCount * 0.2) > 0 ? -6 : 6;
  ctx.fillRect(bx + 6, by + 4 + wingY, 12, 4);
  // beak
  ctx.fillStyle = '#ff8800';
  ctx.fillRect(bx + 24, by + 6, 6, 4);
  ctx.shadowBlur = 0;
}

function drawGround() {
  ctx.fillStyle = '#2a2a4a';
  ctx.fillRect(0, GROUND_Y, W, 2);
  // moving dashes
  const offset = (frameCount * speed) % 20;
  ctx.fillStyle = '#1e1e3a';
  for (let x = -offset; x < W; x += 20) {
    ctx.fillRect(x, GROUND_Y + 6, 8, 1);
  }
}

function drawParticles() {
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i];
    p.x += p.vx;
    p.y += p.vy;
    p.life -= 0.03;
    if (p.life <= 0) { particles.splice(i, 1); continue; }
    ctx.globalAlpha = p.life * 0.6;
    ctx.fillStyle = '#606880';
    ctx.fillRect(p.x, p.y, 2, 2);
  }
  ctx.globalAlpha = 1;
}

function drawStars() {
  for (const s of stars) {
    s.brightness += (Math.random() - 0.5) * 0.05;
    s.brightness = Math.max(0.2, Math.min(1, s.brightness));
    ctx.globalAlpha = s.brightness * 0.6;
    ctx.fillStyle = '#aabbcc';
    ctx.beginPath();
    ctx.arc(s.x, s.y, s.r, 0, Math.PI * 2);
    ctx.fill();
    if (state === 'playing') {
      s.x -= s.speed * (speed / 5);
      if (s.x < 0) s.x = W;
    }
  }
  ctx.globalAlpha = 1;
}

function drawClouds() {
  ctx.fillStyle = 'rgba(40, 40, 70, 0.5)';
  for (const c of clouds) {
    ctx.beginPath();
    ctx.ellipse(c.x, c.y, c.w / 2, 10, 0, 0, Math.PI * 2);
    ctx.fill();
    if (state === 'playing') {
      c.x -= c.speed * (speed / 5);
      if (c.x + c.w / 2 < 0) { c.x = W + c.w; c.y = 30 + Math.random() * 80; }
    }
  }
}

// ---- Game logic ----
function spawnObstacle() {
  const distFromLast = W - lastObstacleX + W;
  if (lastObstacleX > W - MIN_OBSTACLE_GAP / (speed / 5)) return;
  if (Math.random() < 0.02 * (speed / 5)) {
    const isBird = speed > 6 && Math.random() < 0.3;
    if (isBird) {
      const birdY = [GROUND_Y - 50, GROUND_Y - 28, GROUND_Y - 70][Math.floor(Math.random() * 3)];
      obstacles.push({ type: 'bird', x: W, y: birdY, w: 30, h: 16 });
    } else {
      const h = 20 + Math.random() * 26;
      const w = 14 + Math.random() * 10;
      obstacles.push({ type: 'cactus', x: W, y: GROUND_Y, w, h });
    }
    lastObstacleX = W;
  }
}

function updateObstacles() {
  for (let i = obstacles.length - 1; i >= 0; i--) {
    const o = obstacles[i];
    o.x -= speed;
    if (o.x + o.w < 0) obstacles.splice(i, 1);
  }
  lastObstacleX -= speed;
}

function collides(o) {
  const dx = dino.x + 4;
  const dw = (dino.ducking ? 36 : dino.w - 8);
  const dh = dino.ducking ? 18 : dino.h - 4;
  const dy = (dino.ducking ? GROUND_Y : dino.y) - dh;

  let ox, oy, ow, oh;
  if (o.type === 'cactus') {
    ox = o.x + 4; oy = GROUND_Y - o.h + 2; ow = o.w - 8; oh = o.h - 2;
  } else {
    ox = o.x + 2; oy = o.y + 2; ow = o.w - 4; oh = o.h - 4;
  }

  return dx < ox + ow && dx + dw > ox && dy < oy + oh && dy + dh > oy;
}

function jump() {
  if (dino.grounded) {
    dino.vy = JUMP_FORCE;
    dino.grounded = false;
    // dust
    for (let i = 0; i < 6; i++) {
      particles.push({
        x: dino.x + 10 + Math.random() * 10,
        y: GROUND_Y - 2,
        vx: -Math.random() * 2 - 1,
        vy: -Math.random() * 2,
        life: 1,
      });
    }
  }
}

function reset() {
  dino.y = GROUND_Y;
  dino.vy = 0;
  dino.grounded = true;
  dino.ducking = false;
  obstacles = [];
  particles = [];
  score = 0;
  speed = 5;
  frameCount = 0;
  lastObstacleX = W;
  document.getElementById('score-display').textContent = '0';
}

// ---- Input ----
const keys = {};
document.addEventListener('keydown', e => {
  keys[e.code] = true;
  if (e.code === 'Space' || e.code === 'ArrowUp') {
    e.preventDefault();
    if (state === 'start') { state = 'playing'; document.getElementById('start-screen').style.display = 'none'; }
    else if (state === 'over') { reset(); state = 'playing'; document.getElementById('game-over-screen').style.display = 'none'; }
    else if (state === 'playing') jump();
  }
  if (e.code === 'ArrowDown' && state === 'playing') {
    dino.ducking = true;
    if (!dino.grounded) dino.vy += 0.8; // fast fall
  }
});
document.addEventListener('keyup', e => {
  keys[e.code] = false;
  if (e.code === 'ArrowDown') dino.ducking = false;
});

canvas.addEventListener('touchstart', e => {
  e.preventDefault();
  if (state === 'start') { state = 'playing'; document.getElementById('start-screen').style.display = 'none'; }
  else if (state === 'over') { reset(); state = 'playing'; document.getElementById('game-over-screen').style.display = 'none'; }
  else if (state === 'playing') jump();
});

// ---- Main loop ----
function loop() {
  ctx.clearRect(0, 0, W, H);

  drawStars();
  drawClouds();
  drawGround();

  if (state === 'playing') {
    frameCount++;

    // dino physics
    if (!dino.grounded) {
      dino.vy += GRAVITY;
      dino.y += dino.vy;
      if (dino.y >= GROUND_Y) {
        dino.y = GROUND_Y;
        dino.vy = 0;
        dino.grounded = true;
      }
    }

    spawnObstacle();
    updateObstacles();

    // collision
    for (const o of obstacles) {
      if (collides(o)) {
        state = 'over';
        if (score > hiScore) {
          hiScore = score;
          localStorage.setItem('dinoHi', hiScore);
          document.getElementById('hi-score-display').textContent = hiScore;
        }
        document.getElementById('final-score').textContent = score;
        document.getElementById('game-over-screen').style.display = 'flex';
      }
    }

    // score & speed
    score = Math.floor(frameCount / 6);
    document.getElementById('score-display').textContent = score;
    speed = 5 + score * 0.008;
  }

  // draw obstacles
  for (const o of obstacles) {
    if (o.type === 'cactus') drawCactus(o);
    else drawBird(o);
  }

  drawDino();
  drawParticles();

  requestAnimationFrame(loop);
}

loop();
</script>

</body>
</html>
'''
