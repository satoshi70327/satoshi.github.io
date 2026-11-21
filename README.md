<!DOCTYPE html>
<html lang="zh-Hant">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<title>霓虹彈珠台 NEON PINBALL</title>
<style>
  * {
    box-sizing: border-box;
    user-select: none;
    -webkit-user-select: none;
  }
  body {
    margin: 0;
    padding: 0;
    background-color: #090a0f;
    color: #fff;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 100vh;
    overflow: hidden;
  }
  #game-wrapper {
    position: relative;
    display: flex;
    flex-direction: column;
    align-items: center;
    box-shadow: 0 0 40px rgba(0, 243, 255, 0.25);
    border-radius: 16px;
    border: 2px solid #00f3ff;
    background: #11131f;
    padding: 10px;
  }
  #hud {
    width: 100%;
    display: flex;
    justify-content: space-between;
    padding: 5px 15px;
    font-size: 18px;
    font-weight: bold;
    letter-spacing: 1px;
    text-shadow: 0 0 8px rgba(0, 243, 255, 0.8);
  }
  .hud-val { color: #00f3ff; }
  .hud-balls { color: #ff007f; text-shadow: 0 0 8px rgba(255, 0, 127, 0.8); }
  canvas {
    background: radial-gradient(circle at center, #1b1e32 0%, #0d0e17 100%);
    border-radius: 8px;
    display: block;
    max-height: 75vh;
    width: auto;
  }
  #controls-hint {
    margin-top: 8px;
    font-size: 13px;
    color: #8a94b8;
    text-align: center;
  }
  #mobile-controls {
    display: none;
    width: 100%;
    justify-content: space-between;
    gap: 8px;
    margin-top: 10px;
  }
  .touch-btn {
    flex: 1;
    padding: 15px 0;
    background: rgba(0, 243, 255, 0.15);
    border: 1px solid #00f3ff;
    color: #00f3ff;
    font-weight: bold;
    font-size: 16px;
    border-radius: 8px;
    text-align: center;
    cursor: pointer;
  }
  .touch-btn:active { background: rgba(0, 243, 255, 0.4); }
  .touch-btn.launch {
    border-color: #ff007f;
    color: #ff007f;
    background: rgba(255, 0, 127, 0.15);
  }
  .touch-btn.launch:active { background: rgba(255, 0, 127, 0.4); }

  #overlay {
    position: absolute;
    inset: 0;
    background: rgba(9, 10, 15, 0.88);
    border-radius: 14px;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    text-align: center;
    padding: 20px;
    z-index: 10;
  }
  #overlay h1 {
    font-size: 32px;
    margin: 0 0 10px;
    color: #00f3ff;
    text-shadow: 0 0 15px #00f3ff;
  }
  #overlay p {
    color: #cbd5e1;
    margin: 5px 0 20px;
    line-height: 1.5;
  }
  .btn-group {
    display: flex;
    gap: 12px;
  }
  button.action-btn {
    padding: 12px 24px;
    font-size: 16px;
    font-weight: bold;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    background: #00f3ff;
    color: #090a0f;
    box-shadow: 0 0 15px rgba(0, 243, 255, 0.6);
    transition: transform 0.1s;
  }
  button.action-btn:active { transform: scale(0.95); }
  button.share-btn {
    background: #ff007f;
    color: #fff;
    box-shadow: 0 0 15px rgba(255, 0, 127, 0.6);
  }
  @media (pointer: coarse) {
    #mobile-controls { display: flex; }
    #controls-hint { display: none; }
  }
</style>
</head>
<body>

