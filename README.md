
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no"/>
  <title>Beach Volleyball Mini App</title>
  <script src="https://telegram.org/js/telegram-web-app.js"></script>
  <style>
    body, html {
      margin: 0;
      padding: 0;
      height: 100%;
      overflow: hidden;
      background: #87CEEB;
      font-family: Arial, sans-serif;
      color: white;
      touch-action: none;
    }
    #canvas {
      display: block;
      width: 100%;
      height: 100%;
    }
    #ui {
      position: absolute;
      top: 0; left: 0; right: 0;
      text-align: center;
      pointer-events: none;
      z-index: 20;
      padding-top: 20px;
    }
    #score {
      font-size: 48px;
      font-weight: bold;
      text-shadow: 3px 3px 6px #000;
    }
    #status {
      font-size: 28px;
      margin: 10px;
      text-shadow: 2px 2px 4px #000;
    }
    #restart-btn {
      position: absolute;
      bottom: 100px;
      left: 50%;
      transform: translateX(-50%);
      padding: 16px 32px;
      font-size: 24px;
      background: #FFD700;
      color: #000;
      border: none;
      border-radius: 12px;
      cursor: pointer;
      pointer-events: auto;
      z-index: 30;
      display: none;
    }
    #virtual-joystick {
      position: absolute;
      bottom: 40px;
      left: 40px;
      width: 140px;
      height: 140px;
      background: rgba(255,255,255,0.15);
      border-radius: 50%;
      pointer-events: none;
      z-index: 10;
      opacity: 0.6;
      transition: opacity 0.3s;
    }
    #jump-zone {
      position: absolute;
      bottom: 40px;
      right: 40px;
      width: 120px;
      height: 120px;
      background: rgba(0,255,0,0.2);
      border: 3px solid #0f0;
      border-radius: 20px;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 32px;
      font-weight: bold;
      pointer-events: auto;
      z-index: 10;
    }
  </style>
</head>
<body>

<canvas id="canvas"></canvas>
<div id="ui">
  <div id="score">0 : 0</div>
  <div id="status">Тапни, чтобы начать</div>
</div>
<button id="restart-btn">Новая игра</button>

<div id="jump-zone">JUMP</div>
<div id="virtual-joystick"></div>

<script>
// Telegram интеграция
const tg = window.Telegram.WebApp;
tg.ready();
tg.expand();
tg.setBackgroundColor('#87CEEB');
tg.setHeaderColor('bg_color');

// Canvas
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
const scoreEl = document.getElementById('score');
const statusEl = document.getElementById('status');
const restartBtn = document.getElementById('restart-btn');
const jumpZone = document.getElementById('jump-zone');

let width, height;
function resize() {
  width = canvas.width = window.innerWidth;
  height = canvas.height = window.innerHeight;
}
window.addEventListener('resize', resize);
resize();

// Константы
const GRAVITY = 0.45;
const NET_X = width / 2;
const NET_W = 14;
const NET_H = height * 0.55;
const PLAYER_W = 70;
const PLAYER_H = 110;
const BALL_R = 20;
const PLAYER_SPEED = 6.2;
const JUMP_FORCE = -13.5;

// Состояние
let ball = { x: width/2, y: 180, vx: 4, vy: -9, lastHitBy: null };
let player = { x: width*0.25, y: height - PLAYER_H - 50, vx: 0, vy: 0, onGround: true, score: 0 };
let ai = { x: width*0.75, y: height - PLAYER_H - 50, vx: 0, vy: 0, score: 0 };
let gameActive = false;
let keys = { left: false, right: false, jump: false };

// Управление тач + виртуальный джойстик
let touchStartX = 0;
let joystickActive = false;

canvas.addEventListener('touchstart', e => {
  e.preventDefault();
  if (!gameActive) {
    startGame();
    return;
  }
  const touch = e.touches[0];
  touchStartX = touch.clientX;
});

canvas.addEventListener('touchmove', e => {
  e.preventDefault();
  if (!gameActive) return;
  const touch = e.touches[0];
  const dx = touch.clientX - touchStartX;

  if (Math.abs(dx) > 20) {
    keys.left = dx < 0;
    keys.right = dx > 0;
  }
});

canvas.addEventListener('touchend', e => {
  e.preventDefault();
  keys.left = keys.right = false;
});

jumpZone.addEventListener('touchstart', e => {
  e.preventDefault();
  if (gameActive && player.onGround) {
    player.vy = JUMP_FORCE;
    player.onGround = false;
    tg.HapticFeedback.impactOccurred('medium');
  }
});

// Старт игры
function startGame() {
  gameActive = true;
  statusEl.textContent = '';
  restartBtn.style.display = 'none';
  resetBall();
  loop();
}

function resetBall(winner = null) {
  ball.x = width / 2 + (Math.random() * 200 - 100);
  ball.y = 120;
  ball.vx = (Math.random() > 0.5 ? 5 : -5) + (Math.random() * 2 - 1);
  ball.vy = -10;
  ball.lastHitBy = null;

  if (winner) {
    if (winner === 'player') player.score++;
    else ai.score++;
    scoreEl.textContent = `${player.score} : ${ai.score}`;

    if (player.score >= 7 || ai.score >= 7) {
      gameActive = false;
      statusEl.textContent = player.score >= 7 ? 'ТЫ ПОБЕДИЛ! 🏆' : 'Победа ИИ 😔';
      tg.HapticFeedback.notificationOccurred(player.score >= 7 ? 'success' : 'error');
      restartBtn.style.display = 'block';
      return;
    }
  }
}

restartBtn.onclick = () => {
  player.score = ai.score = 0;
  scoreEl.textContent = '0 : 0';
  statusEl.textContent = '';
  gameActive = true;
  resetBall();
};