<div id="game-wrapper">
  <div id="hud">
    <div>SCORE: <span id="score" class="hud-val">0</span></div>
    <div>HIGH: <span id="highscore" class="hud-val">0</span></div>
    <div>BALLS: <span id="balls" class="hud-balls">3</span></div>
  </div>

  <canvas id="pinball" width="400" height="650"></canvas>

  <div id="controls-hint">
    電腦操作：<b>← / →</b> 控制擋板 ｜ <b>空白鍵 (長按蓄力)</b> 發射彈珠
  </div>

  <div id="mobile-controls">
    <div class="touch-btn" id="btn-left">左擋板</div>
    <div class="touch-btn launch" id="btn-launch">長按發射</div>
    <div class="touch-btn" id="btn-right">右擋板</div>
  </div>

  <div id="overlay">
    <h1 id="overlay-title">NEON PINBALL</h1>
    <p id="overlay-desc">撞擊發光圓柱獲得高分！<br>準備好挑戰最高紀錄了嗎？</p>
    <div class="btn-group">
      <button class="action-btn" id="start-btn">開始遊戲</button>
      <button class="action-btn share-btn" id="share-btn" style="display:none;">複製戰績分享</button>
    </div>
  </div>
</div>

<script>
const canvas = document.getElementById('pinball');
const ctx = canvas.getContext('2d');
const scoreEl = document.getElementById('score');
const highEl = document.getElementById('highscore');
const ballsEl = document.getElementById('balls');
const overlay = document.getElementById('overlay');
const overlayTitle = document.getElementById('overlay-title');
const overlayDesc = document.getElementById('overlay-desc');
const startBtn = document.getElementById('start-btn');
const shareBtn = document.getElementById('share-btn');

// 簡易音效合成器 (無需外部音檔)
const AudioCtx = window.AudioContext || window.webkitAudioContext;
let audioCtx;
function playTone(freq, type = 'sine', duration = 0.1, vol = 0.15) {
  if (!audioCtx) audioCtx = new AudioCtx();
  if (audioCtx.state === 'suspended') audioCtx.resume();
  const osc = audioCtx.createOscillator();
  const gain = audioCtx.createGain();
  osc.type = type;
  osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
  gain.gain.setValueAtTime(vol, audioCtx.currentTime);
  gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + duration);
  osc.connect(gain);
  gain.connect(audioCtx.destination);
  osc.start();
  osc.stop(audioCtx.currentTime + duration);
}

// 遊戲狀態
let score = 0;
let highScore = localStorage.getItem('neon_pinball_high') || 0;
highEl.textContent = highScore;
let balls = 3;
let isPlaying = false;

// 物理參數
const GRAVITY = 0.22;
const FRICTION = 0.996;
const SUBSTEPS = 6;

// 彈珠物件
const ball = {
  x: 378, y: 550, vx: 0, vy: 0, radius: 9, inLauncher: true
};

// 發射器
const launcher = {
  x: 365, y: 570, w: 26, h: 60, power: 0, charging: false
};

// 彈跳器 (Bumpers)
const bumpers = [
  { x: 185, y: 150, r: 24, score: 100, color: '#ff007f', pulse: 0 },
  { x: 115, y: 210, r: 22, score: 100, color: '#00f3ff', pulse: 0 },
  { x: 255, y: 210, r: 22, score: 100, color: '#00f3ff', pulse: 0 },
  { x: 185, y: 270, r: 20, score: 250, color: '#ffe600', pulse: 0 },
  { x: 95,  y: 110, r: 16, score: 150, color: '#a855f7', pulse: 0 },
  { x: 275, y: 110, r: 16, score: 150, color: '#a855f7', pulse: 0 }
];

// 靜態牆壁線段 (x1, y1, x2, y2)
const walls = [
  // 外框
  { x1: 15, y1: 80, x2: 15, y2: 560 },
  { x1: 390, y1: 80, x2: 390, y2: 630 },
  { x1: 360, y1: 160, x2: 360, y2: 630 }, // 發射道內牆
  // 左下導流斜坡
  { x1: 15, y1: 480, x2: 105, y2: 555 },
  // 右下導流斜坡
  { x1: 360, y1: 480, x2: 265, y2: 555 },
  // 上方三角分流器
  { x1: 135, y1: 55, x2: 155, y2: 85 },
  { x1: 215, y1: 55, x2: 235, y2: 85 },
  // 兩側三角彈跳擋板 (Slingshots)
  { x1: 50, y1: 380, x2: 50, y2: 455 },
  { x1: 50, y1: 455, x2: 95, y2: 490 },
  { x1: 95, y1: 490, x2: 50, y2: 380, bouncy: 1.4, score: 50 },

  { x1: 325, y1: 380, x2: 325, y2: 455 },
  { x1: 325, y1: 455, x2: 280, y2: 490 },
  { x1: 280, y1: 490, x2: 325, y2: 380, bouncy: 1.4, score: 50 }
];

// 擋板 (Flippers)
const flippers = {
  left:  { x: 105, y: 555, len: 68, angle: 0.45, targetAngle: 0.45, restAngle: 0.45, activeAngle: -0.45, angularVel: 0 },
  right: { x: 265, y: 555, len: 68, angle: Math.PI - 0.45, targetAngle: Math.PI - 0.45, restAngle: Math.PI - 0.45, activeAngle: Math.PI + 0.45, angularVel: 0 }
};

// 粒子特效
let particles = [];
function spawnParticles(x, y, color, count = 12) {
  for (let i = 0; i < count; i++) {
    const a = Math.random() * Math.PI * 2;
    const spd = Math.random() * 4 + 1;
    particles.push({
      x, y, vx: Math.cos(a) * spd, vy: Math.sin(a) * spd,
      life: 1, color
    });
  }
}

// 控制輸入
const keys = { left: false, right: false, space: false };

window.addEventListener('keydown', e => {
  if (e.code === 'ArrowLeft' || e.code === 'KeyA') {
    if (!keys.left && isPlaying) playTone(180, 'triangle', 0.08, 0.1);
    keys.left = true;
  }
  if (e.code === 'ArrowRight' || e.code === 'KeyD') {
    if (!keys.right && isPlaying) playTone(180, 'triangle', 0.08, 0.1);
    keys.right = true;
  }
  if (e.code === 'Space') {
    e.preventDefault();
    keys.space = true;
  }
});

window.addEventListener('keyup', e => {
  if (e.code === 'ArrowLeft' || e.code === 'KeyA') keys.left = false;
  if (e.code === 'ArrowRight' || e.code === 'KeyD') keys.right = false;
  if (e.code === 'Space') {
    if (launcher.charging && ball.inLauncher) releaseBall();
    keys.space = false;
  }
});

// 手機觸控支援
function bindTouch(id, keyName, isLaunch = false) {
  const el = document.getElementById(id);
  el.addEventListener('touchstart', e => {
    e.preventDefault();
    keys[keyName] = true;
    if (!isLaunch && isPlaying) playTone(180, 'triangle', 0.08, 0.1);
  });
  el.addEventListener('touchend', e => {
    e.preventDefault();
    if (isLaunch && launcher.charging && ball.inLauncher) releaseBall();
    keys[keyName] = false;
  });
}
bindTouch('btn-left', 'left');
bindTouch('btn-right', 'right');
bindTouch('btn-launch', 'space', true);

function releaseBall() {
  ball.vy = -(10 + launcher.power * 11);
  ball.vx = -0.3;
  ball.inLauncher = false;
  launcher.charging = false;
  launcher.power = 0;
  playTone(320, 'sawtooth', 0.25, 0.2);
}

function resetBall() {
  ball.x = 375;
  ball.y = 550;
  ball.vx = 0;
  ball.vy = 0;
  ball.inLauncher = true;
}

function addScore(pts) {
  score += pts;
  scoreEl.textContent = score;
  if (score > highScore) {
    highScore = score;
    highEl.textContent = highScore;
    localStorage.setItem('neon_pinball_high', highScore);
  }
}

// 點到線段最近點計算
function closestPointOnSegment(px, py, x1, y1, x2, y2) {
  const dx = x2 - x1;
  const dy = y2 - y1;
  const lenSq = dx * dx + dy * dy;
  let t = ((px - x1) * dx + (py - y1) * dy) / lenSq;
  t = Math.max(0, Math.min(1, t));
  return { x: x1 + t * dx, y: y1 + t * dy, t };
}

function updatePhysics() {
  if (!isPlaying) return;

  // 發射器蓄力
  if (keys.space && ball.inLauncher) {
    launcher.charging = true;