// Главный цикл
function loop() {
  if (!gameActive) return;

  update();
  draw();

  requestAnimationFrame(loop);
}

function update() {
  // Игрок
  player.vx = 0;
  if (keys.left) player.vx = -PLAYER_SPEED;
  if (keys.right) player.vx = PLAYER_SPEED;

  player.x += player.vx;
  player.x = Math.max(20, Math.min(NET_X - PLAYER_W - 10, player.x));

  // Прыжок / гравитация
  player.y += player.vy;
  player.vy += GRAVITY;

  if (player.y >= height - PLAYER_H - 50) {
    player.y = height - PLAYER_H - 50;
    player.vy = 0;
    player.onGround = true;
  }

  // AI (улучшенный)
  const predictX = ball.x + ball.vx * 18;
  const dist = predictX - (ai.x + PLAYER_W / 2);

  if (Math.abs(dist) > 35) {
    ai.vx = dist > 0 ? PLAYER_SPEED * 0.95 : -PLAYER_SPEED * 0.95;
  } else {
    ai.vx *= 0.7;
  }

  ai.x += ai.vx;
  ai.x = Math.max(NET_X + 10, Math.min(width - PLAYER_W - 20, ai.x));

  // AI прыжок
  if (ball.y < ai.y + 60 && ball.vy > 2 && Math.abs(ball.x - ai.x - PLAYER_W/2) < 90) {
    if (player.onGround && Math.random() > 0.15) {
      ai.vy = JUMP_FORCE * (0.9 + Math.random() * 0.2);
    }
  }
  ai.y += ai.vy;
  ai.vy += GRAVITY;
  if (ai.y >= height - PLAYER_H - 50) {
    ai.y = height - PLAYER_H - 50;
    ai.vy = 0;
  }

  // Мяч
  ball.x += ball.vx;
  ball.y += ball.vy;
  ball.vy += GRAVITY;

  // Удар игрока
  if (Math.hypot(ball.x - (player.x + PLAYER_W/2), ball.y - (player.y + PLAYER_H/2)) < BALL_R + 50) {
    ball.lastHitBy = 'player';
    const dx = (ball.x - (player.x + PLAYER_W/2)) / 35;
    ball.vx = dx * 8 + player.vx * 0.4;
    ball.vy = -Math.abs(ball.vy * 0.82) - 5.5;
    tg.HapticFeedback.impactOccurred('light');
  }

  // Удар AI
  if (Math.hypot(ball.x - (ai.x + PLAYER_W/2), ball.y - (ai.y + PLAYER_H/2)) < BALL_R + 50) {
    ball.lastHitBy = 'ai';
    const dx = (ball.x - (ai.x + PLAYER_W/2)) / 40;
    ball.vx = dx * 7 + (Math.random() - 0.5) * 1.5;
    ball.vy = -Math.abs(ball.vy * 0.78) - 4.8;
  }

  // Отскок от стен
  if (ball.x < BALL_R || ball.x > width - BALL_R) {
    ball.vx *= -0.82;
    ball.x = ball.x < BALL_R ? BALL_R : width - BALL_R;
  }

  // Пропуск мяча → очко
  if (ball.y > height + BALL_R * 2) {
    resetBall(ball.lastHitBy === 'player' ? 'player' : 'ai');
  }

  // Сетка
  if (Math.abs(ball.x - NET_X) < BALL_R + NET_W/2 && ball.y > height - NET_H - 30) {
    ball.vx *= -0.88;
    ball.x += ball.vx * 1.5;
  }
}

function draw() {
  ctx.clearRect(0, 0, width, height);

  // Песок
  ctx.fillStyle = '#F4C430';
  ctx.fillRect(0, height - 140, width, 140);

  // Небо
  const gr = ctx.createLinearGradient(0, 0, 0, height - 140);
  gr.addColorStop(0, '#87CEFA');
  gr.addColorStop(1, '#E0FFFF');
  ctx.fillStyle = gr;
  ctx.fillRect(0, 0, width, height - 140);

  // Сетка
  ctx.fillStyle = '#8B4513';
  ctx.fillRect(NET_X - NET_W/2, height - NET_H - 50, NET_W, NET_H);

  // Игрок (синий)
  ctx.fillStyle = '#1E90FF';
  ctx.fillRect(player.x, player.y, PLAYER_W, PLAYER_H);
  ctx.fillStyle = '#FFFFFF';
  ctx.beginPath();
  ctx.arc(player.x + PLAYER_W/2, player.y + 35, 28, 0, Math.PI*2);
  ctx.fill();

  // AI (красный)
  ctx.fillStyle = '#FF4500';
  ctx.fillRect(ai.x, ai.y, PLAYER_W, PLAYER_H);
  ctx.fillStyle = '#FFFFFF';
  ctx.beginPath();
  ctx.arc(ai.x + PLAYER_W/2, ai.y + 35, 28, 0, Math.PI*2);
  ctx.fill();

  // Мяч
  ctx.fillStyle = '#FFFF00';
  ctx.beginPath();
  ctx.arc(ball.x, ball.y, BALL_R, 0, Math.PI*2);
  ctx.fill();
  ctx.strokeStyle = '#FFD700';
  ctx.lineWidth = 5;
  ctx.stroke();
}

// Запуск
resize();
draw(); // начальный кадр

// Рестарт по кнопке Telegram MainButton
tg.MainButton.setText('Новая игра');
tg.MainButton.onClick(() => {
  if (!gameActive) {
    player.score = ai.score = 0;
    scoreEl.textContent = '0 : 0';
    statusEl.textContent = '';
    gameActive = true;
    resetBall();
    tg.MainButton.hide();
  }
});
</script>
</body>
</html>